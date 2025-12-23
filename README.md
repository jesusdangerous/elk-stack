# ELK Stack с автоматическим удалением логов (ILM)

## Цель работы

Развернуть стек ELK (Elasticsearch, Logstash, Kibana) с источником логов Nginx и
настроить автоматическое удаление логов старше 7 минут с использованием
Index Lifecycle Management (ILM) в Elasticsearch.

Удаление логов должно происходить автоматически, без использования cron-задач
или фильтрации событий в Logstash.

## Порты
- 9200 — Elasticsearch
- 5601 — Kibana
- 5044 — Logstash Beats
- 8080 — Nginx

## Быстрый старт

```bash
# 0) Сеть и Elasticsearch
docker network create elk-net
cd master
docker compose up -d elasticsearch
cd ..
sleep 20

# 1) ILM policy (rollover 1m, delete 7m)
curl -X PUT "http://localhost:9200/_ilm/policy/nginx-logs-policy" \
  -H "Content-Type: application/json" -u elastic:changeme \
  -d '{"policy":{"phases":{"hot":{"actions":{"rollover":{"max_age":"1m","max_size":"50mb"}}},"delete":{"min_age":"7m","actions":{"delete":{}}}}}}'

# 2) Index template
curl -X PUT "http://localhost:9200/_index_template/nginx-logs-template" \
  -H "Content-Type: application/json" -u elastic:changeme \
  -d '{"index_patterns":["nginx-logs-*"],"template":{"settings":{"index.lifecycle.name":"nginx-logs-policy","index.lifecycle.rollover_alias":"nginx-logs","number_of_shards":1,"number_of_replicas":0}},"priority":100}'

# 3) Стартовый write-индекс
curl -X PUT "http://localhost:9200/nginx-logs-000001" \
  -H "Content-Type: application/json" -u elastic:changeme \
  -d '{"aliases":{"nginx-logs":{"is_write_index":true}}}'

# 4) Ускорить ILM опрос (для тестов)
curl -X PUT "http://localhost:9200/_cluster/settings" \
  -H "Content-Type: application/json" -u elastic:changeme \
  -d '{"transient":{"indices.lifecycle.poll_interval":"10s"}}'

# 5) Поднять остальное
cd master && docker compose up -d logstash kibana && cd ..
cd slave  && docker compose up -d               && cd ..
```

## Поведение
- Ролловер ~каждую минуту (`max_age=1m`).
- Удаление индексов после 7 минут возраста; с `poll_interval=10s` задержка удаления обычно не более ~20s после порога.
- Alias `nginx-logs` всегда смотрит на текущий write-индекс.

## Доступы
- Elasticsearch: `elastic / changeme`
- Kibana system: `kibana_system / kibana123!`
