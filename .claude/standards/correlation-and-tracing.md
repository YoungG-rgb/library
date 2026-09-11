# Correlation & Tracing

Правила сквозной идентификации запроса: бизнес-корреляция (`X-Request-Id`) и
технический трейсинг (`traceId`/`spanId`). Как прокидывать, как класть в MDC,
что логировать, что запрещено.

## Два независимых механизма — не смешивай

| Что                     | Кто владеет                       | Проброс к downstream      | В MDC         | Логируешь          |
|-------------------------|-----------------------------------|---------------------------|---------------|--------------------|
| `traceId` / `spanId`    | `micrometer-tracing` + OTel/Brave | автоматом (`traceparent`) | автоматом     | да, только читаешь |
| `X-Request-Id` (бизнес) | ты, руками                        | твой propagate-фильтр     | твой accessor | да                 |

- **`traceId`/`spanId` руками НЕ трогай.** Не клади в свой Context-ключ, не инжектируй `traceparent` сам. Этим занимается `micrometer-tracing`.
  - **Почему:** ручной проброс + автоматический = двойной хедер и рассинхрон со sampler'ом. Ты только читаешь их из MDC для логов.
- **`X-Request-Id` веди сам** — он есть всегда, `traceId` может отсутствовать при sampling < 100%.

## Трейсинг (traceId/spanId)

- Подключай `micrometer-tracing-bridge-otel` (или `-brave`) — не изобретай проброс трасс.
- Стандарт заголовка — **W3C Trace Context** (`traceparent`/`tracestate`). B3 (`X-B3-*`) — только для legacy-совместимости.
- traceId/spanId попадают в MDC через бридж трейсинга сами — добавь их в лог-паттерн, но не проставляй руками.

## Бизнес-корреляция (X-Request-Id)

- Один сквозной ключ по умолчанию — `X-Request-Id`. Заводи `X-Correlation-Id` / `X-Tenant-Id` **только** если реально есть отдельная бизнес-нить / мультиарендность. Не плоди ключи впрок.
  - **Почему:** `X-Request-Id` часто закрывает и роль correlation-id, если нет edge-gateway, который ставит стабильный correlation.
- Ингресс ставит id **на границе** и возвращает его в заголовок ответа (для саппорта/клиента).
- **Входящий id недоверенный** — валидируй/санитизируй формат (ограничь длину и charset) или генери свой. Не принимай произвольную строку. Строгий UUID — только если это твой контракт, иначе отбросишь легитимные внешние id и порвёшь корреляцию.
  - **Почему:** внешний вызов может залить в логи/метрики мусор или high-cardinality.

### Единый источник — enum `CorrelationHeader`

- Один enum описывает ВСЕ сквозные заголовки и их правила. Ingress, MDC-мост и egress читают его.
- Добавить новый заголовок = одна строка в enum. `traceparent` в enum НЕ добавляй — его ведёт трейсинг.
- `generateIfAbsent` = `true` только для того, что можно выдумать (`X-Request-Id`, `X-Correlation-Id`). Для `X-Tenant-Id`/`X-User-Id` — `false`: если не пришёл, его нет.

```java
@Getter
@RequiredArgsConstructor
public enum CorrelationHeader {
    //           header              generate  echo   propagate  toMdc
    REQUEST_ID  ("X-Request-Id",     true,     true,  true,      true),
    CORRELATION ("X-Correlation-Id", true,     true,  true,      true),
    TENANT_ID   ("X-Tenant-Id",      false,    false, true,      true);

    /** safe-charset + лимит длины: защита от мусора / high-cardinality. Не строгий UUID (это не наш
        контракт) — иначе легитимные не-UUID id из внешних систем отбрасывались бы, рвя корреляцию. */
    public static final Pattern SAFE_ID = Pattern.compile("^[A-Za-z0-9._-]{1,64}$");

    private final String header;
    private final boolean generateIfAbsent;
    private final boolean echoToResponse;
    private final boolean propagateDownstream;
    private final boolean toMdc;
}
```

