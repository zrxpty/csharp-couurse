---
[← К уроку M14-L09](lesson-M14-L09-cors-ratelimiting.md) | [⬆ К модулю M14](../README.md) | [Следующее ДЗ →](homework-M14-L10-httpclient-factory.md)
---

### Домашнее задание M14-L09: CORS, rate limiting (обзор) / Homework M14-L09: CORS, rate limiting (overview)

**Урок / Lesson:** M14-L09
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться самостоятельно конфигурировать CORS (именованные политики, обработка preflight, безопасное сочетание `AllowCredentials` с явно перечисленными origin) и встроенный rate limiter ASP.NET Core (Fixed Window и Token Bucket, корректный `partitionKey`, кастомный `429` + `Retry-After`), соблюдая правильный порядок middleware и избегая частых ошибок из урока. (EN) Learn to configure CORS (named policies, preflight handling, the safe combination of `AllowCredentials` with explicit origins) and the built-in ASP.NET Core rate limiter (Fixed Window and Token Bucket, a correct `partitionKey`, a custom `429` + `Retry-After`), respecting middleware order and avoiding the common mistakes covered in the lesson.

#### Связь с уроком / Connection to the lesson
(RU) Урок объяснял CORS как механизм браузера (origin = scheme + host + port), разницу между простыми и предполётными запросами, и показывал, что `AllowAnyOrigin` + `AllowCredentials` запрещены спецификацией. Также он вводил `Microsoft.AspNetCore.RateLimiting` с алгоритмами Fixed Window и Token Bucket и подчёркивал критичность порядка middleware: `UseRouting → UseCors → UseRateLimiter → MapControllers`. Это ДЗ требует собрать всё это в одном работающем приложении и довести до прохождения проверок curl-ом.
(EN) The lesson explained CORS as a browser mechanism (origin = scheme + host + port), the difference between simple and preflight requests, and that `AllowAnyOrigin` + `AllowCredentials` is forbidden by the spec. It also introduced `Microsoft.AspNetCore.RateLimiting` with Fixed Window and Token Bucket algorithms and stressed the critical middleware order: `UseRouting → UseCors → UseRateLimiter → MapControllers`. This homework asks you to assemble all of it in one working app and verify it with curl.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Вы — backend-разработчик небольшой компании «Курьер-онлайн», у которой есть три отдельных веб-приложения, обращающихся к одному ASP.NET Core 8 API: публичный фронтенд на `https://shop.example.com`, внутренний админ-интерфейс на `https://admin.example.com` и партнёрский портал на `https://partner.example.com`. Браузер по умолчанию блокирует кросс-доменные запросы, поэтому фронтенды получают ошибки CORS, а пользователи жалуются на пустые ответы. Параллельно сервис начал страдать от всплесков трафика: один анонимный клиент может исчерпать общую квоту для всех, потому что текущий прототип лимитирования вообще не различает пользователей. Вам нужно закрыть обе проблемы штатными средствами .NET 8 — без сторонних библиотек — и сделать это так, чтобы безопасные credentialed-запросы работали, публичные read-only эндпоинты оставались открытыми, а «тяжёлые» эндпоинты получали отдельную политику rate limiting с токен-ведром, допускающим контролируемые всплески. Это типичная задача продакшен-готового API: малый объём кода, но высокая цена ошибки — одна неверная строка с `AllowAnyOrigin().AllowCredentials()` ломает все credentialed-запросы, а забытый `app.UseCors()` оставляет политику «зарегистрированной, но не применённой». Урок показал все эти грабли; ваша задача — наступать на них только во время отладки, а в финальном решении обойти.

#### Что нужно сделать (пошагово)

