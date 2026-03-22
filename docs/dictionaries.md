# Модуль dictionaries

## Запуск

```bash
cd /root/bali4.0_data/dictionaries
python3 main.py
# ~65–70 секунд
```

## Фазы

### Фаза 1 — REST (~5 сек)

Все 5 бирж опрашиваются параллельно через `ThreadPoolExecutor`.

```python
fetch_pairs() → (spot: list, futures: list)
```

Фильтр: `quoteAsset ∈ {USDT, USDC}`, статус = торгуется.
Результат сохраняется в `{exchange}/data/{exchange}_{market}.txt`.

### Фаза 2 — WS-валидация (~60 сек)

Все биржи стартуют одновременно через `asyncio.gather`.

Пара считается **активной**, если пришёл хотя бы один ответ WebSocket за 60 секунд.
Неактивные пары (неликвид, остановлены) — отсеиваются.

| Биржа | Канал | Чанк |
|-------|-------|------|
| Binance | `bookTicker` | 300 |
| Bybit | `orderbook.1` | 10 спот / 200 фьючерсы |
| OKX | `tickers` | 300 |
| Gate.io | `book_ticker` | 100 |
| Bitget | `books1` | 100 |

Результат: `{exchange}/data/{exchange}_{market}_active.txt`.

### Фаза 3 — Пересечения

```
combination/{spot_exchange}_spot_{futures_exchange}_futures.txt
  = active_spot ∩ active_futures
```

20 файлов: все комбинации пар бирж (Binance, Bybit, OKX, Gate, Bitget).

### Фаза 4 — Подписка

```
subscribe/{exchange}/{exchange}_{market}.txt
  = ∪ всех combination-файлов, содержащих ключевое слово рынка
```

10 файлов. Дубли убраны. Читаются коллекторами при старте.

### Фаза 5 — Config

Генерируются `config/symbol_ids.json` и `config/source_ids.json`.
Подробнее: [config.md](config.md).

## Нормализация символов

Всё приводится к формату `BTCUSDT` для построения пересечений.

| Биржа | Нативный | Нормализованный |
|-------|----------|-----------------|
| Binance | `BTCUSDT` | `BTCUSDT` |
| Bybit | `BTCUSDT` | `BTCUSDT` |
| OKX | `BTC-USDT` / `BTC-USDT-SWAP` | `BTCUSDT` |
| Gate.io | `BTC_USDT` | `BTCUSDT` |
| Bitget | `BTCUSDT` | `BTCUSDT` |

OKX и Gate.io хранят оба формата: нативный нужен для WS-подписки, нормализованный — для пересечений.

## Диагностика

```bash
# Количество пар после последнего запуска
wc -l combination/*.txt
wc -l subscribe/*/*.txt

# Проверить одну биржу (только REST, ~5 сек)
python3 binance/binance_pairs.py

# Проверить одну биржу (REST + WS, ~65 сек)
python3 binance/binance_ws.py
```
