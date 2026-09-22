---
[← К уроку M16-L09](lesson-M16-L09-ddd-overview.md) | [⬆ К модулю M16](../README.md) | [Следующее ДЗ →](homework-M16-L10-antipatterns.md)
---

### Домашнее задание M16-L09: Domain-Driven Design (обзор, Optional) / Homework M16-L09: Domain-Driven Design (overview, Optional)

**Урок / Lesson:** M16-L09
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться применять тактические строительные блоки DDD (Value Object, Entity, Aggregate Root, Repository) на C# 12 / .NET 8 на примере домена «Заказ», обеспечить инкапсуляцию инвариантов внутри агрегата, отказаться от анемичной модели и магических чисел, а также осознанно выбрать границы Bounded Context. (EN) Learn to apply the tactical DDD building blocks (Value Object, Entity, Aggregate Root, Repository) in C# 12 / .NET 8 using an Order domain; encapsulate invariants inside the aggregate, eliminate the anemic model and magic numbers, and consciously define Bounded Context boundaries.

#### Связь с уроком / Connection to the lesson
(RU) Урок вводит стратегическую и тактическую части DDD: Ubiquitous Language, Bounded Contexts, Entity, Value Object, Aggregate Root и Repository. ДЗ закрепляет тактические блоки на богатом доменном агрегате `Order`, повторяя пример урока, но расширяя его поведениями `Submit`, `Confirm`, `Cancel` и `ChangeLineQuantity`, а также отдельным Value Object `Address` для доставки, чтобы потренировать неизменяемые значения. Ключевая идея урока «моделируйте поведение, а не структуру данных» будет проверена через запрет публичных сеттеров и `UpdateStatus(int)`.
(EN) The lesson introduces the strategic and tactical parts of DDD: Ubiquitous Language, Bounded Contexts, Entity, Value Object, Aggregate Root and Repository. This homework consolidates the tactical blocks on a rich `Order` aggregate, mirroring the lesson example but extending it with `Submit`, `Confirm`, `Cancel` and `ChangeLineQuantity` behaviors plus a separate `Address` Value Object for delivery, to practice immutable values. The lesson's key idea — "model behavior, not data structure" — is enforced by forbidding public setters and `UpdateStatus(int)`.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Вы присоединяетесь к команде интернет-магазина, в котором «доменная логика» сегодня размазана по сервисам-транзакционным скриптам: `OrderService.UpdateStatus(int status)`, `OrderService.AddLine(orderId, productId, qty, price)`, а сам класс `Order` — это просто DTO с публичными сеттерами `Status`, `Lines`, `Total`. Это приводит к багам: заказ подтверждается пустым, строки добавляются в уже отменённый заказ, валюта строки не совпадает с валютой заказа, а слово «товар» (`Product`) используется одновременно как физическая коробка склада и как позиция каталога — класс «франкенштейн» со свойствами всех отделов.

Заказчик просит переписать домен заказа по DDD, потому что бизнес-правила становятся сложнее (скидки, бронирование остатков, отмена с возвратом), и разработчики устали держать их в голове в десяти сервисах. Ваша задача — построить тактическую модель агрегата `Order` так, чтобы инварианты заказа (не пуст, корректная валюта, корректные переходы статусов) проверялись внутри агрегата, а не в вызывающем коде. Дополнительно нужно определить Bounded Context для модуля «Заказы» и показать, почему один общий `Product` не подходит для склада и каталога.

Цель — почувствовать разницу между анемичной моделью (данные + сервисы) и богатой доменной моделью (поведение в объектах), закрепить Ubiquitous Language в именах методов (`Confirm`, `Submit`, `Cancel`, а не `UpdateStatus(2)`), научиться защищать инварианты через Aggregate Root и проектировать Repository, работающий с целыми агрегатами.

#### Что нужно сделать (пошагово)
1. Создайте решение и проект как консольное приложение .NET 8:
   ```
   dotnet new sln -n DddLab
   dotnet new console -n DddLab.Domain -o src/DddLab.Domain --framework net8.0
   dotnet new console -n DddLab.App -o src/DddLab.App --framework net8.0
   dotnet sln add src/DddLab.Domain src/DddLab.App
   dotnet add src/DddLab.App reference src/DddLab.Domain
   ```
   В `DddLab.Domain` будут жить доменные типы, в `DddLab.App` — демонстрационный сценарий.

2. В проекте `DddLab.Domain` создайте файл `Money.cs` с Value Object `Money` на основе `readonly record struct`: поля `Amount` и `Currency`, метод `Add(Money other)`, бросающий `InvalidOperationException` при разных валютах, статический `Zero(string currency)`. Реализуйте `SubTotal`/умножение через метод `Multiply(int quantity)`. Покройте равенство по значению — это даётся `record struct` автоматически, но убедитесь, что две `Money(100, "USD")` равны.

3. Создайте `Address.cs` — `readonly record` (класс) `Address(string Country, string City, string Street, string Zip)` с методом `WithCity(string city)`, возвращающим новый `Address`. Это тренирует неизменяемость Value Object: нельзя «поменять город» у заказа, можно только создать новый адрес.

4. Создайте `OrderStatus.cs` — `enum OrderStatus { Draft, Submitted, Confirmed, Shipped, Cancelled }`. Никаких магических чисел в API агрегата.

5. Создайте `OrderLine.cs` — внутреннюю сущность агрегата (`internal` конструктор), без публичных сеттеров. Свойства `ProductId`, `ProductName`, `UnitPrice`, `Quantity` (с `private set`). Метод `internal void ChangeQuantity(int newQuantity)` с проверкой положительности. `SubTotal` возвращает `Money`. Внешний код не должен иметь возможности создать `OrderLine` напрямую.

