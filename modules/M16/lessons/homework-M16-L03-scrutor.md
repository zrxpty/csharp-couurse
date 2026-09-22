---
[← К уроку M16-L03](lesson-M16-L03-scrutor.md) | [⬆ К модулю M16](../README.md) | [Следующее ДЗ →](homework-M16-L04-layered-clean-architecture.md)
---

### Домашнее задание M16-L03: Scrutor: assembly scan, decorators, adapters / Homework M16-L03: Scrutor: assembly scan, decorators, adapters

**Урок / Lesson:** M16-L03
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться применять библиотеку Scrutor для конвенциональной регистрации сервисов через assembly scanning, строить цепочки декораторов через `Decorate<T,TDec>()` без ручного управления «внутренними» экземплярами и изолировать сторонние SDK за адаптерами через `RegisterAdapter<TImpl,TAdapter>()`, а также доказывать корректность DI-сборки unit-тестом. (EN) Learn to use Scrutor for convention-based registration via assembly scanning, build decorator chains through `Decorate<T,TDec>()` without hand-rolled inner-instance plumbing, isolate third-party SDKs behind adapters via `RegisterAdapter<TImpl,TAdapter>()`, and prove the DI composition is correct with a unit test.

#### Связь с уроком / Connection to the lesson
(RU) Урок вводит три ключевые возможности Scrutor — сканирование сборок по конвенции, декорирование сервисов и регистрацию адаптеров — и приводит рабочий пример с `OrderService`, `LoggingOrderDecorator` и `BillingAdapter`. Это домашнее задание расширяет пример до полноценного мини-проекта «OrderProcessing» с тремя декораторами в цепочке и адаптером для внешнего платёжного клиента, чтобы каждый механизм был опробован на практике и покрыт тестом.)
(EN) The lesson introduces Scrutor's three core capabilities — assembly scanning by convention, service decoration, and adapter registration — with a working example of `OrderService`, `LoggingOrderDecorator`, and `BillingAdapter`. This homework scales that example up into a full mini-project "OrderProcessing" with a three-decorator chain and an adapter for an external payment client, so every mechanism is exercised hands-on and covered by a test.)

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

В больших .NET-приложениях слой сервисов разрастается очень быстро: заказы, инвентарь, уведомления, биллинг, отчёты, аудит — у каждого свой интерфейс и своя реализация. Ручная регистрация каждого сервиса в `Program.cs` через `services.AddScoped<IFoo, Foo>()` через полгода превращается в полотно на несколько сотен строк, которое страшно ревьюить: при переименовании интерфейса легко забыть обновить строку регистрации, при добавлении нового сервиса легко забыть зарегистрировать его вовсе, а дубликаты регистраторов могут молча ломать разрешение. Библиотека Scrutor закрывает эту боль через конвенциональную регистрацию по маркерному интерфейсу: достаточно пометить все «настоящие» сервисы интерфейсом `IServiceMarker`, и одна конструкция `Scan(...)` зарегистрирует их все с единым временем жизни и явной стратегией обработки дубликатов.

Параллельно Scrutor элегантно решает вторую боль — декораторы. Когда нужно добавить логирование, кэш и retry сразу к нескольким сервисам, ручная сборка «матрёшки» превращается в ад из keyed-сервисов, фабрик и `GetService<IOrderService>("inner")`. Метод `Decorate<TInterface, TDecorator>()` позволяет контейнеру самому управлять вложением: каждый вызов `Decorate` оборачивает текущую регистрацию `TInterface`, выравнивает время жизни автоматически и позволяет собирать цепочки любой длины. Третья фича — адаптеры через `RegisterAdapter<TImplementation, TAdapter>()` — изолирует сторонние SDK (платёжные шлюзы, SMS-провайдеры, облачные клиенты) за вашим собственным интерфейсом, чтобы домен не зависел от внешних пакетов, их типов и версий.

В этом задании вы соберёте мини-проект «OrderProcessing» с тремя доменными сервисами, тремя декораторами в одной цепочке и одним адаптером для внешнего платёжного клиента, зарегистрируете всё через Scrutor и докажете корректность сборки unit-тестом, который подменяет `ILogger` и проверяет, что сообщения декораторов действительно появляются в логе.

#### Что нужно сделать (пошагово)

1. **Создайте решение и шесть проектов.** Из корня репозитория выполните команды по очереди, проверяя, что каждая завершается без ошибок:
   ```
   dotnet new sln -n OrderProcessing
   dotnet new classlib -n OrderProcessing.Domain   -o src/OrderProcessing.Domain   -f net8.0
   dotnet new classlib -n OrderProcessing.Services -o src/OrderProcessing.Services -f net8.0
   dotnet new classlib -n OrderProcessing.Adapters -o src/OrderProcessing.Adapters -f net8.0
   dotnet new console  -n OrderProcessing.App      -o src/OrderProcessing.App      -f net8.0
   dotnet new xunit    -n OrderProcessing.Tests    -o tests/OrderProcessing.Tests  -f net8.0
   dotnet sln add src/OrderProcessing.Domain src/OrderProcessing.Services src/OrderProcessing.Adapters src/OrderProcessing.App tests/OrderProcessing.Tests
   ```
   Удалите автосгенерированный `Class1.cs` / `Unit1.cs` — они нам не нужны.

2. **Настройте ссылки между проектами** так, чтобы зависимости шли только внутрь, к домену:
   ```
   dotnet add src/OrderProcessing.Services reference src/OrderProcessing.Domain
   dotnet add src/OrderProcessing.Adapters reference src/OrderProcessing.Domain
   dotnet add src/OrderProcessing.App      reference src/OrderProcessing.Services src/OrderProcessing.Adapters
   dotnet add tests/OrderProcessing.Tests  reference src/OrderProcessing.Services src/OrderProcessing.Adapters
   ```
   Домен (`Domain`) не должен ссылаться ни на что внешнее — это чистая сборка с интерфейсами.

3. **Установите NuGet-пакеты.** Scrutor нужен в Composition Root (`App`) и в тестах (там тоже собирается контейнер):
   ```
   dotnet add src/OrderProcessing.App package Scrutor
   dotnet add src/OrderProcessing.App package Microsoft.Extensions.DependencyInjection
   dotnet add tests/OrderProcessing.Tests package Scrutor
   dotnet add tests/OrderProcessing.Tests package Microsoft.Extensions.DependencyInjection
   ```
   Версия Scrutor — последняя стабильная 6.x (под .NET 8).

4. **В проекте `Domain` опишите контракты.** Создайте файл `Abstractions.cs` с интерфейсами `IOrderService` (`Task<Guid> CreateAsync(string customer, decimal amount)`), `IPaymentGateway` (`void Pay(decimal amount)`), `ILogger` (`void Log(string message)`). Отдельно — `IServiceMarker.cs` с пустым маркерным интерфейсом `IServiceMarker {}`. Маркер нужен для выборочного сканирования: только классы с этим маркером попадут в `Scan`, DTO и модели останутся вне контейнера.

