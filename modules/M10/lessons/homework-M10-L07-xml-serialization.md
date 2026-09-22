---
[← К уроку M10-L07](lesson-M10-L07-xml-serialization.md) | [⬆ К модулю M10](../README.md) | [Следующее ДЗ →](homework-M10-L08-binary-formats.md)
---

### Домашнее задание M10-L07: XML-сериализация (Optional/Deprecated-вектор) / Homework M10-L07: XML serialization (Optional/legacy vector)

**Урок / Lesson:** M10-L07
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться применять `XmlSerializer` для двустороннего обмена XML-документами с legacy-системой, кэшировать сериализатор статически, управлять контрактом через атрибуты и обеспечивать безопасную десериализацию с запрещённым DTD; сравнить подход с `DataContractSerializer`. (EN) Learn to apply `XmlSerializer` for bidirectional XML exchange with a legacy system, cache the serializer statically, control the contract via attributes, and ensure safe deserialization with DTD prohibited; compare the approach with `DataContractSerializer`.

#### Связь с уроком / Connection to the lesson
(RU) Домашнее задание напрямую закрепляет ядро урока M10-L07: атрибуты `[XmlElement]`, `[XmlAttribute]`, `[XmlArray]`/`[XmlArrayItem]`, `[XmlIgnore]`, статическое кэширование `XmlSerializer`, безопасные настройки `XmlReaderSettings` (`DtdProcessing = Prohibit`) и осознанный выбор XML только для legacy/SOAP/XSD-контрактов. Вы также сравните `XmlSerializer` с `DataContractSerializer`, как требует раздел теории урока.
(EN) This homework directly reinforces the core of lesson M10-L07: the attributes `[XmlElement]`, `[XmlAttribute]`, `[XmlArray]`/`[XmlArrayItem]`, `[XmlIgnore]`, static caching of `XmlSerializer`, safe `XmlReaderSettings` (`DtdProcessing = Prohibit`), and a deliberate choice of XML only for legacy/SOAP/XSD contracts. You will also compare `XmlSerializer` with `DataContractSerializer`, as the theory section requires.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Вы — backend-разработчик в компании «ТрансКонтур», которая запускает интеграцию с legacy-системой партнёра «AcmeLogistics». Партнёр обменивается документами по протоколу, появившемуся ещё в эпоху SOAP/WSDL, и обмениваться можно только XML-манифестами отгрузок, форма которых зафиксирована XSD-схемой, доступной только в виде распечатки 2009 года. JSON партнёр не поддерживает, потому что их шина сообщений построена на BizTalk, а gateway ожидает именно XML с конвертом и пространствами имён. Ваша задача — реализовать на C# 12 / .NET 8 модуль `ShipmentXmlGateway`, который умеет сериализовать доменный объект `ShipmentManifest` в XML строго по контракту и десериализовать ответ обратно, не падая на «грязных» данных и не открывая уязвимость XXE.

Урок M10-L07 помечен как Optional/Deprecated-вектор именно для таких ситуаций: вы должны уметь поддерживать и мигрировать legacy-код, но не начинать новые проекты с XML без веской причины. Здесь причина есть — внешний контракт с XSD, и вы честно документируете её в комментарии к классу. Это упражнение также проверяет, что вы понимаете разницу между `XmlSerializer` (публичные свойства, тонкая настройка через атрибуты, медленный первый вызов из-за генерации временной сборки) и `DataContractSerializer` (работает с `[DataContract]`/`[DataMember]`, поддерживает графы объектов, быстрее, но менее «читаемый» XML). По ходу задания вы реализуете основной путь на `XmlSerializer`, а в бонусе — покажете, как тот же граф с циклической ссылкой сериализуется через `DataContractSerializer` с `preserveObjectReferences`.

Особое внимание уделяется безопасности. Партнёр присылает XML, который вы не контролируете полностью, поэтому десериализация обязана идти через `XmlReader` с `DtdProcessing = DtdProcessing.Prohibit`. Вы не должны использовать перегрузки `XmlSerializer`, принимающие типы из самого XML, и должны явно ограничить набор типов. Это защищает от векторов, описанных в уроке: инъекции внешних сущностей (XXE) и атак через `XmlSerializerFactory`.

#### Что нужно сделать (пошагово)

1. Создайте консольный проект .NET 8 с именем `TransContour.Gateway`:
   ```
   dotnet new console -n TransContour.Gateway -o TransContour.Gateway
   cd TransContour.Gateway
   dotnet new sln -n TransContour.Gateway
   dotnet sln add TransContour.Gateway.csproj
   ```
   Целевая платформа — `net8.0`, язык — C# 12 (top-level statements разрешены и рекомендуются для точки входа `Program.cs`).

2. В файле `Models/ShipmentManifest.cs` опишите доменную модель. Класс `ShipmentManifest` содержит: идентификатор манифеста (`ManifestId`, `string`), дату формирования (`CreatedAt`, `DateTimeOffset`), отправителя (`Sender`, вложенный объект `Party`), получателя (`Receiver`, вложенный `Party`), список позиций (`Lines`, `List<ShipmentLine>`), общую стоимость (`TotalAmount`, `decimal`) и служебное поле `InternalCacheKey`, которое **не должно** попадать в XML. Используйте атрибуты `[XmlAttribute]` для идентификатора и даты, `[XmlElement]` для отправителя/получателя/суммы с явным `Order`, `[XmlArray]` + `[XmlArrayItem]` для позиций, и `[XmlIgnore]` для служебного поля. Задайте корневое имя и пространство имён через `[XmlRoot("manifest", Namespace = "urn:acme:logistics:manifest:v1")]`.

