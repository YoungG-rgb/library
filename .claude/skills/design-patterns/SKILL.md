---
name: design-patterns
description: Common design patterns with Java examples (Factory, Builder, Strategy, Observer, Decorator, etc.). Use when user asks "implement pattern", "use factory", "strategy pattern", or when designing extensible components. Триггеры RU - «применить паттерн», «фабрика», «стратегия», «декоратор», «typed-handler», «registry».
---

## ⚠️ Project Standards Override

Если в проекте есть `.claude/standards/` — следуй им. Особенно при выборе паттернов:

- **Builder в JPA-сущностях запрещён** (см. `.claude/standards/jpa-entity.md`). Используй `@Accessors(chain = true)` + сеттеры: `new OrderEntity().setX(...).setY(...)`.
- **Switch/if на 4+ enum-значений → typed-handler registry**, а не generic Strategy. Каноничная реализация описана в `.claude/standards/service-transactional.md` (раздел "Typed-handler registry"). Структура:
  - `@TypedXxxStatus` аннотация над конкретным обработчиком,
  - `XxxStateResolver` — Spring `@Component` со собственным `EnumMap`, дедуп через `IllegalStateException` на старте,
  - `XxxStageProcessor` (abstract) — общие зависимости и хелперы,
  - `Xxx{EventVerb}StageProcessor` — конкретные обработчики, имя по бизнес-событию, не по enum-константе.
- **Singleton/`new` сервисов** — никогда. Только Spring DI.
- **Spring Events / `ApplicationEventPublisher`** предпочтительнее ручного Observer.
- **Factory через Spring** (мапа бинов по типу в конструкторе) — предпочтительнее `static` фабрики.

---

# Design Patterns Skill

Quick reference for common design patterns in Java.

## When to Use
- User asks to implement a specific pattern
- Designing extensible/flexible components
- Refactoring rigid code

## Quick Reference: When to Use What

| Problem                                 | Pattern       | Use When                        |
|-----------------------------------------|---------------|---------------------------------|
| Complex object construction             | **Builder**   | Many parameters, some optional  |
| Create objects without specifying class | **Factory**   | Type determined at runtime      |
| Multiple algorithms, swap at runtime    | **Strategy**  | Behavior varies by context      |
| Add behavior without changing class     | **Decorator** | Dynamic composition needed      |
| Notify multiple objects of changes      | **Observer**  | One-to-many dependency          |
| Convert incompatible interfaces         | **Adapter**   | Integrate legacy/3rd party code |

---

## Creational Patterns

### Builder
**Problem:** Telescoping constructors, many optional parameters

```java
// ✅ Builder pattern
@Data
@NoArgsConstructor
@AllArgsConstructor
@Accessors(chain = true)
@FieldDefaults(level = AccessLevel.PRIVATE)
static class DatasourceProperties {
  String poolName;    // optional
  String url;         // required

  @ToString.Exclude
  String password;    // required

  Map<String, Object> additionalProperties = new HashMap<>();

  public String getPoolNameOrDefault() {
    return this.poolName == null ? ("HikariPool-" + UUID.randomUUID()) : this.poolName;
  }

  public void addAdditionalData(String key, Object val) {
    this.additionalProperties.put(key, val);
  }
}

// Usage
DatasourceProperties properties = new DatasourceProperties()
        .setUrl("databaseUrl")
        .setPassword("password")
        .setPoolName("Hikari-poolName")
        .addAdditionalData("key", 1);
```

> ⚠️ `@Data` здесь допустим, потому что это **обычный value/config-объект вне persistence**. **На JPA-сущностях `@Data` НЕ применять** — его `equals`/`hashCode`/`toString` по всем полям ломают identity-семантику и вызывают LazyInitializationException в `toString`. Для сущностей — `@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Accessors(chain = true)` (см. `.claude/standards/jpa-entity.md`). `@Data` уместен только на immutable value-объектах / DTO вне JPA.

