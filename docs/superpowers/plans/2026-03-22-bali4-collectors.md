# BALI 4.0 Collectors Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Построить систему сбора рыночных данных с Redis-хранилищем, JSON-логами с latency-метриками и детектором спредов, основанную на общем core-модуле без дублирования кода.

**Architecture:** Три абстрактных базовых класса (`BaseMD`, `BaseOB`, `BaseFR`) в `collectors/core/` инкапсулируют WS-логику, батчинг, Redis-запись и логирование. Каждая биржа — маленький файл (~60 строк) с `parse()` и `build_url()`. Spread monitor читает Redis каждые 300ms и пишет сигналы в JSONL/CSV.

**Tech Stack:** Python 3.10+, asyncio, websockets, redis.asyncio, orjson

---

## Структура файлов

```
collectors/
  core/
    __init__.py
    logger.py           # JSON-логгер с latency-трекером
    redis_writer.py     # flush() + Redis pipeline логика
    base_md.py          # абстрактный MD-коллектор (best bid/ask)
    base_ob.py          # абстрактный OB-коллектор (level 10)
    base_fr.py          # абстрактный FR-коллектор (funding rate)
  md/
    __init__.py
    binance_spot.py     binance_futures.py
    bybit_spot.py       bybit_futures.py
    okx_spot.py         okx_futures.py
    gate_spot.py        gate_futures.py
    bitget_spot.py      bitget_futures.py
  ob/
    __init__.py
    binance_spot.py     binance_futures.py
    bybit_spot.py       bybit_futures.py
    okx_spot.py         okx_futures.py
    gate_spot.py        gate_futures.py
    bitget_spot.py      bitget_futures.py
  fr/
    __init__.py
    binance_futures.py
    bybit_futures.py
    okx_futures.py
    gate_futures.py     # REST polling (aiohttp), НЕ WebSocket
    bitget_futures.py
  hist_writer.py        # MD history writer (ZSET, из 3.0)
  ob_hist_writer.py     # OB history writer (ZSET, из 3.0)
  fr_hist_writer.py     # FR history writer (ZSET, из 3.0)
  spread_monitor.py     # детектор спредов → signals/
  staleness_monitor.py  # проверка свежести данных из Redis
runner_md.py            # запускает все 10 MD как asyncio-задачи
runner_ob.py            # запускает все 10 OB как asyncio-задачи
runner_fr.py            # запускает все 5 FR как asyncio-задачи
runner_monitors.py      # запускает spread_monitor + staleness
run_all.py              # оркестратор: subprocess + CPU affinity + дашборд
```

**Redis ключи (без изменений vs 3.0):**
- `md:{ex}:{mkt}:{sym}` → HASH `{b, a, t}`
- `ob:{ex}:{mkt}:{sym}` → HASH `{b1..b10, bq1..bq10, a1..a10, aq1..aq10, t}`
- `fr:{ex}:{sym}` → HASH `{r, nr, ft, ts}`

**Коды бирж:** `bn`=Binance, `bb`=Bybit, `ok`=OKX, `gt`=Gate, `bg`=Bitget
**Коды рынков:** `s`=Spot, `f`=Futures

---

## Task 1: core/logger.py — JSON-логгер

**Files:**
- Create: `collectors/core/__init__.py`
- Create: `collectors/core/logger.py`

- [ ] **Шаг 1: Создать `collectors/core/__init__.py`** (пустой файл)

- [ ] **Шаг 2: Написать `collectors/core/logger.py`**

```python
#!/usr/bin/env python3
import sys
import time
import orjson

def make_logger(script: str):
    """
    Возвращает функцию log(lvl, event, **kw) которая пишет JSON-строку в stdout.
    Все поля latency передаются через **kw.
    """
    def log(lvl: str, event: str, **kw) -> None:
        rec = {"ts": time.time(), "lvl": lvl, "script": script, "event": event, **kw}
        sys.stdout.buffer.write(orjson.dumps(rec) + b"\n")
        sys.stdout.buffer.flush()
    return log
```

**Стандартные события и их поля:**

| event | обязательные поля |
|-------|-------------------|
| `startup` | — |
| `symbols_loaded` | `count`, `file` |
| `redis_connected` | `socket` |
| `connecting` | `chunk_id`, `chunk_symbols` |
| `connected` | `chunk_id`, `chunk_symbols` |
| `first_message` | `chunk_id`, `ms_since_connected`, `first_sym` |
| `stats` | `msgs_total`, `msgs_per_sec`, `flushes_total`, `avg_pipeline_ms`, `tick_age_avg_ms`, `tick_age_max_ms`, `symbols_active` |
| `slow_flush` | `pipeline_ms` |
| `slow_batch` | `batch_age_ms`, `batch_size` |
| `disconnected` | `chunk_id`, `reason` |
| `reconnecting` | `chunk_id`, `delay_sec`, `attempt` |
| `reconnected` | `chunk_id`, `downtime_sec` |
| `shutdown` | — |

- [ ] **Шаг 3: Коммит**
```bash
git add collectors/core/
git commit -m "feat: add core logger module"
```

---

## Task 2: core/redis_writer.py — flush + pipeline

**Files:**
- Create: `collectors/core/redis_writer.py`

- [ ] **Шаг 1: Написать `collectors/core/redis_writer.py`**

```python
#!/usr/bin/env python3
import time
import redis.asyncio as aioredis

SLOW_FLUSH_MS = 50.0   # порог для события slow_flush

async def flush_pipeline(
    r: aioredis.Redis,
    batch: list,          # list of (key: bytes, mapping: dict)
    counters: dict,
    log,                  # функция log из make_logger
    oldest_recv_ms: float = 0.0,
    hist_writer=None,     # опционально: HistWriter/OBHistWriter/FrHistWriter
) -> None:
    """
    Выполняет Redis pipeline для всего батча.
    Обновляет counters: flushes, lat_total, tick_age_sum/count/max, symbols_active_set.
    Логирует slow_flush если pipeline > SLOW_FLUSH_MS.
    """
    t0 = time.monotonic()
    flush_wall_ms = time.time() * 1000
    now_sec = time.time()

    async with r.pipeline(transaction=False) as pipe:
        for key, mapping in batch:
            pipe.hset(key, mapping=mapping)
        if hist_writer is not None:
            hist_writer.add_to_pipe(pipe, batch, now_sec)
            hist_writer.ensure_config(pipe, now_sec)
        await pipe.execute()

    lat_ms = (time.monotonic() - t0) * 1000
    counters["flushes"] += 1
    counters["lat_total"] += lat_ms

    # E2E tick age: от получения первого сообщения батча до записи в Redis
    if oldest_recv_ms:
        tick_ms = flush_wall_ms - oldest_recv_ms
        counters["tick_age_sum"]   += tick_ms
        counters["tick_age_count"] += 1
        if tick_ms > counters["tick_age_max"]:
            counters["tick_age_max"] = tick_ms

    if lat_ms > SLOW_FLUSH_MS:
        log("WARN", "slow_flush", pipeline_ms=round(lat_ms, 2))


def make_counters() -> dict:
    """Начальные значения счётчиков для любого коллектора."""
    return {
        "msgs":           0,
        "flushes":        0,
        "lat_total":      0.0,
        "tick_age_sum":   0.0,
        "tick_age_count": 0,
        "tick_age_max":   0.0,
        "symbols_active": set(),   # символы обновлённые за текущий stats-интервал
    }
```

