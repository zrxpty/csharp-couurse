[⬆ К модулю M10](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M10-L07: XML-сериализация (Optional/Deprecated-вектор) / XML serialization (Optional/legacy vector)

**Модуль / Module:** M10
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

XML-сериализация в .NET — это способ перевести объекты в текстовый формат XML и обратно. Хотя сегодня для новых проектов чаще выбирают JSON (System.Text.Json), XML остаётся важным в трёх случаях: интеграция с legacy-системами, протокол SOAP и строгие контракты с XSD-схемами. Поэтому данный урок помечен как Optional/Deprecated-вектор — это «запасной путь», который нужно знать, но не применять без необходимости.

Главный инструмент — `XmlSerializer` из пространства имён `System.Xml.Serialization`. Он работает с **публичными** свойствами и полями, по умолчанию сериализует всё публичное и игнорирует приватное. Поведением управляют атрибуты: `[XmlElement]` задаёт имя и порядок тега, `[XmlAttribute]` переводит свойство в атрибут, `[XmlIgnore]` исключает поле, `[XmlArray]` / `[XmlArrayItem]` настраивают коллекции. Это похоже на decorating-паттерн: вы «наклеиваете» метки на свойства, а сериализатор читает их как инструкции.

Важная аналогия: `XmlSerializer` — это строгий таможенник. Он заранее генерирует временную сборку с кодом сериализации при первом обращении к типу. Это даёт скорость на повторных вызовах, но первый вызов «дорогой». Кэшируйте экземпляр `XmlSerializer` статически, чтобы не платить за генерацию каждый раз.

Альтернатива — `DataContractSerializer` из `System.Runtime.Serialization`. Он быстрее, поддерживает графы объектов (`preserveObjectReferences`) и работает с типами, помеченными `[DataContract]` / `[DataMember]`. Используйте его, когда нужна сериализация приватных полей или графов с циклическими ссылками. Однако он выдаёт менее «красивый» XML и слабее настраивается, чем `XmlSerializer`.

Когда выбирать XML? 1) Legacy-интеграция: старые сервисы (WCF, SOAP-WSDL, BizTalk) ждут именно XML. 2) SOAP-протокол: стандарт требует XML с конвертом. 3) Сильные контракты: есть XSD-схема, валидация, подписи (XMLDSig). 4) Конфигурационные файлы с человеко-читаемой иерархией. Во всех новых API и хранении данных предпочтение отдаётся JSON.

Deprecated-вектор здесь — намеренный выбор: мы учим XML, чтобы вы могли поддерживать и мигрировать legacy-код, но не начинали новые проекты с XML без веской причины. Практическое правило: «новое — JSON, старое — XML, контракт с XSD — XML, остальное — пересмотрите решение».

Безопасность: `XmlSerializer` уязвим к атакам через типы, указанные в самом XML (вектор `XmlSerializer(Type, extraTypes)` и `XmlSerializerFactory`). В .NET 8+ используйте перегрузки без `XmlSerializer`-а, принимающего типы из входных данных, и отключайте `DtdProcessing`. Никогда не десериализуйте XML из недоверенного источника через `XmlSerializer` без ограничений типов.

#### Theory (EN)

XML serialization in .NET is the mechanism that translates objects into XML text and back. Although modern projects usually prefer JSON via System.Text.Json, XML remains important in three scenarios: integration with legacy systems, the SOAP protocol, and strict contracts backed by XSD schemas. This lesson is therefore labeled Optional/legacy vector — a path you must know but should not adopt without a reason.

The primary tool is `XmlSerializer` from the `System.Xml.Serialization` namespace. It works with **public** properties and fields, serializing everything public by default and ignoring private members. Behavior is controlled through attributes: `[XmlElement]` sets the tag name and order, `[XmlAttribute]` promotes a property to an attribute, `[XmlIgnore]` excludes a member, and `[XmlArray]` / `[XmlArrayItem]` configure collections. Think of it as a decorating pattern: you tag properties with metadata, and the serializer reads those tags as instructions.