5. **В проекте `Services` реализуйте базовый сервис и три декоратора.** Класс `OrderService` должен быть `sealed`, реализовывать `IOrderService` И `IServiceMarker`, и печатать строку вида `[OrderService] Создан заказ {id} для {customer} на {amount:C}`. Затем — три декоратора: `LoggingOrderDecorator` (пишет «перед вызовом» и «после вызова» через `ILogger`), `CachingOrderDecorator` (хранит словарь `order:{customer}:{amount}` → `Guid`, возвращает кэшированный идентификатор при повторном вызове), `RetryOrderDecorator` (до 3 попыток, ловит исключения и логирует ретрай). Декораторы реализуют `IOrderService`, но **НЕ** `IServiceMarker` — иначе `Scan` подхватит их как ещё одну реализацию `IOrderService` и сломает разрешение.

6. **В проекте `Adapters` опишите сторонний клиент и адаптер.** `ExternalPaymentClient` — `sealed` класс, имитирующий сторонний SDK, с методом `void Charge(decimal amount, string currency)`. `PaymentAdapter : IPaymentGateway` принимает `ExternalPaymentClient` через конструктор и в `Pay` вызывает `_client.Charge(amount, "USD")`. Внешний тип НЕ реализует `IPaymentGateway` — контракт существует только в вашем домене.

7. **В проекте `App` напишите Composition Root.** `Program.cs` — top-level statements. Метод расширения `AddAppServices` на `IServiceCollection`: регистрирует `ILogger` → `ConsoleLogger` (Singleton); выполняет `Scan` с `FromAssemblyOf<OrderService>()`, `AddClasses(c => c.AssignableTo<IServiceMarker>())`, `UsingRegistrationStrategy(RegistrationStrategy.Skip)`, `AsMatchingInterface()`, `WithScopedLifetime()`; затем три `Decorate<IOrderService, ...>()` в порядке `Retry → Cache → Logging` (последний вызов `Decorate` — самый внешний слой); затем `AddSingleton<ExternalPaymentClient>()` и `RegisterAdapter<ExternalPaymentClient, IPaymentGateway>(client => new PaymentAdapter(client))`. В `Main` создайте scope, разрешите `IOrderService`, вызовите `CreateAsync("Alice", 9.99m)`, затем разрешите `IPaymentGateway` и вызовите `Pay(9.99m)`.

8. **Запустите приложение:**
   ```
   dotnet run --project src/OrderProcessing.App
   ```
   Ожидаемый вывод содержит (порядок может отличаться на ретраях, но логирующий декоратор должен обрамлять остальные строки):
   ```
   [Logger] [Decorator:Logging] Перед вызовом CreateAsync для Alice
   [Logger] [Decorator:Retry] Попытка 1/3
   [OrderService] Создан заказ ... для Alice на ¤9.99
   [Decorator:Cache] Сохранён в кэш ...
   [Logger] [Decorator:Logging] После вызова, orderId=...
   [ExternalPayment] Списано 9.99 USD
   ```
   Если сообщений декораторов нет — значит `Decorate` не сработал или вызван до `Scan`.

9. **Напишите unit-тест в `OrderProcessing.Tests`.** Используйте xUnit. Создайте `SpyLogger : ILogger`, который складывает сообщения в `List<string>`. В тесте соберите контейнер с тем же `Scan` и цепочкой `Decorate`, разрешите `IOrderService`, вызовите `CreateAsync("Bob", 1m)` и через `Assert.Contains` убедитесь, что в логе есть сообщения и от `LoggingOrderDecorator`, и от `RetryOrderDecorator`. Это доказывает, что декораторы действительно обёрнуты вокруг `OrderService`.

10. **Запустите тесты и убедитесь, что всё зелёное:**
    ```
    dotnet test
    ```

#### Требования к решению

- Целевой фреймворк — `net8.0`, язык — C# 12 (используйте top-level statements в `App`, `sealed`-классы, collection expressions и pattern matching там, где это уместно).
- Scrutor версии 6.x; пакет установлен только там, где действительно собирается контейнер (`App` и `Tests`).
- Структура проектов: `Domain` → `Services`, `Adapters` → `Domain`, `App` ссылается на `Services` и `Adapters`, `Tests` — на `Services` и `Adapters`. Домен не зависит ни от чего.
- Маркерный интерфейс `IServiceMarker` живёт в `Domain` и применяется **только** к реальным сервисам (`OrderService`). Декораторы его не реализуют.
- Регистрация сервисов выполняется исключительно через `Scan` с фильтром `AssignableTo<IServiceMarker>()`, `AsMatchingInterface()`, `WithScopedLifetime()` и явно заданной `RegistrationStrategy.Skip`.
- Цепочка декораторов собрана тремя вызовами `Decorate<IOrderService, TDec>()` в порядке `Retry → Cache → Logging`, чтобы `Logging` оказался внешним слоем.
- Адаптер зарегистрирован через `RegisterAdapter<ExternalPaymentClient, IPaymentGateway>(...)`; внешний тип зарегистрирован отдельно как `Singleton`.
- Время жизни декораторов автоматически унаследовано от декорируемого дескриптора (Scoped); никаких Singleton-декораторов поверх Scoped-сервисов.
- Unit-тест с подменой `ILogger` доказывает, что декораторы применились; тест не обращается к конкретным типам декораторов напрямую (только к интерфейсу и к логу).
- Решение компилируется без предупреждений уровня `Warning` и выше (`dotnet build -warnaserror` в идеале).

#### Тонкости и подводные камни

- **Не сканируйте «все классы сборки».** `AddClasses()` без фильтра подхватит DTO, модели, перечисления, и контейнер забьётся мусорными дескрипторами. Всегда фильтруйте через `AssignableTo<IServiceMarker>()` или атрибут. Это прямая рекомендация из урока.
- **`AsMatchingInterface()` требует конвенции именования.** Класс `OrderService` сопоставляется с `IOrderService` (имя минус префикс `I`). Назовёте класс `OrderSvc` — сопоставления не будет, и `GetRequiredService<IOrderService>()` упадёт с `InvalidOperationException`. Если конвенция нарушается, используйте `As<IFoo>()` или `AsImplementedInterfaces()`.
- **`AsSelf()` вместо `AsMatchingInterface()` — частая ошибка.** Тип будет зарегистрирован сам по себе и разрешится как `OrderService`, но не как `IOrderService`. Проверяйте, что вы регистрируете именно по интерфейсу.
- **Декораторы НЕ должны иметь `IServiceMarker`.** Иначе `Scan` зарегистрирует `LoggingOrderDecorator` как ещё одну реализацию `IOrderService`, и при нескольких реализациях одного интерфейса контейнер вернёт последнюю зарегистрированную — поведение станет непредсказуемым.
- **Порядок вызовов `Decorate`.** Каждый `Decorate<T,TDec>()` заменяет текущий дескриптор `T` на декоратор, который инжектит предыдущую регистрацию. Поэтому последний вызов `Decorate` — самый внешний слой. Чтобы получить `Logging(Caching(Retry(OrderService)))`, вызывайте `Decorate(Retry)`, затем `Decorate(Caching)`, затем `Decorate(Logging)`.
- **`Decorate` должен идти после `Scan`/`Add`.** Если вызвать `Decorate` до регистрации `IOrderService`, декорировать будет нечего — контейнер бросит исключение или просто проигнорирует.
- **Lifetime alignment.** Scrutor наследует время жизни декоратора от декорируемого дескриптора, поэтому Singleton-декоратор поверх Scoped-сервиса в Scrutor невозможен по построению — но если вы смешиваете ручную и скан-регистрацию, следите за тем, чтобы `AddSingleton<IOrderService, ...>()` не оказался обёрнут Scoped-декоратором с состоянием (кэш).
- **`RegistrationStrategy.Skip` обязателен**, если есть любая вероятность повторной регистрации (например, скан + явный `AddScoped`). Без стратегии дубликат бросает исключение. `Append` — альтернатива, когда нужно собрать список реализаций (`IEnumerable<IOrderService>`).
- **Адаптер собирайте в Composition Root, не в домене.** Домен знает только `IPaymentGateway`; тип `ExternalPaymentClient` не должен просачиваться в `Services` или `Domain`. Ссылка на сборку `Adapters` есть только у `App`.
- **Тестируйте факт декорирования, а не тип.** Не делайте `Assert.IsType<LoggingOrderDecorator>(...)` — это хрупко и связывает тест с внутренней структурой. Лучше подменить `ILogger` и проверить, что декоратор отработал по своим побочным эффектам (сообщениям в логе).

