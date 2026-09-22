---
[← К уроку M14-L10](lesson-M14-L10-httpclient-factory.md) | [⬆ К модулю M14](../README.md) | [Следующее ДЗ →]()
---

### Домашнее задание M14-L10: HttpClient, IHttpClientFactory, resilient HTTP (Polly) / Homework M14-L10: HttpClient, IHttpClientFactory, resilient HTTP (Polly)

**Урок / Lesson:** M14-L10
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться правильно создавать и конфигурировать исходящие HTTP-клиенты через `IHttpClientFactory` (typed + named clients), подключать устойчивость с помощью Polly v8 (`Microsoft.Extensions.Http.Resilience`) — retry с exponential backoff и jitter, circuit breaker, timeout — и избежать классических ловушек: исчерпания сокетов, залипшего DNS, ретраев неидемпотентных POST и лавины запросов на упавший сервис. (EN) Learn to create and configure outbound HTTP clients correctly through `IHttpClientFactory` (typed + named clients), wire in resilience with Polly v8 (`Microsoft.Extensions.Http.Resilience`) — retry with exponential backoff and jitter, circuit breaker, timeout — and avoid the classic traps: socket exhaustion, stale DNS, retrying non-idempotent POSTs, and request avalanches against a failing service.

#### Связь с уроком / Connection to the lesson
(RU) Урок вводит антипаттерны `new HttpClient()` на запрос и общего `static HttpClient`, объясняет, почему `IHttpClientFactory` решает обе проблемы (переиспользование handler'ов с ротацией раз в 2 минуты), показывает три способа потребления фабрики и интеграцию Polly v8 через `AddStandardResilienceHandler()`. ДЗ закрепляет всё это на реальном сценарии: клиент к стороннему API с нестабильной сетью и падающими зависимостями.
(EN) The lesson introduces the anti-patterns of `new HttpClient()` per request and a single shared `static HttpClient`, explains why `IHttpClientFactory` solves both problems (handler reuse with 2-minute rotation), shows the three ways to consume the factory, and the Polly v8 integration via `AddStandardResilienceHandler()`. This homework cements all of it on a realistic scenario: a client to a third-party API with an unstable network and failing dependencies.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Вы — backend-разработчик платформы «CourseHub», которая агрегирует данные из нескольких внешних источников: публичный API GitHub (профили пользователей), сервис погоды и внутренний микросервис «Notifications», который периодически «падает» под нагрузкой и отвечает `503 Service Unavailable` или таймаутится. Команда устала от того, что локально всё работает идеально, а в проде при первом же всплеске трафика приложение начинает выбрасывать `SocketException`, тормозит и ложит зависимые сервисы. Анализ показал три проблемы: (1) в коде по историческим причинам используется `new HttpClient()` на каждый вызов — исчерпание сокетов в `TIME_WAIT`; (2) DNS-записи внешних API иногда меняются, а статические клиенты держат старые адреса навсегда; (3) нет ни retry, ни circuit breaker — любой временный сбой превращается в ошибку 500 для конечного пользователя, а упавший микросервис получает лавину повторных запросов, от которой не может восстановиться.

Ваша задача — отрефакторить слой интеграции так, чтобы он соответствовал best practices из урока M14-L10: использовать `IHttpClientFactory` с typed-клиентами для типовых интеграций и named-клиентом там, где нужен императивный контроль, конфигурировать `BaseAddress`, `Timeout` и заголовки при регистрации, прокидывать `CancellationToken` во все вызовы и подключать resilience-стратегии (retry + circuit breaker + timeout) через `Microsoft.Extensions.Http.Resilience`. Дополнительно нужно решить вопрос идемпотентности для одного POST-вызова (отправка уведомления), чтобы ретраи не порождали дублей.

#### Что нужно сделать (пошагово)
1. Создайте solution и три проекта:
   ```bash
   dotnet new sln -n CourseHub.Integration
   dotnet new classlib -n CourseHub.Integration.Core -o src/CourseHub.Integration.Core -f net8.0
   dotnet new xunit -n CourseHub.Integration.Tests -o tests/CourseHub.Integration.Tests -f net8.0
   dotnet new console -n CourseHub.Integration.App -o src/CourseHub.Integration.App -f net8.0
   dotnet sln add src/CourseHub.Integration.Core src/CourseHub.Integration.App tests/CourseHub.Integration.Tests
   dotnet add src/CourseHub.Integration.Core package Microsoft.Extensions.Http
   dotnet add src/CourseHub.Integration.Core package Microsoft.Extensions.Http.Resilience
   dotnet add src/CourseHub.Integration.Core package Microsoft.Extensions.DependencyInjection
   dotnet add src/CourseHub.Integration.Core package Microsoft.Extensions.Logging.Console
   dotnet add src/CourseHub.Integration.App reference src/CourseHub.Integration.Core
   dotnet add tests/CourseHub.Integration.Tests reference src/CourseHub.Integration.Core
   ```
2. В `CourseHub.Integration.Core` создайте модели записей: `GitHubUser(string Login, int Id, string HtmlUrl)`, `WeatherForecast(DateTimeOffset Time, double TemperatureC, string Summary)`, `NotificationResult(bool Accepted, string IdempotencyKey)` — все как `sealed record`.
3. Создайте typed-клиент `GitHubClient` с методом `GetUserAsync(string login, CancellationToken ct)`, использующим `GetFromJsonAsync<GitHubUser>`. Создайте typed-клиент `WeatherClient` с методом `GetCurrentAsync(string station, CancellationToken ct)`. Каждый клиент получает `HttpClient` через конструктор.
4. Создайте сервис `NotificationService`, который использует **named-клиент** `"notifications"` (достаётся через `IHttpClientFactory.CreateClient("notifications")`) и метод `SendAsync(NotificationRequest req, CancellationToken ct)`. Этот метод выполняет POST с заголовком `Idempotency-Key` — чтобы ретраи были безопасны.
5. Напишите метод расширения `AddCourseHubIntegration(this IServiceCollection services, IConfiguration cfg)`, который регистрирует оба typed-клиента и named-клиент. Для `GitHubClient` и `WeatherClient` подключите `.AddStandardResilienceHandler()`. Для named-клиента `"notifications"` подключите ручной конвейер через `.AddResilienceHandler("notif", pipeline => ...)` с retry (3 попытки, exponential + jitter) и circuit breaker (sampling 30 с, failure ratio 0.5, minimum throughput 8, break 15 с). Назначьте `Timeout = 30s` и `User-Agent: coursehub/1.0`.
6. В `CourseHub.Integration.App` (top-level statements) соберите `Host`, вызовите `AddCourseHubIntegration`, получите `GitHubClient`, вызовите `GetUserAsync("dotnet")` и распечатайте результат. Используйте `CancellationToken` с таймаутом 20 секунд.
7. В тестах используйте `HttpClientHandler`-заглушку или `WireMock.Net`/`HttpMessageHandler`-наследника, который первые два вызова GitHub возвращает `503`, а третий — `200 OK` с JSON. Утвердите, что typed-клиент с `AddStandardResilienceHandler` успешно отдаёт результат после ретраев. Затем симулируйте 10 подряд ошибок `500` и убедитесь, что circuit breaker разомкнул цепь — следующий вызов падает быстро с `BrokenCircuitException` (или `HttpRequestException`-обёрткой), не доходя до handler'а.
8. Запустите `dotnet build` и `dotnet test` — всё должно быть зелёным.
9. Запустите `dotnet run --project src/CourseHub.Integration.App` — в консоли должен появиться профиль пользователя `dotnet` с `Login`, `Id`, `HtmlUrl`.

Ожидаемые выводы: `dotnet build` → `Build succeeded. 0 Warning(s) 0 Error(s)`. `dotnet test` → `Passed: 2-3`. `dotnet run` → строка вида `dotnet | 9919 | https://github.com/dotnet`.

#### Требования к решению
- Целевой фреймворк `net8.0`, язык C# 12: разрешены top-level statements, `record`, collection expressions, pattern matching, `required`, `init`, raw string literals где уместно.
- Весь исходящий HTTP-трафик идёт исключительно через `IHttpClientFactory`. Запрещены `new HttpClient()` и общий `static HttpClient` в горячем пути. Допускается `new HttpClient(handler, disposeHandler: false)` только в тестах и только для оборачивания stub-handler'а.
- Для типовых интеграций (GitHub, Weather) — typed-клиенты. Для императивного/динамического сценария (Notifications) — named-клиент через `IHttpClientFactory.CreateClient(...)`. Выбор должен быть обоснован комментарием.
- Конфигурация (`BaseAddress`, `Timeout`, заголовки) задаётся при регистрации в DI, а не в точке вызова.
- Resilience: как минимум один клиент использует `AddStandardResilienceHandler()`, как минимум один — кастомный `AddResilienceHandler` с явными retry + circuit breaker. Таймаут должен быть на уровне клиента и/или стратегии.
- `CancellationToken` передаётся во все async-методы — без `default`-заглушек в публичном API.
- POST-запрос в `NotificationService` обязан быть идемпотентным: заголовок `Idempotency-Key` (значение —Guid из запроса), что явно разрешает безопасный retry.
- Код компилируется без warning-ов уровня error и проходит тесты.

#### Тонкости и подводные камни
- **Socket exhaustion.** Если вы случайно оставите `using var http = new HttpClient();` в горячем цикле — на Windows сокеты уйдут в `TIME_WAIT` на ~240 секунд и при нагрузке порты закончатся. Фабрика решает это пулом handler'ов. Не вызывайте `Dispose()` на `HttpClient`, полученном из `CreateClient()` — он лёгкий, его сборка ничего не стоит, а handler живёт в пуле.
- **DNS caching.** Один `static HttpClient` навсегда кэширует DNS. Фабрика пересоздаёт handler каждые 2 минуты (`HandlerLifetime`), поэтому DNS периодически обновляется. Не снижайте `HandlerLifetime` до секунд без причины — это убивает переиспользование.
- **Порядок стратегий.** В конвейере `HttpMessageHandler` стратегия-обёртка должна быть ближе к вызову, чем ваши кастомные handler'ы. `AddStandardResilienceHandler` добавляет retry, circuit breaker, timeout и hedge в правильном порядке — не пытайтесь «обернуть» его ещё одним своим retry.
- **Идемпотентность POST.** Retry безопасен для GET/HEAD/PUT и опасен для POST без idempotency-key: повтор может создать второе уведомление. Включайте `Idempotency-Key`, либо отключайте retry для неидемпотентных вызовов.
- **Circuit breaker не отменяет retry.** Разомкнутая цепь бросает `BrokenCircuitException` немедленно — это и есть защита от лавины. Не «глушите» её blanket-`catch`, иначе потеряете сигнал.
- **Чтение тела до проверки статуса.** Если читать `ReadAsStringAsync` до `EnsureSuccessStatusCode`, можно случайно проглотить ошибку. Используйте `GetFromJsonAsync` или сначала проверьте `response.IsSuccessStatusCode`.
- **Timeout дублирования.** `HttpClient.Timeout` и таймаут resilience-стратегии могут конфликтовать. Обычно оставляют `HttpClient.Timeout = Infinite` или согласовывают значения, чтобы не получать `TaskCanceledException` от клиента раньше, чем сработает стратегия.
- **`CancellationToken` vs retry.** Если токен отмены сработал во время задержки ретрая, Polly корректно пробрасывает `OperationCanceledException` — не глушите его.

#### Критерии приёмки
- [ ] Созданы три проекта, solution собирается без ошибок и warning-ов.
- [ ] Вся работа с HTTP идёт через `IHttpClientFactory` — `grep` по коду не находит `new HttpClient()` в горячем пути.
- [ ] Есть как минимум два typed-клиента (`GitHubClient`, `WeatherClient`) и один named-клиент (`"notifications"`).
- [ ] `BaseAddress`, `Timeout` и заголовки заданы при регистрации в DI.
- [ ] Для GitHub/Weather применён `AddStandardResilienceHandler()`.
- [ ] Для Notifications применён кастомный `AddResilienceHandler` с retry + circuit breaker + timeout.
- [ ] Retry использует exponential backoff + jitter; max attempts ≤ 4.
- [ ] Circuit breaker сконфигурирован с явными параметрами (sampling, failure ratio, minimum throughput, break duration).
- [ ] `NotificationService.SendAsync` отправляет заголовок `Idempotency-Key` и обосновывает безопасный retry.
- [ ] `CancellationToken` передаётся во все async-вызовы.
- [ ] Тест «503, 503, 200 → успех после ретраев» проходит.
- [ ] Тест «10×500 → circuit open → быстрый fail без вызова handler'а» проходит.
- [ ] `dotnet run` выводит профиль пользователя `dotnet`.
- [ ] В коде нет «бог»-клиента на все API — каждый typed-клиент сфокусирован.
- [ ] В README/комментариях объяснено, почему выбран typed vs named стиль для каждого клиента.

#### Подсказки (без прямого ответа)
- Для stub-handler'а в тестах наследуйте `HttpMessageHandler` и переопределите `SendAsync`, используя счётчик вызовов и `Interlocked.Increment`.
- Для проверки circuit breaker ловите `HttpRequestException` с внутренним `BrokenCircuitException` (в v8 обёртка может быть `BrokenCircuitException` напрямую — читайте тип).
- `AddStandardResilienceHandler` уже включает таймаут 30 с по умолчанию — не дублируйте его без согласования.
- Не забудьте `User-Agent`: GitHub API возвращает `403`, если заголовок отсутствует.
- Для idempotency-key генерируйте `Guid` в `NotificationRequest` через `required`-свойство с `init`.

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 — CourseHub integration layer
// Слой интеграции с внешними API через IHttpClientFactory + Polly v8

using System.Net.Http.Headers;
using System.Net.Http.Json;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Http.Resilience;
using Polly;

namespace CourseHub.Integration.Core;

// ---- Модели / Models ----
public sealed record GitHubUser(string Login, int Id, string HtmlUrl);
public sealed record WeatherForecast(DateTimeOffset Time, double TemperatureC, string Summary);
public sealed record NotificationRequest(string Channel, string Message)
{
    public required Guid IdempotencyKey { get; init; } = Guid.NewGuid();
}
public sealed record NotificationResult(bool Accepted, string IdempotencyKey);

// ---- Typed-клиент GitHub ----
public sealed class GitHubClient
{
    private readonly HttpClient _http;
    public GitHubClient(HttpClient http) => _http = http;

    public async Task<GitHubUser?> GetUserAsync(string login, CancellationToken ct = default)
        => await _http.GetFromJsonAsync<GitHubUser>($"users/{Uri.EscapeDataString(login)}", ct);
}

// ---- Typed-клиент Weather ----
public sealed class WeatherClient
{
    private readonly HttpClient _http;
    public WeatherClient(HttpClient http) => _http = http;

    public async Task<WeatherForecast?> GetCurrentAsync(string station, CancellationToken ct = default)
        => await _http.GetFromJsonAsync<WeatherForecast>($"stations/{Uri.EscapeDataString(station)}/observations/latest", ct);
}

// ---- Сервис на named-клиенте с идемпотентным POST ----
public sealed class NotificationService
{
    private readonly IHttpClientFactory _factory;
    public NotificationService(IHttpClientFactory factory) => _factory = factory;

    public async Task<NotificationResult?> SendAsync(NotificationRequest req, CancellationToken ct = default)
    {
        // Императивный контроль: достаём именованного клиента при каждом вызове.
        // Imperative control: resolve a named client on each call.
        var http = _factory.CreateClient("notifications");
        using var msg = new HttpRequestMessage(HttpMethod.Post, "v1/notify")
        {
            Content = JsonContent.Create(req)
        };
        // Idempotency-Key делает retry безопасным даже для POST.
        // Idempotency-Key makes retry safe even for POST.
        msg.Headers.Add("Idempotency-Key", req.IdempotencyKey.ToString());
        using var resp = await http.SendAsync(msg, ct);
        if (!resp.IsSuccessStatusCode) return null;
        return await resp.Content.ReadFromJsonAsync<NotificationResult>(ct);
    }
}

// ---- Регистрация в DI / DI registration ----
public static class CourseHubIntegrationExtensions
{
    public static IServiceCollection AddCourseHubIntegration(this IServiceCollection services, IConfiguration cfg)
    {
        // Typed clients: конфигурируем один раз при регистрации.
        // Typed clients: configure once at registration.
        services.AddHttpClient<GitHubClient>((sp, c) =>
        {
            c.BaseAddress = new Uri("https://api.github.com");
            c.DefaultRequestHeaders.UserAgent.ParseAdd("coursehub/1.0");
            c.Timeout = TimeSpan.FromSeconds(30);
        })
        .AddStandardResilienceHandler(); // retry + circuit breaker + timeout + hedge (Polly v8)

        services.AddHttpClient<WeatherClient>((sp, c) =>
        {
            c.BaseAddress = new Uri("https://api.weather.gov");
            c.DefaultRequestHeaders.UserAgent.ParseAdd("coursehub/1.0");
            c.Timeout = TimeSpan.FromSeconds(30);
        })
        .AddStandardResilienceHandler();

        // Named client: когда нужна императивная логика и/или динамический BaseAddress.
        // Named client: when imperative logic and/or dynamic BaseAddress is needed.
        services.AddHttpClient("notifications", (sp, c) =>
        {
            c.BaseAddress = new Uri(cfg["Notifications:BaseUrl"] ?? "https://notif.internal.local");
            c.DefaultRequestHeaders.UserAgent.ParseAdd("coursehub/1.0");
            c.Timeout = TimeSpan.FromSeconds(20);
        })
        .AddResilienceHandler("notif", pipeline =>
        {
            pipeline.AddRetry(new HttpRetryStrategyOptions
            {
                MaxRetryAttempts = 3,
                BackoffType = DelayBackoffType.Exponential,
                UseJitter = true
            })
            .AddCircuitBreaker(new HttpCircuitBreakerStrategyOptions
            {
                SamplingDuration = TimeSpan.FromSeconds(30),
                FailureRatio = 0.5,
                MinimumThroughput = 8,
                BreakDuration = TimeSpan.FromSeconds(15)
            })
            .AddTimeout(TimeSpan.FromSeconds(20));
        });

        services.AddSingleton<NotificationService>();
        return services;
    }
}
```

```csharp
// Program.cs — CourseHub.Integration.App (top-level statements)
// Точка входа демонстрирует вызов typed-клиента.

using CourseHub.Integration.Core;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

var builder = Host.CreateApplicationBuilder(args);
builder.Services.AddCourseHubIntegration(builder.Configuration);
var host = builder.Build();

using var scope = host.Services.CreateScope();
var github = scope.ServiceProvider.GetRequiredService<GitHubClient>();

using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(20));
var user = await github.GetUserAsync("dotnet", cts.Token);
Console.WriteLine($"{user?.Login} | {user?.Id} | {user?.HtmlUrl}");

