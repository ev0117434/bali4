# Архитектура системы

## Обзор

Bali 4.0 — система мониторинга спредов между спот- и фьючерсными рынками.
Отслеживает расхождение цен между биржами в реальном времени.

## Компоненты

```
bali4.0_data/
├── dictionaries/   — сбор и валидация торговых пар
└── setup_redis.sh  — настройка хранилища данных
```

## Поток данных

```
REST API (5 бирж)
      │
      ▼
  Фаза 1: fetch_pairs()
  Получить все пары, сохранить в {exchange}/data/
      │
      ▼
  Фаза 2: WS-валидация (~60 сек, параллельно)
  Оставить только пары с активным трафиком
      │
      ▼
  Фаза 3: combination/
  Пересечения: spot_A ∩ futures_B (20 файлов)
      │
      ▼
  Фаза 4: subscribe/
  Агрегация по рынкам (10 файлов)
      │
      ▼
  Фаза 5: config/
  symbol_ids.json + source_ids.json
      │
      ▼
  Коллекторы читают subscribe/ и config/
  → пишут в Redis (HSET по symbol_id + source_id)
```

## Биржи

| Биржа   | Спот | Фьючерсы |
|---------|------|----------|
| Binance | ✓ | ✓ USD-M |
| Bybit   | ✓ | ✓ Linear |
| OKX     | ✓ | ✓ SWAP |
| Gate.io | ✓ | ✓ USDT-M |
| Bitget  | ✓ | ✓ USDT-M |

## Redis

Все данные хранятся в Redis через Unix socket `/var/run/redis/redis.sock`.
Persistence отключена — данные volatile, при перезапуске Redis данные сбрасываются.