- [ ] **Шаг 2: Коммит**
```bash
git add collectors/core/redis_writer.py
git commit -m "feat: add core redis_writer module"
```

---

## Task 3: core/base_md.py — базовый MD-коллектор

**Files:**
- Create: `collectors/core/base_md.py`

Базовый класс реализует весь WS-цикл: connect → recv → batch → flush.
Подклассы переопределяют только `build_url(syms)` и `parse(raw) → (sym, bid, ask) | None`.

- [ ] **Шаг 1: Написать `collectors/core/base_md.py`**

```python
#!/usr/bin/env python3
import asyncio
import random
import time
from abc import ABC, abstractmethod

import orjson
import redis.asyncio as aioredis
import websockets
from websockets.exceptions import ConnectionClosed

from .redis_writer import flush_pipeline, make_counters

BATCH_TIMEOUT   = 0.200 + random.uniform(0, 0.100)
MAX_BATCH       = 400
STATS_INTERVAL  = 30
RECONNECT_DELAY = 3
SLOW_BATCH_MS   = 500.0


class BaseMDCollector(ABC):
    """
    Абстрактный коллектор best bid/ask.
    Подкласс задаёт: SCRIPT, EX, MKT, CHUNK_SIZE, SUBSCRIBE_FILE, REDIS_SOCK.
    """
    SCRIPT: str
    EX: str        # bn / bb / ok / gt / bg
    MKT: str       # s / f
    CHUNK_SIZE: int = 300
    SUBSCRIBE_FILE: str
    REDIS_SOCK: str = "/var/run/redis/redis.sock"
    MAX_CONN_SEC: int = 0   # 0 = нет принудительного реконнекта по таймеру

    def __init__(self):
        from .logger import make_logger
        self.log = make_logger(self.SCRIPT)

    @abstractmethod
    def build_url(self, syms: list[str]) -> str:
        """Формирует WS URL для списка символов."""

    @abstractmethod
    def parse(self, raw: bytes) -> tuple[str, str, str] | None:
        """
        Парсит сырое сообщение.
        Возвращает (sym, bid, ask) или None если пропустить.
        """

    def subscribe_message(self, syms: list[str]):
        """Переопределить в подклассе если нужно subscribe-сообщение после connect. По умолчанию None (Binance — streams в URL)."""
        return None

    def redis_key(self, sym: str) -> bytes:
        return f"md:{self.EX}:{self.MKT}:{sym}".encode()

    def load_symbols(self) -> list[str]:
        from pathlib import Path
        return [s for s in Path(self.SUBSCRIBE_FILE).read_text().strip().splitlines() if s]

    @staticmethod
    def chunk(lst, n):
        return [lst[i:i+n] for i in range(0, len(lst), n)]

    async def collect_chunk(
        self,
        r: aioredis.Redis,
        syms: list[str],
        chunk_id: int,
        hw,
        counters: dict,
    ) -> None:
        url = self.build_url(syms)
        t_disconnect = None
        attempt = 0

        while True:
            batch: list = []
            last_flush = time.monotonic()
            oldest_recv_ms = 0.0

            try:
                self.log("INFO", "connecting", chunk_id=chunk_id, chunk_symbols=len(syms))
                async with websockets.connect(
                    url,
                    ping_interval=20,
                    ping_timeout=20,
                    close_timeout=5,
                    max_queue=4096,
                ) as ws:
                    t_connected = time.monotonic()
                    attempt = 0
                    if t_disconnect is not None:
                        self.log("INFO", "reconnected",
                                 chunk_id=chunk_id,
                                 downtime_sec=round(time.monotonic() - t_disconnect, 1))
                        t_disconnect = None
                    self.log("INFO", "connected", chunk_id=chunk_id, chunk_symbols=len(syms))
                    first_msg_logged = False

                    async for raw in ws:
                        now = time.monotonic()

                        # Принудительный реконнект по таймеру (напр. Binance 24h)
                        if self.MAX_CONN_SEC and (now - t_connected) >= self.MAX_CONN_SEC:
                            self.log("INFO", "planned_reconnect",
                                     chunk_id=chunk_id, reason="max_conn_sec")
                            break

                        try:
                            result = self.parse(raw)
                            if result:
                                sym, bid, ask = result
                                recv_ms = time.time() * 1000
                                if not batch:
                                    oldest_recv_ms = recv_ms
                                ts_ms = str(int(recv_ms)).encode()
                                batch.append((
                                    self.redis_key(sym),
                                    {b"b": bid.encode() if isinstance(bid, str) else bid,
                                     b"a": ask.encode() if isinstance(ask, str) else ask,
                                     b"t": ts_ms},
                                ))
                                counters["msgs"] += 1
                                counters["symbols_active"].add(sym)

                                if not first_msg_logged:
                                    self.log("INFO", "first_message",
                                             chunk_id=chunk_id,
                                             ms_since_connected=round((now - t_connected)*1000, 1),
                                             first_sym=sym)
                                    first_msg_logged = True
                        except Exception:
                            pass

                        # slow_batch алерт
                        if batch and oldest_recv_ms:
                            age_ms = time.time()*1000 - oldest_recv_ms
                            if age_ms > SLOW_BATCH_MS:
                                self.log("WARN", "slow_batch",
                                         chunk_id=chunk_id,
                                         batch_age_ms=round(age_ms, 1),
                                         batch_size=len(batch))

                        if batch and (now - last_flush >= BATCH_TIMEOUT or len(batch) >= MAX_BATCH):
                            _bs = len(batch)
                            await flush_pipeline(r, batch, counters, self.log, oldest_recv_ms, hw)
                            batch.clear()
                            oldest_recv_ms = 0.0
                            last_flush = now
                            if _bs > 100:
                                await asyncio.sleep(0)

            except (ConnectionClosed, OSError, Exception) as e:
                self.log("WARN", "disconnected",
                         chunk_id=chunk_id, reason=str(e)[:120])
                t_disconnect = time.monotonic()
                if batch:
                    try:
                        await flush_pipeline(r, batch, counters, self.log, oldest_recv_ms, hw)
                    except Exception as fe:
                        self.log("WARN", "flush_on_disconnect_failed",
                                 chunk_id=chunk_id, batch_size=len(batch), reason=str(fe)[:80])

            attempt += 1
            delay = min(RECONNECT_DELAY * (2 ** min(attempt - 1, 5)), 300)
            self.log("INFO", "reconnecting",
                     chunk_id=chunk_id, delay_sec=round(delay, 1), attempt=attempt)
            await asyncio.sleep(delay)

    async def stats_loop(self, counters: dict) -> None:
        prev_msgs    = 0
        prev_flushes = 0
        prev_lat     = 0.0
        while True:
            await asyncio.sleep(STATS_INTERVAL)
            msgs        = counters["msgs"]
            rate        = (msgs - prev_msgs) / STATS_INTERVAL
            int_flushes = counters["flushes"] - prev_flushes
            avg_lat     = ((counters["lat_total"] - prev_lat) / int_flushes
                           if int_flushes > 0 else 0.0)
            tc = counters["tick_age_count"]
            tick_avg = round(counters["tick_age_sum"] / tc, 1) if tc else 0.0
            tick_max = round(counters["tick_age_max"], 1)
            sym_active = len(counters["symbols_active"])

            self.log("INFO", "stats",
                     msgs_total=msgs,
                     msgs_per_sec=round(rate, 1),
                     flushes_total=counters["flushes"],
                     avg_pipeline_ms=round(avg_lat, 3),
                     tick_age_avg_ms=tick_avg,
                     tick_age_max_ms=tick_max,
                     symbols_active=sym_active)

            prev_msgs    = msgs
            prev_flushes = counters["flushes"]
            prev_lat     = counters["lat_total"]
            counters["symbols_active"] = set()   # сбрасываем за интервал

    async def run(self) -> None:
        self.log("INFO", "startup")
        syms = self.load_symbols()
        self.log("INFO", "symbols_loaded",
                 count=len(syms), file=self.SUBSCRIBE_FILE)

        r = aioredis.Redis(
            unix_socket_path=self.REDIS_SOCK,
            decode_responses=False,
            max_connections=20,
        )
        self.log("INFO", "redis_connected", socket=self.REDIS_SOCK)

        from collectors.hist_writer import HistWriter
        hw = HistWriter(self.EX, self.MKT, syms)

        counters = make_counters()
        chunks = self.chunk(syms, self.CHUNK_SIZE)
        self.log("INFO", "subscribing",
                 total_symbols=len(syms), connections=len(chunks))

        tasks = [
            asyncio.create_task(
                self.collect_chunk(r, ch, idx, hw, counters)
            )
            for idx, ch in enumerate(chunks)
        ]
        tasks.append(asyncio.create_task(self.stats_loop(counters)))
        await asyncio.gather(*tasks)
```