#### Критерии приёмки

- [ ] Создано решение `OrderProcessing` с шестью проектами и настроенными ссылками по принципу «зависимости внутрь».
- [ ] `Domain` содержит `IServiceMarker`, `IOrderService`, `IPaymentGateway`, `ILogger` и не зависит от внешних пакетов.
- [ ] `OrderService` помечен `IServiceMarker` и зарегистрируется через `Scan`; декораторы `IServiceMarker` не имеют.
- [ ] `AsMatchingInterface()` корректно сопоставляет `OrderService` → `IOrderService` (имена соответствуют конвенции).
- [ ] `RegistrationStrategy.Skip` задан явно в вызове `Scan`.
- [ ] Все три декоратора реализуют `IOrderService`, принимают `IOrderService inner` через конструктор и делегируют вызов.
- [ ] Цепочка `Decorate` построена в порядке `Retry → Cache → Logging`, так что `Logging` — внешний слой.
- [ ] `ExternalPaymentClient` зарегистрирован как `Singleton` отдельно.
- [ ] `PaymentAdapter` зарегистрирован через `RegisterAdapter<ExternalPaymentClient, IPaymentGateway>(...)`.
- [ ] `dotnet run --project src/OrderProcessing.App` выводит строки декораторов, базового сервиса и внешнего клиента.
- [ ] Unit-тест с `SpyLogger` проходит и `Assert.Contains` находит сообщения как минимум двух декораторов.
- [ ] `dotnet test` зелёный.
- [ ] Нет Singleton-декоратора поверх Scoped-сервиса (lifetime выровнен).
- [ ] Код компилируется без предупреждений `Warning` и выше.
- [ ] Внешний тип `ExternalPaymentClient` не появляется в `Domain` и `Services` (только в `Adapters` и `App`).

#### Подсказки (без прямого ответа)

- Если `GetRequiredService<IOrderService>()` бросает `InvalidOperationException`, проверьте: реализует ли `OrderService` именно `IOrderService`, и совпадает ли имя класса с интерфейсом по конвенции `I`+Имя.
- Если декораторов в выводе нет, перенесите вызовы `Decorate` после `Scan` и убедитесь, что они идут для того же интерфейса `IOrderService`.
- Если `RegisterAdapter` подчёркивается красным — вы забыли `using Scrutor;` или пакет не установлен в текущем проекте.
- Если кэш «работает» между запросами (возвращает тот же Guid сутками), значит `CachingOrderDecorator` оказался Singleton — проверьте lifetime исходной регистрации.
- Для теста не нужно подменять `OrderService` — достаточно подменить `ILogger`: если декораторы на месте, их сообщения попадут в `SpyLogger`.

#### Эталонное решение (разбор)

```csharp
// === src/OrderProcessing.Domain/IServiceMarker.cs ===
namespace OrderProcessing.Domain;

/// <summary>Маркер для выборочного сканирования сборки. / Marker for selective scanning.</summary>
public interface IServiceMarker { }
```

```csharp
// === src/OrderProcessing.Domain/Abstractions.cs ===
namespace OrderProcessing.Domain;

public interface IOrderService
{
    Task<Guid> CreateAsync(string customer, decimal amount); // Создать заказ / Create an order
}

public interface IPaymentGateway
{
    void Pay(decimal amount); // Провести платёж / Charge a payment
}

public interface ILogger
{
    void Log(string message);
}
```

```csharp
// === src/OrderProcessing.Services/OrderService.cs ===
namespace OrderProcessing.Services;

using OrderProcessing.Domain;

// Реальный сервис. Помечен IServiceMarker — попадёт в Scan.
// Real service. Marked with IServiceMarker — will be picked up by Scan.
public sealed class OrderService : IOrderService, IServiceMarker
{
    public Task<Guid> CreateAsync(string customer, decimal amount)
    {
        var id = Guid.NewGuid();
        Console.WriteLine($"[OrderService] Создан заказ {id} для {customer} на {amount:C}");
        return Task.FromResult(id);
    }
}
```

```csharp
// === src/OrderProcessing.Services/Decorators.cs ===
namespace OrderProcessing.Services;

using OrderProcessing.Domain;

// Внешний слой: логирование. НЕ реализует IServiceMarker.
// Outermost layer: logging. Does NOT implement IServiceMarker.
public sealed class LoggingOrderDecorator : IOrderService
{
    private readonly IOrderService _inner;
    private readonly ILogger _logger;

    public LoggingOrderDecorator(IOrderService inner, ILogger logger)
    {
        _inner = inner;
        _logger = logger;
    }

    public async Task<Guid> CreateAsync(string customer, decimal amount)
    {
        _logger.Log($"[Decorator:Logging] Перед вызовом CreateAsync для {customer}");
        try
        {
            var id = await _inner.CreateAsync(customer, amount);
            _logger.Log($"[Decorator:Logging] После вызова, orderId={id}");
            return id;
        }
        catch (Exception ex)
        {
            _logger.Log($"[Decorator:Logging] Ошибка: {ex.Message}");
            throw;
        }
    }
}

// Средний слой: кэш по ключу customer:amount. Состояние в Scoped-словаре.
// Middle layer: cache keyed by customer:amount. State in a Scoped dictionary.
public sealed class CachingOrderDecorator : IOrderService
{
    private readonly IOrderService _inner;
    private readonly Dictionary<string, Guid> _cache = new();

    public CachingOrderDecorator(IOrderService inner) => _inner = inner;

    public async Task<Guid> CreateAsync(string customer, decimal amount)
    {
        var key = $"order:{customer}:{amount}";
        if (_cache.TryGetValue(key, out var cached))
        {
            Console.WriteLine($"[Decorator:Cache] Возврат кэшированного {cached}");
            return cached;
        }

        var id = await _inner.CreateAsync(customer, amount);
        _cache[key] = id;
        Console.WriteLine($"[Decorator:Cache] Сохранён в кэш {id}");
        return id;
    }
}

// Внутренний слой: ретрай до 3 попыток.
// Innermost layer: retry up to 3 attempts.
public sealed class RetryOrderDecorator : IOrderService
{
    private readonly IOrderService _inner;
    private readonly ILogger _logger;
    private readonly int _maxAttempts;

    public RetryOrderDecorator(IOrderService inner, ILogger logger, int maxAttempts = 3)
    {
        _inner = inner;
        _logger = logger;
        _maxAttempts = maxAttempts;
    }

    public async Task<Guid> CreateAsync(string customer, decimal amount)
    {
        for (var attempt = 1; attempt <= _maxAttempts; attempt++)
        {
            try
            {
                _logger.Log($"[Decorator:Retry] Попытка {attempt}/{_maxAttempts}");
                return await _inner.CreateAsync(customer, amount);
            }
            catch (Exception ex) when (attempt < _maxAttempts)
            {
                _logger.Log($"[Decorator:Retry] Ошибка {ex.Message}, повтор...");
            }
        }

        return await _inner.CreateAsync(customer, amount); // последняя попытка без catch / final attempt
    }
}
```

