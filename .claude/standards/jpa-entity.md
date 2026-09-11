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
- `@Accessors(chain = true)` — сеттеры возвращают `this`, для fluent-сборки: `new OrderEntity().setClientCode(...).setAmount(...)`. **Не используем `@Builder`.**
- `@FieldDefaults(level = AccessLevel.PRIVATE)` — поля автоматически `private`, не пиши модификатор руками. **Почему:** меньше визуального шума, единый стиль.

## Поля

### ID

- Тип ID — `Long` по умолчанию.
- Стратегия — `GenerationType.SEQUENCE` с собственной последовательностью на каждую таблицу.
- Имя последовательности = `<имя_таблицы>_seq`. Имя генератора в `@SequenceGenerator` совпадает с именем sequence. `allocationSize` — см. ниже.

```java
@Id
@SequenceGenerator(name = "bonus_settings_seq", sequenceName = "bonus_settings_seq", allocationSize = 50)
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "bonus_settings_seq")
Long id;
```

**Почему `Long` + sequence по умолчанию, а не IDENTITY:**
- Hibernate может батчить инсерты с sequence; с `IDENTITY` каждый insert требует отдельного `RETURNING` и батчинг ломается.
- Sequence-имя предсказуемое и читаемое в скриптах/миграциях.

**Про `allocationSize`:**
- `allocationSize = 1` требует отдельный `nextval` на каждую вставку — это **убивает** батчинг инсертов, ради которого мы и выбрали sequence вместо IDENTITY. Не ставь `1` по умолчанию.
- Ставь **pooled** `allocationSize > 1` (напр. `50`) — совпадает с `INCREMENT BY` последовательности в БД. Hibernate резервирует диапазон id одним `nextval` и раздаёт их без обращений к БД. **Почему:** батчинг работает, круговых походов к БД в разы меньше.
- Компромисс: при рестарте приложения незанятый хвост диапазона теряется → в id появляются **gap'ы**. Для суррогатного ключа это норма, не полагайся на непрерывность нумерации.
- `allocationSize = 1` допустим осознанно, только когда gap'ы недопустимы, а батчинг не нужен; тогда синхронизируй с `INCREMENT BY 1` в БД.

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

- **Выбирай тип по семантике поля, а не по привычке:**
  - `Instant` / `OffsetDateTime` — **ОК и предпочтительны** для UTC-меток аудита и машинных timestamp'ов (`createdAt`, `updatedAt`, `receivedAt`, «когда система записала»). Они хранят момент времени однозначно (timezone-safe), без привязки к локальной зоне.
  - `LocalDateTime` / `ZonedDateTime` — для **бизнес-времени**: дневные окна, расписания, DST-логика, «в котором часу по локальной зоне произошло». `ZonedDateTime` — когда важна конкретная зона и переходы DST.
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

### Оптимистичная блокировка

- Для сущностей, которые **конкурентно изменяются** (два потока/запроса читают и пишут одну строку), добавь поле `@Version` — Hibernate проверяет версию на flush и бросает `OptimisticLockException` при затирании чужого изменения (lost update).
- Тип — `Long` (или `int`), `nullable = false`.

```java
@Version
@Column(name = "version", nullable = false)
Long version;
```

- Retry-паттерн на конфликте версий (`@Retryable` + `409 Conflict` на границе) — см. скилл `jpa-patterns` (раздел «Optimistic Locking»).

## Запреты

- **Не пиши `equals`/`hashCode` вручную.** Не нужны для работы с JPA-репозиторием по id. Если очень нужно — только по id.
- **Не клади бизнес-логику в `@PrePersist`/`@PostLoad`.** Только инициализация дефолтов.
- **Не делай сущности immutable / `record`.**
- **Не выставляй сущность наружу через REST/AMQP.** На границе — отдельные DTO.

## Чек-лист перед PR

- [ ] Имя таблицы прописано через `@Table(name = ...)`.
- [ ] Все колонки имеют `@Column(name = ...)`.
- [ ] Тип времени выбран по семантике: `Instant`/`OffsetDateTime` для UTC-меток аудита, `LocalDateTime`/`ZonedDateTime` для бизнес-времени и DST.
- [ ] Все обязательные колонки помечены `nullable = false`.
- [ ] Уникальные бизнес-ключи имеют `unique = true` (либо составной `@Table(uniqueConstraints = ...)`).
- [ ] Enum-поля — `@Enumerated(EnumType.STRING)`.
- [ ] Нет `@Data` и нет `@Builder`. Есть `@Getter`/`@Setter`/`@NoArgsConstructor`/`@AllArgsConstructor`/`@Accessors(chain = true)`/`@FieldDefaults(level = PRIVATE)`.
- [ ] Поля без модификатора `private` (выставляет `@FieldDefaults`).
