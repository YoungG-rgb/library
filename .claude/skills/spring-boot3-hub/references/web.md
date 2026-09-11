# Web Layer - Controllers & REST APIs

## REST Controller Pattern

```java
@Validated
@RestController
@RequiredArgsConstructor
@RequestMapping("/api/users")
public class UserController {
    private final UserService userService;
    
    @GetMapping
    public ResponseEntity<Page<UserDto>> getUsers(
            @PageableDefault(size = 20, sort = "createdAt") Pageable pageable) {
        return ResponseEntity.ok(userService.findAll(pageable));
    }

    @GetMapping("/{id}")
    public ResponseEntity<UserDto> getUser(@PathVariable Long id) {
        return ResponseEntity.ok(userService.findById(id));
    }

    @PostMapping
    public ResponseEntity<UserDto> createUser(@Valid @RequestBody UserCreateRequest request) {
        UserDto user = userService.create(request);
        URI location = ServletUriComponentsBuilder
                .fromCurrentRequest()
                .path("/{id}")
                .buildAndExpand(user.id())
                .toUri();
        return ResponseEntity.created(location).body(user);
    }

    @PutMapping("/{id}")
    public ResponseEntity<UserDto> updateUser(@Valid @RequestBody UserUpdateRequest request) {
        return ResponseEntity.ok(userService.update(request));
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void deleteUser(@PathVariable Long id) {
        userService.delete(id);
    }
}
```

## Request DTOs with Validation

```java
public record UserCreateRequest(
    @NotBlank(message = "Email is required")
    @Email(message = "Email must be valid")
    String email,

    @NotBlank(message = "Password is required")
    @Size(min = 8, max = 100, message = "Password must be 8-100 characters")
    @Pattern(regexp = "^(?=.*[A-Z])(?=.*[a-z])(?=.*\\d).*$",
             message = "Password must contain uppercase, lowercase, and digit")
    String password,

    @NotBlank(message = "Username is required")
    @Size(min = 3, max = 50)
    @Pattern(regexp = "^[a-zA-Z0-9_]+$", message = "Username must be alphanumeric")
    String username,

    @Min(value = 18, message = "Must be at least 18")
    @Max(value = 120, message = "Must be at most 120")
    Integer age
) {}

public record UserUpdateRequest(
    @Email(message = "Email must be valid")
    String email,

    @Size(min = 3, max = 50)
    String username
) {}
```

## Response DTOs

```java
public record UserDto (
    Long id,
    String email,
    String username,
    Integer age,
    Boolean active,
    LocalDateTime createdAt,
    LocalDateTime updatedAt
) {}
```

## Mapping DTOs Entity (mapstruct)
```xml
<!-- Tools -->
<dependency>
    <groupId>org.mapstruct</groupId>
    <artifactId>mapstruct</artifactId>
    <version>${mapstruct.version}</version>
    <scope>compile</scope>
</dependency>

<plugin>
<groupId>org.apache.maven.plugins</groupId>
<artifactId>maven-compiler-plugin</artifactId>
<configuration>
    <source>${java.version}</source>
    <target>${java.version}</target>
    <annotationProcessorPaths>
        <path>
            <groupId>org.mapstruct</groupId>
            <artifactId>mapstruct-processor</artifactId>
            <version>${mapstruct.version}</version>
        </path>
        <path>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
        </path>
        <path>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok-mapstruct-binding</artifactId>
            <version>${mapstruct.binding.version}</version>
        </path>
    </annotationProcessorPaths>
    <compilerArgs>
        <arg>
            -Amapstruct.defaultComponentModel=spring
        </arg>
    </compilerArgs>
</configuration>
</plugin>
```

```java
@Mapper
public interface UserMapper {
    UserDto toDto(User entity);
    User toEntity(UserCreateRequest userCreateRequest);
}
```

## Global Exception Handling