```csharp
// === src/OrderProcessing.Adapters/ExternalPaymentClient.cs ===
namespace OrderProcessing.Adapters;

// Имитация стороннего SDK. Не реализует наши интерфейсы.
// Simulated third-party SDK. Does not implement our interfaces.
public sealed class ExternalPaymentClient
{
    public void Charge(decimal amount, string currency)
        => Console.WriteLine($"[ExternalPayment] Списано {amount} {currency}");
}
```

```csharp
// === src/OrderProcessing.Adapters/PaymentAdapter.cs ===
namespace OrderProcessing.Adapters;

using OrderProcessing.Domain;

// Адаптер переводит наш IPaymentGateway в контракт ExternalPaymentClient.
// Adapter translates our IPaymentGateway into ExternalPaymentClient's contract.
public sealed class PaymentAdapter : IPaymentGateway
{
    private readonly ExternalPaymentClient _client;
    public PaymentAdapter(ExternalPaymentClient client) => _client = client;
    public void Pay(decimal amount) => _client.Charge(amount, "USD");
}
```

```csharp
// === src/OrderProcessing.App/Program.cs ===
using Microsoft.Extensions.DependencyInjection;
using OrderProcessing.Domain;
using OrderProcessing.Services;
using OrderProcessing.Adapters;
using Scrutor;

var services = new ServiceCollection();
services.AddAppServices();

var provider = services.BuildServiceProvider();
using var scope = provider.CreateScope();

var orders = scope.ServiceProvider.GetRequiredService<IOrderService>();
await orders.CreateAsync("Alice", 9.99m);

var billing = scope.ServiceProvider.GetRequiredService<IPaymentGateway>();
billing.Pay(9.99m);

public static class ServiceRegistration
{
    public static IServiceCollection AddAppServices(this IServiceCollection services)
    {
        // 1) Инфраструктура для декораторов / Infrastructure for decorators
        services.AddSingleton<ILogger, ConsoleLogger>();

        // 2) Сканирование по маркеру: только реальные сервисы, Scoped, Skip дублей
        // 2) Scan by marker: real services only, Scoped, Skip duplicates
        services.Scan(scan => scan
            .FromAssemblyOf<OrderService>()
            .AddClasses(c => c.AssignableTo<IServiceMarker>())
            .UsingRegistrationStrategy(RegistrationStrategy.Skip)
            .AsMatchingInterface()
            .WithScopedLifetime());

        // 3) Цепочка декораторов. Порядок вызовов: каждый Decorate оборачивает текущий
        //    дескриптор IOrderService, поэтому последний вызов — внешний слой.
        // 3) Decorator chain. Each Decorate wraps the current IOrderService descriptor,
        //    so the last call is the outermost layer.
        services.Decorate<IOrderService, RetryOrderDecorator>();    // внутренний / inner
        services.Decorate<IOrderService, CachingOrderDecorator>();  // средний   / middle
        services.Decorate<IOrderService, LoggingOrderDecorator>();  // внешний   / outer
        // Итог / Result: Logging(Caching(Retry(OrderService)))

        // 4) Адаптер внешнего типа / Adapter for external type
        services.AddSingleton<ExternalBillingClient>();
        services.RegisterAdapter<ExternalPaymentClient, IPaymentGateway>(
            client => new PaymentAdapter(client));

        return services;
    }
}

public sealed class ConsoleLogger : ILogger
{
    public void Log(string message) => Console.WriteLine($"[Logger] {message}");
}
```

```csharp
// === tests/OrderProcessing.Tests/OrderServiceDecorationTests.cs ===
using Microsoft.Extensions.DependencyInjection;
using OrderProcessing.Domain;
using OrderProcessing.Services;
using Scrutor;

public class OrderServiceDecorationTests
{
    [Fact]
    public async Task Resolved_IOrderService_Runs_Through_Logging_And_Retry_Decorators()
    {
        var spy = new SpyLogger();
        var services = new ServiceCollection();
        services.AddSingleton<ILogger>(spy);

        services.Scan(scan => scan
            .FromAssemblyOf<OrderService>()
            .AddClasses(c => c.AssignableTo<IServiceMarker>())
            .UsingRegistrationStrategy(RegistrationStrategy.Skip)
            .AsMatchingInterface()
            .WithScopedLifetime());

        services.Decorate<IOrderService, RetryOrderDecorator>();
        services.Decorate<IOrderService, CachingOrderDecorator>();
        services.Decorate<IOrderService, LoggingOrderDecorator>();

        var provider = services.BuildServiceProvider();
        var orderService = provider.GetRequiredService<IOrderService>();

        await orderService.CreateAsync("Bob", 1m);

        Assert.Contains(spy.Messages, m => m.Contains("[Decorator:Logging]"));
        Assert.Contains(spy.Messages, m => m.Contains("[Decorator:Retry]"));
    }

    private sealed class SpyLogger : ILogger
    {
        public List<string> Messages { get; } = [];
        public void Log(string message) => Messages.Add(message);
    }
}
```

**Разбор по строкам.** `IServiceMarker` — пустой маркерный интерфейс в `Domain`; именно он работает как фильтр в `AddClasses(c => c.AssignableTo<IServiceMarker>())`, благодаря чему DTO и модели не попадают в контейнер (прямая рекомендация из best practices урока). `OrderService` помечен и `IOrderService`, и `IServiceMarker`, а декораторы — только `IOrderService`: это критично, иначе `Scan` подхватил бы декораторы как вторую реализацию `IOrderService` и разрушил разрешение (частая ошибка из урока). Декораторы принимают `IOrderService inner` через конструктор — контейнер сам подставит туда «внутреннюю» регистрацию, никаких keyed-сервисов.

`Scan` использует `FromAssemblyOf<OrderService>()` (тип из сборки `Services`), `AsMatchingInterface()` (конвенция `I`+Имя → `OrderService` разрешается как `IOrderService`), `WithScopedLifetime()` и явно `RegistrationStrategy.Skip` — последний пункт обязателен, иначе дубликат бросает исключение (тонкость из урока). Цепочка `Decorate` построена в порядке `Retry → Cache → Logging`: каждый вызов заменяет дескриптор `IOrderService` на декоратор с инжекцией предыдущей регистрации, поэтому последний вызов (`Logging`) становится внешним слоем. Итоговая композиция — `Logging(Caching(Retry(OrderService)))`, что мы и хотим.

