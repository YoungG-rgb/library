# Service & @Transactional

Правила сервисного слоя и транзакционных границ.

## Структура: интерфейс + impl

- Интерфейс — в пакете `<feature>.services`.
- Имплементация — в `<feature>.services.impl`, имя `<Interface>Impl`.
- См. [package-structure.md](package-structure.md) — `services` это слой внутри фичи, не корневой пакет.

```
<feature>/services/
├── OrderService.java          ← interface
├── PaymentService.java
└── impl/
    ├── OrderServiceImpl.java  ← @Service
    └── PaymentServiceImpl.java
```

## Аннотации класса

- `@Service` — на impl. Не на интерфейсе.
- `@Slf4j` — на impl, для логирования через Lombok.
- DI — через конструктор. Используй Lombok `@RequiredArgsConstructor` + `private final` поля. **Никаких `@Autowired` на полях.**

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class OrderServiceImpl implements OrderService {
    private final OrderRepository orderRepository;
    // ...
}
```

Если нужна сложная инициализация (читать `@Value`, парсить настройки, собирать ключи) — пиши конструктор руками без `@RequiredArgsConstructor`.

## @Transactional
### Где ставить

- **На сервисном слое**, на impl или на отдельных методах. Не на репозитории, не на контроллере.
- Если 90% методов сервиса пишут — ставь на класс. Для read-only методов в этом сервисе — переопредели локально:
  ```java
  @Transactional(readOnly = true)
  public Foo findFoo(...) { ... }
  ```
- Если сервис в основном read-only — `@Transactional(readOnly = true)` на класс, отдельные write-методы помечай `@Transactional` явно.

### Propagation

- **По умолчанию `REQUIRED`** — не указывай явно.
- **`REQUIRES_NEW`** — для side-effect'ов, которые должны коммититься независимо от внешней транзакции: архивирование, аудит-логи, попытки записи ошибок.
  ```java
  @Transactional(propagation = Propagation.REQUIRES_NEW)
  public void archive(Event event) { ... }
  ```
- `NESTED`, `SUPPORTS`, `MANDATORY` — не используем без отдельного обоснования в PR.

## Бизнес-логика

- Сервис — место для бизнес-логики. Контроллеры/консьюмеры — только маршалинг и вызов сервиса.
- Не вызывай репозитории других сущностей через сервис-сервис цепочки длиной > 2. Если нужна координация 3+ доменов — выдели отдельный orchestration-сервис.
- Маппинг сущность ↔ DTO — в сервисе или в отдельном `Mapper`-компоненте, **не в репозитории и не в контроллере**.

## Идемпотентность

Идемпотентность — обязательная часть бизнес-логики сервиса:

- Входящие события: дедуп по `event_id` (Redis / БД-таблица с TTL).
- Outbox-записи: стабильный `idempotency_key` + UNIQUE-индекс в БД.
- Денежные операции: UNIQUE на бизнес-ключе + стабильный `external_id` + pre-check под row-lock.

Каждый новый side-effect должен иметь свой idempotency-механизм.

## Логирование

- `log.info` — границы операций (получено, сохранено, отправлено).
- `log.debug` — дубликаты, skip-ы, детали.
- `log.warn` — нештатные, но восстановимые ситуации (stale event, отсутствие state).
- `log.error` — только с пробросом исключения или явным алертом.
- **PII (msisdn, PAN, email, паспорт) — никогда в логах в исходном виде.** Маскируй через утилиту проекта.

### MDC

MDC (Mapped Diagnostic Context) пробрасывает сквозной контекст в каждую строку логов — без повторения в каждом `log.info(...)`. Используется для корреляции записей одной операции/запроса/события.

- Подходящие ключи: `requestId`, `traceId`, `correlationId` — стабильные идентификаторы операции, можно задать.
- **НЕ клади в MDC**: PII в исходном виде (даже под другим именем ключа), payload'ы, объекты, изменчивые значения.
- Ставится один раз на границе операции (контроллер/консьюмер/scheduler-tick), не в каждом сервисном методе.
- **Всегда снимай в `finally`**, иначе значение протечёт в другие задачи на том же потоке (особенно с тред-пулами):
  ```java
  try {
      MDC.put("eventId", dto.getEventId());
      // business logic
  } finally {
      MDC.clear();
  }
  ```
  Или `MDC.clear()` на выходе из границы, если ставил несколько ключей.

### Правило для агента

**Не добавляй `MDC.put(...)` молча.** Если по ходу работы видишь место, где MDC улучшит трассируемость (новая границы операции, новый consumer, новый scheduler) — **спроси у пользователя**, какие ключи положить и нужно ли вообще. **Почему:** набор MDC-ключей — продуктовое решение (как потом фильтруют логи в Kibana/Grafana), а не техническое.

## Typed-handler registry: разбивка switch-case

Если сервис/процессор содержит `if/switch` на **4+ значений одного enum** — обязательно применить паттерн из 4 элементов ниже.

### 1. Аннотация `@Typed{EnumName}`

Имя = `Typed` + имя enum-класса. Одно поле `value()` того же enum-типа.

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface TypedOrderStatus {
    OrderStatus value();
}
```

