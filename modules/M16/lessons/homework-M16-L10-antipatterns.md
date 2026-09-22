---
[← К уроку M16-L10](lesson-M16-L10-antipatterns.md) | [⬆ К модулю M16](../README.md) | [Следующее ДЗ →]()
---

### Домашнее задание M16-L10: Anti-patterns: god object, anemic domain, service-locator / Homework M16-L10: Anti-patterns: god object, anemic domain, service-locator

**Урок / Lesson:** M16-L10
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться диагностировать три классических антипаттерна (God Object, Anemic Domain Model, Service Locator) в реальном коде на C# 12 / .NET 8 и устранять их через декомпозицию по ответственности, перенос поведения в доменные сущности и явное внедрение зависимостей через конструктор; закрепить работу с `enum`, `private set`, top-level statements, collection expressions и raw string literals. (EN) Learn to diagnose three classic anti-patterns (God Object, Anemic Domain Model, Service Locator) in real C# 12 / .NET 8 code and remove them through responsibility-driven decomposition, moving behaviour into domain entities, and explicit constructor-based dependency injection; reinforce `enum`, `private set`, top-level statements, collection expressions, and raw string literals.

#### Связь с уроком / Connection to the lesson
(RU) Урок определяет God Object как класс, который «знает слишком много и делает слишком много», Anemic Domain Model как «контейнер с публичными свойствами и пустыми методами», а Service Locator как скрытое внедрение зависимостей через статический `Locator.GetService<T>()`. В этом задании вы встретите все три ловушки в одном учебном проекте «MiniShop» и устраните их точно так, как показано в примерах урока: выделите доменную сущность `Order` с инкапсулированным состоянием, обогатите `Account` поведением `Withdraw` с защитой инварианта и замените статический локатор на конструкторный DI. Дополнительно вы затронете Shotgun Surgery и Magic Strings, упомянутые в теории.
(EN) The lesson defines God Object as a class that "knows too much and does too much", Anemic Domain Model as "a container with public properties and no behaviour", and Service Locator as hidden dependency injection through a static `Locator.GetService<T>()`. In this homework you will meet all three traps inside a single "MiniShop" exercise project and remove them exactly as shown in the lesson examples: extract a domain `Order` with encapsulated state, enrich `Account` with a guarded `Withdraw` behaviour, and replace the static locator with constructor DI. You will also touch Shotgun Surgery and Magic Strings mentioned in the theory.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Стартап «MiniShop» начинался с одного класса `OrderProcessor`, который «удобно» держал в себе всю логику интернет-магазина: считал итоги заказа, сохранял данные в «базу», отправлял письма, рисовал PDF-отчёты и даже управлял балансом клиента. За полгода класс разросся до 1200 строк и сорока методов, а любое изменение статуса заказа тянуло за собой правки в десятке файлов — классический Shotgun Surgery. Параллельно доменные сущности `Account` и `Customer` превратились в мешки с публичными сеттерами: ни один инвариант не защищён, баланс можно уйти в минус простым присваиванием. Чтобы «не тянуть зависимости через конструктор», разработчики завели статический `ServiceLocator.Root` и стали вызывать `GetRequiredService<IEmailGateway>()` прямо из обработчиков. Теперь по сигнатуре класса невозможно понять, что ему нужно, а юнит-тесты требуют инициализации глобального контейнера.

Руководство ставит новую задачу: добавить логику отмены заказа с возвратом средств на счёт клиента и применить скидку постоянства (loyalty discount) для заказов от 10 000 ₽. В текущем коде это потребует правок везде и почти гарантированно сломает согласованность данных. Ваша задача — провести целевой рефакторинг: разрезать God Object по границам ответственности, вернуть поведение в домен, заменить Service Locator явным DI и убрать magic strings. Делать это нужно инкрементально, под прикрытием тестов, чтобы не скатиться в Big Bang Rewrite, о котором предупреждает урок.