- [ ] **Шаг 2: Коммит**
```bash
git add collectors/core/base_md.py
git commit -m "feat: add BaseMDCollector abstract class"
```

---

## Task 4: core/base_ob.py — базовый OB-коллектор

**Files:**
- Create: `collectors/core/base_ob.py`

Аналогично `base_md.py`, но `parse()` возвращает `(sym, bids, asks)` где bids/asks — list of [price, qty].
Строит маппинг для 10 уровней через `build_mapping()`.

- [ ] **Шаг 1: Написать `collectors/core/base_ob.py`**

```python
#!/usr/bin/env python3
import asyncio
import random
import time
from abc import ABC, abstractmethod

import redis.asyncio as aioredis
import websockets
from websockets.exceptions import ConnectionClosed

from .redis_writer import flush_pipeline, make_counters

LEVELS        = 10
BATCH_TIMEOUT = 0.200 + random.uniform(0, 0.100)
MAX_BATCH     = 150
STATS_INTERVAL  = 30
RECONNECT_DELAY = 3


class BaseOBCollector(ABC):
    SCRIPT: str
    EX: str
    MKT: str
    CHUNK_SIZE: int = 200
    SUBSCRIBE_FILE: str
    REDIS_SOCK: str = "/var/run/redis/redis.sock"
    MAX_CONN_SEC: int = 0

    def __init__(self):
        from .logger import make_logger
        self.log = make_logger(self.SCRIPT)

    @abstractmethod
    def build_url(self, syms: list[str]) -> str: ...

    @abstractmethod
    def parse(self, raw: bytes) -> tuple[str, list, list] | None:
        """Возвращает (sym, bids[[price,qty],...], asks[[price,qty],...]) или None."""

    def redis_key(self, sym: str) -> bytes:
        return f"ob:{self.EX}:{self.MKT}:{sym}".encode()

    def load_symbols(self) -> list[str]:
        from pathlib import Path
        return [s for s in Path(self.SUBSCRIBE_FILE).read_text().strip().splitlines() if s]

    @staticmethod
    def chunk(lst, n):
        return [lst[i:i+n] for i in range(0, len(lst), n)]

    @staticmethod
    def build_mapping(bids: list, asks: list, ts_ms: bytes) -> dict:
        m = {b"t": ts_ms}
        for i, row in enumerate(bids[:LEVELS], 1):
            m[f"b{i}".encode()]  = str(row[0]).encode()
            m[f"bq{i}".encode()] = str(row[1]).encode()
        for i, row in enumerate(asks[:LEVELS], 1):
            m[f"a{i}".encode()]  = str(row[0]).encode()
            m[f"aq{i}".encode()] = str(row[1]).encode()
        return m

    async def collect_chunk(self, r, syms, chunk_id, hw, counters) -> None:
        url = self.build_url(syms)
        t_disconnect = None
        attempt = 0

        while True:
            batch: list = []
            last_flush = time.monotonic()
            try:
                self.log("INFO", "connecting", chunk_id=chunk_id, chunk_symbols=len(syms))
                async with websockets.connect(url,
                        ping_interval=20, ping_timeout=20,
                        close_timeout=5, max_queue=4096) as ws:
                    t_connected = time.monotonic()
                    attempt = 0
                    if t_disconnect is not None:
                        self.log("INFO", "reconnected", chunk_id=chunk_id,
                                 downtime_sec=round(time.monotonic()-t_disconnect, 1))
                        t_disconnect = None
                    self.log("INFO", "connected", chunk_id=chunk_id, chunk_symbols=len(syms))
                    first_msg_logged = False

                    # Некоторые биржи (Bybit, OKX, Gate, Bitget) требуют subscribe-сообщение
                    subscribe_msg = self.subscribe_message(syms)
                    if subscribe_msg:
                        import orjson
                        await ws.send(orjson.dumps(subscribe_msg))

                    async for raw in ws:
                        now = time.monotonic()
                        if self.MAX_CONN_SEC and (now - t_connected) >= self.MAX_CONN_SEC:
                            self.log("INFO", "planned_reconnect", chunk_id=chunk_id)
                            break
                        try:
                            result = self.parse(raw)
                            if result:
                                sym, bids, asks = result
                                ts_ms = str(int(time.time()*1000)).encode()
                                n_bid = len(bids[:LEVELS])
                                n_ask = len(asks[:LEVELS])
                                counters["levels_total"] += n_bid + n_ask
                                counters["levels_count"] += 2
                                if n_bid < LEVELS or n_ask < LEVELS:
                                    counters["shallow"] += 1
                                batch.append((self.redis_key(sym), self.build_mapping(bids, asks, ts_ms)))
                                counters["msgs"] += 1
                                counters["symbols_active"].add(sym)
                                if not first_msg_logged:
                                    self.log("INFO", "first_message", chunk_id=chunk_id,
                                             ms_since_connected=round((now-t_connected)*1000, 1),
                                             first_sym=sym)
                                    first_msg_logged = True
                        except Exception:
                            pass
                        if batch and (now - last_flush >= BATCH_TIMEOUT or len(batch) >= MAX_BATCH):
                            _bs = len(batch)
                            await flush_pipeline(r, batch, counters, self.log, 0.0, hw)
                            batch.clear()
                            last_flush = now
                            if _bs > 100:
                                await asyncio.sleep(0)

            except (ConnectionClosed, OSError, Exception) as e:
                self.log("WARN", "disconnected", chunk_id=chunk_id, reason=str(e)[:120])
                t_disconnect = time.monotonic()
                if batch:
                    try:
                        await flush_pipeline(r, batch, counters, self.log, 0.0, hw)
                    except Exception as fe:
                        self.log("WARN", "flush_on_disconnect_failed",
                                 chunk_id=chunk_id, batch_size=len(batch), reason=str(fe)[:80])

            attempt += 1
            delay = min(RECONNECT_DELAY * (2 ** min(attempt-1, 5)), 300)
            self.log("INFO", "reconnecting", chunk_id=chunk_id,
                     delay_sec=round(delay, 1), attempt=attempt)
            await asyncio.sleep(delay)

    def subscribe_message(self, syms: list[str]):
        """Переопределить в подклассе если нужно subscribe-сообщение. По умолчанию None."""
        return None

    async def stats_loop(self, counters: dict) -> None:
        prev_msgs = prev_flushes = 0
        prev_lat = 0.0
        while True:
            await asyncio.sleep(STATS_INTERVAL)
            msgs = counters["msgs"]
            rate = (msgs - prev_msgs) / STATS_INTERVAL
            int_f = counters["flushes"] - prev_flushes
            avg_lat = (counters["lat_total"] - prev_lat) / int_f if int_f else 0.0
            avg_lvl = (counters["levels_total"] / counters["levels_count"]
                       if counters["levels_count"] else 0.0)
            self.log("INFO", "stats",
                     msgs_total=msgs,
                     msgs_per_sec=round(rate, 1),
                     flushes_total=counters["flushes"],
                     avg_pipeline_ms=round(avg_lat, 3),
                     avg_levels=round(avg_lvl, 2),
                     shallow_warnings=counters["shallow"],
                     symbols_active=len(counters["symbols_active"]))
            prev_msgs = msgs
            prev_flushes = counters["flushes"]
            prev_lat = counters["lat_total"]
            counters["symbols_active"] = set()

    async def run(self) -> None:
        self.log("INFO", "startup")
        syms = self.load_symbols()
        self.log("INFO", "symbols_loaded", count=len(syms), file=self.SUBSCRIBE_FILE)
        r = aioredis.Redis(unix_socket_path=self.REDIS_SOCK,
                           decode_responses=False, max_connections=20)
        self.log("INFO", "redis_connected", socket=self.REDIS_SOCK)

        from collectors.ob_hist_writer import OBHistWriter
        hw = OBHistWriter(self.EX, self.MKT, syms)

        counters = make_counters()
        counters["levels_total"] = 0
        counters["levels_count"] = 0
        counters["shallow"]      = 0

        chunks = self.chunk(syms, self.CHUNK_SIZE)
        self.log("INFO", "subscribing", total_symbols=len(syms), connections=len(chunks))

        tasks = [asyncio.create_task(self.collect_chunk(r, ch, idx, hw, counters))
                 for idx, ch in enumerate(chunks)]
        tasks.append(asyncio.create_task(self.stats_loop(counters)))
        await asyncio.gather(*tasks)
```

