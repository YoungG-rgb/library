---
name: jpa-patterns
description: JPA/Hibernate patterns and common pitfalls (N+1, lazy loading, transactions, queries). Use when user has JPA performance issues, LazyInitializationException, or asks about entity relationships and fetching strategies. Триггеры RU - «создай сущность», «новая entity», «репозиторий», «N+1», «JPA», «Hibernate», «LazyInitializationException», «FOR UPDATE», «claim».
---

## ⚠️ Project Standards Override

Если в проекте есть `.claude/standards/` — следуй им. Эти файлы переопределяют generic-примеры ниже:

- [`.claude/standards/jpa-entity.md`](.claude/standards/jpa-entity.md) — обязательная структура сущности.
- [`.claude/standards/spring-data-repository.md`](.claude/standards/spring-data-repository.md) — правила репозиториев.
- [`.claude/standards/service-transactional.md`](.claude/standards/service-transactional.md) — где ставить `@Transactional`.

Ключевые расхождения generic-примеров со стандартами проекта:

**Entity**:
- `@Id` — `GenerationType.SEQUENCE` с явной `@SequenceGenerator(name="<table>_seq", sequenceName="<table>_seq", allocationSize=1)`. **Не** `IDENTITY` (ломает Hibernate-батчинг).
- Lombok: `@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Accessors(chain = true) @FieldDefaults(level = PRIVATE)`. **Никогда `@Data` / `@Builder` / `record`.**
- `@Table(name = "...")` + `@Column(name = "snake_case")` всегда явно. Множественное число для таблиц.
- Время — `LocalDateTime`, не `Instant`. `@CreationTimestamp` / `@UpdateTimestamp` для системных полей; `@PrePersist`-fallback для полей, приходящих извне (`occurredAt`, `receivedAt`).
- Enum — `@Enumerated(EnumType.STRING)`. **Никогда `ORDINAL`.**
- Деньги — `BigDecimal` с `precision`/`scale`. Никогда `double`/`float`.
- `nullable = false` на все обязательные. `unique = true` на бизнес-ключи / `idempotency_key`.
- JSON — `@JdbcTypeCode(SqlTypes.JSON)` + `columnDefinition = "jsonb"`, поле `String`.
- Не пиши `equals`/`hashCode` вручную. Не выставляй сущность через REST/AMQP.

**Repository**:
- Размещение — `<feature>.repositories`, не корневой пакет. Имя — `<EntityName>Repository` без суффикса `Entity`.
- Не `@Repository`-аннотация (Spring сам). Только `extends JpaRepository<...>`.
- Запросы длиннее одной строки — text-block (`"""`). Только именованные `@Param`, никаких `?1`.
- JPQL по умолчанию. Native — только если JPQL не покрывает (`FOR UPDATE SKIP LOCKED`, оконные функции).
- Локи: `@Lock(LockModeType.PESSIMISTIC_WRITE)` + `LIMIT :limit` + `FOR UPDATE SKIP LOCKED`.
- `@Modifying` — возвращает `int`/`void`, не сущность. Без `clearAutomatically = true` по умолчанию.
- `Optional<T>` для поиска одного, `List<T>` для коллекций (никогда `null`).

**Claim+dispatch для шедулеров** (см. `.claude/standards/scheduler.md`):
- Атомарный claim через CTE: `WITH locked AS (SELECT ... FOR UPDATE SKIP LOCKED LIMIT :batch) UPDATE ... SET status = 'IN_PROGRESS' FROM locked WHERE ... RETURNING *`.
- Возвращает детачнутые сущности (`clearAutomatically = true` уместен здесь).
- Per-row работа — в `@Component @Scope(SCOPE_PROTOTYPE)` Task-бине, обёрнутом в `TransactionTemplate.executeWithoutResult(...)`.

**Идемпотентность денежных операций**: UNIQUE-индекс + стабильный `external_id` + pre-check под row-lock. Тройная защита.

---

