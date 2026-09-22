---
[← К уроку M04-L04](lesson-M04-L04-encapsulation.md) | [⬆ К модулю M04](../README.md) | [Следующее ДЗ →](homework-M04-L05-static-readonly-const.md)
---

### Домашнее задание M04-L04: Инкапсуляция: модификаторы доступа / Homework M04-L04: Encapsulation: access modifiers

**Урок / Lesson:** M04-L04
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) Научиться скрывать внутреннее состояние объектов и управлять публичным контрактом через свойства, поля и методы, грамотно применяя все семь модификаторов доступа C# 12 (.NET 8), включая пары `protected internal`/`private protected`, и обеспечивая инкапсуляцию коллекций через неизменяемые проекции. (EN) Learn to hide internal object state and govern the public contract through properties, fields and methods, correctly applying all seven C# 12 (.NET 8) access modifiers — including the `protected internal` / `private protected` pair — and protecting collections with read-only projections.

#### Связь с уроком / Connection to the lesson
(RU) Урок вводит инкапсуляцию как контракт между объектом и внешним миром, разбирает семь модификаторов доступа, демонстрирует полный цикл: приватное поле → свойство с валидацией → публичный метод → неизменяемая проекция коллекции. Это ДЗ закрепляет каждую из этих концепций на сквозном примере складского учёта, где нарушение инварианта (отрицательный остаток, «дырявая» коллекция, утечка `internal`-типа в публичный API) немедленно ломает бизнес-логику.
(EN) The lesson frames encapsulation as a contract between an object and the outside world, walks through all seven access modifiers, and demonstrates the full loop: private field → validating property → public method → read-only collection projection. This homework fixes each of those concepts in a single end-to-end warehouse-inventory example, where breaking an invariant (negative stock, a "leaky" collection, an `internal` type leaking into the public API) immediately corrupts business logic.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Вы — инженер небольшой команды, пишущей библиотеку складского учёта `Warehouse.Core`, которую подключает к себе несколько потребителей: монолит-приложение `Warehouse.App` в том же solution и сторонний плагин `Warehouse.Plugin.Shipping`, компилируемый в отдельную сборку и подключаемый через AssemblyLoadContext. Библиотека должна выставить наружу минимальный, продуманный публичный контракт, оставив внутри себя вспомогательные типы, кэши и алгоритмы, которые внешнему коду знать не нужно и знать вредно. Ключевой инвариант: остаток товара на складе не может стать отрицательным ни при каких операциях, а список транзакций по товару — неизменяемая историческая правда, которую нельзя «подчистить» снаружи. Любой публичный сеттер, принимающий количество или цену, обязан валидировать ввод и бросать осмысленные исключения (`ArgumentException`, `ArgumentOutOfRangeException`) с указанием имени параметра через `nameof`. Свойства-геттеры обязаны быть чистыми: без побочных эффектов, без исключений, без скрытого изменения состояния. Это классический сценарий, где модификаторы доступа — не косметика, а инструмент обеспечения корректности: `private` прячет поля, `private protected` открывает реализацию только «своим» наследникам в сборке, `internal` прячет вспомогательный тип от плагина, а `IReadOnlyCollection<T>` защищает внутренний список.

