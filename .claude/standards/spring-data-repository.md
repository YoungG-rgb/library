# Spring Data JPA Repository

Правила создания репозиториев.

## Расположение и именование

- Репозиторий живёт в `<feature>.repositories` своей фичи. **Не** в едином корневом пакете. См. [package-structure.md](package-structure.md).
- Имя — `<EntityName>Repository` без суффикса `Entity`: `OrderEntity` → `OrderRepository`, `ClientFunnelStateEntity` → `ClientFunnelStateRepository`.
- Интерфейс расширяет `JpaRepository<XxxEntity, IdType>`. Без `@Repository`-аннотации (Spring сам её регистрирует).

## Что должно жить в репозитории

- Методы доступа к данным — derived-queries (`findByclientCodeAndStatus`), JPQL/SQL через `@Query`.
- Простые агрегаты (`existsBy...`, `countBy...`, `maxBy...`).

**Не должно**: бизнес-логика, оркестрация, вызовы других сервисов/репозиториев, маппинг в DTO для REST. Это всё — в сервисном слое.

**Не инжектится** в контроллер напрямую. Только через сервис.

## Derived queries

Хороши для запросов из 1–3 предикатов:

```java
Optional<OrderEntity> findByclientCodeAndStatus(UUID clientCode, OrderStatus status);
```

Если имя метода становится длиннее ~60 символов или содержит 4+ предиката — переключайся на `@Query`.

## @Query: JPQL vs native

- **JPQL по умолчанию.** Привязан к JPA-модели, не ломается при ренейме колонки.
- **Native** (`nativeQuery = true`) — только если JPQL не покрывает: `FOR UPDATE SKIP LOCKED`, нативные функции Postgres, оконные функции.

```java
@Query("""
        SELECT COUNT(o) > 0 FROM OrderEntity o
        WHERE o.clientCode = :clientCode
          AND o.createdAt >= :since
        """)
boolean existsRecentOrder(@Param("clientCode") UUID clientCode,
                          @Param("since") LocalDateTime since);
```

- Текст запроса — text-block (`"""`). Однострочные `@Query` запрещены, если запрос длиннее одной строки.
- Параметры — только именованные (`:name`), позиционные (`?1`) **запрещены**.
- `@Param("name")` обязателен, даже если имя совпадает с именем параметра.

## Локи

- `@Lock(LockModeType.PESSIMISTIC_WRITE)` — для очередей и шедулеров, где параллельные воркеры тянут одни и те же строки.
- Совмещай с `FOR UPDATE SKIP LOCKED` в native-SQL.
- Метод с локом возвращает `List<...>` с `LIMIT :limit` — не тяни «всё»: блокировка живёт до конца транзакции.

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query(value = """
        SELECT * FROM outboxes
        WHERE status = 'PENDING' AND scheduled_at <= :now
        ORDER BY scheduled_at
        LIMIT :limit
        FOR UPDATE SKIP LOCKED
        """, nativeQuery = true)
List<OutboxEntity> findPendingWithLock(@Param("now") LocalDateTime now,
                                        @Param("limit") int limit);
```

## @Modifying

- Только на update/delete-запросы.
- Метод не должен возвращать `Entity` — только `int` (кол-во затронутых строк) или `void`.
- Обязательно внутри транзакции (вызывающий сервис помечен `@Transactional`).
- **Не используй `@Modifying(clearAutomatically = true)` бездумно** — перетрёт persistence-context, ломает кэш сущностей в той же транзакции.

```java
@Modifying
@Query("""
        UPDATE OutboxEntity o SET o.status = :status
        WHERE o.groupId = :groupId AND o.status = OutboxStatus.PENDING
        """)
int cancelPendingByGroupId(@Param("groupId") UUID groupId,
                            @Param("status") OutboxStatus status);
```

## Типы параметров

- Время — `LocalDateTime`. Не `Instant`, не `Date`.
- ID — `UUID` или `Long` в зависимости от сущности.
- Enum в JPQL — пиши значение enum'а напрямую (`OrderStatus.NEW`) или передавай через `@Param`.

## Возвращаемые типы

- `Optional<T>` — для запросов «найти один».
- `List<T>` — для коллекций. Никогда `null`, пустой список.
- `Page<T>` — если нужен пейджинг.
- Для агрегатов — примитивы (`long`, `boolean`) или `int` для `@Modifying`.

## Запреты

- Никаких `findAll()` без явного `Pageable` в продовом коде.
- Не возвращай сущности как DTO в API. Для проекций — отдельный интерфейс или `record`.
- Не пиши `@Transactional` на репозитории. Транзакция начинается в сервисе.

## Чек-лист перед PR

- [ ] Интерфейс расширяет `JpaRepository`, без `@Repository`.
- [ ] Все `@Query` используют именованные параметры с `@Param`.
- [ ] Многострочные запросы — в text-block (`"""`).
- [ ] Native-SQL только там, где JPQL не покрывает.
- [ ] Запросы с локами имеют `LIMIT` и `SKIP LOCKED`.
- [ ] `@Modifying`-методы возвращают `int`/`void`.
- [ ] Поиск одного — `Optional<T>`.
- [ ] Время в параметрах — `LocalDateTime`.
