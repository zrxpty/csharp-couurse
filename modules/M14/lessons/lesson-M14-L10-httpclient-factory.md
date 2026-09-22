[⬆ К модулю M14](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M14-L10: HttpClient, IHttpClientFactory, resilient HTTP (Polly) / HttpClient, IHttpClientFactory, resilient HTTP (Polly)

**Модуль / Module:** M14
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

`HttpClient` — это класс для отправки HTTP-запросов и получения ответов. На первый взгляд он выглядит простым: создал экземпляр, вызвал `GetAsync`, получил `HttpResponseMessage`. Но за этой простотой скрывается серьёзная ловушка — **исчерпание сокетов (socket exhaustion)**.

Каждый `HttpClient` оборачивает `HttpClientHandler`, который держит пул TCP-соединений. Когда вы создаёте `new HttpClient()` на каждый запрос и тут же уничтожаете его через `Dispose`,底层 соединения не закрываются мгновенно — они переходят в состояние `TIME_WAIT` и живут ещё около 240 секунд (по умолчанию на Windows). При высокой нагрузке свободные порты заканчиваются, и приложение начинает получать `SocketException` или зависать. Это та самая проблема, из-за которой «всё работало в тестах, а на проде упало».

Аналогия: представьте, что каждый звонок по телефону требует купить новый аппарат и выбросить его сразу после разговора. Дорого и глупо — ведь можно один аппарат и переиспользовать. Но если все в офисе делят один общий аппарат и одновременно пытаются говорить — тоже хаос. Нужен умный баланс.

**Антипаттерн №1:** `new HttpClient()` на каждый запрос → исчерпание сокетов.
**Антипаттерн №2:** статический одиночный `static HttpClient` для всего приложения → DNS-изменения не подхватываются (адрес кэшируется навсегда), плюс сложно конфигурировать разные клиенты по-разному.

Решение — **`IHttpClientFactory`**. Он управляет пулом `HttpMessageHandler`'ов: handler'ы переиспользуются (по умолчанию handler живёт 2 минуты, затем пересоздаётся), а обёртки `HttpClient` — лёгкие и создаются заново без штрафа. `IHttpClientFactory` решает обе проблемы: сокеты не утекают, а DNS периодически обновляется.

Есть три способа использовать фабрику:

1. **Named clients** — регистрируете клиента по имени: `builder.AddHttpClient("github", c => c.BaseAddress = ...);`. Достаёте через `factory.CreateClient("github")`. Удобно для небольшого числа клиентов, когда не хочется плодить типы.

2. **Typed clients** — регистрируете конкретный тип: `builder.AddHttpClient<GithubClient>(...)`. Контейнер сам внедряет готовый `HttpClient` в ваш сервис. Чистый DI, типобезопасность, тестируемость — рекомендуемый путь в большинстве случаев.

3. **Прямой `IHttpClientFactory`** — когда нужен императивный контроль.

**Resilience с Polly.** Сетевые вызовы ненадёжны: таймауты, 503, пакетные потери. Polly — библиотека resilience-стратегий. Главные две — `Retry` (повтор с задержкой/экспоненциальной отсрочкой) и `CircuitBreaker` (размыкатель цепи). `CircuitBreaker` считает подряд идущие ошибки; при превышении порога «размыкает» цепь на период `durationOfBreak`, сразу возвращая ошибку без обращения к серверу — даёт ему время восстановиться. Это как автомат пробок в щитке: короткое замыкание — выбило, восстановись, потом включишь обратно.

В современном .NET Polly интегрируется через `AddHttpClient(...).AddTransientHttpErrorPolicy(...)` или через `Microsoft.Extensions.Http.Resilience` (на базе Polly v8). Стратегии применяются через `HttpMessageHandler` — прозрачно для вашего кода.

#### Theory (EN)

`HttpClient` is the .NET class for sending HTTP requests and receiving responses. At first glance it looks trivial: instantiate it, call `GetAsync`, read the `HttpResponseMessage`. But beneath that simplicity hides a serious trap — **socket exhaustion**.

Each `HttpClient` wraps an `HttpClientHandler` that holds a pool of TCP connections. When you create `new HttpClient()` per request and immediately dispose it, the underlying sockets are not released instantly — they enter the `TIME_WAIT` state and linger for about 240 seconds (default on Windows). Under load, ephemeral ports run out, and your app starts getting `SocketException`s or hangs. This is the classic "it worked in tests, it died in prod" story.

Analogy: imagine every phone call required buying a brand-new handset and throwing it away right after the conversation. Expensive and silly — clearly you should reuse one handset. But if the entire office shares a single handset and everyone tries to talk at once — also chaos. You need a smart balance.

**Anti-pattern #1:** `new HttpClient()` per request → socket exhaustion.
**Anti-pattern #2:** a single static `static HttpClient` for the whole app → DNS changes are never picked up (the address is cached forever), and it is hard to configure different clients differently.

The fix is **`IHttpClientFactory`**. It manages a pool of `HttpMessageHandler` instances: handlers are reused (by default a handler lives 2 minutes, then is recreated), while the `HttpClient` wrappers are cheap and can be created over and over without penalty. `IHttpClientFactory` solves both problems: sockets do not leak, and DNS is periodically refreshed.

There are three ways to consume the factory:

1. **Named clients** — register a client by name: `builder.AddHttpClient("github", c => c.BaseAddress = ...);`. Resolve it via `factory.CreateClient("github")`. Handy when you have a few clients and do not want to invent a type for each.

2. **Typed clients** — register a concrete type: `builder.AddHttpClient<GithubClient>(...)`. The container injects a ready `HttpClient` into your service. Clean DI, type safety, testability — the recommended path in most cases.

3. **Direct `IHttpClientFactory`** — when you need imperative control.

**Resilience with Polly.** Network calls are unreliable: timeouts, 503s, packet loss. Polly is a resilience library. The two headline strategies are `Retry` (retry with delay/exponential backoff) and `CircuitBreaker`. The circuit breaker counts consecutive failures; when the threshold is exceeded it "opens" the circuit for `durationOfBreak` and immediately returns an error without contacting the server — giving it time to recover. Think of it as a fuse box: a short circuit trips the breaker, the system rests, then you reset it.

In modern .NET Polly is integrated through `AddHttpClient(...).AddTransientHttpErrorPolicy(...)` or through `Microsoft.Extensions.Http.Resilience` (built on Polly v8). Strategies are applied as `HttpMessageHandler`s — transparently to your code.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — Typed client + Polly resilience (v8 Microsoft.Extensions.Http.Resilience)
// Typed-клиент + устойчивость через Polly v8 (Microsoft.Extensions.Http.Resilience)

using System.Net.Http.Json;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Http.Resilience;
using Polly;

// ---- Модель ответа / Response model ----
public sealed record GitHubUser(string Login, int Id, string HtmlUrl);

// ---- Typed client — получает готовый HttpClient через DI ----
// Typed client — receives a pre-configured HttpClient via DI
public sealed class GitHubClient
{
    private readonly HttpClient _http;
    public GitHubClient(HttpClient http) => _http = http;

    public async Task<GitHubUser?> GetUserAsync(string login, CancellationToken ct = default)
        // GetFromJsonAsyncAsync — десериализация JSON в тип / deserialize JSON into type
        => await _http.GetFromJsonAsync<GitHubUser>($"users/{login}", ct);
}

// ---- Регистрация в контейнере / Container registration ----
public static class DependencyInjection
{
    public static IServiceCollection AddGitHubApi(this IServiceCollection services)
    {
        services.AddHttpClient<GitHubClient>((sp, c) =>
        {
            c.BaseAddress = new Uri("https://api.github.com");
            c.DefaultRequestHeaders.UserAgent.ParseAdd("dsh-course/1.0");
            c.Timeout = TimeSpan.FromSeconds(30); // таймаут на весь запрос / overall request timeout
        })
        // Стандартный resilience-конвейер (Polly v8) / standard resilience pipeline (Polly v8)
        .AddStandardResilienceHandler();
        // Альтернатива — ручная конфигурация:
        // Alternative — manual configuration:
        // .AddResilienceHandler("custom", pipeline =>
        //     pipeline.AddRetry(new HttpRetryStrategyOptions
        //     {
        //         MaxRetryAttempts = 3,
        //         BackoffType = DelayBackoffType.Exponential,
        //         UseJitter = true
        //     })
        //     .AddCircuitBreaker(new HttpCircuitBreakerStrategyOptions
        //     {
        //         SamplingDuration = TimeSpan.FromSeconds(30),
        //         FailureRatio = 0.5,
        //         MinimumThroughput = 8,
        //         BreakDuration = TimeSpan.FromSeconds(15)
        //     }));

        return services;
    }
}

// ---- Named client (альтернативный стиль) / Named client (alternative style) ----
// services.AddHttpClient("weather", c => c.BaseAddress = new Uri("https://api.weather.gov"));
// using var client = factory.CreateClient("weather"); // лёгкая обёртка над пулом handler'ов

// ---- Использование / Usage ----
// var github = scope.ServiceProvider.GetRequiredService<GitHubClient>();
// var user = await github.GetUserAsync("dotnet");
```

#### Best Practices
- Используйте `IHttpClientFactory` (typed или named clients), а не `new HttpClient()` и не один общий `static HttpClient`. / Use `IHttpClientFactory` (typed or named clients) — never `new HttpClient()` nor a single shared `static HttpClient`.
- Для typed clients держите клиента с небольшим responsibility; не делайте «бог»-клиента на весь сторонний API. / Keep typed clients focused; do not build a "god" client for an entire third-party API.
- Всегда задавайте `BaseAddress`, `Timeout` и заголовки (например `User-Agent`) при регистрации, а не в точке вызова. / Always set `BaseAddress`, `Timeout`, and headers (e.g. `User-Agent`) at registration time, not at call sites.
- Применяйте resilience (retry + circuit breaker) для внешних вызовов; используйте exponential backoff + jitter. / Apply resilience (retry + circuit breaker) to outbound calls; use exponential backoff with jitter.
- Идемпотентность: retry безопасен только для GET/HEAD/PUT; для POST продумайте защиту от дублей. / Idempotency: retry is safe only for GET/HEAD/PUT; for POST, plan for duplicate protection.
- Передавайте `CancellationToken` во все асинхронные HTTP-вызовы. / Propagate `CancellationToken` into every async HTTP call.

#### Частые ошибки / Common Mistakes
- `new HttpClient()` на каждый запрос → исчерпание сокетов в `TIME_WAIT`. Используйте фабрику. / `new HttpClient()` per request → socket exhaustion in `TIME_WAIT`. Use the factory.
- `using var client = new HttpClient()` внутри hot path — то же самое. Используйте typed/named client. / `using var client = new HttpClient()` in a hot path — same trap. Use a typed/named client.
- Общий `static HttpClient` без фабрики → навсегда кэшируется DNS. Используйте фабрику (handler пересоздаётся каждые 2 мин). / A shared `static HttpClient` without the factory → DNS cached forever. Use the factory (handler rotates every 2 min).
- Retry на POST без идемпотентного ключа → дублирующие副作用. Используйте idempotency-key или не ретрайте. / Retry on POST without an idempotency key → duplicate side effects. Use an idempotency key or do not retry.
- Бесконечные retry без circuit breaker → лавина запросов на упавший сервис. Добавьте circuit breaker и timeout. / Infinite retries without a circuit breaker → request avalanche on a down service. Add a circuit breaker and a timeout.
- Чтение тела ответа до проверки `EnsureSuccessStatusCode()` молча глотает ошибки. Сначала проверяйте статус. / Reading the body before `EnsureSuccessStatusCode()` silently swallows failures. Check status first.

#### Чек-лист самопроверки / Self-check Checklist
- [ ] Я не использую `new HttpClient()` в hot path; клиенты создаются через `IHttpClientFactory`. / I do not use `new HttpClient()` in a hot path; clients come from `IHttpClientFactory`.
- [ ] Выбран и обоснован стиль: typed или named client. / I chose and justified a style: typed or named client.
- [ ] Заданы `BaseAddress`, `Timeout` и обязательные заголовки при регистрации. / `BaseAddress`, `Timeout`, and required headers are set at registration.
- [ ] Подключена resilience-стратегия (retry + circuit breaker) с exponential backoff + jitter. / A resilience strategy (retry + circuit breaker) with exponential backoff + jitter is wired in.
- [ ] Продумана идемпотентность для POST-запросов. / POST idempotency is handled.
- [ ] `CancellationToken` прокидывается во все вызовы. / `CancellationToken` flows into all calls.
- [ ] Я могу объяснить, почему фабрика решает и socket exhaustion, и DNS-кэширование. / I can explain why the factory solves both socket exhaustion and DNS caching.

#### Ресурсы / Resources
- [Microsoft Learn — Make HTTP requests using IHttpClientFactory in ASP.NET Core](https://learn.microsoft.com/dotnet/core/extensions/httpclient-factory)
- [Polly — resilience strategies for .NET](https://www.pollydocs.org/)
- [Microsoft.Extensions.Http.Resilience](https://learn.microsoft.com/dotnet/core/extensions/http-client-resilience)

---

[⬆ К модулю M14](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