- [ ] **Шаг 2: Коммит**
```bash
git add collectors/core/base_ob.py
git commit -m "feat: add BaseOBCollector abstract class"
```

---

## Task 5: core/base_fr.py — базовый FR-коллектор

**Files:**
- Create: `collectors/core/base_fr.py`

FR-коллекторы не делают chunking (обычно одно соединение на все символы).
`parse()` возвращает список `[(sym, rate, next_rate, funding_time_ms)]` из одного сообщения.

- [ ] **Шаг 1: Написать `collectors/core/base_fr.py`**

```python
#!/usr/bin/env python3
import asyncio
import random
import time
from abc import ABC, abstractmethod

import orjson
import redis.asyncio as aioredis
import websockets
from websockets.exceptions import ConnectionClosed

from .redis_writer import flush_pipeline, make_counters

BATCH_TIMEOUT   = 0.200 + random.uniform(0, 0.100)
MAX_BATCH       = 300
STATS_INTERVAL  = 30
RECONNECT_DELAY = 3


class BaseFRCollector(ABC):
    SCRIPT: str
    EX: str          # только futures
    SUBSCRIBE_FILE: str
    REDIS_SOCK: str = "/var/run/redis/redis.sock"
    MAX_CONN_SEC: int = 0

    def __init__(self):
        from .logger import make_logger
        self.log = make_logger(self.SCRIPT)

    @abstractmethod
    def build_url(self, syms: list[str]) -> str: ...

    @abstractmethod
    def parse(self, raw: bytes, sym_set: set) -> list[tuple]:
        """
        Возвращает список (sym, rate_bytes, next_rate_bytes, ft_bytes, ts_bytes).
        rate = funding rate строка, next_rate = predicted (или b""),
        ft = next funding time ms (или b""), ts = receive ts ms.
        """

    def subscribe_message(self, syms: list[str]):
        return None

    def redis_key(self, sym: str) -> bytes:
        return f"fr:{self.EX}:{sym}".encode()

    def load_symbols(self) -> list[str]:
        from pathlib import Path
        return [s for s in Path(self.SUBSCRIBE_FILE).read_text().strip().splitlines() if s]

    async def collect(self, r, sym_set, hw, counters) -> None:
        url = self.build_url(list(sym_set))
        t_disconnect = None
        attempt = 0

        while True:
            batch: list = []
            last_flush = time.monotonic()
            try:
                self.log("INFO", "connecting")
                async with websockets.connect(url,
                        ping_interval=20, ping_timeout=20,
                        close_timeout=5, max_queue=4096) as ws:
                    t_connected = time.monotonic()
                    attempt = 0
                    if t_disconnect is not None:
                        self.log("INFO", "reconnected",
                                 downtime_sec=round(time.monotonic()-t_disconnect, 1))
                        t_disconnect = None
                    self.log("INFO", "connected")
                    first_msg_logged = False

                    sub = self.subscribe_message(list(sym_set))
                    if sub:
                        await ws.send(orjson.dumps(sub))

                    async for raw in ws:
                        now = time.monotonic()
                        if self.MAX_CONN_SEC and (now-t_connected) >= self.MAX_CONN_SEC:
                            self.log("INFO", "planned_reconnect", reason="max_conn_sec")
                            break
                        try:
                            items = self.parse(raw, sym_set)
                            if items:
                                batch.extend(items)
                                counters["msgs"] += len(items)
                                if not first_msg_logged:
                                    self.log("INFO", "first_message",
                                             ms_since_connected=round((now-t_connected)*1000, 1))
                                    first_msg_logged = True
                        except Exception:
                            pass

                        if batch and (now - last_flush >= BATCH_TIMEOUT or len(batch) >= MAX_BATCH):
                            # FR batch flush: особый формат mapping
                            _bs = len(batch)
                            await self._flush_fr(r, batch, hw, counters)
                            batch.clear()
                            last_flush = now
                            if _bs > 100:
                                await asyncio.sleep(0)

            except (ConnectionClosed, OSError, Exception) as e:
                self.log("WARN", "disconnected", reason=str(e)[:120])
                t_disconnect = time.monotonic()
                if batch:
                    try:
                        await self._flush_fr(r, batch, hw, counters)
                    except Exception as fe:
                        self.log("WARN", "flush_on_disconnect_failed",
                                 batch_size=len(batch), reason=str(fe)[:80])

            attempt += 1
            delay = min(RECONNECT_DELAY * (2 ** min(attempt-1, 5)), 300)
            self.log("INFO", "reconnecting", delay_sec=round(delay, 1), attempt=attempt)
            await asyncio.sleep(delay)

    async def _flush_fr(self, r, batch, hw, counters) -> None:
        t0 = time.monotonic()
        now_sec = time.time()
        async with r.pipeline(transaction=False) as pipe:
            for sym, r_b, nr_b, ft_b, ts_b in batch:
                pipe.hset(self.redis_key(sym),
                          mapping={b"r": r_b, b"nr": nr_b, b"ft": ft_b, b"ts": ts_b})
            if hw is not None:
                hw.add_to_pipe(pipe, batch, now_sec)
                hw.ensure_config(pipe, now_sec)
            await pipe.execute()
        lat_ms = (time.monotonic() - t0) * 1000
        counters["flushes"]   += 1
        counters["lat_total"] += lat_ms
        if lat_ms > 50.0:
            self.log("WARN", "slow_flush", pipeline_ms=round(lat_ms, 2))

    async def stats_loop(self, counters) -> None:
        prev_msgs = prev_flushes = 0
        prev_lat = 0.0
        while True:
            await asyncio.sleep(STATS_INTERVAL)
            msgs = counters["msgs"]
            rate = (msgs - prev_msgs) / STATS_INTERVAL
            int_f = counters["flushes"] - prev_flushes
            avg_lat = (counters["lat_total"] - prev_lat) / int_f if int_f else 0.0
            self.log("INFO", "stats",
                     msgs_total=msgs,
                     msgs_per_sec=round(rate, 1),
                     flushes_total=counters["flushes"],
                     avg_pipeline_ms=round(avg_lat, 3))
            prev_msgs = msgs
            prev_flushes = counters["flushes"]
            prev_lat = counters["lat_total"]

    async def run(self) -> None:
        self.log("INFO", "startup")
        syms = self.load_symbols()
        self.log("INFO", "symbols_loaded", count=len(syms), file=self.SUBSCRIBE_FILE)
        r = aioredis.Redis(unix_socket_path=self.REDIS_SOCK,
                           decode_responses=False, max_connections=20)
        self.log("INFO", "redis_connected", socket=self.REDIS_SOCK)

        from collectors.fr_hist_writer import FrHistWriter
        hw = FrHistWriter(self.EX, syms)
        counters = make_counters()

        await asyncio.gather(
            self.collect(r, set(syms), hw, counters),
            self.stats_loop(counters),
        )
```