await host.RunAsync(cts.Token);
```

Разбор по строкам. `GitHubClient` и `WeatherClient` — типовые typed-клиенты: конструктор принимает готовый `HttpClient`, который контейнер конфигурирует сам. Это решает сразу две проблемы из урока: нет `new HttpClient()` на запрос (нет исчерпания сокетов) и нет статического клиента (DNS обновляется за счёт ротации handler'а фабрикой). `GetFromJsonAsync` берётся из `System.Net.Http.Json` — он сам проверяет статус и десериализует, поэтому мы не наступаем на ловушку «прочитали тело до `EnsureSuccessStatusCode`». `AddStandardResilienceHandler()` подключает готовый конвейер Polly v8 — retry с exponential backoff + jitter, circuit breaker, timeout и hedge; порядок стратегий уже корректен. Для `NotificationService` выбран **named-клиент**, потому что адрес определяется из конфигурации и/или может меняться в рантайме — это императивный сценарий, где typed-клиент был бы избыточен. POST-вызову добавлен `Idempotency-Key` — это ключевой момент урока: retry безопасен только для идемпотентных операций, а заголовок делает POST идемпотентным на стороне сервера. Кастомный `AddResilienceHandler("notif", ...)` показывает ручную сборку конвейера: `HttpRetryStrategyOptions` с `MaxRetryAttempts = 3` и `UseJitter = true` (jitter критичен — без него синхронные ретраи создают «стадо»), `HttpCircuitBreakerStrategyOptions` с `FailureRatio = 0.5` и `MinimumThroughput = 8` — breaker разомкнётся, если в окне 30 с хотя бы 8 запросов и доля ошибок ≥ 50 %. `AddTimeout` ограничивает общее время. В `Program.cs` `CancellationToken` создаётся с таймаутом 20 с и прокидывается в `GetUserAsync` — это best practice урока: токен должен течь во все async-вызовы. `CreateScope` нужен, потому что typed-клиенты обычно зарегистрированы как scoped (по умолчанию `AddHttpClient<T>` регистрирует `T` как transient, но сам `HttpClient` живёт в scope-生命周期 handler'ов) — безопаснее работать из scope.

#### Задания на углубление (бонус)
1. **Hedging.** Включите hedging-стратегию для `GitHubClient` через `AddStandardHedgingHandler` и измерьте, как меняется p95 latency при искусственной задержке handler'а.
2. **Метрики.** Подключите `Microsoft.Extensions.Diagnostics` и снимайте метрики Polly (` resilience_polly_retry_count`, `circuit_breaker_state`) — выведите их в консоль.
3. **Twin тест.** Напишите тест, который сравнивает производительность двух версий: `new HttpClient()` на запрос vs `IHttpClientFactory`, и покажите, что первая быстрее исчерпывает порты при нагрузке в 1000 RPS.
4. **Динамический named-клиент.** Реализуйте фабрику клиентов, которая выбирает BaseAddress по тенантy из запроса, и убедитесь, что circuit breaker общий для всех тенантов или изолирован — на ваше усмотрение, но обоснуйте выбор.

---

## Statement in English / Постановка на английском

#### Context & motivation
You are a backend engineer at "CourseHub", a platform that aggregates data from several external sources: the public GitHub API (user profiles), a weather service, and an internal "Notifications" microservice that periodically falls over under load and returns `503 Service Unavailable` or times out. The team is tired of a recurring pattern: everything works beautifully locally, but the moment traffic spikes in production the app starts throwing `SocketException`, slows to a crawl, and takes dependent services down with it. A post-mortem surfaced three problems: (1) legacy code uses `new HttpClient()` per call — sockets exhaust via `TIME_WAIT`; (2) DNS records of the external APIs occasionally change, but static clients hold the old addresses forever; (3) there is no retry and no circuit breaker — any transient failure becomes a 500 for the end user, and a failing microservice receives an avalanche of retried requests it cannot recover from.

Your job is to refactor the integration layer so it matches the best practices from lesson M14-L10: use `IHttpClientFactory` with typed clients for the typical integrations and a named client where imperative control is required, configure `BaseAddress`, `Timeout`, and headers at registration time, propagate `CancellationToken` into every call, and wire in resilience strategies (retry + circuit breaker + timeout) through `Microsoft.Extensions.Http.Resilience`. Additionally, you must solve idempotency for one POST call (sending a notification) so retries do not create duplicates.

#### What to do step by step
1. Create a solution and three projects:
   ```bash
   dotnet new sln -n CourseHub.Integration
   dotnet new classlib -n CourseHub.Integration.Core -o src/CourseHub.Integration.Core -f net8.0
   dotnet new xunit -n CourseHub.Integration.Tests -o tests/CourseHub.Integration.Tests -f net8.0
   dotnet new console -n CourseHub.Integration.App -o src/CourseHub.Integration.App -f net8.0
   dotnet sln add src/CourseHub.Integration.Core src/CourseHub.Integration.App tests/CourseHub.Integration.Tests
   dotnet add src/CourseHub.Integration.Core package Microsoft.Extensions.Http
   dotnet add src/CourseHub.Integration.Core package Microsoft.Extensions.Http.Resilience
   dotnet add src/CourseHub.Integration.Core package Microsoft.Extensions.DependencyInjection
   dotnet add src/CourseHub.Integration.Core package Microsoft.Extensions.Logging.Console
   dotnet add src/CourseHub.Integration.App reference src/CourseHub.Integration.Core
   dotnet add tests/CourseHub.Integration.Tests reference src/CourseHub.Integration.Core
   ```
2. In `CourseHub.Integration.Core`, create record models: `GitHubUser(string Login, int Id, string HtmlUrl)`, `WeatherForecast(DateTimeOffset Time, double TemperatureC, string Summary)`, `NotificationResult(bool Accepted, string IdempotencyKey)` — all as `sealed record`.
3. Create a typed client `GitHubClient` with `GetUserAsync(string login, CancellationToken ct)` that uses `GetFromJsonAsync<GitHubUser>`. Create a typed client `WeatherClient` with `GetCurrentAsync(string station, CancellationToken ct)`. Each client receives an `HttpClient` via its constructor.
4. Create a `NotificationService` that uses a **named client** `"notifications"` (resolved through `IHttpClientFactory.CreateClient("notifications")`) and a method `SendAsync(NotificationRequest req, CancellationToken ct)`. The method performs a POST with an `Idempotency-Key` header so retries are safe.
5. Write an extension method `AddCourseHubIntegration(this IServiceCollection services, IConfiguration cfg)` that registers both typed clients and the named client. For `GitHubClient` and `WeatherClient`, attach `.AddStandardResilienceHandler()`. For the named client `"notifications"`, attach a custom pipeline through `.AddResilienceHandler("notif", pipeline => ...)` with retry (3 attempts, exponential + jitter) and a circuit breaker (sampling 30 s, failure ratio 0.5, minimum throughput 8, break 15 s). Set `Timeout = 30s` and `User-Agent: coursehub/1.0`.
6. In `CourseHub.Integration.App` (top-level statements), build a `Host`, call `AddCourseHubIntegration`, resolve `GitHubClient`, call `GetUserAsync("dotnet")`, and print the result. Use a `CancellationToken` with a 20-second timeout.
7. In the tests, use a stub `HttpMessageHandler` (or `WireMock.Net`) whose first two GitHub calls return `503` and the third returns `200 OK` with JSON. Assert that the typed client with `AddStandardResilienceHandler` successfully returns the result after retries. Then simulate 10 consecutive `500` errors and verify the circuit breaker opens — the next call fails fast with a `BrokenCircuitException` (or its `HttpRequestException` wrapper) without reaching the handler.
8. Run `dotnet build` and `dotnet test` — everything must be green.
9. Run `dotnet run --project src/CourseHub.Integration.App` — the console must show the profile of the `dotnet` user.

Expected outputs: `dotnet build` → `Build succeeded. 0 Warning(s) 0 Error(s)`. `dotnet test` → `Passed: 2-3`. `dotnet run` → a line such as `dotnet | 9919 | https://github.com/dotnet`.