# JPA Patterns Skill

Best practices and common pitfalls for JPA/Hibernate in Spring applications.

## When to Use
- User mentions "N+1 problem" / "too many queries"
- LazyInitializationException errors
- Questions about fetch strategies (EAGER vs LAZY)
- Transaction management issues
- Entity relationship design
- Query optimization

---

## Quick Reference: Common Problems

| Problem                     | Symptom                   | Solution                                         |
|-----------------------------|---------------------------|--------------------------------------------------|
| N+1 queries                 | Many SELECT statements    | JOIN FETCH, @EntityGraph                         |
| LazyInitializationException | Error outside transaction | Open Session in View, DTO projection, JOIN FETCH |
| Slow queries                | Performance issues        | Pagination, projections, indexes                 |
| Dirty checking overhead     | Slow updates              | Read-only transactions, DTOs                     |
| Lost updates                | Concurrent modifications  | Optimistic locking (@Version)                    |

---

## N+1 Problem

> The #1 JPA performance killer

### The Problem

```java
// ❌ BAD: N+1 queries
@Entity
public class Author {
    @Id private Long id;
    private String name;

    @OneToMany(mappedBy = "author", fetch = FetchType.LAZY)
    private List<Book> books;
}

// This innocent code...
List<Author> authors = authorRepository.findAll();  // 1 query
for (Author author : authors) {
    System.out.println(author.getBooks().size());   // N queries!
}
// Result: 1 + N queries (if 100 authors = 101 queries)
```

### Solution 1: JOIN FETCH (JPQL)

```java
// ✅ GOOD: Single query with JOIN FETCH
public interface AuthorRepository extends JpaRepository<Author, Long> {

    @Query("SELECT a FROM Author a JOIN FETCH a.books")
    List<Author> findAllWithBooks();
}

// Usage - single query
List<Author> authors = authorRepository.findAllWithBooks();
```

### Solution 2: @EntityGraph

```java
// ✅ GOOD: EntityGraph for declarative fetching
public interface AuthorRepository extends JpaRepository<Author, Long> {

    @EntityGraph(attributePaths = {"books"})
    List<Author> findAll();

    // Or with named graph
    @EntityGraph(value = "Author.withBooks")
    List<Author> findAllWithBooks();
}

// Define named graph on entity
@Entity
@NamedEntityGraph(
    name = "Author.withBooks",
    attributeNodes = @NamedAttributeNode("books")
)
public class Author {
    // ...
}
```

### Solution 3: Batch Fetching

```java
// ✅ GOOD: Batch fetching (Hibernate-specific)
@Entity
public class Author {

    @OneToMany(mappedBy = "author")
    @BatchSize(size = 25)  // Fetch 25 at a time
    private List<Book> books;
}

// Or globally in application.properties
spring.jpa.properties.hibernate.default_batch_fetch_size=25
```

### Detecting N+1

```yaml
# Enable SQL logging to detect N+1
spring:
  jpa:
    show-sql: true
    properties:
      hibernate:
        format_sql: true

logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.type.descriptor.sql.BasicBinder: TRACE
```

---

## Lazy Loading

### FetchType Basics

```java
@Entity
public class Order {

    // LAZY: Load only when accessed (default for collections)
    @OneToMany(mappedBy = "order", fetch = FetchType.LAZY)
    private List<OrderItem> items;

    // EAGER: Always load immediately (default for @ManyToOne, @OneToOne)
    @ManyToOne(fetch = FetchType.EAGER)  // ⚠️ Usually bad
    private Customer customer;
}
```

### Best Practice: Default to LAZY

```java
// ✅ GOOD: Always use LAZY, fetch when needed
@Entity
public class Order {

    @ManyToOne(fetch = FetchType.LAZY)  // Override EAGER default
    private Customer customer;

    @OneToMany(mappedBy = "order", fetch = FetchType.LAZY)
    private List<OrderItem> items;
}
```

### LazyInitializationException

