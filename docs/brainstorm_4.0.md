# BALI 4.0 — Брейнсторм архитектуры
> Дата: 2026-03-22

## Что имеем в 3.0 (база)

| Проблема | Описание |
|----------|----------|
| Дублирование | ~25 почти одинаковых файлов по 200 строк |
| Логи | Хорошие, но нет: E2E latency ws→redis, lag exchange_ts vs local_ts |
| Нет обмена | Каждый скрипт — изолированный процесс, нет внутренней шины |
| Мониторинг | Только stats каждые 30с, нет алертов на аномальные задержки внутри скрипта |

---

## Предлагаемая архитектура 4.0

### Слой 1 — Core (общий модуль)

```
collectors/
  core/
    base_collector.py   # абстрактный WS-коллектор
    redis_writer.py     # flush + pipeline логика
    logger.py           # JSON логгер с latency-трекером
    stats.py            # stats_loop, counters
```

**`base_collector.py`** — один раз пишем:
- WS connect/reconnect с exponential backoff
- batch accumulation + flush каждые 200ms
- `first_message`, `connected`, `disconnected`, `reconnecting` события
- абстрактный метод `parse(raw) → (key, mapping) | None`

Каждая биржа — маленький файл ~60 строк:
```python
class BinanceSpot(BaseCollector):
    WS_URL = "wss://stream.binance.com:9443/stream?streams="
    REDIS_PREFIX = "md:bn:s"

    def build_url(self, syms): ...
    def parse(self, raw): ...  # вытащить s, b, a
```

---

### Слой 2 — Коллекторы (что пишем)

#### MD — Best Bid/Ask (10 скриптов → 10 классов)
```
md/
  binance_spot.py
  binance_futures.py
  bybit_spot.py
  bybit_futures.py
  okx_spot.py
  okx_futures.py
  gate_spot.py
  gate_futures.py
  bitget_spot.py
  bitget_futures.py
```
Redis: `md:{ex}:{mkt}:{sym}` → `{b, a, t, et}`
- `b` — best bid
- `a` — best ask
- `t` — local timestamp ms
- `et` — exchange timestamp ms (если биржа предоставляет)

#### OB — Orderbook Level 10 (10 скриптов)
```
ob/
  binance_spot.py
  binance_futures.py
  bybit_spot.py
  bybit_futures.py
  okx_spot.py
  okx_futures.py
  gate_spot.py
  gate_futures.py
  bitget_spot.py
  bitget_futures.py
```
Redis: `ob:{ex}:{mkt}:{sym}` → `{b1..b10, bq1..bq10, a1..a10, aq1..aq10, t}`

#### FR — Funding Rate (5 скриптов, только futures)
```
fr/
  binance_futures.py
  bybit_futures.py
  okx_futures.py
  gate_futures.py
  bitget_futures.py
```
Redis: `fr:{ex}:{sym}` → `{r, nr, ft, ts}`

---

### Слой 3 — Логи (главное улучшение 4.0)

Каждый JSON лог содержит:

```json
{
  "ts": 1742600000.123,
  "lvl": "INFO",
  "script": "binance_spot",
  "event": "stats",

  "ws_recv_to_redis_ms": 12.4,
  "redis_pipeline_ms": 1.8,
  "batch_age_ms": 215.0,
  "exchange_lag_ms": 45.0,

  "msgs_total": 125000,
  "msgs_per_sec": 4200.0,
  "flushes_total": 312,
  "symbols_active": 487,

  "connections": 3,
  "ws_uptime_sec": 3600,
  "reconnects": 0
}
```

**Таблица событий:**

| event | когда |
|-------|-------|
| `startup` | старт скрипта |
| `symbols_loaded` | загрузка списка символов |
| `redis_connected` | подключение к Redis |
| `connecting` | попытка WS подключения (chunk_id, symbols_count) |
| `connected` | успешное подключение (subscribe_ms) |
| `first_message` | первое сообщение (ms_since_connected) |
| `stats` | каждые 30с — полные метрики |
| `slow_flush` | если redis_pipeline_ms > 50ms |
| `slow_batch` | если batch_age_ms > 500ms |
| `disconnected` | обрыв (reason, downtime_sec) |
| `reconnecting` | delay_sec, attempt |
| `reconnected` | успешно (downtime_sec) |
| `shutdown` | SIGTERM получен |

---

### Слой 4 — Spread Monitor

- Читает `md:*` ключи для всех 20 направлений (5×4 комбинации spot/futures)
- Считает `(fut_bid - spot_ask) / spot_ask * 100` каждые 300ms
- Сигнал: спред ≥ 1%, данные свежее 300с, кулдаун 3500с
- Аномалия: спред ≥ 100% подтверждена 2 цикла подряд

**Логи spread_monitor:**
```json
{
  "event": "cycle_stats",
  "cycle_ms": 18.4,
  "redis_read_ms": 12.1,
  "pairs_checked": 9420,
  "signals_fired": 2,
  "cooldowns_active": 14,
  "stale_skipped": 83
}
```

**Файлы сигналов:**
- `signals/signals.jsonl` + `signals.csv`
- `signals/anomalies.jsonl` + `anomalies.csv`

---

### Слой 5 — Оркестрация (run_all.py)

```
ob_group  → cores 12-19  (10 OB коллекторов, тяжёлые)
md_group  → cores 20-23  (10 MD коллекторов, I/O)
fr_group  → cores 24-25  (5 FR коллекторов, лёгкие)
monitors  → cores 26-27  (spread_monitor, staleness, redis_mon)
run_all   → cores 28-31  (оркестратор, дашборд)
```

---

## Порядок реализации

```
1. core/logger.py          — JSON логгер с latency tracker
2. core/redis_writer.py    — flush + pipeline с метриками
3. core/base_collector.py  — базовый WS коллектор
4. core/stats.py           — stats_loop
5. md/ × 10 коллекторов   — MD best bid/ask
6. ob/ × 10 коллекторов   — OB level 10
7. fr/ × 5 коллекторов    — Funding rates
8. spread_monitor.py       — спреды + сигналы
9. run_all.py              — оркестратор
```

---

## Ключевые улучшения vs 3.0

| | 3.0 | 4.0 |
|--|-----|-----|
| Структура | 1 большой файл/биржа | core/ + маленькие файлы |
| Код на биржу | ~200 строк | ~60 строк |
| E2E latency лог | нет | есть |
| Exchange lag | нет | есть (где биржа даёт ts) |
| slow_flush алерт | нет | есть |
| symbols_active | нет | есть |