> Полные правила (формат, реестр типов, PII в ошибках, корреляция, вынос в общий
> starter) — стандарт [`error-handling.md`](../../../standards/error-handling.md).
> Ниже — self-contained пример для одного сервиса.

Доменные исключения несут **свой `ProblemType`** — тогда один handler на всю иерархию,
без per-exception `@ExceptionHandler`. (`ProblemType` определён ниже.)

> **Мультисервис:** при ≥2 сервисах вынеси **механизм** (контракт
> `interface ProblemType { String slug(); String title(); HttpStatus status(); }`,
> `DomainException`, advice, `base-uri` из конфига, auto-configuration) в общий starter;
> **каталог типов** (enum) остаётся в каждом сервисе. См. `error-handling.md`.

```java
// База: доменное исключение знает свой тип проблемы + несёт свои extension-поля
@Getter
public abstract class DomainException extends RuntimeException {
    private final ProblemType problemType;
    // transient: значения могут быть несериализуемыми, а на web-путь это не влияет
    private final transient Map<String, Object> properties = new LinkedHashMap<>();

    protected DomainException(ProblemType problemType, String message) {
        super(message);
        this.problemType = problemType;
    }
    protected DomainException(ProblemType problemType, String message, Object... args) {
        super(org.slf4j.helpers.MessageFormatter.arrayFormat(message, args).getMessage());
        this.problemType = problemType;
    }

    /** Добавить extension-поле для ProblemDetail (fluent). */
    public DomainException with(String key, Object value) {
        this.properties.put(key, value);
        return this;
    }
}

public class ResourceNotFoundException extends DomainException {
    public ResourceNotFoundException(String message) {
        super(ProblemType.RESOURCE_NOT_FOUND, message);
    }
    public ResourceNotFoundException(String message, Object... messageParameters) {
        super(ProblemType.RESOURCE_NOT_FOUND, message, messageParameters);
    }
}

// Новое доменное исключение = enum-строка + подкласс; extension-поля — через with(...):
// public class DuplicateEmailException extends DomainException {
//     public DuplicateEmailException(String email) {
//         super(ProblemType.CONFLICT, "Email {} already exists", email);
//         with("email", email);                 // уйдёт в ProblemDetail
//     }
// }
// Либо на месте броска: throw new ResourceNotFoundException("Order {} not found", id).with("orderId", id);
```

### Рекомендуемо: ProblemDetail (RFC 9457 / `application/problem+json`)

В Boot 3 `@ExceptionHandler` может **возвращать `ProblemDetail` напрямую** — Spring сам
ставит content-type `application/problem+json`. Стандартные поля: `type` (стабильный
URI типа проблемы — по нему клиент ветвит логику), `title`, `status`, `detail` (текст
конкретного случая), `instance`; всё остальное — extension через `setProperty(...)`.

Реестр типов — enum: единый источник правды для `type`-URI + `title` + `status`.
Новый тип ошибки = одна строка. `type()` резолвиться не обязан, но в идеале ведёт
на доку типа (см. правила ниже).

```java
@Getter
@RequiredArgsConstructor
public enum ProblemType {
    //                 slug                  title                     status
    VALIDATION_ERROR  ("validation-error",   "Validation error",       HttpStatus.BAD_REQUEST),
    RESOURCE_NOT_FOUND("resource-not-found", "Resource not found",     HttpStatus.NOT_FOUND),
    CONFLICT          ("conflict",           "Conflict",               HttpStatus.CONFLICT),
    INTERNAL_ERROR    ("internal-error",     "Internal server error",  HttpStatus.INTERNAL_SERVER_ERROR);

    private static final String BASE = "https://errors.example.com/problems/";  // свой домен / URN

    private final String slug;
    private final String title;
    private final HttpStatus status;

    public URI type() { return URI.create(BASE + slug); }
}
```