#### Что нужно сделать (пошагово)
1. Создайте решение и четыре проекта. Выполните `dotnet new sln -n MiniShop` в пустой папке, затем `dotnet new classlib -n MiniShop.Domain -f net8.0`, `dotnet new classlib -n MiniShop.Infrastructure -f net8.0`, `dotnet new worker -n MiniShop.App -f net8.0` и `dotnet new xunit -n MiniShop.Tests -f net8.0`. Добавьте все проекты в решение: `dotnet sln add **/*.csproj`. Свяжите ссылки: `MiniShop.Infrastructure` → `MiniShop.Domain`, `MiniShop.App` → `MiniShop.Infrastructure` + `MiniShop.Domain`, `MiniShop.Tests` → все три.
2. В `MiniShop.Domain` создайте стартовый «плохой» код (скопируйте его из раздела «Эталонное решение» ниже в вариант `// BAD`): класс `OrderProcessor` (God Object), классы `Account` и `Customer` с публичными сеттерами (Anemic), статический `ServiceLocator` и обработчик `CancelOrderHandler`, который берёт зависимости из локатора. Убедитесь, что проект компилируется: `dotnet build`.
3. Проведите инвентаризацию антипаттернов. В файле `REFACTOR-NOTES.md` перечислите: (а) какие ответственности смешаны в `OrderProcessor`; (б) какие инварианты нарушаются у `Account`/`Customer`; (в) в каких точках используется `ServiceLocator` и какие зависимости скрыты; (г) какие magic strings/numbers присутствуют (например, `"paid"`, `"admin"`, `10000`).
4. Разрежьте God Object. Выделите доменную сущность `Order` с инкапсулированным состоянием (`Id`, `Total`, `Status` через `enum OrderStatus` с `[Description]`), методом `Confirm()` и новым методом `Cancel()` с доменным правилом: отменять можно только `Confirmed` заказы. Вынесите инфраструктуру в интерфейсы `IOrderRepository`, `IEmailGateway`, `IAccountRepository` и сервис `OrderService`, который только оркеструет.
5. Обогатите анемичную модель. В `Account` сделайте `Balance` с `private set`, добавьте `Withdraw(decimal amount)` с защитой от перерасхода и `Deposit(decimal amount)` с проверкой положительности. Метод `Cancel` у `Order` должен возвращать `decimal RefundAmount`, который сервис зачислит обратно через `Account.Deposit`.
6. Удалите Service Locator. Замените `ServiceLocator.Root.GetRequiredService<T>()` на конструкторный DI. В `Program.cs` (top-level statements) зарегистрируйте зависимости: `builder.Services.AddScoped<IOrderRepository, InMemoryOrderRepository>();` и т. п. Убедитесь, что ни одного статического локатора не осталось: `grep -r "ServiceLocator"` должен вернуть пусто.
7. Уберите magic strings. Введите `enum OrderStatus { New, Confirmed, Cancelled }` и `enum Role { Admin, Customer }` с `[Description]`. Литерал скидки `10000m` вынесите в `const decimal LoyaltyThreshold = 10_000m;`. Скидку считайте методом `Order.ApplyLoyaltyDiscount()` внутри сущности.
8. Напишите тесты в `MiniShop.Tests`: `Order_Confirm_FromNew_ChangesStatus`, `Order_Cancel_FromConfirmed_RefundsAndSetsCancelled`, `Account_Withdraw_MoreThanBalance_Throws`, `Order_ApplyLoyaltyDiscount_AboveThreshold_Applies10Percent`, `Order_Cancel_FromNew_Throws`. Используйте fake-реализации `IOrderRepository`/`IEmailGateway` (простые заглушки, не мок-фреймворк).
9. Запустите `dotnet test` — все тесты должны быть зелёными. Запустите `dotnet build -warnaserror` — без предупреждений. Проверьте, что ни один класс домена не ссылается на инфраструктуру (зависимости направлены внутрь).

#### Требования к решению
- Целевая платформа — .NET 8, язык C# 12. Используйте file-scoped namespaces, top-level statements в `Program.cs`, collection expressions для инициализации списков (например, `Order[] seed = [new(1, 5000m), new(2, 15_000m)];`), raw string literals для шаблонов писем и switch expressions для маппинга статусов.
- Архитектура должна соблюдать правило зависимостей: `MiniShop.Domain` не ссылается ни на `Infrastructure`, ни на `App`. Все доменные сущности — `sealed`, с `private set` на изменяемом состоянии; мутации только через методы с проверкой инвариантов.
- Все зависимости — исключительно через конструктор; статических локаторов и хранимого `IServiceProvider` в полях быть не должно. Сервисы (`OrderService`) не владеют бизнес-правилами — они лишь вызывают доменные методы и координируют репозитории и шлюзы.
- Magic strings/numbers запрещены: статусы, роли и пороги заданы через `enum` и `const`. Каждый `enum` снабжён `[Description]` для человекочитаемого вывода.
- Тесты покрывают доменные правила (отмена, возврат, перерасход, скидка) и не ходят в реальную базу или SMTP. Рефакторинг выполнен инкрементально: каждый шаг сопровождается тестом, Big Bang Rewrite недопустим.