#### Что нужно сделать (пошагово)
1. Создайте solution и три проекта: `dotnet new sln -n Warehouse`, затем `dotnet new classlib -n Warehouse.Core -o src/Warehouse.Core --framework net8.0`, `dotnet new console -n Warehouse.App -o src/Warehouse.App --framework net8.0`, `dotnet new classlib -n Warehouse.Plugin.Shipping -o src/Warehouse.Plugin.Shipping --framework net8.0`. Добавьте ссылки: `dotnet add src/Warehouse.App reference src/Warehouse.Core` и `dotnet add src/Warehouse.Plugin.Shipping reference src/Warehouse.Core`.
2. В `Warehouse.Core` включите `<Nullable>enable</Nullable>` и `<ImplicitUsings>enable</ImplicitUsings>` в `.csproj` (C# 12 по умолчанию для net8.0). Все типы — в `namespace Warehouse.Core;`.
3. Реализуйте класс `Product` (записывать в `Product.cs`): приватное поле `private decimal _unitPrice;` и полное свойство `public decimal UnitPrice { get => _unitPrice; private set { if (value < 0) throw new ArgumentOutOfRangeException(nameof(value), "Цена не может быть отрицательной."); _unitPrice = value; } }`; get-only автосвойства `public string Sku { get; }` и `public string Title { get; }`; метод `public void ChangePrice(decimal newPrice)`, делегирующий в сеттер. Конструктор валидирует `sku`/`title` на `null` через `ArgumentNullException` и вызывает сеттер `UnitPrice`.
4. Реализуйте абстрактный `public abstract class WarehouseItem` с `protected Guid Id { get; }`, `internal string Department { get; set; }`, `protected internal string Note { get; set; }`, `private protected DateTime CreatedAt { get; }`. Конструктор принимает `Guid id` и присваивает `CreatedAt = DateTime.UtcNow`. Добавьте `public abstract string Describe();`.
5. Реализуйте `public sealed class StockItem : WarehouseItem`, хранящий `Product` и количество. Приватное поле `private int _quantity;` с полным свойством `public int Quantity { get => _quantity; private set { if (value < 0) throw new ArgumentOutOfRangeException(nameof(value), "Остаток не может быть отрицательным."); _quantity = value; } }`. Методы `public void Restock(int amount)` и `public void Ship(int amount)` идут только через сеттер, чтобы инвариант проверялся в одном месте.
6. Реализуйте `internal sealed class StockRegistry` — тип НЕ публичный, нужен только внутри сборки. Хранит `private readonly List<StockItem> _items = new();`. Метод `internal void Register(StockItem item)`. Свойство `public IReadOnlyCollection<StockItem> Items => _items;` — обратите внимание: `public` на члене `internal` класса всё равно невидим снаружи, но такой код самодокументирующий.
7. Реализуйте `public sealed class Warehouse` с приватным полем `private readonly StockRegistry _registry = new();`. Публичный контракт: `public IReadOnlyCollection<StockItem> Items => _registry.Items;`, `public void Add(StockItem item)`, `public void Ship(string sku, int amount)`. В `Ship` используйте pattern matching: `if (_registry.Items.FirstOrDefault(i => i.Product.Sku == sku) is { } item) item.Ship(amount); else throw new InvalidOperationException($"Товар {sku} не найден.");`.
8. Создайте файл `Demo.cs` со статическим классом `public static class Demo` и методом `public static void Run()`, демонстрирующим создание продукта, остатка, пополнение, отгрузку и вывод `Describe()`.
9. В `Warehouse.App/Program.cs` используйте top-level statements: `using Warehouse.Core; Demo.Run();`. Запустите `dotnet run --project src/Warehouse.App` — ожидаемый вывод включает строки вида `STOCK-0001 / Винт М6: 80 шт.`.
10. В `Warehouse.Plugin.Shipping` попробуйте обратиться к `StockRegistry` — компилятор выдаст `CS0122` («недоступен из-за уровня защиты»). Это и есть проверка, что `internal` работает. Закомментируйте или удалите эту строку после проверки.
11. Покройте инварианты юнит-тестами в `dotnet new xunit -n Warehouse.Tests -o tests/Warehouse.Tests` со ссылкой на `Warehouse.Core`: тест, что `Ship` сверх остатка бросает `InvalidOperationException`; тест, что `Restock` с отрицательным `amount` бросает `ArgumentOutOfRangeException`; тест, что коллекция `Items` не приводится к `List<StockItem>` (проверьте, что `is List<StockItem>` даёт `false`).

#### Требования к решению
- Целевая платформа — строго .NET 8, язык C# 12; включены `<Nullable>enable</Nullable>` и `<ImplicitUsings>enable</ImplicitUsings>`.
- Все поля классов — `private`. Никаких публичных или `internal` полей. Публичный контракт — только через свойства и методы.
- Все свойства, изменяющие критичное состояние (`UnitPrice`, `Quantity`), — полные свойства с приватным сеттером и валидацией; валидация кидает типизированные исключения с `nameof`.
- Неизменяемые данные (`Sku`, `Title`, `Id`, `CreatedAt`) — get-only автосвойства, задаваемые только в конструкторе; ни одного публичного сеттера у них нет.
- Все get-аксессоры — чистые: не меняют состояние, не бросают исключений, не имеют побочных эффектов.
- Иерархия `WarehouseItem → StockItem` демонстрирует все четыре «наследственных» модификатора: `protected`, `internal`, `protected internal`, `private protected`. В `StockItem.Describe()` докажите, что из наследника в той же сборке видны все четыре.
- Типы верхнего уровня, не нужные потребителям (`StockRegistry`), — `internal`. Типы публичного контракта (`Product`, `WarehouseItem`, `StockItem`, `Warehouse`, `Demo`) — `public`.
- Коллекции наружу — только через `IReadOnlyCollection<T>` (или `IReadOnlyList<T>`). Возвращать внутренний `List<T>` напрямую или выставлять его через публичное поле запрещено.
- Код компилируется без предупреждений уровня Error (`TreatWarningsAsErrors` не обязателен, но приветствуется), проходит `dotnet build` и `dotnet test` зелёным.

#### Тонкости и подводные камни
- Не путайте `protected internal` (логика «ИЛИ»: доступен внутри сборки ИЛИ из наследника в любой сборке) и `private protected` (логика «И»: доступен только если код одновременно в сборке И является наследником). Урок явно подчёркивает эту разницу — выучите таблицу доступности.
- Модификатор члена всегда ограничен модификатором объемлющего типа: `public` поле внутри `internal` класса всё равно невидимо за пределами сборки. Поэтому ставить `public` на члены `internal`-типа можно (самодокументирование), но не нужно обманываться, будто это откроет доступ.
- Валидация должна жить в одном месте — в сеттере свойства. Если `Restock` и `Ship` будут дублировать проверку `>= 0` каждый, инвариант разойдётся между тремя точками. Пусть методы идут через сеттер: `Quantity += amount;` — тогда проверка автоматически срабатывает.
- `get` должен быть безопасным: никогда не валидируйте и не бросайте в нём исключения. Если валидация нужна при чтении — это сигнал, что состояние уже нарушено и нужно менять модель, а не маскировать через геттер.
- Возврат `List<T>` напрямую через публичное свойство ломает инкапсуляцию: внешний код вызовет `.Clear()` или `.Add()` и испортит внутреннее состояние. Возвращайте `IReadOnlyCollection<T>` — это проекция, а не копия, поэтому изменения внутреннего списка видны наружу (что обычно и нужно), но изменить коллекцию снаружи нельзя.
- `init`-сеттеры (C# 9+) удобны для immutable-данных, задаваемых в инициализаторе объекта, но в этом ДЗ требуются get-only автосвойства, задаваемые строго в конструкторе, — это даёт единую точку валидации аргументов конструктора.
- Не делайте `StockRegistry` публичным «на всякий случай» — это загрязняет публичный API библиотеки. Если позже понадобится доступ — откроете точечно, а вот закрыть публичный тип без ломающей совместимости уже нельзя.
- При использовании pattern matching `is { } item` помните: это `is not null and assign`. Удобнее и читабельнее, чем классическая `if (item != null)`, и страхует от `NullReferenceException`.

#### Критерии приёмки
- [ ] Solution собирается: `dotnet build` проходит без ошибок в трёх проектах + тестах.
- [ ] Все поля классов — `private`; ни одного публичного/`internal` поля.
- [ ] `UnitPrice` и `Quantity` — полные свойства с приватным сеттером и валидацией `>= 0`.
- [ ] Валидация бросает `ArgumentOutOfRangeException` с `nameof`.
- [ ] `Sku`, `Title`, `Id`, `CreatedAt` — get-only автосвойства, задаваемые только в конструкторе.
- [ ] Конструкторы `Product` и `StockItem` проверяют `null`-аргументы через `ArgumentNullException`.
- [ ] `WarehouseItem` содержит все четыре «наследственных» модификатора: `protected`, `internal`, `protected internal`, `private protected`.
- [ ] `StockItem.Describe()` обращается ко всем четырём, доказывая видимость из наследника в той же сборке.
- [ ] `StockRegistry` — `internal`; попытка обращения из `Warehouse.Plugin.Shipping` даёт `CS0122`.
- [ ] Коллекции наружу возвращаются как `IReadOnlyCollection<T>`; тест подтверждает, что `Items is List<StockItem>` == `false`.
- [ ] `Ship` сверх остатка бросает `InvalidOperationException`; тест зелёный.
- [ ] `Restock(-1)` бросает `ArgumentOutOfRangeException`; тест зелёный.
- [ ] Все get-аксессоры чистые — без побочных эффектов и исключений.
- [ ] `dotnet run --project src/Warehouse.App` выводит корректное состояние склада.
- [ ] Код — на C# 12 (.NET 8): file-scoped namespaces, top-level statements в `Program.cs`, pattern matching `is { }`.

#### Подсказки (без прямого ответа)
- Подумайте, почему проверка `value < 0` в сеттере `Quantity` автоматически защищает и `Restock`, и `Ship`, и любой будущий метод, который будет менять остаток.
- Если `StockItem.Describe()` «не видит» `CreatedAt` — проверьте, что `StockItem` действительно наследует от `WarehouseItem` и находится в той же сборке; `private protected` требует одновременного выполнения двух условий.
- Чтобы коллекция `Items` осталась проекцией, а не копией, просто верните ссылку на внутренний список, приведённую к `IReadOnlyCollection<T>` — каст не копирует.
- Для теста «не приводится к `List`» используйте `Assert.False(items is List<StockItem>);` — это проверка времени выполнения, а не компиляции.

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 — Эталонное решение ДЗ M04-L04
// Reference solution for homework M04-L04

using System;
using System.Collections.Generic;
using System.Linq;

namespace Warehouse.Core;

// Публичный контракт: товар с ценой и неизменяемым SKU.
// Public contract: a product with a price and an immutable SKU.
public sealed class Product
{
    private decimal _unitPrice; // backing field — всегда private / always private

    public string Sku { get; }       // get-only: задаётся только в конструкторе / set only in ctor
    public string Title { get; }     // immutable

    public decimal UnitPrice
    {
        get => _unitPrice;           // чистый геттер / pure getter
        private set                  // приватный сеттер с валидацией / private setter with validation
        {
            if (value < 0)
                throw new ArgumentOutOfRangeException(nameof(value), "Цена не может быть отрицательной. / Price cannot be negative.");
            _unitPrice = value;
        }
    }

    public Product(string sku, string title, decimal unitPrice)
    {
        Sku = sku ?? throw new ArgumentNullException(nameof(sku));
        Title = title ?? throw new ArgumentNullException(nameof(title));
        UnitPrice = unitPrice;        // идёт через сеттер → валидация / goes through setter → validation
    }

    public void ChangePrice(decimal newPrice) => UnitPrice = newPrice;
}

// Базовый класс демонстрирует все четыре «наследственных» модификатора.
// Base class demonstrates all four "inheritance" modifiers.
public abstract class WarehouseItem
{
    protected Guid Id { get; }                    // виден наследникам в любой сборке / visible to derived types in any assembly
    internal string Department { get; set; } = "Main"; // только внутри сборки / only within the assembly
    protected internal string Note { get; set; } = ""; // сборка ИЛИ наследник / assembly OR derived
    private protected DateTime CreatedAt { get; } // сборка И наследник / assembly AND derived

    protected WarehouseItem(Guid id) { Id = id; CreatedAt = DateTime.UtcNow; }

    public abstract string Describe();
}

// Наследник в той же сборке — видит все четыре.
// Derived type in the same assembly — sees all four.
public sealed class StockItem : WarehouseItem
{
    private int _quantity;

    public Product Product { get; }

    public int Quantity
    {
        get => _quantity;
        private set
        {
            if (value < 0)
                throw new ArgumentOutOfRangeException(nameof(value), "Остаток не может быть отрицательным. / Stock cannot be negative.");
            _quantity = value;
        }
    }

    public StockItem(Product product, int initialQuantity, Guid id) : base(id)
    {
        Product = product ?? throw new ArgumentNullException(nameof(product));
        Quantity = initialQuantity;
    }

    public void Restock(int amount)
    {
        if (amount <= 0) throw new ArgumentException("Количество должно быть положительным. / Amount must be positive.", nameof(amount));
        Quantity += amount; // через сеттер → инвариант проверяется в одном месте / via setter → invariant checked once
    }

    public void Ship(int amount)
    {
        if (amount <= 0) throw new ArgumentException("Количество должно быть положительным. / Amount must be positive.", nameof(amount));
        if (amount > Quantity) throw new InvalidOperationException("Недостаточно остатка. / Insufficient stock.");
        Quantity -= amount; // через сеттер / via setter
    }

    public override string Describe() =>
        // Из наследника в той же сборке видны Id, Department, Note, CreatedAt — все четыре модификатора.
        // From a derived type in the same assembly, Id, Department, Note, CreatedAt are all visible.
        $"{Id} / {Product.Sku}: {Product.Title}: {Quantity} шт. (отдел {Department}, заметка «{Note}», создан {CreatedAt:O})";
}

// internal-тип: не загрязняет публичный API. / internal type: does not pollute the public API.
internal sealed class StockRegistry
{
    private readonly List<StockItem> _items = new();

    internal void Register(StockItem item) => _items.Add(item);

    // public на члене internal-класса всё равно невидим снаружи, но самодокументирующий.
    // public on a member of an internal class is still invisible outside, yet self-documenting.
    public IReadOnlyCollection<StockItem> Items => _items;
}

// Публичный фасад библиотеки. / Public facade of the library.
public sealed class Warehouse
{
    private readonly StockRegistry _registry = new();

    public IReadOnlyCollection<StockItem> Items => _registry.Items; // проекция, не копия / projection, not a copy

    public void Add(StockItem item)
    {
        ArgumentNullException.ThrowIfNull(item);
        _registry.Register(item);
    }

    public void Ship(string sku, int amount)
    {
        if (_registry.Items.FirstOrDefault(i => i.Product.Sku == sku) is { } item)
            item.Ship(amount);
        else
            throw new InvalidOperationException($"Товар / Product {sku} не найден / not found.");
    }
}

public static class Demo
{
    public static void Run()
    {
        var product = new Product("STOCK-0001", "Винт М6 / M6 screw", 0.12m);
        var item = new StockItem(product, 100, Guid.NewGuid());
        item.Note = "Полка A3 / Shelf A3";

        var warehouse = new Warehouse();
        warehouse.Add(item);
        item.Restock(50);
        warehouse.Ship("STOCK-0001", 70);

        foreach (var s in warehouse.Items)
            Console.WriteLine(s.Describe());

        Console.WriteLine($"Всего позиций / Total items: {warehouse.Items.Count}");
    }
}
```

Разбор по строкам. `Product._unitPrice` — приватное поле-хранилище, единственное место, где реально хранится цена; всё остальное — проекция через свойство. `UnitPrice` — полное свойство с приватным сеттером: это реализует принцип из урока «поля — всегда private, публичный контракт — через свойства». Валидация `value < 0` в сеттере — единственная точка контроля, поэтому метод `ChangePrice` не дублирует проверку, а просто делегирует `UnitPrice = newPrice`. `Sku` и `Title` — get-only автосвойства (C# 6+), неизменяемые «по построению»: урок подчёркивает, что это путь к потокобезопасным immutable-объектам. Конструктор валидирует `null` через `ArgumentNullException` и присваивает `UnitPrice` через сеттер, поэтому конструктор тоже не дублирует проверку цены. `WarehouseItem` — абстрактный базовый класс, концентрирующий демонстрацию четырёх модификаторов: `protected Id` доступен наследникам даже в другой сборке; `internal Department` — только в своей сборке; `protected internal Note` — объединение (сборка ИЛИ наследник где угодно); `private protected CreatedAt` — пересечение (сборка И наследник). Урок прямо предостерегает от путаницы этих двух — здесь она намеренно показана рядом. `StockItem.Describe()` обращается ко всем четырём, доказывая, что из наследника в той же сборке видны все: это и есть конструктивная проверка таблицы доступности. `Quantity` — снова полное свойство с приватным сеттером и валидацией `>= 0`; `Restock` и `Ship` идут через `Quantity += amount` / `Quantity -= amount`, поэтому инвариант «остаток не отрицательный» проверяется ровно в одной точке — это применение best practice «валидация в одном месте». `Ship` дополнительно проверяет `amount > Quantity` и бросает `InvalidOperationException` (бизнес-ошибка, а не ошибка аргумента). `StockRegistry` — `internal`, что реализует принцип «internal для внутреннего API»: тип нужен только внутри `Warehouse.Core`, не должен попадать в публичный контракт. Его метод `Register` — `internal` (доступен в сборке), а свойство `Items` помечено `public` — это намеренный пример из урока: модификатор члена ограничен модификатором типа, поэтому `public` на члене `internal`-класса всё равно невидим снаружи, но сохраняет самодокументируемость. Возврат `IReadOnlyCollection<StockItem>` через `Items => _items` — проекция внутреннего списка, а не копия: внешний код не может вызвать `.Clear()` или `.Add()`, но видит актуальное состояние — это best practice урока «не открывай коллекции напрямую». В `Warehouse.Ship` pattern matching `is { } item` страхует от `NullReferenceException` и реализует современный идиоматичный C# 12. `ArgumentNullException.ThrowIfNull(item)` — компактный helper .NET 7+, эквивалент классической конструкции `?? throw new ArgumentNullException`. Top-level statements в `Program.cs` (не показаны в разборе, но в файле `Warehouse.App/Program.cs`) — это требование C# 12 и рекомендация урока по современному стилю. Совокупно решение применяет все семь модификаторов доступа, полный цикл приватное поле → валидирующее свойство → публичный метод → неизменяемая проекция коллекции, и каждый выбор модификатора обоснован принципом минимальной открытости.

#### Задания на углубление (бонус)
1. Добавьте `init`-сеттер для `Department` в `WarehouseItem` и сравните с get-only + приватный сеттер: когда `init` удобнее, а когда опаснее? Напишите короткое обоснование в комментариях.
2. Реализуйте второй наследник `WarehouseItem` — `ServiceItem` (услуга, не физический товар) в `Warehouse.Plugin.Shipping` (другая сборка). Убедитесь, что из него видны `Id` и `Note`, но НЕ видны `Department` и `CreatedAt`. Объясните, почему.
3. Добавьте кэш `internal static class PriceCache` с `Dictionary<string, decimal>` и публичным методом `internal decimal Get(string sku)`. Подумайте, как защитить внутренний словарь от модификации снаружи, не копируя его каждый раз.
4. Замените `IReadOnlyCollection<T>` на `IReadOnlyList<T>` и опишите в комментарии, какие новые возможности даёт индексатор `[index]` и какие новые риски открывает.

---

## Statement in English / Постановка на английском

#### Context & motivation
You are the engineer on a small team writing a warehouse-inventory library called `Warehouse.Core` that is consumed in two ways: a monolith application `Warehouse.App` living in the same solution, and a third-party plugin `Warehouse.Plugin.Shipping` compiled into a separate assembly and loaded at runtime through AssemblyLoadContext. The library must expose a minimal, deliberate public contract while keeping helper types, caches and algorithms inside, invisible to consumers — and ignorance of them is a feature, not a bug. The key invariant is absolute: the stock level of any product on any shelf can never go negative under any operation, and the list of transactions tied to a product is an immutable historical truth that no outside code may "tidy up". Every public setter that accepts a quantity or a price must validate its input and throw a meaningful, typed exception (`ArgumentException`, `ArgumentOutOfRangeException`) carrying the parameter name through `nameof`. Every getter must be pure: no side effects, no exceptions, no hidden state mutation. This is the textbook scenario where access modifiers are not cosmetic decoration but a correctness tool: `private` hides the fields, `private protected` opens an implementation seam only to "trusted" derived types inside the assembly, `internal` hides a helper type from the plugin, and `IReadOnlyCollection<T>` shields the internal list from outside mutation.

#### What to do step by step
1. Create a solution and three projects: `dotnet new sln -n Warehouse`, then `dotnet new classlib -n Warehouse.Core -o src/Warehouse.Core --framework net8.0`, `dotnet new console -n Warehouse.App -o src/Warehouse.App --framework net8.0`, `dotnet new classlib -n Warehouse.Plugin.Shipping -o src/Warehouse.Plugin.Shipping --framework net8.0`. Wire up references: `dotnet add src/Warehouse.App reference src/Warehouse.Core` and `dotnet add src/Warehouse.Plugin.Shipping reference src/Warehouse.Core`.
2. In `Warehouse.Core` enable `<Nullable>enable</Nullable>` and `<ImplicitUsings>enable</ImplicitUsings>` in the `.csproj` (C# 12 is the default for net8.0). Place every type in `namespace Warehouse.Core;` (file-scoped namespace, C# 10+).
3. Implement the `Product` class (in `Product.cs`): a private field `private decimal _unitPrice;` plus a full property `public decimal UnitPrice { get => _unitPrice; private set { if (value < 0) throw new ArgumentOutOfRangeException(nameof(value), "Price cannot be negative."); _unitPrice = value; } }`; get-only auto-properties `public string Sku { get; }` and `public string Title { get; }`; a method `public void ChangePrice(decimal newPrice)` that delegates to the setter. The constructor must validate `sku`/`title` against `null` via `ArgumentNullException` and assign `UnitPrice` through its setter.
4. Implement an abstract `public abstract class WarehouseItem` with `protected Guid Id { get; }`, `internal string Department { get; set; }`, `protected internal string Note { get; set; }`, `private protected DateTime CreatedAt { get; }`. The constructor takes `Guid id` and assigns `CreatedAt = DateTime.UtcNow`. Add `public abstract string Describe();`.
5. Implement `public sealed class StockItem : WarehouseItem` holding a `Product` and a quantity. Private field `private int _quantity;` with a full property `public int Quantity { get => _quantity; private set { if (value < 0) throw new ArgumentOutOfRangeException(nameof(value), "Stock cannot be negative."); _quantity = value; } }`. Methods `public void Restock(int amount)` and `public void Ship(int amount)` must go through the setter only, so the invariant is checked in exactly one place.
6. Implement `internal sealed class StockRegistry` — explicitly NOT public; it is needed only inside the assembly. It holds `private readonly List<StockItem> _items = new();`. A method `internal void Register(StockItem item)`. A property `public IReadOnlyCollection<StockItem> Items => _items;` — note that `public` on a member of an `internal` class is still invisible outside, but the code is self-documenting.
7. Implement `public sealed class Warehouse` with a private field `private readonly StockRegistry _registry = new();`. Public contract: `public IReadOnlyCollection<StockItem> Items => _registry.Items;`, `public void Add(StockItem item)`, `public void Ship(string sku, int amount)`. In `Ship` use pattern matching: `if (_registry.Items.FirstOrDefault(i => i.Product.Sku == sku) is { } item) item.Ship(amount); else throw new InvalidOperationException($"Product {sku} not found.");`.
8. Create `Demo.cs` with a `public static class Demo` and a `public static void Run()` method that demonstrates creating a product, an item, restocking, shipping, and printing `Describe()`.
9. In `Warehouse.App/Program.cs` use top-level statements: `using Warehouse.Core; Demo.Run();`. Run `dotnet run --project src/Warehouse.App` — the expected output includes lines like `STOCK-0001 / M6 screw: 80 pcs.`
10. In `Warehouse.Plugin.Shipping` attempt to reference `StockRegistry` — the compiler emits `CS0122` ("inaccessible due to its protection level"). This is the proof that `internal` works. Comment out or delete that line after the check.
11. Cover the invariants with unit tests in `dotnet new xunit -n Warehouse.Tests -o tests/Warehouse.Tests` referencing `Warehouse.Core`: a test that `Ship` beyond stock throws `InvalidOperationException`; a test that `Restock` with a negative `amount` throws `ArgumentOutOfRangeException`; a test that the `Items` collection does not downcast to `List<StockItem>` (assert that `is List<StockItem>` yields `false`).

#### Requirements
- Target framework is strictly .NET 8, language C# 12; `<Nullable>enable</Nullable>` and `<ImplicitUsings>enable</ImplicitUsings>` are on.
- All class fields are `private`. No public or `internal` fields. The public contract is exposed only through properties and methods.
- Every property that changes critical state (`UnitPrice`, `Quantity`) is a full property with a private setter and validation; the validation throws typed exceptions carrying `nameof`.
- Immutable data (`Sku`, `Title`, `Id`, `CreatedAt`) is exposed as get-only auto-properties assigned only in the constructor; there is no public setter on any of them.
- All get accessors are pure: they do not mutate state, do not throw, and have no side effects.
- The `WarehouseItem → StockItem` hierarchy demonstrates all four "inheritance" modifiers: `protected`, `internal`, `protected internal`, `private protected`. `StockItem.Describe()` must prove that all four are visible from a derived type in the same assembly.
- Top-level types not needed by consumers (`StockRegistry`) are `internal`. The types forming the public contract (`Product`, `WarehouseItem`, `StockItem`, `Warehouse`, `Demo`) are `public`.
- Collections are exposed to the outside only through `IReadOnlyCollection<T>` (or `IReadOnlyList<T>`). Returning the internal `List<T>` directly or via a public field is forbidden.
- The code compiles without Error-level warnings (`TreatWarningsAsErrors` is optional but welcome), passes `dotnet build`, and runs `dotnet test` green.

#### Pitfalls
- Do not confuse `protected internal` (OR logic: accessible inside the assembly OR from derived types anywhere) and `private protected` (AND logic: accessible only when the code is both inside the assembly AND a derived type). The lesson stresses this distinction explicitly — memorise the accessibility table.
- A member's modifier is always bounded by the modifier of the containing type: a `public` field inside an `internal` class is still unreachable outside the assembly. Putting `public` on members of an `internal` type is fine (self-documenting) but do not kid yourself that it widens access.
- Validation must live in one place — in the property setter. If `Restock` and `Ship` each duplicate the `>= 0` check, the invariant drifts across three sites. Let the methods go through the setter: `Quantity += amount;` — the check then fires automatically.
- A getter must be safe: never validate and never throw inside it. If reading seems to need validation, that is a signal that state is already broken and the model needs to change, not that the getter should mask it.
- Returning a `List<T>` directly through a public property breaks encapsulation: outside code will call `.Clear()` or `.Add()` and corrupt internal state. Return `IReadOnlyCollection<T>` — it is a projection, not a copy, so internal-list changes remain visible outside (usually what you want), but the collection cannot be mutated from outside.
- `init` setters (C# 9+) are handy for immutable data set through an object initialiser, but this homework specifically requires get-only auto-properties assigned strictly in the constructor — that gives a single validation point for constructor arguments.
- Do not make `StockRegistry` public "just in case" — that pollutes the library's public API. If you later need access, you can open it up narrowly; closing a public type later without a breaking change is impossible.
- When using the `is { } item` pattern remember: it means "is not null and assign". It is cleaner and safer than the classic `if (item != null)` and prevents `NullReferenceException`.

#### Acceptance criteria
- [ ] The solution builds: `dotnet build` is clean across the three projects plus tests.
- [ ] All class fields are `private`; there is no public/`internal` field anywhere.
- [ ] `UnitPrice` and `Quantity` are full properties with a private setter and `>= 0` validation.
- [ ] The validation throws `ArgumentOutOfRangeException` carrying `nameof`.
- [ ] `Sku`, `Title`, `Id`, `CreatedAt` are get-only auto-properties assigned only in the constructor.
- [ ] The `Product` and `StockItem` constructors check `null` arguments with `ArgumentNullException`.
- [ ] `WarehouseItem` carries all four "inheritance" modifiers: `protected`, `internal`, `protected internal`, `private protected`.
- [ ] `StockItem.Describe()` touches all four, proving visibility from a derived type in the same assembly.
- [ ] `StockRegistry` is `internal`; an access attempt from `Warehouse.Plugin.Shipping` yields `CS0122`.
- [ ] Collections are exposed to the outside as `IReadOnlyCollection<T>`; a test asserts `Items is List<StockItem>` == `false`.
- [ ] `Ship` beyond stock throws `InvalidOperationException`; the test is green.
- [ ] `Restock(-1)` throws `ArgumentOutOfRangeException`; the test is green.
- [ ] All get accessors are pure — no side effects, no exceptions.
- [ ] `dotnet run --project src/Warehouse.App` prints a correct warehouse state.
- [ ] The code is C# 12 (.NET 8): file-scoped namespaces, top-level statements in `Program.cs`, `is { }` pattern matching.

#### Hints (no direct answer)
- Consider why a `value < 0` check in the `Quantity` setter automatically protects both `Restock` and `Ship` and any future method that changes stock.
- If `StockItem.Describe()` "cannot see" `CreatedAt`, verify that `StockItem` really derives from `WarehouseItem` and lives in the same assembly; `private protected` requires both conditions at once.
- To keep `Items` a projection rather than a copy, return the internal list cast to `IReadOnlyCollection<T>` — the cast does not copy.
- For the "does not downcast to `List`" test, use `Assert.False(items is List<StockItem>);` — that is a runtime check, not a compile-time one.

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8 — Reference solution for homework M04-L04

using System;
using System.Collections.Generic;
using System.Linq;

namespace Warehouse.Core;

// Public contract: a product with a price and an immutable SKU.
public sealed class Product
{
    private decimal _unitPrice; // backing field — always private

    public string Sku { get; }       // get-only: assigned only in the constructor
    public string Title { get; }     // immutable

    public decimal UnitPrice
    {
        get => _unitPrice;           // pure getter
        private set                  // private setter with validation
        {
            if (value < 0)
                throw new ArgumentOutOfRangeException(nameof(value), "Price cannot be negative.");
            _unitPrice = value;
        }
    }

    public Product(string sku, string title, decimal unitPrice)
    {
        Sku = sku ?? throw new ArgumentNullException(nameof(sku));
        Title = title ?? throw new ArgumentNullException(nameof(title));
        UnitPrice = unitPrice;        // goes through the setter → validation
    }

    public void ChangePrice(decimal newPrice) => UnitPrice = newPrice;
}

// Base class demonstrating all four "inheritance" modifiers.
public abstract class WarehouseItem
{
    protected Guid Id { get; }                    // visible to derived types in any assembly
    internal string Department { get; set; } = "Main"; // only within the assembly
    protected internal string Note { get; set; } = ""; // assembly OR derived
    private protected DateTime CreatedAt { get; } // assembly AND derived

    protected WarehouseItem(Guid id) { Id = id; CreatedAt = DateTime.UtcNow; }

    public abstract string Describe();
}

// Derived type in the same assembly — sees all four.
public sealed class StockItem : WarehouseItem
{
    private int _quantity;

    public Product Product { get; }

    public int Quantity
    {
        get => _quantity;
        private set
        {
            if (value < 0)
                throw new ArgumentOutOfRangeException(nameof(value), "Stock cannot be negative.");
            _quantity = value;
        }
    }

    public StockItem(Product product, int initialQuantity, Guid id) : base(id)
    {
        Product = product ?? throw new ArgumentNullException(nameof(product));
        Quantity = initialQuantity;
    }

    public void Restock(int amount)
    {
        if (amount <= 0) throw new ArgumentException("Amount must be positive.", nameof(amount));
        Quantity += amount; // via setter → invariant checked in one place
    }

    public void Ship(int amount)
    {
        if (amount <= 0) throw new ArgumentException("Amount must be positive.", nameof(amount));
        if (amount > Quantity) throw new InvalidOperationException("Insufficient stock.");
        Quantity -= amount; // via setter
    }

    public override string Describe() =>
        // From a derived type in the same assembly, Id, Department, Note, CreatedAt are all visible.
        $"{Id} / {Product.Sku}: {Product.Title}: {Quantity} pcs. (dept {Department}, note \"{Note}\", created {CreatedAt:O})";
}

// internal type: does not pollute the public API.
internal sealed class StockRegistry
{
    private readonly List<StockItem> _items = new();

    internal void Register(StockItem item) => _items.Add(item);

    // public on a member of an internal class is still invisible outside, yet self-documenting.
    public IReadOnlyCollection<StockItem> Items => _items;
}

// Public facade of the library.
public sealed class Warehouse
{
    private readonly StockRegistry _registry = new();

    public IReadOnlyCollection<StockItem> Items => _registry.Items; // projection, not a copy

    public void Add(StockItem item)
    {
        ArgumentNullException.ThrowIfNull(item);
        _registry.Register(item);
    }

    public void Ship(string sku, int amount)
    {
        if (_registry.Items.FirstOrDefault(i => i.Product.Sku == sku) is { } item)
            item.Ship(amount);
        else
            throw new InvalidOperationException($"Product {sku} not found.");
    }
}

public static class Demo
{
    public static void Run()
    {
        var product = new Product("STOCK-0001", "M6 screw", 0.12m);
        var item = new StockItem(product, 100, Guid.NewGuid());
        item.Note = "Shelf A3";

        var warehouse = new Warehouse();
        warehouse.Add(item);
        item.Restock(50);
        warehouse.Ship("STOCK-0001", 70);

        foreach (var s in warehouse.Items)
            Console.WriteLine(s.Describe());

        Console.WriteLine($"Total items: {warehouse.Items.Count}");
    }
}
```

Line-by-line walk-through. `Product._unitPrice` is the private backing field — the single place the price is actually stored; everything else is a projection through the property. `UnitPrice` is a full property with a private setter, realising the lesson's principle "fields are always private; the public contract goes through properties". The `value < 0` check inside the setter is the single point of control, so `ChangePrice` does not duplicate it — it simply assigns `UnitPrice = newPrice`. `Sku` and `Title` are get-only auto-properties (C# 6+), immutable by construction: the lesson highlights this as the route to thread-safe immutable objects. The constructor validates `null` via `ArgumentNullException` and assigns `UnitPrice` through the setter, so the constructor also does not duplicate the price check. `WarehouseItem` is an abstract base class that concentrates the demonstration of all four modifiers: `protected Id` is visible to derived types even in another assembly; `internal Department` only within its own assembly; `protected internal Note` is a union (assembly OR derived anywhere); `private protected CreatedAt` is an intersection (assembly AND derived). The lesson explicitly warns against conflating these two — here they are shown side by side on purpose. `StockItem.Describe()` touches all four, proving that from a derived type in the same assembly every modifier is visible — a constructive check of the accessibility table. `Quantity` is again a full property with a private setter and a `>= 0` check; `Restock` and `Ship` go through `Quantity += amount` / `Quantity -= amount`, so the "stock never negative" invariant is verified at exactly one site — that is the best practice "validation in one place". `Ship` additionally checks `amount > Quantity` and throws `InvalidOperationException` (a business error, not an argument error). `StockRegistry` is `internal`, realising "internal for the internal API": the type is needed only inside `Warehouse.Core` and must not enter the public contract. Its `Register` method is `internal` (reachable in the assembly), and its `Items` property is marked `public` — an intentional example from the lesson: a member's modifier is bounded by the type's modifier, so `public` on a member of an `internal` class is still invisible outside, yet it keeps the code self-documenting. Returning `IReadOnlyCollection<StockItem>` via `Items => _items` is a projection of the internal list, not a copy: outside code cannot call `.Clear()` or `.Add()`, but it does see the up-to-date state — this is the lesson's best practice "do not expose collections directly". In `Warehouse.Ship` the `is { } item` pattern guards against `NullReferenceException` and embodies modern idiomatic C# 12. `ArgumentNullException.ThrowIfNull(item)` is the compact .NET 7+ helper, equivalent to the classic `?? throw new ArgumentNullException`. Top-level statements in `Program.cs` (not shown in the walk-through but present in `Warehouse.App/Program.cs`) are a C# 12 requirement and the lesson's preferred modern style. Taken together, the solution applies all seven access modifiers, the full loop private field → validating property → public method → immutable collection projection, and every modifier choice is justified by the principle of least privilege.

#### Going deeper (bonus)
1. Add an `init` setter for `Department` on `WarehouseItem` and contrast it with a get-only plus private setter: when is `init` more convenient, and when is it riskier? Write a short justification in the comments.
2. Implement a second `WarehouseItem` derived type — `ServiceItem` (a non-physical service) inside `Warehouse.Plugin.Shipping` (a different assembly). Confirm that from it `Id` and `Note` are visible but `Department` and `CreatedAt` are not. Explain why.
3. Add an `internal static class PriceCache` with a `Dictionary<string, decimal>` and a method `internal decimal Get(string sku)`. Reason about how to protect the internal dictionary from outside mutation without copying it on every call.
4. Replace `IReadOnlyCollection<T>` with `IReadOnlyList<T>` and describe in a comment what new capabilities the `[index]` indexer unlocks and what new risks it opens up.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Solution `Warehouse` содержит проекты `Warehouse.Core`, `Warehouse.App`, `Warehouse.Plugin.Shipping`, `Warehouse.Tests`.
- [ ] (RU) Все поля — `private`; публичный контракт — через свойства и методы.
- [ ] (RU) `UnitPrice` и `Quantity` — полные свойства с приватным сеттером и валидацией `>= 0`.
- [ ] (RU) `Sku`, `Title`, `Id`, `CreatedAt` — get-only автосвойства.
- [ ] (RU) `WarehouseItem` содержит `protected`, `internal`, `protected internal`, `private protected`.
- [ ] (RU) `StockRegistry` — `internal`; обращение из плагина даёт `CS0122`.
- [ ] (RU) Коллекции возвращаются как `IReadOnlyCollection<T>`.
- [ ] (RU) Юнит-тесты покрывают инварианты и зелёные.
- [ ] (EN) The `Warehouse` solution contains `Warehouse.Core`, `Warehouse.App`, `Warehouse.Plugin.Shipping`, `Warehouse.Tests`.
- [ ] (EN) All fields are `private`; the public contract is properties and methods.
- [ ] (EN) `UnitPrice` and `Quantity` are full properties with private setters and `>= 0` validation.
- [ ] (EN) `Sku`, `Title`, `Id`, `CreatedAt` are get-only auto-properties.
- [ ] (EN) `WarehouseItem` carries `protected`, `internal`, `protected internal`, `private protected`.
- [ ] (EN) `StockRegistry` is `internal`; access from the plugin yields `CS0122`.
- [ ] (EN) Collections are returned as `IReadOnlyCollection<T>`.
- [ ] (EN) Unit tests cover the invariants and are green.

#### Ресурсы / Resources
- [Access modifiers — Microsoft Learn](https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/access-modifiers)
- [Properties (C# Programming Guide) — Microsoft Learn](https://learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/properties)
- [Access Modifiers (C# Programming Guide) — Microsoft Learn](https://learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/access-modifiers)
- [init (C# Reference) — Microsoft Learn](https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/init)
- [IReadOnlyCollection&lt;T&gt; — .NET API Browser](https://learn.microsoft.com/dotnet/api/system.collections.generic.ireadonlycollection-1)

---

[⬆ К модулю M04](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