### Factory
**Problem:** Create objects without knowing exact class upfront

```java
public interface Notification {
    void send(String message);
}

// ❌ Статическая фабрика — избегаем: жёсткая связка, не тестируется, нет DI,
//    добавление типа = правка switch. Приемлема только вне Spring-контекста.
public class NotificationFactory {
    public static Notification create(String type) {
        return switch (type.toUpperCase()) {
            case "EMAIL" -> new EmailNotification();
            case "SMS" -> new SmsNotification();
            case "PUSH" -> new PushNotification();
            default -> throw new IllegalArgumentException("Unknown: " + type);
        };
    }
}

// ✅ Spring-фабрика — предпочтительно: мапа бинов по типу, новый тип = новый @Component,
//    без правки фабрики (см. service-transactional.md, раздел typed-handler registry).
@Component
public class NotificationSenderFactory {
    private final Map<String, NotificationSender> senders;

    public NotificationSenderFactory(List<NotificationSender> senderList) {
        this.senders = senderList.stream()
            .collect(Collectors.toMap(
                NotificationSender::getType,
                Function.identity()
            ));
    }

    public NotificationSender get(String type) {
        return Optional.ofNullable(senders.get(type))
            .orElseThrow(() -> new IllegalArgumentException("Unknown: " + type));
    }
}
```

---

## Behavioral Patterns

### Strategy
**Problem:** Multiple algorithms for same operation, choose at runtime

```java
// ✅ Strategy pattern (SAM-интерфейс — годится и для лямбд)
@FunctionalInterface
public interface PaymentStrategy {
    void pay(BigDecimal amount);
}

public class CreditCardPayment implements PaymentStrategy {
    private final String cardNumber;

    @Override
    public void pay(BigDecimal amount) {
        System.out.println("Paid " + amount + " with card");
    }
}

public class ShoppingCart {
    private PaymentStrategy paymentStrategy;

    public void setPaymentStrategy(PaymentStrategy strategy) {
        this.paymentStrategy = strategy;
    }

    public void checkout(BigDecimal total) {
        paymentStrategy.pay(total);
    }
}

// Usage
cart.setPaymentStrategy(new CreditCardPayment("4111..."));
cart.checkout(new BigDecimal("99.99"));

// Functional variant (Java 8+): PaymentStrategy уже SAM — передавай лямбду,
// не переобъявляй интерфейс.
PaymentStrategy creditCard = amount -> System.out.println("Card: " + amount);
cart.setPaymentStrategy(creditCard);
```

### Observer
**Problem:** Notify multiple objects when state changes

```java
// ✅ Spring Events (preferred)
public record OrderPlacedEvent(Order order) {}

@Service
public class OrderService {
    private final ApplicationEventPublisher eventPublisher;

    public void placeOrder(Order order) {
        saveOrder(order);
        eventPublisher.publishEvent(new OrderPlacedEvent(order));
    }
}

@Component
public class InventoryListener {
    @EventListener
    public void handleOrderPlaced(OrderPlacedEvent event) {
        // Reduce inventory
    }
}

@Component
public class EmailListener {
    @EventListener
    @Async
    public void handleOrderPlaced(OrderPlacedEvent event) {
        // Send email
    }
}
```

> ⚠️ `@Async` работает только при наличии `@EnableAsync` в конфигурации приложения — без него аннотация **молча игнорируется** и слушатель выполняется синхронно в потоке публикатора. Убедись, что где-то есть `@Configuration @EnableAsync`.

### Typed-Handler Registry

**Problem:** `if/switch` на 4+ значений одного enum, каждая ветка — своя логика. Generic Strategy тут заменяется registry, где Spring сам собирает обработчики.

**Каноничная реализация в проекте — `.claude/standards/service-transactional.md` (раздел «Typed-handler registry»).** Минимальный скелет:

