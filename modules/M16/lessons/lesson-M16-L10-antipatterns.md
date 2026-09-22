[⬆ К модулю M16](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M16-L10: Anti-patterns: god object, anemic domain, service-locator / Anti-patterns: god object, anemic domain, service-locator

**Модуль / Module:** M16
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Антипаттерны — это повторяемые решения, которые выглядят как правильные, но на деле приносят больше вреда, чем пользы. Они возникают не от незнания, а от эволюции кода: «сделаем пока так», «добавим ещё одно поле», «вызовем сервис напрямую». Понимание антипаттернов критично, потому что зрелый код отличается от наивного именно умением замечать эти ловушки на ранней стадии.

**God Object (Божественный объект).** Это класс, который знает слишком много и делает слишком много. Он хранит состояние, валидирует данные, ходит в базу, отправляет письма и ещё рисует отчёты. Аналогия: один сотрудник в компании, который одновременно бухгалтер, юрист, программист и охранник — пока он один, всё работает, но стоит ему заболеть, встаёт вся фирма. God Object нарушает принцип единственной ответственности (SRP), его невозможно тестировать изолированно, а каждое изменение несёт риск регрессии во всей системе. Признаки: класс длиннее 500–1000 строк, десятки публичных методов из разных предметных областей, зависимости от половины проекта. Исправляют декомпозицией — выделяют узкоспециализированные сервисы, value objects, агрегаты по DDD.

**Anemic Domain Model (Анемичная доменная модель).** Обратная крайность: классы-контейнеры с публичными свойствами и пустыми методами. Вся логика вынесена в «сервисы», а доменные сущности превращаются в структуры данных. Аналогия: библиотека, где книги есть, но читать их нельзя — можно только переписать содержимое. Анемичная модель нарушает инкапсуляцию: состояние и поведение оказываются в разных местах, инварианты не защищены, любой сервис может сломать согласованность агрегата. Это часто побочный эффект ORM-генерации и культуры «DTO-везде». Исправляют переносом поведения в саму сущность: методы `Withdraw`, `Activate`, `Reassign`, которые проверяют правила и меняют состояние атомарно.

**Service Locator (анти-DI).** Паттерн «реестр сервисов», где объект сам запрашивает зависимости через статический `Locator.GetService<T>()`. В отличие от конструкторного внедрения зависимостей, зависимости становятся скрытыми: по сигнатуре класса нельзя понять, что ему нужно. Аналогия: ресторан, где вместо меню официант кричит на кухню «дай то, дай сё» — гость не знает, что ему подадут. Service Locator ломает тестируемость (статика плохо мокается), скрывает жизненный цикл зависимостей и порождает temporal coupling. Исправляют явным DI через конструктор.

**Shotgun Surgery (Хирургия дробовиком).** Одно логическое изменение требует правок в десятке файлов. Добавление поля «отчество» трогает сущность, DTO, маппер, валидатор, контроллер, базу, миграцию, отчёт. Это сигнал, что связанная логика разрезана неправильно. Аналогия: чтобы поменять лампочку, приходится перестраивать стену, крышу и проводку. Лечат группировкой изменений: то, что меняется вместе, должно жить вместе (cohesion).

**Magic Strings / Magic Numbers.** Жёстко закодированные литералы `"admin"`, `"POST"`, `42`, разбросанные по коду. Аналогия: пароль, написанный на стикере, приклеенном к каждому монитору. Их нельзя найти рефакторингом, легко опечататься, невозможно локализовать. Решение — константы, перечисления, конфигурация, `enum` с `DescriptionAttribute`.

**Как обнаружить.** Code review, метрики (cyclomatic complexity, afferent/efferent coupling), статические анализаторы (Roslyn, SonarQube, ArchUnitNET), тесты на архитектурные правила. Красные флаги: классы-«всё-в-одном», сущности без методов, статические вызовы `Locator`, строки-идентификаторы, правки в 10+ файлах на одно требование.

**Как исправить.** Декомпозиция по ответственности, перенос поведения в домен, явный DI, группировка изменений, замена литералов константами. Делать это инкрементально, прикрываясь тестами, чтобы не получить ещё один антипаттерн — Big Bang Rewrite.

#### Theory (EN)

Anti-patterns are recurring solutions that look reasonable but do more harm than good. They rarely come from ignorance; they grow out of code evolution — a quick fix here, one more field there, a direct service call instead of an abstraction. Mature code differs from naïve code mainly by the ability to spot these traps early.

**God Object.** A class that knows too much and does too much. It holds state, validates data, talks to the database, sends emails, and renders reports. Analogy: one employee who is simultaneously accountant, lawyer, programmer, and security guard. While they are at work, everything runs; when they fall sick, the whole company stops. A God Object violates the Single Responsibility Principle, is impossible to test in isolation, and turns every change into a system-wide regression risk. Symptoms: classes over 500–1000 lines, dozens of public methods spanning unrelated domains, dependencies on half the codebase. The cure is decomposition — extracting focused services, value objects, and DDD aggregates.

**Anemic Domain Model.** The opposite extreme: container classes with public properties and no behaviour. All logic lives in "services", and domain entities degrade into data structures. Analogy: a library where books exist but cannot be read — you can only rewrite their contents. Anemia breaks encapsulation: state and behaviour live apart, invariants are unprotected, and any service can corrupt aggregate consistency. It often grows from ORM code generation and a "DTO everywhere" culture. The fix is moving behaviour back into the entity: methods such as `Withdraw`, `Activate`, `Reassign` that enforce rules and mutate state atomically.

**Service Locator (anti-DI).** A "service registry" pattern where an object fetches its dependencies through a static `Locator.GetService<T>()`. Unlike constructor injection, dependencies become invisible — the class signature does not reveal what it needs. Analogy: a restaurant with no menu, where the waiter just shouts orders at the kitchen and the guest never knows what will arrive. Service Locator harms testability (statics are hard to mock), hides dependency lifetimes, and creates temporal coupling. The remedy is explicit dependency injection through constructors.

**Shotgun Surgery.** One logical change requires edits across dozens of files. Adding a "middle name" field touches the entity, DTO, mapper, validator, controller, database, migration, and report. This is a signal that related logic has been sliced incorrectly. Analogy: changing a lightbulb forces you to rebuild the wall, the roof, and the wiring. The cure is grouping changes — things that change together should live together (cohesion).

**Magic Strings / Magic Numbers.** Hard-coded literals such as `"admin"`, `"POST"`, or `42` scattered across the code. Analogy: a password written on a sticky note glued to every monitor. They cannot be found by refactoring tools, are easy to misspell, and resist localization. The solution is constants, enumerations, configuration, and `enum` with `DescriptionAttribute`.

**How to detect.** Code review, metrics (cyclomatic complexity, afferent/efferent coupling), static analysers (Roslyn, SonarQube, ArchUnitNET), and architecture tests. Red flags: "all-in-one" classes, entities without methods, static `Locator` calls, string-based identifiers, and changes spread across 10+ files for a single requirement.

**How to fix.** Decompose by responsibility, move behaviour into the domain, use explicit DI, group changes, and replace literals with constants. Do it incrementally under test coverage, otherwise you risk trading one anti-pattern for another — the Big Bang Rewrite.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — Примеры антипаттернов и их исправления
// Examples of anti-patterns and their fixes

using System.ComponentModel;

namespace Course.M16.Antipatterns;

// ===================================================================
// 1) ANTI-PATTERN: God Object — знает и делает слишком много
//    God Object: knows and does too much
// ===================================================================
public class OrderGod // НЕ ДЕЛАЙТЕ ТАК / DO NOT DO THIS
{
    public int Id { get; set; }
    public decimal Total { get; set; }
    public string Status { get; set; } = "new";      // magic string
    public string CustomerEmail { get; set; } = "";

    // Смешение ответственности: домен + БД + почта + отчёт
    public void SaveToDatabase(string connStr) { /* ... */ }       // RU: инфраструктура в домене
    public void SendEmail(string smtpHost) { /* ... */ }           // EN: infra inside domain
    public string RenderPdfReport() => $"Report #{Id}: {Total}";   // презентация в домене
}

// ===================================================================
// 1) FIX: декомпозиция по ответственности
//    Fix: decompose by responsibility
// ===================================================================
public sealed class Order // Богатая доменная модель / Rich domain model
{
    public int Id { get; }
    public OrderStatus Status { get; private set; }   // enum вместо magic string
    public decimal Total { get; private set; }        // состояние инкапсулировано

    public Order(int id, decimal total)
    {
        if (total < 0) throw new ArgumentOutOfRangeException(nameof(total));
        Id = id; Total = total; Status = OrderStatus.New;
    }

    // Поведение рядом с состоянием / Behaviour next to state
    public void Confirm()
    {
        if (Status != OrderStatus.New)
            throw new InvalidOperationException("Order cannot be confirmed"); // RU+EN guard
        Status = OrderStatus.Confirmed;
    }
}

public enum OrderStatus
{
    [Description("Новый / New")]      New,
    [Description("Подтверждён / Confirmed")] Confirmed,
    [Description("Отменён / Cancelled")] Cancelled
}

// Инфраструктура вынесена из домена / Infrastructure outside the domain
public interface IOrderRepository { Task SaveAsync(Order order, CancellationToken ct); }
public interface IEmailGateway     { Task SendAsync(string to, string body, CancellationToken ct); }

public sealed class OrderService // одна ответственность: оркестрация
{
    private readonly IOrderRepository _repo;
    private readonly IEmailGateway _email;
    public OrderService(IOrderRepository repo, IEmailGateway email) // явный DI
    {
        _repo = repo; _email = email;
    }

    public async Task PlaceAsync(Order order, string email, CancellationToken ct)
    {
        order.Confirm();                              // доменное правило внутри сущности
        await _repo.SaveAsync(order, ct);
        await _email.SendAsync(email, $"Order #{order.Id} confirmed", ct);
    }
}

// ===================================================================
// 2) ANTI-PATTERN: Anemic Domain Model — сущность без поведения
//    Anemic Domain Model: entity without behaviour
// ===================================================================
public class AccountAnemic // НЕ ДЕЛАЙТЕ ТАК / DO NOT DO THIS
{
    public decimal Balance { get; set; } // любой может сломать инвариант
}

public static class AccountServiceAnemic // логика отдельно от данных
{
    public static void Withdraw(AccountAnemic a, decimal amount)
    {
        a.Balance -= amount; // нет защиты от перерасхода / no overdraft guard
    }
}

// FIX: богатая модель с защищённым инвариантом
//     Rich model with a protected invariant
public sealed class Account
{
    public decimal Balance { get; private set; }
    public Account(decimal initial) => Balance = initial;

    public void Withdraw(decimal amount)
    {
        if (amount <= 0) throw new ArgumentOutOfRangeException(nameof(amount));
        if (amount > Balance) throw new InvalidOperationException("Insufficient funds");
        Balance -= amount; // инвариант проверен атомарно / invariant checked atomically
    }
}

// ===================================================================
// 3) ANTI-PATTERN: Service Locator — скрытые зависимости
//    Service Locator: hidden dependencies
// ===================================================================
public static class ServiceLocator // НЕ ДЕЛАЙТЕ ТАК / DO NOT DO THIS
{
    public static IServiceProvider Root { get; set; } = null!;
}

public sealed class BadOrderHandler
{
    public Task HandleAsync(Order order, CancellationToken ct)
    {
        // зависимость невидима в конструкторе / dependency hidden from constructor
        var email = ServiceLocator.Root.GetRequiredService<IEmailGateway>();
        return email.SendAsync("", "ok", ct);
    }
}

// FIX: явный DI через конструктор / explicit DI via constructor
public sealed class GoodOrderHandler
{
    private readonly IEmailGateway _email;
    public GoodOrderHandler(IEmailGateway email) => _email = email;

    public Task HandleAsync(Order order, CancellationToken ct)
        => _email.SendAsync("", $"Order #{order.Id}", ct);
}
```

#### Best Practices

- Держите классы сфокусированными: одна ответственность, измеримая через причину изменения (SRP).
- Переносите поведение в доменные сущности; сервисы должны оркестровать, а не владеть правилами.
- Внедряйте зависимости только через конструктор; избегайте статических локаторов и `serviceProvider` в поле класса.
- Заменяйте строковые идентификаторы перечислениями и константами; группируйте то, что меняется вместе.
- Покрывайте рефакторинг тестами и делайте его инкрементально, а не «большим взрывом».

- Keep classes focused: one responsibility, verifiable by a single reason to change (SRP).
- Move behaviour into domain entities; services should orchestrate, not own business rules.
- Inject dependencies only through constructors; avoid static locators and stored `serviceProvider`.
- Replace string identifiers with enumerations and constants; group things that change together.
- Cover refactors with tests and do them incrementally, not as a "big bang" rewrite.

#### Частые ошибки / Common Mistakes

- God Object «на вырост»: один класс копит методы ради «удобства» → выделяйте агрегаты и сервисы по границам ответственности.
- Публичные сеттеры на доменных сущностях, ломающие инварианты → делайте свойства `private set` и меняйте состояние только через методы.
- `ServiceLocator.Get<T>()` в коде приложения → требуйте зависимости через конструктор и проверяйте через DI-контейнер.
- Строки `"admin"`, `"paid"` в условиях → используйте `enum` или `Roles.Admin`-константы.
- Правка одного требования в 10+ файлах → пересмотрите границы модулей, объедините то, что меняется вместе.

- God Object "for later": one class accumulates methods for convenience → extract aggregates and services along responsibility boundaries.
- Public setters on domain entities that break invariants → use `private set` and mutate only via methods.
- `ServiceLocator.Get<T>()` in application code → require dependencies through constructors and verify via the DI container.
- Hard-coded strings like `"admin"`, `"paid"` in conditions → use `enum` or constants such as `Roles.Admin`.
- One requirement touching 10+ files → reconsider module boundaries; group what changes together.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Ни один класс не превышает одной ответственности и не разросся свыше разумного размера.
- [ ] Доменные сущности содержат поведение, а не только данные; инварианты защищены.
- [ ] Все зависимости внедряются через конструктор; нет статических локаторов.
- [ ] В коде нет magic strings/numbers; используются перечисления и константы.
- [ ] Одно логическое изменение затрагивает минимальное число файлов; границы модулей выровнены с изменениями.

- [ ] No class exceeds a single responsibility or grows beyond a reasonable size.
- [ ] Domain entities contain behaviour, not just data; invariants are protected.
- [ ] All dependencies are constructor-injected; no static locators remain.
- [ ] No magic strings/numbers remain; enumerations and constants are used.
- [ ] A single logical change touches a minimal number of files; module boundaries align with change.

#### Ресурсы / Resources

- Microsoft Learn — https://learn.microsoft.com/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/

---

[⬆ К модулю M16](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