6. Создайте `Order.cs` — Aggregate Root. Приватный конструктор, статическая фабрика `Create(string currency, Address shippingAddress)`. Внутренний `List<OrderLine>` с публичным `ReadOnlyCollection<OrderLine> Lines`. Методы поведения: `AddLine`, `ChangeLineQuantity`, `RemoveLine`, `Submit`, `Confirm`, `Cancel`, `Ship`. Каждый метод вызывает `EnsureMutable()` и/или проверяет переход статуса. Реализуйте `Total` через `_lines.Aggregate(Money.Zero(Currency), (acc, l) => acc.Add(l.SubTotal))`. Запретите менять строки в статусах `Confirmed`, `Shipped`, `Cancelled`.

7. Создайте `IOrderRepository.cs` в домене: методы `Order? GetById(Guid id)`, `void Add(Order order)`, `void Save(Order order)`. Никакого `IQueryable`, никаких DTO — только целые агрегаты.

8. В `DddLab.App` реализуйте `InMemoryOrderRepository` (словарь `Guid -> Order`) и `Program.cs` с top-level statements, демонстрирующими: создание заказа → добавление строк → смена количества → `Submit` → `Confirm` → попытку изменить строку в `Confirmed` (должно бросить) → `Cancel` → попытку `Confirm` после `Cancel` (должно бросить). Выводите каждый шаг в консоль с `Console.WriteLine`.

9. Ожидаемый вывод сценария — последовательность шагов с сообщениями вида `Order created: ...`, `Order submitted`, `Order confirmed`, `Cannot modify order in status Confirmed`, `Order cancelled`, `Only a submitted order can be confirmed`. Убедитесь, что исключения перехватываются и печатаются, но не роняют программу.

10. В отдельном файле `BoundedContexts.md` (коротко, 150–250 слов) опишите, почему `Product` в складском контексте (коробка с габаритами, остаток) и `Product` в каталоге (карточка с фото, описание, цена) — это разные классы, и какие отношения между контекстами вы бы выбрали (Customer/Supplier, ACL и т.д.).

#### Требования к решению
- Целевая платформа: .NET 8, C# 12. Разрешено и поощряется: top-level statements, `record`/`record struct`, pattern matching (`is ... or ...`), collection expressions для инициализации, `init`-сеттеры, `required`, `raw string literals` в текстовых сообщениях при необходимости.
- Все доменные типы в проекте `DddLab.Domain`; приложение не содержит бизнес-правил — только оркестрацию сценария и репозиторий-заглушку.
- Запрещены публичные сеттеры у `Order` и `OrderLine`; изменять состояние можно только через методы поведения с семантическими именами.
- Запрещён метод `UpdateStatus(int)` или любой числовой/строковый аналог; переходы статусов инкапсулированы в `Submit`/`Confirm`/`Cancel`/`Ship`.
- Внешний код не должен создавать `OrderLine` напрямую и не должен держать ссылку на внутреннюю сущность, мутируемую снаружи (допускается читать через `Lines` как `ReadOnlyCollection`).
- Все инварианты (не пустой заказ при `Submit`/`Confirm`, одинаковая валюта, положительное количество, допустимый переход статуса) проверяются внутри агрегата и бросают `InvalidOperationException` с понятным сообщением.
- `IOrderRepository` возвращает и принимает только целые агрегаты `Order`; не возвращается `IQueryable`, `OrderDto` или `OrderLine`.
- Код компилируется без предупреждений (`dotnet build -warnaserror` желательно), `dotnet run` из `src/DddLab.App` выполняет сценарий и печатает ожидаемые шаги.

#### Тонкости и подводные камни
- **Анемичная модель — главная ловушка.** Если вы оставите у `Order` публичный `set` для `Status` или `Lines`, вы проиграете: любой код сможет выставить `Confirmed` пустому заказу. Запретите сеттеры и управляйте состоянием только методами поведения — это прямое следствие best practice урока «моделируйте поведение, а не данные».
- **Прямые ссылки на внутренние сущности.** Если вернуть `List<OrderLine>` с публичным аксессором, вызывающий код сделает `order.Lines[0].Quantity = 5`, и инвариант «корень контролирует все изменения» нарушится. Возвращайте `ReadOnlyCollection<OrderLine>` (как в уроке) или `IReadOnlyList<OrderLine>`, и держите конструктор `OrderLine` `internal`.
- **Валюта строки и валюта заказа.** Урок явно показывает проверку `unitPrice.Currency != Currency`. Не пропускайте её — иначе `Money.Add` упадёт при подсчёте `Total`, но уже поздно. Проверяйте на входе в `AddLine`.
- **«Гигантский агрегат».** Не тяните в `Order` клиента, склад, оплату. Достаточно `Guid ProductId` и `ProductName` (snapshot на момент строки) — это паттерн из урока: агрегат хранит то, что нужно для его инвариантов, остальное — по ID и через события.
- **Переходы статусов.** `Confirm` требует `Submitted`, `Ship` требует `Confirmed`, `Cancel` запрещён из `Shipped`. Используйте pattern matching `is OrderStatus.Confirmed or OrderStatus.Shipped` — это из примера урока и одновременно защита от «магии чисел».
- **Value Object и идентичность.** `Money` и `Address` не имеют `Id` и сравниваются по значению. Если случайно добавить `Id` в `Money` — это уже Entity, и вы нарушите тактическое правило.
- **Repository не должен утекать.** Возврат `IQueryable<Order>` превращает репозиторий в обёртку над EF и позволяет вызывающему коду писать бизнес-правила в виде LINQ-запросов вне домена. Возвращайте целые агрегаты; DTO стройте в слое приложения.
- **Одна транзакция = один агрегат.** Если в `BoundedContexts.md` вы предложите сохранять `Order` и `Inventory` в одной транзакции — это ошибка урока. Координация между агрегатами — через domain events и eventual consistency.