1. Создайте новый проект minimal API на .NET 8: выполните `dotnet new web -n CourierApi` в пустой папке, перейдите в неё `cd CourierApi` и проверьте версию SDK командой `dotnet --version` — должно быть `8.x`. Убедитесь, что в `CourierApi.csproj` стоит `<TargetFramework>net8.0</TargetFramework>`; если нет — поправьте вручную. Пространства имён `System.Threading.RateLimiting` и `Microsoft.AspNetCore.RateLimiting` доступны из коробки, дополнительных NuGet-пакетов ставить не нужно.
2. Откройте `Program.cs` и замените шаблон на конфигурацию с двумя CORS-политиками через `AddCors`. Первая — политика по умолчанию для credentialed-запросов: вызовите `options.AddDefaultPolicy(...)`, внутри используйте `WithOrigins("https://shop.example.com", "https://admin.example.com", "https://partner.example.com")`, затем `.AllowAnyHeader()`, `.WithMethods("GET", "POST", "PUT", "DELETE")`, `.AllowCredentials()` и `.SetPreflightMaxAge(TimeSpan.FromMinutes(10))`. Вторая — именованная политика `"Public"` для read-only публичных эндпоинтов: `.AllowAnyOrigin().WithMethods("GET").AllowAnyHeader().DisallowCredentials()`. Обратите внимание: здесь `AllowAnyOrigin` допустим именно потому, что credentialed-запросы отключены через `DisallowCredentials` — это единственное безопасное сочетание.
3. Зарегистрируйте rate limiter через `AddRateLimiter`. Настройте `GlobalLimiter` через `PartitionedRateLimiter.Create<HttpContext, string>(...)`, где `partitionKey` вычисляется как `httpContext.Connection.RemoteIpAddress?.ToString() ?? "anon"` — это гарантирует, что каждый IP-адрес получит собственную квоту. Внутри `factory` создайте `FixedWindowRateLimiter` с `AutoReplenishment = true`, `PermitLimit = 100`, `Window = TimeSpan.FromMinutes(1)`. Затем добавьте именованную политику `"TokenBucket"` через `options.AddPolicy("TokenBucket", httpContext => ...)`, используя тот же `partitionKey`, но уже `TokenBucketRateLimiter` с параметрами `TokenLimit = 50`, `TokensPerPeriod = 20`, `ReplenishmentPeriod = TimeSpan.FromSeconds(1)`, `QueueLimit = 0`, `QueueProcessingOrder = QueueProcessingOrder.OldestFirst`, `AutoReplenishment = true`.
4. Добавьте колбэк `options.OnRejected`, который выставляет `StatusCode = StatusCodes.Status429TooManyRequests`, заголовок `Retry-After` со значением `"60"` и пишет в тело строку `"Rate limit exceeded."` через `WriteAsync`. Это превращает дефолтный пустой ответ в информативный и соответствует best practice из урока.
5. Зарегистрируйте контроллеры или сразу используйте minimal API endpoints. Добавьте три эндпоинта: `app.MapGet("/api/public", ...)` с `.RequireCors("Public").DisableRateLimiting()` — публичный, без лимита; `app.MapGet("/api/profile", ...)` с `.RequireCors` по умолчанию (можно не указывать — применится default policy) — защищённый credentialed-эндпоинт; `app.MapGet("/api/report", ...)` с `.EnableRateLimiting("TokenBucket")` — «тяжёлый» эндпоинт под токен-ведром.
6. Выстроите middleware в правильном порядке, как требовал урок: `app.UseRouting()`, затем `app.UseCors()`, затем `app.UseRateLimiter()`, затем `app.MapControllers()` или `MapGet`. Любая перестановка ломает либо CORS, либо лимитирование.
7. Запустите приложение: `dotnet run`. По умолчанию слушается `http://localhost:5000` (или `5001`). Запишите фактический URL из вывода консоли — он понадобится для curl.
8. Проверьте CORS эмуляцией origin: `curl -i -H "Origin: https://shop.example.com" http://localhost:5000/api/profile`. В ответе должен появиться заголовок `Access-Control-Allow-Origin: https://shop.example.com` и `Access-Control-Allow-Credentials: true`.
9. Проверьте preflight: `curl -i -X OPTIONS -H "Origin: https://shop.example.com" -H "Access-Control-Request-Method: POST" -H "Access-Control-Request-Headers: content-type" http://localhost:5000/api/profile`. Ответ должен содержать `Access-Control-Allow-Methods`, `Access-Control-Allow-Headers` и `Access-Control-Max-Age`.
10. Проверьте rate limiting: в цикле запросите `/api/report` 60 раз подряд `for /L %i in (1,1,60) do curl -s -o nul -w "%{http_code}\n" http://localhost:5000/api/report` (Windows cmd). Последние запросы должны вернуть `429` с заголовком `Retry-After: 60` и телом `"Rate limit exceeded."`.
11. Проверьте негативный сценарий CORS: `curl -i -H "Origin: https://evil.example.com" http://localhost:5000/api/profile`. В ответе заголовок `Access-Control-Allow-Origin` должен ОТСУТСТВОВАТЬ — браузер заблокирует такой запрос.
12. Зафиксируйте результаты в `README.md`: какие заголовки вы получили на каждом шаге, какой HTTP-код и тело пришло на 429. Это и есть отчёт о прохождении.

#### Требования к решению

- Целевой фреймворк строго `net8.0`; язык C# 12 (top-level statements в `Program.cs`, можно использовать collection expressions и raw string literals для заголовков/сообщений, где это уместно).
- Решение должно собираться без ошибок и предупреждений командой `dotnet build` и запускаться командой `dotnet run` без сторонних NuGet-пакетов (только встроенные `Microsoft.AspNetCore.RateLimiting` и `System.Threading.RateLimiting`).
- Должны быть зарегистрированы минимум две CORS-политики: default с явно перечисленными origin и `AllowCredentials`, и именованная `"Public"` с `AllowAnyOrigin` + `DisallowCredentials`. Комбинация `AllowAnyOrigin().AllowCredentials()` запрещена категорически — её появление в коде сразу проваливает приёмку.
- Должны быть зарегистрированы минимум две политики rate limiting: `GlobalLimiter` на основе Fixed Window и именованная `"TokenBucket"` на основе Token Bucket. Параметры `AutoReplenishment`, `PermitLimit`, `Window`, `TokenLimit`, `TokensPerPeriod`, `ReplenishmentPeriod` должны быть заданы явно, не через дефолты.
- `partitionKey` обязан строиться из IP-адреса клиента (с fallback на константу вроде `"anon"`); использование пустой строки или общей константы для всех недопустимо — это классическая ошибка из урока.
- Должен быть реализован `OnRejected`, возвращающий `429`, заголовок `Retry-After` и тело с текстом сообщения.
- Порядок middleware обязан быть: `UseRouting → UseCors → UseRateLimiter → endpoints`. Любое отклонение должно быть обосновано в `README.md`, но по умолчанию следуйте порядку из урока.
- Код снабжён двуязычными комментариями (RU + EN), объясняющими, почему выбран тот или иной алгоритм и почему не использована запрещённая комбинация.
- В `README.md` должны быть записаны фактические заголовки и коды ответов из curl-проверок, доказывающие, что CORS и rate limiting действительно работают.

#### Тонкости и подводные камни

- **`AllowAnyOrigin` + `AllowCredentials`**: эта пара не просто «не рекомендуется», а запрещена спецификацией CORS и принудительно игнорируется браузером — credentialed-запрос не пройдёт. Если вам нужны cookies или авторизация через заголовок, который браузер считает чувствительным, всегда используйте `WithOrigins(...)` с явным списком и `.AllowCredentials()`.
- **Забытый `app.UseCors()`**: регистрация через `AddCors` добавляет сервисы, но НЕ включает middleware. Без `UseCors()` в pipeline заголовки CORS никогда не добавятся в ответ, и вы будете долго искать, почему «всё настроено, но не работает».
- **Preflight не обрабатывается**: для «сложных» запросов (PUT, DELETE, нестандартные заголовки, JSON-тело) браузер сначала шлёт `OPTIONS`. Если CORS-middleware стоит после авторизации или не стоит вовсе, сервер либо не ответит на `OPTIONS`, либо ответит `401`. Ставьте `UseCors` ДО любых endpoint-specific проверок авторизации.
- **Общий лимит для всех**: если `partitionKey` — константа, то один злоумышленник может исчерпать квоту для всех клиентов. Ключ по IP — минимально необходимое; для авторизованных запросов лучше комбинировать IP + userId.
- **`AutoReplenishment` и синхронизация**: для `FixedWindowRateLimiter` и `TokenBucketRateLimiter` `AutoReplenishment = true` означает, что токены/окна пополняются автоматически по таймеру; при `false` вам придётся вызывать `TryReplenish` вручную. Большинство сценариев используют `true`.
- **`QueueLimit > 0`**: если включить очередь, запросы сверх лимита не получают `429` сразу, а встают в очередь — это меняет семантику и может привести к каскадным таймаутам у клиента. Для публичного API обычно `QueueLimit = 0` и немедленный `429`.
- **`Retry-After` должен быть осмысленным**: значение в секундах должно примерно соответствовать периоду пополнения. Для Fixed Window с окном 60 секунд логично `Retry-After: 60`; для Token Bucket с пополнением 20 токенов/сек достаточно `Retry-After: 1`.
- **`UseRateLimiter` после `UseCors`**: если поставить лимитер до CORS, то preflight `OPTIONS` тоже будет лимитироваться, и браузер получит `429` ещё до основного запроса — это ломает весь flow. Урок явно фиксирует порядок `routing → cors → rate limiter → endpoints`.

