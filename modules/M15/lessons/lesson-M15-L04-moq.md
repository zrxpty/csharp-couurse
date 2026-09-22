[⬆ К модулю M15](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M15-L04: Moq: Mock<T>, Setup, Verify, ItExpr / Moq: Mock<T>, Setup, Verify, ItExpr

**Модуль / Module:** M15
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Moq (произносится «Mock-you» или просто «мок») — самая популярная библиотека для создания тестовых двойников (test doubles) в экосистеме .NET. Она построена поверх Castle DynamicProxy и позволяет на лету генерировать реализацию любого интерфейса или виртуального метода класса. Главная идея: вместо того чтобы писать ручные заглушки (stubs) и шпионы (spies) для каждого теста, вы описываете поведение декларативно через `Setup`, а затем проверяете факты взаимодействия через `Verify`.

Аналогия из жизни: представьте, что вы репетируете сцену в театре, но партнёр по сцене не выучил роль. Вместо живого актёра вы ставите робота-суфлёра, который по команде произносит нужную реплику и записывает, сколько раз вы к нему обращались. `Mock<T>` — это такой робот-суфлёр для вашего кода: он «играет» зависимость по заданному сценарию и ведёт журнал вызовов.

Базовый сценарий использования состоит из трёх шагов:
1. Создать мок: `var repo = new Mock<IUserRepository>();`
2. Настроить поведение: `repo.Setup(r => r.GetByIdAsync(It.IsAny<int>())).ReturnsAsync(user);`
3. Проверить факт вызова: `repo.Verify(r => r.GetByIdAsync(42), Times.Once);`

`Mock<T>` — обёртка, где `T` — тип интерфейса или класса с виртуальными методами. Свойство `.Object` возвращает сам экземпляр-двойник, который можно передавать в тестируемую систему (SUT — System Under Test).

`Setup(...)` описывает, что вернуть или выбросить, когда метод зовут с определёнными аргументами. `It.IsAny<T>()` означает «любое значение типа T», `It.Is<T>(predicate)` — фильтр по условию, а `ItExpr` используется для expression-based сопоставления — прежде всего вместе с `Moq.Protected` для настройки и проверки защищённых (`protected`) методов по строковому имени. Возврат значений задаётся через `Returns` (синхронно) и `ReturnsAsync` (асинхронно, `Task<T>`), а `Throws` / `ThrowsAsync` эмулируют ошибки.

`Verify(...)` проверяет, что метод был вызван с указанными аргументами нужное число раз — `Times.Once`, `Times.AtLeast(2)`, `Times.Never` и т.д. Это и есть **behavior verification**: тест интересует не только финальное состояние, но и сам факт взаимодействия с зависимостью.

`Callback(...)` позволяет выполнить произвольный код при вызове — например, сохранить переданный аргумент или обновить счётчик. Это полезно, когда метод имеет побочные эффекты или нужно симулировать асинхронное событие через `Raise`.

Поведенческая (behavior) проверка отвечает «что SUT сделал с зависимостью», а проверка состояния (state) — «каков результат работы SUT». Хорошее тестирование комбинирует оба подхода: состояние проверяет конечный результат, поведение — корректность маршрутизации вызовов. Перебарщивать с `Verify` легко — каждый лишний вызов делает тест хрупким; поэтому верифицируйте только значимые взаимодействия, влияющие на бизнес-логику.

По умолчанию Moq создаёт «свободные» моки (Loose): ненастроенные методы возвращают `default`. Режим `MockBehavior.Strict` превращает любой ненастроенный вызов в ошибку — это строже, но точнее ловит «забытые» настройки. Выбирайте Loose для широкого заглушивания, Strict — когда нужен явный контракт по каждому тесту.

#### Theory (EN)

Moq (pronounced "Mock-you" or simply "mok") is the most popular library for creating test doubles in the .NET ecosystem. It is built on top of Castle DynamicProxy and can generate, at runtime, an implementation of any interface or virtual method of a class. The core idea: instead of writing hand-rolled stubs and spies for every test, you describe behavior declaratively through `Setup` and then verify interaction facts through `Verify`.

Real-world analogy: imagine rehearsing a stage play where your scene partner hasn't learned their lines. Rather than waiting for a real actor, you bring in a robot prompter that speaks the right line on command and keeps a tally of how many times you addressed it. `Mock<T>` is exactly such a robot prompter for your code: it "plays" a dependency according to a script you define and keeps a journal of calls.

The basic usage flow has three steps:
1. Create the mock: `var repo = new Mock<IUserRepository>();`
2. Configure behavior: `repo.Setup(r => r.GetByIdAsync(It.IsAny<int>())).ReturnsAsync(user);`
3. Verify the call: `repo.Verify(r => r.GetByIdAsync(42), Times.Once);`

`Mock<T>` is a wrapper where `T` is an interface type or a class with virtual methods. The `.Object` property returns the double instance itself, which you pass into the System Under Test (SUT).

`Setup(...)` describes what to return or throw when a method is invoked with specific arguments. `It.IsAny<T>()` means "any value of type T", `It.Is<T>(predicate)` is a conditional filter, and `ItExpr` is used for expression-based matching — primarily together with `Moq.Protected` to set up and verify `protected` members by their string name. Return values are set with `Returns` (synchronous) and `ReturnsAsync` (asynchronous, `Task<T>`), while `Throws` / `ThrowsAsync` simulate exceptions.

`Verify(...)` checks that a method was called with the specified arguments the expected number of times — `Times.Once`, `Times.AtLeast(2)`, `Times.Never`, and so on. This is **behavior verification**: the test cares not only about the final state but also about the fact of interacting with the dependency.

`Callback(...)` lets you run arbitrary code on invocation — for example, capture a passed argument or update a counter. This is useful when a method has side effects or when you need to simulate an asynchronous event through `Raise`.

Behavior verification answers "what did the SUT do with the dependency?" while state verification answers "what is the result of the SUT's work?" Good testing combines both: state checks the end result, behavior checks the correctness of call routing. Over-using `Verify` is easy — every extra call makes tests brittle; verify only meaningful interactions that affect business logic.

By default Moq creates loose mocks: unconfigured methods return `default`. The `MockBehavior.Strict` mode turns any unconfigured call into an error — stricter, but it catches forgotten setups more accurately. Choose Loose for broad stubbing, Strict when you want an explicit contract per test.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — рабочий пример Moq с интерфейсами и защищёнными методами
// Working Moq example covering interfaces and protected methods
using System;
using System.IO;
using System.Net;
using System.Net.Http;
using System.Threading.Tasks;
using Moq;
using Moq.Protected;
using Xunit;

// Интерфейс зависимости / Dependency interface
public interface IUserRepository
{
    Task<User?> GetByIdAsync(int id);
    Task SaveAsync(User user);
    event EventHandler<User>? UserSaved;   // событие для демонстрации Raise / event for Raise demo
}

// Доменная модель / Domain model
public sealed record User(int Id, string Name, int Age);

// Тестируемая система / System under test
public sealed class UserService(IUserRepository repo)
{
    public async Task<string> GetDisplayNameAsync(int id)
    {
        var user = await repo.GetByIdAsync(id)
            ?? throw new InvalidOperationException($"User {id} not found");
        return user.Age >= 18 ? user.Name : $"{user.Name} (minor)";
    }

    public async Task BirthdayAsync(int id)
    {
        var user = await repo.GetByIdAsync(id)
            ?? throw new InvalidOperationException($"User {id} not found");

        var updated = user with { Age = user.Age + 1 };   // record with-expression
        await repo.SaveAsync(updated);
    }
}

public class UserServiceTests
{
    private readonly Mock<IUserRepository> _repo = new(MockBehavior.Loose);
    private readonly UserService _sut;

    public UserServiceTests() => _sut = new UserService(_repo.Object);

    [Fact]
    public async Task GetDisplayName_Adult_ReturnsPlainName()
    {
        // Arrange: настраиваем возвращаемое значение / arrange return value
        var user = new User(42, "Alice", 30);
        _repo.Setup(r => r.GetByIdAsync(42))
             .ReturnsAsync(user);

        // Act
        var name = await _sut.GetDisplayNameAsync(42);

        // Assert (state): проверяем результат / state verification
        Assert.Equal("Alice", name);
        // Assert (behavior): проверяем факт вызова / behavior verification
        _repo.Verify(r => r.GetByIdAsync(42), Times.Once);
    }

    [Fact]
    public async Task GetDisplayName_Minor_AppendsSuffix()
    {
        // It.IsAny — любое значение типа int / any int value
        _repo.Setup(r => r.GetByIdAsync(It.IsAny<int>()))
             .ReturnsAsync(new User(7, "Bob", 12));

        var name = await _sut.GetDisplayNameAsync(7);

        Assert.Equal("Bob (minor)", name);
    }

    [Fact]
    public async Task GetDisplayName_MissingUser_Throws()
    {
        // ReturnsAsync(null) — симулируем отсутствие / simulate absence
        _repo.Setup(r => r.GetByIdAsync(99))
             .ReturnsAsync((User?)null);

        await Assert.ThrowsAsync<InvalidOperationException>(
            () => _sut.GetDisplayNameAsync(99));
    }

    [Fact]
    public async Task Birthday_IncrementsAgeAndSaves()
    {
        var stored = new User(1, "Carol", 17);
        _repo.Setup(r => r.GetByIdAsync(1)).ReturnsAsync(stored);

        // Callback: перехватываем сохранённую модель / capture saved model
        User? saved = null;
        _repo.Setup(r => r.SaveAsync(It.IsAny<User>()))
             .Callback<User>(u => saved = u)
             .Returns(Task.CompletedTask);

        await _sut.BirthdayAsync(1);

        Assert.NotNull(saved);
        Assert.Equal(18, saved!.Age);                          // state verification
        _repo.Verify(r => r.SaveAsync(It.IsAny<User>()), Times.Once); // behavior verification
    }

    [Fact]
    public async Task SaveAsync_Throws_PropagatesError()
    {
        _repo.Setup(r => r.GetByIdAsync(It.IsAny<int>()))
             .ReturnsAsync(new User(1, "X", 20));
        // ThrowsAsync — эмуляция исключения / simulate exception
        _repo.Setup(r => r.SaveAsync(It.IsAny<User>()))
             .ThrowsAsync(new IOException("disk full"));

        await Assert.ThrowsAsync<IOException>(() => _sut.BirthdayAsync(1));
    }

    [Fact]
    public void Event_Raises_OnSave()
    {
        User? raised = null;
        _repo.Object.UserSaved += (_, u) => raised = u;

        // Raise — эмуляция события со стороны мока / simulate event from the mock
        _repo.Raise(r => r.UserSaved += null, new User(5, "Eve", 25));

        Assert.Equal("Eve", raised?.Name);
    }
}

// --- ItExpr + Moq.Protected: настройка защищённых методов ---
// ItExpr + Moq.Protected: setting up protected members
public abstract class HttpClientWrapper
{
    // Защищённый метод — обычный Setup по выражению недоступен
    // Protected member — cannot be set up via a normal expression
    protected internal abstract Task<HttpResponseMessage> SendAsync(HttpRequestMessage req);

    public Task<HttpResponseMessage> ExecuteAsync(HttpRequestMessage req) => SendAsync(req);
}

public class HttpClientWrapperTests
{
    [Fact]
    public async Task SendAsync_IsInvoked_Once()
    {
        var http = new Mock<HttpClientWrapper>();

        // ItExpr.IsAny<T> — expression-матчинг для protected-метода по имени
        // ItExpr.IsAny<T> — expression matching for a protected method by name
        http.Protected()
            .Setup<Task<HttpResponseMessage>>("SendAsync", ItExpr.IsAny<HttpRequestMessage>())
            .ReturnsAsync(new HttpResponseMessage(HttpStatusCode.OK));

        var response = await http.Object.ExecuteAsync(new HttpRequestMessage());

        Assert.Equal(HttpStatusCode.OK, response.StatusCode);

        // Verify защищённого метода — тоже через ItExpr / verify protected member via ItExpr
        http.Protected()
            .Verify("SendAsync", Times.Once(), ItExpr.IsAny<HttpRequestMessage>());
    }
}
```

#### Best Practices
- Верифицируйте только значимые вызовы: один-два `Verify` на тест, иначе тест становится хрупким и ломается при безобидном рефакторинге.
- Предпочитайте `MockBehavior.Loose` для большинства интеграционных сценариев и `Strict` — когда нужен явный контракт по каждому вызову теста.
- Используйте `It.Is<T>(x => ...)` вместо `It.IsAny<T>()`, когда аргумент важен для бизнес-логики — так тест ловит реальные баги, а не просто «любой вызов».
- Выносите создание мока и SUT в конструктор тест-класса и переопределяйте только нужные `Setup` в каждом тесте — меньше дублирования.
- Для `Task` без возвращаемого значения используйте `.Returns(Task.CompletedTask)`, а не `ReturnsAsync` — это точнее передаёт намерение.
- Prefer `It.Is<T>(x => ...)` over `It.IsAny<T>()` when the argument matters to business logic — the test then catches real bugs instead of "any call".
- Keep `Verify` count low (one or two per test); over-verification makes tests brittle under harmless refactoring.
- Use `MockBehavior.Strict` only when an explicit per-call contract adds value; default to `Loose` for broad stubbing.
- Reset shared mocks between tests or create a fresh instance per test to avoid state leakage across cases.
- Prefer `Callback` to capture arguments for later assertions rather than reaching into the mock's internal state.

#### Частые ошибки / Common Mistakes
- `Setup` на не-виртуальный метод класса → мок ничего не перехватит. Избегайте: мокайте интерфейсы или помечайте методы `virtual`, либо используйте `Moq` поверх абстракций.
- Забыли `.Returns(...)` / `.ReturnsAsync(...)` → метод вернёт `default` (включая `null`), и SUT упадёт с `NullReferenceException`. Всегда явно настраивайте возвращаемое значение.
- `Verify(...)` после того, как метод не был вызван из-за раннего `return` → тест зелёный, но ничего не проверяет. Добавляйте и state-проверку результата.
- Использование `It.IsAny` там, где важен конкретный аргумент → баг остаётся незамеченным. Заменяйте на `It.Is<T>(predicate)`.
- Мокание `DateTime.Now`, `Guid.NewGuid()` и других статических членов через Moq → не работает. Используйте обёртки-интерфейсы (`IClock`, `IGuidProvider`).
- `Setup` on a non-virtual class member → the mock silently intercepts nothing. Mock interfaces, mark members `virtual`, or abstract the dependency behind an interface.
- Forgetting `.Returns(...)` / `.ReturnsAsync(...)` → method returns `default` (including `null`) and SUT crashes with `NullReferenceException`. Always set an explicit return.
- `Verify` after an early `return` skipped the call → test is green but asserts nothing. Pair with state verification of the result.
- Using `It.IsAny` where a specific argument matters → the bug slips through. Switch to `It.Is<T>(predicate)`.
- Trying to mock `DateTime.Now`, `Guid.NewGuid()`, or other static members with Moq → not supported. Wrap them behind interfaces (`IClock`, `IGuidProvider`).

#### Чек-лист самопроверки / Self-check Checklist
- [ ] Я создаю мок через `new Mock<T>()` и передаю в SUT свойство `.Object`.
- [ ] Каждый `Setup` явно указывает возвращаемое значение (`Returns` / `ReturnsAsync`) или исключение (`Throws`).
- [ ] Я различаю state-проверку (результат) и behavior-проверку (`Verify` + `Times`).
- [ ] Я использую `It.Is<T>(predicate)` вместо `It.IsAny<T>()`, когда аргумент важен.
- [ ] Я знаю, как вызвать `Moq.Protected` + `ItExpr` для защищённых методов.
- [ ] Я не мокаю статические члены и классы без `virtual` методов.
- [ ] I create a mock via `new Mock<T>()` and pass `.Object` into the SUT.
- [ ] Every `Setup` states an explicit return (`Returns` / `ReturnsAsync`) or exception (`Throws`).
- [ ] I distinguish state verification (result) from behavior verification (`Verify` + `Times`).
- [ ] I use `It.Is<T>(predicate)` instead of `It.IsAny<T>()` when the argument matters.
- [ ] I know how to use `Moq.Protected` + `ItExpr` for protected members.
- [ ] I avoid mocking static members and classes without `virtual` methods.

#### Ресурсы / Resources
- [Moq Quickstart (GitHub Wiki) — https://github.com/Moq/moq/wiki/Quickstart](https://github.com/Moq/moq/wiki/Quickstart)
- [Moq API Reference — https://moq.github.io/moq4/](https://moq.github.io/moq4/)
- [Martin Fowler — Mocks Aren't Stubs — https://martinfowler.com/articles/mocksArentStubs.html](https://martinfowler.com/articles/mocksArentStubs.html)
- [Microsoft Learn — Unit testing best practices — https://learn.microsoft.com/dotnet/core/testing/](https://learn.microsoft.com/dotnet/core/testing/)

---

[⬆ К модулю M15](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