#### Критерии приёмки
- [ ] Решение `dotnet build` собирается без ошибок и предупреждений на .NET 8 / C# 12.
- [ ] `dotnet run --project src/DddLab.App` печатает сценарий с ожидаемыми сообщениями.
- [ ] `Money` — `readonly record struct`, неизменяемый, с `Add`, `Zero`, `Multiply`, равенство по значению.
- [ ] `Address` — `readonly record`, неизменяемый, с `WithCity`, без `Id`.
- [ ] `OrderStatus` — `enum`, в API агрегата нет числовых/строковых статусов.
- [ ] `OrderLine` — `internal` конструктор, без публичных сеттеров, `ChangeQuantity` с проверкой.
- [ ] `Order` — Aggregate Root с приватным конструктором и фабрикой `Create`.
- [ ] `Order.Lines` возвращает `ReadOnlyCollection<OrderLine>` (или `IReadOnlyList`), не мутируемую снаружи.
- [ ] `AddLine`, `ChangeLineQuantity`, `RemoveLine`, `Submit`, `Confirm`, `Cancel`, `Ship` — методы поведения с проверкой инвариантов.
- [ ] `EnsureMutable()` запрещает изменения в `Confirmed`/`Shipped`/`Cancelled`.
- [ ] `Submit`/`Confirm` бросают `InvalidOperationException` для пустого заказа.
- [ ] `AddLine` проверяет совпадение валюты строки и заказа.
- [ ] `IOrderRepository` работает с целыми агрегатами, без `IQueryable` и DTO.
- [ ] `BoundedContexts.md` объясняет разные `Product` в складском и каталоговом контекстах и выбирает отношение.
- [ ] В коде нет `UpdateStatus(int)`, публичных сеттеров состояния и прямых ссылок на внутренние сущности.

#### Подсказки (без прямого ответа)
- Подумайте, какой доступностью (`public`/`internal`/`private`) наделить конструктор `OrderLine`, чтобы его мог создавать только `Order`.
- Для подсчёта `Total` используйте `Aggregate` с начальным `Money.Zero(Currency)` — это повторяет пример урока и автоматически проверяет валюты через `Add`.
- Для `EnsureMutable()` pattern matching `is OrderStatus.Confirmed or OrderStatus.Shipped or OrderStatus.Cancelled` короче и читаемее цепочки `==`.
- В `InMemoryOrderRepository` храните словарь `Guid -> Order`; для имитации «сохранения» можно просто перезаписывать значение по ключу.
- Помните: «snapshot» имени товара в `OrderLine.ProductName` — это не дублирование данных, а фиксация данных на момент заказа (товар в каталоге мог переименовать).

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8 — Тактические блоки DDD: агрегат Order с инкапсулированными инвариантами
// Tactical DDD blocks: Order aggregate with encapsulated invariants

using System.Collections.ObjectModel;

// Value Object: неизменяемый, сравнение по значению, без идентичности
// Value Object: immutable, compared by value, no identity
public readonly record struct Money(decimal Amount, string Currency)
{
    public static Money Zero(string currency) => new(0m, currency);

    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException(
                "Нельзя складывать разные валюты / Cannot add different currencies");
        return this with { Amount = Amount + other.Amount };
    }

    public Money Multiply(int factor) => this with { Amount = Amount * factor };
}

// Value Object адреса доставки — неизменяемый, меняется через With*
// Delivery address Value Object — immutable, changed via With*
public sealed record Address(string Country, string City, string Street, string Zip)
{
    public Address WithCity(string city) => this with { City = city };
}

// Перечисление статусов — вместо магических чисел
// Status enumeration — instead of magic numbers
public enum OrderStatus { Draft, Submitted, Confirmed, Shipped, Cancelled }

// Внутренняя сущность агрегата — внешний код не создаёт и не мутирует напрямую
// Internal entity of the aggregate — external code neither creates nor mutates directly
public sealed class OrderLine
{
    public Guid ProductId { get; }
    public string ProductName { get; }   // snapshot на момент строки / snapshot at line time
    public Money UnitPrice { get; }
    public int Quantity { get; private set; }

    internal OrderLine(Guid productId, string productName, Money unitPrice, int quantity)
    {
        if (quantity <= 0)
            throw new InvalidOperationException(
                "Количество должно быть положительным / Quantity must be positive");
        ProductId = productId;
        ProductName = productName;
        UnitPrice = unitPrice;
        Quantity = quantity;
    }

    public Money SubTotal => UnitPrice.Multiply(Quantity);

    internal void ChangeQuantity(int newQuantity)
    {
        if (newQuantity <= 0)
            throw new InvalidOperationException(
                "Количество должно быть положительным / Quantity must be positive");
        Quantity = newQuantity;
    }
}

// Aggregate Root — единственная точка доступа, защищает все инварианты
// Aggregate Root — the single entry point, protects all invariants
public sealed class Order
{
    public Guid Id { get; }
    public OrderStatus Status { get; private set; }
    public string Currency { get; }
    public Address ShippingAddress { get; private set; }
    private readonly List<OrderLine> _lines = new();
    public ReadOnlyCollection<OrderLine> Lines => _lines.AsReadOnly();

    public Money Total => _lines.Aggregate(
        Money.Zero(Currency),
        (acc, line) => acc.Add(line.SubTotal));

    private Order(Guid id, string currency, Address shippingAddress)
    {
        Id = id;
        Currency = currency;
        ShippingAddress = shippingAddress;
        Status = OrderStatus.Draft;
    }