#### Критерии приёмки

- [ ] Проект создаётся `dotnet new web`, собирается `dotnet build` без ошибок и предупреждений, запускается `dotnet run` на .NET 8.
- [ ] В `Program.cs` используется top-level statements (C# 12), без явного `class Program` / `static Main`.
- [ ] Вызван `builder.Services.AddCors(...)` с минимум двумя политиками: default и `"Public"`.
- [ ] Default-политика использует `WithOrigins(...)` с тремя явными origin, `.AllowCredentials()` и `.SetPreflightMaxAge(...)`.
- [ ] Политика `"Public"` использует `AllowAnyOrigin()` вместе с `DisallowCredentials()`, а НЕ с `AllowCredentials()`.
- [ ] Нигде в коде не встречается `AllowAnyOrigin().AllowCredentials()` (проверка grep-ом по репозиторию).
- [ ] Вызван `builder.Services.AddRateLimiter(...)` с `GlobalLimiter` на основе `FixedWindowRateLimiter`.
- [ ] `partitionKey` строится из `httpContext.Connection.RemoteIpAddress?.ToString() ?? "anon"`.
- [ ] Добавлена именованная политика `"TokenBucket"` через `options.AddPolicy("TokenBucket", ...)` с параметрами `TokenLimit`, `TokensPerPeriod`, `ReplenishmentPeriod`.
- [ ] Реализован `options.OnRejected`, выставляющий `429`, заголовок `Retry-After` и тело ответа.
- [ ] Middleware выстроены в порядке `UseRouting → UseCors → UseRateLimiter → MapControllers/MapGet`.
- [ ] Зарегистрированы минимум три эндпоинта: `/api/public` (`.RequireCors("Public").DisableRateLimiting()`), `/api/profile` (default CORS), `/api/report` (`.EnableRateLimiting("TokenBucket")`).
- [ ] `curl -H "Origin: https://shop.example.com" .../api/profile` возвращает `Access-Control-Allow-Origin` и `Access-Control-Allow-Credentials: true`.
- [ ] `curl -X OPTIONS` (preflight) возвращает `Access-Control-Allow-Methods`, `Access-Control-Allow-Headers`, `Access-Control-Max-Age`.
- [ ] Серия из 60 запросов к `/api/report` завершается кодами `429` с заголовком `Retry-After` и телом `"Rate limit exceeded."`.
- [ ] `README.md` содержит фактические заголовки и коды ответов из всех curl-проверок.

#### Подсказки (без прямого ответа)

- Вспомните, что `AddDefaultPolicy` применяется к эндпоинтам автоматически, если вы НЕ вызвали `RequireCors(...)` с конкретным именем; для публичного эндпоинта нужно явно «перекрыть» default политикой `"Public"` через `RequireCors("Public")`.
- Для preflight не нужно писать свой обработчик `OPTIONS` — middleware CORS сам перехватывает `OPTIONS` и отвечает. Главное — чтобы он стоял в pipeline раньше endpoint matching.
- Если `429` не приходит, хотя вы шлёте 60 запросов, проверьте: действительно ли эндпоинт использует политику `"TokenBucket"`, а не глобальный лимитер; и не закэшировал ли curl ответ (используйте `-H "Cache-Control: no-cache"` или разные URL).
- Если хотите, чтобы `partitionKey` был стабилен при тестировании с одного хоста, можно временно логировать его значение, чтобы убедиться, что IP действительно различается.
- `OnRejected` получает `OnRejectedContext` с полем `HttpContext` и `LeaseId` — не путайте его с `HttpContext` из обычного middleware.

#### Эталонное решение (разбор)

```csharp
// Program.cs — C# 12 / .NET 8
// CORS + Rate Limiting для «Курьер-онлайн» / CORS + Rate Limiting for "Courier-online"
using System.Threading.RateLimiting;
using Microsoft.AspNetCore.RateLimiting;

var builder = WebApplication.CreateBuilder(args);

// === 1. CORS: default для credentialed + именованная Public / Default for credentialed + named Public ===
builder.Services.AddCors(options =>
{
    // Default — строго три origin, credentialed-запросы разрешены / Strict three origins, credentialed allowed
    options.AddDefaultPolicy(policy => policy
        .WithOrigins("https://shop.example.com", "https://admin.example.com", "https://partner.example.com")
        .AllowAnyHeader()
        .WithMethods("GET", "POST", "PUT", "DELETE")
        .AllowCredentials()                // безопасно, т.к. origin явно перечислены / safe: origins are explicit
        .SetPreflightMaxAge(TimeSpan.FromMinutes(10)));

    // Public — read-only, без credentialed / read-only, no credentials
    // AllowAnyOrigin допустим ТОЛЬКО вместе с DisallowCredentials / AllowAnyOrigin only with DisallowCredentials
    options.AddPolicy("Public", policy => policy
        .AllowAnyOrigin()
        .WithMethods("GET")
        .AllowAnyHeader()
        .DisallowCredentials());
});

// === 2. Rate limiter: Fixed Window глобально + Token Bucket для тяжёлых / Fixed Window global + Token Bucket for heavy ===
builder.Services.AddRateLimiter(options =>
{
    // Глобальный Fixed Window, ключ по IP / Global Fixed Window, keyed by IP
    options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(httpContext =>
        RateLimiter.GetPartitionedHTTPRateLimiter(
            partitionKey: httpContext.Connection.RemoteIpAddress?.ToString() ?? "anon",
            factory: key => new FixedWindowRateLimiter(new FixedWindowRateLimiterOptions
            {
                AutoReplenishment = true,
                PermitLimit = 100,
                Window = TimeSpan.FromMinutes(1)
            })));

    // Token Bucket — контролируемые всплески для /api/report / Controlled bursts for /api/report
    options.AddPolicy("TokenBucket", httpContext =>
        RateLimiter.GetPartitionedHTTPRateLimiter(
            partitionKey: httpContext.Connection.RemoteIpAddress?.ToString() ?? "anon",
            factory: key => new TokenBucketRateLimiter(new TokenBucketRateLimiterOptions
            {
                TokenLimit = 50,
                TokensPerPeriod = 20,
                ReplenishmentPeriod = TimeSpan.FromSeconds(1),
                QueueLimit = 0,                       // без очереди → мгновенный 429 / no queue → instant 429
                QueueProcessingOrder = QueueProcessingOrder.OldestFirst,
                AutoReplenishment = true
            })));

    // Кастомный ответ 429 / Custom 429 response
    options.OnRejected = async (context, cancellationToken) =>
    {
        context.HttpContext.Response.StatusCode = StatusCodes.Status429TooManyRequests;
        context.HttpContext.Response.Headers["Retry-After"] = "60";
        await context.HttpContext.Response.WriteAsync("Rate limit exceeded.", cancellationToken);
    };
});

var app = builder.Build();

// Порядок middleware критичен (см. урок) / Middleware order is critical (see lesson)
app.UseRouting();
app.UseCors();          // после routing, до endpoints / after routing, before endpoints
app.UseRateLimiter();   // после CORS, иначе preflight получит 429 / after CORS, else preflight gets 429

// Публичный эндпоинт — открытая CORS-политика, без лимита / Public endpoint — open CORS, no limit
app.MapGet("/api/public", () => Results.Ok(new { message = "public" }))
   .RequireCors("Public")
   .DisableRateLimiting();

// Защищённый credentialed-эндпоинт — default CORS / Protected credentialed endpoint — default CORS
app.MapGet("/api/profile", () => Results.Ok(new { user = "demo", role = "courier" }));

// Тяжёлый эндпоинт — токен-ведро / Heavy endpoint — token bucket
app.MapGet("/api/report", () => Results.Ok(new { generatedAt = DateTime.UtcNow }))
   .EnableRateLimiting("TokenBucket");

app.Run();
```

Разбор по строкам:

- `using System.Threading.RateLimiting;` и `using Microsoft.AspNetCore.RateLimiting;` — эти пространства имён входят в коробку .NET 8, никакие NuGet-пакеты ставить не нужно. Урок явно подчёркивал: `Microsoft.AspNetCore.RateLimiting` доступен с .NET 7+, без сторонних библиотек.
- `AddCors` с двумя политиками демонстрирует ключевой паттерн: одна default для защищённых credentialed-запросов (явные origin), другая именованная для публичных. Это прямое применение best practice «всегда явно перечисляйте разрешённые origin вместо `AllowAnyOrigin` для защищённых API».
- `WithOrigins(...)` + `AllowCredentials()` безопасно, потому что origin заданы списком — браузер вернёт именно тот origin, который пришёл в `Origin:` запроса, если он есть в списке. Запрещённая комбинация `AllowAnyOrigin().AllowCredentials()` здесь отсутствует — это главное требование приёмки.
- `SetPreflightMaxAge(TimeSpan.FromMinutes(10))` кэширует результат preflight на 10 минут, сокращая число `OPTIONS`-запросов — это прямо рекомендовано в best practices урока.
- Политика `"Public"` использует `AllowAnyOrigin()` вместе с `DisallowCredentials()`. Это единственное безопасное сочетание с `AllowAnyOrigin`: без credentialed-запросов серверу безопасно возвращать `Access-Control-Allow-Origin: *`. Появись тут `AllowCredentials()` — приёмка бы провалилась.
- `AddRateLimiter` настраивает `GlobalLimiter` через `PartitionedRateLimiter.Create<HttpContext, string>`. Тип-параметр `string` — это тип `partitionKey`. Внутри лямбды мы берём IP клиента с fallback `"anon"` — это решает классическую проблему «общий лимит для всех», которую урок называл частой ошибкой. Если бы мы использовали константу, один злоумышленник исчерпал бы квоту для всех.
- `FixedWindowRateLimiter` с `AutoReplenishment = true` и `PermitLimit = 100` за минуту — простой базовый лимит, защищающий от перегрузки. `AutoReplenishment = true` означает, что окно сбрасывается автоматически по таймеру, без ручного `TryReplenish`.
- Именованная политика `"TokenBucket"` через `options.AddPolicy(...)` создаёт отдельный лимитер для тяжёлых эндпоинтов. `TokenBucketRateLimiter` с `TokenLimit = 50`, `TokensPerPeriod = 20`, `ReplenishmentPeriod = 1s` допускает контролируемые всплески до 50 запросов, но поддерживает среднюю скорость 20 запросов/сек — это ровно тот сценарий «тяжёлых эндпоинтов», для которого урок рекомендовал Token Bucket.
- `QueueLimit = 0` гарантирует мгновенный `429` при превышении — без очереди, без каскадных таймаутов. Урок предупреждал, что `QueueLimit > 0` меняет семантику и может привести к таймаутам.
- `OnRejected` — единственная точка, где можно кастомизировать ответ `429`. Мы выставляем статус, заголовок `Retry-After: 60` (соответствует окну Fixed Window) и тело сообщения. Урок прямо требовал «возвращайте `Retry-After` вместе с `429`».
- Порядок `UseRouting → UseCors → UseRateLimiter → endpoints` — это критический порядок из урока. Если поменять `UseCors` и `UseRateLimiter`, preflight `OPTIONS` получит `429` до того, как CORS успеет ответить, и весь flow сломается. Если убрать `UseCors` — политика зарегистрирована, но не применяется (частая ошибка).
- Эндпоинт `/api/public` явно применяет политику `"Public"` через `RequireCors("Public")` и отключает лимит через `DisableRateLimiting()` — публичный read-only не должен страдать от лимитов.
- Эндпоинт `/api/profile` не вызывает `RequireCors` — применяется default policy. Это верно, потому что `AddDefaultPolicy` именно для этого и предназначена.
- Эндпоинт `/api/report` использует `EnableRateLimiting("TokenBucket")` — это переключает его с глобального Fixed Window на Token Bucket, как и требовалось.

#### Задания на углубление (бонус)

1. Добавьте `SlidingWindowRateLimiter` как третью именованную политику и примените её к эндпоинту `/api/search`. Сравните поведение со Fixed Window на границе окна: сделайте серию запросов в последние 5 секунд одного окна и первые 5 секунд следующего, запишите разницу в `README.md`.
2. Реализуйте `ConcurrencyLimiter` для эндпоинта `/api/export`, ограничив число одновременных запросов (а не частоту) до 3. Проверьте, что 4-й параллельный запрос получает `429`.
3. Добавьте авторизацию по JWT и постройте `partitionKey` из `userId` для авторизованных запросов, оставив IP-ключ только для анонимных. Покажите, что лимит становится per-user.
4. Напишите интеграционный тест на `WebApplicationFactory`, который эмулирует `Origin` и проверяет наличие `Access-Control-Allow-Origin` в ответе, а также тест, который делает 60 запросов и ожидает хотя бы один `429`.

---

## Statement in English / Постановка на английском

#### Context & motivation

You are a backend engineer at a small company called "Courier-online" that runs three separate web applications talking to a single ASP.NET Core 8 API: a public storefront at `https://shop.example.com`, an internal admin UI at `https://admin.example.com`, and a partner portal at `https://partner.example.com`. By default the browser blocks cross-origin requests, so the frontends see CORS errors and end users complain about empty responses. In parallel the service has started to suffer from traffic bursts: a single anonymous client can exhaust a shared quota for everyone because the current rate-limiting prototype does not distinguish callers at all. You must fix both problems with native .NET 8 tooling — no third-party libraries — and do it in a way that keeps credentialed requests working, leaves public read-only endpoints open, and gives the "heavy" endpoints their own Token Bucket policy that permits controlled bursts. This is a typical production-readiness task: small amount of code, high cost of mistakes — a single wrong line with `AllowAnyOrigin().AllowCredentials()` breaks every credentialed request, and a forgotten `app.UseCors()` leaves the policy "registered but never applied". The lesson walked through every one of these traps; your job is to step on them only while debugging and to dodge them in the final solution.

#### What to do step by step

1. Create a new minimal API project on .NET 8: run `dotnet new web -n CourierApi` in an empty folder, `cd CourierApi` into it, and confirm the SDK version with `dotnet --version` — it must report `8.x`. Open `CourierApi.csproj` and make sure `<TargetFramework>net8.0</TargetFramework>` is set; fix it manually if not. The namespaces `System.Threading.RateLimiting` and `Microsoft.AspNetCore.RateLimiting` ship in the box, so do not add any NuGet packages.
2. Open `Program.cs` and replace the template with a CORS configuration that registers two policies through `AddCors`. The first one is the default policy for credentialed requests: call `options.AddDefaultPolicy(...)` and inside it use `WithOrigins("https://shop.example.com", "https://admin.example.com", "https://partner.example.com")`, then `.AllowAnyHeader()`, `.WithMethods("GET", "POST", "PUT", "DELETE")`, `.AllowCredentials()`, and `.SetPreflightMaxAge(TimeSpan.FromMinutes(10))`. The second is a named policy `"Public"` for read-only public endpoints: `.AllowAnyOrigin().WithMethods("GET").AllowAnyHeader().DisallowCredentials()`. Notice that `AllowAnyOrigin` is acceptable here precisely because credentialed requests are turned off via `DisallowCredentials` — that is the only safe combination.
3. Register the rate limiter through `AddRateLimiter`. Configure `GlobalLimiter` via `PartitionedRateLimiter.Create<HttpContext, string>(...)` where the `partitionKey` is computed as `httpContext.Connection.RemoteIpAddress?.ToString() ?? "anon"` — this guarantees that each IP gets its own quota. Inside the `factory` create a `FixedWindowRateLimiter` with `AutoReplenishment = true`, `PermitLimit = 100`, `Window = TimeSpan.FromMinutes(1)`. Then add a named policy `"TokenBucket"` through `options.AddPolicy("TokenBucket", httpContext => ...)` using the same `partitionKey` logic but a `TokenBucketRateLimiter` with `TokenLimit = 50`, `TokensPerPeriod = 20`, `ReplenishmentPeriod = TimeSpan.FromSeconds(1)`, `QueueLimit = 0`, `QueueProcessingOrder = QueueProcessingOrder.OldestFirst`, `AutoReplenishment = true`.
4. Add an `options.OnRejected` callback that sets `StatusCode = StatusCodes.Status429TooManyRequests`, the `Retry-After` header with value `"60"`, and writes `"Rate limit exceeded."` to the body via `WriteAsync`. This turns the default empty response into an informative one and follows the best practice from the lesson.
5. Register controllers or use minimal API endpoints directly. Add three endpoints: `app.MapGet("/api/public", ...)` with `.RequireCors("Public").DisableRateLimiting()` — public, no limit; `app.MapGet("/api/profile", ...)` with the default CORS policy (you may omit `RequireCors` — the default policy will apply) — a protected credentialed endpoint; `app.MapGet("/api/report", ...)` with `.EnableRateLimiting("TokenBucket")` — a "heavy" endpoint under the token bucket.
6. Order the middleware exactly as the lesson requires: `app.UseRouting()`, then `app.UseCors()`, then `app.UseRateLimiter()`, then `app.MapControllers()` or `MapGet`. Any permutation breaks either CORS or rate limiting.
7. Run the app: `dotnet run`. By default it listens on `http://localhost:5000` (or `5001`). Note the actual URL printed by the console — you will need it for curl.
8. Verify CORS by emulating an origin: `curl -i -H "Origin: https://shop.example.com" http://localhost:5000/api/profile`. The response must include `Access-Control-Allow-Origin: https://shop.example.com` and `Access-Control-Allow-Credentials: true`.
9. Verify preflight: `curl -i -X OPTIONS -H "Origin: https://shop.example.com" -H "Access-Control-Request-Method: POST" -H "Access-Control-Request-Headers: content-type" http://localhost:5000/api/profile`. The response must contain `Access-Control-Allow-Methods`, `Access-Control-Allow-Headers`, and `Access-Control-Max-Age`.
10. Verify rate limiting: in a loop request `/api/report` 60 times — `for /L %i in (1,1,60) do curl -s -o nul -w "%{http_code}\n" http://localhost:5000/api/report` (Windows cmd). The last requests must return `429` with the header `Retry-After: 60` and body `"Rate limit exceeded."`.
11. Verify a negative CORS scenario: `curl -i -H "Origin: https://evil.example.com" http://localhost:5000/api/profile`. The response must NOT contain the `Access-Control-Allow-Origin` header — the browser would block such a request.
12. Record the results in `README.md`: which headers you got at each step, what HTTP code and body came back on the 429. That is your acceptance report.

#### Requirements

- The target framework must be strictly `net8.0`; the language is C# 12 (top-level statements in `Program.cs`; you may use collection expressions and raw string literals for headers/messages where appropriate).
- The solution must build without errors or warnings via `dotnet build` and run via `dotnet run` with no third-party NuGet packages (only the built-in `Microsoft.AspNetCore.RateLimiting` and `System.Threading.RateLimiting`).
- At least two CORS policies must be registered: a default one with explicitly enumerated origins and `AllowCredentials`, and a named `"Public"` one with `AllowAnyOrigin` + `DisallowCredentials`. The combination `AllowAnyOrigin().AllowCredentials()` is strictly forbidden — its presence anywhere in the code fails acceptance immediately.
- At least two rate-limiting policies must be registered: a `GlobalLimiter` based on Fixed Window and a named `"TokenBucket"` based on Token Bucket. The options `AutoReplenishment`, `PermitLimit`, `Window`, `TokenLimit`, `TokensPerPeriod`, `ReplenishmentPeriod` must be set explicitly, not via defaults.
- The `partitionKey` must be built from the client's IP address (with a fallback constant such as `"anon"`); using an empty string or a shared constant for everyone is unacceptable — it is the classic mistake from the lesson.
- An `OnRejected` handler must be implemented that returns `429`, the `Retry-After` header, and a response body with a message.
- The middleware order must be `UseRouting → UseCors → UseRateLimiter → endpoints`. Any deviation must be justified in `README.md`; by default follow the order from the lesson.
- The code must carry bilingual comments (RU + EN) explaining why each algorithm was chosen and why the forbidden combination was not used.
- `README.md` must record the actual headers and status codes from the curl checks proving that CORS and rate limiting actually work.

#### Pitfalls

- **`AllowAnyOrigin` + `AllowCredentials`**: this pair is not merely "discouraged" — it is forbidden by the CORS spec and the browser forcibly ignores it, so the credentialed request never succeeds. When you need cookies or a sensitive header for authorization, always use `WithOrigins(...)` with an explicit list and `.AllowCredentials()`.
- **Forgotten `app.UseCors()`**: registering via `AddCors` adds the services but does NOT turn on the middleware. Without `UseCors()` in the pipeline the CORS headers never appear in the response, and you will waste time wondering why "everything is configured but nothing works".
- **Preflight is not handled**: for "complex" requests (PUT, DELETE, custom headers, JSON body) the browser first sends `OPTIONS`. If the CORS middleware sits after authorization or is missing, the server either ignores `OPTIONS` or returns `401`. Put `UseCors` before any endpoint-specific authorization checks.
- **Shared limit for everyone**: if `partitionKey` is a constant, a single abuser can exhaust the quota for all clients. Keying by IP is the minimum; for authorized requests, combine IP + userId.
- **`AutoReplenishment` and synchronization**: for `FixedWindowRateLimiter` and `TokenBucketRateLimiter`, `AutoReplenishment = true` means tokens/windows replenish automatically on a timer; with `false` you must call `TryReplenish` manually. Most scenarios use `true`.
- **`QueueLimit > 0`**: with a non-zero queue, requests over the limit do not get `429` immediately — they queue, which changes the semantics and can lead to cascading client timeouts. For a public API, `QueueLimit = 0` with an immediate `429` is typical.
- **`Retry-After` should be meaningful**: the value in seconds should roughly match the replenishment period. For a Fixed Window of 60 seconds `Retry-After: 60` is sensible; for a Token Bucket refilling 20 tokens/second, `Retry-After: 1` is enough.
- **`UseRateLimiter` after `UseCors`**: if the limiter runs before CORS, the preflight `OPTIONS` is also rate-limited and the browser gets `429` before the real request — breaking the entire flow. The lesson explicitly fixes the order as `routing → cors → rate limiter → endpoints`.

#### Acceptance criteria

- [ ] The project is created with `dotnet new web`, builds with `dotnet build` without errors or warnings, and runs with `dotnet run` on .NET 8.
- [ ] `Program.cs` uses top-level statements (C# 12) with no explicit `class Program` / `static Main`.
- [ ] `builder.Services.AddCors(...)` is called with at least two policies: default and `"Public"`.
- [ ] The default policy uses `WithOrigins(...)` with three explicit origins, `.AllowCredentials()`, and `.SetPreflightMaxAge(...)`.
- [ ] The `"Public"` policy uses `AllowAnyOrigin()` together with `DisallowCredentials()`, NOT with `AllowCredentials()`.
- [ ] The string `AllowAnyOrigin().AllowCredentials()` does not appear anywhere in the repository (verified by grep).
- [ ] `builder.Services.AddRateLimiter(...)` is called with a `GlobalLimiter` based on `FixedWindowRateLimiter`.
- [ ] The `partitionKey` is built from `httpContext.Connection.RemoteIpAddress?.ToString() ?? "anon"`.
- [ ] A named `"TokenBucket"` policy is added via `options.AddPolicy("TokenBucket", ...)` with `TokenLimit`, `TokensPerPeriod`, `ReplenishmentPeriod`.
- [ ] An `options.OnRejected` handler is implemented that sets `429`, the `Retry-After` header, and the response body.
- [ ] Middleware is ordered as `UseRouting → UseCors → UseRateLimiter → MapControllers/MapGet`.
- [ ] At least three endpoints are registered: `/api/public` (`.RequireCors("Public").DisableRateLimiting()`), `/api/profile` (default CORS), `/api/report` (`.EnableRateLimiting("TokenBucket")`).
- [ ] `curl -H "Origin: https://shop.example.com" .../api/profile` returns `Access-Control-Allow-Origin` and `Access-Control-Allow-Credentials: true`.
- [ ] `curl -X OPTIONS` (preflight) returns `Access-Control-Allow-Methods`, `Access-Control-Allow-Headers`, `Access-Control-Max-Age`.
- [ ] A series of 60 requests to `/api/report` ends with `429` codes, the `Retry-After` header, and body `"Rate limit exceeded."`.
- [ ] `README.md` records the actual headers and status codes from all curl checks.

#### Hints (no direct answer)

- Recall that `AddDefaultPolicy` applies to endpoints automatically when you do NOT call `RequireCors(...)` with a specific name; for the public endpoint you must explicitly "override" the default with the `"Public"` policy via `RequireCors("Public")`.
- For preflight you do not need to write your own `OPTIONS` handler — the CORS middleware intercepts `OPTIONS` itself. Just make sure it sits earlier in the pipeline than endpoint matching.
- If `429` does not arrive even after 60 requests, check: does the endpoint actually use the `"TokenBucket"` policy and not the global limiter; and is curl not caching the response (use `-H "Cache-Control: no-cache"` or vary the URL).
- If you want the `partitionKey` to be stable while testing from one host, you can temporarily log its value to confirm the IP is actually being differentiated.
- `OnRejected` receives an `OnRejectedContext` with an `HttpContext` field and a `LeaseId` — do not confuse it with the `HttpContext` from regular middleware.

#### Reference solution walk-through

```csharp
// Program.cs — C# 12 / .NET 8
// CORS + Rate Limiting for "Courier-online"
using System.Threading.RateLimiting;
using Microsoft.AspNetCore.RateLimiting;

var builder = WebApplication.CreateBuilder(args);

// === 1. CORS: default for credentialed + named Public ===
builder.Services.AddCors(options =>
{
    // Default — strictly three origins, credentialed requests allowed
    options.AddDefaultPolicy(policy => policy
        .WithOrigins("https://shop.example.com", "https://admin.example.com", "https://partner.example.com")
        .AllowAnyHeader()
        .WithMethods("GET", "POST", "PUT", "DELETE")
        .AllowCredentials()                // safe because origins are explicit
        .SetPreflightMaxAge(TimeSpan.FromMinutes(10)));

    // Public — read-only, no credentials
    // AllowAnyOrigin is only valid together with DisallowCredentials
    options.AddPolicy("Public", policy => policy
        .AllowAnyOrigin()
        .WithMethods("GET")
        .AllowAnyHeader()
        .DisallowCredentials());
});

// === 2. Rate limiter: Fixed Window global + Token Bucket for heavy endpoints ===
builder.Services.AddRateLimiter(options =>
{
    // Global Fixed Window, keyed by client IP
    options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(httpContext =>
        RateLimiter.GetPartitionedHTTPRateLimiter(
            partitionKey: httpContext.Connection.RemoteIpAddress?.ToString() ?? "anon",
            factory: key => new FixedWindowRateLimiter(new FixedWindowRateLimiterOptions
            {
                AutoReplenishment = true,
                PermitLimit = 100,
                Window = TimeSpan.FromMinutes(1)
            })));

    // Token Bucket — controlled bursts for /api/report
    options.AddPolicy("TokenBucket", httpContext =>
        RateLimiter.GetPartitionedHTTPRateLimiter(
            partitionKey: httpContext.Connection.RemoteIpAddress?.ToString() ?? "anon",
            factory: key => new TokenBucketRateLimiter(new TokenBucketRateLimiterOptions
            {
                TokenLimit = 50,
                TokensPerPeriod = 20,
                ReplenishmentPeriod = TimeSpan.FromSeconds(1),
                QueueLimit = 0,                       // no queue → instant 429
                QueueProcessingOrder = QueueProcessingOrder.OldestFirst,
                AutoReplenishment = true
            })));

    // Custom 429 response
    options.OnRejected = async (context, cancellationToken) =>
    {
        context.HttpContext.Response.StatusCode = StatusCodes.Status429TooManyRequests;
        context.HttpContext.Response.Headers["Retry-After"] = "60";
        await context.HttpContext.Response.WriteAsync("Rate limit exceeded.", cancellationToken);
    };
});

var app = builder.Build();

// Middleware order is critical (see lesson)
app.UseRouting();
app.UseCors();          // after routing, before endpoints
app.UseRateLimiter();   // after CORS, otherwise preflight gets 429

// Public endpoint — open CORS policy, no limit
app.MapGet("/api/public", () => Results.Ok(new { message = "public" }))
   .RequireCors("Public")
   .DisableRateLimiting();

// Protected credentialed endpoint — default CORS
app.MapGet("/api/profile", () => Results.Ok(new { user = "demo", role = "courier" }));

// Heavy endpoint — token bucket
app.MapGet("/api/report", () => Results.Ok(new { generatedAt = DateTime.UtcNow }))
   .EnableRateLimiting("TokenBucket");

app.Run();
```

Line-by-line walk-through:

- `using System.Threading.RateLimiting;` and `using Microsoft.AspNetCore.RateLimiting;` bring in namespaces that ship with .NET 8 — no NuGet packages required. The lesson explicitly noted that `Microsoft.AspNetCore.RateLimiting` has been available since .NET 7 with no third-party dependencies.
- `AddCors` with two policies demonstrates the core pattern: one default for protected credentialed requests (explicit origins), one named for public ones. This directly applies the best practice "always enumerate allowed origins explicitly instead of `AllowAnyOrigin` for protected APIs".
- `WithOrigins(...)` + `AllowCredentials()` is safe because the origins are an explicit list — the browser echoes back exactly the origin from the `Origin:` request header if it is in the list. The forbidden combination `AllowAnyOrigin().AllowCredentials()` is absent here, which is the central acceptance requirement.
- `SetPreflightMaxAge(TimeSpan.FromMinutes(10))` caches the preflight result for 10 minutes, cutting the number of `OPTIONS` round-trips — exactly as recommended in the lesson's best practices.
- The `"Public"` policy uses `AllowAnyOrigin()` together with `DisallowCredentials()`. That is the only safe combination involving `AllowAnyOrigin`: without credentialed requests the server can safely return `Access-Control-Allow-Origin: *`. If `AllowCredentials()` appeared here, acceptance would fail.
- `AddRateLimiter` configures `GlobalLimiter` via `PartitionedRateLimiter.Create<HttpContext, string>`. The type parameter `string` is the type of the `partitionKey`. Inside the lambda we read the client IP with a fallback to `"anon"` — this resolves the classic "shared limit for everyone" problem that the lesson called out as a common mistake. A constant key would let one abuser drain everyone's quota.
- `FixedWindowRateLimiter` with `AutoReplenishment = true` and `PermitLimit = 100` per minute provides a simple baseline that protects against overload. `AutoReplenishment = true` means the window resets automatically on a timer, with no manual `TryReplenish`.
- The named `"TokenBucket"` policy via `options.AddPolicy(...)` creates a separate limiter for heavy endpoints. `TokenBucketRateLimiter` with `TokenLimit = 50`, `TokensPerPeriod = 20`, `ReplenishmentPeriod = 1s` permits controlled bursts up to 50 requests while keeping an average rate of 20 requests/second — precisely the "heavy endpoints" scenario for which the lesson recommended Token Bucket.
- `QueueLimit = 0` guarantees an instant `429` on overflow — no queue, no cascading timeouts. The lesson warned that `QueueLimit > 0` changes the semantics and can cause client timeouts.
- `OnRejected` is the single point where the `429` response can be customized. We set the status code, the `Retry-After: 60` header (matching the Fixed Window), and the body. The lesson demanded "return `Retry-After` with `429`".
- The order `UseRouting → UseCors → UseRateLimiter → endpoints` is the critical order from the lesson. Swap `UseCors` and `UseRateLimiter` and the preflight `OPTIONS` will get `429` before CORS can answer, breaking the whole flow. Drop `UseCors` entirely and the policy is registered but never applied — a common mistake.
- The `/api/public` endpoint explicitly applies the `"Public"` policy via `RequireCors("Public")` and disables rate limiting via `DisableRateLimiting()` — a public read-only endpoint should not be throttled.
- The `/api/profile` endpoint does not call `RequireCors` — the default policy applies. That is correct, because `AddDefaultPolicy` exists precisely for this purpose.
- The `/api/report` endpoint uses `EnableRateLimiting("TokenBucket")` — switching it from the global Fixed Window to Token Bucket, as required.

#### Going deeper (bonus)

1. Add a `SlidingWindowRateLimiter` as a third named policy and apply it to `/api/search`. Compare its behavior with Fixed Window at the window boundary: fire a burst in the last 5 seconds of one window and the first 5 seconds of the next, record the difference in `README.md`.
2. Implement a `ConcurrencyLimiter` for `/api/export`, capping the number of in-flight requests (not the frequency) at 3. Verify that the 4th concurrent request gets `429`.
3. Add JWT authorization and build the `partitionKey` from `userId` for authenticated requests, keeping the IP key only for anonymous ones. Demonstrate that the limit becomes per-user.
4. Write an integration test with `WebApplicationFactory` that emulates an `Origin` and asserts the presence of `Access-Control-Allow-Origin` in the response, plus a test that fires 60 requests and expects at least one `429`.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Проект `CourierApi` собирается и запускается на .NET 8.
- [ ] (RU) В `Program.cs` top-level statements, два `AddCors`-политики, запретная комбинация отсутствует.
- [ ] (RU) `AddRateLimiter` с `GlobalLimiter` (Fixed Window) и `"TokenBucket"`, `partitionKey` по IP.
- [ ] (RU) `OnRejected` возвращает `429` + `Retry-After` + тело.
- [ ] (RU) Middleware в порядке `routing → cors → rate limiter → endpoints`.
- [ ] (RU) Три эндпоинта с правильными политиками CORS и rate limiting.
- [ ] (RU) `README.md` содержит заголовки и коды из curl-проверок.
- [ ] (EN) Project `CourierApi` builds and runs on .NET 8.
- [ ] (EN) `Program.cs` uses top-level statements, two `AddCors` policies, forbidden combination absent.
- [ ] (EN) `AddRateLimiter` with `GlobalLimiter` (Fixed Window) and `"TokenBucket"`, `partitionKey` from IP.
- [ ] (EN) `OnRejected` returns `429` + `Retry-After` + body.
- [ ] (EN) Middleware ordered `routing → cors → rate limiter → endpoints`.
- [ ] (EN) Three endpoints with correct CORS and rate-limit policies.
- [ ] (EN) `README.md` contains headers and codes from curl checks.

#### Ресурсы / Resources
- [Microsoft Learn — Enable CORS in ASP.NET Core — https://learn.microsoft.com/aspnet/core/security/cors]
- [Microsoft Learn — Rate limiting in ASP.NET Core — https://learn.microsoft.com/aspnet/core/performance/rate-limit]
- [MDN — CORS overview — https://developer.mozilla.org/docs/Web/HTTP/CORS]
- [Microsoft Learn — System.Threading.RateLimiting — https://learn.microsoft.com/dotnet/api/system.threading.ratelimiting]