```java
@Slf4j
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(DomainException.class)
    public ProblemDetail handleDomain(DomainException ex, HttpServletRequest req) {
        log.warn("{}: {}", ex.getProblemType(), ex.getMessage());
        ProblemDetail pd = problem(ex.getProblemType(), ex.getMessage(), req);
        ex.getProperties().forEach(pd::setProperty);   // разные props у разных исключений
        return pd;
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ProblemDetail handleValidation(MethodArgumentNotValidException ex, HttpServletRequest req) {
        ProblemDetail pd = problem(ProblemType.VALIDATION_ERROR, "Request validation failed", req);
        Map<String, String> errors = ex.getBindingResult().getFieldErrors().stream()
                .collect(Collectors.toMap(FieldError::getField,
                        fe -> Optional.ofNullable(fe.getDefaultMessage()).orElse("Invalid value"),
                        (a, b) -> a));
        pd.setProperty("errors", errors);                       // extension-поле
        return pd;
    }

    @ExceptionHandler(DataIntegrityViolationException.class)
    public ProblemDetail handleConflict(DataIntegrityViolationException ex, HttpServletRequest req) {
        log.warn("Data integrity violation", ex);               // детали БД — только в лог
        return problem(ProblemType.CONFLICT, "Resource conflict — may already exist", req);
    }

    @ExceptionHandler(Exception.class)
    public ProblemDetail handleUnexpected(Exception ex, HttpServletRequest req) {
        log.error("Unexpected error", ex);                      // стектрейс — только в лог
        return problem(ProblemType.INTERNAL_ERROR, "Internal error", req);  // detail для 5xx — generic
    }

    private ProblemDetail problem(ProblemType t, String detail, HttpServletRequest req) {
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(t.getStatus(), detail);
        pd.setType(t.type());
        pd.setTitle(t.getTitle());
        pd.setInstance(URI.create(req.getRequestURI()));
        return pd;
    }
}
```

**Правила реестра `type`:** стабильность — опубликовал URI, не меняешь (это контракт);
один тип на **семантику**, не на статус (у 409 могут быть разные типы —
`duplicate-email` vs `version-conflict`); namespace под свой домен или URN; `about:blank`
как дефолт, если специфичного типа нет; резолвиться URI не обязан, но в идеале ведёт на
доку типа (страница/якорь в API-доках).

Чтобы **встроенные** исключения Spring (404 на роут, 405, 415…) тоже отдавались как
problem+json — включи флаг:

```yaml
spring:
  mvc:
    problemdetails:
      enabled: true          # WebFlux: spring.webflux.problemdetails.enabled
```

Для тонкой настройки встроенных — расширь `ResponseEntityExceptionHandler` и переопредели
нужные `handle*`, дополняя `ProblemDetail` extension-полями.

### Что уходит по проводу

```
HTTP/1.1 400 Bad Request
Content-Type: application/problem+json
```
```json
{
  "type": "https://errors.example.com/validation-error",
  "title": "Validation error",
  "status": 400,
  "detail": "Request validation failed",
  "instance": "/api/v1/users",
  "errors": { "email": "must be a well-formed email", "age": "must be >= 18" },
  "requestId": "3f9c2a1e-7b0d-4b2a-9f1e-2c8d5a6b7c1d"
}
```

### Как потребляют другие системы

Ветвление — по `type` (стабильный URI), **не** по тексту `detail`.

```java
// Java-клиент (RestClient): Spring сам десериализует problem+json в ProblemDetail
catch (RestClientResponseException e) {
    ProblemDetail pd = e.getResponseBodyAs(ProblemDetail.class);
    if (pd != null && pd.getType().toString().endsWith("/validation-error")) {
        Object errors = pd.getProperties().get("errors");           // extension
    }
    String requestId = pd == null ? null
            : String.valueOf(pd.getProperties().get("requestId"));  // для саппорта/логов
}
```

```js
// JS/фронт
if (!res.ok && res.headers.get("content-type")?.includes("application/problem+json")) {
  const p = await res.json();
  switch (p.type) {
    case "https://errors.example.com/validation-error": showFieldErrors(p.errors); break;
    default: toast(`${p.title}: ${p.detail} (id: ${p.requestId})`);
  }
}
```

