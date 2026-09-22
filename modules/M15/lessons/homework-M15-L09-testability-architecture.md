---
[← К уроку M15-L09](lesson-M15-L09-testability-architecture.md) | [⬆ К модулю M15](../README.md) | [Следующее ДЗ →]()
---

### Домашнее задание M15-L09: Тестируемость как архитектурное свойство (вступление к M16) / Homework M15-L09: Testability as architectural property (intro to M16)

**Урок / Lesson:** M15-L09
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) На конкретном примере «нетестируемого» сервиса пройти путь от запутанного класса с `new` внутри методов к архитектуре со швами (seams), внедрением зависимостей через конструктор, чистыми функциями и DI-композицией, и убедиться, что итоговые тесты детерминированы, параллельны и не требуют инфраструктуры. (EN) Take a concrete untestable service from a tangled class with `new` inside its methods to an architecture with seams, constructor dependency injection, pure functions and DI composition, and prove that the resulting tests are deterministic, parallel and infrastructure-free.

#### Связь с уроком / Connection to the lesson
(RU) Урок утверждает, что тестируемость — это не «наличие unit-тестов», а архитектурное свойство, измеряющее зацепление и связность системы. ДЗ закрепляет это, давая код, в котором швов нет (`new HttpClient()`, `DateTime.Now` в методах), и требуя создать швы через `IClock`, `IPriceRepository`, `INotifier`, вынести расчёты в чистые функции и собрать всё в `CompositionRoot`. Это прямой мост к M16 (SOLID, Clean Architecture, DDD/CQRS).
(EN) The lesson states that testability is not "having unit tests" but an architectural property measuring coupling and cohesion. This homework reinforces it by giving code with no seams (`new HttpClient()`, `DateTime.Now` inside methods) and requiring you to introduce seams via `IClock`, `IPriceRepository`, `INotifier`, extract calculations into pure functions, and assemble everything in a `CompositionRoot`. It is a direct bridge to M16 (SOLID, Clean Architecture, DDD/CQRS).

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

В стартапе «МаркетОк» есть легаси-сервис `CheckoutService`, который делает сразу всё: читает цены из «базы» (на деле — захардкоженный `switch`), считает скидки по календарю и объёму, отправляет уведомление по HTTP и пишет в лог. Команда пробовала покрыть его unit-тестами, но первый же тест потребовал подмены реального времени, сетевого вызова и логгера — и застрял. Разработчики написали «трудно тестировать», закрыли тикет и пошли писать тесты только на чистые утилиты. Тем временем баги в скидках повторялись каждый квартал, потому что никто не мог воспроизвести поведение декабря в июле.

Урок M15-L09 объясняет, почему так происходит: класс без швов нельзя проверить изолированно. `new DateTime()` и `new HttpClient()` прямо в методе — это отсутствие шва, и моки тут не помогут, потому что подменять нечего: объект создаётся внутри и тут же используется. Боль не в тестах, а в архитектуре. Чтобы вылечить сервис, нужно не «дописать тесты», а перестроить зависимости: вынести время за `IClock`, цены за `IPriceRepository`, уведомления за `INotifier`, а логику скидок — в чистую функцию без состояния и I/O. Тогда домен проверяется без БД, без сети и без реального времени, а инфраструктура тестируется отдельно как integration tests.

Это и есть тестируемость как архитектурное свойство: способность компонентов проверяться изолированно, предсказуемо и быстро. В DЗ вы пройдёте этот путь сами — от легаси-класса к композитной архитектуре с швами, и напишете тесты, которые доказывают, что стало лучше. Заодно вы нащупаете границу M16: где кончается «просто DI» и начинаются SOLID, Clean Architecture и DDD.

#### Что нужно сделать (пошагово)

1. **Создайте решение и проекты.** В папке `M15-L09-HW/` выполните:
   ```
   dotnet new sln -n Marketok
   dotnet new classlib -n Marketok.Domain -f net8.0
   dotnet new classlib -n Marketok.Infrastructure -f net8.0
   dotnet new xunit -n Marketok.Tests -f net8.0
   dotnet sln add **/*.csproj
   dotnet add Marketok.Infrastructure reference Marketok.Domain
   dotnet add Marketok.Tests reference Marketok.Domain Marketok.Infrastructure
   dotnet add Marketok.Tests package Microsoft.Extensions.DependencyInjection
   ```
   Ожидаемо: четыре проекта, ссылки идут внутрь (Infrastructure → Domain, Tests → оба). Это направление зависимостей Clean Architecture — наружные слои зависят от внутренних.

2. **Перенесите легаси-код.** В `Marketok.Domain/Legacy/CheckoutService.cs` положите класс ниже (это отправная точка — он намеренно нетестируем):
   ```csharp
   public class LegacyCheckoutService
   {
       public Receipt Checkout(Order order)
       {
           var basePrice = order.Sku switch { "A1" => 10m, "B2" => 25m, _ => 1m };
           var now = DateTime.Now;
           var seasonal = now.Month is 12 or 1 ? 0.10m : 0m;
           var bulk = order.Quantity >= 100 ? 0.05m : 0m;
           var discount = seasonal + bulk;
           var total = basePrice * order.Quantity * (1m - discount);

           using var http = new HttpClient();
           http.PostAsJsonAsync("https://notify.example.com/receipt", new { order.Sku, total }).Wait();

           File.AppendAllText("checkout.log", $"{now:o} {order.Sku} {total}\n");
           return new Receipt(order.Sku, order.Quantity, basePrice, discount, total);
       }
   }
   public sealed record Order(string Sku, int Quantity);
   public sealed record Receipt(string Sku, int Quantity, decimal BasePrice, decimal Discount, decimal Total);
   ```
   Соберите: `dotnet build`. Должно компилироваться (при необходимости добавьте `using System.Net.Http;` и `using System.Text.Json;`, а `PostAsJsonAsync` замените на ручную сериализацию, если нет `Microsoft.AspNet.WebApi.Client`).

3. **Покажите, что тест сломан.** В `Marketok.Tests` попробуйте написать тест, проверяющий скидку в декабре. Вы обнаружите, что невозможно зафиксировать время (`DateTime.Now` создаётся внутри), нельзя избежать HTTP-вызова (он реально пойдёт в сеть) и нельзя отключить лог в файл. Зафиксируйте вывод `dotnet test` — он красный или недетерминирован. Это и есть «боль в кости».

4. **Введите швы.** В `Marketok.Domain/Seams/` создайте интерфейсы:
   ```csharp
   public interface IClock { DateTime UtcNow { get; } }
   public interface IPriceRepository { decimal GetBasePrice(string sku); }
   public interface INotifier { Task NotifyReceiptAsync(string sku, decimal total, CancellationToken ct); }
   public interface IReceiptSink { void Append(DateTime utc, string sku, decimal total); }
   ```
   Обратите внимание: швы проходят по границе «домен ↔ инфраструктура». Внутри домена работают конкретные типы (`Order`, `Receipt`), наружу торчат только абстракции.