#### Requirements
- Target framework `net8.0`, language C# 12: top-level statements, `record`, collection expressions, pattern matching, `required`, `init`, raw string literals where appropriate are all allowed.
- All outbound HTTP traffic flows exclusively through `IHttpClientFactory`. `new HttpClient()` and a shared `static HttpClient` are forbidden in the hot path. `new HttpClient(handler, disposeHandler: false)` is allowed only in tests and only to wrap a stub handler.
- Typical integrations (GitHub, Weather) use typed clients. The imperative/dynamic scenario (Notifications) uses a named client via `IHttpClientFactory.CreateClient(...)`. The choice must be justified in a comment.
- Configuration (`BaseAddress`, `Timeout`, headers) is set at DI registration time, not at call sites.
- Resilience: at least one client uses `AddStandardResilienceHandler()`, at least one uses a custom `AddResilienceHandler` with explicit retry + circuit breaker. A timeout must be present at the client and/or strategy level.
- `CancellationToken` is propagated into every async method — no `default` placeholders in public APIs.
- The POST request in `NotificationService` must be idempotent: an `Idempotency-Key` header (value: a `Guid` from the request) explicitly authorises safe retry.
- The code compiles without error-level warnings and passes tests.

#### Pitfalls
- **Socket exhaustion.** If you accidentally leave a `using var http = new HttpClient();` in a hot loop, sockets on Windows will go to `TIME_WAIT` for ~240 seconds and, under load, ephemeral ports will run out. The factory solves this with a handler pool. Do not call `Dispose()` on an `HttpClient` obtained from `CreateClient()` — it is cheap, and the handler lives in the pool.
- **DNS caching.** A single `static HttpClient` caches DNS forever. The factory rotates handlers every 2 minutes (`HandlerLifetime`), so DNS is periodically refreshed. Do not lower `HandlerLifetime` to a few seconds without reason — it defeats reuse.
- **Strategy ordering.** In the `HttpMessageHandler` pipeline, the resilience wrapper must sit closer to the call than your custom handlers. `AddStandardResilienceHandler` adds retry, circuit breaker, timeout, and hedge in the correct order — do not wrap it with another retry of your own.
- **POST idempotency.** Retry is safe for GET/HEAD/PUT and unsafe for POST without an idempotency key: a retry may create a second notification. Include `Idempotency-Key`, or disable retry for non-idempotent calls.
- **Circuit breaker does not cancel retry.** An open circuit throws `BrokenCircuitException` immediately — that is exactly the avalanche protection. Do not blanket-`catch` it or you lose the signal.
- **Reading the body before checking status.** If you call `ReadAsStringAsync` before `EnsureSuccessStatusCode`, you may silently swallow an error. Use `GetFromJsonAsync` or check `response.IsSuccessStatusCode` first.
- **Timeout duplication.** `HttpClient.Timeout` and the resilience strategy timeout can conflict. Typically you leave `HttpClient.Timeout = Infinite` or align the values so you do not get a `TaskCanceledException` from the client before the strategy fires.
- **`CancellationToken` vs retry.** If the token fires during a retry delay, Polly correctly propagates `OperationCanceledException` — do not swallow it.