- [ ] **Шаг 2: Коммит**
```bash
git add collectors/core/base_fr.py
git commit -m "feat: add BaseFRCollector abstract class"
```

---

## Task 6: Скопировать hist_writers из 3.0

**Files:**
- Create: `collectors/hist_writer.py`
- Create: `collectors/ob_hist_writer.py`
- Create: `collectors/fr_hist_writer.py`
- Create: `collectors/source_ids.json`
- Create: `collectors/symbol_ids.json`

- [ ] **Шаг 1: Скопировать файлы из bali3.0_data**
```bash
cp bali3.0_data/collectors/hist_writer.py collectors/hist_writer.py
cp bali3.0_data/collectors/ob_hist_writer.py collectors/ob_hist_writer.py
cp bali3.0_data/collectors/fr_hist_writer.py collectors/fr_hist_writer.py
cp bali3.0_data/collectors/source_ids.json collectors/source_ids.json
cp bali3.0_data/collectors/symbol_ids.json collectors/symbol_ids.json
```

- [ ] **Шаг 2: Коммит**
```bash
git add collectors/hist_writer.py collectors/ob_hist_writer.py \
        collectors/fr_hist_writer.py collectors/source_ids.json \
        collectors/symbol_ids.json
git commit -m "feat: port hist_writers from 3.0"
```

---

## Task 7: MD коллекторы — Binance (spot + futures)

**Files:**
- Create: `collectors/md/__init__.py`
- Create: `collectors/md/binance_spot.py`
- Create: `collectors/md/binance_futures.py`

**Binance Spot:** WS `stream.binance.com:9443`, канал `@bookTicker`, streams в URL, CHUNK_SIZE=300.
**Binance Futures:** WS `fstream.binance.com`, канал `@bookTicker`, streams в URL, CHUNK_SIZE=300.

- [ ] **Шаг 1: `collectors/md/__init__.py`** (пустой)

- [ ] **Шаг 2: `collectors/md/binance_spot.py`**
```python
from pathlib import Path
import orjson
from collectors.core.base_md import BaseMDCollector

class BinanceSpotMD(BaseMDCollector):
    SCRIPT = "md_binance_spot"
    EX, MKT = "bn", "s"
    CHUNK_SIZE = 300
    SUBSCRIBE_FILE = str(Path(__file__).parent.parent.parent /
                         "dictionaries/subscribe/binance/binance_spot.txt")

    def build_url(self, syms):
        base = "wss://stream.binance.com:9443/stream?streams="
        return base + "/".join(f"{s.lower()}@bookTicker" for s in syms)

    def parse(self, raw):
        obj  = orjson.loads(raw)
        data = obj.get("data") or obj
        sym, bid, ask = data.get("s"), data.get("b"), data.get("a")
        return (sym, bid, ask) if sym and bid and ask else None

if __name__ == "__main__":
    import asyncio
    asyncio.run(BinanceSpotMD().run())
```

- [ ] **Шаг 3: `collectors/md/binance_futures.py`**
```python
from pathlib import Path
import orjson
from collectors.core.base_md import BaseMDCollector

class BinanceFuturesMD(BaseMDCollector):
    SCRIPT = "md_binance_futures"
    EX, MKT = "bn", "f"
    CHUNK_SIZE = 300
    MAX_CONN_SEC = 23 * 3600
    SUBSCRIBE_FILE = str(Path(__file__).parent.parent.parent /
                         "dictionaries/subscribe/binance/binance_futures.txt")

    def build_url(self, syms):
        base = "wss://fstream.binance.com/stream?streams="
        return base + "/".join(f"{s.lower()}@bookTicker" for s in syms)

    def parse(self, raw):
        obj  = orjson.loads(raw)
        data = obj.get("data") or obj
        sym, bid, ask = data.get("s"), data.get("b"), data.get("a")
        return (sym, bid, ask) if sym and bid and ask else None

if __name__ == "__main__":
    import asyncio
    asyncio.run(BinanceFuturesMD().run())
```

- [ ] **Шаг 4: Коммит**
```bash
git add collectors/md/
git commit -m "feat: add MD collectors for Binance spot+futures"
```

---

## Task 8: MD коллекторы — Bybit (spot + futures)

**Files:**
- Create: `collectors/md/bybit_spot.py`
- Create: `collectors/md/bybit_futures.py`

**Bybit:** WS `wss://stream.bybit.com/v5/public/spot` (spot) и `.../linear` (futures).
Подписка через JSON-сообщение `{"op":"subscribe","args":["orderbook.1.BTCUSDT",...]}`.
Каждый `args` массив — до 10 символов. Нет URL-чанкинга, всё в одном соединении с subscribe.
Ответ содержит `topic`, `data.b[0]` (best bid), `data.a[0]` (best ask).