A key analogy: `XmlSerializer` is a strict customs officer. On first contact with a type it generates a temporary serialization assembly. This makes subsequent calls fast, but the first call is expensive. Cache a static `XmlSerializer` instance per type so you pay the generation cost only once.

The alternative is `DataContractSerializer` from `System.Runtime.Serialization`. It is faster, supports object graphs (`preserveObjectReferences`), and works with types annotated by `[DataContract]` / `[DataMember]`. Use it when you need to serialize private fields or graphs with cyclic references. However, it produces less human-friendly XML and is less configurable than `XmlSerializer`.

When should you pick XML? First, legacy integration: older services (WCF, SOAP-WSDL, BizTalk) expect XML. Second, the SOAP protocol, which mandates XML envelopes. Third, strong contracts: an XSD schema, validation, or XML digital signatures (XMLDSig). Fourth, human-readable hierarchical configuration files. For new APIs and data storage, JSON is the default.

The legacy-vector framing here is deliberate: we teach XML so you can maintain and migrate legacy code, not so you start new projects on XML without a strong reason. The practical rule is: "new is JSON, old is XML, contracts with XSD are XML, anything else — reconsider the decision."

Security matters: `XmlSerializer` can be exploited when types are taken from the XML payload itself (the `XmlSerializer(Type, extraTypes)` and `XmlSerializerFactory` vectors). In .NET 8+, prefer overloads that do not accept types driven by input data, and disable `DtdProcessing`. Never deserialize XML from an untrusted source through `XmlSerializer` without type restrictions.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — XML-сериализация через XmlSerializer
// XML serialization via XmlSerializer

using System.Xml;
using System.Xml.Serialization;

// Модель заказа / Order model
public class Order
{
    // Атрибут становится XML-атрибутом, а не элементом
    // Property becomes an XML attribute instead of an element
    [XmlAttribute("id")]
    public int Id { get; set; }

    // Имя элемента и порядок / Element name and order
    [XmlElement("customer", Order = 1)]
    public string Customer { get; set; } = string.Empty;

    [XmlElement("total", Order = 2)]
    public decimal Total { get; set; }

    // Коллекция с пользовательскими именами
    // Collection with custom names
    [XmlArray("items", Order = 3)]
    [XmlArrayItem("line")]
    public List<OrderLine> Items { get; set; } = new();

    // Внутреннее поле не сериализуется по умолчанию — но явно исключим
    // Internal field is not serialized by default, exclude explicitly
    [XmlIgnore]
    public string? CacheKey { get; set; }
}

public class OrderLine
{
    [XmlAttribute("sku")]
    public string Sku { get; set; } = string.Empty;

    [XmlElement("qty")]
    public int Qty { get; set; }

    [XmlElement("price")]
    public decimal Price { get; set; }
}

public static class OrderXmlSerializer
{
    // Кэшируем сериализатор статически — генерация сборки только один раз
    // Cache serializer statically — assembly generation happens once
    private static readonly XmlSerializer _serializer =
        new(typeof(Order));

    public static string Serialize(Order order)
    {
        using var sw = new StringWriter();
        using var xw = XmlWriter.Create(sw, new XmlWriterSettings
        {
            Indent = true,
            OmitXmlDeclaration = false
        });
        _serializer.Serialize(xw, order);
        return sw.ToString();
    }

    public static Order Deserialize(string xml)
    {
        using var sr = new StringReader(xml);
        // Безопасные настройки: без DTD, без разрешения внешних сущностей
        // Safe settings: no DTD, no external entity resolution
        var settings = new XmlReaderSettings
        {
            DtdProcessing = DtdProcessing.Prohibit
        };
        using var xr = XmlReader.Create(sr, settings);
        return (Order)_serializer.Deserialize(xr)!;
    }
}

// Использование / Usage
var order = new Order
{
    Id = 42,
    Customer = "Иван Петров",
    Total = 125.50m,
    Items =
    {
        new OrderLine { Sku = "A1", Qty = 2, Price = 50.00m },
        new OrderLine { Sku = "B2", Qty = 1, Price = 25.50m }
    }
};