#### Acceptance criteria
- [ ] Three projects created; the solution builds with no errors and no warnings.
- [ ] All HTTP work goes through `IHttpClientFactory` — `grep` finds no `new HttpClient()` in the hot path.
- [ ] At least two typed clients (`GitHubClient`, `WeatherClient`) and one named client (`"notifications"`) exist.
- [ ] `BaseAddress`, `Timeout`, and headers are set at DI registration.
- [ ] GitHub/Weather use `AddStandardResilienceHandler()`.
- [ ] Notifications use a custom `AddResilienceHandler` with retry + circuit breaker + timeout.
- [ ] Retry uses exponential backoff + jitter; max attempts ≤ 4.
- [ ] Circuit breaker has explicit parameters (sampling, failure ratio, minimum throughput, break duration).
- [ ] `NotificationService.SendAsync` sends the `Idempotency-Key` header and justifies safe retry.
- [ ] `CancellationToken` is propagated into every async call.
- [ ] The "503, 503, 200 → success after retries" test passes.
- [ ] The "10×500 → circuit open → fast fail without reaching the handler" test passes.
- [ ] `dotnet run` prints the `dotnet` user profile.
- [ ] No "god" client for all APIs — each typed client is focused.
- [ ] README/comments explain the typed-vs-named choice for each client.