- [ ] **Шаг 1: `collectors/md/bybit_spot.py`**
```python
from pathlib import Path
import orjson
from collectors.core.base_md import BaseMDCollector

BYBIT_SPOT_WS = "wss://stream.bybit.com/v5/public/spot"

class BybitSpotMD(BaseMDCollector):
    SCRIPT = "md_bybit_spot"
    EX, MKT = "bb", "s"
    CHUNK_SIZE = 10   # Bybit принимает до 10 тем за раз
    SUBSCRIBE_FILE = str(Path(__file__).parent.parent.parent /
                         "dictionaries/subscribe/bybit/bybit_spot.txt")

    def build_url(self, syms):
        return BYBIT_SPOT_WS   # URL одинаковый для всех чанков

    def subscribe_message(self, syms):
        return {"op": "subscribe", "args": [f"orderbook.1.{s}" for s in syms]}

    def parse(self, raw):
        obj = orjson.loads(raw)
        if obj.get("op") == "subscribe":
            return None
        data = obj.get("data", {})
        topic = obj.get("topic", "")
        if not topic.startswith("orderbook.1."):
            return None
        sym = topic.split(".")[-1]
        bids = data.get("b", [])
        asks = data.get("a", [])
        if not bids or not asks:
            return None
        return (sym, str(bids[0][0]), str(asks[0][0]))

if __name__ == "__main__":
    import asyncio
    asyncio.run(BybitSpotMD().run())
```

- [ ] **Шаг 2: `collectors/md/bybit_futures.py`**
```python
from pathlib import Path
import orjson
from collectors.core.base_md import BaseMDCollector

class BybitFuturesMD(BaseMDCollector):
    SCRIPT = "md_bybit_futures"
    EX, MKT = "bb", "f"
    CHUNK_SIZE = 10
    SUBSCRIBE_FILE = str(Path(__file__).parent.parent.parent /
                         "dictionaries/subscribe/bybit/bybit_futures.txt")

    def build_url(self, syms):
        return "wss://stream.bybit.com/v5/public/linear"

    def subscribe_message(self, syms):
        return {"op": "subscribe", "args": [f"orderbook.1.{s}" for s in syms]}

    def parse(self, raw):
        obj = orjson.loads(raw)
        if obj.get("op") == "subscribe":
            return None
        topic = obj.get("topic", "")
        if not topic.startswith("orderbook.1."):
            return None
        sym = topic.split(".")[-1]
        data = obj.get("data", {})
        bids, asks = data.get("b", []), data.get("a", [])
        if not bids or not asks:
            return None
        return (sym, str(bids[0][0]), str(asks[0][0]))

if __name__ == "__main__":
    import asyncio
    asyncio.run(BybitFuturesMD().run())
```

- [ ] **Шаг 3: Коммит**
```bash
git add collectors/md/bybit_spot.py collectors/md/bybit_futures.py
git commit -m "feat: add MD collectors for Bybit spot+futures"
```

---

## Task 9: MD коллекторы — OKX (spot + futures)

**Files:**
- Create: `collectors/md/okx_spot.py`
- Create: `collectors/md/okx_futures.py`

**OKX:** WS `wss://ws.okx.com:8443/ws/v5/public`. Subscribe: `{"op":"subscribe","args":[{"channel":"bbo-tbt","instId":"BTC-USDT"},...]}`.
Ответ: `data[0].bidPx`, `data[0].askPx`. instType: `SPOT` / `SWAP`.
OKX symbol format: `BTC-USDT` (spot), `BTC-USDT-SWAP` (futures).

- [ ] **Шаг 1: `collectors/md/okx_spot.py`**
```python
from pathlib import Path
import orjson
from collectors.core.base_md import BaseMDCollector

OKX_WS = "wss://ws.okx.com:8443/ws/v5/public"

class OKXSpotMD(BaseMDCollector):
    SCRIPT = "md_okx_spot"
    EX, MKT = "ok", "s"
    CHUNK_SIZE = 30   # OKX рекомендует <=30 подписок за раз
    SUBSCRIBE_FILE = str(Path(__file__).parent.parent.parent /
                         "dictionaries/subscribe/okx/okx_spot.txt")

    def build_url(self, syms):
        return OKX_WS

    def subscribe_message(self, syms):
        # syms в формате из subscribe-файла (нативный OKX формат BTC-USDT)
        return {"op": "subscribe",
                "args": [{"channel": "bbo-tbt", "instId": s} for s in syms]}

    def parse(self, raw):
        obj = orjson.loads(raw)
        if obj.get("event"):
            return None
        data_list = obj.get("data", [])
        if not data_list:
            return None
        d = data_list[0]
        # instId из arg или из data
        arg = obj.get("arg", {})
        inst_id = arg.get("instId", "")
        # Конвертация BTC-USDT → BTCUSDT для Redis-ключа
        sym = inst_id.replace("-", "")
        bid = d.get("bidPx", "")
        ask = d.get("askPx", "")
        return (sym, bid, ask) if sym and bid and ask else None

if __name__ == "__main__":
    import asyncio
    asyncio.run(OKXSpotMD().run())
```

- [ ] **Шаг 2: `collectors/md/okx_futures.py`** — аналогично, `MKT="f"`, `instType="SWAP"`, нативный файл `okx_futures.txt` (формат `BTC-USDT-SWAP`), конвертация `BTC-USDT-SWAP → BTCUSDT`.

- [ ] **Шаг 3: Коммит**
```bash
git add collectors/md/okx_spot.py collectors/md/okx_futures.py
git commit -m "feat: add MD collectors for OKX spot+futures"
```

---

## Task 10: MD коллекторы — Gate + Bitget (spot + futures каждый)

**Files:**
- Create: `collectors/md/gate_spot.py`
- Create: `collectors/md/gate_futures.py`
- Create: `collectors/md/bitget_spot.py`
- Create: `collectors/md/bitget_futures.py`

**Gate Spot:** WS `wss://api.gateio.ws/ws/v4/`. Channel `spot.book_ticker`.
Subscribe: `{"time":..., "channel":"spot.book_ticker", "event":"subscribe", "payload":["BTC_USDT",...]}`.
Ответ: `result.s` (sym), `result.b` (bid), `result.a` (ask).

**Gate Futures:** WS `wss://fx-ws.gateio.ws/v4/ws/usdt`. Channel `futures.book_ticker`.
Subscribe: `{"time":..., "channel":"futures.book_ticker", "event":"subscribe", "payload":["BTC_USDT",...]}`.
Ответ: `result.s` (sym), `result.b` (bid), `result.a` (ask). Символы в нативном формате из `gate_futures_native.txt`.

**Bitget Spot:** WS `wss://ws.bitget.com/v2/ws/public`. Channel `books1`.
Subscribe: `{"op":"subscribe","args":[{"instType":"SPOT","channel":"books1","instId":"BTCUSDT"},...]}`.
Ответ: `data[0].bids[0]`, `data[0].asks[0]`.

**Bitget Futures:** Аналогично, `instType="USDT-FUTURES"`.