    // Фабрика — создаёт агрегат в валидном начальном состоянии
    // Factory — creates the aggregate in a valid initial state
    public static Order Create(string currency, Address shippingAddress)
    {
        if (string.IsNullOrWhiteSpace(currency))
            throw new InvalidOperationException("Валюта обязательна / Currency is required");
        return new(Guid.NewGuid(), currency, shippingAddress);
    }

    public void AddLine(Guid productId, string productName, Money unitPrice, int quantity)
    {
        EnsureMutable();
        if (unitPrice.Currency != Currency)
            throw new InvalidOperationException(
                "Валюта строки не совпадает с валютой заказа / Line currency differs from order currency");
        _lines.Add(new OrderLine(productId, productName, unitPrice, quantity));
    }

    public void ChangeLineQuantity(Guid productId, int newQuantity)
    {
        EnsureMutable();
        var line = _lines.Single(l => l.ProductId == productId);
        line.ChangeQuantity(newQuantity);
    }

    public void RemoveLine(Guid productId)
    {
        EnsureMutable();
        _lines.RemoveAll(l => l.ProductId == productId);
    }

    public void UpdateShippingAddress(Address newAddress)
    {
        EnsureMutable();
        ShippingAddress = newAddress;
    }

    public void Submit()
    {
        EnsureMutable();
        if (_lines.Count == 0)
            throw new InvalidOperationException(
                "Нельзя отправить пустой заказ / Cannot submit an empty order");
        Status = OrderStatus.Submitted;
    }

    public void Confirm()
    {
        if (Status != OrderStatus.Submitted)
            throw new InvalidOperationException(
                "Только отправленный заказ можно подтвердить / Only a submitted order can be confirmed");
        if (_lines.Count == 0)
            throw new InvalidOperationException(
                "Нельзя подтвердить пустой заказ / Cannot confirm an empty order");
        Status = OrderStatus.Confirmed;
    }

    public void Ship()
    {
        if (Status != OrderStatus.Confirmed)
            throw new InvalidOperationException(
                "Только подтверждённый заказ можно отправить / Only a confirmed order can be shipped");
        Status = OrderStatus.Shipped;
    }

    public void Cancel()
    {
        if (Status is OrderStatus.Shipped)
            throw new InvalidOperationException(
                "Отправленный заказ нельзя отменить / Cannot cancel a shipped order");
        Status = OrderStatus.Cancelled;
    }

    private void EnsureMutable()
    {
        if (Status is OrderStatus.Confirmed or OrderStatus.Shipped or OrderStatus.Cancelled)
            throw new InvalidOperationException(
                "Заказ в этом статусе нельзя изменять / Order cannot be modified in this status");
    }
}