3. Класс `Party` должен иметь атрибут `[XmlType("party")]` и свойства `Code` (атрибут `code`), `Name` (элемент `name`), `Country` (элемент `country`). Класс `ShipmentLine` — `[XmlType("line")]` с `Sku` (атрибут), `Qty` (элемент), `UnitPrice` (элемент) и вычисляемым свойством `LineTotal` (только для чтения, `Qty * UnitPrice`), которое **обязательно** помечается `[XmlIgnore]`, иначе оно попадёт в XML и нарушит контракт, как предостерегает раздел «Частые ошибки» урока.

4. В файле `Serialization/ShipmentXmlGateway.cs` реализуйте статический класс `ShipmentXmlGateway` с двумя методами: `string Serialize(ShipmentManifest manifest)` и `ShipmentManifest Deserialize(string xml)`. Кэшируйте `XmlSerializer` статически (`private static readonly XmlSerializer _serializer = new(typeof(ShipmentManifest));`) — это требование урока, иначе вы платите за генерацию временной сборки при каждом вызове. В `Serialize` используйте `StringWriter` + `XmlWriter` с `XmlWriterSettings { Indent = true, OmitXmlDeclaration = false }`. В `Deserialize` создавайте `XmlReader` с `XmlReaderSettings { DtdProcessing = DtdProcessing.Prohibit }` — это защищает от XXE.

5. В `Program.cs` постройте образец `ShipmentManifest` с двумя позициями, сериализуйте его и выведите XML в консоль. Затем десериализуйте строку обратно и выведите восстановленные поля. Убедитесь, что `InternalCacheKey` отсутствует в XML, а `LineTotal` не сериализуется. Ожидаемый вывод должен содержать корневой тег `<manifest xmlns="urn:acme:logistics:manifest:v1">`, атрибуты `id` и `created-at`, элементы `<sender>`, `<receiver>`, `<total>`, массив `<lines>` с вложенными `<line>`.

6. Добавьте файл `Tests/RoundTripTests.cs` (можно через xUnit: `dotnet add package xunit xunit.runner.visualstudio`). Напишите тесты: (a) сериализация + десериализация возвращает эквивалентный объект; (b) XML содержит атрибут `id` и не содержит `InternalCacheKey`; (c) десериализация XML с `<!DOCTYPE ... SYSTEM "http://evil/evil.dtd">` выбрасывает исключение (XXE-проверка); (d) пустые и `null` коллекции обрабатываются корректно.

7. Запустите: `dotnet test`. Должно быть 4 зелёных теста. Запустите `dotnet run --project TransContour.Gateway` и приложите вывод к сдаче.

8. В комментариях к классу `ShipmentXmlGateway` явно укажите причину выбора XML: «legacy-контракт с XSD, партнёр AcmeLogistics, протокол BizTalk — JSON недоступен». Это закрывает требование урока документировать rationale.

#### Требования к решению

- Целевая платформа `net8.0`, язык C# 12; в `Program.cs` используйте top-level statements, collection expressions для инициализации списков (`new()` + инициализатор коллекции или `Items = [..]` где уместно).
- Используйте `XmlSerializer` из `System.Xml.Serialization`; не пишите собственный парсер XML.
- Сериализатор обязан быть статически кэширован — не создавайте `new XmlSerializer(typeof(ShipmentManifest))` внутри методов. Урок явно называет это частой ошибкой.
- Десериализация обязана идти через `XmlReader` с `DtdProcessing = DtdProcessing.Prohibit`. Использование `StringReader` без `XmlReader` и без настроек — принимается только в «доверенной» ветке, но в задании данные партнёра считаются недоверенными, поэтому настройки обязательны.
- Все свойства, не входящие в контракт (вычисляемые, служебные, кэш-ключи), помечаются `[XmlIgnore]`. Особенно `LineTotal` — это вычисляемое свойство, утечка которого в XML ломает валидацию по XSD.
- Корневой элемент обязан иметь пространство имён `urn:acme:logistics:manifest:v1` — это часть контракта, без него партнёр отклонит документ.
- Код компилируется без предупреждений `nullable`. Включите `<Nullable>enable</Nullable>` в `.csproj` и обрабатывайте `null` осознанно (например, `string.Empty` по умолчанию для строк).
- Тесты используют xUnit, покрывают round-trip, отсутствие служебных полей и XXE-защиту. Запрещено отключать тест через `Assert.True(true)` и подобные заглушки.

#### Тонкости и подводные камни