- [ ] **Шаг 1:** Написать все 4 файла по описанным выше паттернам.

- [ ] **Шаг 2: Коммит**
```bash
git add collectors/md/gate_spot.py collectors/md/gate_futures.py \
        collectors/md/bitget_spot.py collectors/md/bitget_futures.py
git commit -m "feat: add MD collectors for Gate+Bitget spot+futures"
```

---

## Task 11: OB коллекторы — Binance (spot + futures)

**Files:**
- Create: `collectors/ob/__init__.py`
- Create: `collectors/ob/binance_spot.py`
- Create: `collectors/ob/binance_futures.py`

**Binance OB Spot:** канал `@depth10@100ms`, CHUNK_SIZE=200, streams в URL.
**Binance OB Futures:** канал `@depth10@100ms` на `fstream.binance.com`, CHUNK_SIZE=200.

- [ ] **Шаг 1: `collectors/ob/binance_spot.py`**
```python
from pathlib import Path
import orjson
from collectors.core.base_ob import BaseOBCollector

class BinanceSpotOB(BaseOBCollector):
    SCRIPT = "ob_binance_spot"
    EX, MKT = "bn", "s"
    CHUNK_SIZE = 200
    SUBSCRIBE_FILE = str(Path(__file__).parent.parent.parent /
                         "dictionaries/subscribe/binance/binance_spot.txt")

    def build_url(self, syms):
        base = "wss://stream.binance.com:9443/stream?streams="
        return base + "/".join(f"{s.lower()}@depth10@100ms" for s in syms)

    def parse(self, raw):
        obj  = orjson.loads(raw)
        data = obj.get("data") or obj
        bids = data.get("bids", [])
        asks = data.get("asks", [])
        stream = obj.get("stream", "")
        sym = stream.split("@")[0].upper() if stream else data.get("s", "")
        return (sym, bids, asks) if sym and (bids or asks) else None

if __name__ == "__main__":
    import asyncio
    asyncio.run(BinanceSpotOB().run())
```

- [ ] **Шаг 2:** `binance_futures.py` — аналогично, `fstream.binance.com`, `MKT="f"`, `MAX_CONN_SEC=23*3600`.

- [ ] **Шаг 3: Коммит**
```bash
git add collectors/ob/
git commit -m "feat: add OB collectors for Binance spot+futures"
```

---

## Task 12: OB коллекторы — Bybit, OKX, Gate, Bitget

**Files:** `collectors/ob/bybit_spot.py`, `bybit_futures.py`, `okx_spot.py`, `okx_futures.py`, `gate_spot.py`, `gate_futures.py`, `bitget_spot.py`, `bitget_futures.py`

**Паттерны:**

**Bybit OB:** Subscribe `orderbook.10.{SYM}`, ответ `data.b` (bids), `data.a` (asks) — list of [price, qty].

**OKX OB:** Channel `books` с `depth=10`, ответ `data[0].bids`, `data[0].asks`.

**Gate OB Spot:** Channel `spot.order_book`, payload `[sym, "10", "100ms"]`.
Ответ: `result.bids`, `result.asks`.

**Gate OB Futures:** Channel `futures.order_book`, аналогично.

**Bitget OB:** Channel `books`, depth 10. Subscribe аналогично MD но канал `books`.
Ответ: `data[0].bids`, `data[0].asks`.

- [ ] **Шаг 1:** Написать все 8 файлов по паттернам выше.

- [ ] **Шаг 2: Коммит**
```bash
git add collectors/ob/bybit_spot.py collectors/ob/bybit_futures.py \
        collectors/ob/okx_spot.py collectors/ob/okx_futures.py \
        collectors/ob/gate_spot.py collectors/ob/gate_futures.py \
        collectors/ob/bitget_spot.py collectors/ob/bitget_futures.py
git commit -m "feat: add OB collectors for Bybit/OKX/Gate/Bitget"
```

---

## Task 13: FR коллекторы (5 файлов)

**Files:**
- Create: `collectors/fr/__init__.py`
- Create: `collectors/fr/binance_futures.py`
- Create: `collectors/fr/bybit_futures.py`
- Create: `collectors/fr/okx_futures.py`
- Create: `collectors/fr/gate_futures.py`
- Create: `collectors/fr/bitget_futures.py`

**Binance FR:** WS `fstream.binance.com/ws/!markPrice@arr@1s`. Один поток, все символы.
Поле `r`=rate, `T`=next funding time ms. `MAX_CONN_SEC=23*3600`.

**Bybit FR:** WS `stream.bybit.com/v5/public/linear`. Subscribe `tickers.{SYM}`.
Поле `fundingRate`, `nextFundingTime`.

**OKX FR:** WS `ws.okx.com:8443/ws/v5/public`. Channel `funding-rate`, instId нативный.
Поле `fundingRate`, `nextFundingTime`.

**Gate FR:** REST poll каждые 30с (Gate не имеет WS для FR).
`GET https://fx-api.gateio.ws/api/v4/futures/usdt/tickers` → массив `[{contract, funding_rate, next_funding_time}]`.
**НЕ наследует `BaseFRCollector`** — отдельный standalone-класс с собственным `run()` на основе `aiohttp`.
Использует `aiohttp.ClientSession` с таймаутом 10с. Зависимость: `pip install aiohttp`.

**Bitget FR:** WS `ws.bitget.com/v2/ws/public`. Subscribe `{"channel":"ticker","instId":sym}`.
Поле `fundingRate`, `nextSettleTime`.

- [ ] **Шаг 1: `collectors/fr/binance_futures.py`**
```python
from pathlib import Path
import orjson
from collectors.core.base_fr import BaseFRCollector

class BinanceFuturesFR(BaseFRCollector):
    SCRIPT = "fr_binance_futures"
    EX = "bn"
    MAX_CONN_SEC = 23 * 3600
    SUBSCRIBE_FILE = str(Path(__file__).parent.parent.parent /
                         "dictionaries/subscribe/binance/binance_futures.txt")

    def build_url(self, syms):
        return "wss://fstream.binance.com/ws/!markPrice@arr@1s"

    def parse(self, raw, sym_set):
        data = orjson.loads(raw)
        if not isinstance(data, list):
            return []
        ts_b = str(int(__import__("time").time() * 1000)).encode()
        result = []
        for item in data:
            sym   = item.get("s", "")
            r_val = item.get("r", "")
            if sym not in sym_set or not r_val:
                continue
            ft_raw = item.get("T", 0)
            ft_b   = str(ft_raw).encode() if ft_raw else b""
            result.append((sym, r_val.encode(), b"", ft_b, ts_b))
        return result

if __name__ == "__main__":
    import asyncio
    asyncio.run(BinanceFuturesFR().run())
```

- [ ] **Шаг 2:** Написать Bybit/OKX/Bitget FR по WS-паттерну `BaseFRCollector`.

- [ ] **Шаг 3: `collectors/fr/gate_futures.py`** — STANDALONE (не наследует BaseFRCollector):

