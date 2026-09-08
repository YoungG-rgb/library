# Application Config (application.yml)

Правила оформления конфигурации Spring Boot (`application.yml` и профильные файлы).

Формат — **YAML**, не `.properties`. **Почему:** вложенность читается лучше плоских ключей, поддерживает многострочные значения и группировку по неймспейсам.

## Структура файлов

Разбивай конфиг **по концернам (технологиям), а не по окружениям**. Один файл = одна инфраструктурная зона.

```
src/main/resources/
├── application.yml                 # master: server, spring.application, profiles.include, общее
├── application-database.yml        # datasource, jpa, hikari, redis
├── application-amqp.yml            # rabbit / amqp
├── application-kafka.yml           # kafka
├── application-connectors.yml      # внешние REST-коннекторы
└── application-management.yml      # actuator / metrics
```

```yaml
# application.yml  // ОК
spring:
  profiles:
    include: >
      database,
      amqp,
      kafka,
      connectors,
      management
```

- Имя профиля = имя концерна (`database`, `amqp`, `kafka`), **не** окружения (`dev`, `prod`). **Почему:** окружение задаётся значениями переменных, а не отдельным набором файлов — так конфиг не расходится между стендами.

## Плейсхолдеры и переменные окружения

- Каждое значение, которое меняется между стендами, — через `${ENV_VAR:default}`. Константы (driver-class, dialect, media-types) — литералом.
- Имя переменной — `SCREAMING_SNAKE_CASE`, по смыслу ключа: `DB_HOST`, `HIKARI_MAX_POOL_SIZE`, `BANK_ACCOUNT_RESPONSE_TOPIC_NAME`.
- Дефолт после `:` — значение для **локального запуска**. Оно попадает в git, поэтому в дефолт кладём только безопасные вещи: локальные хосты/порты, таймауты, имена топиков, тогглы.

```yaml
url: jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:app_local}  // ОК
maximum-pool-size: ${HIKARI_MAX_POOL_SIZE:20}                                     // ОК
```

## Чувствительные данные — без дефолта

Секреты **никогда не имеют дефолтного значения**. Только `${VAR}` — пусть приложение падает на старте, если переменная не задана.

К секретам относятся: пароли, `secret-key` / `access-key`, `api-key` / `secure-key`, токены, `hash-key`, логины/пароли внешних API, строки подключения с встроенными кредами.

```yaml
# ПЛОХО — секрет с дефолтом уезжает в git
spring.datasource.password: ${DB_PASSWORD:123password}
minio.secret-key: ${MINIO_SECRET_KEY:19238H2H9DSADU929BDSAIDN12}

# ОК — секрет обязателен, значения в файле нет
spring.datasource.password: ${DB_PASSWORD}
minio.secret-key: ${MINIO_SECRET_KEY}
externals.connectors.rest.bankApiRestTemplate.password: ${BANK_API_PASSWORD}
```

**Почему:** дефолт секрета = утёкший секрет. Он навсегда остаётся в истории git, даже если позже его удалить. Отсутствие дефолта делает пропуск переменной ошибкой конфигурации, а не тихой работой с боевым ключом.

- Не-секретные, но окруженческие значения (хосты, порты) дефолт иметь **могут** — но не боевые адреса, а локальные (`localhost`, `127.0.0.1`).

## Кастомные неймспейсы и `@ConfigurationProperties`

- Свои настройки клади в собственный корневой неймспейс (`rabbit`, `amqp`, `externals`, `schedulers`, `<app>`), а не подмешивай в `spring.*`. **Почему:** `spring.*` — зона Spring Boot; свои ключи там конфликтуют с автоконфигурацией и не находятся в документации.
- Каждый неймспейс маппится на типизированный `@ConfigurationProperties`-класс, **не** на россыпь `@Value`. **Почему:** один класс — один источник правды, валидация через `@Validated`, IDE-автокомплит.
- Ключи — `kebab-case` (`max-file-size`, `read-timeout`, `is-running`). **Почему:** relaxed binding Spring корректно свяжет `read-timeout` с полем `readTimeout`.

## Фиче-тогглы

- Включение/выключение consumer'ов и шедулеров — через булев ключ `is-running` с дефолтом `false`.

```yaml
schedulers:
  canceling:
    is-running: ${CANCELING_IS_RUNNING:false}   // ОК — по умолчанию выключено
```

**Почему:** дефолт `false` безопасен — новый инстанс не начнёт молча обрабатывать очередь/шедулер, пока его явно не включат на стенде.

## Читаемость

- Большие числа — с разделителем `_`: `10_000`, `1_800_000`. **Почему:** `1800000` глазами не читается.
- Числовые таймауты/интервалы комментируй единицами: `connectionTimeout: 30000 # 30s`.
- Порядок внутри файла: сверху — то, что чаще правят (хосты, креды, пулы), ниже — тюнинг-константы.

## Запрещено

- Секрет с дефолтным значением (см. выше) — **жёсткий флаг ревьюера**.
- Боевые хосты/IP/URL в дефолтах плейсхолдеров — только локальные.
- Свои ключи внутри `spring.*`.
- Россыпь `@Value("${...}")` вместо `@ConfigurationProperties` для связанной группы настроек.
- `.properties` для новой конфигурации — только YAML.
- Дублирование одного значения в нескольких профильных файлах — вынеси в master или в один профиль.