`PaymentAdapter` инкапсулирует `ExternalPaymentClient`: домен и сервисы знают только `IPaymentGateway`, внешний тип живёт в сборке `Adapters` и регистрируется в Composition Root через `AddSingleton<ExternalBillingClient>()` плюс `RegisterAdapter<ExternalBillingClient, IPaymentGateway>(client => new PaymentAdapter(client))`. Это реализует принцип «изоляции внешних библиотек» из best practices урока. Тест `SpyLogger` подменяет `ILogger` и через `Assert.Contains` проверяет, что в логе есть сообщения декораторов — это доказывает факт декорирования, не привязываясь к конкретным типам декораторов (хрупкому `IsType`).

#### Задания на углубление (бонус)

1. **Обобщённый декоратор.** Реализуйте `LoggingDecorator<TService> where TService : IServiceMarker` как open-generic декоратор и примените его сразу к нескольким сервисам (`IOrderService`, `IInventoryService`) через `services.Decorate(typeof(LoggingDecorator<>), ...)` или несколько `Decorate<T,TDec>()`. Сравните удобство с конкретными декораторами.
2. **Условное декорирование по конфигу.** Добавьте `IConfiguration` с флагом `Features:EnableRetry` и оборачивайте `RetryOrderDecorator` только когда флаг включён. Используйте перегрузку `Decorate` с фабрикой или условный `if` в `AddAppServices`.
3. **Сканирование нескольких сборок.** Добавьте второй слой (например, `OrderProcessing.Handlers`) с маркером `IHandler`, и зарегистрируйте его через `FromAssembliesOf<OrderService, SomeHandler>()`. Убедитесь, что `AsMatchingInterface()` корректно сопоставляет типы из обеих сборок.
4. **Атрибутная регистрация вместо маркера.** Замените `IServiceMarker` атрибутом `[ScopedService]` и сканируйте через `AddClasses(c => c.AssignableTo<IServiceMarker>().Where(t => t.GetCustomAttribute<ScopedServiceAttribute>() is not null))` с разными lifetimes по атрибуту.

---

## Statement in English / Постановка на английском

#### Context & motivation

In large .NET applications the service layer grows very quickly: orders, inventory, notifications, billing, reporting, audit — each with its own interface and its own implementation. Registering every service by hand in `Program.cs` with `services.AddScoped<IFoo, Foo>()` turns, after six months, into a wall of several hundred lines that nobody dares to review: rename an interface and you can easily forget to update the registration line; add a new service and you can easily forget to register it at all; duplicate registrations can silently break resolution. The Scrutor library fixes this pain through convention-based registration by a marker interface: just mark every "real" service with `IServiceMarker`, and a single `Scan(...)` call registers them all with a uniform lifetime and an explicit duplicate-handling strategy.

In parallel Scrutor elegantly solves a second pain — decorators. When you need to add logging, caching, and retry to several services at once, hand-rolling the "matryoshka" turns into a nightmare of keyed services, factories, and `GetService<IOrderService>("inner")`. The `Decorate<TInterface, TDecorator>()` method lets the container manage the nesting itself: each `Decorate` call wraps the current registration of `TInterface`, aligns the lifetime automatically, and lets you build chains of any length. The third feature — adapters through `RegisterAdapter<TImplementation, TAdapter>()` — isolates third-party SDKs (payment gateways, SMS providers, cloud clients) behind your own interface, so the domain never depends on external packages, their types, or their versions.

In this assignment you will assemble a mini-project "OrderProcessing" with three domain services, three decorators in a single chain, and one adapter for an external payment client, register everything through Scrutor, and prove the composition is correct with a unit test that substitutes `ILogger` and asserts that the decorators' messages actually appear in the log.

#### What to do step by step

1. **Create the solution and six projects.** From the repository root, run the commands one by one, checking that each finishes without errors:
   ```
   dotnet new sln -n OrderProcessing
   dotnet new classlib -n OrderProcessing.Domain   -o src/OrderProcessing.Domain   -f net8.0
   dotnet new classlib -n OrderProcessing.Services -o src/OrderProcessing.Services -f net8.0
   dotnet new classlib -n OrderProcessing.Adapters -o src/OrderProcessing.Adapters -f net8.0
   dotnet new console  -n OrderProcessing.App      -o src/OrderProcessing.App      -f net8.0
   dotnet new xunit    -n OrderProcessing.Tests    -o tests/OrderProcessing.Tests  -f net8.0
   dotnet sln add src/OrderProcessing.Domain src/OrderProcessing.Services src/OrderProcessing.Adapters src/OrderProcessing.App tests/OrderProcessing.Tests
   ```
   Delete the auto-generated `Class1.cs` / `Unit1.cs` — they are not needed.

2. **Configure project references** so dependencies only point inward, toward the domain:
   ```
   dotnet add src/OrderProcessing.Services reference src/OrderProcessing.Domain
   dotnet add src/OrderProcessing.Adapters reference src/OrderProcessing.Domain
   dotnet add src/OrderProcessing.App      reference src/OrderProcessing.Services src/OrderProcessing.Adapters
   dotnet add tests/OrderProcessing.Tests  reference src/OrderProcessing.Services src/OrderProcessing.Adapters
   ```
   The `Domain` project must not reference anything external — it is a pure contract assembly.

3. **Install NuGet packages.** Scrutor is needed in the Composition Root (`App`) and in the tests (the container is assembled there too):
   ```
   dotnet add src/OrderProcessing.App package Scrutor
   dotnet add src/OrderProcessing.App package Microsoft.Extensions.DependencyInjection
   dotnet add tests/OrderProcessing.Tests package Scrutor
   dotnet add tests/OrderProcessing.Tests package Microsoft.Extensions.DependencyInjection
   ```
   Use the latest stable Scrutor 6.x (compatible with .NET 8).

4. **Describe the contracts in `Domain`.** Create `Abstractions.cs` with interfaces `IOrderService` (`Task<Guid> CreateAsync(string customer, decimal amount)`), `IPaymentGateway` (`void Pay(decimal amount)`), `ILogger` (`void Log(string message)`). Separately, add `IServiceMarker.cs` with an empty marker interface `IServiceMarker {}`. The marker exists for selective scanning: only classes carrying it will enter the `Scan`, while DTOs and models stay out of the container.

5. **In `Services`, implement the base service and three decorators.** `OrderService` should be `sealed`, implement both `IOrderService` and `IServiceMarker`, and print a line like `[OrderService] Created order {id} for {customer} of {amount:C}`. Then three decorators: `LoggingOrderDecorator` (writes "before call" and "after call" through `ILogger`), `CachingOrderDecorator` (keeps a dictionary `order:{customer}:{amount}` → `Guid` and returns the cached id on a repeat call), `RetryOrderDecorator` (up to 3 attempts, catches exceptions and logs the retry). The decorators implement `IOrderService` but **NOT** `IServiceMarker` — otherwise `Scan` will pick them up as a second implementation of `IOrderService` and break resolution.

6. **In `Adapters`, describe the third-party client and the adapter.** `ExternalPaymentClient` is a `sealed` class that mimics a third-party SDK with a method `void Charge(decimal amount, string currency)`. `PaymentAdapter : IPaymentGateway` takes `ExternalPaymentClient` through the constructor and in `Pay` calls `_client.Charge(amount, "USD")`. The external type does NOT implement `IPaymentGateway` — the contract lives only in your domain.

