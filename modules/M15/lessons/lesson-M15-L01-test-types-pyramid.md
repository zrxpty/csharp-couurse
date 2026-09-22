[⬆ К модулю M15](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M15-L01: Виды тестов (unit/integration/e2e), пирамида / Test types (unit/integration/e2e), pyramid

**Модуль / Module:** M15
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Когда мы пишем код, мы неизбежно допускаем ошибки. Тесты — это наша система раннего оповещения: они сообщают об ошибках раньше, чем о них узнают пользователи. Но не все тесты одинаково полезны, и не все ошибки стоит ловить одинаковыми способами. Чтобы разобраться, представьте **пирамиду тестов** (Test Pyramid) — модель, которую популяризовал Майк Кон (Mike Cohn). В её основании лежит множество быстрых дешёвых проверок, а на вершине — единицы медленных и дорогих.

**Три уровня пирамиды.**

1. **Unit-тесты (модульные)** — основание пирамиды. Проверяют одну единицу кода — метод или класс — в изоляции от внешнего мира: базы данных, сети, файловой системы. Они быстрые (тысячи в секунду), детерминированные (один и тот же результат при каждом запуске) и точно указывают на сломанное место. Аналогия: проверка каждого кирпича отдельно перед укладкой в стену.

2. **Integration-тесты (интеграционные)** — середина. Проверяют, как несколько модулей работают вместе: например, репозиторий вместе с реальной (или in-memory) базой данных, контроллер с маршрутизацией. Они медленнее и сложнее, но ловят ошибки «на стыках», которые unit-тесты не видят. Аналогия: проверка, что кирпичи сложены в стену и между ними нет щелей.

3. **End-to-End (E2E, сквозные)** — вершина. Проверяют всю систему целиком, от пользовательского ввода до ответа базы данных, через реальный HTTP-запрос и браузер (или HTTP-клиент). Они самые медленные, самые хрупкие и самые дорогие в поддержке, но ближе всего к тому, что реально видит пользователь. Аналогия: проверка, что построенный дом не протекает под дождём.

**Соотношение слоёв.** Классическая пропорция — примерно 70% unit / 20% integration / 10% e2e. Это не закон, а ориентир. Чем выше по пирамиде, тем меньше тестов должно быть, потому что каждый верхний тест дороже в выполнении и в поддержке.

**Cost vs value.** Стоимость теста — это не только время его выполнения, но и время написания, сопровождения, расследования падений. Unit-тест стоит копейки и приносит огромную ценность: мгновенная обратная связь при разработке. E2E-тест стоит дорого и приносит меньше ценности «на единицу времени», но незаменим для проверки критичных пользовательских сценариев. Дешёвый и быстрый тест, который ловит баг, лучше дорогого и медленного, который ловит тот же баг. Поэтому стратегия проста: **сдвигайте тесты вниз** (shift-left) — старайтесь покрывать логику unit-тестами, а E2E оставлять для ключевых flow.

**Что тестировать.** Тестируйте поведение, а не реализацию: «что должен делать код», а не «как именно он это делает». Тестируйте публичный контракт, а не приватные методы. Тестируйте граничные случаи (пустая коллекция, отрицательные числа, null). Не тестируйте фреймворк и сторонние библиотеки — их уже протестировали авторы.

**Что НЕ стоит тестировать unit-тестами:** код работы с БД, HTTP-вызовы, UI-рендеринг. Эти слои либо выносятся за границу тестируемого класса через интерфейсы (и заменяются моками), либо покрываются integration/E2E-тестами.

**Антипаттерны пирамиды.** «Перевёрнутая пирамида» — много E2E, мало unit: набор медленный и хрупкий, баги локализуются плохо. «Леденец на палочке» (Ice Cream Cone) — мало unit, средне integration, много ручного E2E-тестирования: ещё хуже. «Конус мороженого» — те же проблемы плюс высокое содержание ручных проверок.

Хороший набор тестов — это инвестиция: чем умнее вы распределите их по пирамиде, тем быстрее будете получать обратную связь и тем увереннее — вносить изменения.

#### Theory (EN)

When we write code, we inevitably make mistakes. Tests are our early-warning system: they tell us about mistakes before users do. But not all tests are equally useful, and not all defects should be caught the same way. To make sense of this, picture the **Test Pyramid**, a model popularized by Mike Cohn. Its base holds many fast, cheap checks, while its peak holds a few slow, expensive ones.

**The three layers of the pyramid.**

1. **Unit tests** — the base of the pyramid. They test a single unit of code — a method or a class — in isolation from the outside world: database, network, file system. They are fast (thousands per second), deterministic (the same result every run), and pinpoint exactly what broke. Analogy: inspecting each brick individually before laying it into a wall.

2. **Integration tests** — the middle. They test how several modules work together: for example, a repository with a real (or in-memory) database, a controller with routing. They are slower and more complex, but catch defects "at the seams" that unit tests cannot see. Analogy: checking that the bricks are laid into a wall with no gaps between them.

3. **End-to-End (E2E)** — the peak. They test the whole system, from user input to database response, through a real HTTP request and a browser (or an HTTP client). They are the slowest, the most brittle, and the most expensive to maintain, but they are closest to what the user actually experiences. Analogy: checking that the finished house does not leak in the rain.

**The ratio of the layers.** A classic proportion is roughly 70% unit / 20% integration / 10% e2e. This is not a law, but a guideline. The higher up the pyramid, the fewer tests there should be, because each higher test is more expensive to run and to maintain.

**Cost vs value.** The cost of a test is not only its execution time, but also the time to write, maintain, and investigate failures. A unit test costs pennies and delivers huge value: instant feedback during development. An E2E test is expensive and yields less value "per unit of time", but is indispensable for verifying critical user scenarios. A cheap and fast test that catches a bug is better than an expensive and slow one that catches the same bug. So the strategy is simple: **shift tests down** (shift-left) — cover logic with unit tests, and reserve E2E for the key flows.

**What to test.** Test behavior, not implementation: "what the code should do", not "exactly how it does it". Test the public contract, not private methods. Test boundary cases (empty collection, negative numbers, null). Do not test the framework or third-party libraries — their authors have already tested them.

**What NOT to test with unit tests:** database-access code, HTTP calls, UI rendering. These layers are either moved outside the class under test through interfaces (and replaced with mocks), or covered by integration/E2E tests.

**Pyramid anti-patterns.** The "inverted pyramid" — many E2E, few unit: the suite is slow and brittle, and defects are hard to localize. The "ice cream cone" — few unit, some integration, lots of manual E2E: even worse. These shapes share the same problems: slow feedback, fragile suites, and poor defect localization.

A good test suite is an investment: the smarter you distribute tests across the pyramid, the faster your feedback loop and the more confident your changes.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — Пример трёх уровней пирамиды тестов на xUnit.
// Example of the three pyramid layers with xUnit.

using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using Xunit;

// --- Тестируемая система / System under test -----------------------------

public sealed class Cart
{
    private readonly List<CartItem> _items = new();

    // Публичный контракт — то, что мы покрываем unit-тестами.
    // Public contract — what we cover with unit tests.
    public IReadOnlyList<CartItem> Items => _items;

    public void Add(CartItem item)
    {
        ArgumentNullException.ThrowIfNull(item);
        if (item.Quantity <= 0)
            throw new ArgumentOutOfRangeException(nameof(item), "Quantity must be positive.");
        _items.Add(item);
    }

    public decimal Total() => _items.Sum(i => i.Price * i.Quantity);
}

public sealed record CartItem(string Sku, decimal Price, int Quantity);

public interface ICartRepository
{
    Task<Cart?> GetAsync(Guid userId, CancellationToken ct);
    Task SaveAsync(Cart cart, CancellationToken ct);
}

public sealed class CartController(ICartRepository repo) : ControllerBase
{
    // Контроллер тестируется integration-тестами (WebApplicationFactory).
    // The controller is tested with integration tests (WebApplicationFactory).
    [HttpGet("api/users/{userId:guid}/cart/total")]
    public async Task<ActionResult<decimal>> GetTotal(Guid userId, CancellationToken ct)
    {
        var cart = await repo.GetAsync(userId, ct);
        return cart is null ? NotFound() : Ok(cart.Total());
    }
}

public sealed class CartDbContext(DbContextOptions<CartDbContext> options) : DbContext(options)
{
    public DbSet<Cart> Carts => Set<Cart>();
}

// --- 1) UNIT-тест: быстро, изоляция, детерминированность -----------------

public sealed class CartUnitTests
{
    [Fact]
    public void Total_returns_zero_for_empty_cart() // ПУСТАЯ КОРЗИНА — граничный случай
    {
        var cart = new Cart();
        Assert.Equal(0m, cart.Total());
    }

    [Fact]
    public void Add_with_non_positive_quantity_throws() // ГРАНИЧНЫЙ СЛУЧАЙ
    {
        var cart = new Cart();
        var bad = new CartItem("SKU-1", 10m, quantity: 0);
        Assert.Throws<ArgumentOutOfRangeException>(() => cart.Add(bad));
    }

    [Theory]
    [InlineData(10, 2, 20)]
    [InlineData(7.5, 3, 22.5)]
    public void Total_sums_price_times_quantity(decimal price, int qty, decimal expected)
    {
        var cart = new Cart();
        cart.Add(new CartItem("SKU-1", price, qty));
        Assert.Equal(expected, cart.Total());
    }
}

// --- 2) INTEGRATION-тест: несколько модулей вместе, реальная БД (in-memory) --

public sealed class CartIntegrationTests
{
    [Fact]
    public async Task Repository_persists_and_retrieves_cart()
    {
        // In-memory provider EF Core — реальный DbContext, без сетевого диска.
        // EF Core in-memory provider — a real DbContext without a network disk.
        var options = new DbContextOptionsBuilder<CartDbContext>()
            .UseInMemoryDatabase("cart-it")
            .Options;

        await using var db = new CartDbContext(options);
        db.Carts.Add(new Cart());
        await db.SaveChangesAsync();

        var saved = await db.Carts.SingleAsync();
        Assert.NotNull(saved);
    }
}

// --- 3) E2E-тест (упрощённый): весь путь через HTTP -----------------------

public sealed class CartE2eTests
{
    [Fact]
    public async Task Get_total_returns_200_for_existing_user()
    {
        // В реальном проекте здесь используется WebApplicationFactory<Program>.
        // In a real project, you would use WebApplicationFactory<Program>.
        var stubRepo = new StubRepo(new Cart());
        var controller = new CartController(stubRepo);

        var result = await controller.GetTotal(Guid.NewGuid(), default);

        // Проверяем весь публичный путь: контроллер → репозиторий → модель.
        // We verify the whole public path: controller → repository → model.
        var ok = Assert.IsType<OkObjectResult>(result.Result);
        Assert.Equal(0m, Assert.IsType<decimal>(ok.Value));
    }

    private sealed class StubRepo(Cart cart) : ICartRepository
    {
        public Task<Cart?> GetAsync(Guid userId, CancellationToken ct) => Task.FromResult<Cart?>(cart);
        public Task SaveAsync(Cart c, CancellationToken ct) => Task.CompletedTask;
    }
}

// Саммари пирамиды / Pyramid summary:
//  • Unit — тысячи за секунду, точно локализуют баг. // localize bugs precisely.
//  • Integration — десятки за секунду, ловят ошибки на стыках. // catch seam bugs.
//  • E2E — единицы за минуту, проверяют пользовательские сценарии. // verify user flows.
```

#### Best Practices

- Держите пропорцию ~70/20/10 между unit/integration/e2e; сдвигайте проверки вниз, где это возможно.
- Unit-тесты должны быть детерминированными: никаких случайных дат, сетевых вызовов и реальных часов. Используйте интерфейсы и фейковые реализации (fakes/stubs/mocks).
- Один тест — одна причина упасть (одно логическое утверждение, аккуратный Arrange-Act-Assert).
- Дайте тестам понятные имена: `MethodName_Scenario_ExpectedResult` — имя должно читаться как спецификация.
- Integration-тесты должны поднимать общий контекст один раз на класс/сборку, а не на каждый тест, иначе набор станет медленным.
- E2E-тесты оставьте для критичных пользовательских сценариев (happy path и самые важные edge cases); не пытайтесь покрыть ими каждую ветку логики.
- Запускайте unit-тесты на каждом сохранении (в IDE/CI), integration — на PR, E2E — перед релизом или по расписанию.

#### Best Practices (EN)

- Keep a ~70/20/10 ratio between unit/integration/e2e; shift checks down wherever possible.
- Unit tests must be deterministic: no random dates, no network calls, no real clocks. Use interfaces and fake implementations (fakes/stubs/mocks).
- One test — one reason to fail (one logical assertion, a clean Arrange-Act-Assert).
- Give tests readable names: `MethodName_Scenario_ExpectedResult` — the name should read like a spec.
- Integration tests should set up shared context once per class/assembly, not per test, otherwise the suite becomes slow.
- Reserve E2E tests for critical user scenarios (the happy path and the most important edge cases); do not try to cover every logic branch with them.
- Run unit tests on every save (IDE/CI), integration on a PR, E2E before release or on a schedule.

#### Частые ошибки / Common Mistakes

- [Тестирование приватных методов через reflection] → [Тестируйте поведение через публичный контракт; приватные методы — деталь реализации] (RU)
- [Unit-тест, который реально лезет в БД или сеть] → [Вынесите зависимости в интерфейсы и подставьте fakes; иначе это уже integration-тест] (RU)
- [Один тест с десятью Assert на разные сценарии] → [Разделите на отдельные тесты или Theory, чтобы сразу видеть, что сломалось] (RU)
- [Тесты с разделяемым изменяемым состоянием между собой] → [Каждый тест должен быть независимым; поднимайте чистый контекст] (RU)
- [Перевёрнутая пирамида: 80% E2E, 10% unit] → [Сдвигайте проверки вниз; покрывайте логику unit-тестами, а E2E — только ключевые flow] (RU)
- [Тестирование фреймворка/библиотеки вместо своего кода] → [Тестируйте свой контракт; фреймворк уже протестировали его авторы] (RU)
- [Testing private methods via reflection] → [Test behavior through the public contract; private methods are an implementation detail] (EN)
- [A unit test that actually hits a database or the network] → [Move dependencies behind interfaces and inject fakes; otherwise it is already an integration test] (EN)
- [One test with ten Asserts over different scenarios] → [Split into separate tests or a Theory, so you can see exactly what broke] (EN)
- [Tests that share mutable state with each other] → [Each test must be independent; set up a fresh context] (EN)
- [Inverted pyramid: 80% E2E, 10% unit] → [Shift checks down; cover logic with unit tests, keep E2E for the key flows only] (EN)
- [Testing a framework/library instead of your own code] → [Test your own contract; the framework was already tested by its authors] (EN)

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Я знаю три уровня пирамиды и могу объяснить их отличия одним предложением каждому. (RU)
- [ ] Мои unit-тесты детерминированы и не трогают БД/сеть/файловую систему. (RU)
- [ ] Я тестирую поведение, а не реализацию; приватные методы не покрываю напрямую. (RU)
- [ ] Соотношение в проекте близко к 70/20/10, без перевёрнутой пирамиды. (RU)
- [ ] Каждый тест имеет понятное имя вида Method_Scenario_Expected и одну причину упасть. (RU)
- [ ] У меня есть хотя бы один integration-тест, проверяющий репозиторий с реальным DbContext. (RU)
- [ ] У меня есть хотя бы один E2E-тест на ключевой пользовательский сценарий через HTTP. (RU)
- [ ] I know the three layers of the pyramid and can explain each in a single sentence. (EN)
- [ ] My unit tests are deterministic and do not touch DB/network/file system. (EN)
- [ ] I test behavior, not implementation; I do not cover private methods directly. (EN)
- [ ] The project ratio is close to 70/20/10, without an inverted pyramid. (EN)
- [ ] Every test has a readable name like Method_Scenario_Expected and a single reason to fail. (EN)
- [ ] I have at least one integration test verifying a repository with a real DbContext. (EN)
- [ ] I have at least one E2E test for a key user scenario over HTTP. (EN)

#### Ресурсы / Resources

- [Microsoft Learn — Testing ASP.NET Core microservices and web apps — https://learn.microsoft.com/dotnet/architecture/microservices/multi-container-microservice-net-applications/testing-asp-net-core-microservices-web-apps](https://learn.microsoft.com/dotnet/architecture/microservices/multi-container-microservice-net-applications/testing-asp-net-core-microservices-web-apps)
- Mike Cohn, *Succeeding with Agile* — источник классической пирамиды тестов. / The origin of the classic test pyramid.
- Martin Fowler, «Test Pyramid» — https://martinfowler.com/bliki/TestPyramid.html

---

[⬆ К модулю M15](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