**Правила для потребителей** (задокументируй в API-контракте): проверяй content-type
перед парсингом; ветви по `type`, не по `detail`; неизвестный `type` → fallback на
`status` + `title`; `instance` + `requestId` → корреляция с логами.

### Legacy: `ApiError` (только для обратной совместимости)

Плоский самописный формат. Оставляй, **только** если есть старые потребители, завязанные
на `{status,message,path}`; новый код — на `ProblemDetail` выше.

> ⚠️ Не называй класс `ErrorResponse` — коллизия с интерфейсом
> `org.springframework.web.ErrorResponse` (обёртка над `ProblemDetail`). Отдаётся как
> `application/json`, а не `application/problem+json` — вне RFC 9457.

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
@Accessors(chain = true)
@FieldDefaults(level = AccessLevel.PRIVATE)
public class ApiError {
    int status = 500;
    String message;
    String path;
    Map<String, String> errors = new HashMap<>();
    LocalDateTime timestamp = LocalDateTime.now();

    public static ApiError of(HttpStatus httpStatus, String message, String path) {
        return new ApiError()
                .setPath(path)
                .setMessage(message)
                .setStatus(httpStatus.value());
    }

    public void addError(FieldError fieldError) {
        String err = Optional.ofNullable(fieldError.getDefaultMessage()).orElse("Invalid value");
        this.errors.put(fieldError.getField(), err);
    }
}
```

## Custom Validation

```java
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = UniqueEmailValidator.class)
public @interface UniqueEmail {
    String message() default "Email already exists";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

@Component
@RequiredArgsConstructor
public class UniqueEmailValidator implements ConstraintValidator<UniqueEmail, String> {
    private final UserRepository userRepository;
    
    @Override
    public boolean isValid(String email, ConstraintValidatorContext context) {
        if (email == null) return true;
        return !userRepository.existsByEmail(email);
    }
}
```

## WebClient for External APIs (Configuration components)

> Проброс `X-Request-Id` / correlation / трейсинга через WebClient-фильтры и
> Context↔MDC-мост — см. стандарт [`correlation-and-tracing.md`](../../../standards/correlation-and-tracing.md).

### CorrelationHeader (единый источник правды)

Один enum управляет всеми тремя звеньями: ingress (создание), MDC (логи), egress
(проброс). Добавить новый сквозной заголовок = одна строка. `traceparent` сюда НЕ
добавляй — трейсинг ведёт `micrometer-tracing`.

```java
@Getter
@RequiredArgsConstructor
public enum CorrelationHeader {
    //           header              generate  echo   propagate  toMdc
    REQUEST_ID  ("X-Request-Id",     true,     true,  true,      true),
    CORRELATION ("X-Correlation-Id", true,     true,  true,      true),
    TENANT_ID   ("X-Tenant-Id",      false,    false, true,      true);

    /** Общий валидатор: safe-charset + лимит длины (защита от мусора / high-cardinality). */
    public static final Pattern SAFE_ID = Pattern.compile("^[A-Za-z0-9._-]{1,64}$");

    private final String header;
    private final boolean generateIfAbsent;   // request/correlation — да; tenant/user — нет
    private final boolean echoToResponse;     // вернуть клиенту в заголовке ответа
    private final boolean propagateDownstream;// уходит в исходящий WebClient-запрос
    private final boolean toMdc;              // попадает в MDC → в логи
}
```

### ReactorHooksConfiguration
```java
@Slf4j
@Configuration
public class ReactorHooksConfiguration {
    private static final String CANCELLATION_MARKER = "has been released already due to cancellation";

    @PostConstruct
    public void registerHooks(){
        this.contextPropagation();
        this.errorDroppedHook();
    }