- **Статическое кэширование обязательно.** Каждый `new XmlSerializer(typeof(T))` на первом обращении генерирует временную сборку через `XmlSerializerCompiler`. В горячем пути это катастрофа по скорости и фрагментирует память. Урок называет это первой частой ошибкой — кэшируйте в `static readonly`.
- **`XmlSerializer` видит только публичные члены.** Если вы пометите свойство `private` или `internal` — оно не сериализуется, и `DataMember`-стиль здесь не работает. Для приватных полей нужен `DataContractSerializer`, что и является темой бонуса.
- **XXE через DTD.** Без `DtdProcessing = Prohibit` (или хотя бы `Parse` с отключённым резолвером) парсер может загрузить внешний DTD с `file://` или `http://`, что ведёт к утечке файлов (классический XXE). В .NET 8+ по умолчанию настройки безопаснее, чем в .NET Framework, но полагаться на дефолты нельзя — задавайте `XmlReaderSettings` явно.
- **Не используйте `XmlReaderSettings` с `DtdProcessing.Parse` и `XmlResolver` по умолчанию** — это открывает вектор. В уроке указано: «отключайте `DtdProcessing`».
- **Вычисляемые свойства «утекают».** `LineTotal` без `[XmlIgnore]` попадёт в XML как элемент `<LineTotal>`, и партнёр, валидирующий по XSD, отклонит документ. Это вторая частая ошибка из урока.
- **`[XmlArray]` vs `[XmlElement]` для коллекций.** `[XmlArray("lines")] + [XmlArrayItem("line")]` даёт `<lines><line/><line/></lines>`. Если вы хотите плоский список `<line/><line/>` без обёртки, используйте `[XmlElement("line")]` прямо на свойстве-коллекции. Контракт AcmeLogistics требует обёртки, поэтому выбран `XmlArray`.
- **`Order` в `[XmlElement]` фиксирует последовательность.** XSD часто описывает `sequence`, и нарушение порядка (sender после receiver) невалидно. Указывайте `Order` явно.
- **`null` коллекция сериализуется как `<lines xsi:nil="true"/>`** если не задано `IsNullable = false` и не инициализировано `= new()`. Инициализируйте коллекции в объявлении, чтобы избежать `xsi:nil` и пустых обёрток.
- **`DateTimeOffset` сериализуется как строка ISO 8601**, но `XmlSerializer` не уважает `XmlDateTimeSerializationMode` так же гибко, как `DateTime`. Если партнёр ждёт `xs:dateTime`, используйте `DateTime` с `Utc` или строковое свойство-обёртку с `[XmlElement]`.
- **Пространства имён.** `[XmlRoot(Namespace=...)]` задаёт дефолтное пространство для корня; вложенные элементы наследуют его, если не переопределены `[XmlElement(Namespace=...)]`. Контракт AcmeLogistics требует единого `urn:acme:logistics:manifest:v1`.

#### Критерии приёмки

- [ ] Проект `TransContour.Gateway` собирается под `net8.0` без ошибок и предупреждений `nullable`.
- [ ] Используется C# 12: top-level statements в `Program.cs`, collection expressions/инициализаторы где уместно.
- [ ] `ShipmentManifest` помечен `[XmlRoot("manifest", Namespace = "urn:acme:logistics:manifest:v1")]`.
- [ ] `ManifestId` и `CreatedAt` — XML-атрибуты через `[XmlAttribute]`.
- [ ] `Sender`, `Receiver`, `TotalAmount` — элементы с явным `Order`.
- [ ] `Lines` — `[XmlArray("lines")] + [XmlArrayItem("line")]`.
- [ ] `InternalCacheKey` помечен `[XmlIgnore]` и не появляется в XML.
- [ ] `LineTotal` (вычисляемое) помечен `[XmlIgnore]`.
- [ ] `XmlSerializer` кэширован статически (`static readonly`).
- [ ] `Deserialize` использует `XmlReader` с `DtdProcessing = DtdProcessing.Prohibit`.
- [ ] Round-trip тест проходит: сериализация → десериализация даёт эквивалентный объект.
- [ ] Тест XXE: XML с `<!DOCTYPE>` выбрасывает исключение при десериализации.
- [ ] Вывод `dotnet run` содержит корректный XML с нужным пространством имён.
- [ ] Комментарий к классу `ShipmentXmlGateway` документирует причину выбора XML.
- [ ] Все 4+ теста зелёные (`dotnet test`).
- [ ] Бонус `DataContractSerializer` (если сделан) реализован с `preserveObjectReferences`.

#### Подсказки (без прямого ответа)

- Вспомните аналогию из урока: «`XmlSerializer` — строгий таможенник, генерирует сборку при первом обращении». Где «посадить» экземпляр, чтобы генерация случилась один раз?
- Для XXE-теста достаточно подать строку, начинающуюся с `<!DOCTYPE manifest SYSTEM "http://evil/evil.dtd">`, и ожидать `XmlException` при создании `XmlReader` или при десериализации.
- `LineTotal` объявлен как `public decimal LineTotal => Qty * UnitPrice;`. Подумайте, почему `XmlSerializer` всё равно попытается его сериализовать (он видит публичное get-only свойство), и какой атрибут его исключит.
- Если round-trip падает на `DateTimeOffset`, попробуйте `DateTime` с `.UtcDateTime` или строковое свойство с ручным форматированием ISO 8601.
- `XmlWriterSettings.Indent = true` и `OmitXmlDeclaration = false` дают человеко-читаемый вывод; для контрактных тестов отступы роли не играют, но облегчают ручную проверку.

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8 — ShipmentXmlGateway
// Эталонное решение ДЗ M10-L07
// Reference solution for homework M10-L07

using System.Xml;
using System.Xml.Serialization;

namespace TransContour.Gateway.Models;

// Корень XML-документа с пространством имён контракта AcmeLogistics
// Root of the XML document with the AcmeLogistics contract namespace
[XmlRoot("manifest", Namespace = "urn:acme:logistics:manifest:v1")]
public class ShipmentManifest
{
    // Атрибуты — компактно и строго по XSD
    // Attributes — compact and strictly per the XSD
    [XmlAttribute("id")]
    public string ManifestId { get; set; } = string.Empty;

