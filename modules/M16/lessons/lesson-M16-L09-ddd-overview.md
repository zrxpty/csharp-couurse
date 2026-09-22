[⬆ К модулю M16](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M16-L09: Domain-Driven Design (обзор, Optional) / Domain-Driven Design (overview, Optional)

**Модуль / Module:** M16
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Domain-Driven Design (DDD) — это подход к разработке, предложенный Эриком Эвансом в 2003 году, в центре которого находится **предметная область** бизнеса, а не базы данных или фреймворки. Главная идея: код должен говорить на том же языке, что и эксперты предметной области, а архитектура системы должна отражать реальную структуру бизнеса.

DDD принято делить на две части: **стратегическую (strategic)** и **тактическую (tactical)**. Стратегическая DDD отвечает на вопрос «как разбить большую систему на части», а тактическая — «как смоделировать каждую часть в коде».

**Стратегическое проектирование** начинается с **Ubiquitous Language (Единый язык)**. Это общий словарь терминов, который используют и разработчики, и эксперты бизнеса, и который напрямую отражается в коде: классы, методы, свойства называются так же, как звучат в разговоре. Если эксперт говорит «Заказ подтверждён», в коде должен быть метод `Order.Confirm()`, а не `UpdateStatus(2)`. Это устраняет постоянный «перевод» между языком бизнеса и языком кода.

Далее система разбивается на **Bounded Contexts (Ограниченные контексты)**. Bounded Context — это логическая граница, внутри которой модель и единый язык имеют однозначный смысл. Аналогия: слово «товар» в складском контексте означает физическую коробку с габаритами, в каталоге — позицию с описанием и фото, а в бухгалтерии — строку счёта. Попытка сделать один класс `Product` для всей системы порождает «франкенштейна» со свойствами из всех отделов. Bounded Context позволяет иметь три разных `Product`-класса, каждый со своим смыслом. Между контекстами определяются отношения (например, Customer/Supplier, Partnership, Conformist, Anti-corruption Layer), чтобы они интегрировались без смешивания моделей.

**Тактическое проектирование** — это строительные блоки для модели внутри контекста:

- **Entity (Сущность)** — объект с уникальной идентичностью (ID), которая сохраняется во времени, даже если меняются все свойства. Например, `Customer` с `Id` остаётся тем же клиентом после смены адреса.
- **Value Object (Объект-значение)** — объект без идентичности, описываемый только своими значениями; неизменяемый. `Money`, `Address`, `DateRange`. Два `Money(100, "USD")` равны, и нет смысла их различать.
- **Aggregate (Агрегат)** — группа связанных сущностей и value objects, которая рассматривается как единое целое с точки зрения консистентности. У каждого агрегата есть **Aggregate Root (Корень агрегата)** — единственная точка входа; внешние объекты держат ссылки только на корень, а не на внутренние сущности. Аналогия: заказ (`Order`) — корень, его строки (`OrderLine`) доступны только через `Order.AddLine(...)`. Все инварианты (бизнес-правила) агрегата проверяются при каждом изменении через корень.
- **Repository (Репозиторий)** — абстракция, предоставляющая доступ к агрегатам как к коллекциям в памяти (`IOrderRepository.GetById(id)`, `Add(order)`). Репозитории работают с целыми агрегатами и сохраняют их консистентность.

Ключевое правило агрегатов: **одна транзакция = один агрегат**. Если нужно согласовать изменения нескольких агрегатов, используют domain events и eventual consistency, а не одну большую транзакцию.

Главный практический вывод DDD: моделируйте поведение, а не структуру данных. Богатая доменная модель содержит бизнес-логику в самих объектах (`Order.Confirm()`, `OrderLine.ChangeQuantity()`), а не в «анемичных» DTO, которыми управляют сервисы-«транзакционные скрипты». DDD — не серебряная пуля: он окупается в системах со сложной бизнес-логикой и теряет смысл в простых CRUD-приложениях.

#### Theory (EN)

Domain-Driven Design (DDD) is an approach introduced by Eric Evans in 2003 that puts the **business domain** at the center of software design rather than databases or frameworks. The core idea: code should speak the same language as domain experts, and the architecture should mirror the real structure of the business.

DDD is traditionally split into two parts: **strategic** and **tactical**. Strategic DDD answers “how do we break a large system into parts?”, while tactical DDD answers “how do we model each part in code?”.

**Strategic design** starts with the **Ubiquitous Language** — a shared vocabulary used by developers, domain experts, and reflected directly in code. Classes, methods, and properties are named exactly as they sound in conversation. If an expert says “the order is confirmed”, the code should have `Order.Confirm()`, not `UpdateStatus(2)`. This removes the constant translation between business language and code language.

Next, the system is divided into **Bounded Contexts**. A Bounded Context is a logical boundary inside which a model and ubiquitous language have one unambiguous meaning. Analogy: the word “product” in a warehouse context means a physical box with dimensions, in a catalog it is a listing with photos, and in accounting it is an invoice line. Trying to make a single `Product` class for the whole system produces a “Frankenstein” with properties from every department. Bounded Contexts allow three different `Product` classes, each with its own meaning. Between contexts you define relationships (Customer/Supplier, Partnership, Conformist, Anti-corruption Layer) so they integrate without merging models.

**Tactical design** provides building blocks for the model inside a context:

- **Entity** — an object with a unique identity (ID) that persists over time even if all its attributes change. `Customer` with `Id` stays the same client after moving to a new address.
- **Value Object** — an object without identity, defined only by its values, and immutable. `Money`, `Address`, `DateRange`. Two `Money(100, "USD")` are equal and there is no point distinguishing them.
- **Aggregate** — a cluster of related entities and value objects treated as a single unit for consistency. Each aggregate has an **Aggregate Root** — the only entry point; external objects hold references only to the root, never to internal entities. Analogy: `Order` is the root, its lines (`OrderLine`) are accessed only through `Order.AddLine(...)`. All invariants (business rules) of the aggregate are enforced through the root on every change.
- **Repository** — an abstraction that presents aggregates as in-memory collections (`IOrderRepository.GetById(id)`, `Add(order)`). Repositories work with whole aggregates and preserve their consistency.

The key aggregate rule: **one transaction = one aggregate**. When changes across multiple aggregates must be coordinated, use domain events and eventual consistency instead of one giant transaction.

The main practical takeaway of DDD: model behavior, not data structure. A rich domain model keeps business logic inside the objects themselves (`Order.Confirm()`, `OrderLine.ChangeQuantity()`) rather than in anemic DTOs manipulated by “transaction-script” services. DDD is not a silver bullet: it pays off in systems with complex business logic and is overkill for simple CRUD applications.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — Тактические блоки DDD на примере агрегата Order
// Tactical DDD blocks shown on an Order aggregate

using System.Collections.ObjectModel;

// Value Object — неизменяемый, сравнение по значению
// Value Object — immutable, compared by value
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
}

