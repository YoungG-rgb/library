# Package Structure

Гибрид: **feature на верхнем уровне, layer внутри**.

## Общая схема

```
<root>
├── <feature-a>
│   ├── communications ← внешние точки входа (REST, AMQP, Kafka, gRPC)
│   ├── domain         ← entities, enums, value objects, доменные модели
│   ├── repositories   ← Spring Data репозитории
│   ├── services       ← интерфейсы + impl/ (бизнес-логика)
│   └── config         ← @Configuration, специфичный для фичи
├── <feature-b>
│   └── ...
├── shared           ← реально кросс-фичевое
│   ├── config       ← глобальные @Configuration (Jackson, OpenApi, Security)
│   ├── events       ← общие event-контракты (если внешний contract один на всех)
│   ├── utilities    ← маскеры, утилиты и т д
│   └── ...
└── Application.java ← @SpringBootApplication
```

## Уровень 1: фичи

- **Фича = bounded context**, одна бизнес-возможность. Имя — существительное в единственном числе (`order`, `payment`, `notification`, `funnel`).
- Имя фичи на латинице, в нижнем регистре. Никаких camelCase, никаких суффиксов (`orderService` ← плохо как имя пакета).
- Если две «фичи» постоянно меняются вместе и зависят друг от друга в обе стороны — это **одна** фича, объедини.
- Корневой класс приложения (`@SpringBootApplication`) — в корне, на одном уровне с фичами. **Почему:** Spring компонент-скан по умолчанию подхватит все вложенные пакеты.

## Уровень 2: слои внутри фичи

Канонические подпакеты — `communications`, `domain`, `repositories`, `services`, `config`. Дополнительные подпакеты добавляй **только когда в существующем становится >5 файлов** или появляется новая концептуальная группа.

### `communications`
- REST-контроллеры, AMQP-консьюмеры, Kafka-листенеры, gRPC-эндпоинты, scheduled-таски (входные триггеры), HTTP-клиенты к внешним системам.
- DTO запросов/ответов **этой** фичи — в `communications.dto` (если файлов > 3) или прямо в `communications`.
- Не вызывает напрямую `repositories` других фич. Не импортирует `services.impl` других фич.
- Внутри уместно дробить по транспорту: `communications.amqp`, `communications.rest`, `communications.kafka`.

### `domain`
- JPA-сущности (`XxxEntity`), enum'ы домена, value objects, доменные exceptions.
- **Никаких** Spring-зависимостей: ни `@Service`, ни `@Component`, ни `@Autowired`.
- Чистая Java + JPA-аннотации + Lombok.

### `repositories`
- Spring Data репозитории (`extends JpaRepository`).
- Native/JPQL-запросы.
- Кастомные `*RepositoryImpl` для сложных запросов — рядом.

### `services`
- Интерфейсы сервисов на верхнем уровне пакета (`OrderService`).
- Имплементации — в подпакете `impl/` (`OrderServiceImpl`).
- Внутренние DTO между сервисами (если не доменные) — в `services.dto` (если файлов > 3).

### `config`
- `@Configuration`-классы, специфичные для фичи: AMQP exchange/queue для этой фичи, специфичные `RestClient`-бины, `@ConfigurationProperties` фичи.
- Глобальные конфиги (Jackson, Security, общий ObjectMapper) — в `shared.config`, **не** здесь.

### Дополнительные подпакеты (по необходимости)
- `mapper` — если маппинг entity↔dto разрастается.
- `event` / `listener` — внутренние domain events (Spring `ApplicationEvent`).
- `scheduler` — `@Scheduled`-таски, если их несколько.

## Уровень 2: `shared`

Для **реально** кросс-фичевого. Не используй как помойку.

