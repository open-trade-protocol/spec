# OpenTrade Protocol — Specification

Спецификация протокола OpenTrade: форматы данных, схемы, JSON-LD контексты и OpenAPI документация.

## Структура

```
spec/
├── openapi/
│   └── openapi.yaml    # OpenAPI 3.1 спецификация API
├── schemas/
│   └── *.json          # JSON Schema для категорий товаров
├── contexts/
│   └── *.jsonld        # JSON-LD контексты для semantic web
├── protocols/
│   └── *.md            # Спецификации протоколов (escrow, federation и т.д.)
├── examples/
│   └── *.json          # Примеры сообщений и листингов
└── README.md
```

## Быстрый старт

```bash
# Просмотр OpenAPI документации
# Откройте spec/openapi/openapi.yaml в IDE с поддержкой OpenAPI
# или используйте swagger-ui:
docker run -p 8080:8080 -e SWAGGER_JSON=/openapi.yaml \
  -v $(pwd)/openapi:/openapi swaggerapi/swagger-ui
```

## Конвенции

- Все схемы следуют OpenAPI 3.1
- JSON-LD контексты хранятся в `contexts/`
- Примеры листингов — в `examples/`
- Протоколы (escrow, federation, trust) — в `protocols/`

## Лицензия

CC-BY 4.0
