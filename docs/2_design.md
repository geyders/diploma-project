# Розділ 2. Проєктування архітектури системи

Вимоги: [srs.md](./srs.md). Рішення: [ADR-001](./adr/0001-choice-of-technology.md).

## 2.1 Контекстне та контейнерне моделювання системи (C4 Model)

### Рівень 1. System Context

```mermaid
flowchart LR
    author["Автор<br/>[Person]<br/>Створює, структурує та редагує художній твір"]
    editor["Редактор<br/>[Person]<br/>Читає книгу та створює коментарі"]
    novelize["Novelize<br/>[Software System]<br/>Контроль структурної та змістової цілісності художніх текстів"]
    smtp["SMTP-провайдер<br/>[Software System, зовнішній]<br/>Доставка листів підтвердження адреси"]

    author -->|"HTTPS, WSS: редагує текст, отримує результати аналізу"| novelize
    editor -->|"HTTPS: читає книгу, коментує"| novelize
    novelize -->|"SMTP / TLS: надсилає лист підтвердження"| smtp

    classDef person fill:#08427b,color:#fff,stroke:#052e56
    classDef sys fill:#1168bd,color:#fff,stroke:#0b4884
    classDef ext fill:#999,color:#fff,stroke:#666
    class author,editor person
    class novelize sys
    class smtp ext
```

