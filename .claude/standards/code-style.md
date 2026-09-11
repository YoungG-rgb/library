# Code Style (Checkstyle)

Единый стиль Java-кода проверяется Checkstyle. Конфиг — `checkstyle.xml` в корне репозитория
(база — Google Java Style с двумя проектными правками). Стандарт описывает конфиг и как его подключать.

## Конфиг

- Источник правды — **`checkstyle.xml`** в корне. Не форкай правила по модулям, не держи второй конфиг.
- База — официальный `google_checks.xml` (Checkstyle distribution). **Не «стоковый Google»** — есть проектные отклонения (ниже). Имя файла — `checkstyle.xml`, не `google_checks.xml`.
- Проектные правки относительно стока Google:
  - **Отступ — 4 пробела** (`Indentation` = 4; в стоке Google 2). Совпадает с [jpa-entity.md](jpa-entity.md)/примерами стандартов.
  - **Длина строки — 145** (`LineLength max=145`; в стоке 100). Табы запрещены (`FileTabCharacter`).

## Требования к окружению

- **Checkstyle ≥ 10.18.** Конфиг использует токены `LITERAL_WHEN` (guarded patterns), `RECORD_PATTERN_DEF`, `ConstructorsDeclarationGrouping`.
  - **Почему:** на более старой версии конфиг не загрузится вообще — Checkstyle упадёт на unknown token, а не «пропустит правило».
- Suppression-файлы (`checkstyle-suppressions.xml`, `checkstyle-xpath-suppressions.xml`) — опциональны (`optional=true`). Заводи только когда реально нужны точечные подавления; их отсутствие не ломает сборку.

## Подключение (обязательно)

Конфиг без плагина ничего не проверяет. Заведи checkstyle в сборке и **гейти сборку**.

### Maven

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-checkstyle-plugin</artifactId>
  <version>3.6.0</version>
  <dependencies>
    <dependency>
      <groupId>com.puppycrawl.tools</groupId>
      <artifactId>checkstyle</artifactId>
      <version>10.21.0</version>   <!-- ≥ 10.18 -->
    </dependency>
  </dependencies>
  <configuration>
    <configLocation>checkstyle.xml</configLocation>
    <violationSeverity>warning</violationSeverity>   <!-- иначе warning'и не гейтят -->
    <failOnViolation>true</failOnViolation>
    <includeTestSourceDirectory>true</includeTestSourceDirectory>
  </configuration>
  <executions>
    <execution>
      <id>checkstyle</id>
      <phase>validate</phase>
      <goals><goal>check</goal></goals>
    </execution>
  </executions>
</plugin>
```

### Gradle

```groovy
plugins { id 'checkstyle' }

checkstyle {
    toolVersion = '10.21.0'            // ≥ 10.18
    configFile = file("$rootDir/checkstyle.xml")
    maxWarnings = 0                    // иначе warning'и не гейтят
}
```

- **`violationSeverity=warning` / `maxWarnings=0` — обязательно.** В конфиге `severity` по умолчанию `warning`; без этой настройки нарушения не валят сборку и линтер бесполезен.
  - **Почему:** «конфиг есть, но ничего не гейтит» — худший вариант: стиль дрейфует, а все думают, что он под контролем.
- Проверяй и тесты (`includeTestSourceDirectory` / стандартная gradle-задача `checkstyleTest`).

## Форматтер IDE

- Настрой code style IntelliJ **под конфиг** (или приложи `.editorconfig` в корень). Ключевое расхождение с дефолтом IDEA: `OperatorWrap=NL` и `SeparatorWrap` для `.`/method-ref требуют перенос **перед** оператором/точкой, а IDEA по умолчанию переносит после.
  - **Почему:** без синхронизации форматтера каждый reformat порождает поток нарушений — команда начнёт глушить правила вместо соблюдения.

## Взаимодействие с Lombok

- Checkstyle работает по исходникам, сгенерированный Lombok-код не проверяется — геттеры/сеттеры/конструкторы правила стиля не задевают.
- Строгие Javadoc-проверки (`SummaryJavadoc`, `JavadocParagraph`, `MissingJavadocMethod` для `public`) на Lombok/Spring-коде дают шум. Не отключай их глобально из-за шума — подавляй точечно через suppression-файл или `// CHECKSTYLE.SUPPRESS`.

## Чек-лист

- [ ] Файл называется `checkstyle.xml`, лежит в корне, он единственный источник правил.
- [ ] В сборке подключён checkstyle-плагин с `configLocation=checkstyle.xml`.
- [ ] `violationSeverity=warning` (Maven) / `maxWarnings=0` (Gradle) — сборка падает на нарушениях.
- [ ] Версия Checkstyle ≥ 10.18 зафиксирована в плагине.
- [ ] Проверяются и main-, и test-исходники.
- [ ] IDE-форматтер / `.editorconfig` согласованы с правилами (перенос перед оператором/точкой, отступ 4, строка ≤145).
- [ ] Подавления — только точечные (suppression-файл или `CHECKSTYLE.SUPPRESS`), не глобальное отключение проверок.