    [XmlAttribute("created-at")]
    public DateTimeOffset CreatedAt { get; set; }

    // Элементы с фиксированным порядком — XSD sequence
    // Elements with fixed order — XSD sequence
    [XmlElement("sender", Order = 1)]
    public Party Sender { get; set; } = new();

    [XmlElement("receiver", Order = 2)]
    public Party Receiver { get; set; } = new();

    [XmlElement("total", Order = 3)]
    public decimal TotalAmount { get; set; }

    // Обёрнутая коллекция / Wrapped collection
    [XmlArray("lines", Order = 4)]
    [XmlArrayItem("line")]
    public List<ShipmentLine> Lines { get; set; } = new();

    // Служебное поле — никогда в XML / Internal field — never in XML
    [XmlIgnore]
    public string? InternalCacheKey { get; set; }
}

[XmlType("party")]
public class Party
{
    [XmlAttribute("code")]
    public string Code { get; set; } = string.Empty;

    [XmlElement("name")]
    public string Name { get; set; } = string.Empty;

    [XmlElement("country")]
    public string Country { get; set; } = string.Empty;
}

[XmlType("line")]
public class ShipmentLine
{
    [XmlAttribute("sku")]
    public string Sku { get; set; } = string.Empty;

    [XmlElement("qty")]
    public int Qty { get; set; }

    [XmlElement("unit-price")]
    public decimal UnitPrice { get; set; }

    // Вычисляемое свойство — обязательный [XmlIgnore], иначе утечёт в контракт
    // Computed property — mandatory [XmlIgnore], otherwise it leaks into the contract
    [XmlIgnore]
    public decimal LineTotal => Qty * UnitPrice;
}

/// <summary>
/// Шлюз обмена XML с legacy-системой AcmeLogistics.
/// XML выбран осознанно: партнёр использует BizTalk + XSD-контракт, JSON недоступен.
/// Gateway for XML exchange with the legacy AcmeLogistics system.
/// XML is a deliberate choice: the partner uses BizTalk + an XSD contract; JSON is unavailable.
/// </summary>
public static class ShipmentXmlGateway
{
    // Статический кэш — генерация временной сборки только один раз
    // Static cache — temporary assembly generation happens only once
    private static readonly XmlSerializer _serializer =
        new(typeof(ShipmentManifest));

    public static string Serialize(ShipmentManifest manifest)
    {
        using var sw = new StringWriter();
        using var xw = XmlWriter.Create(sw, new XmlWriterSettings
        {
            Indent = true,
            OmitXmlDeclaration = false,
            Encoding = System.Text.Encoding.UTF8
        });
        _serializer.Serialize(xw, manifest);
        return sw.ToString();
    }

    public static ShipmentManifest Deserialize(string xml)
    {
        using var sr = new StringReader(xml);
        // Безопасно: DTD запрещён, внешние сущности не резолвятся → нет XXE
        // Safe: DTD prohibited, external entities not resolved → no XXE
        var settings = new XmlReaderSettings
        {
            DtdProcessing = DtdProcessing.Prohibit
        };
        using var xr = XmlReader.Create(sr, settings);
        return (ShipmentManifest)_serializer.Deserialize(xr)!;
    }
}
```

Разбор по строкам. `[XmlRoot]` задаёт имя корня `manifest` и пространство имён `urn:acme:logistics:manifest:v1` — это требование контракта AcmeLogistics; без него партнёр отклонит документ. `[XmlAttribute("id")]` и `[XmlAttribute("created-at")]` превращают идентификатор и дату в атрибуты корня, что компактнее и соответствует XSD-стилю «ключи в атрибутах, данные в элементах». `Order = 1..4` на `[XmlElement]` фиксирует `sequence` — это та самая «тонкость» урока: XSD часто описывает последовательность, и нарушение порядка (sender после receiver) невалидно. `[XmlArray("lines")] + [XmlArrayItem("line")]` даёт обёрнутый список `<lines><line/><line/></lines>`, что требует контракт; альтернатива `[XmlElement("line")]` на коллекции дала бы плоский список без обёртки — это другая частая ошибка. `[XmlIgnore]` на `InternalCacheKey` исключает служебное поле; урок явно предостерегает от утечки вычисляемых и служебных свойств. На `LineTotal` (get-only вычисляемое) `[XmlIgnore]` критичен: `XmlSerializer` по умолчанию видит все публичные свойства с геттером и попытается сериализовать его, что сломает валидацию по XSD. Статический `private static readonly XmlSerializer _serializer` — ядро производительности: первый вызов дорогой (генерация сборки через `XmlSerializerCompiler`), но мы платим один раз, как требует урок. В `Deserialize` `XmlReaderSettings { DtdProcessing = DtdProcessing.Prohibit }` — защита от XXE: без неё парсер мог бы загрузить внешний DTD с `file://` или `http://`. Приведение `(ShipmentManifest)_serializer.Deserialize(xr)!` использует `!`, потому что для валидного XML по контракту результат не `null`; в библиотечном коде стоит добавить проверку и бросать доменное исключение. Концепции урока, применённые здесь: кэширование, атрибуты управления контрактом, безопасный `XmlReader`, осознанный выбор XML с задокументированной причиной.

#### Задания на углубление (бонус)

