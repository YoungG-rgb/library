# Scheduler

Правила написания запланированных задач (`@Scheduled`).

## Структура пакетов

```
<feature>/schedulers/
├── FooScheduler.java         ← claim + dispatch (или прямая делегация)
└── tasks/
    └── FooTask.java          ← per-row логика, @Component @Scope(prototype)
```

`schedulers/tasks/` нужен только при фан-аут-варианте (см. ниже). Если scheduler делегирует в сервис без фан-аута — папки `tasks/` нет.

## Главное

- `@EnableScheduling` — один раз, на `@SpringBootApplication` или отдельном `@Configuration`.
- Расписание — **только cron** через `@Scheduled(cron = "${...}")`. Не используем `fixedDelay`/`fixedRate`.
- Метод scheduler-а называется `process()`, не `tick()`.
- `@Async` на scheduler-методе **не нужен**: тело тика делает быстрый claim + submit и сразу возвращается на дефолтный TaskScheduler-thread.
- Над классом scheduler-а — `@ConditionalOnExpression("${schedulers.<feature>.is-running} == 'true'")`. Если `is-running=false` или отсутствует — бин не создаётся.

## Обязательные пропсы scheduler-а

Все — под общим префиксом `schedulers.<feature-name>.*`, kebab-case. **Дефолтов в плейсхолдерах нет** — пропсы должны быть в `application.yaml` (там уже разрешено брать дефолты из env-переменных `UPPER_SNAKE` через `${ENV_NAME:default}`).

| Свойство                          | Обязательно                   | Назначение                                                                                              |
|-----------------------------------|-------------------------------|---------------------------------------------------------------------------------------------------------|
| `schedulers.<feature>.is-running` | да                            | feature flag для `@ConditionalOnExpression`. Если `false` — бин не создаётся, scheduler не запускается. |
| `schedulers.<feature>.cron`       | да, если `is-running=true`    | cron-выражение. Пустая строка / отсутствие → Spring отвергнет на старте, это сознательная защита.       |
| `schedulers.<feature>.batch-size` | да, если есть фан-аут task-ов | размер партии за тик.                                                                                   |

Пример блока в `application.yaml`:

```yaml
schedulers:
  foo:
    is-running: ${FOO_SCHEDULER_IS_RUNNING:false}
    cron: ${FOO_SCHEDULER_CRON:0 * * * * *}
    batch-size: ${FOO_SCHEDULER_BATCH_SIZE:100}
```

## Вариант A. Scheduler с фан-аутом (per-row Task)

Когда выбираем: за тик надо обработать партию строк, и **per-row работа содержит I/O** (REST к внешним сервисам, AMQP publish, и т.п.) → параллелим на виртуальных потоках.

### Scheduler

```java
@Component
@RequiredArgsConstructor
@ConditionalOnExpression(value = "'${schedulers.foo.is-running}' == 'true'")
public class FooScheduler {
    private final FooRepository fooRepository;
    private final ApplicationContext applicationContext;
    private final ExecutorService fooTaskExecutor;

    @Value("${schedulers.foo.batch-size}")
    private Integer batchSize;

    @Scheduled(cron = "${schedulers.foo.cron}")
    public void process() {
        fooRepository
                .findAndLockByStatus(OldStatus.NEW.name(), NewStatus.IN_PROGRESS.name(), batchSize)
                .stream()
                .map(state -> applicationContext.getBean(FooTask.class, state))
                .forEach(fooTaskExecutor::execute);
    }
}
```

Что важно:
- Claim+смена статуса — атомарно в одном SQL через CTE с `FOR UPDATE SKIP LOCKED + UPDATE … RETURNING` (см. пример в `ClientBonusStateRepository.findAndLockByStatus`). После коммита другие инстансы строки не подберут — статус уже не `NEW`.
- Task создаётся через `applicationContext.getBean(FooTask.class, state)` — Spring передаёт `state` в конструктор и автовайрит остальные зависимости.
- Никакой бизнес-логики в scheduler-е — только claim, обёртывание в Task, submit.

### Task

```java
@Slf4j
@Component
@Scope(SCOPE_PROTOTYPE)
@RequiredArgsConstructor
public class FooTask implements Runnable {
    private final FooEntity state;                              // конструктор-аргумент
    
    // ... остальные зависимости через @Autowired
    @Autowired private FooRepository fooRepository;             // смешанной инъекции — final только для state

    @Override
    public void run() {
        try {
            MDC.put("traceId", "traceId");
            // per-row логика; state приходит детачнутой (CTE-fetcher делает clearAutomatically=true),
            // bonusStateRepository.save(state) корректно мержит.
        } catch (Exception e) {
            log.error("FooTask failed for id={}", state.getId(), e);
        } finally {
            MDC.clear();
        }
    }
}
```

Правила Task:
- `@Component @Scope(SCOPE_PROTOTYPE)` (константа `org.springframework.beans.factory.config.BeanDefinition.SCOPE_PROTOTYPE`).
- Runtime-аргумент (entity) — через конструктор (`final` поле + `@RequiredArgsConstructor`).
- Остальные зависимости — `@Autowired` на полях. **Это исключение из правила «DI через конструктор»** — иначе нельзя смешать runtime-arg + Spring deps для prototype-бина.
- `run()` оборачивает работу в `TransactionTemplate.executeWithoutResult(...)` (`@Transactional` на не-Spring-вызываемом методе через прокси не сработал бы — но prototype-бин под прокси, поэтому `@Transactional` тоже допустим; `TransactionTemplate` короче и явнее).
- Ловим `Exception` в `run()` и логируем — иначе виртуальный поток упадёт молча.

