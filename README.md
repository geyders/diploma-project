# Novelize

Інформаційна система підтримки створення та редагування художніх творів. Проєкт зберігає структуру книги й версії тексту, веде граф сутностей та виконує асинхронний аналіз тексту й перевірку змістової узгодженості.

## Документація

- [Розділ 1 — аналіз предметної області та посилання на робочий документ](docs/README.md)
- [Розділ 2 — проєктування архітектури](docs/2_design.md)
- [SRS — специфікація вимог](docs/srs.md)
- [OpenAPI 3.0.3](docs/api/openapi.yaml)
- [ADR-001 — вибір брокера повідомлень](docs/adr/0001-choice-of-technology.md)
- [Лабораторна робота №7](docs/lab7.md)
- [AI Insights для ЛР№5 та ЛР№6](AI_Insights_LR5_LR6.md)

## Архітектура

Архітектура описана за моделлю C4. Основні технології: React, TypeScript, NestJS, Python/FastAPI, PostgreSQL, Neo4j, ChromaDB, Apache Kafka, Keycloak та NGINX.

## Перевірка документації

Markdown-файли в `docs/` перевіряються GitHub Actions через markdownlint під час Pull Request до `main` та push у робочі гілки.
