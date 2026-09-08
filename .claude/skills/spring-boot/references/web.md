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
```java
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) {
        super(message);
    }
    public ResourceNotFoundException(String message, Object... messageParameters) {
        super(org.slf4j.helpers.MessageFormatter.arrayFormat(message, messageParameters).getMessage());
    }
}
```

```java
@Slf4j
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex, WebRequest request) {
        log.error("Resource not found: {}", ex.getMessage());
        ErrorResponse errorResponse = ErrorResponse.of(HttpStatus.NOT_FOUND, ex.getMessage(),
                request.getDescription(false));

        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(errorResponse);
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ValidationErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        ValidationErrorResponse response = ValidationErrorResponse.of(HttpStatus.BAD_REQUEST);
        ex.getBindingResult().getFieldErrors().forEach(response::addError);

        return new ResponseEntity<>(response, HttpStatus.BAD_REQUEST);
    }

    @ExceptionHandler(DataIntegrityViolationException.class)
    public ResponseEntity<ErrorResponse> handleDataIntegrity(DataIntegrityViolationException ex, WebRequest request) {
        log.error("Data integrity violation", ex);
        ErrorResponse error = ErrorResponse.of(HttpStatus.CONFLICT,
                "Data integrity violation - resource may already exist",
                request.getDescription(false)
        );
        return new ResponseEntity<>(error, HttpStatus.CONFLICT);
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGlobalException(Exception ex, WebRequest request) {
        log.error("Unexpected error", ex);
        ErrorResponse error = ErrorResponse.of(
                HttpStatus.INTERNAL_SERVER_ERROR,
                "An unexpected error occurred",
                request.getDescription(false)
        );
        return new ResponseEntity<>(error, HttpStatus.INTERNAL_SERVER_ERROR);
    }
}

@Data
@NoArgsConstructor
@AllArgsConstructor
@Accessors(chain = true)
@FieldDefaults(level = AccessLevel.PRIVATE)
public class ErrorResponse {
    int status;
    String message;
    String path;
    LocalDateTime timestamp;

    public ErrorResponse of(HttpStatus httpStatus, String message, String path){
        return new ErrorResponse()
                .setTimestamp(LocalDateTime.now())
                .setPath(path)
                .setMessage(message)
                .setStatus(httpStatus.value());
    }
}

@Data
@NoArgsConstructor
@AllArgsConstructor
@Accessors(chain = true)
@FieldDefaults(level = AccessLevel.PRIVATE)
public class ValidationErrorResponse {
    int status;
    String message;
    Map<String, String> errors = new HashMap<>();
    LocalDateTime timestamp;

    public static ValidationErrorResponse of(HttpStatus httpStatus, String message, Map<String, String> errors) {
        return new ValidationErrorResponse()
                .setTimestamp(LocalDateTime.now())
                .setMessage(message)
                .setStatus(httpStatus.value())
                .setErrors(errors);
    }

    public static ValidationErrorResponse of(HttpStatus httpStatus) {
        return new ValidationErrorResponse()
                .setTimestamp(LocalDateTime.now())
                .setMessage("Validation failed")
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
        ContextRegistry.getInstance().registerThreadLocalAccessor(new RequestIdMdcAccessor());
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

    public static class RequestIdMdcAccessor implements ThreadLocalAccessor<String> {
        @NotNull
        @Override public Object key()            { return RequestIdWebFilter.X_REQUEST_ID; }
        @Override public String getValue()       { return MDC.get(RequestIdWebFilter.X_REQUEST_ID); }
        @Override public void setValue(String v) { MDC.put(RequestIdWebFilter.X_REQUEST_ID, v); }
        @Override public void setValue()         { MDC.remove(RequestIdWebFilter.X_REQUEST_ID); }
    }
}
```

### RequestIdWebFilter
```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class RequestIdWebFilter implements WebFilter {

    public static final String X_REQUEST_ID = "X-Request-Id";
    private static final Pattern UUID_RE = Pattern.compile(
        "^[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12}$");

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        String incoming = exchange.getRequest().getHeaders().getFirst(X_REQUEST_ID);
        // недоверенный вход: принимаем только валидный UUID, иначе генерим свой
        String id = (incoming != null && UUID_RE.matcher(incoming).matches())
                ? incoming
                : UUID.randomUUID().toString();

        exchange.getResponse().getHeaders().set(X_REQUEST_ID, id);

        return chain.filter(exchange)
                .contextWrite(ctx -> ctx.put(X_REQUEST_ID, id));
    }
}
```

### RequestIdPropagationFilter
```java
@Component
@RequiredArgsConstructor
public class RequestIdPropagationFilter implements ExchangeFilterFunction {

    @Override
    public Mono<ClientResponse> filter(ClientRequest request, ExchangeFunction next) {
        return Mono.deferContextual(ctx -> {
            ClientRequest mutated = ctx.getOrEmpty(RequestIdWebFilter.X_REQUEST_ID)
                .map(id -> ClientRequest.from(request)
                    .header(RequestIdWebFilter.X_REQUEST_ID, id.toString())
                    .build())
                .orElse(request);
            return next.exchange(mutated);
        });
    }
}
```

### WebClientLoggingFilter
```java
@Slf4j
@Component
public class WebClientLoggingFilter implements ExchangeFilterFunction {
    public static final String X_REQUEST_ID = "X-Request-Id";

    @Override
    public Mono<ClientResponse> filter(ClientRequest request, ExchangeFunction next) {
        long startNanos = System.nanoTime();
        return next.exchange(request)
                // deferContextual — читаем requestId из Reactor Context, НЕ из ThreadLocal MDC
                .doOnEach(signal -> {
                    if (signal.isOnNext()) {
                        String requestId = signal.getContextView().getOrDefault(X_REQUEST_ID, "-");
                        long ms = (System.nanoTime() - startNanos) / 1_000_000;
                        log.info("[{}] {} {} -> {} ({} ms)",
                                requestId, request.method(),
                                UriUtils.normalizeUri(request.url().toString()),
                                signal.get().statusCode(), ms);
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
    private final RequestIdPropagationFilter requestIdPropagationFilter;

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
                .filter(requestIdPropagationFilter)      // 1. добавляем X-Request-Id в исходящий запрос
                .filter(webClientLoggingFilter)          // 2. логируем method/url/status/ms (видит проброшенный id)
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