1. Реализуйте альтернативный сериализатор `ShipmentDataContractGateway` на `DataContractSerializer` с `preserveObjectReferences = true`. Сериализуйте граф, где две позиции ссылаются на один и тот же объект `Party` (общий получатель), и покажите, что `DataContractSerializer` сохраняет ссылку (атрибут `z:Id`/`z:Ref`), а `XmlSerializer` дублирует объект. Объясните, почему для AcmeLogistics это неприемлемо (контракт не содержит reference-семантики).
2. Добавьте валидацию XML по XSD: сгенерируйте или напишите вручную простую XSD-схему для `manifest`, загрузите её через `XmlReaderSettings.Schemas` с `ValidationType = ValidationType.Schema`, и убедитесь, что неверный порядок элементов (`receiver` перед `sender`) вызывает `XmlSchemaValidationException`.
3. Измерьте стоимость первого вызова `XmlSerializer` vs `DataContractSerializer`: в `Program.cs` запустите `Stopwatch` на первом `Serialize` каждого сериализатора, повторите 1000 раз второй вызов и сравните. Запишите числа в комментарий — это иллюстрирует тезис урока о «дорогом первом вызове».
4. Реализуйте потоковую сериализацию большого манифеста (100 000 позиций) через `XmlWriter` напрямую, без загрузки всего `List<ShipmentLine>` в память, и сравните пиковое потребление памяти с `XmlSerializer.Serialize`.

---

## Statement in English / Постановка на английском

#### Context & motivation

You are a backend developer at "TransContour", a company launching an integration with a partner's legacy system, "AcmeLogistics". The partner exchanges documents over a protocol that dates back to the SOAP/WSDL era, and the only accepted format is an XML shipment manifest whose shape is fixed by an XSD schema that exists solely as a 2009 printout. JSON is not supported, because the partner's message bus is built on BizTalk and the gateway expects XML with an envelope and namespaces. Your task is to implement, on C# 12 / .NET 8, a `ShipmentXmlGateway` module that can serialize the domain object `ShipmentManifest` to XML strictly per the contract and deserialize the reply back, without crashing on dirty data and without opening an XXE vulnerability.

Lesson M10-L07 is labeled Optional/legacy-vector precisely for situations like this: you must be able to maintain and migrate legacy code, but not start new projects on XML without a strong reason. Here the reason is real — an external XSD contract — and you honestly document it in the class comment. This exercise also verifies that you understand the difference between `XmlSerializer` (public properties, fine-grained attribute control, slow first call due to temporary assembly generation) and `DataContractSerializer` (works with `[DataContract]`/`[DataMember]`, supports object graphs, is faster, but yields less readable XML). Along the way you will implement the main path with `XmlSerializer` and, in the bonus, show how the same graph with a cyclic reference serializes through `DataContractSerializer` with `preserveObjectReferences`.

Special attention is paid to security. The partner sends XML that you do not fully control, so deserialization must go through an `XmlReader` with `DtdProcessing = DtdProcessing.Prohibit`. You must not use `XmlSerializer` overloads that accept types from the XML payload itself, and you must explicitly restrict the type set. This guards against the vectors described in the lesson: external entity injection (XXE) and attacks through `XmlSerializerFactory`.

#### What to do step by step

1. Create a .NET 8 console project named `TransContour.Gateway`:
   ```
   dotnet new console -n TransContour.Gateway -o TransContour.Gateway
   cd TransContour.Gateway
   dotnet new sln -n TransContour.Gateway
   dotnet sln add TransContour.Gateway.csproj
   ```
   Target framework is `net8.0`, language is C# 12 (top-level statements are allowed and recommended for the `Program.cs` entry point).

2. In `Models/ShipmentManifest.cs`, describe the domain model. The class `ShipmentManifest` contains: a manifest identifier (`ManifestId`, `string`), a creation timestamp (`CreatedAt`, `DateTimeOffset`), a sender (`Sender`, a nested `Party`), a receiver (`Receiver`, a nested `Party`), a list of lines (`Lines`, `List<ShipmentLine>`), a total amount (`TotalAmount`, `decimal`), and an internal field `InternalCacheKey` that must not appear in XML. Use `[XmlAttribute]` for the identifier and timestamp, `[XmlElement]` with explicit `Order` for sender/receiver/total, `[XmlArray]` + `[XmlArrayItem]` for the lines, and `[XmlIgnore]` for the internal field. Set the root name and namespace through `[XmlRoot("manifest", Namespace = "urn:acme:logistics:manifest:v1")]`.

3. The `Party` class must have `[XmlType("party")]` and properties `Code` (attribute `code`), `Name` (element `name`), `Country` (element `country`). The `ShipmentLine` class must have `[XmlType("line")]` with `Sku` (attribute), `Qty` (element), `UnitPrice` (element), and a computed read-only property `LineTotal` (`Qty * UnitPrice`) that is mandatory to mark with `[XmlIgnore]`, otherwise it leaks into XML and breaks the contract, as the "Common Mistakes" section of the lesson warns.

4. In `Serialization/ShipmentXmlGateway.cs`, implement a static class `ShipmentXmlGateway` with two methods: `string Serialize(ShipmentManifest manifest)` and `ShipmentManifest Deserialize(string xml)`. Cache the `XmlSerializer` statically (`private static readonly XmlSerializer _serializer = new(typeof(ShipmentManifest));`) — this is a lesson requirement, otherwise you pay for temporary assembly generation on every call. In `Serialize`, use `StringWriter` + `XmlWriter` with `XmlWriterSettings { Indent = true, OmitXmlDeclaration = false }`. In `Deserialize`, create an `XmlReader` with `XmlReaderSettings { DtdProcessing = DtdProcessing.Prohibit }` — this protects against XXE.