// Repository — доменный интерфейс, работает с целыми агрегатами
// Repository — domain interface, works with whole aggregates
public interface IOrderRepository
{
    Order? GetById(Guid id);
    void Add(Order order);
    void Save(Order order);
}
```

Разбор по строкам. `Money` объявлен как `readonly record struct` — это даёт неизменяемость и автоматическое сравнение по значению, что и требуется от Value Object по уроку. Метод `Add` использует `this with { ... }` — синтаксис C# 12 для создания копии с изменённым полем, что физически гарантирует неизменяемость. Проверка валюты здесь же защищает инвариант «нельзя складывать деньги в разных валютах» — это пример правила, которое должно жить в самом объекте, а не в сервисе.

`Address` — `sealed record`, тоже Value Object: у него нет `Id`, он сравнивается по значениям полей. Метод `WithCity` возвращает новый `Address`, не мутируя исходный. Это тренирует привычку «изменить значение = создать новое».

`OrderStatus` — `enum`, чтобы в API не было числовых статусов. Это прямо закрывает ошибку урока «`UpdateStatus(int)` и магические числа».

`OrderLine` имеет `internal` конструктор: внешний код не может создать строку в обход `Order`. `Quantity` — `private set`, меняется только через `internal ChangeQuantity` с проверкой положительности. `SubTotal` считает `UnitPrice.Multiply(Quantity)` — вся арифметика денег инкапсулирована в `Money`.

`Order` — Aggregate Root: приватный конструктор, статическая фабрика `Create` для валидного начального состояния. `Lines` возвращает `ReadOnlyCollection<OrderLine>`, что закрывает возможность `order.Lines[0].Quantity = 5` из внешнего кода — это прямой ответ на частую ошибку «прямые ссылки на внутренние сущности». `_lines` — `private readonly List`, единственный источник правды. `Total` через `Aggregate` с `Money.Zero(Currency)` автоматически проверяет валюты через `Add`, повторяя пример урока.

Поведенческие методы `Submit`, `Confirm`, `Ship`, `Cancel` кодируют конечный автомат статусов: каждый проверяет допустимый переход и бросает `InvalidOperationException` с понятным сообщением — это реализация Ubiquitous Language: метод называется `Confirm`, как сказал бы эксперт, а не `UpdateStatus(2)`. `EnsureMutable()` через pattern matching `is OrderStatus.Confirmed or OrderStatus.Shipped or OrderStatus.Cancelled` централизованно защищает «нельзя менять после подтверждения».

`IOrderRepository` возвращает целые `Order` и принимает целые `Order` — нет `IQueryable`, нет DTO. Реализация живёт в инфраструктуре (в ДЗ — `InMemoryOrderRepository` в `DddLab.App`), интерфейс — в домене, как требует best practice урока.

#### Задания на углубление (бонус)
1. Добавьте Value Object `Discount` (процент или фиксированная сумма в валюте) и метод `Order.ApplyDiscount(Discount)`, который пересчитывает `Total` через отдельный `Money`-расчёт, не мутируя строки. Подумайте, где живут правила скидок — в агрегате или в отдельном доменном сервисе.
2. Введите domain events: после `Confirm` агрегат публикует `OrderConfirmed` (просто `record OrderConfirmed(Guid OrderId, DateTime At)`), а в `DddLab.App` — простой `IEventDispatcher`, который «резервирует остатки на складе» (печатью в консоль). Это покажет, как согласовывать несколько агрегатов без общей транзакции.
3. Смоделируйте второй Bounded Context «Каталог» с собственным `Product` (карточка) и его `Product`-ID, который совпадает с `OrderLine.ProductId` по значению, но типы разные. Реализуйте Anti-corruption Layer — маппер, который переводит `Catalog.Product` в snapshot для `OrderLine`.
4. Покройте агрегат юнит-тестами xUnit: позитивные сценарии (create → submit → confirm → ship) и негативные (confirm пустого, mutate после confirmed, cancel shipped). Используйте `FluentAssertions` для читаемости ассертов.

---

## Statement in English / Постановка на английском

#### Context & motivation
You join an e-commerce team where the “domain logic” today is scattered across transaction-script services: `OrderService.UpdateStatus(int status)`, `OrderService.AddLine(orderId, productId, qty, price)`, and the `Order` class itself is a mere DTO with public setters on `Status`, `Lines`, `Total`. This produces bugs: orders get confirmed while empty, lines are added to already-cancelled orders, a line’s currency does not match the order currency, and the word “product” (`Product`) is used both as a warehouse box and as a catalog listing — a “Frankenstein” class that carries properties of every department.

The customer asks to rewrite the order domain following DDD, because business rules are getting more complex (discounts, stock reservation, refunds on cancel) and developers are tired of holding them in their heads across ten services. Your task is to build the tactical model of the `Order` aggregate so that the order invariants (non-empty on submit/confirm, single currency, positive quantity, valid status transition) are enforced inside the aggregate, not in the calling code. You also need to define a Bounded Context for the Orders module and show why a single shared `Product` does not fit both warehouse and catalog.

The goal is to feel the difference between an anemic model (data + services) and a rich domain model (behavior inside objects), to bake Ubiquitous Language into method names (`Confirm`, `Submit`, `Cancel`, never `UpdateStatus(2)`), to protect invariants through the Aggregate Root, and to design a Repository that works with whole aggregates.

#### What to do step by step
1. Create a solution and projects as .NET 8 console apps:
   ```
   dotnet new sln -n DddLab
   dotnet new console -n DddLab.Domain -o src/DddLab.Domain --framework net8.0
   dotnet new console -n DddLab.App -o src/DddLab.App --framework net8.0
   dotnet sln add src/DddLab.Domain src/DddLab.App
   dotnet add src/DddLab.App reference src/DddLab.Domain
   ```
   `DddLab.Domain` holds the domain types; `DddLab.App` holds the demo scenario.

2. In `DddLab.Domain`, create `Money.cs` as a Value Object based on `readonly record struct`: fields `Amount` and `Currency`, method `Add(Money other)` throwing `InvalidOperationException` for mismatched currencies, a static `Zero(string currency)`. Implement multiplication through `Multiply(int quantity)`. Rely on `record struct` for value equality — but explicitly assert that two `Money(100, "USD")` are equal.

3. Create `Address.cs` — a `readonly record` (class) `Address(string Country, string City, string Street, string Zip)` with a `WithCity(string city)` method returning a new `Address`. This trains Value Object immutability: you cannot “change the city” of an order, you can only produce a new address.

4. Create `OrderStatus.cs` — `enum OrderStatus { Draft, Submitted, Confirmed, Shipped, Cancelled }`. No magic numbers in the aggregate API.

5. Create `OrderLine.cs` — an internal entity of the aggregate (`internal` constructor), with no public setters. Properties `ProductId`, `ProductName`, `UnitPrice`, `Quantity` (with `private set`). Method `internal void ChangeQuantity(int newQuantity)` validates positivity. `SubTotal` returns `Money`. External code must not be able to instantiate `OrderLine` directly.

6. Create `Order.cs` — the Aggregate Root. Private constructor, static factory `Create(string currency, Address shippingAddress)`. Internal `List<OrderLine>` exposed through a public `ReadOnlyCollection<OrderLine> Lines`. Behavior methods: `AddLine`, `ChangeLineQuantity`, `RemoveLine`, `Submit`, `Confirm`, `Cancel`, `Ship`. Each method calls `EnsureMutable()` and/or validates the status transition. Implement `Total` as `_lines.Aggregate(Money.Zero(Currency), (acc, l) => acc.Add(l.SubTotal))`. Forbid line changes in `Confirmed`, `Shipped`, `Cancelled` states.

7. Create `IOrderRepository.cs` in the domain: methods `Order? GetById(Guid id)`, `void Add(Order order)`, `void Save(Order order)`. No `IQueryable`, no DTOs — only whole aggregates.

8. In `DddLab.App`, implement `InMemoryOrderRepository` (a `Guid -> Order` dictionary) and `Program.cs` with top-level statements demonstrating: create order → add lines → change quantity → `Submit` → `Confirm` → try to modify a line in `Confirmed` (must throw) → `Cancel` → try to `Confirm` after `Cancel` (must throw). Print every step with `Console.WriteLine`.

9. The expected output is a sequence of messages like `Order created: ...`, `Order submitted`, `Order confirmed`, `Cannot modify order in status Confirmed`, `Order cancelled`, `Only a submitted order can be confirmed`. Make sure exceptions are caught and printed without crashing the program.

10. In a separate `BoundedContexts.md` file (150–250 words), explain why `Product` in the warehouse context (a physical box with dimensions and stock) and `Product` in the catalog (a listing with photo, description, price) are different classes, and which inter-context relationship you would choose (Customer/Supplier, ACL, etc.).

#### Requirements
- Target platform: .NET 8, C# 12. Allowed and encouraged: top-level statements, `record`/`record struct`, pattern matching (`is ... or ...`), collection expressions for initialization, `init` setters, `required`, `raw string literals` for messages where useful.
- All domain types live in the `DddLab.Domain` project; the application project contains no business rules — only scenario orchestration and an in-memory repository stub.
- Public setters on `Order` and `OrderLine` are forbidden; state can only change through semantic behavior methods.
- `UpdateStatus(int)` (or any numeric/string equivalent) is forbidden; status transitions are encapsulated in `Submit`/`Confirm`/`Cancel`/`Ship`.
- External code must not construct `OrderLine` directly and must not hold a reference to an internal entity that it can mutate from the outside (reading through `Lines` as `ReadOnlyCollection` is allowed).
- All invariants (non-empty order on `Submit`/`Confirm`, single currency, positive quantity, valid status transition) are enforced inside the aggregate and throw `InvalidOperationException` with a clear message.
- `IOrderRepository` returns and accepts only whole `Order` aggregates; it never returns `IQueryable`, `OrderDto`, or `OrderLine`.
- The code compiles without warnings (`dotnet build -warnaserror` is desirable); `dotnet run` from `src/DddLab.App` executes the scenario and prints the expected steps.

#### Pitfalls
- **The anemic model is the main trap.** If you leave `Status` or `Lines` with a public setter on `Order`, you lose: any code can set `Confirmed` on an empty order. Forbid setters and drive state only through behavior methods — this is the direct consequence of the lesson’s best practice “model behavior, not data”.
- **Direct references to internal entities.** If you expose `List<OrderLine>` with a public accessor, the caller writes `order.Lines[0].Quantity = 5` and the “root controls all mutations” invariant is broken. Return `ReadOnlyCollection<OrderLine>` (as in the lesson) or `IReadOnlyList<OrderLine>`, and keep the `OrderLine` constructor `internal`.
- **Line currency vs order currency.** The lesson explicitly checks `unitPrice.Currency != Currency`. Do not skip it — otherwise `Money.Add` will throw while computing `Total`, but too late. Validate at the `AddLine` entry.
- **The giant aggregate.** Do not pull customer, warehouse, and payment into `Order`. A `Guid ProductId` plus a `ProductName` snapshot is enough — this is the lesson pattern: the aggregate stores what its invariants need; the rest is IDs and events.
- **Status transitions.** `Confirm` requires `Submitted`, `Ship` requires `Confirmed`, `Cancel` is forbidden from `Shipped`. Use pattern matching `is OrderStatus.Confirmed or OrderStatus.Shipped` — straight from the lesson example and a guard against “number magic”.
- **Value Object and identity.** `Money` and `Address` have no `Id` and compare by value. If you accidentally add `Id` to `Money`, it becomes an Entity and you break the tactical rule.
- **Repository must not leak.** Returning `IQueryable<Order>` turns the repository into an EF wrapper and lets calling code write business rules as LINQ outside the domain. Return whole aggregates; build DTOs in the application layer.
- **One transaction = one aggregate.** If your `BoundedContexts.md` proposes saving `Order` and `Inventory` in one transaction, that contradicts the lesson. Cross-aggregate coordination goes through domain events and eventual consistency.

#### Acceptance criteria
- [ ] `dotnet build` succeeds with no errors and no warnings on .NET 8 / C# 12.
- [ ] `dotnet run --project src/DddLab.App` prints the scenario with the expected messages.
- [ ] `Money` is a `readonly record struct`, immutable, with `Add`, `Zero`, `Multiply`, value equality.
- [ ] `Address` is a `readonly record`, immutable, with `WithCity`, no `Id`.
- [ ] `OrderStatus` is an `enum`; the aggregate API has no numeric/string statuses.
- [ ] `OrderLine` has an `internal` constructor, no public setters, a `ChangeQuantity` with validation.
- [ ] `Order` is an Aggregate Root with a private constructor and a `Create` factory.
- [ ] `Order.Lines` returns a `ReadOnlyCollection<OrderLine>` (or `IReadOnlyList`) not mutable from the outside.
- [ ] `AddLine`, `ChangeLineQuantity`, `RemoveLine`, `Submit`, `Confirm`, `Cancel`, `Ship` are behavior methods that validate invariants.
- [ ] `EnsureMutable()` forbids changes in `Confirmed`/`Shipped`/`Cancelled`.
- [ ] `Submit`/`Confirm` throw `InvalidOperationException` for an empty order.
- [ ] `AddLine` checks that the line currency matches the order currency.
- [ ] `IOrderRepository` works with whole aggregates, no `IQueryable` and no DTOs.
- [ ] `BoundedContexts.md` explains the different `Product` classes in warehouse and catalog contexts and chooses a relationship.
- [ ] The code has no `UpdateStatus(int)`, no public state setters, no direct references to internal entities.

#### Hints (no direct answer)
- Think about which accessibility (`public`/`internal`/`private`) the `OrderLine` constructor needs so that only `Order` can create it.
- For `Total`, use `Aggregate` starting from `Money.Zero(Currency)` — this mirrors the lesson and automatically validates currencies through `Add`.
- For `EnsureMutable()`, the pattern `is OrderStatus.Confirmed or OrderStatus.Shipped or OrderStatus.Cancelled` is shorter and clearer than a chain of `==`.
- In `InMemoryOrderRepository`, keep a `Guid -> Order` dictionary; for a “save”, just overwrite the value by key.
- Remember: the `ProductName` snapshot in `OrderLine` is not duplication — it fixes the data at order time (the catalog product may have been renamed later).

#### Reference solution walk-through

```csharp
// C# 12 / .NET 8 — Tactical DDD blocks: Order aggregate with encapsulated invariants