#### Hints (no direct answer)
- For a stub handler in tests, derive from `HttpMessageHandler` and override `SendAsync`, using a call counter with `Interlocked.Increment`.
- To verify the circuit breaker, catch `HttpRequestException` with an inner `BrokenCircuitException` (in v8 the wrapper may be `BrokenCircuitException` directly — inspect the type).
- `AddStandardResilienceHandler` already includes a 30 s timeout by default — do not duplicate it without alignment.
- Do not forget `User-Agent`: the GitHub API returns `403` without it.
- For the idempotency key, generate a `Guid` in `NotificationRequest` via a `required` property with `init`.

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8 — CourseHub integration layer
// Integration layer with external APIs via IHttpClientFactory + Polly v8

using System.Net.Http.Headers;
using System.Net.Http.Json;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Http.Resilience;
using Polly;

namespace CourseHub.Integration.Core;

// ---- Models ----
public sealed record GitHubUser(string Login, int Id, string HtmlUrl);
public sealed record WeatherForecast(DateTimeOffset Time, double TemperatureC, string Summary);
public sealed record NotificationRequest(string Channel, string Message)
{
    public required Guid IdempotencyKey { get; init; } = Guid.NewGuid();
}
public sealed record NotificationResult(bool Accepted, string IdempotencyKey);