```java
// 1. Контракт обработчика — типизирован по enum-значению
public interface OrderHandler {
    OrderStatus supports();          // за какое значение отвечает
    void handle(OrderEvent event);
}

// 2. Конкретные обработчики — по одному @Component на значение,
//    имя по бизнес-событию, не по enum-константе
@Component
class OrderCreatedHandler implements OrderHandler {
    public OrderStatus supports() { return OrderStatus.CREATED; }
    public void handle(OrderEvent event) { /* ... */ }
}

// 3. Resolver — Spring инжектит List<OrderHandler>, собираем в EnumMap
@Component
public class OrderHandlerRegistry {
    private final Map<OrderStatus, OrderHandler> handlers;

    public OrderHandlerRegistry(List<OrderHandler> beans) {
        this.handlers = new EnumMap<>(OrderStatus.class);
        for (OrderHandler h : beans) {
            if (handlers.putIfAbsent(h.supports(), h) != null) {
                // дублирующая регистрация → падаем на старте, а не в рантайме
                throw new IllegalStateException("Duplicate handler for " + h.supports());
            }
        }
    }

    public OrderHandler resolve(OrderStatus status) {
        OrderHandler h = handlers.get(status);
        if (h == null) throw new IllegalArgumentException("No handler for " + status);
        return h;
    }
}
```

**Почему:** новый тип = новый `@Component`, без правки `switch`. Дедуп через `IllegalStateException` ловит коллизии при старте. Общие зависимости выноси в абстрактный базовый обработчик (см. стандарт).

---

## Structural Patterns

### Decorator
**Problem:** Add behavior dynamically without modifying class

```java
// ✅ Decorator pattern
public interface Coffee {
    String getDescription();
    BigDecimal getCost();
}

public class SimpleCoffee implements Coffee {
    public String getDescription() { return "Coffee"; }
    public BigDecimal getCost() { return new BigDecimal("2.00"); }
}

public abstract class CoffeeDecorator implements Coffee {
    protected final Coffee coffee;
    public CoffeeDecorator(Coffee coffee) { this.coffee = coffee; }
}

public class MilkDecorator extends CoffeeDecorator {
    public MilkDecorator(Coffee coffee) { super(coffee); }

    public String getDescription() {
        return coffee.getDescription() + ", Milk";
    }

    public BigDecimal getCost() {
        return coffee.getCost().add(new BigDecimal("0.50"));
    }
}

// Usage
Coffee coffee = new SimpleCoffee();
coffee = new MilkDecorator(coffee);
coffee = new SugarDecorator(coffee);
```

### Adapter
**Problem:** Make incompatible interfaces work together

```java
// ✅ Adapter pattern
public interface MediaPlayer {
    void play(String filename);
}

// Legacy code
public class LegacyAudioPlayer {
    public void playMp3(String filename) { /* ... */ }
}

// Adapter
public class Mp3PlayerAdapter implements MediaPlayer {
    private final LegacyAudioPlayer legacyPlayer = new LegacyAudioPlayer();

    @Override
    public void play(String filename) {
        legacyPlayer.playMp3(filename);
    }
}

// Usage
MediaPlayer player = new Mp3PlayerAdapter();
player.play("song.mp3");
```

---

## Pattern Selection Guide

| Situation                             | Pattern          |
|---------------------------------------|------------------|
| Object creation is complex            | Builder, Factory |
| Need to add features dynamically      | Decorator        |
| Multiple implementations of algorithm | Strategy         |
| React to state changes                | Observer         |
| Integrate with legacy code            | Adapter          |

## Anti-Patterns to Avoid

| Anti-Pattern          | Problem                    | Better Approach                |
|-----------------------|----------------------------|--------------------------------|
| Singleton abuse       | Global state, hard to test | Dependency Injection           |
| Factory everywhere    | Over-engineering           | Simple `new` if type known     |
| Deep decorator chains | Hard to debug              | Composition, keep chains short |