using System.Collections.ObjectModel;

// Value Object: immutable, compared by value, no identity
public readonly record struct Money(decimal Amount, string Currency)
{
    public static Money Zero(string currency) => new(0m, currency);

    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException(
                "Cannot add different currencies / Нельзя складывать разные валюты");
        return this with { Amount = Amount + other.Amount };
    }

    public Money Multiply(int factor) => this with { Amount = Amount * factor };
}

// Delivery address Value Object — immutable, changed via With*
public sealed record Address(string Country, string City, string Street, string Zip)
{
    public Address WithCity(string city) => this with { City = city };
}

// Status enumeration — instead of magic numbers
public enum OrderStatus { Draft, Submitted, Confirmed, Shipped, Cancelled }

// Internal entity of the aggregate — external code neither creates nor mutates directly
public sealed class OrderLine
{
    public Guid ProductId { get; }
    public string ProductName { get; }   // snapshot at line time
    public Money UnitPrice { get; }
    public int Quantity { get; private set; }

    internal OrderLine(Guid productId, string productName, Money unitPrice, int quantity)
    {
        if (quantity <= 0)
            throw new InvalidOperationException(
                "Quantity must be positive / Количество должно быть положительным");
        ProductId = productId;
        ProductName = productName;
        UnitPrice = unitPrice;
        Quantity = quantity;
    }