### 2. Резолвер `{Domain}StateResolver`

Spring `@Component`. Строит `EnumMap` в конструкторе: обходит все бины базового типа через `ctx.getBeansOfType(...)`, разворачивает AOP-прокси через `AopProxyUtils.ultimateTargetClass()`, читает аннотацию. Дублирующая регистрация → `IllegalStateException` при старте.

Публичный метод: `resolve(EnumType) → Processor`.

### 3. Базовый процессор `{Domain}StageProcessor` (abstract)

Все общие `protected`-зависимости и хелперы — только здесь. Конкретные обработчики ничего лишнего не инжектируют.

Один абстрактный метод: `void process(BaseEvent event)`.

Класс не помечать `@Service` — только `abstract class`.

### 4. Конкретные обработчики

Имя: **`{Domain}{EventVerb}StageProcessor`**

`EventVerb` — бизнес-смысл **события**, которое вызывает переход, не имя enum-константы.

```
OrderCreatedStageProcessor     ← событие "создан заказ"
OrderCancelledStageProcessor   ← событие "отменён"
OrderCompletedStageProcessor   ← событие "выполнен"
OrderExpiredStageProcessor     ← событие "истёк"
```

Структура пакетов:
```
<feature>/services/
├── base_impl/
│   ├── OrderStageProcessor.java   ← abstract base
│   └── OrderStateResolver.java    ← registry
└── impl/
    ├── OrderCreatedStageProcessor.java
    ├── OrderCancelledStageProcessor.java
    └── ...
```

## Запреты

- `new MyServiceImpl(...)` вручную — никогда. Только через Spring DI.
- Статические зависимости — никогда. Если нужен утилити-метод без состояния — клади в `utilities`/`utils` как `@UtilityClass` (Lombok).
- `Thread.sleep` — никогда. Задержки — через outbox + диспатчер либо `@Scheduled`.
- Ручные `ExecutorService` в произвольных местах — нет. Если нужен пул для `@Scheduled` / `@Async` — заводи бин по правилам [scheduler.md](scheduler.md).

## Чек-лист перед PR

- [ ] Есть интерфейс + impl, impl в подпакете `.impl`.
- [ ] `@Service` только на impl.
- [ ] DI через `final` поля + конструктор (`@RequiredArgsConstructor` или ручной).
- [ ] `@Transactional` стоит на сервисном слое, не на репозитории.
- [ ] Если используется `REQUIRES_NEW` — это обосновано в PR-описании.
- [ ] Нет self-invocation между `@Transactional`-методами одного бина.
- [ ] Поймал исключение → не глотаешь молча, либо пробрасываешь, либо явный `setRollbackOnly`.
- [ ] Для нового side-effect добавлен idempotency-механизм.
- [ ] PII не попадает в логи в исходном виде.
- [ ] Switch/if на 4+ enum-значений → применён typed-handler registry.
- [ ] Аннотация `@Typed{EnumName}` есть на каждом конкретном обработчике.
- [ ] Резолвер бросает `IllegalStateException` на дублирующую регистрацию.
- [ ] Общие зависимости и хелперы — только в абстрактном базовом классе, не дублируются в impl.
- [ ] Имя конкретного обработчика отражает бизнес-событие, а не enum-константу.