// Entity внутри агрегата — не имеет публичных сеттеров, изменяется через поведение
// Entity inside the aggregate — no public setters, changed through behavior
public sealed class OrderLine
{
    public Guid ProductId { get; }
    public string ProductName { get; }
    public Money UnitPrice { get; }
    public int Quantity { get; private set; }

    internal OrderLine(Guid productId, string productName, Money unitPrice, int quantity)
    {
        ProductId = productId;
        ProductName = productName;
        UnitPrice = unitPrice;
        Quantity = quantity;
    }

    public Money SubTotal => new Money(UnitPrice.Amount * Quantity, UnitPrice.Currency);

    internal void ChangeQuantity(int newQuantity)
    {
        if (newQuantity <= 0)
            throw new InvalidOperationException(
                "Количество должно быть положительным / Quantity must be positive");
        Quantity = newQuantity;
    }
}

// Перечисление статусов на основе типа, а не магических чисел
// Type-based status, not magic numbers
public enum OrderStatus { Draft, Submitted, Confirmed, Shipped, Cancelled }

// Aggregate Root — единственная точка доступа к агрегату, защищает инварианты
// Aggregate Root — the only entry point to the aggregate, protects invariants
public sealed class Order
{
    public Guid Id { get; }
    public OrderStatus Status { get; private set; }
    public string Currency { get; }
    private readonly List<OrderLine> _lines = new();
    public ReadOnlyCollection<OrderLine> Lines => _lines.AsReadOnly();

    public Money Total => _lines.Aggregate(
        Money.Zero(Currency),
        (acc, line) => acc.Add(line.SubTotal));

    private Order(Guid id, string currency)
    {
        Id = id;
        Currency = currency;
        Status = OrderStatus.Draft;
    }

    // Фабрика — создаёт агрегат в корректном начальном состоянии
    // Factory — creates the aggregate in a valid initial state
    public static Order Create(string currency) => new(Guid.NewGuid(), currency);

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

    // Поведение вместо «UpdateStatus» — инкапсулирует бизнес-правило
    // Behavior instead of «UpdateStatus» — encapsulates a business rule
    public void Confirm()
    {
        if (Status != OrderStatus.Submitted)
            throw new InvalidOperationException(
                "Только подтверждённый заказ можно отправить / Only a submitted order can be confirmed");
        if (_lines.Count == 0)
            throw new InvalidOperationException(
                "Нельзя подтвердить пустой заказ / Cannot confirm an empty order");
        Status = OrderStatus.Confirmed;
    }

    public void Submit()
    {
        EnsureMutable();
        if (_lines.Count == 0)
            throw new InvalidOperationException(
                "Нельзя отправить пустой заказ / Cannot submit an empty order");
        Status = OrderStatus.Submitted;
    }

    private void EnsureMutable()
    {
        if (Status is OrderStatus.Confirmed or OrderStatus.Shipped or OrderStatus.Cancelled)
            throw new InvalidOperationException(
                "Заказ в этом статусе нельзя изменять / Order cannot be modified in this status");
    }
}