    public Money SubTotal => UnitPrice.Multiply(Quantity);

    internal void ChangeQuantity(int newQuantity)
    {
        if (newQuantity <= 0)
            throw new InvalidOperationException(
                "Quantity must be positive / Количество должно быть положительным");
        Quantity = newQuantity;
    }
}

// Aggregate Root — the single entry point, protects all invariants
public sealed class Order
{
    public Guid Id { get; }
    public OrderStatus Status { get; private set; }
    public string Currency { get; }
    public Address ShippingAddress { get; private set; }
    private readonly List<OrderLine> _lines = new();
    public ReadOnlyCollection<OrderLine> Lines => _lines.AsReadOnly();

    public Money Total => _lines.Aggregate(
        Money.Zero(Currency),
        (acc, line) => acc.Add(line.SubTotal));

    private Order(Guid id, string currency, Address shippingAddress)
    {
        Id = id;
        Currency = currency;
        ShippingAddress = shippingAddress;
        Status = OrderStatus.Draft;
    }

    // Factory — creates the aggregate in a valid initial state
    public static Order Create(string currency, Address shippingAddress)
    {
        if (string.IsNullOrWhiteSpace(currency))
            throw new InvalidOperationException("Currency is required / Валюта обязательна");
        return new(Guid.NewGuid(), currency, shippingAddress);
    }

    public void AddLine(Guid productId, string productName, Money unitPrice, int quantity)
    {
        EnsureMutable();
        if (unitPrice.Currency != Currency)
            throw new InvalidOperationException(
                "Line currency differs from order currency / Валюта строки не совпадает с валютой заказа");
        _lines.Add(new OrderLine(productId, productName, unitPrice, quantity));
    }

    public void ChangeLineQuantity(Guid productId, int newQuantity)
    {
        EnsureMutable();
        var line = _lines.Single(l => l.ProductId == productId);
        line.ChangeQuantity(newQuantity);
    }

    public void RemoveLine(Guid productId)
    {
        EnsureMutable();
        _lines.RemoveAll(l => l.ProductId == productId);
    }

    public void UpdateShippingAddress(Address newAddress)
    {
        EnsureMutable();
        ShippingAddress = newAddress;
    }

    public void Submit()
    {
        EnsureMutable();
        if (_lines.Count == 0)
            throw new InvalidOperationException(
                "Cannot submit an empty order / Нельзя отправить пустой заказ");
        Status = OrderStatus.Submitted;
    }

    public void Confirm()
    {
        if (Status != OrderStatus.Submitted)
            throw new InvalidOperationException(
                "Only a submitted order can be confirmed / Только отправленный заказ можно подтвердить");
        if (_lines.Count == 0)
            throw new InvalidOperationException(
                "Cannot confirm an empty order / Нельзя подтвердить пустой заказ");
        Status = OrderStatus.Confirmed;
    }

    public void Ship()
    {
        if (Status != OrderStatus.Confirmed)
            throw new InvalidOperationException(
                "Only a confirmed order can be shipped / Только подтверждённый заказ можно отправить");
        Status = OrderStatus.Shipped;
    }

    public void Cancel()
    {
        if (Status is OrderStatus.Shipped)
            throw new InvalidOperationException(
                "Cannot cancel a shipped order / Отправленный заказ нельзя отменить");
        Status = OrderStatus.Cancelled;
    }

    private void EnsureMutable()
    {
        if (Status is OrderStatus.Confirmed or OrderStatus.Shipped or OrderStatus.Cancelled)
            throw new InvalidOperationException(
                "Order cannot be modified in this status / Заказ в этом статусе нельзя изменять");
    }
}