    public void contextPropagation(){
        Arrays.stream(CorrelationHeader.values())
                .filter(CorrelationHeader::isToMdc)
                .forEach(h -> ContextRegistry.getInstance()
                        .registerThreadLocalAccessor(new HeaderMdcAccessor(h.getHeader())));
        Hooks.enableAutomaticContextPropagation();
    }

    public void errorDroppedHook() {
        Hooks.onErrorDropped(error -> {
            if (isBenignCancellation(error)) {
                log.debug("Dropped benign post-cancellation error: {}", error.toString());
            } else {
                log.warn("Reactor dropped an error with no subscriber to receive it", error);
            }
        });
    }

    private boolean isBenignCancellation(Throwable error) {
        Throwable prev = null;
        for (Throwable t = error; t != null && t != prev; prev = t, t = t.getCause()) {
            if (t instanceof AbortedException) {
                return true;
            }
            String message = t.getMessage();
            if (message != null && message.contains(CANCELLATION_MARKER)) {
                return true;
            }
        }
        return false;
    }

    @RequiredArgsConstructor
    public static class HeaderMdcAccessor implements ThreadLocalAccessor<String> {
        private final String key;
        @NotNull
        @Override public Object key()            { return key; }
        @Override public String getValue()       { return MDC.get(key); }
        @Override public void setValue(String v) { MDC.put(key, v); }
        @Override public void setValue()         { MDC.remove(key); }
    }
}
```

### CorrelationWebFilter (ingress)

Дескриптор-driven: идём по `CorrelationHeader`, у каждого свои правила
(генерить/эхо). `X-Request-Id`/`X-Correlation-Id` генерятся при отсутствии,
`X-Tenant-Id` — нет (его нельзя выдумать).

```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class CorrelationWebFilter implements WebFilter {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        HttpHeaders in = exchange.getRequest().getHeaders();
        Map<String, String> resolved = new LinkedHashMap<>();

        for (CorrelationHeader h : CorrelationHeader.values()) {
            String incoming = in.getFirst(h.getHeader());
            String value;
            if (incoming != null && CorrelationHeader.SAFE_ID.matcher(incoming).matches()) {
                value = incoming;                        // пришёл валидный — берём как есть
            } else if (h.isGenerateIfAbsent()) {
                value = UUID.randomUUID().toString();    // request/correlation — генерим
            } else {
                continue;                                // tenant/user — нет и не выдумываем
            }
            resolved.put(h.getHeader(), value);
            if (h.isEchoToResponse()) {
                exchange.getResponse().getHeaders().set(h.getHeader(), value);
            }
        }

        return chain.filter(exchange)
                .contextWrite(ctx -> ctx.putAllMap(resolved)); // reactor-core 3.4+
    }
}
```

### ContextHeadersPropagationFilter

Один фильтр с белым списком сквозных ключей — не плодим по фильтру на заголовок.
Добавление нового ключа = одна строка в `PROPAGATED`. `traceparent` сюда НЕ кладём —
его проставляет `micrometer-tracing` сам (см. `correlation-and-tracing.md`).

```java
@Component
public class ContextHeadersPropagationFilter implements ExchangeFilterFunction {

    // из enum берём только заголовки с флагом propagateDownstream
    private static final List<String> PROPAGATED = Arrays.stream(CorrelationHeader.values())
            .filter(CorrelationHeader::isPropagateDownstream)
            .map(CorrelationHeader::getHeader)
            .toList();

    @Override
    public Mono<ClientResponse> filter(ClientRequest request, ExchangeFunction next) {
        return Mono.deferContextual(ctx -> {
            ClientRequest.Builder mutated = ClientRequest.from(request);
            for (String key : PROPAGATED) {
                ctx.getOrEmpty(key).ifPresent(v -> mutated.header(key, v.toString()));
            }
            return next.exchange(mutated.build());
        });
    }
}
```

### WebClientLoggingFilter
```java
@Slf4j
@Component
public class WebClientLoggingFilter implements ExchangeFilterFunction {