#### Тонкости и подводные камни
- Не путайте «богатую модель» с возвращением God Object в домен. `Order` должен содержать только правила, относящиеся к самому заказу (`Confirm`, `Cancel`, `ApplyLoyaltyDiscount`); отправку писем и сохранение в базу оставьте в сервисах и шлюзах. Если вы начнёте добавлять в `Order` метод `SendEmail`, вы снова получите God Object — только доменный.
- `private set` защищает инвариант только тогда, когда метод-мутатор проверяет пред- и постусловия атомарно. Частая ошибка — оставить `public decimal Balance { get; set; }`, «забыв», что любой код может записать минус. Урок явно предостерегает от публичных сеттеров на доменных сущностях.
- Service Locator коварен тем, что компилируется и даже работает. Вред проявляется в тестах: чтобы проверить `CancelOrderHandler`, приходится инициализировать глобальный контейнер, что превращает юнит-тест в интеграционный. Заменяя локатор на конструкторный DI, не впадайте в обратную крайность — не передавайте `IServiceProvider` в конструктор: это тот же локатор, только в виде параметра.
- Magic strings выживают рефакторинг, потому что инструменты поиска не отличают `"paid"`-статус от `"paid"`-строки в тексте письма. Используйте `enum` и обращайтесь к статусу только через него; для вывода пользователю — расширение, читающее `[Description]`. Порог скидки задавайте как `const` с читаемым именем и числовым разделителем `10_000m`.
- При отмене заказа следите за порядком операций и атомарностью: сначала доменный `order.Cancel()` (бросает исключение, если статус не `Confirmed`), затем `account.Deposit(refund)`, затем `repo.SaveAsync`. Если сохранить в базу до `Deposit`, можно зафиксировать отмену без возврата. Рассмотрите `CancellationToken` на всех асинхронных вызовах — урок передаёт `ct` в каждый метод репозитория и шлюза.
- Shotgun Surgery лечится группировкой: если добавление поля «отчество» трогает 10 файлов, значит, границы нарезаны неверно. В этом задании новый статус `Cancelled` должен изменять ровно `enum OrderStatus`, метод `Order.Cancel` и, опционально, тесты — но не контроллер, DTO и маппер одновременно.

#### Критерии приёмки
- [ ] Решение `MiniShop.sln` собирается командой `dotnet build` без ошибок и предупреждений на .NET 8 / C# 12.
- [ ] Проект `MiniShop.Domain` не имеет ссылок на `Infrastructure` и `App` (зависимости направлены внутрь).
- [ ] Ни один класс домена не содержит инфраструктурных методов (`SaveToDatabase`, `SendEmail`, `RenderPdfReport`).
- [ ] `Order` — `sealed`, статус хранится в `enum OrderStatus`, изменяется только через методы `Confirm`/`Cancel`.
- [ ] `Order.Cancel()` бросает `InvalidOperationException` для статусов, отличных от `Confirmed`, и возвращает сумму возврата.
- [ ] `Account.Balance` имеет `private set`; `Withdraw` и `Deposit` проверяют предусловия.
- [ ] `Account.Withdraw` бросает исключение при попытке снять больше баланса.
- [ ] В коде нет статического `ServiceLocator` и хранимого `IServiceProvider` в полях классов.
- [ ] Все зависимости `OrderService` и обработчиков переданы через конструктор.
- [ ] В `Program.cs` используется top-level statements и регистрация DI через `builder.Services`.
- [ ] Magic strings/numbers отсутствуют: статусы и роли — `enum`, пороги — `const`.
- [ ] Применены collection expressions и raw string literals минимум в одном месте каждый.
- [ ] Тесты `dotnet test` зелёные; покрыты отмена, возврат, перерасход, скидка, попытка отмены из неверного статуса.
- [ ] Файл `REFACTOR-NOTES.md` содержит инвентаризацию всех четырёх групп антипаттернов.
- [ ] Рефакторинг выполнен инкрементально (виден по истории коммитов или пошаговым заметкам), без Big Bang Rewrite.

#### Подсказки (без прямого ответа)
- Спросите себя: «Какая единственная причина изменения есть у этого класса?» Если ответов больше одного — это God Object.
- Для `Order.Cancel` подумайте, какой статус допустим перед отменой и что вернуть вызывающему коду, чтобы он мог вернуть деньги.
- Чтобы убрать Service Locator, идите от «границы» приложения: DI-контейнер собирает граф объектов в `Program.cs`, а обработчики просто получают готовые зависимости.
- Для чтения `[Description]` напишите метод расширения `GetDescription(this Enum value)` через `typeof(T).GetField(name).GetCustomAttribute<DescriptionAttribute>()`.
- Тесты на домен не требуют контейнера: создавайте `new Order(...)` напрямую и вызывайте методы — если для теста нужен контейнер, это сигнал скрытой зависимости.

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 — Эталонное решение MiniShop (фрагменты)
// Reference solution for MiniShop (fragments)

using System.ComponentModel;
using System.Reflection;

namespace MiniShop.Domain;

// ===== enum вместо magic strings / enum instead of magic strings =====
public enum OrderStatus
{
    [Description("Новый / New")]         New,
    [Description("Подтверждён / Confirmed")] Confirmed,
    [Description("Отменён / Cancelled")] Cancelled
}