// Repository — domain interface, works with whole aggregates
public interface IOrderRepository
{
    Order? GetById(Guid id);
    void Add(Order order);
    void Save(Order order);
}
```

Walk-through. `Money` is a `readonly record struct` — this gives immutability and automatic value equality, exactly what a Value Object needs per the lesson. `Add` uses `this with { ... }` — the C# 12 syntax for a copy with a changed field, which physically enforces immutability. The currency check inside `Add` protects the “cannot add different currencies” invariant — an example of a rule that lives in the object itself, not in a service.

`Address` is a `sealed record`, also a Value Object: it has no `Id` and compares by field values. `WithCity` returns a new `Address` without mutating the original, training the “to change a value is to create a new one” habit.

`OrderStatus` is an `enum`, so the API has no numeric statuses — this directly closes the lesson’s “`UpdateStatus(int)` and magic numbers” mistake.

`OrderLine` has an `internal` constructor: external code cannot create a line bypassing `Order`. `Quantity` is `private set`, changed only through the `internal ChangeQuantity` with positivity validation. `SubTotal` computes `UnitPrice.Multiply(Quantity)`, so all money arithmetic stays inside `Money`.

`Order` is the Aggregate Root: private constructor, static `Create` factory for a valid initial state. `Lines` returns `ReadOnlyCollection<OrderLine>`, closing the door on `order.Lines[0].Quantity = 5` from outside — a direct answer to the common mistake “direct references to internal entities”. `_lines` is the single source of truth. `Total` via `Aggregate` from `Money.Zero(Currency)` automatically validates currencies through `Add`, mirroring the lesson.

Behavior methods `Submit`, `Confirm`, `Ship`, `Cancel` encode the status state machine: each validates the allowed transition and throws `InvalidOperationException` with a clear message — this is Ubiquitous Language in action: the method is called `Confirm` as an expert would say, not `UpdateStatus(2)`. `EnsureMutable()` with pattern matching `is OrderStatus.Confirmed or OrderStatus.Shipped or OrderStatus.Cancelled` centrally enforces “no changes after confirmation”.

`IOrderRepository` returns whole `Order` aggregates and accepts whole `Order` aggregates — no `IQueryable`, no DTOs. The implementation lives in infrastructure (in this homework, `InMemoryOrderRepository` in `DddLab.App`), the interface in the domain, exactly as the lesson’s best practice demands.

#### Going deeper (bonus)
1. Add a `Discount` Value Object (percentage or a fixed amount in a currency) and an `Order.ApplyDiscount(Discount)` method that recomputes `Total` through a separate `Money` calculation without mutating the lines. Think about where the discount rules live — in the aggregate or in a separate domain service.
2. Introduce domain events: after `Confirm`, the aggregate publishes `OrderConfirmed` (a simple `record OrderConfirmed(Guid OrderId, DateTime At)`), and in `DddLab.App` a minimal `IEventDispatcher` “reserves stock” (by printing to the console). This shows how to coordinate multiple aggregates without a shared transaction.
3. Model a second Bounded Context “Catalog” with its own `Product` (a listing) whose `Product` ID matches `OrderLine.ProductId` by value, but the types are different. Implement an Anti-corruption Layer — a mapper that turns a `Catalog.Product` into a snapshot for `OrderLine`.
4. Cover the aggregate with xUnit tests: positive flows (create → submit → confirm → ship) and negative ones (confirm empty, mutate after confirmed, cancel shipped). Use `FluentAssertions` for readable asserts.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Решение собирается через `dotnet build` без ошибок и предупреждений.
- [ ] (RU) `dotnet run --project src/DddLab.App` печатает сценарий с ожидаемыми сообщениями.
- [ ] (RU) `Money`, `Address`, `OrderStatus`, `OrderLine`, `Order`, `IOrderRepository` созданы в проекте `DddLab.Domain`.
- [ ] (RU) У `Order` и `OrderLine` нет публичных сеттеров состояния; нет `UpdateStatus(int)`.
- [ ] (RU) `Lines` возвращает `ReadOnlyCollection<OrderLine>`; конструктор `OrderLine` `internal`.
- [ ] (RU) Все инварианты проверяются внутри агрегата и бросают `InvalidOperationException`.
- [ ] (RU) `IOrderRepository` работает с целыми агрегатами, без `IQueryable` и DTO.
- [ ] (RU) Файл `BoundedContexts.md` описывает разные `Product` в складском и каталоговом контекстах.
- [ ] (EN) Solution builds with `dotnet build` with no errors and no warnings.
- [ ] (EN) `dotnet run --project src/DddLab.App` prints the scenario with the expected messages.
- [ ] (EN) `Money`, `Address`, `OrderStatus`, `OrderLine`, `Order`, `IOrderRepository` exist in `DddLab.Domain`.
- [ ] (EN) No public state setters on `Order`/`OrderLine`; no `UpdateStatus(int)`.
- [ ] (EN) `Lines` returns `ReadOnlyCollection<OrderLine>`; `OrderLine` constructor is `internal`.
- [ ] (EN) All invariants are enforced inside the aggregate and throw `InvalidOperationException`.
- [ ] (EN) `IOrderRepository` works with whole aggregates, no `IQueryable` and no DTOs.
- [ ] (EN) `BoundedContexts.md` explains the different `Product` classes in warehouse and catalog contexts.

#### Ресурсы / Resources
- [Microsoft Learn — Domain-Driven Design (DDD) and CQRS patterns](https://learn.microsoft.com/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/)
- [Microsoft Learn — Microservices architecture — Domain model design](https://learn.microsoft.com/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/domain-model-design)
- Эрик Эванс, «Domain-Driven Design: Tackling Complexity in the Heart of Software» (2003) / Eric Evans, “Domain-Driven Design” (2003)
- Вон Вернон, «Implementing Domain-Driven Design» / Vaughn Vernon, “Implementing Domain-Driven Design”
- [Microsoft Learn — C# records](https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-9#record-types) и [pattern matching в C# 12](https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-12#pattern-matching)