    @Override
    public Mono<ClientResponse> filter(ClientRequest request, ExchangeFunction next) {
        long startNanos = System.nanoTime();
        return next.exchange(request)
                // читаем requestId из Reactor Context, НЕ из ThreadLocal MDC
                .doOnEach(signal -> {
                    // логируем и успех (onNext, включая 4xx/5xx), и падения (onError: таймаут, обрыв, DNS)
                    if (!signal.isOnNext() && !signal.isOnError()) {
                        return;
                    }
                    long ms = (System.nanoTime() - startNanos) / 1_000_000;
                    String requestId = signal.getContextView()
                            .getOrDefault(CorrelationHeader.REQUEST_ID.getHeader(), "-");
                    String target = UriUtils.normalizeUri(request.url().toString());
                    if (signal.isOnNext()) {
                        log.info("[{}] {} {} -> {} ({} ms)",
                                requestId, request.method(), target,
                                signal.get().statusCode(), ms);
                    } else {
                        log.warn("[{}] {} {} -> FAILED: {} ({} ms)",
                                requestId, request.method(), target,
                                signal.getThrowable().toString(), ms);
                    }
                })
                .doFirst(() -> log.debug("→ {} {}", request.method(), UriUtils.normalizeUri(request.url().toString())));
    }
}
```

### DownStreamFilter
```java
/**
 * Counts downstream (WebClient) timeout terminations that reactor-netty does not classify on its own:
 * response (read) timeout, connect timeout and connection-pool acquire timeout.
 *
 * <p>A single shared, stateless filter used by every WebClient bean. Exposed metric (Prometheus):
 * {@code webflux_client_requests_timeout_total{reason,service}} where {@code reason} in
 * {read_timeout, connect_timeout, pool_acquire_timeout, timeout} and {@code service} is the
 * downstream {@code host[:port]} (aligns with reactor-netty's {@code remote_address} label).
 *
 * <p>Complements {@code reactor_netty_http_client_errors_total} (which counts ALL client errors
 * without a type). HTTP 4xx/5xx arrive as normal responses (onNext) at this layer and are NOT counted.
 */
@Slf4j
@Component
@RequiredArgsConstructor
public class DownstreamTimeoutMetricsFilter implements ExchangeFilterFunction {

    static final String METRIC = "webflux.client.requests.timeout";
    private static final String TAG_REASON = "reason";
    private static final String TAG_SERVICE = "service";
    private static final String SERVICE_UNKNOWN = "unknown";
    private static final int MAX_CAUSE_DEPTH = 10;

    private final MeterRegistry meterRegistry;

    @Override
    public Mono<ClientResponse> filter(ClientRequest request, ExchangeFunction next) {
        String service = serviceOf(request.url());
        return next.exchange(request)
                .doOnError(ex -> classify(ex).ifPresent(reason -> increment(reason, service)));
    }

    private String serviceOf(URI uri) {
        if (uri == null || uri.getHost() == null) {
            return SERVICE_UNKNOWN;
        }
        return uri.getPort() > 0 ? uri.getHost() + ":" + uri.getPort() : uri.getHost();
    }

    private Optional<String> classify(Throwable ex) {
        Throwable t = ex;
        for (int depth = 0; t != null && depth < MAX_CAUSE_DEPTH; depth++) {
            String name = t.getClass().getSimpleName();
            switch (name) {
                case "ReadTimeoutException":
                    return Optional.of("read_timeout");
                case "ConnectTimeoutException":
                    return Optional.of("connect_timeout");
                case "PoolAcquireTimeoutException":
                    return Optional.of("pool_acquire_timeout");
                default:
                    if (t instanceof TimeoutException || name.contains("Timeout")) {
                        return Optional.of("timeout");
                    }
            }
            if (t == t.getCause()) {
                break;
            }
            t = t.getCause();
        }
        return Optional.empty();
    }

