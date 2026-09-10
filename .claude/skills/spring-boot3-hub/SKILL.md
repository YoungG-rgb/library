---
name: spring-boot3-hub
description: Точка входа (hub) для разработки на Spring Boot 3.x - маршрутизирует к специализированным скиллам (JPA, логирование, конкурентность, паттерны, ревью), к проектным стандартам и к собственным reference-файлам (web/data/security/cloud/testing), задаёт общий workflow, слои и сквозные ограничения. Только Spring Boot 3+. Use for Spring Boot app structure, wiring, REST endpoints, security, config. Триггеры RU - «добавь контроллер», «создай сервис», «новый эндпоинт», «настрой Spring», «структура проекта», «Spring Boot 3».
metadata:
  version: "3.0.0"
  domain: backend
  role: hub (orchestrator + references)
  scope: routing + cross-cutting
  output-format: routing
---

## ⚠️ Project Standards Override

Если в проекте есть каталог `.claude/standards/` — эти файлы являются истиной в последней инстанции и ПЕРЕОПРЕДЕЛЯЮТ любые generic-рекомендации при конфликте. Читай их ПЕРЕД работой:

| Тема                                                     | Файл                                           |
|----------------------------------------------------------|------------------------------------------------|
| Структура пакетов (feature × layer)                      | `.claude/standards/package-structure.md`       |
| JPA-сущности                                             | `.claude/standards/jpa-entity.md`              |
| Spring Data репозитории                                  | `.claude/standards/spring-data-repository.md`  |
| Сервисы и `@Transactional`                               | `.claude/standards/service-transactional.md`   |
| Шедулеры (`@Scheduled`, virtual threads, claim+dispatch) | `.claude/standards/scheduler.md`               |
| Конфигурация (`application.yml`, профили, секреты)       | `.claude/standards/application-config.md`      |
| Корреляция и трейсинг (`X-Request-Id`, MDC, downstream)  | `.claude/standards/correlation-and-tracing.md` |

**CLAUDE.md** проекта — всегда главнее всего. Прочитай его перед структурными изменениями.

Если `.claude/standards/` отсутствует — опирайся на специализированные скиллы ниже как fallback.

---

# Spring Boot 3 Hub

Это **не** склад примеров, а точка входа (только Spring Boot 3+). Задача скилла — понять область запроса,
делегировать её нужному специализированному скиллу / стандарту / reference-файлу и
проследить за сквозными ограничениями. Глубину не дублируем — она живёт в одном месте.

## Куда делегировать (routing map)

| Область запроса                                                 | Специализированный скилл | Проектный стандарт                                                       | Reference                |
|-----------------------------------------------------------------|--------------------------|--------------------------------------------------------------------------|--------------------------|
| JPA: сущности, репозитории, N+1, lazy, транзакции, locking      | **`jpa-patterns`**       | `jpa-entity.md`, `spring-data-repository.md`, `service-transactional.md` | `references/data.md`     |
| Логи, MDC, structured logging, PII-маскирование                 | **`logging-patterns`**   | `service-transactional.md` (MDC), `correlation-and-tracing.md`           | —                        |
| Корреляция / трейсинг / проброс к downstream                    | **`logging-patterns`**   | `correlation-and-tracing.md`                                             | `references/web.md`      |
| Async, потоки, `@Async`, `CompletableFuture`, virtual threads   | **`concurrency-review`** | `scheduler.md`                                                           | —                        |
| Шедулеры, claim+dispatch, батчи                                 | —                        | `scheduler.md`                                                           | —                        |
| Design patterns, typed-handler registry, factory                | **`design-patterns`**    | `service-transactional.md` (registry)                                    | —                        |
| Код-ревью, чистота, API-контракты, null-safety                  | **`code-quality`**       | все применимые                                                           | —                        |
| Web/REST: контроллеры, валидация, exception handling, WebClient | —                        | —                                                                        | `references/web.md`      |
| Security: Spring Security 6, OAuth2, JWT                        | —                        | —                                                                        | `references/security.md` |
| Cloud/Config: config server, discovery, resilience              | —                        | `application-config.md`                                                  | `references/cloud.md`    |
| Тесты: unit, slice, integration, TestContainers                 | —                        | —                                                                        | `references/testing.md`  |

