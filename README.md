# Bali 4.0 — Crypto Spread Monitor

Система мониторинга спредов между спот- и фьючерсными рынками пяти бирж в реальном времени.

## Содержание

- [Структура проекта](#структура-проекта)
- [Быстрый старт](#быстрый-старт)
- [Компоненты](#компоненты)
  - [dictionaries/](#dictionaries)
  - [setup\_redis.sh](#setup_redissh)
- [Конфигурационные файлы](#конфигурационные-файлы)
- [Зависимости](#зависимости)

---

## Структура проекта

```
bali4.0_data/
├── README.md
├── setup_redis.sh          # Установка и настройка Redis
└── dictionaries/           # Модуль сбора и валидации торговых пар
    ├── main.py             # Оркестратор — запускает все фазы
    ├── binance/
    ├── bybit/
    ├── okx/
    ├── gate/
    ├── bitget/
    ├── combination/        # Пересечения активных пар (генерируется)
    ├── subscribe/          # Файлы подписки для коллекторов (генерируется)
    └── config/             # Числовые словари символов и источников (генерируется)
        ├── symbol_ids.json
        └── source_ids.json
```

---

## Быстрый старт

**1. Настроить Redis (один раз):**

```bash
sudo bash setup_redis.sh
```

**2. Обновить торговые пары (перед каждым стартом коллекторов):**

```bash
cd dictionaries
python3 main.py
```

Время выполнения: ~65–70 секунд.

**3. Проверить здоровье Redis:**

```bash
sudo bash setup_redis.sh --check
```

---

## Компоненты

### dictionaries/

Модуль собирает активные торговые пары с 5 бирж, строит пересечения и файлы подписки.

#### Фазы выполнения

```
Фаза 1 — REST API (~5 сек)
  Параллельный опрос всех 5 бирж через ThreadPoolExecutor.
  Результат: сырые списки пар (spot + futures) по каждой бирже.

Фаза 2 — WS-валидация (~60 сек)
  Все биржи стартуют одновременно в одном asyncio event loop.
  Пара считается активной, если пришёл хотя бы один ответ за 60 сек.
  Результат: списки *активных* пар по каждой бирже.

Фаза 3 — Пересечения
  Построение combination/*.txt по схеме: spot_A ∩ futures_B.

Фаза 4 — Файлы подписки
  Агрегация subscribe/*.txt из combination-файлов без дублей.

Фаза 5 — Конфигурационные словари
  config/symbol_ids.json и config/source_ids.json.
```

#### Биржи и их особенности

| Биржа | Нативный формат | Нормализованный | WS-канал |
|-------|----------------|-----------------|----------|
| Binance | `BTCUSDT` | `BTCUSDT` | `bookTicker`, чанки по 300 |
| Bybit | `BTCUSDT` | `BTCUSDT` | `orderbook.1`, батчи 10/200 |
| OKX | `BTC-USDT-SWAP` | `BTCUSDT` | `tickers`, чанки по 300 |
| Gate.io | `BTC_USDT` | `BTCUSDT` | `book_ticker`, батчи 100 |
| Bitget | `BTCUSDT` | `BTCUSDT` | `books1`, батчи 100 |

OKX и Gate.io хранят нативные символы отдельно (`*_native.txt`) — они нужны для WS-подписки, поскольку биржа принимает только собственный формат.

#### Выходные файлы

**`combination/`** — 20 файлов, каждый — отсортированное пересечение двух рынков:

```
binance_spot_bybit_futures.txt   = binance_spot  ∩ bybit_futures   (~332 пар)
bybit_spot_binance_futures.txt   = bybit_spot    ∩ binance_futures  (~309 пар)
binance_spot_okx_futures.txt     = binance_spot  ∩ okx_futures      (~195 пар)
okx_spot_binance_futures.txt     = okx_spot      ∩ binance_futures  (~262 пар)
bybit_spot_okx_futures.txt       = bybit_spot    ∩ okx_futures      (~189 пар)
okx_spot_bybit_futures.txt       = okx_spot      ∩ bybit_futures    (~229 пар)
binance_spot_gate_futures.txt    = binance_spot  ∩ gate_futures     (~327 пар)
gate_spot_binance_futures.txt    = gate_spot     ∩ binance_futures  (~481 пар)
bybit_spot_gate_futures.txt      = bybit_spot    ∩ gate_futures     (~300 пар)
gate_spot_bybit_futures.txt      = gate_spot     ∩ bybit_futures    (~457 пар)
okx_spot_gate_futures.txt        = okx_spot      ∩ gate_futures     (~234 пар)
gate_spot_okx_futures.txt        = gate_spot     ∩ okx_futures      (~237 пар)
binance_spot_bitget_futures.txt  = binance_spot  ∩ bitget_futures   (~326 пар)
bitget_spot_binance_futures.txt  = bitget_spot   ∩ binance_futures  (~396 пар)
bybit_spot_bitget_futures.txt    = bybit_spot    ∩ bitget_futures   (~273 пар)
bitget_spot_bybit_futures.txt    = bitget_spot   ∩ bybit_futures    (~372 пар)
okx_spot_bitget_futures.txt      = okx_spot      ∩ bitget_futures   (~225 пар)
bitget_spot_okx_futures.txt      = bitget_spot   ∩ okx_futures      (~210 пар)
gate_spot_bitget_futures.txt     = gate_spot     ∩ bitget_futures   (~418 пар)
bitget_spot_gate_futures.txt     = bitget_spot   ∩ gate_futures     (~384 пар)
```

**`subscribe/`** — 10 файлов, по одному на каждый рынок каждой биржи. Формируются объединением всех combination-файлов, содержащих ключевое слово данного рынка:

```
subscribe/binance/binance_spot.txt      (~367 пар)
subscribe/binance/binance_futures.txt   (~529 пар)
subscribe/bybit/bybit_spot.txt          (~344 пар)
subscribe/bybit/bybit_futures.txt       (~497 пар)
subscribe/okx/okx_spot.txt             (~285 пар)
subscribe/okx/okx_futures.txt          (~249 пар)
subscribe/gate/gate_spot.txt           (~533 пар)
subscribe/gate/gate_futures.txt        (~480 пар)
subscribe/bitget/bitget_spot.txt       (~432 пар)
subscribe/bitget/bitget_futures.txt    (~455 пар)
```

**`config/`** — числовые словари (см. раздел [Конфигурационные файлы](#конфигурационные-файлы)).

---

### setup_redis.sh

Скрипт установки и настройки Redis, оптимизированного для системы реального времени.

#### Режимы запуска

```bash
sudo bash setup_redis.sh           # Полная установка + настройка + проверка
sudo bash setup_redis.sh --check   # Только проверка (без изменений)
```

#### Что делает скрипт

| Этап | Действие |
|------|----------|
| 1/6 Установка | Устанавливает Redis через apt, проверяет версию (6 / 7+) |
| 2/6 ОС | Отключает Transparent Huge Pages, настраивает sysctl и fd-лимиты |
| 3/6 redis.conf | Генерирует конфиг: Unix socket, без TCP, без persistence |
| 4/6 Права | Создаёт сокет-директорию, добавляет пользователя в группу redis |
| 5/6 Перезапуск | Перезапускает Redis через systemd с override-файлом |
| 6/6 Health check | Проверяет доступность, латентность, конфиг, память, slowlog, ОС, бенчмарк |

#### Ключевые параметры Redis

| Параметр | Значение | Причина |
|----------|----------|---------|
| `port` | `0` | TCP выключен — только Unix socket |
| `unixsocket` | `/var/run/redis/redis.sock` | Минимальная задержка для локальных клиентов |
| `maxmemory` | `40gb` | Ограничение под volatile данные |
| `maxmemory-policy` | `volatile-ttl` | Вытеснение по TTL при нехватке памяти |
| `save ""` | — | RDB persistence отключена |
| `appendonly` | `no` | AOF отключён |
| `hz` | `100` | Высокочастотный таймер событий |
| `io-threads` | `8` | Параллельный I/O (Redis 7+) |

#### Подключение из Python

```python
import redis
r = redis.Redis(unix_socket_path='/var/run/redis/redis.sock', decode_responses=False)
r.ping()  # → True
```

---

## Конфигурационные файлы

Генерируются автоматически при запуске `dictionaries/main.py`.

### config/symbol_ids.json

Все уникальные символы из всех combination-файлов, отсортированные и пронумерованные.

```json
{
  "AAVEUSDT": 0,
  "ADAUSDT": 1,
  "BTCUSDT": 42,
  ...
}
```

**Текущий размер:** ~615 символов.

Используется коллекторами для записи `symbol_id` (integer) вместо строки — уменьшает объём данных и ускоряет поиск.

> ID **не стабильны** между запусками: при изменении набора пар индексы могут сдвигаться. Всегда используй актуальный файл, сгенерированный последним запуском `main.py`.

### config/source_ids.json

Числовые ID рынков (биржа + тип). Порядок фиксированный, алфавитный.

```json
{
  "binance_futures": 0,
  "binance_spot":    1,
  "bitget_futures":  2,
  "bitget_spot":     3,
  "bybit_futures":   4,
  "bybit_spot":      5,
  "gate_futures":    6,
  "gate_spot":       7,
  "okx_futures":     8,
  "okx_spot":        9
}
```

Используется коллекторами для записи `source_id` (integer) вместо строки.

---

## Зависимости

```bash
pip install websockets
```

Стандартная библиотека: `asyncio`, `concurrent.futures`, `json`, `urllib.request`, `pathlib`, `time`.

Python **3.10+**.

Redis **6+** (рекомендуется 7+ для `io-threads`).