```java
// ❌ BAD: Accessing lazy field outside transaction
@Service
public class OrderService {

    public Order getOrder(Long id) {
        return orderRepository.findById(id).orElseThrow();
    }
}

// In controller (no transaction)
Order order = orderService.getOrder(1L);
order.getItems().size();  // 💥 LazyInitializationException!
```

### Solutions for LazyInitializationException

**Solution 1: JOIN FETCH in query**
```java
// ✅ Fetch needed associations in query
@Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.id = :id")
Optional<Order> findByIdWithItems(@Param("id") Long id);
```

**Solution 2: @Transactional on service method**
```java
// ✅ Keep transaction open while accessing
@Service
public class OrderService {

    public OrderDTO getOrderWithItems(Long id) {
        Order order = orderRepository.findById(id).orElseThrow();
        // Access within transaction — lazy collection initialized here
        int itemCount = order.getItems().size();
        return OrderDTO.from(order, itemCount);
    }
}
```

**Solution 3: Explicit fetch by FK (aggregate split) — preferred for read paths**

Не навигируй по lazy-коллекции вообще. Фетчим `Order` сам по себе, а `items`
грузим отдельным явным запросом по `orderId` — только когда они реально нужны.

```java
// ✅ Fetch Order alone; load items separately, on demand
public OrderDTO getOrder(Long id) {
    Order order = orderRepository.findById(id).orElseThrow();
    List<OrderItem> items = orderItemRepository.findByOrderId(id);
    return OrderDTO.from(order, items);
}
```

⚠️ **Для СПИСКА заказов это снова N+1**, если звать `findByOrderId` в цикле.
Используй один `...In(...)`-запрос и группируй в памяти:

```java
// ✅ One query for all items, then group by orderId
List<Long> orderIds = orders.stream().map(Order::getId).toList();
Map<Long, List<OrderItem>> itemsByOrder = orderItemRepository.findByOrderIdIn(orderIds)
    .stream()
    .collect(Collectors.groupingBy(OrderItem::getOrderId));
```

> **Важно:** это про *чтение*. `@OneToMany` на сущности всё ещё нужен, если
> у тебя каскадное сохранение / orphan removal — для записи связь остаётся.

### When to use what

| Ситуация                                        | Решение                          |
|-------------------------------------------------|----------------------------------|
| items нужны почти всегда, один Order / страница | **JOIN FETCH** (один round-trip) |
| items нужны иногда, разные пути чтения          | **Explicit fetch by FK** (Sol. 3)|
| Список Order'ов + их items                      | **`findByOrderIdIn(...)`** — никогда `findByOrderId` в цикле |
| Нужна навигация по графу внутри одной операции  | **@Transactional(readOnly)** (Sol. 2) |

---

## Projections & Pagination

> Не тащи всю сущность, если нужно 3 поля. Не грузи всю таблицу, если нужна страница.

### DTO Projections (read-only)

```java
// ❌ Fetch full entity graph just to render a list row
List<Order> orders = orderRepository.findAll();

// ✅ Interface projection — Hibernate selects only these columns
public interface OrderSummary {
    Long getId();
    String getStatus();
    BigDecimal getTotal();
}

public interface OrderRepository extends JpaRepository<Order, Long> {
    List<OrderSummary> findByStatus(String status);
}

// ✅ Constructor (DTO) projection via JPQL — explicit and refactor-safe
@Query("""
       SELECT new com.example.order.dto.OrderSummaryDto(o.id, o.status, o.total)
       FROM Order o
       WHERE o.status = :status
       """)
List<OrderSummaryDto> findSummaries(@Param("status") String status);
```

Плюс проекций: не грузится весь граф, нет dirty-checking (это read-only данные,
не managed-сущности), меньше памяти и трафика к БД.

### Pagination

```java
// ❌ Может вернуть миллионы строк
List<Order> all = orderRepository.findAll();

// ✅ Страница
Page<Order> page = orderRepository.findAll(PageRequest.of(0, 20, Sort.by("id")));
Slice<Order> slice = orderRepository.findByStatus("NEW", PageRequest.of(0, 20));
```