| Елемент | Опис | Вимоги |
|---|---|---|
| Автор | Основний актор: ієрархія твору, сутності, запуск аналізу | [FUN-01](./srs.md#srs-fun-01), [FUN-07](./srs.md#srs-fun-07) |
| Редактор | Роль із правами читання та коментування | [FUN-10](./srs.md#srs-fun-10) |
| Novelize | Система в межах Scope | [srs.md, розділ 1](./srs.md) |
| SMTP-провайдер | Зовнішній сервіс листів підтвердження | [FUN-02](./srs.md#srs-fun-02) |

### Рівень 2. Container Diagram

```mermaid
flowchart TB
    author(["Автор<br/>[Person]"])
    editor(["Редактор<br/>[Person]"])
    smtp["SMTP-провайдер<br/>[Software System, зовнішній]"]

    subgraph sys["Novelize [System]"]
        spa["Web SPA<br/>[React, TypeScript]"]
        gw["API Gateway<br/>[Kong Gateway]"]
        idp["Identity Provider<br/>[Keycloak]"]
        core["Core API Service<br/>[NestJS, TypeScript]"]
        egs["Entity Graph Service<br/>[NestJS, TypeScript]"]
        nlp["NLP Analysis Service<br/>[Python, FastAPI, spaCy]"]
        rt["Realtime Service<br/>[Node.js, ws]"]
        lt["LanguageTool Server<br/>[Java]"]
        kafka[["Apache Kafka<br/>[KRaft]"]]
        pg[("PostgreSQL 16")]
        neo4j[("Neo4j 5")]
        redis[("Redis 7")]
        prom["Prometheus + Grafana"]
    end

    author -->|"HTTPS"| spa
    editor -->|"HTTPS"| spa
    spa -->|"OIDC Authorization Code + PKCE / HTTPS"| idp
    spa -->|"REST / HTTPS, WSS"| gw
    gw -->|"JWKS / HTTP"| idp
    gw -->|"RESP / TCP: лічильники запитів"| redis
    gw -->|"REST / HTTP"| core
    gw -->|"REST / HTTP"| egs
    gw -->|"WebSocket / HTTP"| rt
    idp -->|"SMTP / TLS"| smtp
    idp -->|"SQL / TCP"| pg
    core -->|"SQL / TCP"| pg
    core -->|"Kafka protocol / TCP: text.fragment.saved"| kafka
    nlp -->|"Kafka protocol / TCP: споживає text.fragment.saved, публікує analysis.completed, entities.extracted"| kafka
    nlp -->|"REST / HTTP"| lt
    nlp -->|"RESP / TCP: ідемпотентність"| redis
    egs -->|"Kafka protocol / TCP: споживає entities.extracted, book.access.changed, публікує consistency.checked"| kafka
    egs -->|"Cypher / Bolt"| neo4j
    egs -->|"RESP / TCP: ідемпотентність"| redis
    rt -->|"Kafka protocol / TCP: споживає analysis.completed, consistency.checked"| kafka
    prom -.->|"HTTP /metrics"| gw
    prom -.->|"HTTP /metrics"| core
    prom -.->|"HTTP /metrics"| egs
    prom -.->|"HTTP /metrics"| nlp
    prom -.->|"HTTP /metrics"| rt

    classDef person fill:#08427b,color:#fff,stroke:#052e56
    classDef cont fill:#1168bd,color:#fff,stroke:#0b4884
    classDef ext fill:#999,color:#fff,stroke:#666
    class author,editor person
    class spa,gw,idp,core,egs,nlp,rt,lt,kafka,pg,neo4j,redis,prom cont
    class smtp ext
```

| Контейнер | Відповідальність | Вимоги |
|---|---|---|
| Web SPA | Інтерфейс редагування; отримання результатів | [FUN-17](./srs.md#srs-fun-17) |
| API Gateway | Перевірка JWT, Rate Limiting, маршрутизація | [SEC-01](./srs.md#srs-nfr-sec-01), [SEC-02](./srs.md#srs-nfr-sec-02) |
| Identity Provider | Реєстрація, підтвердження, OAuth 2.0 / OIDC, видача JWT | [FUN-01…04](./srs.md#srs-fun-01), [SEC-01](./srs.md#srs-nfr-sec-01) |
| Core API Service | Ієрархія, версії, профіль, ролі, видалення та експорт даних, Outbox Relay | [FUN-05…10](./srs.md#srs-fun-05), [FUN-13](./srs.md#srs-fun-13), [REL-02](./srs.md#srs-nfr-rel-02) |
| Entity Graph Service | Реєстр сутностей, хронологія, перевірка узгодженості | [FUN-11, 12, 16](./srs.md#srs-fun-11) |
| NLP Analysis Service | Мовна перевірка, стилістика, виділення згадок | [FUN-14, 15](./srs.md#srs-fun-14), [REL-04](./srs.md#srs-nfr-rel-04) |
| LanguageTool Server | Правила граматики та пунктуації | [FUN-13](./srs.md#srs-fun-13) |
| Realtime Service | WebSocket/SSE-доставка | [FUN-17](./srs.md#srs-fun-17), [PERF-03](./srs.md#srs-nfr-perf-03) |
| Apache Kafka | Асинхронний обмін подіями, DLT | [ADR-001](./adr/0001-choice-of-technology.md) |
| PostgreSQL | Реляційні дані, outbox | [FUN-07, 08](./srs.md#srs-fun-07) |
| Neo4j | Граф сутностей | [FUN-11, 12](./srs.md#srs-fun-11) |
| Redis | Лічильники, ідемпотентність | [SEC-02](./srs.md#srs-nfr-sec-02), [REL-02](./srs.md#srs-nfr-rel-02) |
| Prometheus + Grafana | Метрики | [FUN-18](./srs.md#srs-fun-18) |

Кожен зв'язок рівня 2 має протокол у третьому аргументі `Rel`. Топіки Kafka: `text.fragment.saved`, `analysis.completed`, `entities.extracted`, `consistency.checked`, `book.access.changed`.

## 2.2 Компонентне моделювання критичного сервісу системи (C4 Level 3)

Обрано контейнер **Entity Graph Service** (NestJS, TypeScript): реалізує графову перевірку узгодженості, що відрізняє Novelize від аналогів (розділ 1.3).

```mermaid
flowchart TB
    gw["API Gateway<br/>[Kong Gateway]"]

    subgraph boundary["Entity Graph Service [Container: NestJS, TypeScript]"]
        subgraph guards["Захист і валідація"]
            jwt["JwtAuthGuard<br/>[NestJS Guard]<br/>підпис, exp, aud"]
            acl["AccessGuard<br/>[NestJS Guard]<br/>RBAC: власник або редактор"]
            val["ValidationPipe<br/>[class-validator]<br/>валідація DTO"]
        end
        subgraph inb["Inbound"]
            entctl["EntityController<br/>[REST Controller]"]
            chkctl["ConsistencyController<br/>[REST Controller]"]
            cons["ExtractedEntitiesConsumer<br/>[Kafka Consumer]"]
        end
        subgraph cross["Наскрізні"]
            err["GlobalExceptionFilter<br/>[NestJS Filter]"]
            tel["TelemetryInterceptor<br/>[pino, prom-client]"]
        end
        subgraph corel["Core: Application Services"]
            regsvc["EntityRegistryService"]
            chksvc["ConsistencyCheckService"]
            rules["ConsistencyRule<br/>[Strategy]"]
        end
        subgraph ports["Domain: Ports"]
            repoport{{"EntityRepository<br/>[Port]"}}
            pubport{{"EventPublisher<br/>[Port]"}}
            idemport{{"IdempotencyStore<br/>[Port]"}}
        end
        subgraph adapters["Outbound: Adapters"]
            neoad["Neo4jEntityRepository<br/>[neo4j-driver]"]
            kafad["KafkaEventPublisher<br/>[kafkajs]"]
            redad["RedisIdempotencyStore<br/>[ioredis]"]
        end
    end

    kafka[["Apache Kafka<br/>[KRaft]"]]
    neo4j[("Neo4j 5")]
    redis[("Redis 7")]

    gw -->|"REST / HTTP, Authorization header"| jwt
    jwt -->|"claims, in-process"| acl
    acl -->|"авторизований запит, in-process"| val
    val -->|"валідований DTO, in-process"| entctl
    val -->|"валідований DTO, in-process"| chkctl
    entctl -.->|"винятки, in-process"| err
    chkctl -.->|"винятки, in-process"| err
    entctl -.->|"журнали, метрики, in-process"| tel
    chkctl -.->|"журнали, метрики, in-process"| tel
    entctl -->|"Constructor DI"| regsvc
    chkctl -->|"Constructor DI"| chksvc
    cons -->|"Constructor DI"| regsvc
    cons -->|"Constructor DI"| chksvc
    cons -->|"Interface call: перевірка messageId"| idemport
    cons -->|"Kafka protocol / TCP: підписка на entities.extracted"| kafka
    regsvc -->|"Interface call"| repoport
    chksvc -->|"Strategy, interface call"| rules
    chksvc -->|"Interface call"| pubport
    rules -->|"Interface call"| repoport
    repoport -.-|"реалізує, Dependency Inversion"| neoad
    pubport -.-|"реалізує, Dependency Inversion"| kafad
    idemport -.-|"реалізує, Dependency Inversion"| redad
    neoad -->|"Cypher / Bolt"| neo4j
    kafad -->|"Kafka protocol / TCP: consistency.checked"| kafka
    redad -->|"RESP / TCP"| redis

    classDef ext fill:#1168bd,color:#fff,stroke:#0b4884
    classDef comp fill:#85bbf0,color:#000,stroke:#5d99c6
    class gw,kafka,neo4j,redis ext
    class jwt,acl,val,entctl,chkctl,cons,err,tel,regsvc,chksvc,rules,repoport,pubport,idemport,neoad,kafad,redad comp
```

| Компонент | Шар | Роль і взаємодія | Вимоги |
|---|---|---|---|
| JwtAuthGuard | Захист | Перевіряє підпис RS256, `exp`, `aud`; 401 при невдачі | [SEC-01](./srs.md#srs-nfr-sec-01) |
| AccessGuard | Захист | RBAC: доступ до `bookId` лише власнику або редактору (локальна копія ACL з `book.access.changed`) | [SEC-06](./srs.md#srs-nfr-sec-06), [FUN-10](./srs.md#srs-fun-10) |
| ValidationPipe | Валідація | Відхиляє невалідні DTO кодом 400 | [SEC-03](./srs.md#srs-nfr-sec-03) |
| GlobalExceptionFilter | Наскрізний | Єдиний формат помилок без внутрішніх деталей | [SEC-03](./srs.md#srs-nfr-sec-03) |
| TelemetryInterceptor | Наскрізний | JSON-журнали з `correlationId`, метрики `/metrics` | [OBS-01](./srs.md#srs-nfr-obs-01), [FUN-18](./srs.md#srs-fun-18) |
| EntityController, ConsistencyController | Inbound | REST-ендпоінти; містять лише маршрутизацію | [FUN-11, 12, 16](./srs.md#srs-fun-11) |
| ExtractedEntitiesConsumer | Inbound | Споживає `entities.extracted`; пропускає дублікати за `messageId` | [FUN-15](./srs.md#srs-fun-15), [REL-02](./srs.md#srs-nfr-rel-02) |
| EntityRegistryService | Core | Правила реєстру сутностей і хронології | [FUN-11, 12](./srs.md#srs-fun-11) |
| ConsistencyCheckService | Core | Виконує набір правил, формує звіт, публікує подію | [FUN-16](./srs.md#srs-fun-16), [PERF-04](./srs.md#srs-nfr-perf-04) |
| ConsistencyRule | Domain | Інтерфейс Strategy; реалізації: атрибути персонажа, хронологія подій, розташування дії | [FUN-16](./srs.md#srs-fun-16) |
| EntityRepository, EventPublisher, IdempotencyStore | Ports | Інтерфейси в доменному шарі; домен не залежить від драйверів | Dependency Inversion |
| Neo4jEntityRepository, KafkaEventPublisher, RedisIdempotencyStore | Outbound | Адаптери портів, інжектуються через DI | [ADR-001](./adr/0001-choice-of-technology.md) |

Напрям залежностей: Controller → Application Service → Domain (порти) ← Adapter. Контролери не мають доступу до `Neo4jEntityRepository`.

## 2.3 Документування архітектурних рішень (Architecture Decision Records)

| ADR | Назва | Статус |
|---|---|---|
| [ADR-001](./adr/0001-choice-of-technology.md) | Вибір брокера повідомлень: Apache Kafka | Approved |