**Правило маршрутизации:** если для области есть специализированный скилл — приоритет у него; этот скилл лишь связывает области и следит за сквозными правилами.

**Порядок приоритета при конфликте:** CLAUDE.md > стандарт > специализированный скилл > reference > generic.

## Core Workflow

1. **Analyze** — требования, границы сервисов, API, модель данных.
2. **Design** — архитектура; подтверди дизайн до кода.
3. **Route** — определи области (см. routing map) и открой нужные скиллы/стандарты.
4. **Implement** — конструкторная инъекция, слоистость, стандарты проекта.
5. **Secure** — Spring Security / method security; тесты зелёные.
6. **Test** — unit + slice + integration; `./mvnw test` проходит.
7. **Verify** — Actuator health, `/actuator/health` = UP.

## Package Structure & Layering

Раскладка — **feature × layer** (см. `package-structure.md`), НЕ корневые `controller/service/repository/model`.

```
<feature>/
├── communications/   # REST-контроллеры, consumers, WebClient-коннекторы
├── domain/           # сущности, DTO, enum
├── repositories/     # Spring Data репозитории
├── services/         # бизнес-логика (+ impl/)
└── config/           # конфигурация фичи
```

- Поток зависимостей: communications → services → repositories → domain.
- Контроллер — только HTTP/валидация; сервис — бизнес-логика/транзакции; репозиторий — доступ к данным.
- Domain-модель независима от фреймворков.

## Cross-cutting Constraints

Действуют всегда, поверх любой области.

### MUST DO
- Конструкторная инъекция (`@RequiredArgsConstructor` + `private final`).
- `@Valid` на всех request-body.
- `@Transactional` на multi-step записи; `@Transactional(readOnly = true)` на чтения.
- Type-safe конфиг через `@ConfigurationProperties`.
- Глобальная обработка ошибок через `@RestControllerAdvice`.
- Секреты — из env, не из properties-файлов (см. `application-config.md`).

### MUST NOT DO
- Field injection (`@Autowired` на полях) — кроме prototype-Task (см. `scheduler.md`).
- Пропуск валидации на эндпоинтах.
- Мешать блокирующий и реактивный код в одном пути.
- Секреты в `application.properties` / хардкод URL/кредов.
- Deprecated-паттерны Spring Boot 2.x.
- PII (msisdn/PAN/email) в логах или MDC в исходном виде (см. `logging-patterns`).

## Common Annotations

| Annotation                 | Purpose                                           |
|----------------------------|---------------------------------------------------|
| `@RestController`          | REST-контроллер (`@Controller` + `@ResponseBody`) |
| `@Service`                 | Бизнес-логика (на impl)                           |
| `@Transactional`           | Транзакционные границы (сервисный слой)           |
| `@Valid`                   | Триггер валидации                                 |
| `@ConfigurationProperties` | Type-safe биндинг настроек                        |
| `@RestControllerAdvice`    | Глобальная обработка ошибок                       |
| `@EnableMethodSecurity`    | Method-level security                             |

## Reference Guide (progressive disclosure)

Подробные паттерны — в reference-файлах этого скилла; подгружай по контексту:

| Topic                | Reference                | When to Load                                               |
|----------------------|--------------------------|------------------------------------------------------------|
| Web/REST + WebClient | `references/web.md`      | Контроллеры, валидация, exception handling, внешние вызовы |
| Data Access          | `references/data.md`     | JPA, репозитории, транзакции, запросы                      |
| Security             | `references/security.md` | Spring Security 6, OAuth2, JWT                             |
| Cloud/Config         | `references/cloud.md`    | Config server, discovery, resilience                       |
| Testing              | `references/testing.md`  | Unit, integration, slice-тесты                             |

## Knowledge Base

Spring Boot 3.x, Java 17+, Spring WebFlux, Project Reactor, Spring Data JPA, Spring Security 6, OAuth2/JWT, Hibernate, R2DBC, Spring Cloud, Resilience4j, Micrometer, JUnit 5, TestContainers, Mockito, Maven/Gradle.