7. **Write the Composition Root in `App`.** `Program.cs` is top-level statements. An extension method `AddAppServices` on `IServiceCollection` registers `ILogger` → `ConsoleLogger` (Singleton); runs `Scan` with `FromAssemblyOf<OrderService>()`, `AddClasses(c => c.AssignableTo<IServiceMarker>())`, `UsingRegistrationStrategy(RegistrationStrategy.Skip)`, `AsMatchingInterface()`, `WithScopedLifetime()`; then three `Decorate<IOrderService, ...>()` calls in the order `Retry → Cache → Logging` (the last `Decorate` is the outermost layer); then `AddSingleton<ExternalBillingClient>()` and `RegisterAdapter<ExternalBillingClient, IPaymentGateway>(client => new PaymentAdapter(client))`. In `Main`, create a scope, resolve `IOrderService`, call `CreateAsync("Alice", 9.99m)`, then resolve `IPaymentGateway` and call `Pay(9.99m)`.

8. **Run the application:**
   ```
   dotnet run --project src/OrderProcessing.App
   ```
   Expected output contains (order may vary on retries, but the logging decorator must wrap the other lines):
   ```
   [Logger] [Decorator:Logging] Before CreateAsync for Alice
   [Logger] [Decorator:Retry] Attempt 1/3
   [OrderService] Created order ... for Alice of ¤9.99
   [Decorator:Cache] Cached ...
   [Logger] [Decorator:Logging] After call, orderId=...
   [ExternalPayment] Charged 9.99 USD
   ```
   If the decorators' messages are missing, `Decorate` did not take effect or was called before `Scan`.

9. **Write a unit test in `OrderProcessing.Tests`** using xUnit. Create a `SpyLogger : ILogger` that stores messages in a `List<string>`. In the test, assemble the container with the same `Scan` and `Decorate` chain, resolve `IOrderService`, call `CreateAsync("Bob", 1m)`, and use `Assert.Contains` to confirm that the log holds messages from both `LoggingOrderDecorator` and `RetryOrderDecorator`. That proves the decorators are actually wrapping `OrderService`.

10. **Run the tests and make sure everything is green:**
    ```
    dotnet test
    ```

#### Requirements

- Target framework `net8.0`, language C# 12 (use top-level statements in `App`, `sealed` classes, collection expressions and pattern matching where appropriate).
- Scrutor 6.x; the package is installed only where the container is actually assembled (`App` and `Tests`).
- Project structure: `Domain` ← `Services`, `Adapters` ← `Domain`; `App` references `Services` and `Adapters`; `Tests` references `Services` and `Adapters`. The domain depends on nothing.
- The marker interface `IServiceMarker` lives in `Domain` and is applied **only** to real services (`OrderService`). Decorators do not implement it.
- Service registration happens exclusively through `Scan` with the `AssignableTo<IServiceMarker>()` filter, `AsMatchingInterface()`, `WithScopedLifetime()`, and an explicit `RegistrationStrategy.Skip`.
- The decorator chain is built with three `Decorate<IOrderService, TDec>()` calls in the order `Retry → Cache → Logging`, so `Logging` ends up as the outermost layer.
- The adapter is registered through `RegisterAdapter<ExternalBillingClient, IPaymentGateway>(...)`; the external type is registered separately as `Singleton`.
- Decorator lifetimes are inherited from the decorated descriptor (Scoped); no Singleton decorators over Scoped services.
- A unit test with a substituted `ILogger` proves the decorators were applied; the test does not reach into concrete decorator types directly (only the interface and the log).
- The solution compiles without `Warning`-level diagnostics or higher (`dotnet build -warnaserror` ideally).

#### Pitfalls

- **Do not scan "every class in the assembly".** `AddClasses()` without a filter will pick up DTOs, models, and enums and bloat the container. Always filter with `AssignableTo<IServiceMarker>()` or an attribute. This is a direct recommendation from the lesson.
- **`AsMatchingInterface()` requires a naming convention.** Class `OrderService` maps to `IOrderService` (name minus the `I` prefix). Name the class `OrderSvc` and the mapping fails, so `GetRequiredService<IOrderService>()` throws `InvalidOperationException`. If the convention is broken, use `As<IFoo>()` or `AsImplementedInterfaces()`.
- **`AsSelf()` instead of `AsMatchingInterface()` is a common mistake.** The type will be registered as itself and resolvable as `OrderService`, but not as `IOrderService`. Verify you are registering by the interface.
- **Decorators must NOT carry `IServiceMarker`.** Otherwise `Scan` registers `LoggingOrderDecorator` as another implementation of `IOrderService`, and with multiple implementations the container returns the last registered one — behavior becomes unpredictable.
- **Order of `Decorate` calls.** Each `Decorate<T,TDec>()` replaces the current `T` descriptor with a decorator that injects the previous registration. So the last `Decorate` call is the outermost layer. To get `Logging(Caching(Retry(OrderService)))`, call `Decorate(Retry)`, then `Decorate(Caching)`, then `Decorate(Logging)`.
- **`Decorate` must run after `Scan`/`Add`.** Calling `Decorate` before registering `IOrderService` leaves nothing to decorate — the container throws or silently ignores it.
- **Lifetime alignment.** Scrutor inherits the decorator lifetime from the decorated descriptor, so a Singleton decorator over a Scoped service is structurally impossible in Scrutor — but if you mix manual and scan registration, make sure `AddSingleton<IOrderService, ...>()` is not wrapped by a stateful Scoped decorator (cache).
- **`RegistrationStrategy.Skip` is mandatory** whenever a duplicate registration is possible (for example, scan plus explicit `AddScoped`). Without a strategy, a duplicate throws. `Append` is the alternative when you want to collect a list of implementations (`IEnumerable<IOrderService>`).
- **Build the adapter in the Composition Root, not in the domain.** The domain only knows `IPaymentGateway`; the type `ExternalPaymentClient` must not leak into `Services` or `Domain`. Only `App` references the `Adapters` assembly.
- **Test the fact of decoration, not the concrete type.** Do not `Assert.IsType<LoggingOrderDecorator>(...)` — it is brittle and couples the test to internals. Substitute `ILogger` instead and verify the decorator fired via its side effects (log messages).

#### Acceptance criteria

- [ ] Solution `OrderProcessing` created with six projects and references configured "inward".
- [ ] `Domain` contains `IServiceMarker`, `IOrderService`, `IPaymentGateway`, `ILogger` and depends on no external packages.
- [ ] `OrderService` is marked with `IServiceMarker` and gets registered through `Scan`; decorators are not marked.
- [ ] `AsMatchingInterface()` correctly maps `OrderService` → `IOrderService` (names follow the convention).
- [ ] `RegistrationStrategy.Skip` is set explicitly in the `Scan` call.
- [ ] All three decorators implement `IOrderService`, take `IOrderService inner` through the constructor, and delegate the call.
- [ ] The `Decorate` chain is built in the order `Retry → Cache → Logging`, so `Logging` is the outermost layer.
- [ ] `ExternalPaymentClient` is registered separately as `Singleton`.
- [ ] `PaymentAdapter` is registered through `RegisterAdapter<ExternalBillingClient, IPaymentGateway>(...)`.
- [ ] `dotnet run --project src/OrderProcessing.App` prints lines from the decorators, the base service, and the external client.
- [ ] The `SpyLogger` unit test passes and `Assert.Contains` finds messages from at least two decorators.
- [ ] `dotnet test` is green.
- [ ] No Singleton decorator over a Scoped service (lifetimes aligned).
- [ ] The code compiles without `Warning`-level diagnostics or higher.
- [ ] The external type `ExternalPaymentClient` does not appear in `Domain` or `Services` (only in `Adapters` and `App`).