public enum Role
{
    [Description("Администратор / Admin")] Admin,
    [Description("Клиент / Customer")]     Customer
}

// ===== Богатая доменная сущность / Rich domain entity =====
public sealed class Order
{
    private const decimal LoyaltyThreshold = 10_000m;       // нет magic number
    private const decimal LoyaltyDiscount   = 0.10m;        // 10%

    public int Id { get; }
    public decimal Total { get; private set; }
    public OrderStatus Status { get; private set; }

    public Order(int id, decimal total)
    {
        if (total < 0) throw new ArgumentOutOfRangeException(nameof(total));
        Id = id; Total = total; Status = OrderStatus.New;
    }

    public void Confirm()
    {
        if (Status != OrderStatus.New)
            throw new InvalidOperationException("Order cannot be confirmed"); // guard
        Status = OrderStatus.Confirmed;
    }

    // Возвращает сумму возврата / Returns refund amount
    public decimal Cancel()
    {
        if (Status != OrderStatus.Confirmed)
            throw new InvalidOperationException("Only confirmed orders can be cancelled");
        Status = OrderStatus.Cancelled;
        return Total; // полный возврат / full refund
    }

    public void ApplyLoyaltyDiscount()
    {
        if (Total >= LoyaltyThreshold)
            Total *= (1m - LoyaltyDiscount); // правило внутри сущности
    }

    public string DisplayStatus => Status.GetDescription(); // raw string не нужен тут
}

// ===== Account с защищённым инвариантом / Account with protected invariant =====
public sealed class Account
{
    public decimal Balance { get; private set; }            // private set — ключевой момент
    public Account(decimal initial) => Balance = initial;

    public void Deposit(decimal amount)
    {
        if (amount <= 0) throw new ArgumentOutOfRangeException(nameof(amount));
        Balance += amount;
    }

    public void Withdraw(decimal amount)
    {
        if (amount <= 0) throw new ArgumentOutOfRangeException(nameof(amount));
        if (amount > Balance) throw new InvalidOperationException("Insufficient funds");
        Balance -= amount; // инвариант проверен атомарно / invariant checked atomically
    }
}

// ===== Инфраструктурные порты вне домена / Ports outside the domain =====
public interface IOrderRepository   { Task SaveAsync(Order order, CancellationToken ct); }
public interface IAccountRepository { Task<Account?> GetAsync(int customerId, CancellationToken ct); }
public interface IEmailGateway      { Task SendAsync(string to, string body, CancellationToken ct); }

// ===== Сервис-оркестратор: НЕ владеет правилами / Orchestrator: no business rules =====
public sealed class OrderService
{
    private readonly IOrderRepository _orders;
    private readonly IAccountRepository _accounts;
    private readonly IEmailGateway _email;

    public OrderService(IOrderRepository orders, IAccountRepository accounts, IEmailGateway email)
        => (_orders, _accounts, _email) = (orders, accounts, email);

    public async Task CancelAsync(Order order, int customerId, string email, CancellationToken ct)
    {
        var refund = order.Cancel();                         // доменное правило в сущности
        var account = await _accounts.GetAsync(customerId, ct)
            ?? throw new InvalidOperationException("Account not found");
        account.Deposit(refund);                             // возврат на счёт
        await _orders.SaveAsync(order, ct);
        await _email.SendAsync(email,
            $$"""                                          // raw string literal
              Заказ #{{order.Id}} отменён.
              Возврат: {{refund}} ₽ зачислен на счёт.
              Order #{{order.Id}} cancelled. Refund: {{refund}}.
              """, ct);
    }
}