string xml = OrderXmlSerializer.Serialize(order);
Console.WriteLine(xml);

Order restored = OrderXmlSerializer.Deserialize(xml);
Console.WriteLine($"Restored: {restored.Id} / {restored.Customer} / {restored.Total}");
```

#### Best Practices
- Кэшируйте `XmlSerializer` статически для каждого типа, чтобы избежать повторной генерации временной сборки.
- Используйте `XmlReader`/`XmlWriter` с явными `XmlReaderSettings` (`DtdProcessing = Prohibit`) при десериализации недоверенных данных.
- Предпочитайте `DataContractSerializer` для графов объектов с циклическими ссылками или приватных полей.
- Не используйте XML для новых публичных API — JSON компактнее и де-факто стандарт.
- Включайте XML только при legacy/SOAP/XSD-контрактах и документируйте причину выбора.

#### Best Practices
- Cache `XmlSerializer` statically per type to avoid regenerating the temporary assembly.
- Use `XmlReader`/`XmlWriter` with explicit `XmlReaderSettings` (`DtdProcessing = Prohibit`) when deserializing untrusted data.
- Prefer `DataContractSerializer` for object graphs with cycles or private fields.
- Do not use XML for new public APIs — JSON is more compact and the de-facto standard.
- Adopt XML only for legacy/SOAP/XSD contracts, and document the rationale.

#### Частые ошибки / Common Mistakes
- Создание `new XmlSerializer(typeof(T))` на каждый вызов → кэшируйте статически; иначе платите за генерацию сборки каждый раз (RU).
- Десериализация без `XmlReaderSettings` с запрещённым DTD → уязвимость к XXE; всегда запрещайте `DtdProcessing` (RU).
- Ожидание сериализации приватных полей через `XmlSerializer` → он работает только с публичными; используйте `DataContractSerializer` (RU).
- Использование XML в новом REST API без legacy-причины → выбирайте JSON, XML избыточен (RU).
- Игнорирование `[XmlIgnore]` на вычисляемых свойствах → они попадают в XML и ломают контракт (RU).

- Creating `new XmlSerializer(typeof(T))` per call → cache statically; otherwise you pay for assembly generation every time (EN).
- Deserializing without `XmlReaderSettings` disabling DTD → XXE vulnerability; always prohibit `DtdProcessing` (EN).
- Expecting `XmlSerializer` to serialize private fields → it only handles public members; use `DataContractSerializer` (EN).
- Using XML in a new REST API without a legacy reason → prefer JSON; XML is verbose (EN).
- Forgetting `[XmlIgnore]` on computed properties → they leak into XML and break the contract (EN).

#### Чек-лист самопроверки / Self-check Checklist
- [ ] Я кэширую `XmlSerializer` статически, а не создаю на каждый вызов.
- [ ] Я запрещаю `DtdProcessing` при чтении недоверенного XML.
- [ ] Я использую атрибуты `[XmlElement]`, `[XmlAttribute]`, `[XmlIgnore]` осознанно.
- [ ] Я выбрал XML только потому, что есть legacy/SOAP/XSD-контракт.
- [ ] Я знаю разницу между `XmlSerializer` и `DataContractSerializer`.

- [ ] I cache `XmlSerializer` statically instead of creating it per call.
- [ ] I disable `DtdProcessing` when reading untrusted XML.
- [ ] I use `[XmlElement]`, `[XmlAttribute]`, `[XmlIgnore]` attributes deliberately.
- [ ] I chose XML only because of a legacy/SOAP/XSD contract.
- [ ] I understand the difference between `XmlSerializer` and `DataContractSerializer`.

#### Ресурсы / Resources
- [Microsoft Learn — Introducing XML Serialization](https://learn.microsoft.com/dotnet/standard/serialization/introducing-xml-serialization)
- [Microsoft Learn — XmlSerializer Class](https://learn.microsoft.com/dotnet/api/system.xml.serialization.xmlserializer)
- [Microsoft Learn — DataContractSerializer](https://learn.microsoft.com/dotnet/api/system.runtime.serialization.datacontractserializer)

---

[⬆ К модулю M10](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
