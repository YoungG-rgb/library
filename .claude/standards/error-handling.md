# Error Handling

Правила обработки ошибок в REST: формат ответа, доменные исключения, реестр типов,
вынос в общий starter. Основа — RFC 9457 (`ProblemDetail`), встроенный в Spring Boot 3.

## Формат ответа

- Новый код — **только `ProblemDetail`** (`application/problem+json`, RFC 9457). Не изобретай свой JSON-формат ошибок.
  - **Почему:** стандарт, встроен в Boot 3, понимается клиентами/gateway/инструментами; поле `type` даёт машинную маршрутизацию.
- В Boot 3 `@ExceptionHandler` **возвращает `ProblemDetail` напрямую** — Spring сам ставит content-type.
- Самописный плоский формат (`{status,message,path}`) — **только legacy**, ради обратной совместимости старых потребителей. Класс называй `ApiError`, **не** `ErrorResponse` (коллизия с `org.springframework.web.ErrorResponse`).
- Встроенные исключения Spring (404 на роут, 405, 415…) — включи `spring.mvc.problemdetails.enabled=true` (WebFlux: `spring.webflux.problemdetails.enabled`), тонкую настройку — через `extends ResponseEntityExceptionHandler`.

## Поля ProblemDetail

- Стандартные: `type` (URI типа), `title`, `status`, `detail` (текст случая), `instance` (URI возникновения).
- Всё остальное — extension через `setProperty(...)` (`errors`, `requestId`, доменные id).
- **`detail` для 5xx — generic** («Internal error»). Стектрейс, SQL, имена ограничений — **только в лог**, не в ответ.
  - **Почему:** тело ошибки уходит клиенту; внутренности — утечка (см. запрет PII/секретов).
- **PII/секреты не клади** ни в `detail`, ни в `properties`. Только безопасные идентификаторы (id, requestId).

## Реестр типов (`type`)

- Ведётся через **enum** — единый источник правды: `slug` + `title` + `status`. Новый тип = одна строка.
- `type`-URI = `<base-uri>` + `slug`. **`base-uri` — из конфига**, не хардкод (`problem.base-uri`).
- Правила URI:
  - **Стабильность** — опубликовал, не меняешь (это контракт с потребителями).
  - **Один тип на семантику**, не на статус (у одного `409` могут быть разные типы: `duplicate-email` vs `version-conflict`).
  - **Namespace под свой домен или URN** (`https://errors.<домен>/problems/...` или `urn:problem:<app>:...`). Резолвиться URI не обязан, но в идеале ведёт на доку типа.
  - `about:blank` — дефолт, когда специфичного типа нет.
- Каталог `type` не является внешним реестром — это твой enum + опциональная страница-описание. Централизованного IANA-реестра для прикладных типов нет.

## Доменные исключения

- Базовый `DomainException extends RuntimeException` **несёт свой `ProblemType`** и `Map<String,Object> properties` (extension-поля). Свойства — fluent `with(key, value)`, поле `transient`.
- Подкласс фиксирует тип: `super(SomeProblemType.X, message)`. Extension-поля — в конструкторе или на месте броска: `throw new OrderNotFoundException(id).with("orderId", id)`.
- **Один `@ExceptionHandler(DomainException.class)`** на всю иерархию — тип и `properties` берутся из исключения. Новый тип ошибки не требует нового handler-а.
  - **Почему:** advice не растёт; добавление ошибки = enum-строка + подкласс.

```java
@Getter
public abstract class DomainException extends RuntimeException {
    private final ProblemType problemType;
    private final transient Map<String, Object> properties = new LinkedHashMap<>();
    // ctors(ProblemType, message[, args]) + public DomainException with(String, Object)
}
```

## Корреляция

- В каждый `ProblemDetail` клади `requestId` (и `traceId`, если есть) как extension.
  - **Почему:** по нему клиент/саппорт находит запрос в логах. Источник — MDC (см. [correlation-and-tracing.md](correlation-and-tracing.md)).

## Мультисервис: вынос в общий starter

Когда обработка ошибок нужна в **≥2 сервисах** — вынеси **механизм** в общий Spring Boot starter. **Каталог типов НЕ выноси** — он у каждого сервиса свой.

- В starter: контракт `interface ProblemType { String slug(); String title(); HttpStatus status(); }`, база `DomainException`, `GlobalExceptionHandler`, фабрика `ProblemDetail`, `@ConfigurationProperties("problem")` с `base-uri`, `@AutoConfiguration`.
- В сервисе: **свой** enum `implements ProblemType`, свои доменные исключения, `problem.base-uri` в `application.yml`.
- Advice собирает `type` = `baseUri.resolve(slug)` — база централизована, enum чистый.
- **YAGNI:** на одном сервисе не выноси — держи self-contained. Преждевременная библиотека дороже дублирования.
- Starter держи **тонким и стабильным** (контракт + advice, не «фреймворк») — ломающие изменения бьют по всем сервисам. Логично объединить с correlation/tracing в один `platform-web-commons`.

## Потребители (другие системы)

- Ветви логику по **`type`** (стабильный URI), **не по тексту `detail`**.
- Перед разбором проверяй `Content-Type: application/problem+json`.
- Неизвестный `type` → fallback на `status` + `title`.
- `instance` + `requestId` → корреляция с логами/саппортом.
- В Java-клиенте: `e.getResponseBodyAs(ProblemDetail.class)`; extension-поля — из `getProperties()`.

## Чек-лист

- [ ] Новые ошибки — `ProblemDetail`/`application/problem+json`, не самописный формат
- [ ] `type` из enum-реестра, `base-uri` из конфига, URI стабилен
- [ ] `DomainException` несёт `ProblemType` + `properties`; один advice на иерархию
- [ ] `detail` для 5xx — generic; детали/стектрейс — в лог; PII/секреты не в теле
- [ ] `requestId` в каждом ProblemDetail
- [ ] Встроенные исключения Spring — `problemdetails.enabled=true`
- [ ] ≥2 сервисов → механизм в общий starter, каталог типов остаётся в сервисе
- [ ] Legacy-формат (если есть) — класс `ApiError`, не `ErrorResponse`