// ===== Расширение для [Description] / Extension for [Description] =====
public static class EnumExtensions
{
    public static string GetDescription(this Enum value) =>
        value.GetType().GetField(value.ToString())?
            .GetCustomAttribute<DescriptionAttribute>()?.Description
            ?? value.ToString();
}
```

Разбор по строкам. `enum OrderStatus` с `[Description]` заменяет строковые литералы `"new"`, `"paid"` — это прямое исправление Magic Strings из урока. Класс `Order` помечен `sealed` и хранит `Status`/`Total` с `private set`: мутация возможна только через методы, что защищает инвариант «отменить можно только подтверждённый заказ». Конструктор проверяет `total < 0` — это предусловие, без которого инвариант согласованности легко нарушить. Метод `Confirm()` содержит guard `Status != OrderStatus.New`, точно как в примере урока; `Cancel()` вводит новое правило и возвращает сумму возврата, чтобы сервис мог зачислить её на `Account` — поведение остаётся в домене, а оркестрация (сохранение, письмо) — в `OrderService`. `ApplyLoyaltyDiscount()` держит порог и процент в `const`, устраняя magic number `10000`. `Account` — зеркальное исправление анемичной модели: `Balance` с `private set`, `Withdraw` с проверкой `amount > Balance`, как в уроке. Интерфейсы `IOrderRepository`/`IEmailGateway`/`IAccountRepository` выносят инфраструктуру из домена, а `OrderService` получает их через конструктор — это замена Service Locator на явный DI, рекомендованная уроком. Raw string literal `"""..."""` с интерполяцией `{{order.Id}}` демонстрирует C# 12 для шаблона письма. Расширение `GetDescription` читает атрибут через рефлексию, чтобы не дублировать строки. Важно: ни одна доменная сущность не ссылается на инфраструктуру, статических локаторов нет, бизнес-правила живут в сущностях — выполнены все пять best practices из урока.

#### Задания на углубление (бонус)
1. Добавьте архитектурный тест с помощью NetArchTest или ручной проверки через рефлексию: убедитесь, что `MiniShop.Domain` не ссылается на `Microsoft.Extensions.DependencyInjection` и на `MiniShop.Infrastructure`.
2. Реализуйте `Order.Refund()` как value object `Money` с валютой и инкапсулированной арифметикой; покажите, как это уменьшает Shotgun Surgery при добавлении второй валюты.
3. Покройте `OrderService.CancelAsync` интеграционным тестом с in-memory репозиториями и проверьте порядок вызовов через логирующую заглушку `IEmailGateway`.
4. Добавьте статус `Refunded` и правило частичного возврата; проследите, сколько файлов изменилось, и сравните с тем, сколько пришлось бы тронуть в «плохой» версии — продемонстрируйте лечение Shotgun Surgery.

---

## Statement in English / Постановка на английском

#### Context & motivation
The "MiniShop" startup began as a single `OrderProcessor` class that "conveniently" held the entire logic of an online store: it computed order totals, persisted data to a "database", sent emails, rendered PDF reports, and even managed customer balances. Over six months the class grew to 1200 lines and forty methods, and any change to order status dragged edits across a dozen files — classic Shotgun Surgery. In parallel, the domain entities `Account` and `Customer` degenerated into bags of public setters: no invariant is protected, the balance can go negative with a single assignment. To "avoid passing dependencies through constructors", the developers introduced a static `ServiceLocator.Root` and started calling `GetRequiredService<IEmailGateway>()` directly from handlers. Now the class signature no longer reveals what it needs, and unit tests require a fully initialised global container.

Management sets a new task: add order cancellation logic that refunds the customer's account, and apply a loyalty discount for orders of 10 000 ₽ and above. In the current code this would require edits everywhere and almost certainly break data consistency. Your job is to perform a targeted refactor: slice the God Object along responsibility boundaries, return behaviour to the domain, replace the Service Locator with explicit DI, and eliminate magic strings. Do it incrementally, under test coverage, to avoid sliding into the Big Bang Rewrite the lesson warns about.

#### What to do step by step
1. Create the solution and four projects. Run `dotnet new sln -n MiniShop` in an empty folder, then `dotnet new classlib -n MiniShop.Domain -f net8.0`, `dotnet new classlib -n MiniShop.Infrastructure -f net8.0`, `dotnet new worker -n MiniShop.App -f net8.0`, and `dotnet new xunit -n MiniShop.Tests -f net8.0`. Add them all to the solution: `dotnet sln add **/*.csproj`. Wire references: `MiniShop.Infrastructure` → `MiniShop.Domain`, `MiniShop.App` → `MiniShop.Infrastructure` + `MiniShop.Domain`, `MiniShop.Tests` → all three.
2. In `MiniShop.Domain` create the initial "bad" code (copy the `// BAD` variant from the reference solution below): a `OrderProcessor` God Object, `Account` and `Customer` classes with public setters (anemic), a static `ServiceLocator`, and a `CancelOrderHandler` that pulls dependencies from the locator. Make sure it compiles: `dotnet build`.
3. Inventory the anti-patterns. In `REFACTOR-NOTES.md` list: (a) which responsibilities are mixed inside `OrderProcessor`; (b) which invariants are broken in `Account`/`Customer`; (c) where `ServiceLocator` is used and which dependencies it hides; (d) which magic strings/numbers are present (e.g. `"paid"`, `"admin"`, `10000`).
4. Slice the God Object. Extract a domain entity `Order` with encapsulated state (`Id`, `Total`, `Status` via `enum OrderStatus` with `[Description]`), a `Confirm()` method, and a new `Cancel()` method that enforces the domain rule: only `Confirmed` orders can be cancelled. Move infrastructure into interfaces `IOrderRepository`, `IEmailGateway`, `IAccountRepository` and an `OrderService` that only orchestrates.
5. Enrich the anemic model. In `Account` make `Balance` a `private set`, add `Withdraw(decimal amount)` with an overdraft guard, and `Deposit(decimal amount)` with a positivity check. `Order.Cancel` should return a `decimal RefundAmount` that the service credits back through `Account.Deposit`.
6. Remove the Service Locator. Replace `ServiceLocator.Root.GetRequiredService<T>()` with constructor DI. In `Program.cs` (top-level statements) register dependencies: `builder.Services.AddScoped<IOrderRepository, InMemoryOrderRepository>();` and so on. Verify no static locator remains: `grep -r "ServiceLocator"` should return nothing.
7. Eliminate magic strings. Introduce `enum OrderStatus { New, Confirmed, Cancelled }` and `enum Role { Admin, Customer }` with `[Description]`. Move the discount threshold `10000m` to `const decimal LoyaltyThreshold = 10_000m;`. Compute the discount inside the entity via `Order.ApplyLoyaltyDiscount()`.
8. Write tests in `MiniShop.Tests`: `Order_Confirm_FromNew_ChangesStatus`, `Order_Cancel_FromConfirmed_RefundsAndSetsCancelled`, `Account_Withdraw_MoreThanBalance_Throws`, `Order_ApplyLoyaltyDiscount_AboveThreshold_Applies10Percent`, `Order_Cancel_FromNew_Throws`. Use simple fake implementations of `IOrderRepository`/`IEmailGateway` (plain stubs, no mocking framework).
9. Run `dotnet test` — all tests must be green. Run `dotnet build -warnaserror` — no warnings. Confirm that no domain class references infrastructure (dependencies point inward).

#### Requirements
- Target .NET 8, language C# 12. Use file-scoped namespaces, top-level statements in `Program.cs`, collection expressions for list seeding (e.g. `Order[] seed = [new(1, 5000m), new(2, 15_000m)];`), raw string literals for email templates, and switch expressions for status mapping.
- The architecture must respect the dependency rule: `MiniShop.Domain` references neither `Infrastructure` nor `App`. All domain entities are `sealed`, with `private set` on mutable state; mutations happen only through methods that check invariants.
- All dependencies come through constructors exclusively; no static locators and no stored `IServiceProvider` in fields. Services (`OrderService`) do not own business rules — they only call domain methods and coordinate repositories and gateways.
- Magic strings/numbers are forbidden: statuses, roles, and thresholds are defined through `enum` and `const`. Every `enum` carries a `[Description]` for human-readable output.
- Tests cover the domain rules (cancellation, refund, overdraft, discount) and do not touch a real database or SMTP. The refactor is incremental: each step is covered by a test; a Big Bang Rewrite is not acceptable.

#### Pitfalls
- Do not confuse a "rich model" with bringing the God Object back into the domain. `Order` should hold only the rules that belong to the order itself (`Confirm`, `Cancel`, `ApplyLoyaltyDiscount`); email sending and persistence belong in services and gateways. If you start adding `SendEmail` to `Order`, you get a God Object again — only a domain one.
- `private set` protects an invariant only when the mutator method checks pre- and post-conditions atomically. A common mistake is leaving `public decimal Balance { get; set; }`, "forgetting" that any code can write a negative value. The lesson explicitly warns against public setters on domain entities.
- Service Locator is insidious because it compiles and even runs. The harm shows up in tests: to verify `CancelOrderHandler` you must initialise the global container, which turns a unit test into an integration test. When replacing the locator with constructor DI, do not swing to the opposite extreme — do not pass `IServiceProvider` into a constructor: that is the same locator, only as a parameter.
- Magic strings survive refactoring because search tools cannot tell a `"paid"` status from a `"paid"` substring in an email body. Use `enum` and refer to statuses only through it; for user-facing output, use an extension that reads `[Description]`. Set the discount threshold as a `const` with a readable name and a digit separator `10_000m`.
- When cancelling an order, mind operation ordering and atomicity: first the domain `order.Cancel()` (throws if the status is not `Confirmed`), then `account.Deposit(refund)`, then `repo.SaveAsync`. If you persist before `Deposit`, you may record a cancellation without a refund. Pass a `CancellationToken` to every async call — the lesson threads `ct` through every repository and gateway method.
- Shotgun Surgery is cured by grouping: if adding a "middle name" field touches 10 files, the slicing is wrong. In this task the new `Cancelled` status should change only `enum OrderStatus`, the `Order.Cancel` method, and optionally tests — not the controller, DTO, and mapper all at once.

#### Acceptance criteria
- [ ] The `MiniShop.sln` solution builds with `dotnet build` without errors or warnings on .NET 8 / C# 12.
- [ ] The `MiniShop.Domain` project has no references to `Infrastructure` or `App` (dependencies point inward).
- [ ] No domain class contains infrastructure methods (`SaveToDatabase`, `SendEmail`, `RenderPdfReport`).
- [ ] `Order` is `sealed`, its status is stored in `enum OrderStatus`, and changes only via `Confirm`/`Cancel`.
- [ ] `Order.Cancel()` throws `InvalidOperationException` for any status other than `Confirmed` and returns the refund amount.
- [ ] `Account.Balance` has a `private set`; `Withdraw` and `Deposit` check preconditions.
- [ ] `Account.Withdraw` throws when you try to withdraw more than the balance.
- [ ] No static `ServiceLocator` and no stored `IServiceProvider` in class fields remain in the code.
- [ ] All dependencies of `OrderService` and handlers are passed through the constructor.
- [ ] `Program.cs` uses top-level statements and DI registration through `builder.Services`.
- [ ] No magic strings/numbers remain: statuses and roles are `enum`, thresholds are `const`.
- [ ] Collection expressions and raw string literals are each used in at least one place.
- [ ] `dotnet test` is green; cancellation, refund, overdraft, discount, and invalid-status cancellation are covered.
- [ ] `REFACTOR-NOTES.md` contains an inventory of all four anti-pattern groups.
- [ ] The refactor is incremental (visible from commit history or step-by-step notes), with no Big Bang Rewrite.

#### Hints (without giving the answer away)
- Ask yourself: "What single reason to change does this class have?" If there is more than one, it is a God Object.
- For `Order.Cancel`, think about which status is legal before cancellation and what you should return to the caller so it can refund the money.
- To remove the Service Locator, work from the application boundary: the DI container builds the object graph in `Program.cs`, and handlers simply receive ready-made dependencies.
- To read `[Description]`, write an extension method `GetDescription(this Enum value)` using `typeof(T).GetField(name).GetCustomAttribute<DescriptionAttribute>()`.
- Domain tests should not need a container: create `new Order(...)` directly and call methods — if a test needs a container, that is a sign of a hidden dependency.

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8 — Reference solution for MiniShop (fragments)

using System.ComponentModel;
using System.Reflection;

namespace MiniShop.Domain;

// ===== enum instead of magic strings =====
public enum OrderStatus
{
    [Description("New / Новый")]         New,
    [Description("Confirmed / Подтверждён")] Confirmed,
    [Description("Cancelled / Отменён")] Cancelled
}

public enum Role
{
    [Description("Admin / Администратор")] Admin,
    [Description("Customer / Клиент")]     Customer
}

// ===== Rich domain entity =====
public sealed class Order
{
    private const decimal LoyaltyThreshold = 10_000m;     // no magic number
    private const decimal LoyaltyDiscount   = 0.10m;      // 10%

    public int Id { get; }
    public decimal Total { get; private set; }
    public OrderStatus Status { get; private set; }

    public Order(int id, decimal total)
    {
        if (total < 0) throw new ArgumentOutOfRangeException(nameof(total));
        Id = id; Total = total; Status = OrderStatus.New;
    }

    public void Confirm()
    {
        if (Status != OrderStatus.New)
            throw new InvalidOperationException("Order cannot be confirmed"); // guard
        Status = OrderStatus.Confirmed;
    }

    // Returns the refund amount
    public decimal Cancel()
    {
        if (Status != OrderStatus.Confirmed)
            throw new InvalidOperationException("Only confirmed orders can be cancelled");
        Status = OrderStatus.Cancelled;
        return Total; // full refund
    }

    public void ApplyLoyaltyDiscount()
    {
        if (Total >= LoyaltyThreshold)
            Total *= (1m - LoyaltyDiscount); // rule inside the entity
    }

    public string DisplayStatus => Status.GetDescription();
}

// ===== Account with a protected invariant =====
public sealed class Account
{
    public decimal Balance { get; private set; }          // private set is the key point
    public Account(decimal initial) => Balance = initial;

    public void Deposit(decimal amount)
    {
        if (amount <= 0) throw new ArgumentOutOfRangeException(nameof(amount));
        Balance += amount;
    }

    public void Withdraw(decimal amount)
    {
        if (amount <= 0) throw new ArgumentOutOfRangeException(nameof(amount));
        if (amount > Balance) throw new InvalidOperationException("Insufficient funds");
        Balance -= amount; // invariant checked atomically
    }
}

// ===== Infrastructure ports outside the domain =====
public interface IOrderRepository   { Task SaveAsync(Order order, CancellationToken ct); }
public interface IAccountRepository { Task<Account?> GetAsync(int customerId, CancellationToken ct); }
public interface IEmailGateway      { Task SendAsync(string to, string body, CancellationToken ct); }

// ===== Orchestrator service: does NOT own rules =====
public sealed class OrderService
{
    private readonly IOrderRepository _orders;
    private readonly IAccountRepository _accounts;
    private readonly IEmailGateway _email;

    public OrderService(IOrderRepository orders, IAccountRepository accounts, IEmailGateway email)
        => (_orders, _accounts, _email) = (orders, accounts, email);

    public async Task CancelAsync(Order order, int customerId, string email, CancellationToken ct)
    {
        var refund = order.Cancel();                       // domain rule in the entity
        var account = await _accounts.GetAsync(customerId, ct)
            ?? throw new InvalidOperationException("Account not found");
        account.Deposit(refund);                           // refund to the account
        await _orders.SaveAsync(order, ct);
        await _email.SendAsync(email,
            $$"""                                          // raw string literal
              Order #{{order.Id}} cancelled.
              Refund: {{refund}} credited to the account.
              Заказ #{{order.Id}} отменён. Возврат: {{refund}}.
              """, ct);
    }
}

// ===== Extension for [Description] =====
public static class EnumExtensions
{
    public static string GetDescription(this Enum value) =>
        value.GetType().GetField(value.ToString())?
            .GetCustomAttribute<DescriptionAttribute>()?.Description
            ?? value.ToString();
}
```

Walk-through. `enum OrderStatus` with `[Description]` replaces the string literals `"new"`, `"paid"` — a direct fix of the Magic Strings anti-pattern from the lesson. The `Order` class is `sealed` and keeps `Status`/`Total` behind `private set`: mutation is possible only through methods, which protects the invariant "only a confirmed order can be cancelled". The constructor checks `total < 0` — a precondition without which consistency is easily broken. `Confirm()` carries the guard `Status != OrderStatus.New`, exactly as in the lesson example; `Cancel()` introduces a new rule and returns the refund amount so the service can credit it to the `Account` — behaviour stays in the domain while orchestration (persistence, email) stays in `OrderService`. `ApplyLoyaltyDiscount()` keeps the threshold and percentage in `const`, eliminating the magic number `10000`. `Account` is a mirror fix of the anemic model: `Balance` has a `private set`, `Withdraw` checks `amount > Balance`, as in the lesson. The interfaces `IOrderRepository`/`IEmailGateway`/`IAccountRepository` move infrastructure out of the domain, and `OrderService` receives them through the constructor — this is the replacement of Service Locator with explicit DI recommended by the lesson. The raw string literal `"""..."""` with `{{order.Id}}` interpolation showcases C# 12 for the email template. The `GetDescription` extension reads the attribute via reflection so the display strings are not duplicated. Crucially, no domain entity references infrastructure, no static locators remain, and business rules live inside entities — all five best practices from the lesson are satisfied.

#### Going deeper (bonus)
1. Add an architecture test with NetArchTest or a manual reflection-based check confirming that `MiniShop.Domain` references neither `Microsoft.Extensions.DependencyInjection` nor `MiniShop.Infrastructure`.
2. Implement `Order.Refund()` as a `Money` value object with a currency and encapsulated arithmetic; show how this reduces Shotgun Surgery when a second currency is introduced.
3. Cover `OrderService.CancelAsync` with an integration test using in-memory repositories and verify the call order through a logging stub of `IEmailGateway`.
4. Add a `Refunded` status and a partial-refund rule; track how many files changed and compare it with how many the "bad" version would have required — demonstrate the cure for Shotgun Surgery.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Решение собирается `dotnet build` без предупреждений на .NET 8 / C# 12.
- [ ] (RU) Домен не ссылается на инфраструктуру; сущности `sealed` с `private set`.
- [ ] (RU) Удалён `ServiceLocator`; все зависимости — через конструктор.
- [ ] (RU) Magic strings/numbers заменены на `enum` и `const`.
- [ ] (RU) Тесты зелёные; `REFACTOR-NOTES.md` приложен.
- [ ] (EN) Solution builds with `dotnet build`, no warnings, on .NET 8 / C# 12.
- [ ] (EN) Domain has no infrastructure references; entities are `sealed` with `private set`.
- [ ] (EN) `ServiceLocator` removed; all dependencies are constructor-injected.
- [ ] (EN) Magic strings/numbers replaced with `enum` and `const`.
- [ ] (EN) Tests are green; `REFACTOR-NOTES.md` is attached.

#### Ресурсы / Resources
- Microsoft Learn — DDD/microservice patterns: https://learn.microsoft.com/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/
- Microsoft Learn — Dependency injection in .NET: https://learn.microsoft.com/dotnet/core/extensions/dependency-injection
- Martin Fowler — AnemicDomainModel: https://martinfowler.com/bliki/AnemicDomainModel.html
- Microsoft Learn — `enum` and `[Description]`: https://learn.microsoft.com/dotnet/api/system.componentmodel.descriptionattribute
- C# 12 — Raw string literals: https://learn.microsoft.com/dotnet/csharp/language-reference/proposals/csharp-11.0/raw-string-literal