### Ингресс — реактивный сервер (WebFlux)

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
                value = incoming;
            } else if (h.isGenerateIfAbsent()) {
                value = UUID.randomUUID().toString();
            } else {
                continue; // tenant/user — не выдумываем
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

### Ингресс — сервлетный сервер (MVC)

- `WebFilter` НЕ работает в сервлетном стеке. Используй `OncePerRequestFilter` + `MDC.put`, снимай в `finally`.

- Тот же enum `CorrelationHeader`; в сервлете кладём напрямую в MDC (мост не нужен — один поток), снимаем ВСЕ ключи в `finally`.

```java
@Component
public class CorrelationServletFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse resp,
                                    FilterChain chain) throws ServletException, IOException {
        List<String> putKeys = new ArrayList<>();
        for (CorrelationHeader h : CorrelationHeader.values()) {
            String incoming = req.getHeader(h.getHeader());
            String value;
            if (incoming != null && CorrelationHeader.SAFE_ID.matcher(incoming).matches()) {
                value = incoming;
            } else if (h.isGenerateIfAbsent()) {
                value = UUID.randomUUID().toString();
            } else {
                continue;
            }
            if (h.isToMdc()) { MDC.put(h.getHeader(), value); putKeys.add(h.getHeader()); }
            if (h.isEchoToResponse()) { resp.setHeader(h.getHeader(), value); }
        }
        try {
            chain.doFilter(req, resp);
        } finally {
            putKeys.forEach(MDC::remove); // иначе протечёт в другой запрос на том же потоке
        }
    }
}
```

## Мост Context → MDC (только реактивный стек)

- В реактивном коде MDC ThreadLocal-bound, а пайплайн прыгает по потокам → `MDC.get(...)` в операторах пуст. Нужен мост.
- Регистрируй **свой** `ThreadLocalAccessor`, чтобы ключ Context == ключ MDC. Не полагайся на дефолтный ключ micrometer.
- Один параметризованный accessor на ключ; регистрируй по одному на каждый `CorrelationHeader` с `toMdc = true`.

```java
@RequiredArgsConstructor
public class HeaderMdcAccessor implements ThreadLocalAccessor<String> {
    private final String key;
    @Override public Object key()            { return key; }
    @Override public String getValue()       { return MDC.get(key); }
    @Override public void setValue(String v) { MDC.put(key, v); }
    @Override public void setValue()         { MDC.remove(key); }
}

@Configuration
public class ContextPropagationConfig {
    @PostConstruct
    void init() {
        Arrays.stream(CorrelationHeader.values())
                .filter(CorrelationHeader::isToMdc)
                .forEach(h -> ContextRegistry.getInstance()
                        .registerThreadLocalAccessor(new HeaderMdcAccessor(h.getHeader())));
        Hooks.enableAutomaticContextPropagation();
    }
}
```

- Зависимость: `io.micrometer:context-propagation` (в Boot 3.x тянется micrometer'ом).
- В сервлетном стеке мост не нужен — MDC живёт на том же потоке.

## Egress — проброс к downstream (WebClient)

- Один фильтр с **белым списком** ключей из Context. Не плоди фильтр на каждый заголовок.
- В список кладёшь **только бизнес-ключи**. `traceparent` НЕ указывай — его проставит трейсинг.

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
            ClientRequest.Builder b = ClientRequest.from(request);
            for (String key : PROPAGATED) {
                ctx.getOrEmpty(key).ifPresent(v -> b.header(key, v.toString()));
            }
            return next.exchange(b.build());
        });
    }
}
```

- Порядок фильтров на `WebClient` (первый добавленный = внешний): `propagate` → `logging` → `metrics/timeout`.
  - **Почему:** id проставляется до наблюдения; логирующий — внешний, меряет весь вызов; метрик-фильтр — внутренний, видит сырую сетевую ошибку.
- Если WebClient зовётся вне пайплайна запроса (шедулер, `.block()` в отдельном потоке) — Context пуст. Генери id на старте задачи и клади через `.contextWrite(...)`.

## Запрещено

- **Секреты в лог — никогда**: `Authorization`, `Cookie`, api-key-заголовки. Проброс — только внутри доверенного контура и осознанно.
- **PII в лог/MDC — никогда в исходном виде** (см. основной запрет по PII в логах). `X-Forwarded-For` — это IP, в лог только по необходимости.
- **`Idempotency-Key` — НЕ в общий белый список.** Он живёт на уровне конкретной записывающей операции, а не всего запроса; его ставит инициатор, downstream дедуплицирует. В логах — на `debug`.

## Лог-паттерн

- Пробрасывай оба уровня в лог. В JSON-логах — отдельные поля, не строка.

```
[%X{X-Request-Id:-},%X{traceId:-},%X{spanId:-}]
```

## Чек-лист

- [ ] traceId/spanId — через `micrometer-tracing`, не руками
- [ ] `X-Request-Id` ставится на границе, входящий валидируется, возвращается в ответ
- [ ] Реактив: зарегистрирован `ThreadLocalAccessor` + `enableAutomaticContextPropagation()`
- [ ] Сервлет: `MDC.put` на входе, `MDC.remove` в `finally`
- [ ] Egress-проброс — белый список, без `traceparent`
- [ ] Порядок WebClient-фильтров: propagate → logging → metrics
- [ ] Секреты/PII не логируются, `Idempotency-Key` не в общем пробросе