    private void increment(String reason, String service) {
        Counter.builder(METRIC)
                .description("Downstream WebClient timeout terminations by type and service")
                .tag(TAG_REASON, reason)
                .tag(TAG_SERVICE, service)
                .register(meterRegistry)
                .increment();
        if (log.isDebugEnabled()) {
            log.debug("Downstream timeout: reason={}, service={}", reason, service);
        }
    }
}
```

### ExternalConnectorsProperties (Connector properties)
```java
@Data
@Validated
@Configuration
@ConfigurationProperties(prefix = "externals.connectors")
public class ExternalConnectorsProperties {
    private Map<String, @Valid ConnectorProperties> rest;

    @Data
    public static class ConnectorProperties {
        private Boolean enableMetrics = Boolean.TRUE;
        private String baseUrl;
        private Map<String, String> defaultHeaders = new HashMap<>();
        private HttpProtocol protocol = HttpProtocol.HTTP11;

        private int connectTimeout = 2_000;
        private Duration readTimeout = Duration.parse("PT10S");

        @Valid
        private Pool pool = new Pool();

        @Data
        public static class Pool {
            @NotBlank
            private String name;
            /** Максимум одновременных соединений в пуле. */
            private int maxConnections = 500;
            /** Максимум запросов, ожидающих свободного соединения в очереди. */
            private int pendingAcquireMaxCount = 2_000;
            /** Таймаут ожидания соединения из пула, мс. */
            private Duration pendingAcquireTimeout = Duration.parse("PT0.5S");
            /** Максимальное время простоя соединения перед закрытием, мс. */
            private Duration maxIdleTime = Duration.parse("PT30S");
            /** Периодичность фоновой очистки просроченных соединений, мс. */
            private Duration evictInBackground = Duration.parse("PT120S");
        }
    }
}
```
```yaml
#Usage
#Создать проперти файл application-connectors.yml
externals:
  connectors:
    rest:
      externalSystemWebClient:
        description: Интеграция с системой - ${externalSystemName} по задаче - ${TASK_URL}
        base-url: ${EXTERNAL_SYSTEM_API_BASE_URL}
        read-timeout: PT2S
        default-headers:
          X-Api-Secure-Key: ${EXTERNAL_SYSTEM_SECURE_KEY}
        pool:
          name: externalSystemPool
          pending-acquire-timeout: PT1S
```

### Utils
```java
@UtilityClass
public class UriUtils {
    
    private static final Pattern QUERY_STRING = Pattern.compile("\\?.*$");
    private static final Pattern UUID_SEGMENT = Pattern.compile("/[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12}");
    private static final Pattern HEX_SEGMENT = Pattern.compile("/[0-9a-fA-F]{16,}");
    private static final Pattern TOKEN_SEGMENT = Pattern.compile("/[A-Za-z0-9_=-]{20,}");
    private static final Pattern NUMERIC_SEGMENT = Pattern.compile("/\\d+");
    private static final int MAX_URI_TAG_LENGTH = 120;

    /**
     * Нормализует URI для использования в качестве тега метрик Prometheus.
     *
     * <p>Без нормализации URI с ID в пути (например {@code /users/12345}) создали бы
     * high-cardinality тег и убили бы Prometheus. Правила применяются по порядку
     * «от специфичного к общему»:
     * <ol>
     *   <li>UUID → {@code /{uuid}}</li>
     *   <li>Hex (≥16 символов: SHA1, MD5, Mongo ObjectId, tx-hash) → {@code /{hex}}</li>
     *   <li>Длинные alphanumeric-токены (≥20: JWT, base64, session id) → {@code /{token}}</li>
     *   <li>Числа (ID, msisdn, timestamp) → {@code /{id}}</li>
     * </ol>
     * Query-string отсекается. Если результат длиннее {@link #MAX_URI_TAG_LENGTH} — обрезается.
     */
    public static String normalizeUri(String uri) {
        if (uri == null || uri.isBlank()) {
            return "unknown";
        }
        String path = QUERY_STRING.matcher(uri).replaceAll("");
        path = UUID_SEGMENT.matcher(path).replaceAll("/{uuid}");
        path = HEX_SEGMENT.matcher(path).replaceAll("/{hex}");
        path = TOKEN_SEGMENT.matcher(path).replaceAll("/{token}");
        path = NUMERIC_SEGMENT.matcher(path).replaceAll("/{id}");
        if (path.length() > MAX_URI_TAG_LENGTH) {
            path = path.substring(0, MAX_URI_TAG_LENGTH);
        }
        return path;
    }
}
```

### WebClient configuration
```java
@Configuration
@RequiredArgsConstructor
public class WebClientConfiguration {

