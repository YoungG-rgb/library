---
name: logging-patterns
description: Java logging best practices with SLF4J, structured logging (JSON), and MDC for request tracing. Includes AI-friendly log formats for Claude Code debugging. Use when user asks about logging, debugging application flow, or analyzing logs. Триггеры RU - «добавь логи», «логирование», «трассировка», «MDC», «маскирование», «PII в логах», «structured logging».
---

## ⚠️ Project Standards Override

Если в проекте есть `.claude/standards/` — следуй им, особенно [`service-transactional.md`](.claude/standards/service-transactional.md) (раздел Логирование/MDC) и [`correlation-and-tracing.md`](.claude/standards/correlation-and-tracing.md) (`X-Request-Id`, трейсинг, Context↔MDC-мост в реактивном коде, проброс к downstream).

**Жёсткие правила логирования (всегда)**:

- **PII в логах — никогда в исходном виде.** `msisdn` маскируется (формат `+9967001234**`), `PAN` — никогда даже в маскированном виде в обычных логах. Email/паспорт — через утилиту-маскер проекта (`shared.utilities`).
- **Запрещено в MDC**: PII (даже под другим ключом), payload'ы, объекты, изменчивые значения. Только стабильные идентификаторы операции: `requestId`, `traceId`, `correlationId`, `eventId`.
- **MDC ставится на границе** (controller / consumer / scheduler-tick), **снимается в `finally`**. Иначе значение протечёт в другие задачи на том же потоке (особенно с тред-пулами и виртуальными потоками).
- **Виртуальные потоки и `@Scheduled`-Task НЕ наследуют MDC автоматически.** Ставь MDC внутри `Task.run()`, не рассчитывай на родительский поток.
- **Не добавляй `MDC.put(...)` молча.** Набор MDC-ключей — продуктовое решение (как фильтруют в Kibana/Grafana). Спрашивай у пользователя, что класть.
- **Уровни**:
  - `log.info` — границы операций (получено, сохранено, отправлено).
  - `log.debug` — дубликаты, skip-ы, детали.
  - `log.warn` — нештатные, но восстановимые ситуации.
  - `log.error` — только с пробросом исключения или явным алертом.
- **`log.error(msg, e)`** — всегда с throwable вторым аргументом, не `e.getMessage()` в строке. Stack trace обязателен.
- **Параметризованное логирование**: `log.info("Order {} created", id)` — не конкатенация.
- **Catch `Exception` в `Runnable.run()` виртуального потока** — обязательно, иначе поток умрёт молча и stuck-строки останутся в БД (см. `scheduler.md`).

---

# Logging Patterns Skill

Effective logging for Java applications with focus on structured, AI-parsable formats.

## When to Use
- User says "add logging" / "improve logs" / "debug this"
- Analyzing application flow from logs
- Setting up structured logging (JSON)
- Request tracing with correlation IDs
- AI/Claude Code needs to analyze application behavior

---

## AI-Friendly Logging

> **Key insight:** JSON logs are better for AI analysis - faster parsing, fewer tokens, direct field access.

### Why JSON for AI/Claude Code?

```
# Text format - AI must "interpret" the string
2026-01-29 10:15:30 INFO OrderService - Order 12345 created for user-789, total: 99.99

# JSON format - AI extracts fields directly
{"timestamp":"2026-01-29T10:15:30Z","level":"INFO","orderId":12345,"userId":"user-789","total":99.99}
```

| Aspect           | Text                       | JSON                |
|------------------|----------------------------|---------------------|
| Parsing          | Regex/interpretation       | Direct field access |
| Token usage      | Higher (repeated patterns) | Lower (structured)  |
| Error extraction | Parse stack trace text     | `exception` field   |
| Filtering        | grep patterns              | `jq` queries        |

### Log Format Optimized for AI Analysis

```json
{
  "timestamp": "2026-01-29T10:15:30.123Z",
  "level": "INFO",
  "logger": "com.example.OrderService",
  "message": "Order created",
  "requestId": "req-abc123",
  "traceId": "trace-xyz",
  "orderId": 12345,
  "userId": "user-789",
  "duration_ms": 45,
  "step": "payment_completed"
}
```

