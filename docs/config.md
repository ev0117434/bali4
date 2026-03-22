# Конфигурационные файлы

Генерируются автоматически при каждом запуске `dictionaries/main.py`.
Расположение: `dictionaries/config/`.

## symbol_ids.json

Числовые ID всех торговых символов.

```json
{
  "AAVEUSDT": 0,
  "ADAUSDT":  1,
  "BTCUSDT":  42,
  ...
}
```

- Источник: объединение всех `combination/*.txt`
- Сортировка: алфавитная
- Размер: ~615 символов

**Важно:** ID не стабильны между запусками. При изменении набора пар индексы сдвигаются. Всегда используй файл от последнего запуска `main.py`.

```python
import json
from pathlib import Path

SYMBOL_IDS = json.loads(
    (Path(__file__).parent.parent / "dictionaries/config/symbol_ids.json").read_text()
)
sid = SYMBOL_IDS.get("BTCUSDT", -1)  # → 42
```

## source_ids.json

Числовые ID рынков (биржа + тип).

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

- Порядок фиксированный (алфавитный), не меняется между запусками
- 10 источников: 5 бирж × 2 рынка (spot + futures)

```python
SOURCE_IDS = json.loads(
    (Path(__file__).parent.parent / "dictionaries/config/source_ids.json").read_text()
)
src = SOURCE_IDS["binance_spot"]  # → 1
```
