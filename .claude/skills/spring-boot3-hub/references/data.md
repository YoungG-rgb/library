# Data Access - Spring Data JPA

> ⚠️ Правила сущностей, репозиториев и транзакций живут в одном месте — НЕ дублируем здесь:
> - **Сущности** (SEQUENCE, Lombok-набор, суффикс `Entity`, время `LocalDateTime`) → `.claude/standards/jpa-entity.md`
> - **Репозитории** (размещение, именованные `@Param`, JPQL/native, locking) → `.claude/standards/spring-data-repository.md`
> - **Транзакции** (`@Transactional`, propagation, self-invocation) → `.claude/standards/service-transactional.md`
> - **N+1 / lazy / projections / pagination / optimistic locking** → скилл `jpa-patterns`
>
> Ниже — только то, чего нет в стандартах и `jpa-patterns`: Specifications, аудит, миграции.

## Repository with Specifications

Динамические предикаты через Criteria API — когда набор фильтров зависит от рантайма.

```java
public class UserSpecifications {

    public static Specification<UserEntity> hasEmail(String email) {
        return (root, query, cb) ->
            email == null ? null : cb.equal(root.get("email"), email);
    }

    public static Specification<UserEntity> isActive() {
        return (root, query, cb) -> cb.isTrue(root.get("active"));
    }

    public static Specification<UserEntity> createdAfter(LocalDateTime date) {
        return (root, query, cb) ->
            date == null ? null : cb.greaterThanOrEqualTo(root.get("createdAt"), date);
    }

    public static Specification<UserEntity> hasRole(String roleName) {
        return (root, query, cb) -> {
            Join<UserEntity, RoleEntity> roles = root.join("roles", JoinType.INNER);
            return cb.equal(roles.get("name"), roleName);
        };
    }
}

// Usage in service (репозиторий extends JpaSpecificationExecutor<UserEntity>)
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class UserSearchService {
    private final UserRepository userRepository;

    public Page<UserEntity> searchUsers(UserSearchCriteria criteria, Pageable pageable) {
        Specification<UserEntity> spec = Specification
            .where(UserSpecifications.hasEmail(criteria.email()))
            .and(UserSpecifications.isActive())
            .and(UserSpecifications.createdAfter(criteria.createdAfter()));

        return userRepository.findAll(spec, pageable);
    }
}
```

## Auditing Configuration

Автозаполнение `createdBy`/`updatedBy` через `@EnableJpaAuditing` + `AuditorAware`.

```java
@Configuration
@EnableJpaAuditing
public class JpaAuditingConfig {

    @Bean
    public AuditorAware<String> auditorProvider() {
        return () -> {
            Authentication authentication = SecurityContextHolder
                .getContext()
                .getAuthentication();

            if (authentication == null || !authentication.isAuthenticated()) {
                return Optional.of("system");
            }
            return Optional.of(authentication.getName());
        };
    }
}

@Getter
@Setter
@MappedSuperclass
@FieldDefaults(level = AccessLevel.PRIVATE)
@EntityListeners(AuditingEntityListener.class)
public abstract class AuditableEntity {

    @CreatedDate
    @Column(nullable = false, updatable = false)
    LocalDateTime createdAt;

    @CreatedBy
    @Column(nullable = false, updatable = false, length = 100)
    String createdBy;

    @LastModifiedDate
    @Column(nullable = false)
    LocalDateTime updatedAt;

    @LastModifiedBy
    @Column(nullable = false, length = 100)
    String updatedBy;
}
```

> Системные `@CreationTimestamp`/`@UpdateTimestamp` vs аудит-поля из БД — см. `jpa-entity.md`.

## Database Migrations (Flyway)

Схема — только через версионированные миграции, не через `ddl-auto`. Имя таблицы —
`snake_case` во множественном числе (см. `jpa-entity.md`).

```sql
-- V1__create_users_table.sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY,                          -- SEQUENCE-генерация на стороне приложения
    email VARCHAR(100) NOT NULL UNIQUE,
    password VARCHAR(100) NOT NULL,
    username VARCHAR(50) NOT NULL UNIQUE,
    active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    version BIGINT NOT NULL DEFAULT 0
);
CREATE SEQUENCE users_seq START WITH 1 INCREMENT BY 1;  -- под @SequenceGenerator(allocationSize = 1)

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_active ON users(active);

-- V2__create_addresses_table.sql
CREATE TABLE addresses (
    id BIGINT PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    street VARCHAR(200) NOT NULL,
    city VARCHAR(100) NOT NULL,
    country VARCHAR(2) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE SEQUENCE addresses_seq START WITH 1 INCREMENT BY 1;

CREATE INDEX idx_addresses_user_id ON addresses(user_id);
```

## Quick Reference

| Annotation               | Purpose                               |
|--------------------------|---------------------------------------|
| `@Entity`                | Marks class as JPA entity             |
| `@Table`                 | Specifies table details and indexes   |
| `@Id`                    | Marks primary key field               |
| `@GeneratedValue`        | Auto-generated primary key strategy   |
| `@Column`                | Column constraints and mapping        |
| `@OneToMany/@ManyToOne`  | One-to-many/many-to-one relationships |
| `@ManyToMany`            | Many-to-many relationships            |
| `@JoinColumn/@JoinTable` | Join column/table configuration       |
| `@Query`                 | Custom JPQL/native queries            |
| `@Modifying`             | Marks query as UPDATE/DELETE          |
| `@EntityGraph`           | Defines fetch graph for associations  |
| `@Version`               | Optimistic locking version field      |
| `@MappedSuperclass`      | Shared mapped fields (e.g. audit base)|
