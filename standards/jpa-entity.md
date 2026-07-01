# JPA Entity

Правила создания JPA-сущностей.

## Расположение и именование

- Сущность живёт в `<feature>.domain` своей фичи. **Не** в едином корневом пакете `entities/`. См. [package-structure.md](package-structure.md).
- Имя класса — `PascalCase` + суффикс `Entity`: `OrderEntity`, `UserEntity`, `ClientFunnelStateEntity`. **Почему:** суффикс однозначно отличает persistence-модель от DTO/доменной модели/проекций, особенно когда в одном пакете лежат и `Order` (DTO/record) и `OrderEntity`.
- Имя таблицы — `snake_case` во **множественном** числе через `@Table(name = "...")`: класс `OrderEntity` → таблица `orders`, `ClientFunnelStateEntity` → `client_funnel_states`. Не полагайся на Hibernate-naming-strategy: имя пиши явно.

## Структура класса

```java
@Entity
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Accessors(chain = true)
@Table(name = "orders")
@FieldDefaults(level = AccessLevel.PRIVATE)
public class OrderEntity {
    // поля
}
```

- `@Getter` / `@Setter` — Lombok, на класс. Не используй `@Data`: он добавляет `equals`/`hashCode`/`toString` по всем полям и ломает JPA-семантику identity.
- `@NoArgsConstructor` обязателен — без него Hibernate не сможет инстанцировать.
- `@AllArgsConstructor` — для удобной сборки в тестах и сервисах вместе с `@Accessors(chain = true)`.
- `@Accessors(chain = true)` — сеттеры возвращают `this`, для fluent-сборки: `new OrderEntity().setclientCode(...).setAmount(...)`. **Не используем `@Builder`.**
- `@FieldDefaults(level = AccessLevel.PRIVATE)` — поля автоматически `private`, не пиши модификатор руками. **Почему:** меньше визуального шума, единый стиль.

## Поля

### ID

- Тип ID — `Long` по умолчанию.
- Стратегия — `GenerationType.SEQUENCE` с собственной последовательностью на каждую таблицу.
- Имя последовательности = `<имя_таблицы>_seq`. Имя генератора в `@SequenceGenerator` совпадает с именем sequence. `allocationSize = 1`.

```java
@Id
@SequenceGenerator(name = "bonus_settings_seq", sequenceName = "bonus_settings_seq", allocationSize = 1)
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "bonus_settings_seq")
Long id;
```

**Почему `Long` + sequence по умолчанию, а не IDENTITY:**
- Hibernate может батчить инсерты с sequence; с `IDENTITY` каждый insert требует отдельного `RETURNING` и батчинг ломается.
- Sequence-имя предсказуемое и читаемое в скриптах/миграциях.

**Когда предложить `UUID`** (агент **предлагает** пользователю, не выставляет молча):
- ID должен быть известен на стороне Java **до flush** в БД (outbox-паттерн, составной idempotency-ключ вида `<id>:<plan>:<step>`).
- Распределённая генерация: пишут несколько инстансов/регионов без общей последовательности.
- ID светится во внешнем API/URL, и предсказуемая нумерация — утечка (можно «прокрутить» соседние записи).

Для UUID-варианта (только после согласования):
```java
@Id
@GeneratedValue(strategy = GenerationType.UUID)
UUID id;
```

### Время

- Тип — `java.time.LocalDateTime`.
- Имя поля — `xxxAt` (`createdAt`, `receivedAt`). 
- Имя колонки через `@Column(name = "xxx_at")`.

#### Поле всегда выставляется системой → `@CreationTimestamp` / `@UpdateTimestamp`

Для полей, значение которых **всегда** = момент записи в БД, и которые **никогда** не приходят извне:

```java
@CreationTimestamp
@Column(name = "created_at")
LocalDateTime createdAt;

@UpdateTimestamp
@Column(name = "updated_at")
LocalDateTime updatedAt;
```

**Когда уместно:** `createdAt`, `recordedAt`, `updatedAt`, технический `at` в аудит-логе.

#### Поле может прийти извне → `@PrePersist`-fallback

Если время приходит из внешнего контракта (event'овский `occurredAt`, `receivedAt` входящего сообщения, время из миграции), но иногда его нет — `@CreationTimestamp` **не подходит**: он **затирает** значение, выставленное извне.

```java
@PrePersist
void prePersist() {
    if (occurredAt == null) occurredAt = LocalDateTime.now();
}
```

**Когда уместно:** `occurredAt`, `receivedAt`, любые поля «время события», которое существует до того, как мы записали запись в БД.

### Колонки

- `@Column(name = "snake_case")` **всегда** — даже если совпадает с camelCase-полем.
- `nullable = false` — на все обязательные поля. Не полагайся на Hibernate-defaults.
- `unique = true` — на бизнес-ключи (`external_id`, `idempotency_key`).
- Для enum: `@Enumerated(EnumType.STRING)`. **Никогда `ORDINAL`** — переименование значения сломает БД.
- Для JSON: `@JdbcTypeCode(SqlTypes.JSON)` + `columnDefinition = "jsonb"`. Поле — `String` (JSON-строка).
- Для массивов: `@JdbcTypeCode(SqlTypes.ARRAY)` + `columnDefinition = "text[]"`. Поле — `List<String>`.

### Денежные суммы

- Тип — `BigDecimal`. Никогда `double`/`float`.
- `@Column(precision = 12, scale = 2)` или согласно требованию.

## Запреты

- **Не пиши `equals`/`hashCode` вручную.** Не нужны для работы с JPA-репозиторием по id. Если очень нужно — только по id.
- **Не клади бизнес-логику в `@PrePersist`/`@PostLoad`.** Только инициализация дефолтов.
- **Не делай сущности immutable / `record`.**
- **Не выставляй сущность наружу через REST/AMQP.** На границе — отдельные DTO.

## Чек-лист перед PR

- [ ] Имя таблицы прописано через `@Table(name = ...)`.
- [ ] Все колонки имеют `@Column(name = ...)`.
- [ ] Время — `LocalDateTime`, нет ни одного `Instant`.
- [ ] Все обязательные колонки помечены `nullable = false`.
- [ ] Уникальные бизнес-ключи имеют `unique = true` (либо составной `@Table(uniqueConstraints = ...)`).
- [ ] Enum-поля — `@Enumerated(EnumType.STRING)`.
- [ ] Нет `@Data` и нет `@Builder`. Есть `@Getter`/`@Setter`/`@NoArgsConstructor`/`@AllArgsConstructor`/`@Accessors(chain = true)`/`@FieldDefaults(level = PRIVATE)`.
- [ ] Поля без модификатора `private` (выставляет `@FieldDefaults`).