#### Hints (no direct answer)

- If `GetRequiredService<IOrderService>()` throws `InvalidOperationException`, check whether `OrderService` actually implements `IOrderService` and whether the class name matches the interface by the `I`+Name convention.
- If the decorators are missing from the output, move the `Decorate` calls after `Scan` and make sure they target the same `IOrderService` interface.
- If `RegisterAdapter` is underlined in red, you forgot `using Scrutor;` or the package is not installed in that project.
- If the cache "works" across requests (returns the same Guid for days), `CachingOrderDecorator` became a Singleton — check the original registration's lifetime.
- For the test you do not need to substitute `OrderService` — substituting `ILogger` is enough: if the decorators are in place, their messages land in `SpyLogger`.

#### Reference solution walk-through

```csharp
// === src/OrderProcessing.Domain/IServiceMarker.cs ===
namespace OrderProcessing.Domain;

/// <summary>Marker for selective assembly scanning.</summary>
public interface IServiceMarker { }
```

```csharp
// === src/OrderProcessing.Domain/Abstractions.cs ===
namespace OrderProcessing.Domain;

public interface IOrderService
{
    Task<Guid> CreateAsync(string customer, decimal amount);
}

public interface IPaymentGateway
{
    void Pay(decimal amount);
}

public interface ILogger
{
    void Log(string message);
}
```

```csharp
// === src/OrderProcessing.Services/OrderService.cs ===
namespace OrderProcessing.Services;

using OrderProcessing.Domain;

// Real service. Marked with IServiceMarker — picked up by Scan.
public sealed class OrderService : IOrderService, IServiceMarker
{
    public Task<Guid> CreateAsync(string customer, decimal amount)
    {
        var id = Guid.NewGuid();
        Console.WriteLine($"[OrderService] Created order {id} for {customer} of {amount:C}");
        return Task.FromResult(id);
    }
}
```

```csharp
// === src/OrderProcessing.Services/Decorators.cs ===
namespace OrderProcessing.Services;

using OrderProcessing.Domain;

// Outermost layer: logging. Does NOT implement IServiceMarker.
public sealed class LoggingOrderDecorator : IOrderService
{
    private readonly IOrderService _inner;
    private readonly ILogger _logger;

    public LoggingOrderDecorator(IOrderService inner, ILogger logger)
    {
        _inner = inner;
        _logger = logger;
    }

    public async Task<Guid> CreateAsync(string customer, decimal amount)
    {
        _logger.Log($"[Decorator:Logging] Before CreateAsync for {customer}");
        try
        {
            var id = await _inner.CreateAsync(customer, amount);
            _logger.Log($"[Decorator:Logging] After call, orderId={id}");
            return id;
        }
        catch (Exception ex)
        {
            _logger.Log($"[Decorator:Logging] Error: {ex.Message}");
            throw;
        }
    }
}

// Middle layer: cache keyed by customer:amount. State in a Scoped dictionary.
public sealed class CachingOrderDecorator : IOrderService
{
    private readonly IOrderService _inner;
    private readonly Dictionary<string, Guid> _cache = new();

    public CachingOrderDecorator(IOrderService inner) => _inner = inner;

    public async Task<Guid> CreateAsync(string customer, decimal amount)
    {
        var key = $"order:{customer}:{amount}";
        if (_cache.TryGetValue(key, out var cached))
        {
            Console.WriteLine($"[Decorator:Cache] Returning cached {cached}");
            return cached;
        }

        var id = await _inner.CreateAsync(customer, amount);
        _cache[key] = id;
        Console.WriteLine($"[Decorator:Cache] Cached {id}");
        return id;
    }
}

// Innermost layer: retry up to 3 attempts.
public sealed class RetryOrderDecorator : IOrderService
{
    private readonly IOrderService _inner;
    private readonly ILogger _logger;
    private readonly int _maxAttempts;

    public RetryOrderDecorator(IOrderService inner, ILogger logger, int maxAttempts = 3)
    {
        _inner = inner;
        _logger = logger;
        _maxAttempts = maxAttempts;
    }

    public async Task<Guid> CreateAsync(string customer, decimal amount)
    {
        for (var attempt = 1; attempt <= _maxAttempts; attempt++)
        {
            try
            {
                _logger.Log($"[Decorator:Retry] Attempt {attempt}/{_maxAttempts}");
                return await _inner.CreateAsync(customer, amount);
            }
            catch (Exception ex) when (attempt < _maxAttempts)
            {
                _logger.Log($"[Decorator:Retry] Error {ex.Message}, retrying...");
            }
        }

        return await _inner.CreateAsync(customer, amount); // final attempt, no catch
    }
}
```

```csharp
// === src/OrderProcessing.Adapters/ExternalPaymentClient.cs ===
namespace OrderProcessing.Adapters;

// Simulated third-party SDK. Does not implement our interfaces.
public sealed class ExternalPaymentClient
{
    public void Charge(decimal amount, string currency)
        => Console.WriteLine($"[ExternalPayment] Charged {amount} {currency}");
}
```

```csharp
// === src/OrderProcessing.Adapters/PaymentAdapter.cs ===
namespace OrderProcessing.Adapters;

using OrderProcessing.Domain;

// Adapter translates our IPaymentGateway into ExternalPaymentClient's contract.
public sealed class PaymentAdapter : IPaymentGateway
{
    private readonly ExternalPaymentClient _client;
    public PaymentAdapter(ExternalPaymentClient client) => _client = client;
    public void Pay(decimal amount) => _client.Charge(amount, "USD");
}
```

```csharp
// === src/OrderProcessing.App/Program.cs ===
using Microsoft.Extensions.DependencyInjection;
using OrderProcessing.Domain;
using OrderProcessing.Services;
using OrderProcessing.Adapters;
using Scrutor;

var services = new ServiceCollection();
services.AddAppServices();

var provider = services.BuildServiceProvider();
using var scope = provider.CreateScope();

var orders = scope.ServiceProvider.GetRequiredService<IOrderService>();
await orders.CreateAsync("Alice", 9.99m);

var billing = scope.ServiceProvider.GetRequiredService<IPaymentGateway>();
billing.Pay(9.99m);

public static class ServiceRegistration
{
    public static IServiceCollection AddAppServices(this IServiceCollection services)
    {
        // 1) Infrastructure used by decorators
        services.AddSingleton<ILogger, ConsoleLogger>();

        // 2) Scan by marker: real services only, Scoped, Skip duplicates
        services.Scan(scan => scan
            .FromAssemblyOf<OrderService>()
            .AddClasses(c => c.AssignableTo<IServiceMarker>())
            .UsingRegistrationStrategy(RegistrationStrategy.Skip)
            .AsMatchingInterface()
            .WithScopedLifetime());

        // 3) Decorator chain. Each Decorate wraps the current IOrderService descriptor,
        //    so the last call becomes the outermost layer.
        services.Decorate<IOrderService, RetryOrderDecorator>();    // inner
        services.Decorate<IOrderService, CachingOrderDecorator>();  // middle
        services.Decorate<IOrderService, LoggingOrderDecorator>();  // outer
        // Result: Logging(Caching(Retry(OrderService)))

        // 4) Adapter for the external type
        services.AddSingleton<ExternalBillingClient>();
        services.RegisterAdapter<ExternalPaymentClient, IPaymentGateway>(
            client => new PaymentAdapter(client));

        return services;
    }
}

public sealed class ConsoleLogger : ILogger
{
    public void Log(string message) => Console.WriteLine($"[Logger] {message}");
}
```