5. In `Program.cs`, build a sample `ShipmentManifest` with two lines, serialize it, and print the XML to the console. Then deserialize the string back and print the restored fields. Verify that `InternalCacheKey` is absent from the XML and that `LineTotal` is not serialized. The expected output must contain the root tag `<manifest xmlns="urn:acme:logistics:manifest:v1">`, attributes `id` and `created-at`, elements `<sender>`, `<receiver>`, `<total>`, the array `<lines>` with nested `<line>` elements.

6. Add `Tests/RoundTripTests.cs` (you can use xUnit: `dotnet add package xunit xunit.runner.visualstudio`). Write tests: (a) serialize + deserialize returns an equivalent object; (b) the XML contains attribute `id` and does not contain `InternalCacheKey`; (c) deserializing XML with `<!DOCTYPE ... SYSTEM "http://evil/evil.dtd">` throws (XXE check); (d) empty and `null` collections are handled correctly.

7. Run `dotnet test`. There must be 4 green tests. Run `dotnet run --project TransContour.Gateway` and attach the output to the submission.

8. In the comments on the `ShipmentXmlGateway` class, explicitly state the reason for choosing XML: "legacy XSD contract, partner AcmeLogistics, BizTalk protocol — JSON unavailable". This closes the lesson requirement to document the rationale.

#### Requirements

- Target framework `net8.0`, language C# 12; in `Program.cs` use top-level statements, collection expressions/initializers where appropriate.
- Use `XmlSerializer` from `System.Xml.Serialization`; do not write your own XML parser.
- The serializer must be cached statically — do not create `new XmlSerializer(typeof(ShipmentManifest))` inside methods. The lesson explicitly calls this a common mistake.
- Deserialization must go through an `XmlReader` with `DtdProcessing = DtdProcessing.Prohibit`. Using a bare `StringReader` without `XmlReader` and without settings is only acceptable in a "trusted" branch; in this assignment the partner data is untrusted, so settings are mandatory.
- All properties outside the contract (computed, internal, cache keys) must be marked `[XmlIgnore]`. Especially `LineTotal`, a computed property whose leakage into XML breaks XSD validation.
- The root element must carry the namespace `urn:acme:logistics:manifest:v1` — this is part of the contract; without it the partner rejects the document.
- The code compiles without `nullable` warnings. Enable `<Nullable>enable</Nullable>` in the `.csproj` and handle `null` deliberately (for example, default strings to `string.Empty`).
- Tests use xUnit and cover round-trip, absence of internal fields, and XXE protection. Disabling a test with `Assert.True(true)` or similar placeholders is forbidden.

#### Pitfalls

- **Static caching is mandatory.** Every `new XmlSerializer(typeof(T))` generates a temporary assembly via `XmlSerializerCompiler` on first contact. In a hot path this is catastrophic for speed and fragments memory. The lesson names this the first common mistake — cache in `static readonly`.
- **`XmlSerializer` sees only public members.** If you mark a property `private` or `internal`, it is not serialized, and the `[DataMember]` style does not apply here. For private fields you need `DataContractSerializer`, which is the bonus topic.
- **XXE via DTD.** Without `DtdProcessing = Prohibit` (or at least `Parse` with a disabled resolver) the parser can load an external DTD via `file://` or `http://`, leaking files (classic XXE). In .NET 8+ the defaults are safer than in .NET Framework, but do not rely on defaults — set `XmlReaderSettings` explicitly.
- **Do not use `XmlReaderSettings` with `DtdProcessing.Parse` and the default `XmlResolver`** — that opens the vector. The lesson says: "disable `DtdProcessing`".
- **Computed properties leak.** `LineTotal` without `[XmlIgnore]` appears in XML as `<LineTotal>`, and the partner validating against XSD rejects the document. This is the second common mistake from the lesson.
- **`[XmlArray]` vs `[XmlElement]` for collections.** `[XmlArray("lines")] + [XmlArrayItem("line")]` yields `<lines><line/><line/></lines>`. If you want a flat list `<line/><line/>` without a wrapper, use `[XmlElement("line")]` directly on the collection property. The AcmeLogistics contract requires the wrapper, so `XmlArray` is chosen.
- **`Order` on `[XmlElement]` fixes the sequence.** XSD often describes a `sequence`, and breaking the order (sender after receiver) is invalid. Specify `Order` explicitly.
- **A `null` collection serializes as `<lines xsi:nil="true"/>`** unless `IsNullable = false` is set or it is initialized with `= new()`. Initialize collections in their declaration to avoid `xsi:nil` and empty wrappers.
- **`DateTimeOffset` serializes as an ISO 8601 string**, but `XmlSerializer` does not honor `XmlDateTimeSerializationMode` as flexibly as it does for `DateTime`. If the partner expects `xs:dateTime`, use a `DateTime` in `Utc` or a string wrapper property with `[XmlElement]`.
- **Namespaces.** `[XmlRoot(Namespace=...)]` sets the default namespace for the root; nested elements inherit it unless overridden by `[XmlElement(Namespace=...)]`. The AcmeLogistics contract requires a single `urn:acme:logistics:manifest:v1`.

#### Acceptance criteria