5. **Вынесите чистую функцию.** В `Marketok.Domain/Pure/DiscountPolicy.cs`:
   ```csharp
   public static class DiscountPolicy
   {
       public static decimal CalculateDiscount(Order order, DateTime utcNow)
       {
           var seasonal = utcNow.Month is 12 or 1 ? 0.10m : 0m;
           var bulk = order.Quantity >= 100 ? 0.05m : 0m;
           return seasonal + bulk;
       }
   }
   ```
   Здесь нет состояния и нет I/O — это «чистая зона». Используется паттерн-матчинг `is 12 or 1` (C# 12).

6. **Перепишите сервис.** В `Marketok.Domain/CheckoutService.cs` создайте новый класс, принимающий все зависимости через конструктор (primary constructor C# 12):
   ```csharp
   public sealed class CheckoutService(
       IClock clock, IPriceRepository prices, INotifier notifier, IReceiptSink sink)
   {
       public async Task<Receipt> CheckoutAsync(Order order, CancellationToken ct = default)
       {
           var basePrice = prices.GetBasePrice(order.Sku);
           var discount = DiscountPolicy.CalculateDiscount(order, clock.UtcNow);
           var total = basePrice * order.Quantity * (1m - discount);
           await notifier.NotifyReceiptAsync(order.Sku, total, ct);
           sink.Append(clock.UtcNow, order.Sku, total);
           return new Receipt(order.Sku, order.Quantity, basePrice, discount, total);
       }
   }
   ```
   Никакого `new HttpClient()`, `DateTime.Now` или `File.AppendAllText` в методе больше нет. Появились швы.

7. **Реализуйте инфраструктуру.** В `Marketok.Infrastructure/`:
   - `SystemClock : IClock` → `DateTime.UtcNow`;
   - `DbPriceRepository : IPriceRepository` → имитация запроса к БД (хардкод `switch`);
   - `HttpNotifier : INotifier` → реальный `HttpClient` (в ДЗ достаточно скелета);
   - `FileReceiptSink : IReceiptSink` → `File.AppendAllText`.

8. **Соберите CompositionRoot.** В `Marketok.Infrastructure/CompositionRoot.cs`:
   ```csharp
   public static class CompositionRoot
   {
       public static ServiceProvider Build() => new ServiceCollection()
           .AddSingleton<IClock, SystemClock>()
           .AddSingleton<IPriceRepository, DbPriceRepository>()
           .AddSingleton<INotifier, HttpNotifier>()
           .AddSingleton<IReceiptSink, FileReceiptSink>()
           .AddSingleton<CheckoutService>()
           .BuildServiceProvider();
   }
   ```
   Шов зашивается один раз, на границе системы.

9. **Напишите тесты.** В `Marketok.Tests` используйте фейки:
   ```csharp
   internal sealed class FakeClock(DateTime fixedUtc) : IClock { public DateTime UtcNow => fixedUtc; }
   internal sealed class StubPrices : IPriceRepository { public decimal GetBasePrice(string sku) => 10m; }
   internal sealed class SpyNotifier : INotifier {
       public (string Sku, decimal Total)? Last;
       public Task NotifyReceiptAsync(string sku, decimal total, CancellationToken ct) { Last = (sku, total); return Task.CompletedTask; } }
   internal sealed class SpySink : IReceiptSink {
       public List<(DateTime, string, decimal)> Entries = new();
       public void Append(DateTime utc, string sku, decimal total) => Entries.Add((utc, sku, total)); }
   ```
   Покройте: декабрь + bulk = 15%, июль без bulk = 0%, январь = 10%, неизвестный SKU с ценой по умолчанию, отмена по `CancellationToken`. Все тесты запускаются параллельно, ни одного обращения к сети/файлу.

10. **Запустите и сравните.** `dotnet test -v n`. Должно быть зелено и детерминированно. Сравните с выводом шага 3 — теперь тесты не зависят от реального времени и сети.

#### Требования к решению

- Решение на C# 12 / .NET 8 (target framework `net8.0`). Используйте top-level statements в `Program.cs` (если будет демонстрационный хост), primary constructors, pattern matching (`is 12 or 1`, `switch` expression), sealed records для DTO, `CancellationToken` во всех асинхронных методах.
- Структура: `Marketok.Domain` (домен + швы + чистые функции), `Marketok.Infrastructure` (реализации + CompositionRoot), `Marketok.Tests` (только fakes/stubs/spies, без Moq — чтобы прочувствовать швы руками). Зависимости проектов идут внутрь: Infrastructure → Domain, Tests → оба.
- Ни один unit-тест не должен обращаться к реальному времени, файловой системе или сети. Все эти источники недетерминированности спрятаны за интерфейсами и заменены фейками.
- Чистая функция `DiscountPolicy.CalculateDiscount` не должна принимать ничего, кроме данных (`Order`, `DateTime`). Никаких `IClock` внутри чистой функции — время туда приходит параметром, чтобы функция оставалась чистой.
- В `CheckoutService` не должно быть ни одного `new` инфраструктурного класса внутри метода. Все коллабораторы приходят через конструктор. Если для теста нужно 4+ фейка — допустимо для этого класса (он координатор); но если бы их было 7+, это уже запах SRP-нарушения.
- Тесты должны проверять поведение, а не реализацию: не привязывайтесь к приватным полям, не считайте порядок вызовов, не мокайте мапперы и DTO. Утверждайте на итоговом `Receipt`, на `Last` спая и на списке `Entries`.

#### Тонкости и подводные камни

- **`DateTime.Now` vs `DateTime.UtcNow` vs `IClock`.** Легаси использует `DateTime.Now` — это вдвойне плохо: локальный часовой пояс делает тест недетерминированным на CI в другом регионе. Шов `IClock` должен возвращать `UtcNow`, а чистая функция сравнивает `Month` уже по UTC-дате. В тесте передавайте `new DateTime(2024,12,15,0,0,0,DateTimeKind.Utc)` — без `DateTimeKind.Utc` конвертации могут дать сюрпризы.
- **`new HttpClient()` в методе — это не только тестируемость, но и производительность.** Класс `HttpClient` предназначен для переиспользования; частое создание исчерпывает сокеты. Правильно — `IHttpClientFactory` или синглтон-`HttpClient`, обёрнутый в `INotifier`. Шов решает обе проблемы сразу.
- **`Task.Wait()` в легаси — синхронная блокировка над async.** В асинхронном пути используйте `await` и прокидывайте `CancellationToken`. Не пишите `.Result` и `.Wait()` — это тупики в UI/ASP.NET-контекстах и потеря токена отмены.
- **Чистая функция не должна «знать» про `IClock`.** Если протащить `IClock` внутрь `DiscountPolicy`, функция перестанет быть чистой (зависимость от интерфейса — всё равно зависимость от состояния времени). Передавайте туда `DateTime` значением — тогда её можно тестировать без DI-контейнера вообще.
- **Граница швов.** Швы должны лежать на границе слоёв, а не пронизывать домен целиком. Внутри домена классы могут ссылаться друг на друга конкретно; интерфейс появляется там, где домен «касается» инфраструктуры (БД, сеть, файлы, время).
- **Spy vs Mock.** В ДЗ намеренно нет Moq. Ручной `SpyNotifier` фиксирует то, что реально произошло (`Last`), а не то, что вы приказали ему «ожидать». Это ближе к «тестируй поведение» и меньше хрупкости при рефакторинге.
- **`File.AppendAllText` в `FileReceiptSink`.** Это инфраструктура — её место в `Marketok.Infrastructure`, не в домене. В unit-тестах домена её быть не должно; её тестят отдельным integration-тестом на временную папку.
- **Не мокайте value-объекты и DTO.** `Order` и `Receipt` — records, используйте их как есть. Мокать `Receipt` — запах из «частых ошибок» урока.
- **5+ моков = сигнал.** В нашем классе 4 коллаборатора — на грани. Если бы добавился `IAudit`, `ITelemetry`, `ILocalizer` — это уже повод выделить фасад. Следите за цикломатической сложностью и числом коллабораторов.

#### Критерии приёмки

- [ ] Решение собирается `dotnet build` без ошибок и предупреждений на .NET 8 / C# 12.
- [ ] Структура проектов: Domain, Infrastructure, Tests; ссылки направлены внутрь (Infrastructure → Domain, Tests → оба).
- [ ] В `LegacyCheckoutService` оставлен исходный нетестируемый код для сравнения.
- [ ] В новом `CheckoutService` нет ни одного `new` инфраструктурного класса внутри метода.
- [ ] Присутствуют интерфейсы `IClock`, `IPriceRepository`, `INotifier`, `IReceiptSink` на границе домен ↔ инфраструктура.
- [ ] `DiscountPolicy.CalculateDiscount` — статическая чистая функция, принимает `Order` и `DateTime`, не ссылается на `IClock`.
- [ ] `CompositionRoot.Build()` регистрирует все реализации и `CheckoutService` в `ServiceProvider`.
- [ ] Все асинхронные методы принимают `CancellationToken` и пробрасывают его в коллабораторов.
- [ ] Unit-тесты не обращаются к реальному времени, сети, файлам; используют только fakes/stubs/spies, написанные вручную.
- [ ] Есть тесты: декабрь + bulk = 15%, июль без bulk = 0%, январь = 10%, неизвестный SKU, отмена по токену.
- [ ] Тесты проходят параллельно (`dotnet test`) и детерминированно (два прогона подряд дают тот же результат).
- [ ] Чистая функция покрыта отдельными тестами без единого фейка.
- [ ] Утверждения в тестах проверяют итоговое состояние и результат, а не счётчик вызовов или приватные поля.
- [ ] В `README` описан вывод `dotnet test` для легаси (красный/недетерминированный) и для новой архитектуры (зелёный).
- [ ] Код использует C# 12: primary constructors, pattern matching, sealed records, collection expressions там, где уместно.

#### Подсказки (без прямого ответа)

- Подумайте, что именно делает класс «нетестируемым»: какой конкретно оператор в методе закрывает шов? Уберите его — и шов появится сам.
- Время — это тоже «внешний мир». Если чистая функция принимает `DateTime` параметром, то источник времени (`IClock`) живёт в сервисе, а не в политике.
- Спай (spy) отличается от мока тем, что он записывает то, что случилось, а не то, что вы предписали. Что вы хотите зафиксировать — факт вызова или итоговый результат?
- Где должен лежать `File.AppendAllText`? Если в домене — почему это ломает чистоту? Если в инфраструктуре — кто тогда пишет тест на него?
- Задайте себе вопрос: «Если я запущу тест в декабре и в июле, результат будет одинаковым?» Если нет — где спрятан источник недетерминированности?

#### Эталонное решение (разбор)

```csharp
// Marketok.Domain/Seams/Seams.cs
namespace Marketok.Domain.Seams;

public interface IClock
{
    DateTime UtcNow { get; } // шов для времени / seam for time
}

public interface IPriceRepository
{
    decimal GetBasePrice(string sku); // шов для «БД» / seam for "DB"
}

public interface INotifier
{
    Task NotifyReceiptAsync(string sku, decimal total, CancellationToken ct); // шов для сети / seam for network
}

public interface IReceiptSink
{
    void Append(DateTime utc, string sku, decimal total); // шов для лога/файла / seam for log/file
}

// Marketok.Domain/Model/Records.cs
namespace Marketok.Domain.Model;

public sealed record Order(string Sku, int Quantity);
public sealed record Receipt(string Sku, int Quantity, decimal BasePrice, decimal Discount, decimal Total);

// Marketok.Domain/Pure/DiscountPolicy.cs
namespace Marketok.Domain.Pure;

using Marketok.Domain.Model;

public static class DiscountPolicy
{
    // Чистая функция: нет состояния, нет I/O → тестируется без моков.
    // Pure function: no state, no I/O → tested without mocks.
    public static decimal CalculateDiscount(Order order, DateTime utcNow)
    {
        var seasonal = utcNow.Month is 12 or 1 ? 0.10m : 0m;   // C# 12 pattern matching
        var bulk = order.Quantity >= 100 ? 0.05m : 0m;
        return seasonal + bulk;
    }
}

// Marketok.Domain/CheckoutService.cs
namespace Marketok.Domain;

using Marketok.Domain.Model;
using Marketok.Domain.Seams;
using Marketok.Domain.Pure;

// Высокоуровневый доменный сервис зависит только от абстракций (DIP).
// High-level domain service depends only on abstractions (DIP).
public sealed class CheckoutService(
    IClock clock,
    IPriceRepository prices,
    INotifier notifier,
    IReceiptSink sink)
{
    public async Task<Receipt> CheckoutAsync(Order order, CancellationToken ct = default)
    {
        var basePrice = prices.GetBasePrice(order.Sku);
        var discount = DiscountPolicy.CalculateDiscount(order, clock.UtcNow);
        var total = basePrice * order.Quantity * (1m - discount);

        await notifier.NotifyReceiptAsync(order.Sku, total, ct);   // сеть за швом / network behind seam
        sink.Append(clock.UtcNow, order.Sku, total);               // лог за швом / log behind seam

        return new Receipt(order.Sku, order.Quantity, basePrice, discount, total);
    }
}

// Marketok.Infrastructure/Implementations.cs
namespace Marketok.Infrastructure;

using Marketok.Domain.Seams;

internal sealed class SystemClock : IClock
{
    public DateTime UtcNow => DateTime.UtcNow;
}

internal sealed class DbPriceRepository : IPriceRepository
{
    public decimal GetBasePrice(string sku) =>
        sku switch { "A1" => 10m, "B2" => 25m, _ => 1m }; // имитация БД / DB stub
}

internal sealed class HttpNotifier : INotifier
{
    public async Task NotifyReceiptAsync(string sku, decimal total, CancellationToken ct)
    {
        // Реальный HttpClient в продакшене; в unit-тестах домена не вызывается.
        // Real HttpClient in production; not called in domain unit tests.
        await Task.CompletedTask;
    }
}

internal sealed class FileReceiptSink : IReceiptSink
{
    public void Append(DateTime utc, string sku, decimal total) =>
        File.AppendAllText("checkout.log", $"{utc:o} {sku} {total}\n");
}

// Marketok.Infrastructure/CompositionRoot.cs
namespace Marketok.Infrastructure;

using Microsoft.Extensions.DependencyInjection;
using Marketok.Domain;

public static class CompositionRoot
{
    public static ServiceProvider Build() => new ServiceCollection()
        .AddSingleton<IClock, SystemClock>()
        .AddSingleton<IPriceRepository, DbPriceRepository>()
        .AddSingleton<INotifier, HttpNotifier>()
        .AddSingleton<IReceiptSink, FileReceiptSink>()
        .AddSingleton<CheckoutService>()
        .BuildServiceProvider();
}

// Marketok.Tests/CheckoutServiceTests.cs
using Marketok.Domain;
using Marketok.Domain.Model;
using Marketok.Domain.Seams;
using Xunit;

internal sealed class FakeClock(DateTime fixedUtc) : IClock
{
    public DateTime UtcNow => fixedUtc;
}

internal sealed class StubPrices : IPriceRepository
{
    public decimal GetBasePrice(string sku) => 10m; // детерминированный ответ / deterministic
}

internal sealed class SpyNotifier : INotifier
{
    public (string Sku, decimal Total)? Last;
    public Task NotifyReceiptAsync(string sku, decimal total, CancellationToken ct)
    {
        Last = (sku, total);
        return Task.CompletedTask;
    }
}

internal sealed class SpySink : IReceiptSink
{
    public List<(DateTime Utc, string Sku, decimal Total)> Entries { get; } = [];
    public void Append(DateTime utc, string sku, decimal total) => Entries.Add((utc, sku, total));
}

public class CheckoutServiceTests
{
    private static DateTime Dec(int day) => new(2024, 12, day, 0, 0, 0, DateTimeKind.Utc);
    private static DateTime Jul(int day) => new(2024, 7, day, 0, 0, 0, DateTimeKind.Utc);
    private static DateTime Jan(int day) => new(2024, 1, day, 0, 0, 0, DateTimeKind.Utc);

    [Fact]
    public async Task December_bulk_order_gets_combined_discount()
    {
        var clock = new FakeClock(Dec(15));
        var notifier = new SpyNotifier();
        var sink = new SpySink();
        var sut = new CheckoutService(clock, new StubPrices(), notifier, sink);

        var receipt = await sut.CheckoutAsync(new Order("A1", 100));

        Assert.Equal(10m, receipt.BasePrice);
        Assert.Equal(0.15m, receipt.Discount);     // 10% seasonal + 5% bulk
        Assert.Equal(850m, receipt.Total);         // 10 * 100 * 0.85
        Assert.Equal(("A1", 850m), notifier.Last);
        Assert.Single(sink.Entries);
    }

    [Fact]
    public async Task July_small_order_has_no_discount()
    {
        var sut = new CheckoutService(new FakeClock(Jul(1)), new StubPrices(), new SpyNotifier(), new SpySink());
        var receipt = await sut.CheckoutAsync(new Order("A1", 1));
        Assert.Equal(0m, receipt.Discount);
        Assert.Equal(10m, receipt.Total);
    }

    [Fact]
    public async Task January_gives_seasonal_only()
    {
        var sut = new CheckoutService(new FakeClock(Jan(10)), new StubPrices(), new SpyNotifier(), new SpySink());
        var receipt = await sut.CheckoutAsync(new Order("A1", 1));
        Assert.Equal(0.10m, receipt.Discount);
        Assert.Equal(9m, receipt.Total);           // 10 * 1 * 0.9
    }

    [Fact]
    public async Task Unknown_sku_uses_default_price()
    {
        var sut = new CheckoutService(new FakeClock(Jul(1)), new StubPrices(), new SpyNotifier(), new SpySink());
        var receipt = await sut.CheckoutAsync(new Order("ZZ", 1));
        Assert.Equal(10m, receipt.BasePrice);      // stub всегда возвращает 10
        Assert.Equal(10m, receipt.Total);
    }

    [Fact]
    public async Task Cancellation_short_circuits_notification()
    {
        var cts = new CancellationTokenSource();
        cts.Cancel();
        var sut = new CheckoutService(new FakeClock(Dec(1)), new StubPrices(), new SpyNotifier(), new SpySink());
        await Assert.ThrowsAnyAsync<OperationCanceledException>(
            () => sut.CheckoutAsync(new Order("A1", 1), cts.Token));
    }
}
```

Разбор по строкам. `IClock`, `IPriceRepository`, `INotifier`, `IReceiptSink` — это швы (seams) из урока: абстракции, через которые домен общается с «грязной» зоной (время, БД, сеть, файлы). Без них класс не имел бы точки, в которую можно вставить фейк. `DiscountPolicy.CalculateDiscount` — чистая функция: она принимает `Order` и `DateTime` значениями и не ссылается на `IClock`; это намеренно, чтобы функция оставалась в «чистой зоне» и тестировалась без DI и без моков (как требует best practice урока). Здесь же используется C# 12 pattern matching `is 12 or 1` вместо `month == 12 || month == 1` — короче и выразительнее. `CheckoutService` объявлен как `sealed class` с primary constructor C# 12 — зависимости видны прямо в сигнатуре и не могут быть «забыты». Внутри `CheckoutAsync` нет ни одного `new` инфраструктурного класса: это главное архитектурное отличие от легаси. Коллабораторы приходят через конструктор, вызовы идут через интерфейсы — значит, в тесте их можно подменить. Используется `await` с `CancellationToken` — это устраняет синхронную блокировку `.Wait()` из легаси и даёт возможность протестировать отмену.

Инфраструктурные реализации (`SystemClock`, `DbPriceRepository`, `HttpNotifier`, `FileReceiptSink`) лежат в `Marketok.Infrastructure` — они не «пачкают» домен. `CompositionRoot.Build()` зашивает швы один раз на границе системы: в продакшене — реальные реализации, в тестах — фейки. Это и есть DIP из урока: оба слоя зависят от абстракций, а направление зависимостей — внутрь, к домену (Clean Architecture).

Тесты используют только ручные fakes/stubs/spies (без Moq), чтобы студент физически почувствовал швы. `FakeClock` фиксирует время декабря — тест не зависит от реального месяца. `StubPrices` возвращает константу — нет «БД». `SpyNotifier` и `SpySink` записывают произошедшее, а не предписанное: это соответствует правилу «тестируй поведение, а не реализацию». Утверждения идут на итоговый `Receipt`, на `Last` спая и на `Entries` — а не на счётчик вызовов или приватные поля. Тест на `CancellationToken` доказывает, что токен реально пробрасывается в коллабораторов (иначе отмена прошла бы «вхолостую»), что закрывает частую ошибку «async без отмены».

Итог: класс, который в шаге 3 нельзя было протестировать, теперь проверяется пятью детерминированными тестами, идущими параллельно без инфраструктуры. Тестируемость пришла не от «дописанных тестов», а от перестроенной архитектуры — ровно тезис урока M15-L09 и мост к M16.

#### Задания на углубление (бонус)

1. **Измерьте тестируемость количественно.** Посчитайте цикломатическую сложность `LegacyCheckoutService.Checkout` и нового `CheckoutService.CheckoutAsync` (например, через `dotnet format --report` или ручной подсчёт ветвлений). Сравните число коллабораторов. Опишите, как изменились индикаторы тестируемости.
2. **Integration-тест для инфраструктуры.** Добавьте отдельный тестовый проект `Marketok.IntegrationTests` и напишите тест на `FileReceiptSink` с реальной временной папкой (`Path.GetTempPath`) и на `HttpNotifier` через `IHttpClientFactory` с делегирующим обработчиком. Покажите, где проходит граница между unit и integration.
3. **Превращение в M16.** Выделите из `CheckoutService` два обработчика CQRS: `PriceOrderCommand`/`PriceOrderHandler` и `NotifyReceiptCommand`/`NotifyReceiptHandler`. Опишите, как это меняет SRP и тестируемость, и почему это уже шаг в сторону M16.
4. **Порты и адаптеры.** Переименуйте сборку `Marketok.Infrastructure` в `Marketok.Adapters` и явно разделите «порт» (интерфейс в домене) и «адаптер» (реализацию). Опишите, как это соотносится с гексагональной архитектурой из预告ы M16.

---

## Statement in English / Постановка на английском

#### Context & motivation

The startup "MarketOk" has a legacy service, `CheckoutService`, that does everything at once: it reads prices from a "database" (in practice a hardcoded `switch`), computes calendar- and volume-based discounts, sends an HTTP notification and writes to a log file. The team tried to cover it with unit tests, but the very first test needed to stub the real clock, the network call and the logger — and stalled. The developers wrote "hard to test", closed the ticket and went on to write tests only for pure utilities. Meanwhile discount bugs came back every quarter, because nobody could reproduce December's behaviour in July.

Lesson M15-L09 explains why this happens: a class without seams cannot be verified in isolation. `new DateTime()` and `new HttpClient()` directly inside a method are the absence of a seam, and mocks will not help here because there is nothing to substitute: the object is created inside and immediately consumed. The pain is not in the tests, it is in the architecture. To cure the service you do not "add more tests"; you rebuild the dependencies: move time behind `IClock`, prices behind `IPriceRepository`, notifications behind `INotifier`, logging behind `IReceiptSink`, and the discount logic into a pure function with no state and no I/O. Then the domain is verifiable without a database, without a network and without real time, while the infrastructure is exercised separately as integration tests.

This is exactly testability as an architectural property: the ability of components to be verified in isolation, predictably and quickly. In this homework you will walk that path yourself — from the legacy class to a composed architecture with seams — and write tests that prove the improvement. Along the way you will feel the boundary of M16: where "just DI" ends and SOLID, Clean Architecture and DDD begin.

#### What to do step by step

1. **Create the solution and projects.** In a folder `M15-L09-HW/` run:
   ```
   dotnet new sln -n Marketok
   dotnet new classlib -n Marketok.Domain -f net8.0
   dotnet new classlib -n Marketok.Infrastructure -f net8.0
   dotnet new xunit -n Marketok.Tests -f net8.0
   dotnet sln add **/*.csproj
   dotnet add Marketok.Infrastructure reference Marketok.Domain
   dotnet add Marketok.Tests reference Marketok.Domain Marketok.Infrastructure
   dotnet add Marketok.Tests package Microsoft.Extensions.DependencyInjection
   ```
   Expected: four projects with references pointing inward (Infrastructure → Domain, Tests → both). This is the Clean Architecture direction of dependencies — outer layers depend on inner ones.

2. **Bring the legacy code in.** Put the following class into `Marketok.Domain/Legacy/CheckoutService.cs` (the deliberate, untestable starting point):
   ```csharp
   public class LegacyCheckoutService
   {
       public Receipt Checkout(Order order)
       {
           var basePrice = order.Sku switch { "A1" => 10m, "B2" => 25m, _ => 1m };
           var now = DateTime.Now;
           var seasonal = now.Month is 12 or 1 ? 0.10m : 0m;
           var bulk = order.Quantity >= 100 ? 0.05m : 0m;
           var discount = seasonal + bulk;
           var total = basePrice * order.Quantity * (1m - discount);

           using var http = new HttpClient();
           http.PostAsJsonAsync("https://notify.example.com/receipt", new { order.Sku, total }).Wait();

           File.AppendAllText("checkout.log", $"{now:o} {order.Sku} {total}\n");
           return new Receipt(order.Sku, order.Quantity, basePrice, discount, total);
       }
   }
   public sealed record Order(string Sku, int Quantity);
   public sealed record Receipt(string Sku, int Quantity, decimal BasePrice, decimal Discount, decimal Total);
   ```
   Build with `dotnet build`. It should compile (add `using System.Net.Http;` / `using System.Text.Json;` if needed, and if `PostAsJsonAsync` is unavailable, replace it with manual JSON serialisation — the point is the real network call inside the method).

3. **Demonstrate that the test is broken.** In `Marketok.Tests` try to write a test for the December discount. You will discover that the time cannot be pinned (`DateTime.Now` is created inside), the HTTP call cannot be avoided (it really goes to the network) and the file log cannot be switched off. Capture the `dotnet test` output — it is red or non-deterministic. This is the "pain in the bone".

4. **Introduce seams.** In `Marketok.Domain/Seams/` create the interfaces:
   ```csharp
   public interface IClock { DateTime UtcNow { get; } }
   public interface IPriceRepository { decimal GetBasePrice(string sku); }
   public interface INotifier { Task NotifyReceiptAsync(string sku, decimal total, CancellationToken ct); }
   public interface IReceiptSink { void Append(DateTime utc, string sku, decimal total); }
   ```
   Note that the seams sit on the boundary "domain ↔ infrastructure". Inside the domain you keep concrete types (`Order`, `Receipt`); only abstractions poke outward.

5. **Extract the pure function.** In `Marketok.Domain/Pure/DiscountPolicy.cs`:
   ```csharp
   public static class DiscountPolicy
   {
       public static decimal CalculateDiscount(Order order, DateTime utcNow)
       {
           var seasonal = utcNow.Month is 12 or 1 ? 0.10m : 0m;
           var bulk = order.Quantity >= 100 ? 0.05m : 0m;
           return seasonal + bulk;
       }
   }
   ```
   No state, no I/O — this is the "clean zone". Note the C# 12 pattern matching `is 12 or 1`.

6. **Rewrite the service.** In `Marketok.Domain/CheckoutService.cs` create a new class that takes every dependency through the constructor (C# 12 primary constructor):
   ```csharp
   public sealed class CheckoutService(
       IClock clock, IPriceRepository prices, INotifier notifier, IReceiptSink sink)
   {
       public async Task<Receipt> CheckoutAsync(Order order, CancellationToken ct = default)
       {
           var basePrice = prices.GetBasePrice(order.Sku);
           var discount = DiscountPolicy.CalculateDiscount(order, clock.UtcNow);
           var total = basePrice * order.Quantity * (1m - discount);
           await notifier.NotifyReceiptAsync(order.Sku, total, ct);
           sink.Append(clock.UtcNow, order.Sku, total);
           return new Receipt(order.Sku, order.Quantity, basePrice, discount, total);
       }
   }
   ```
   No more `new HttpClient()`, `DateTime.Now` or `File.AppendAllText` inside the method. The seams exist.

7. **Implement the infrastructure.** In `Marketok.Infrastructure/`:
   - `SystemClock : IClock` → `DateTime.UtcNow`;
   - `DbPriceRepository : IPriceRepository` → fake DB lookup (hardcoded `switch`);
   - `HttpNotifier : INotifier` → real `HttpClient` (a skeleton is enough for the HW);
   - `FileReceiptSink : IReceiptSink` → `File.AppendAllText`.

8. **Build the CompositionRoot.** In `Marketok.Infrastructure/CompositionRoot.cs`:
   ```csharp
   public static class CompositionRoot
   {
       public static ServiceProvider Build() => new ServiceCollection()
           .AddSingleton<IClock, SystemClock>()
           .AddSingleton<IPriceRepository, DbPriceRepository>()
           .AddSingleton<INotifier, HttpNotifier>()
           .AddSingleton<IReceiptSink, FileReceiptSink>()
           .AddSingleton<CheckoutService>()
           .BuildServiceProvider();
   }
   ```
   The seam is stitched exactly once, at the system boundary.

9. **Write tests.** In `Marketok.Tests` use hand-written fakes:
   ```csharp
   internal sealed class FakeClock(DateTime fixedUtc) : IClock { public DateTime UtcNow => fixedUtc; }
   internal sealed class StubPrices : IPriceRepository { public decimal GetBasePrice(string sku) => 10m; }
   internal sealed class SpyNotifier : INotifier {
       public (string Sku, decimal Total)? Last;
       public Task NotifyReceiptAsync(string sku, decimal total, CancellationToken ct) { Last = (sku, total); return Task.CompletedTask; } }
   internal sealed class SpySink : IReceiptSink {
       public List<(DateTime, string, decimal)> Entries = new();
       public void Append(DateTime utc, string sku, decimal total) => Entries.Add((utc, sku, total)); }
   ```
   Cover: December + bulk = 15%, July without bulk = 0%, January = 10%, unknown SKU with default price, cancellation via `CancellationToken`. All tests run in parallel, with zero network/file access.

10. **Run and compare.** `dotnet test -v n`. Expect green and deterministic. Compare with the output of step 3 — tests no longer depend on the real clock or the network.

#### Requirements

- The solution targets C# 12 / .NET 8 (`net8.0`). Use top-level statements in `Program.cs` (if you add a demo host), primary constructors, pattern matching (`is 12 or 1`, `switch` expressions), sealed records for DTOs and `CancellationToken` in every async method.
- Structure: `Marketok.Domain` (domain + seams + pure functions), `Marketok.Infrastructure` (implementations + CompositionRoot), `Marketok.Tests` (only fakes/stubs/spies, no Moq — to feel the seams by hand). Project references point inward: Infrastructure → Domain, Tests → both.
- No unit test may touch real time, the file system or the network. All sources of non-determinism are hidden behind interfaces and replaced with fakes.
- The pure function `DiscountPolicy.CalculateDiscount` must accept only data (`Order`, `DateTime`). No `IClock` inside the pure function — time arrives as a parameter so the function stays pure.
- `CheckoutService` must contain no `new` of an infrastructure class inside the method. All collaborators arrive via the constructor. Four fakes in a test is acceptable for this coordinator class; if it were seven or more, that would be an SRP smell.
- Tests must assert behaviour, not implementation: do not reach into private fields, do not count call order, do not mock mappers or DTOs. Assert on the final `Receipt`, on the spy's `Last` and on the `Entries` list.

#### Pitfalls

- **`DateTime.Now` vs `DateTime.UtcNow` vs `IClock`.** The legacy uses `DateTime.Now` — doubly bad: a local time zone makes a test non-deterministic on a CI runner in another region. The `IClock` seam should return `UtcNow`, and the pure function compares `Month` against a UTC date. In tests pass `new DateTime(2024,12,15,0,0,0,DateTimeKind.Utc)` — without `DateTimeKind.Utc` conversions may surprise you.
- **`new HttpClient()` in a method is not only a testability issue, it is a performance one.** `HttpClient` is designed for reuse; frequent creation exhausts sockets. The right shape is `IHttpClientFactory` or a singleton `HttpClient` wrapped in `INotifier`. The seam fixes both problems at once.
- **`Task.Wait()` in the legacy is a synchronous block over async.** In the async path use `await` and propagate `CancellationToken`. Avoid `.Result` and `.Wait()` — they deadlock in UI/ASP.NET contexts and drop the cancellation token.
- **The pure function must not "know" about `IClock`.** If you leak `IClock` into `DiscountPolicy`, the function stops being pure (a dependency on an interface is still a dependency on the time state). Pass `DateTime` by value — then you can test it with no DI container at all.
- **Where seams live.** Seams belong on layer boundaries, not woven all through the domain. Inside the domain, classes may reference each other concretely; an interface appears where the domain "touches" infrastructure (DB, network, files, time).
- **Spy vs Mock.** This homework deliberately omits Moq. A hand-written `SpyNotifier` records what actually happened (`Last`), not what you prescribed it to "expect". This is closer to "test behaviour" and less brittle under refactoring.
- **`File.AppendAllText` in `FileReceiptSink`.** This is infrastructure — it belongs in `Marketok.Infrastructure`, not in the domain. It must not appear in domain unit tests; test it separately with an integration test against a temp folder.
- **Do not mock value objects or DTOs.** `Order` and `Receipt` are records — use them as-is. Mocking `Receipt` is a smell from the lesson's "common mistakes".
- **Five or more mocks is a signal.** Our class has four collaborators — on the edge. If you added `IAudit`, `ITelemetry`, `ILocalizer`, that would already justify a facade. Watch cyclomatic complexity and the collaborator count.

#### Acceptance criteria

- [ ] The solution builds with `dotnet build` without errors or warnings on .NET 8 / C# 12.
- [ ] Project structure: Domain, Infrastructure, Tests; references point inward (Infrastructure → Domain, Tests → both).
- [ ] `LegacyCheckoutService` keeps the original untestable code for comparison.
- [ ] The new `CheckoutService` contains no `new` of an infrastructure class inside the method.
- [ ] Interfaces `IClock`, `IPriceRepository`, `INotifier`, `IReceiptSink` are present on the domain ↔ infrastructure boundary.
- [ ] `DiscountPolicy.CalculateDiscount` is a static pure function taking `Order` and `DateTime`, with no reference to `IClock`.
- [ ] `CompositionRoot.Build()` registers all implementations and `CheckoutService` in a `ServiceProvider`.
- [ ] All async methods accept `CancellationToken` and propagate it to collaborators.
- [ ] Unit tests do not touch real time, network or files; they use only hand-written fakes/stubs/spies.
- [ ] Tests exist for: December + bulk = 15%, July without bulk = 0%, January = 10%, unknown SKU, cancellation by token.
- [ ] Tests run in parallel (`dotnet test`) and deterministically (two consecutive runs produce the same result).
- [ ] The pure function is covered by separate tests with no fakes at all.
- [ ] Test assertions check final state and result, not call counts or private fields.
- [ ] The `README` describes the `dotnet test` output for the legacy (red/non-deterministic) and for the new architecture (green).
- [ ] The code uses C# 12: primary constructors, pattern matching, sealed records, collection expressions where appropriate.

#### Hints (without the direct answer)

- Ask what exactly makes the class untestable: which single operator in the method closes the seam? Remove it and the seam appears on its own.
- Time is part of the "outside world". If the pure function takes `DateTime` as a parameter, the source of time (`IClock`) lives in the service, not in the policy.
- A spy differs from a mock in that it records what happened, not what you prescribed. What do you want to capture — the fact of a call or the final result?
- Where should `File.AppendAllText` live? If in the domain, why does it break purity? If in infrastructure, who writes a test for it?
- Ask yourself: "If I run the test in December and in July, will the result be the same?" If not — where is the source of non-determinism hidden?

#### Reference solution walk-through

```csharp
// Marketok.Domain/Seams/Seams.cs
namespace Marketok.Domain.Seams;

public interface IClock
{
    DateTime UtcNow { get; } // seam for time
}

public interface IPriceRepository
{
    decimal GetBasePrice(string sku); // seam for "DB"
}

public interface INotifier
{
    Task NotifyReceiptAsync(string sku, decimal total, CancellationToken ct); // seam for network
}

public interface IReceiptSink
{
    void Append(DateTime utc, string sku, decimal total); // seam for log/file
}

// Marketok.Domain/Model/Records.cs
namespace Marketok.Domain.Model;

public sealed record Order(string Sku, int Quantity);
public sealed record Receipt(string Sku, int Quantity, decimal BasePrice, decimal Discount, decimal Total);

// Marketok.Domain/Pure/DiscountPolicy.cs
namespace Marketok.Domain.Pure;

using Marketok.Domain.Model;

public static class DiscountPolicy
{
    // Pure function: no state, no I/O → tested without mocks.
    public static decimal CalculateDiscount(Order order, DateTime utcNow)
    {
        var seasonal = utcNow.Month is 12 or 1 ? 0.10m : 0m;   // C# 12 pattern matching
        var bulk = order.Quantity >= 100 ? 0.05m : 0m;
        return seasonal + bulk;
    }
}

// Marketok.Domain/CheckoutService.cs
namespace Marketok.Domain;

using Marketok.Domain.Model;
using Marketok.Domain.Seams;
using Marketok.Domain.Pure;

// High-level domain service depends only on abstractions (DIP).
public sealed class CheckoutService(
    IClock clock,
    IPriceRepository prices,
    INotifier notifier,
    IReceiptSink sink)
{
    public async Task<Receipt> CheckoutAsync(Order order, CancellationToken ct = default)
    {
        var basePrice = prices.GetBasePrice(order.Sku);
        var discount = DiscountPolicy.CalculateDiscount(order, clock.UtcNow);
        var total = basePrice * order.Quantity * (1m - discount);

        await notifier.NotifyReceiptAsync(order.Sku, total, ct);   // network behind a seam
        sink.Append(clock.UtcNow, order.Sku, total);               // log behind a seam

        return new Receipt(order.Sku, order.Quantity, basePrice, discount, total);
    }
}

// Marketok.Infrastructure/Implementations.cs
namespace Marketok.Infrastructure;

using Marketok.Domain.Seams;

internal sealed class SystemClock : IClock
{
    public DateTime UtcNow => DateTime.UtcNow;
}

internal sealed class DbPriceRepository : IPriceRepository
{
    public decimal GetBasePrice(string sku) =>
        sku switch { "A1" => 10m, "B2" => 25m, _ => 1m }; // DB stub
}

internal sealed class HttpNotifier : INotifier
{
    public async Task NotifyReceiptAsync(string sku, decimal total, CancellationToken ct)
    {
        // Real HttpClient in production; not called from domain unit tests.
        await Task.CompletedTask;
    }
}

internal sealed class FileReceiptSink : IReceiptSink
{
    public void Append(DateTime utc, string sku, decimal total) =>
        File.AppendAllText("checkout.log", $"{utc:o} {sku} {total}\n");
}

// Marketok.Infrastructure/CompositionRoot.cs
namespace Marketok.Infrastructure;

using Microsoft.Extensions.DependencyInjection;
using Marketok.Domain;

public static class CompositionRoot
{
    public static ServiceProvider Build() => new ServiceCollection()
        .AddSingleton<IClock, SystemClock>()
        .AddSingleton<IPriceRepository, DbPriceRepository>()
        .AddSingleton<INotifier, HttpNotifier>()
        .AddSingleton<IReceiptSink, FileReceiptSink>()
        .AddSingleton<CheckoutService>()
        .BuildServiceProvider();
}

// Marketok.Tests/CheckoutServiceTests.cs
using Marketok.Domain;
using Marketok.Domain.Model;
using Marketok.Domain.Seams;
using Xunit;

internal sealed class FakeClock(DateTime fixedUtc) : IClock
{
    public DateTime UtcNow => fixedUtc;
}

internal sealed class StubPrices : IPriceRepository
{
    public decimal GetBasePrice(string sku) => 10m; // deterministic
}

internal sealed class SpyNotifier : INotifier
{
    public (string Sku, decimal Total)? Last;
    public Task NotifyReceiptAsync(string sku, decimal total, CancellationToken ct)
    {
        Last = (sku, total);
        return Task.CompletedTask;
    }
}

internal sealed class SpySink : IReceiptSink
{
    public List<(DateTime Utc, string Sku, decimal Total)> Entries { get; } = [];
    public void Append(DateTime utc, string sku, decimal total) => Entries.Add((utc, sku, total));
}

public class CheckoutServiceTests
{
    private static DateTime Dec(int day) => new(2024, 12, day, 0, 0, 0, DateTimeKind.Utc);
    private static DateTime Jul(int day) => new(2024, 7, day, 0, 0, 0, DateTimeKind.Utc);
    private static DateTime Jan(int day) => new(2024, 1, day, 0, 0, 0, DateTimeKind.Utc);

    [Fact]
    public async Task December_bulk_order_gets_combined_discount()
    {
        var clock = new FakeClock(Dec(15));
        var notifier = new SpyNotifier();
        var sink = new SpySink();
        var sut = new CheckoutService(clock, new StubPrices(), notifier, sink);

        var receipt = await sut.CheckoutAsync(new Order("A1", 100));

        Assert.Equal(10m, receipt.BasePrice);
        Assert.Equal(0.15m, receipt.Discount);     // 10% seasonal + 5% bulk
        Assert.Equal(850m, receipt.Total);         // 10 * 100 * 0.85
        Assert.Equal(("A1", 850m), notifier.Last);
        Assert.Single(sink.Entries);
    }

    [Fact]
    public async Task July_small_order_has_no_discount()
    {
        var sut = new CheckoutService(new FakeClock(Jul(1)), new StubPrices(), new SpyNotifier(), new SpySink());
        var receipt = await sut.CheckoutAsync(new Order("A1", 1));
        Assert.Equal(0m, receipt.Discount);
        Assert.Equal(10m, receipt.Total);
    }

    [Fact]
    public async Task January_gives_seasonal_only()
    {
        var sut = new CheckoutService(new FakeClock(Jan(10)), new StubPrices(), new SpyNotifier(), new SpySink());
        var receipt = await sut.CheckoutAsync(new Order("A1", 1));
        Assert.Equal(0.10m, receipt.Discount);
        Assert.Equal(9m, receipt.Total);           // 10 * 1 * 0.9
    }

    [Fact]
    public async Task Unknown_sku_uses_default_price()
    {
        var sut = new CheckoutService(new FakeClock(Jul(1)), new StubPrices(), new SpyNotifier(), new SpySink());
        var receipt = await sut.CheckoutAsync(new Order("ZZ", 1));
        Assert.Equal(10m, receipt.BasePrice);      // stub always returns 10
        Assert.Equal(10m, receipt.Total);
    }

    [Fact]
    public async Task Cancellation_short_circuits_notification()
    {
        var cts = new CancellationTokenSource();
        cts.Cancel();
        var sut = new CheckoutService(new FakeClock(Dec(1)), new StubPrices(), new SpyNotifier(), new SpySink());
        await Assert.ThrowsAnyAsync<OperationCanceledException>(
            () => sut.CheckoutAsync(new Order("A1", 1), cts.Token));
    }
}
```

Walk-through, line by line. `IClock`, `IPriceRepository`, `INotifier`, `IReceiptSink` are the seams from the lesson: the abstractions through which the domain talks to the "dirty" zone (time, DB, network, files). Without them, the class has no point at which a fake can be inserted. `DiscountPolicy.CalculateDiscount` is a pure function: it takes `Order` and `DateTime` by value and never references `IClock`; this is deliberate so the function stays in the "clean zone" and is testable with no DI container and no mocks (the lesson's best practice). It also uses C# 12 pattern matching `is 12 or 1` instead of `month == 12 || month == 1` — shorter and clearer. `CheckoutService` is declared `sealed` with a C# 12 primary constructor — dependencies are visible right in the signature and cannot be "forgotten". Inside `CheckoutAsync` there is no `new` of any infrastructure class: this is the key architectural difference from the legacy. Collaborators arrive through the constructor, calls go through interfaces — so in a test they can be substituted. `await` with `CancellationToken` removes the synchronous `.Wait()` block from the legacy and lets cancellation be tested.

The infrastructure implementations (`SystemClock`, `DbPriceRepository`, `HttpNotifier`, `FileReceiptSink`) live in `Marketok.Infrastructure` — they do not "dirty" the domain. `CompositionRoot.Build()` stitches the seams once at the system boundary: real implementations in production, fakes in tests. This is the DIP from the lesson: both layers depend on abstractions, and the direction of dependencies points inward, toward the domain (Clean Architecture).

Tests use only hand-written fakes/stubs/spies (no Moq) so the student physically feels the seams. `FakeClock` fixes December — the test does not depend on the real month. `StubPrices` returns a constant — there is no "DB". `SpyNotifier` and `SpySink` record what happened, not what was prescribed: this matches the rule "test behaviour, not implementation". Assertions target the final `Receipt`, the spy's `Last` and the `Entries` list — never call counts or private fields. The `CancellationToken` test proves the token really propagates to collaborators (otherwise cancellation would be a no-op), closing the common "async without cancellation" mistake.

The bottom line: the class that in step 3 could not be tested is now covered by five deterministic tests that run in parallel with no infrastructure. Testability arrived not from "extra tests" but from a restructured architecture — exactly the thesis of lesson M15-L09 and a bridge to M16.

#### Going deeper (bonus)

1. **Quantify testability.** Compute the cyclomatic complexity of `LegacyCheckoutService.Checkout` and of the new `CheckoutService.CheckoutAsync` (e.g. via `dotnet format --report` or a manual branch count). Compare the collaborator counts. Describe how the testability indicators changed.
2. **An integration test for infrastructure.** Add a separate project `Marketok.IntegrationTests` and write a test for `FileReceiptSink` against a real temp folder (`Path.GetTempPath`) and for `HttpNotifier` through `IHttpClientFactory` with a delegating handler. Show where the unit/integration boundary lies.
3. **Turning toward M16.** Split `CheckoutService` into two CQRS handlers: `PriceOrderCommand`/`PriceOrderHandler` and `NotifyReceiptCommand`/`NotifyReceiptHandler`. Explain how this changes SRP and testability, and why this is already a step toward M16.
4. **Ports and adapters.** Rename the `Marketok.Infrastructure` assembly to `Marketok.Adapters` and explicitly separate the "port" (interface in the domain) from the "adapter" (implementation). Explain how this aligns with hexagonal architecture from the M16 preview.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Решение собирается на .NET 8 / C# 12 без предупреждений.
- [ ] (RU) Структура проектов: Domain, Infrastructure, Tests; ссылки внутрь.
- [ ] (RU) Легаси-класс оставлен для сравнения; новый сервис без `new` в методах.
- [ ] (RU) Швы `IClock`/`IPriceRepository`/`INotifier`/`IReceiptSink` на границе слоёв.
- [ ] (RU) `DiscountPolicy` — чистая функция без `IClock`.
- [ ] (RU) `CompositionRoot.Build()` собирает все реализации.
- [ ] (RU) Все async-методы принимают `CancellationToken`.
- [ ] (RU) Тесты детерминированы, параллельны, без сети/файлов/реального времени.
- [ ] (RU) Покрыты 5 сценариев: декабрь+bulk, июль, январь, неизвестный SKU, отмена.
- [ ] (RU) Утверждения на поведение, не на счётчик вызовов.
- [ ] (EN) Solution builds on .NET 8 / C# 12 with no warnings.
- [ ] (EN) Project structure: Domain, Infrastructure, Tests; references point inward.
- [ ] (EN) Legacy class kept for comparison; new service has no `new` in methods.
- [ ] (EN) Seams `IClock`/`IPriceRepository`/`INotifier`/`IReceiptSink` on the layer boundary.
- [ ] (EN) `DiscountPolicy` is a pure function with no `IClock`.
- [ ] (EN) `CompositionRoot.Build()` wires all implementations.
- [ ] (EN) All async methods accept `CancellationToken`.
- [ ] (EN) Tests are deterministic, parallel, with no network/files/real time.
- [ ] (EN) Five scenarios covered: December+bulk, July, January, unknown SKU, cancellation.
- [ ] (EN) Assertions are on behaviour, not on call counts.

#### Ресурсы / Resources
- [Microsoft Learn — Dependency injection in .NET](https://learn.microsoft.com/dotnet/core/extensions/dependency-injection)
- [Microsoft Learn — .NET Microservices: DDD/CQRS patterns](https://learn.microsoft.com/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/)
- [Microsoft Learn — Dependency inversion principle & testability](https://learn.microsoft.com/dotnet/architecture/microservices/architect-microservice-container-applications/)
- [xunit.net — Getting started with xUnit for .NET](https://xunit.net/docs/getting-started/netcore/cmdline)
- [Martin Fowler — TestDouble (stubs, spies, mocks)](https://martinfowler.com/bliki/TestDouble.html)
- [Microsoft Learn — Use IHttpClientFactory to implement resilient HTTP requests](https://learn.microsoft.com/dotnet/core/extensions/http-client-factory)
