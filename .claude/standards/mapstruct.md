# MapStruct

Правила маппинга entity ↔ DTO через MapStruct.

## Когда использовать

- Маппинг entity → view/DTO, который представляет собой **чистое копирование полей** (1:1 или через один-два уровня вложенных ассоциаций).
- Как только ручной `toView`/`toDto` метод перестал быть тривиальным упражнением и начал дублироваться (одинаковый `EntityX → ViewX` собирается в двух разных сервисах) — выноси в `@Mapper`, не копируй метод дальше.

**Не используй MapStruct**, если маппинг несёт бизнес-логику:
- условное перезаписывание полей («не затирать существующее значение, если новое пустое»);
- guard-условия перед копированием (idempotency-проверки, «пропустить, если `updated <= stored`»);
- парсинг форматов (кастомный `DateTimeFormatter`, ручной URL-конкат), особенно с несколькими `null`-ветвлениями подряд.

Такой код остаётся обычным Java-методом (статическим маппером или методом сервиса) — генератор не должен прятать бизнес-правило внутри сгенерированного класса, который никто не читает при ревью.

**Почему:** MapStruct генерирует код по декларативным аннотациям; как только в маппинге появляется ветвление или побочная бизнес-логика, аннотации перестают быть декларативными (`@Named`/`expression = "java(...)"` превращает `@Mapper`-интерфейс в место, где спрятан код, который никто не видит при обычном чтении файла).

## Интерфейс, не класс

```java
@Mapper(componentModel = "spring")
public interface OrderMapper {

    OrderView toView(OrderEntity entity);
}
```

- Всегда `interface`, не абстрактный класс — реализацию генерирует annotation processor.
- `componentModel = "spring"` обязателен — сгенерированный `OrderMapperImpl` получает `@Component` и внедряется как обычный бин. Без этого параметра сгенерированный класс не будет виден Spring DI.
- Название — `<Feature>Mapper`, лежит в подпакете `mapper` фичи (см. [package-structure.md](package-structure.md), правило «`mapper` — если маппинг entity↔dto разрастается»).

## Вложенные ассоциации

Для поля, которое приходит через связь (`entity.getParent().getName()`), не пиши геттер-цепочку руками — используй `@Mapping` с dot-путём:

```java
@Mapping(target = "parentName", source = "parent.name")
ChildView toView(ChildEntity entity);
```

MapStruct сам вставит null-проверку на каждый шаг цепочки. Ручной эквивалент (`entity.getParent().getName()`) этого не делает и падает `NullPointerException`, если `parent` не заполнен — MapStruct в этом смысле безопаснее, а не просто короче.

## Коллекции

Для мэппинга `Collection<XxxEntity> → List<XxxView>` — отдельный метод в том же интерфейсе, не `.stream().map(mapper::toView).toList()` на каждом call site:

```java
List<OrderView> toViews(Collection<OrderEntity> entities);
```

Если из коллекции сущностей нужно вытащить не вью, а плоский набор значений (`Set<PermissionEntity> → Set<String>` кодов), не полагайся на автоматическое угадывание — опиши явный метод в том же интерфейсе с однозначной сигнатурой `Set<Entity> → Set<String>`, тогда MapStruct подхватит его для соответствующего `@Mapping` без `@Named`/`qualifiedByName`.

## Тесты

Без Spring-контекста получай инстанс маппера через фабрику, не `new XxxMapperImpl()` (генерируемое имя класса — деталь реализации, на неё не полагаемся) и не `new XxxMapper()` (это интерфейс, не создаётся):

```java
import org.mapstruct.factory.Mappers;

OrderMapper mapper = Mappers.getMapper(OrderMapper.class);
```

## Подключение зависимости (один раз на проект)

`mapstruct` + `mapstruct-processor` версия — параметр `mapstruct.version` в корневом `pom.xml`. Если в проекте уже используется Lombok на entity (`@Getter`/`@Setter`), `lombok-mapstruct-binding` обязателен в `annotationProcessorPaths` **между** `lombok` и `mapstruct-processor` — иначе MapStruct не видит Lombok-сгенерированные геттеры и собирает маппер с пустыми полями без ошибки компиляции.

```xml
<annotationProcessorPaths>
    <path>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <version>${lombok.version}</version>
    </path>
    <path>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok-mapstruct-binding</artifactId>
        <version>${lombok-mapstruct-binding.version}</version>
    </path>
    <path>
        <groupId>org.mapstruct</groupId>
        <artifactId>mapstruct-processor</artifactId>
        <version>${mapstruct.version}</version>
    </path>
</annotationProcessorPaths>
```

## Чек-лист перед PR

- [ ] Маппер — `interface` с `@Mapper(componentModel = "spring")`, не класс.
- [ ] В маппер вынесен только чистый copy-маппинг; логика (guard-условия, условная перезапись, парсинг) осталась обычным Java-кодом.
- [ ] Вложенные ассоциации — через `@Mapping(target = ..., source = "a.b.c")`, не ручную геттер-цепочку.
- [ ] Дублирующийся `Entity → View` маппинг в нескольких сервисах вынесен в один `@Mapper`, а не переписан второй раз.
- [ ] Тесты получают маппер через `Mappers.getMapper(XxxMapper.class)`, не `new`.