- [ ] The `TransContour.Gateway` project builds under `net8.0` with no errors and no `nullable` warnings.
- [ ] C# 12 is used: top-level statements in `Program.cs`, collection expressions/initializers where appropriate.
- [ ] `ShipmentManifest` is annotated with `[XmlRoot("manifest", Namespace = "urn:acme:logistics:manifest:v1")]`.
- [ ] `ManifestId` and `CreatedAt` are XML attributes via `[XmlAttribute]`.
- [ ] `Sender`, `Receiver`, `TotalAmount` are elements with explicit `Order`.
- [ ] `Lines` uses `[XmlArray("lines")] + [XmlArrayItem("line")]`.
- [ ] `InternalCacheKey` is marked `[XmlIgnore]` and never appears in XML.
- [ ] `LineTotal` (computed) is marked `[XmlIgnore]`.
- [ ] The `XmlSerializer` is cached statically (`static readonly`).
- [ ] `Deserialize` uses an `XmlReader` with `DtdProcessing = DtdProcessing.Prohibit`.
- [ ] The round-trip test passes: serialize → deserialize yields an equivalent object.
- [ ] The XXE test: XML with `<!DOCTYPE>` throws on deserialization.
- [ ] The `dotnet run` output contains correct XML with the required namespace.
- [ ] The `ShipmentXmlGateway` class comment documents the reason for choosing XML.
- [ ] All 4+ tests are green (`dotnet test`).
- [ ] The `DataContractSerializer` bonus (if done) uses `preserveObjectReferences`.

#### Hints (no direct answer)

- Recall the lesson analogy: "`XmlSerializer` is a strict customs officer that generates an assembly on first contact." Where do you place the instance so generation happens once?
- For the XXE test, it is enough to feed a string starting with `<!DOCTYPE manifest SYSTEM "http://evil/evil.dtd">` and expect an `XmlException` when the `XmlReader` is created or when deserializing.
- `LineTotal` is declared as `public decimal LineTotal => Qty * UnitPrice;`. Think about why `XmlSerializer` still tries to serialize it (it sees any public get-only property) and which attribute excludes it.
- If the round-trip fails on `DateTimeOffset`, try `DateTime` with `.UtcDateTime` or a string property with manual ISO 8601 formatting.
- `XmlWriterSettings.Indent = true` and `OmitXmlDeclaration = false` produce human-readable output; indenting does not matter for contract tests but eases manual review.

#### Reference solution walk-through

```csharp
// C# 12 / .NET 8 — ShipmentXmlGateway
// Reference solution for homework M10-L07

using System.Xml;
using System.Xml.Serialization;

namespace TransContour.Gateway.Models;

// Root of the XML document with the AcmeLogistics contract namespace
[XmlRoot("manifest", Namespace = "urn:acme:logistics:manifest:v1")]
public class ShipmentManifest
{
    // Attributes — compact and strictly per the XSD
    [XmlAttribute("id")]
    public string ManifestId { get; set; } = string.Empty;

    [XmlAttribute("created-at")]
    public DateTimeOffset CreatedAt { get; set; }

    // Elements with fixed order — XSD sequence
    [XmlElement("sender", Order = 1)]
    public Party Sender { get; set; } = new();

    [XmlElement("receiver", Order = 2)]
    public Party Receiver { get; set; } = new();

    [XmlElement("total", Order = 3)]
    public decimal TotalAmount { get; set; }

    // Wrapped collection
    [XmlArray("lines", Order = 4)]
    [XmlArrayItem("line")]
    public List<ShipmentLine> Lines { get; set; } = new();

    // Internal field — never in XML
    [XmlIgnore]
    public string? InternalCacheKey { get; set; }
}

[XmlType("party")]
public class Party
{
    [XmlAttribute("code")]
    public string Code { get; set; } = string.Empty;

    [XmlElement("name")]
    public string Name { get; set; } = string.Empty;

    [XmlElement("country")]
    public string Country { get; set; } = string.Empty;
}

[XmlType("line")]
public class ShipmentLine
{
    [XmlAttribute("sku")]
    public string Sku { get; set; } = string.Empty;

    [XmlElement("qty")]
    public int Qty { get; set; }

    [XmlElement("unit-price")]
    public decimal UnitPrice { get; set; }

    // Computed property — mandatory [XmlIgnore], otherwise it leaks into the contract
    [XmlIgnore]
    public decimal LineTotal => Qty * UnitPrice;
}

/// <summary>
/// Gateway for XML exchange with the legacy AcmeLogistics system.
/// XML is a deliberate choice: the partner uses BizTalk + an XSD contract; JSON is unavailable.
/// </summary>
public static class ShipmentXmlGateway
{
    // Static cache — temporary assembly generation happens only once
    private static readonly XmlSerializer _serializer =
        new(typeof(ShipmentManifest));

    public static string Serialize(ShipmentManifest manifest)
    {
        using var sw = new StringWriter();
        using var xw = XmlWriter.Create(sw, new XmlWriterSettings
        {
            Indent = true,
            OmitXmlDeclaration = false,
            Encoding = System.Text.Encoding.UTF8
        });
        _serializer.Serialize(xw, manifest);
        return sw.ToString();
    }

    public static ShipmentManifest Deserialize(string xml)
    {
        using var sr = new StringReader(xml);
        // Safe: DTD prohibited, external entities not resolved → no XXE
        var settings = new XmlReaderSettings
        {
            DtdProcessing = DtdProcessing.Prohibit
        };
        using var xr = XmlReader.Create(sr, settings);
        return (ShipmentManifest)_serializer.Deserialize(xr)!;
    }
}
```