    private final ExternalConnectorsProperties externalConnectorsProperties;
    private final DownstreamTimeoutMetricsFilter downstreamTimeoutMetricsFilter;
    private final WebClientLoggingFilter webClientLoggingFilter;
    private final ContextHeadersPropagationFilter contextHeadersPropagationFilter;

    private ConnectionProvider customConnectionProvider(ExternalConnectorsProperties.ConnectorProperties properties) {
        ExternalConnectorsProperties.ConnectorProperties.Pool pool = properties.getPool();
        return ConnectionProvider.builder(pool.getName())
                .lifo()
                .metrics(Boolean.TRUE.equals(properties.getEnableMetrics()))
                .maxConnections(pool.getMaxConnections())
                .pendingAcquireMaxCount(pool.getPendingAcquireMaxCount())
                .pendingAcquireTimeout(pool.getPendingAcquireTimeout())
                .maxIdleTime(pool.getMaxIdleTime())
                .evictInBackground(pool.getEvictInBackground())
                .build();
    }

    private WebClient createWebClient(ExternalConnectorsProperties.ConnectorProperties properties) {
        HttpClient httpClient = HttpClient.create(customConnectionProvider(properties))
                .keepAlive(true)
                .protocol(properties.getProtocol())
                .responseTimeout(properties.getReadTimeout())
                .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, properties.getConnectTimeout())
                .metrics(Boolean.TRUE.equals(properties.getEnableMetrics()), UriUtils::normalizeUri);

        WebClient.Builder builder = WebClient.builder()
                .baseUrl(properties.getBaseUrl())
                .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
                .clientConnector(new ReactorClientHttpConnector(httpClient));

        properties.getDefaultHeaders().forEach(builder::defaultHeader);
        return builder
                .filter(contextHeadersPropagationFilter) // 1. пробрасываем сквозные заголовки (X-Request-Id и др.)
                .filter(webClientLoggingFilter)          // 2. логируем method/url/status/ms (видит проброшенные заголовки)
                .filter(downstreamTimeoutMetricsFilter)  // 3. ближе всего к сети — классифицирует тайм-ауты
                .build();
    }

    @Bean
    public WebClient externalSystemWebClient() {
        return createWebClient(externalConnectorsProperties.getRest().get("externalSystemWebClient"));
    }
}
```

## CORS Configuration

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("http://localhost:3000", "https://example.com")
            .allowedMethods("GET", "POST", "PUT", "DELETE", "OPTIONS")
            .allowedHeaders("*")
            .allowCredentials(true)
            .maxAge(3600);
    }
}
```

## Quick Reference

| Annotation                 | Purpose                                                               |
|----------------------------|-----------------------------------------------------------------------|
| `@RestController`          | Marks class as REST controller (combines @Controller + @ResponseBody) |
| `@RequestMapping`          | Maps HTTP requests to handler methods                                 |
| `@GetMapping/@PostMapping` | HTTP method-specific mappings                                         |
| `@PathVariable`            | Extracts values from URI path                                         |
| `@RequestParam`            | Extracts query parameters                                             |
| `@RequestBody`             | Binds request body to method parameter                                |
| `@Valid`                   | Triggers validation on request body                                   |
| `@RestControllerAdvice`    | Global exception handling for REST controllers                        |
| `@ResponseStatus`          | Sets HTTP status code for method                                      |
