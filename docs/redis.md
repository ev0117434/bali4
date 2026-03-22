# Redis

## Установка и настройка

```bash
sudo bash setup_redis.sh           # установка + настройка + проверка
sudo bash setup_redis.sh --check   # только проверка (без изменений)
```

## Ключевые параметры

| Параметр | Значение | Зачем |
|----------|----------|-------|
| `port` | `0` | TCP выключен |
| `unixsocket` | `/var/run/redis/redis.sock` | минимальная задержка |
| `maxmemory` | `40gb` | потолок памяти |
| `maxmemory-policy` | `volatile-ttl` | вытеснение по TTL |
| `save ""` | — | RDB отключена |
| `appendonly` | `no` | AOF отключён |
| `hz` | `100` | высокочастотный таймер |
| `io-threads` | `8` | параллельный I/O (Redis 7+) |
| `databases` | `2` | два DB |

Persistence полностью отключена — данные volatile. Перезапуск Redis сбрасывает всё.

## Подключение из Python

```python
import redis

r = redis.Redis(
    unix_socket_path='/var/run/redis/redis.sock',
    decode_responses=False,
)
r.ping()  # → True
```

## Что настраивает скрипт

1. **Установка** — `apt-get install redis-server`, проверка версии
2. **ОС** — отключение THP, `vm.overcommit_memory=1`, `net.core.somaxconn=1024`, fd-лимиты 65536
3. **redis.conf** — генерирует конфиг с учётом версии (6 / 7+)
4. **Права** — директория сокета, пользователь добавляется в группу `redis`
5. **Systemd** — override: `Type=notify`, `TimeoutStartSec=30`, `LimitNOFILE=65536`
6. **Health check** — доступность, латентность, конфиг, память, slowlog, THP, бенчмарк

## Типичные проблемы

| Симптом | Решение |
|---------|---------|
| Permission denied на сокет | `sudo usermod -aG redis $USER`, переlogin |
| Redis не стартует | `journalctl -u redis-server -n 30` |
| THP включены (предупреждение) | `sudo systemctl restart disable-thp` |
| Evicted keys > 0 | Данных больше `maxmemory` — увеличить лимит |