Walk-through, line by line. `[XmlRoot]` sets the root name `manifest` and the namespace `urn:acme:logistics:manifest:v1` — a contract requirement of AcmeLogistics; without it the partner rejects the document. `[XmlAttribute("id")]` and `[XmlAttribute("created-at")]` turn the identifier and timestamp into attributes of the root, which is more compact and matches the XSD style "keys in attributes, data in elements". `Order = 1..4` on `[XmlElement]` fixes the `sequence` — this is exactly the lesson pitfall: an XSD often describes a sequence, and breaking the order (sender after receiver) is invalid. `[XmlArray("lines")] + [XmlArrayItem("line")]` produces the wrapped list `<lines><line/><line/></lines>` that the contract requires; the alternative `[XmlElement("line")]` on the collection would yield a flat list without a wrapper — another common mistake. `[XmlIgnore]` on `InternalCacheKey` excludes the internal field; the lesson explicitly warns against leaking computed and internal properties. On `LineTotal` (a get-only computed property), `[XmlIgnore]` is critical: `XmlSerializer` sees every public property with a getter by default and would try to serialize it, breaking XSD validation. The static `private static readonly XmlSerializer _serializer` is the performance core: the first call is expensive (assembly generation via `XmlSerializerCompiler`), but we pay once, as the lesson requires. In `Deserialize`, `XmlReaderSettings { DtdProcessing = DtdProcessing.Prohibit }` is XXE protection: without it the parser could load an external DTD via `file://` or `http://`. The cast `(ShipmentManifest)_serializer.Deserialize(xr)!` uses `!` because, for valid XML matching the contract, the result is not `null`; in library code you would add a check and throw a domain exception. Lesson concepts applied here: caching, contract-controlling attributes, safe `XmlReader`, and a deliberate, documented choice of XML.

#### Going deeper (bonus)

1. Implement an alternative serializer `ShipmentDataContractGateway` on `DataContractSerializer` with `preserveObjectReferences = true`. Serialize a graph where two lines reference the same `Party` object (a shared receiver), and show that `DataContractSerializer` preserves the reference (the `z:Id`/`z:Ref` attributes) while `XmlSerializer` duplicates the object. Explain why this is unacceptable for AcmeLogistics (the contract has no reference semantics).
2. Add XSD validation: generate or hand-write a simple XSD schema for `manifest`, load it through `XmlReaderSettings.Schemas` with `ValidationType = ValidationType.Schema`, and confirm that an invalid element order (`receiver` before `sender`) raises `XmlSchemaValidationException`.
3. Measure the cost of the first `XmlSerializer` call vs `DataContractSerializer`: in `Program.cs`, run a `Stopwatch` on the first `Serialize` of each serializer, then repeat the second call 1000 times and compare. Record the numbers in a comment — this illustrates the lesson thesis about the "expensive first call".
4. Implement streaming serialization of a large manifest (100,000 lines) through `XmlWriter` directly, without loading the whole `List<ShipmentLine>` into memory, and compare peak memory usage with `XmlSerializer.Serialize`.

---

#### Чек-лист сдачи / Submission checklist
- [ ] Проект `TransContour.Gateway` собирается под `net8.0` без предупреждений `nullable`.
- [ ] `ShipmentManifest`, `Party`, `ShipmentLine` с корректными XML-атрибутами.
- [ ] `ShipmentXmlGateway` со статически кэшированным `XmlSerializer`.
- [ ] `Deserialize` использует `XmlReader` с `DtdProcessing = Prohibit`.
- [ ] Round-trip и XXE тесты зелёные.
- [ ] Вывод `dotnet run` приложен к сдаче.
- [ ] Комментарий документирует причину выбора XML.
- [ ] (RU) Бонус `DataContractSerializer` реализован (опционально).
- [ ] The `TransContour.Gateway` project builds under `net8.0` with no `nullable` warnings.
- [ ] `ShipmentManifest`, `Party`, `ShipmentLine` with correct XML attributes.
- [ ] `ShipmentXmlGateway` with a statically cached `XmlSerializer`.
- [ ] `Deserialize` uses an `XmlReader` with `DtdProcessing = Prohibit`.
- [ ] Round-trip and XXE tests are green.
- [ ] The `dotnet run` output is attached to the submission.
- [ ] A comment documents the reason for choosing XML.
- [ ] (EN) The `DataContractSerializer` bonus is implemented (optional).

#### Ресурсы / Resources
- [Microsoft Learn — Introducing XML Serialization](https://learn.microsoft.com/dotnet/standard/serialization/introducing-xml-serialization)
- [Microsoft Learn — XmlSerializer Class](https://learn.microsoft.com/dotnet/api/system.xml.serialization.xmlserializer)
- [Microsoft Learn — DataContractSerializer](https://learn.microsoft.com/dotnet/api/system.runtime.serialization.datacontractserializer)
- [Microsoft Learn — XmlReaderSettings.DtdProcessing](https://learn.microsoft.com/dotnet/api/system.xml.xmlreadersettings.dtdprocessing)
- [OWASP — XML External Entity (XXE) Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/XML_External_Entity_Prevention_Cheat_Sheet.html)

---
[← К уроку M10-L07](lesson-M10-L07-xml-serialization.md) | [⬆ К модулю M10](../README.md) | [Следующее ДЗ →](homework-M10-L08-binary-formats.md)