⚠️ **`Pageable` + `JOIN FETCH` коллекции = пагинация в памяти.** Hibernate
подтянет ВСЁ и порежет страницу в Java (в логах — `HHH000104: firstResult/maxResults
specified with collection fetch; applying in memory`). Для страницы с коллекцией:
пагинируй по корню (id), затем добери коллекции отдельным `...In(...)`-запросом
(см. Solution 3 выше), либо используй `@EntityGraph` с `@ManyToOne`, а не `@OneToMany`.

---

## Read-only Transactions & Dirty Checking

Для read-путей всегда `@Transactional(readOnly = true)`:

```java
// ✅ readOnly = true
@Transactional(readOnly = true)
public List<OrderSummaryDto> listNew() {
    return orderRepository.findSummaries("NEW");
}
```

Что это даёт:
- **Нет dirty-checking snapshot'ов** — Hibernate не хранит копию каждой сущности
  для сравнения на flush → меньше памяти и CPU на больших выборках.
- **`FlushMode.MANUAL`** — не будет случайного `UPDATE` при чтении.
- Подсказка драйверу/реплике, что транзакция только читает (роутинг на read-replica).

```java
// ❌ Молчаливый UPDATE: изменил managed-сущность в read-методе без readOnly
public Order getOrder(Long id) {
    Order o = orderRepository.findById(id).orElseThrow();
    o.setViewedAt(LocalDateTime.now()); // dirty checking → UPDATE на flush!
    return o;
}
```

---

## Optimistic Locking (Lost Updates)

Два потока читают заказ, оба меняют, второй затирает первого — **lost update**.
Защита — `@Version`:

```java
// ✅ Version column
@Entity
@Table(name = "orders")
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "orders_seq")
    @SequenceGenerator(name = "orders_seq", sequenceName = "orders_seq", allocationSize = 1)
    private Long id;

    @Version
    @Column(name = "version", nullable = false)
    private Long version;
}
```

Hibernate добавит `... WHERE id = ? AND version = ?` и бросит
`OptimisticLockException` / `ObjectOptimisticLockingFailureException`, если версия
уже сменилась. Обработай на границе — верни `409 Conflict` и дай клиенту повторить.

```java
// ✅ Retry на конфликте версий
@Retryable(retryFor = ObjectOptimisticLockingFailureException.class,
           maxAttempts = 3, backoff = @Backoff(delay = 50))
@Transactional
public void applyDiscount(Long id, BigDecimal pct) { ... }
```

### Optimistic vs Pessimistic

| | Optimistic (`@Version`) | Pessimistic (`FOR UPDATE`) |
|---|---|---|
| Когда | Конфликты редки | Конфликты частые / деньги / claim |
| Стоимость | Дёшево, без блокировок в БД | Держит row-lock всю транзакцию |
| Провал | Исключение на commit → retry | Ждёт лок / таймаут |

> **Для шедулеров и денежных claim'ов** проект использует pessimistic-путь
> (`FOR UPDATE SKIP LOCKED` + idempotency-key) — см. блок «Project Standards
> Override» и `.claude/standards/scheduler.md`.

---

## Quick Checklist перед PR

- [ ] Нет навигации по lazy-коллекции вне транзакции (или сознательный Solution 1/2/3)
- [ ] Нет `findByX` в цикле по списку — только `...In(...)` + группировка
- [ ] Read-методы помечены `@Transactional(readOnly = true)`
- [ ] Списки отдаются проекцией/DTO, а не полной сущностью
- [ ] Выборки, способные вырасти, — через `Pageable`
- [ ] Нет `Pageable` + `JOIN FETCH` коллекции (пагинация в памяти)
- [ ] Конкурентно изменяемые сущности имеют `@Version`
- [ ] `OptimisticLockException` обработан (409 / retry)