// ---- Typed client for GitHub ----
public sealed class GitHubClient
{
    private readonly HttpClient _http;
    public GitHubClient(HttpClient http) => _http = http;

    public async Task<GitHubUser?> GetUserAsync(string login, CancellationToken ct = default)
        => await _http.GetFromJsonAsync<GitHubUser>($"users/{Uri.EscapeDataString(login)}", ct);
}

// ---- Typed client for Weather ----
public sealed class WeatherClient
{
    private readonly HttpClient _http;
    public WeatherClient(HttpClient http) => _http = http;

    public async Task<WeatherForecast?> GetCurrentAsync(string station, CancellationToken ct = default)
        => await _http.GetFromJsonAsync<WeatherForecast>($"stations/{Uri.EscapeDataString(station)}/observations/latest", ct);
}

// ---- Service on a named client with idempotent POST ----
public sealed class NotificationService
{
    private readonly IHttpClientFactory _factory;
    public NotificationService(IHttpClientFactory factory) => _factory = factory;

    public async Task<NotificationResult?> SendAsync(NotificationRequest req, CancellationToken ct = default)
    {
        // Imperative control: resolve a named client on each call.
        var http = _factory.CreateClient("notifications");
        using var msg = new HttpRequestMessage(HttpMethod.Post, "v1/notify")
        {
            Content = JsonContent.Create(req)
        };
        // Idempotency-Key makes retry safe even for POST.
        msg.Headers.Add("Idempotency-Key", req.IdempotencyKey.ToString());
        using var resp = await http.SendAsync(msg, ct);
        if (!resp.IsSuccessStatusCode) return null;
        return await resp.Content.ReadFromJsonAsync<NotificationResult>(ct);
    }
}