**Key fields for AI debugging:**
- `X-Request-Id` - group all logs from same request
- `step` - track progress through flow
- `duration_ms` - identify slow operations
- `level` - quick filter for errors

### Reading Logs with AI/Claude Code

When asking AI to analyze logs:

```bash
# Get recent errors
cat app.log | jq 'select(.level == "ERROR")' | tail -20

# Follow specific request
cat app.log | jq 'select(.requestId == "req-abc123")'

# Find slow operations
cat app.log | jq 'select(.duration_ms > 1000)'
```

AI can then:
1. Parse JSON directly (no guessing)
2. Follow request flow via requestId
3. Identify exactly where errors occurred
4. Measure timing between steps

---

### Profile-Based Switching

```yaml
# application.yml (default - JSON for AI/prod)
spring:
  application:
    environment: ${SPRING_APPLICATION_ENVIRONMENT:local}
    name: ${SPRING_APPLICATION_NAME:application_name}
```

---

### Logstash Logback Encoder

**pom.xml:**
```xml
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <!-- версия из BOM (Spring Boot управляет ей сам) -->
</dependency>
<dependency>
    <groupId>org.codehaus.janino</groupId>
    <artifactId>janino</artifactId>
</dependency>
```

**logback-spring.xml:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <include resource="org/springframework/boot/logging/logback/defaults.xml"/>
  <springProperty scope="context" name="service-name" source="spring.application.name"/>
  <springProperty scope="context" name="env" source="spring.application.environment" defaultValue="local"/>
  <property name="LOG_PATTERN"
            value="%d{yyyy-MM-dd HH:mm:ss.SSS} %highlight(%-5level) %cyan([${service-name}]) %green([%logger{1}]) %yellow([%X{X-Request-Id}]) %magenta([%thread]) - %msg%n"/>

  <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>${LOG_PATTERN}</pattern>
    </encoder>
  </appender>

  <root level="INFO">
    <appender-ref ref="CONSOLE"/>
  </root>

  <if condition='property("env").equalsIgnoreCase("test") || property("env").equalsIgnoreCase("prod")'>
    <then>
      <appender name="LOGSTASH" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
        <destination>${LOGSTASH_HOST:-localhost}:${LOGSTASH_PORT:-5000}</destination>
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
          <includeMdcKeyName>X-Request-Id</includeMdcKeyName>
          <includeMdcKeyName>traceId</includeMdcKeyName>
          <includeMdcKeyName>spanId</includeMdcKeyName>
          <customFields>{"application":"service-name"}</customFields>
        </encoder>
        <keepAliveDuration>5 minutes</keepAliveDuration>
      </appender>
      <root level="INFO">
        <appender-ref ref="LOGSTASH"/>
      </root>
    </then>
  </if>
</configuration>
```

### Adding Custom Fields (Logstash Encoder)

```java
import static net.logstash.logback.argument.StructuredArguments.kv;

// Fields appear as separate JSON keys
log.info("Order created",
    kv("orderId", order.getId()),
    kv("userId", user.getId()),
    kv("total", order.getTotal()),
    kv("step", "order_created")
);

// Output:
// {"message":"Order created","orderId":123,"userId":"u-456","total":99.99,"step":"order_created"}
```

---

## SLF4J Basics

### Logger Declaration

```java

@Slf4j
@Service
public class OrderService { }
```

### Parameterized Logging

```java
// ✅ GOOD: Evaluated only if level enabled
log.debug("Processing order {} for user {}", orderId, userId);

// ❌ BAD: Always concatenates
log.debug("Processing order " + orderId + " for user " + userId);

// ✅ For expensive operations
if (log.isDebugEnabled()) {
    log.debug("Order snapshot: {}", expensiveSerialize(order));
}
```

### Placeholders & Exceptions

```java
// ✅ Плейсхолдеры {}, аргументы по порядку — никакой конкатенации
log.info("Order {} created for user {}", orderId, userId);

// ✅ Исключение — последним аргументом, БЕЗ своего {}; SLF4J возьмёт stack trace
log.error("Order {} failed", orderId, exception);

// ❌ Не клади e.getMessage() в строку — потеряешь stack trace
log.error("Order failed: " + exception.getMessage());
```