```csharp
// === tests/OrderProcessing.Tests/OrderServiceDecorationTests.cs ===
using Microsoft.Extensions.DependencyInjection;
using OrderProcessing.Domain;
using OrderProcessing.Services;
using Scrutor;

public class OrderServiceDecorationTests
{
    [Fact]
    public async Task Resolved_IOrderService_Runs_Through_Logging_And_Retry_Decorators()
    {
        var spy = new SpyLogger();
        var services = new ServiceCollection();
        services.AddSingleton<ILogger>(spy);

        services.Scan(scan => scan
            .FromAssemblyOf<OrderService>()
            .AddClasses(c => c.AssignableTo<IServiceMarker>())
            .UsingRegistrationStrategy(RegistrationStrategy.Skip)
            .AsMatchingInterface()
            .WithScopedLifetime());

        services.Decorate<IOrderService, RetryOrderDecorator>();
        services.Decorate<IOrderService, CachingOrderDecorator>();
        services.Decorate<IOrderService, LoggingOrderDecorator>();

        var provider = services.BuildServiceProvider();
        var orderService = provider.GetRequiredService<IOrderService>();

        await orderService.CreateAsync("Bob", 1m);

        Assert.Contains(spy.Messages, m => m.Contains("[Decorator:Logging]"));
        Assert.Contains(spy.Messages, m => m.Contains("[Decorator:Retry]"));
    }

    private sealed class SpyLogger : ILogger
    {
        public List<string> Messages { get; } = [];
        public void Log(string message) => Messages.Add(message);
    }
}
```

**Line-by-line walk-through.** `IServiceMarker` is an empty marker interface in `Domain`; it is the filter used by `AddClasses(c => c.AssignableTo<IServiceMarker>())`, which keeps DTOs and models out of the container (a direct best practice from the lesson). `OrderService` carries both `IOrderService` and `IServiceMarker`, while the decorators carry only `IOrderService`: this is critical, otherwise `Scan` would pick the decorators up as a second implementation of `IOrderService` and wreck resolution (a common mistake from the lesson). The decorators take `IOrderService inner` through the constructor — the container injects the "inner" registration itself, with no keyed services.

`Scan` uses `FromAssemblyOf<OrderService>()` (a type from the `Services` assembly), `AsMatchingInterface()` (the `I`+Name convention — `OrderService` resolves as `IOrderService`), `WithScopedLifetime()`, and an explicit `RegistrationStrategy.Skip` — the last point is mandatory, otherwise a duplicate throws (a pitfall from the lesson). The `Decorate` chain is built in the order `Retry → Cache → Logging`: each call replaces the `IOrderService` descriptor with a decorator that injects the previous registration, so the last call (`Logging`) becomes the outermost layer. The final composition is `Logging(Caching(Retry(OrderService)))`, exactly what we want.

`PaymentAdapter` encapsulates `ExternalPaymentClient`: the domain and services know only `IPaymentGateway`; the external type lives in the `Adapters` assembly and is registered in the Composition Root via `AddSingleton<ExternalBillingClient>()` plus `RegisterAdapter<ExternalBillingClient, IPaymentGateway>(client => new PaymentAdapter(client))`. This realizes the "isolate third-party libraries" principle from the lesson's best practices. The `SpyLogger` test substitutes `ILogger` and uses `Assert.Contains` to confirm the decorators' messages are present — proving decoration without reaching for the brittle `IsType` assertion on concrete decorator types.

#### Going deeper (bonus)

1. **Generic decorator.** Implement `LoggingDecorator<TService> where TService : IServiceMarker` as an open-generic decorator and apply it to several services (`IOrderService`, `IInventoryService`) at once via `services.Decorate(typeof(LoggingDecorator<>), ...)` or several `Decorate<T,TDec>()` calls. Compare the ergonomics with concrete decorators.
2. **Conditional decoration from config.** Add `IConfiguration` with a flag `Features:EnableRetry` and only wrap `RetryOrderDecorator` when the flag is on. Use the `Decorate` overload with a factory or a conditional `if` in `AddAppServices`.
3. **Scanning multiple assemblies.** Add a second layer (say `OrderProcessing.Handlers`) with a marker `IHandler`, and register it through `FromAssembliesOf<OrderService, SomeHandler>()`. Confirm that `AsMatchingInterface()` maps types from both assemblies correctly.
4. **Attribute-based registration instead of a marker.** Replace `IServiceMarker` with a `[ScopedService]` attribute and scan with `AddClasses(c => c.AssignableTo<IServiceMarker>().Where(t => t.GetCustomAttribute<ScopedServiceAttribute>() is not null))`, assigning different lifetimes per attribute.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Решение `OrderProcessing` собирается командой `dotnet build` без ошибок и предупреждений.
- [ ] (RU) `dotnet run --project src/OrderProcessing.App` выводит строки декораторов, сервиса и внешнего клиента.
- [ ] (RU) `dotnet test` зелёный; тест доказывает применение декораторов через `SpyLogger`.
- [ ] (RU) Маркерный интерфейс применён только к реальным сервисам, не к декораторам.
- [ ] (RU) `RegistrationStrategy.Skip` задан явно; цепочка `Decorate` построена в порядке `Retry → Cache → Logging`.
- [ ] (RU) Внешний тип изолирован за адаптером в Composition Root.
- [ ] (EN) Solution `OrderProcessing` builds with `dotnet build` without errors or warnings.
- [ ] (EN) `dotnet run --project src/OrderProcessing.App` prints decorator, service, and external-client lines.
- [ ] (EN) `dotnet test` is green; the test proves decoration through `SpyLogger`.
- [ ] (EN) The marker interface is applied only to real services, not to decorators.
- [ ] (EN) `RegistrationStrategy.Skip` is explicit; the `Decorate` chain is `Retry → Cache → Logging`.
- [ ] (EN) The external type is isolated behind an adapter in the Composition Root.

#### Ресурсы / Resources
- Scrutor на GitHub — https://github.com/khellang/Scrutor
- Microsoft Learn — Dependency injection in .NET — https://learn.microsoft.com/dotnet/core/extensions/dependency-injection
- Microsoft Learn — Dependency injection guidelines — https://learn.microsoft.com/dotnet/core/extensions/dependency-injection-guidelines
- Martin Fowler — Decorator pattern & injection — https://martinfowler.com/articles/injection.html#DealingWithSimilarServicesOrANeedForDecoration
- Scrutor README — Decorate / RegisterAdapter API — https://github.com/khellang/Scrutor#decoration

---

[← К уроку M16-L03](lesson-M16-L03-scrutor.md) | [⬆ К модулю M16](../README.md) | [Следующее ДЗ →](homework-M16-L04-layered-clean-architecture.md)