- Канонические подпакеты — `config`, `events`, `utilities`. Доп. дроби по концерну: `shared.exception`, `shared.annotation`.
- `shared.utilities` — единственное санкционированное место для stateless-утилит (маскеры, time helpers, string utils). Внутри допустимо плоско, но классы — **по одному концерну на файл**, не «`Helpers` на 800 строк».
- Контракты внешних входящих событий (если они одни на несколько фич) — `shared.events.<source>` (например `shared.events.ow4`).
- Если что-то нужно ровно одной фиче — это **не** shared. Перенеси.
- **Запрещены** альтернативные имена-помойки на уровне `shared`: `common`, `commons`, `utils`, `helpers`, `core`, `base`. Используй `utilities`.

## Правила зависимостей

```
communications ──→ services ──→ repositories ──→ domain
                                      │
                                      └─────────→ domain
config — самостоятельный, может зависеть от чего угодно внутри фичи
```

- `domain` ни от кого не зависит (кроме `shared`).
- `repositories` зависит от `domain`.
- `services` зависит от `repositories` и `domain`.
- `communications` зависит от `services` и `domain` (для DTO-маппинга).
- **Обратные зависимости запрещены**: `domain` не импортирует `services`, `repositories` не импортирует `communications`.

### Между фичами

- Фича `A` **не импортирует** `B.repositories`, `B.domain`, `B.services.impl` напрямую.
- Допустимые способы коммуникации:
  1. **Через `B.services` интерфейс** — `A` инжектит `BService`, `B` публикует контракт.
  2. **Через domain events** — `A` публикует `ApplicationEvent`, `B` слушает.
  3. **Через outbox + асинхронный канал** — для слабосвязанных доменов.
- Если две фичи постоянно ходят друг в друга — пересмотри границы, возможно это одна фича.

### Зависимости на `shared`

- Любая фича может зависеть от `shared`.
- `shared` **не** зависит от фич. Никогда.

## Запреты

- **`common` / `commons` / `utils` / `core` / `base` как корневой пакет** — нет. Только `shared` с разрешёнными подпакетами.
- **Один пакет на все entities проекта** (`<root>.entities`) — нет. Entity живёт в `<feature>.domain` своей фичи.
- **Один пакет на все repositories проекта** — нет. Репозиторий живёт в `<feature>.repositories` своей фичи.
- **`controllers` / `consumers` как корневые пакеты** — нет. Они живут внутри фичи в `<feature>.communications`.
- **Циклы между фичами** — нет. Если зависимость нужна в обе стороны, либо это одна фича, либо коммуникация через event/outbox.
- **`shared.dto` как свалка DTO** — нет. DTO принадлежит фиче.

## Когда отступать от стандарта

- **Очень маленький сервис (< 15 классов)** — package-by-layer окей, выделение фич — оверхед. Зафиксируй размер в проектном `CLAUDE.md`.
- **Готовишь распил на микросервисы** — поверх этой схемы поставь Spring Modulith / ArchUnit с публичным `communications`-пакетом и `internal` package-private. Это надстройка, не противоречие.

## Чек-лист перед PR

- [ ] Новый класс лежит в фиче, а не в корневых техпакетах (`entities/`, `services/`, `controllers/`).
- [ ] Имя фичи — существительное в единственном числе, в нижнем регистре.
- [ ] Внутри фичи использованы канонические слои (`communications`/`domain`/`repositories`/`services`/`config`).
- [ ] `domain` не импортирует Spring-аннотации (`@Service`, `@Component`, `@Autowired`).
- [ ] Нет импортов из `B.repositories` / `B.domain` / `B.services.impl` другой фичи.
- [ ] Нет цикла между фичами.
- [ ] Если положил в `shared` — это реально нужно ≥ 2 фичам.
- [ ] В `shared` использованы только разрешённые имена (`config`, `events`, `utilities`, `exception`, `annotation`). Никаких `common`/`utils`/`helpers`/`core`.
- [ ] `@Configuration` фичи лежит в `<feature>.config`, глобальный — в `shared.config`.