// Repository — доступ к целым агрегатам как к коллекции в памяти
// Repository — access to whole aggregates as an in-memory collection
public interface IOrderRepository
{
    Order? GetById(Guid id);        // Возвращает агрегат целиком / Returns the whole aggregate
    void Add(Order order);          // Сохраняет агрегат целиком / Saves the whole aggregate
    void Save(Order order);
}
```

#### Best Practices
- Моделируйте поведение, а не данные: бизнес-правила живут в агрегате, а не в сервисах-«транзакционных скриптах». / Model behavior, not data: business rules live in the aggregate, not in “transaction-script” services.
- Используйте Ubiquitous Language в коде дословно — имена классов и методов совпадают с терминами экспертов. / Use the Ubiquitous Language verbatim in code — class and method names match the experts’ terms.
- Делайте агрегаты маленькими: одна транзакция изменяет один агрегат, согласованность между агрегатами — через domain events. / Keep aggregates small: one transaction changes one aggregate; cross-aggregate consistency goes through domain events.
- Внешний код работает только с Aggregate Root; внутренние сущности не имеют публичных ссылок. / External code works only with the Aggregate Root; internal entities expose no public references.
- Value Object должен быть неизменяемым (record / readonly struct); идентичность только у Entity. / Value Objects must be immutable (record / readonly struct); identity belongs only to Entities.
- Repository работает с целыми агрегатами и принадлежит доменному слою (интерфейс), реализация — инфраструктуре. / A repository works with whole aggregates, lives in the domain layer (interface) and is implemented in infrastructure.
- Не применяйте DDD вслепую к простым CRUD-модулям — он окупается при сложной бизнес-логике. / Do not apply DDD blindly to simple CRUD modules — it pays off with complex business logic.

#### Частые ошибки / Common Mistakes
- **Анемичная модель**: данные в DTO, вся логика в сервисах → выносить поведение в агрегат и его сущности. / **Anemic model**: data in DTOs, all logic in services → push behavior into the aggregate and its entities.
- **Гигантский агрегат «на всю систему»**: один `Order` тянет клиента, склад, оплату → делить по Bounded Context и инвариантам. / **Giant “system-wide” aggregate**: one `Order` drags customer, warehouse, payment → split by Bounded Context and invariants.
- **Прямые ссылки на внутренние сущности** (`order.Lines[0].Quantity = 5`) → изменять только через методы корня. / **Direct references to internal entities** (`order.Lines[0].Quantity = 5`) → mutate only through root methods.
- **Один класс `Product` для всех отделов** → отдельная модель в каждом Bounded Context. / **One `Product` class for every department** → a separate model in each Bounded Context.
- **Транзакция через несколько агрегатов** → использовать domain events + eventual consistency. / **One transaction across several aggregates** → use domain events + eventual consistency.
- **Repository возвращает IQueryable или DTO** → возвращать целые агрегаты; DTO формируются в слое приложения. / **Repository returns IQueryable or DTOs** → return whole aggregates; build DTOs in the application layer.
- **«UpdateStatus(int)» и магические числа** → методы с семантикой (`Confirm()`, `Cancel()`) и enum. / **«UpdateStatus(int)» and magic numbers** → semantic methods (`Confirm()`, `Cancel()`) and enums.

#### Чек-лист самопроверки / Self-check Checklist
- [ ] Я определил Bounded Context и границы, где модель и язык однозначны. / I defined Bounded Contexts and boundaries where the model and language are unambiguous.
- [ ] Имена в коде совпадают с терминами экспертов (Ubiquitous Language). / Names in code match the experts’ terms (Ubiquitous Language).
- [ ] Агрегаты маленькие, с одним корнем и защищёнными инвариантами. / Aggregates are small, each with one root and protected invariants.
- [ ] Внешний код не обращается к внутренним сущностям напрямую. / External code never reaches internal entities directly.
- [ ] Value Objects неизменяемы, идентичность только у Entities. / Value Objects are immutable; identity belongs only to Entities.
- [ ] Repository работает с целыми агрегатами и не утекает IQueryable/DTO. / The repository works with whole aggregates and does not leak IQueryable/DTOs.
- [ ] Бизнес-правила инкапсулированы в методах агрегата, а не в сервисах. / Business rules are encapsulated in aggregate methods, not in services.
- [ ] Одна транзакция = один агрегат; межагрегатная согласованность — через events. / One transaction = one aggregate; cross-aggregate consistency goes through events.
- [ ] Я осознанно решил, где DDD оправдан, а где достаточно CRUD. / I consciously decided where DDD is justified and where CRUD is enough.

#### Ресурсы / Resources
- [Microsoft Learn — Domain-Driven Design (DDD) and CQRS patterns](https://learn.microsoft.com/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/)
- Эрик Эванс, «Domain-Driven Design: Tackling Complexity in the Heart of Software» (2003) / Eric Evans, “Domain-Driven Design” (2003)
- Вон Вернон, «Implementing Domain-Driven Design» / Vaughn Vernon, “Implementing Domain-Driven Design”

---

[⬆ К модулю M16](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