// ---- DI registration ----
public static class CourseHubIntegrationExtensions
{
    public static IServiceCollection AddCourseHubIntegration(this IServiceCollection services, IConfiguration cfg)
    {
        // Typed clients: configured once at registration.
        services.AddHttpClient<GitHubClient>((sp, c) =>
        {
            c.BaseAddress = new Uri("https://api.github.com");
            c.DefaultRequestHeaders.UserAgent.ParseAdd("coursehub/1.0");
            c.Timeout = TimeSpan.FromSeconds(30);
        })
        .AddStandardResilienceHandler(); // retry + circuit breaker + timeout + hedge (Polly v8)

        services.AddHttpClient<WeatherClient>((sp, c) =>
        {
            c.BaseAddress = new Uri("https://api.weather.gov");
            c.DefaultRequestHeaders.UserAgent.ParseAdd("coursehub/1.0");
            c.Timeout = TimeSpan.FromSeconds(30);
        })
        .AddStandardResilienceHandler();

        // Named client: when imperative logic and/or dynamic BaseAddress is needed.
        services.AddHttpClient("notifications", (sp, c) =>
        {
            c.BaseAddress = new Uri(cfg["Notifications:BaseUrl"] ?? "https://notif.internal.local");
            c.DefaultRequestHeaders.UserAgent.ParseAdd("coursehub/1.0");
            c.Timeout = TimeSpan.FromSeconds(20);
        })
        .AddResilienceHandler("notif", pipeline =>
        {
            pipeline.AddRetry(new HttpRetryStrategyOptions
            {
                MaxRetryAttempts = 3,
                BackoffType = DelayBackoffType.Exponential,
                UseJitter = true
            })
            .AddCircuitBreaker(new HttpCircuitBreakerStrategyOptions
            {
                SamplingDuration = TimeSpan.FromSeconds(30),
                FailureRatio = 0.5,
                MinimumThroughput = 8,
                BreakDuration = TimeSpan.FromSeconds(15)
            })
            .AddTimeout(TimeSpan.FromSeconds(20));
        });

        services.AddSingleton<NotificationService>();
        return services;
    }
}
```

Line-by-line walk-through. `GitHubClient` and `WeatherClient` are typical typed clients: the constructor accepts a pre-configured `HttpClient` that the container wires up. This solves both lesson problems: no `new HttpClient()` per call (no socket exhaustion) and no static client (DNS is refreshed through the factory's handler rotation). `GetFromJsonAsync` from `System.Net.Http.Json` checks the status and deserialises in one call, so we avoid the "read the body before `EnsureSuccessStatusCode`" trap. `AddStandardResilienceHandler()` attaches a ready-made Polly v8 pipeline — retry with exponential backoff + jitter, circuit breaker, timeout, and hedge; the strategy order is already correct. For `NotificationService` we chose a **named client** because the address comes from configuration and may change at runtime — this is an imperative scenario where a typed client would be overkill. The POST carries an `Idempotency-Key` header — a key lesson point: retry is safe only for idempotent operations, and the header makes the POST idempotent on the server side. The custom `AddResilienceHandler("notif", ...)` shows a hand-built pipeline: `HttpRetryStrategyOptions` with `MaxRetryAttempts = 3` and `UseJitter = true` (jitter is critical — without it synchronous retries create a "thundering herd"), `HttpCircuitBreakerStrategyOptions` with `FailureRatio = 0.5` and `MinimumThroughput = 8` — the breaker opens if, in a 30 s window, at least 8 requests were made and the failure ratio is ≥ 50 %. `AddTimeout` bounds the total time. In `Program.cs`, the `CancellationToken` is created with a 20 s timeout and propagated into `GetUserAsync` — a lesson best practice: tokens must flow into every async call. `CreateScope` is needed because typed clients are scoped to a handler lifetime scope and it is safer to resolve them from a scope.

#### Going deeper (bonus)
1. **Hedging.** Enable the hedging strategy for `GitHubClient` via `AddStandardHedgingHandler` and measure how p95 latency changes when the stub handler introduces artificial delay.
2. **Metrics.** Plug in `Microsoft.Extensions.Diagnostics` and read Polly metrics (`resilience_polly_retry_count`, `circuit_breaker_state`) — print them to the console.
3. **Twin test.** Write a benchmark that compares `new HttpClient()` per call vs `IHttpClientFactory` and demonstrates that the former exhausts ports faster at 1000 RPS.
4. **Dynamic named client.** Implement a client factory that selects the BaseAddress per tenant from the request, and decide whether the circuit breaker is shared across tenants or isolated — justify your choice.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Solution из трёх проектов собирается без ошибок и warning-ов.
- [ ] (RU) Нет `new HttpClient()` в горячем пути; всё через `IHttpClientFactory`.
- [ ] (RU) Есть два typed-клиента и один named-клиент.
- [ ] (RU) `BaseAddress`, `Timeout`, заголовки заданы при регистрации.
- [ ] (RU) Подключены resilience-стратегии: `AddStandardResilienceHandler` + кастомный `AddResilienceHandler`.
- [ ] (RU) POST идемпотентен через `Idempotency-Key`.
- [ ] (RU) `CancellationToken` прокидывается во все async-вызовы.
- [ ] (RU) Тесты на retry и circuit breaker проходят.
- [ ] (EN) Three-project solution builds with no errors or warnings.
- [ ] (EN) No `new HttpClient()` in the hot path; everything via `IHttpClientFactory`.
- [ ] (EN) Two typed clients and one named client present.
- [ ] (EN) `BaseAddress`, `Timeout`, headers set at registration.
- [ ] (EN) Resilience strategies wired: `AddStandardResilienceHandler` + custom `AddResilienceHandler`.
- [ ] (EN) POST is idempotent via `Idempotency-Key`.
- [ ] (EN) `CancellationToken` flows into every async call.
- [ ] (EN) Retry and circuit-breaker tests pass.

#### Ресурсы / Resources
- [Microsoft Learn — Make HTTP requests using IHttpClientFactory in ASP.NET Core](https://learn.microsoft.com/dotnet/core/extensions/httpclient-factory)
- [Microsoft Learn — Microsoft.Extensions.Http.Resilience](https://learn.microsoft.com/dotnet/core/extensions/http-client-resilience)
- [Polly — resilience strategies for .NET](https://www.pollydocs.org/)
- [Microsoft Learn — Implement HTTP call retries with exponential backoff with Polly](https://learn.microsoft.com/dotnet/architecture/microservices/implement-resilient-applications/implement-http-call-retries-exponential-backoff-polly)
- [GitHub REST API — User-Agent requirement](https://docs.github.com/rest/overview/resources-in-the-rest-api#user-agent-required)