```python
#!/usr/bin/env python3
import asyncio
import time
from pathlib import Path

import aiohttp
import orjson
import redis.asyncio as aioredis

from collectors.core.logger import make_logger
from collectors.fr_hist_writer import FrHistWriter

SCRIPT         = "fr_gate_futures"
EX             = "gt"
REDIS_SOCK     = "/var/run/redis/redis.sock"
POLL_INTERVAL  = 30   # секунд
API_URL        = "https://fx-api.gateio.ws/api/v4/futures/usdt/tickers"
SUBSCRIBE_FILE = str(Path(__file__).parent.parent.parent /
                     "dictionaries/subscribe/gate/gate_futures.txt")

log = make_logger(SCRIPT)

def redis_key(sym: str) -> bytes:
    return f"fr:{EX}:{sym}".encode()

def load_symbols() -> set:
    return set(Path(SUBSCRIBE_FILE).read_text().strip().splitlines())

async def poll(r: aioredis.Redis, sym_set: set, hw: FrHistWriter) -> None:
    flushes = 0
    lat_total = 0.0
    async with aiohttp.ClientSession(timeout=aiohttp.ClientTimeout(total=10)) as session:
        while True:
            t_poll = time.monotonic()
            try:
                async with session.get(API_URL) as resp:
                    data = orjson.loads(await resp.read())
                ts_b = str(int(time.time() * 1000)).encode()
                batch = []
                for item in data:
                    sym = item.get("contract", "").replace("_", "")
                    if sym not in sym_set:
                        continue
                    r_val  = str(item.get("funding_rate", "")).encode()
                    nr_val = str(item.get("funding_rate_indicative", "")).encode()
                    ft_raw = item.get("funding_next_apply", 0)
                    ft_b   = str(ft_raw * 1000).encode() if ft_raw else b""
                    batch.append((sym, r_val, nr_val, ft_b, ts_b))

                if batch:
                    t0 = time.monotonic()
                    now_sec = time.time()
                    async with r.pipeline(transaction=False) as pipe:
                        for sym, r_b, nr_b, ft_b, ts_b2 in batch:
                            pipe.hset(redis_key(sym),
                                      mapping={b"r": r_b, b"nr": nr_b, b"ft": ft_b, b"ts": ts_b2})
                        hw.add_to_pipe(pipe, batch, now_sec)
                        hw.ensure_config(pipe, now_sec)
                        await pipe.execute()
                    lat_ms = (time.monotonic() - t0) * 1000
                    flushes += 1
                    lat_total += lat_ms
                    if lat_ms > 50:
                        log("WARN", "slow_flush", pipeline_ms=round(lat_ms, 2))
                    log("INFO", "poll_ok",
                        symbols_updated=len(batch),
                        pipeline_ms=round(lat_ms, 2))

            except Exception as e:
                log("WARN", "poll_error", reason=str(e)[:120])

            elapsed = time.monotonic() - t_poll
            await asyncio.sleep(max(0, POLL_INTERVAL - elapsed))

async def main() -> None:
    log("INFO", "startup")
    syms = load_symbols()
    log("INFO", "symbols_loaded", count=len(syms))
    r = aioredis.Redis(unix_socket_path=REDIS_SOCK, decode_responses=False, max_connections=5)
    log("INFO", "redis_connected", socket=REDIS_SOCK)
    hw = FrHistWriter(EX, list(syms))
    await poll(r, syms, hw)

if __name__ == "__main__":
    asyncio.run(main())
```

- [ ] **Шаг 3: Коммит**
```bash
git add collectors/fr/
git commit -m "feat: add FR collectors for all 5 exchanges"
```

---

## Task 14: staleness_monitor.py

**Files:**
- Create: `collectors/staleness_monitor.py`

Каждые 30с сканирует все `md:*` ключи в Redis, проверяет поле `t` (timestamp ms).
Символы с данными старше `STALE_THRESHOLD=300s` логирует как `stale`.

- [ ] **Шаг 1:** Скопировать и адаптировать из 3.0:
```bash
cp bali3.0_data/collectors/staleness_monitor.py collectors/staleness_monitor.py
```

- [ ] **Шаг 2: Коммит**
```bash
git add collectors/staleness_monitor.py
git commit -m "feat: port staleness_monitor"
```

---

## Task 15: spread_monitor.py

**Files:**
- Create: `collectors/spread_monitor.py`

Логика идентична 3.0, но улучшены логи cycle_stats.

- [ ] **Шаг 1:** Скопировать и адаптировать из 3.0:
```bash
cp bali3.0_data/collectors/spread_monitor.py collectors/spread_monitor.py
```

- [ ] **Шаг 2:** Добавить в stats-лог поля:
```python
self.log("INFO", "cycle_stats",
    cycle_ms=round((time.monotonic() - t_cycle_start) * 1000, 2),
    redis_read_ms=round(redis_read_ms, 2),
    pairs_checked=pairs_checked,
    signals_fired=signals_fired,
    cooldowns_active=len(self.cooldowns),
    stale_skipped=stale_skipped)
```

- [ ] **Шаг 3: Коммит**
```bash
git add collectors/spread_monitor.py
git commit -m "feat: port spread_monitor with cycle_stats logging"
```

---

## Task 16: Runners + run_all.py

**Files:**
- Create: `runner_md.py`
- Create: `runner_ob.py`
- Create: `runner_fr.py`
- Create: `runner_monitors.py`
- Create: `run_all.py`

**runner_md.py** — импортирует все 10 MD-классов, запускает как asyncio.gather():
```python
from collectors.md.binance_spot import BinanceSpotMD
# ... все 10
async def main():
    await asyncio.gather(
        BinanceSpotMD().run(),
        BinanceFuturesMD().run(),
        # ...
    )
asyncio.run(main())
```

**runner_ob.py** — аналогично для 10 OB-классов.
**runner_fr.py** — аналогично для 5 FR-классов.
**runner_monitors.py** — запускает spread_monitor и staleness_monitor.

**run_all.py** — аналог 3.0: subprocess + CPU affinity + rotating logs + TUI dashboard.
Задержки старта: OB → +10s MD → +5s FR → +3s monitors.

- [ ] **Шаг 1:** Написать все runner-файлы.

- [ ] **Шаг 2:** Написать `run_all.py` — адаптировать из 3.0, заменить пути скриптов.

- [ ] **Шаг 3: Коммит**
```bash
git add runner_md.py runner_ob.py runner_fr.py runner_monitors.py run_all.py
git commit -m "feat: add runners and orchestrator run_all.py"
```

---

## Task 17: Финальная проверка

- [ ] Убедиться что Redis запущен: `redis-cli -s /var/run/redis/redis.sock ping`
- [ ] Запустить один MD-коллектор вручную:
  ```bash
  cd /root/bali4.0_data
  PYTHONPATH=/root/bali4.0_data python3 collectors/md/binance_spot.py 2>&1 | head -20
  ```
  Ожидаем JSON-события: `startup` → `symbols_loaded` → `redis_connected` → `connecting` → `connected` → `first_message`
- [ ] Проверить Redis-ключ:
  ```bash
  redis-cli -s /var/run/redis/redis.sock hgetall "md:bn:s:BTCUSDT"
  ```
- [ ] Запустить полную систему: `python3 run_all.py --no-dash`
- [ ] Убедиться что сигналы пишутся: `tail -f signals/signals.jsonl`
- [ ] **Финальный коммит и push:**
  ```bash
  git push origin main
  ```