### Executor

```java
@Bean("fooTaskExecutor")
public ExecutorService fooTaskExecutor() {
    return Executors.newVirtualThreadPerTaskExecutor();   // Java 21+
}
```

Для Java 17 и ниже — `ThreadPoolTaskExecutor` с `corePoolSize`/`maxPoolSize`/`queueCapacity` через `@Value` (см. ниже отдельный раздел).

## Вариант B. Scheduler без фан-аута (прямая делегация)

Когда выбираем: тик короткий, чисто DB-bound, фан-аут не оправдан. Scheduler просто вызывает существующий `@Transactional` метод сервиса.

```java
@Slf4j
@Component
@RequiredArgsConstructor
@ConditionalOnExpression(value = "${schedulers.bar.is-running} == 'true'")
public class BarScheduler {
    private final BarService barService;

    @Scheduled(cron = "${schedulers.bar.cron}")
    public void process() {
        try {
            barService.tick();
        } catch (Exception e) {
            log.error("BarScheduler process failed", e);
        }
    }
}
```

Никаких `ApplicationContext`, executor-ов, batch-size — этого всего нет, потому что нет фан-аута. Структурный shell тот же: пакет `schedulers/`, `@ConditionalOnExpression`, cron-проперть, метод `process()`.

## Executor (если фан-аут)

Один executor — один scheduler. Не шарь между разными фичами.

### Java 21+ → виртуальные потоки per task

```java
@Bean("fooTaskExecutor")
public ExecutorService fooTaskExecutor() {
    return Executors.newVirtualThreadPerTaskExecutor();
}
```

### Java 17 и ниже → `ThreadPoolTaskExecutor`

```java
@Bean("fooTaskExecutor")
public ThreadPoolTaskExecutor fooTaskExecutor(
        @Value("${FOO_CORE_POOL_SIZE:1}") int corePoolSize,
        @Value("${FOO_MAX_POOL_SIZE:1}") int maxPoolSize,
        @Value("${FOO_QUEUE_CAPACITY:0}") int queueCapacity) {
    var ex = new ThreadPoolTaskExecutor();
    ex.setCorePoolSize(corePoolSize);
    ex.setMaxPoolSize(maxPoolSize);
    ex.setQueueCapacity(queueCapacity);
    ex.setThreadNamePrefix("foo-task-");
    ex.initialize();
    return ex;
}
```

## Правила безопасности

- **Перекрытие тиков.** В рамках одного инстанса cron-`@Scheduled` не запустит новый тик, пока предыдущий не завершился. Так как `process()` сам не блокируется на работе тасок (только submit), это редко актуально — но если делегирующая `BarService.tick()` идёт дольше cron — это сигнал увеличить интервал.
- **Shutdown.** Для `ExecutorService`-бинов Spring сам вызовет `shutdown()` при остановке контекста. Не пиши `@PreDestroy`, если ничего нестандартного нет.
- **MDC.** Виртуальные потоки и пулы не наследуют MDC автоматически. Нужен `traceId` в логах задачи — ставь MDC внутри `Task.run()`, не рассчитывай на родительский поток.
- **Stuck `IN_PROGRESS`.** При фан-аут-варианте, если JVM упадёт между fetcher-commit и task-run, строки останутся в `IN_PROGRESS` навсегда. Документируй процедуру ручного восстановления (`UPDATE … paid_status='NEW' WHERE …`) в support-runbook.

## Координация нескольких инстансов

`@Scheduled` срабатывает **на каждом** инстансе. Координация — на стороне БД через `FOR UPDATE SKIP LOCKED` в claim-запросе репозитория. Каждый инстанс берёт свою партию, никто не дублирует.

## Чек-лист перед PR

- [ ] `@EnableScheduling` подключён один раз.
- [ ] Класс scheduler-а лежит в `<feature>/schedulers/`, имя — `<Feature>Scheduler`.
- [ ] Над классом — `@ConditionalOnExpression("${schedulers.<feature>.is-running} == 'true'")`.
- [ ] Метод scheduler-а — `process()`. Расписание — `@Scheduled(cron = "${schedulers.<feature>.cron}")`.
- [ ] Без `@Async` на scheduler-методе.
- [ ] В `application.yaml` заведён блок `schedulers.<feature>.{is-running,cron[,batch-size]}` с env-override через `${ENV_NAME:default}`.
- [ ] При фан-ауте: Task лежит в `<feature>/schedulers/tasks/`, помечен `@Component @Scope(SCOPE_PROTOTYPE)`, конструктор принимает только runtime-arg, остальные deps — `@Autowired` field-injection.
- [ ] При фан-ауте: per-row claim через CTE-запрос `FOR UPDATE SKIP LOCKED + UPDATE … RETURNING`, не «фетч + ручной UPDATE».
- [ ] При фан-ауте: один `ExecutorService`-бин на scheduler, виртуальные потоки (Java 21+) либо `ThreadPoolTaskExecutor` с параметрами через `@Value` (Java ≤17).
- [ ] Task оборачивает работу в `TransactionTemplate.executeWithoutResult(...)` и ловит `Exception` в `run()` (иначе виртуальный поток умрёт молча).
- [ ] MDC ставится внутри тика/таски, не наследуется с родительского потока.
