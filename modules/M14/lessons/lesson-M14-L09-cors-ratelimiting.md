[⬆ К модулю M14](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M14-L09: CORS, rate limiting (обзор) / CORS, rate limiting (overview)

**Модуль / Module:** M14
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

CORS (Cross-Origin Resource Sharing) — это механизм безопасности браузера, который разрешает или запрещает веб-странице делать запросы к серверу с другим доменом, портом или схемой. Представьте браузер как охранника в офисе: по умолчанию он не пускает курьеров (запросы) из других зданий (origin) в ваш сервер, если вы явно не составили «список допущенных». Origin складывается из трёх частей: схемы (https), хоста (example.com) и порта (443). Если хоть одна отличается — это уже «чужой» origin, и браузер применяет Same-Origin Policy.

Когда фронтенд на `https://app.example.com` обращается к API на `https://api.example.com`, браузер проверяет, разрешает ли сервер такой origin. Сервер отвечает специальными HTTP-заголовками: `Access-Control-Allow-Origin`, `Access-Control-Allow-Methods`, `Access-Control-Allow-Headers`. Если ответа нет или origin не разрешён — браузер блокирует ответ, и фронтенд получает ошибку в консоли.

Существуют простые запросы (simple requests) и предполётные (preflight). Простые запросы — это GET/POST/HEAD с базовыми заголовками, браузер шлёт их сразу. Для «сложных» запросов (PUT, DELETE, нестандартные заголовки, JSON-тело) браузер сначала отправляет `OPTIONS`-запрос — это и есть preflight. Сервер в preflight отвечает, какие методы и заголовки разрешены, на какое время кэшировать это разрешение (`Access-Control-Max-Age`). Только получив «зелёный свет», браузер шлёт настоящий запрос. Поэтому отсутствие обработки `OPTIONS` — частая причина неработающего CORS.

В ASP.NET Core CORS настраивается через `AddCors()` и `UseCors()`. В .NET 8 доступен удобный `AddCors(options => options.AddDefaultPolicy(...))` или именованные политики. Главное правило безопасности: **никогда не используйте `AllowAnyOrigin` вместе с `AllowCredentials`** — это запрещено спецификацией и создаёт уязвимость. Для credentialed-запросов (cookies, JWT из заголовка, который браузер считает чувствительным) явно перечисляйте разрешённые origin.

Rate limiting (ограничение частоты запросов) — это «таможня» для трафика: защищает API от перегрузки, DDoS, bruteforce и злоупотреблений со стороны одного клиента. Начиная с .NET 7 в коробке есть `Microsoft.AspNetCore.RateLimiting` — middleware, которое не требует сторонних библиотек. Вы регистрируете лимит-политики в `AddRateLimiter()` и применяете их через `UseRateLimiter()`, глобально или через атрибут `[EnableRateLimiting("policy")]`.

Существует несколько классических алгоритмов. **Fixed Window** — окно фиксированной длины (например, 60 секунд), счётчик сбрасывается в начале окна. Просто, но даёт всплески на границах: клиент может сделать 100 запросов в конце одного окна и 100 в начале следующего — 200 за секунду. **Token Bucket** — ведро с токенами, пополняется с фиксированной скоростью; каждый запрос забирает токен. Позволяет контролируемые всплески (burst) до размера ведра, сохраняя среднюю скорость. **Sliding Window** и **Concurrency Limiter** дополняют набор: первый сглаживает границу окна, второй ограничивает число одновременных запросов, а не их частоту.

Когда лимит превышен, middleware возвращает `429 Too Many Requests` и, по хорошему тону, заголовок `Retry-After`. В `FixedWindowRateLimiter` и других есть опция `AutoReplenishment`, а `TokenBucketRateLimiter` управляется параметрами `tokenLimit`, `tokensPerPeriod`, `replenishmentPeriod`. Ключ идентификации клиента (`partitionKey`) обычно строят из IP или userId — иначе лимит будет общим для всех.

#### Theory (EN)

CORS (Cross-Origin Resource Sharing) is a browser security mechanism that allows or denies a web page from making requests to a server on a different domain, port, or scheme. Picture the browser as a building security guard: by default it refuses couriers (requests) coming from other buildings (origins) unless you have explicitly put them on an allowed list. An origin is built from three parts: scheme (https), host (example.com), and port (443). If even one differs, the origin is considered foreign and the browser applies the Same-Origin Policy.

When a frontend at `https://app.example.com` calls an API at `https://api.example.com`, the browser checks whether the server permits that origin. The server answers with special HTTP headers: `Access-Control-Allow-Origin`, `Access-Control-Allow-Methods`, `Access-Control-Allow-Headers`. If the answer is missing or the origin is not allowed, the browser blocks the response and the frontend sees a CORS error in the console.

There are simple requests and preflight requests. Simple requests are GET/POST/HEAD with basic headers; the browser sends them straight away. For "complex" requests (PUT, DELETE, custom headers, JSON body) the browser first sends an `OPTIONS` request — that is the preflight. The server replies with the allowed methods, headers, and for how long this permission may be cached (`Access-Control-Max-Age`). Only after a green light does the browser send the real request. That is why missing `OPTIONS` handling is a frequent cause of broken CORS.

In ASP.NET Core you configure CORS with `AddCors()` and `UseCors()`. .NET 8 offers the convenient `AddCors(options => options.AddDefaultPolicy(...))` plus named policies. The key security rule: **never combine `AllowAnyOrigin` with `AllowCredentials`** — the spec forbids it and it creates a real vulnerability. For credentialed requests (cookies, sensitive headers) always list the exact allowed origins.

Rate limiting is the "customs checkpoint" of traffic: it protects an API from overload, DDoS, brute force, and abuse by a single client. Since .NET 7 there is a built-in `Microsoft.AspNetCore.RateLimiting` middleware — no third-party packages needed. You register limit policies in `AddRateLimiter()` and apply them with `UseRateLimiter()`, globally or via the `[EnableRateLimiting("policy")]` attribute.

There are several classic algorithms. **Fixed Window** uses a window of fixed length (say, 60 seconds) with a counter that resets at the window start. It is simple but allows bursts at the edges: a client can fire 100 requests at the end of one window and 100 at the start of the next — 200 in a single second. **Token Bucket** is a bucket of tokens refilled at a fixed rate; each request consumes a token. It permits controlled bursts up to the bucket size while keeping the average rate. **Sliding Window** and **Concurrency Limiter** round out the set: the first smooths the window boundary, the second caps concurrent in-flight requests instead of request frequency.

When the limit is exceeded, the middleware returns `429 Too Many Requests` and, by good manners, a `Retry-After` header. `FixedWindowRateLimiter` and friends expose an `AutoReplenishment` option, while `TokenBucketRateLimiter` is driven by `tokenLimit`, `tokensPerPeriod`, and `replenishmentPeriod`. The client identity key (`partitionKey`) is usually built from IP or userId — otherwise the limit becomes shared across all callers.

#### Пример кода / Code Example

```csharp
// Program.cs — C# 12 / .NET 8
// CORS + Rate Limiting: полная конфигурация / Full configuration
using System.Threading.RateLimiting;
using Microsoft.AspNetCore.RateLimiting;

var builder = WebApplication.CreateBuilder(args);

// === 1. CORS: именованные политики / Named policies ===
builder.Services.AddCors(options =>
{
    // Политика по умолчанию — для credentialed-запросов / Default policy for credentialed requests
    options.AddDefaultPolicy(policy => policy
        .WithOrigins("https://app.example.com", "https://admin.example.com")
        .AllowAnyHeader()
        .WithMethods("GET", "POST", "PUT", "DELETE")
        .AllowCredentials()
        .SetPreflightMaxAge(TimeSpan.FromMinutes(10)));

    // Открытая политика — только для публичных read-only эндпоинтов / Open policy for public read-only endpoints
    options.AddPolicy("Public", policy => policy
        .AllowAnyOrigin()
        .WithMethods("GET")
        .AllowAnyHeader()
        .DisallowCredentials()); // AllowAnyOrigin + AllowCredentials запрещено спецификацией / forbidden by spec
});

// === 2. Rate Limiter: несколько алгоритмов / Multiple algorithms ===
builder.Services.AddRateLimiter(options =>
{
    options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(httpContext =>
        // Ключ по IP клиента, fallback на "anon" / Key by client IP, fallback to "anon"
        RateLimiter.GetPartitionedHTTPRateLimiter(
            partitionKey: httpContext.Connection.RemoteIpAddress?.ToString() ?? "anon",
            factory: key => new FixedWindowRateLimiter(new FixedWindowRateLimiterOptions
            {
                AutoReplenishment = true,
                PermitLimit = 100,             // 100 запросов на окно / 100 requests per window
                Window = TimeSpan.FromMinutes(1)
            })));

    // Token Bucket — допускает всплески / Allows bursts
    options.AddPolicy("TokenBucket", httpContext =>
        RateLimiter.GetPartitionedHTTPRateLimiter(
            partitionKey: httpContext.Connection.RemoteIpAddress?.ToString() ?? "anon",
            factory: key => new TokenBucketRateLimiter(new TokenBucketRateLimiterOptions
            {
                TokenLimit = 50,               // макс. размер ведра / max bucket size
                TokensPerPeriod = 20,          // пополнение за период / tokens added per period
                ReplenishmentPeriod = TimeSpan.FromSeconds(1),
                QueueLimit = 0,                // без очереди — сразу 429 / no queue — 429 immediately
                QueueProcessingOrder = QueueProcessingOrder.OldestFirst,
                AutoReplenishment = true
            })));

    // Custom response для 429 / Custom response on rejection
    options.OnRejected = async (context, cancellationToken) =>
    {
        context.HttpContext.Response.StatusCode = StatusCodes.Status429TooManyRequests;
        context.HttpContext.Response.Headers["Retry-After"] = "60";
        await context.HttpContext.Response.WriteAsync(
            "Rate limit exceeded. / Лимит запросов превышен.", cancellationToken);
    };
});

builder.Services.AddControllers();

var app = builder.Build();

// Порядок middleware критичен / Middleware order matters
app.UseRouting();
app.UseCors();          // до endpoints, после routing / before endpoints, after routing
app.UseRateLimiter();   // после CORS / after CORS
app.MapControllers();

// Применение политик на конкретных эндпоинтах / Apply policies on specific endpoints
app.MapGet("/api/public", () => Results.Ok("public"))
   .RequireCors("Public")
   .DisableRateLimiting();          // публичный — без лимита / public — no limit

app.MapGet("/api/expensive", () => Results.Ok("expensive"))
   .EnableRateLimiting("TokenBucket"); // токен-ведро для «тяжёлых» / token bucket for heavy calls

app.Run();
```

#### Best Practices

- Всегда явно перечисляйте разрешённые origin вместо `AllowAnyOrigin` для защищённых API. / Always enumerate allowed origins explicitly instead of `AllowAnyOrigin` for protected APIs.
- Кэшируйте preflight через `SetPreflightMaxAge` (10–30 минут), чтобы сократить число `OPTIONS`-запросов. / Cache preflight with `SetPreflightMaxAge` (10–30 min) to reduce `OPTIONS` round-trips.
- Не комбинируйте `AllowAnyOrigin` с `AllowCredentials` — это запрещено спецификацией и небезопасно. / Never combine `AllowAnyOrigin` with `AllowCredentials` — the spec forbids it and it is unsafe.
- Для идентификации клиента в rate limiter используйте IP + userId, а не пустую строку. / Identify rate-limit clients by IP + userId, not an empty string.
- Выбирайте Token Bucket, если нужны контролируемые всплески; Fixed Window — для простых случаев. / Pick Token Bucket for controlled bursts; Fixed Window for simple cases.
- Возвращайте `Retry-After` вместе с `429`, чтобы клиент знал, когда повторить запрос. / Return `Retry-After` with `429` so clients know when to retry.
- Применяйте `UseCors` после `UseRouting`, но до `MapControllers`; `UseRateLimiter` — после `UseCors`. / Apply `UseCors` after `UseRouting` but before `MapControllers`; `UseRateLimiter` after `UseCors`.

#### Частые ошибки / Common Mistakes

- `AllowAnyOrigin().AllowCredentials()` → браузер блокирует запрос; используйте `WithOrigins(...)` со списком. (RU) / `AllowAnyOrigin().AllowCredentials()` → browser blocks the request; use `WithOrigins(...)` with an explicit list. (EN)
- Забыли вызвать `app.UseCors()` — политика зарегистрирована, но не применяется. → Всегда добавляйте `UseCors` в pipeline. (RU) / Forgot `app.UseCors()` — policy registered but never applied. → Always add `UseCors` to the pipeline. (EN)
- Сервер не отвечает на `OPTIONS` (preflight) → проверьте, что CORS-middleware стоит до endpoint-сопоставления и не закрыт авторизацией. (RU) / Server ignores `OPTIONS` (preflight) → ensure CORS middleware runs before endpoint matching and is not blocked by auth. (EN)
- Общий лимит для всех клиентов (один `partitionKey`) → один злоумышленник исчерпывает квоту для всех. → Ключ по IP/userId. (RU) / Shared limit for all clients (single `partitionKey`) → one abuser exhausts everyone's quota. → Key by IP/userId. (EN)
- `PermitLimit` слишком мал для легитимных сценариев → подбирайте лимит под реальные паттерны нагрузки. (RU) / `PermitLimit` too small for legit flows → tune the limit to real load patterns. (EN)
- Лимит считается по авторизованному userId, но `OnRejected` возвращает 500 из-за исключения → тестируйте путь отказа. (RU) / Limit keyed by authenticated userId, but `OnRejected` throws 500 → test the rejection path. (EN)

#### Чек-лист самопроверки / Self-check Checklist

- [ ] CORS-политика зарегистрирована через `AddCors` и применена через `UseCors`. (RU)
- [ ] Разрешённые origin перечислены явно; нет `AllowAnyOrigin + AllowCredentials`. (RU)
- [ ] Обработан preflight (`OPTIONS`), задан `SetPreflightMaxAge`. (RU)
- [ ] `AddRateLimiter` зарегистрирован, `UseRateLimiter` стоит в pipeline. (RU)
- [ ] Выбран подходящий алгоритм: Fixed Window или Token Bucket. (RU)
- [ ] `partitionKey` построен по IP/userId, а не пустой строке. (RU)
- [ ] На отказ возвращается `429` + `Retry-After`. (RU)
- [ ] Порядок middleware: routing → cors → rate limiter → endpoints. (RU)
- [ ] CORS policy registered via `AddCors` and applied via `UseCors`. (EN)
- [ ] Allowed origins are explicit; no `AllowAnyOrigin + AllowCredentials`. (EN)
- [ ] Preflight (`OPTIONS`) handled, `SetPreflightMaxAge` set. (EN)
- [ ] `AddRateLimiter` registered, `UseRateLimiter` in the pipeline. (EN)
- [ ] Right algorithm chosen: Fixed Window or Token Bucket. (EN)
- [ ] `partitionKey` built from IP/userId, not an empty string. (EN)
- [ ] Rejection returns `429` + `Retry-After`. (EN)
- [ ] Middleware order: routing → cors → rate limiter → endpoints. (EN)

#### Ресурсы / Resources

- [Microsoft Learn — Enable CORS in ASP.NET Core — https://learn.microsoft.com/aspnet/core/security/cors]
- [Microsoft Learn — Rate limiting in ASP.NET Core — https://learn.microsoft.com/aspnet/core/performance/rate-limit]
- [MDN — CORS overview — https://developer.mozilla.org/docs/Web/HTTP/CORS]

---

[⬆ К модулю M14](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
