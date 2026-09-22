[⬆ К модулю M13](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M13-L01: Архитектура ASP.NET Core, хост, WebApplication / ASP.NET Core architecture, host, WebApplication

**Модуль / Module:** M13
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

ASP.NET Core — это кроссплатформенный, высокопроизводительный фреймворк для построения веб-приложений и API. Его архитектура построена вокруг нескольких ключевых понятий: **хост (host)**, **сервер (server)**, **конвейер обработки запросов (middleware pipeline)** и **сервисы (DI)**. Давайте разберём их по порядку, используя простые аналогии.

**Хост — это «здание» приложения.** Хост инкапсулирует все ресурсы, необходимые для работы приложения: конфигурацию, логирование, внедрение зависимостей, сервер и время жизни приложения. Представьте ресторан: хост — это само здание вместе с кухней, персоналом и электросетью. Без здания ничего не работает, какие бы вкусные блюда (контроллеры) вы ни готовили.

В .NET 6+ появился **минимальный хостинг-модель (Minimal Hosting Model)**. Вместо двух раздельных объектов `WebHostBuilder` и `Startup`-класса теперь используется один объект `WebApplication`, который создаётся через `WebApplication.CreateBuilder(args)`. Этот вызов делает сразу много работы: настраивает конфигурацию из `appsettings.json` и переменных окружения, подключает логирование, регистрирует Kestrel как сервер по умолчанию и подготавливает DI-контейнер. Раньше для этого требовались десятки строк шаблонного кода — теперь одна строка.

**`WebApplication.CreateBuilder` vs `CreateBuilder` — не путайте.** `WebApplication.CreateBuilder(args)` — это новый API для минимальной модели. `Host.CreateDefaultBuilder(args)` — это более низкоуровневый通用-хост (Generic Host) без веб-специфики. Для веб-приложений всегда используйте `WebApplication.CreateBuilder`.

**Kestrel — это встроенный веб-сервер.** Kestrel — кроссплатформенный, асинхронный, основанный на библиотеке `libuv`/управляемых сокетах. По умолчанию он слушает `http://localhost:5000` и `https://localhost:5001` (порт 80/443 в продакшене через переменные окружения). Kestrel достаточно быстр и безопасен, чтобы работать напрямую в продакшене, особенно за обратным прокси.

**IIS out-of-process — обзор.** На Windows можно хостить приложение в IIS. В режиме **out-of-process** IIS работает как обратный прокси: он принимает HTTP-запрос и перенаправляет его в Kestrel, запущенный в отдельном процессе (через модуль ASP.NET Core). В режиме **in-process** (исторически был по умолчанию в .NET Core 2.x–5.x) приложение выполнялось прямо внутри рабочего процесса IIS `w3wp.exe` через сервер IIS HTTP Server. В .NET 6+ out-of-process используется реже, но поддерживается; in-process-сервер в .NET 8+ объявлен устаревшим в пользу Kestrel. Для большинства сценариев Kestrel за IIS/Nginx/HTTPS-терминатором — это современный стандарт.

**Конвейер middleware.** После `builder.Build()` мы получаем объект `app` типа `WebApplication`. На нём мы регистрируем middleware: `app.UseRouting()`, `app.UseAuthentication()`, `app.UseAuthorization()`, `app.MapGet(...)`. Порядок важен: middleware выполняются в порядке регистрации, и каждый может «обрезать» цепочку, не вызвав `next()`.

**Время жизни приложения.** `app.Run()` запускает хост и блокирует вызывающий поток до завершения (Ctrl+C или `IHostApplicationLifetime.StopApplication()`). `await app.RunAsync()` — асинхронный вариант. `app.Services` даёт доступ к root scope, но в обработчиках запросов всегда используйте scoped-сервисы через DI, не через root.

**Аналогия целиком:** `CreateBuilder` — вы арендуете здание и нанимаете персонал (DI). `builder.Services.Add...` — добавляете отделы (сервисы). `app.Build()` — открываете ресторан. `app.Use...` — выстраиваете порядок обслуживания гостей (middleware). `app.Run()` — ресторан начинает принимать клиентов.

#### Theory (EN)

ASP.NET Core is a cross-platform, high-performance framework for building web apps and APIs. Its architecture revolves around a few core concepts: the **host**, the **server**, the **middleware pipeline**, and **dependency injection (DI)**. Let's walk through them with simple analogies.

**The host is the "building" of your application.** The host encapsulates everything the app needs to run: configuration, logging, dependency injection, the server, and application lifetime. Think of a restaurant: the host is the building itself, complete with the kitchen, staff, and electricity. Without the building, nothing works — no matter how great the dishes (controllers) are.

Starting with .NET 6, we have the **Minimal Hosting Model**. Instead of two separate objects (`WebHostBuilder` and a `Startup` class), there is now a single `WebApplication` object created via `WebApplication.CreateBuilder(args)`. This single call does a lot of heavy lifting: it wires up configuration from `appsettings.json` and environment variables, sets up logging, registers Kestrel as the default server, and prepares the DI container. What used to require dozens of boilerplate lines is now one line.

**Don't confuse `WebApplication.CreateBuilder` with `Host.CreateDefaultBuilder`.** `WebApplication.CreateBuilder(args)` is the modern API for the minimal model. `Host.CreateDefaultBuilder(args)` is the lower-level Generic Host without web specifics. For web apps, always prefer `WebApplication.CreateBuilder`.

**Kestrel is the built-in web server.** Kestrel is cross-platform, asynchronous, and based on managed sockets. By default it listens on `http://localhost:5000` and `https://localhost:5001` (ports 80/443 in production via environment variables). Kestrel is fast and secure enough to serve production traffic directly, especially behind a reverse proxy.

**IIS out-of-process — overview.** On Windows you can host the app in IIS. In **out-of-process** mode, IIS acts as a reverse proxy: it receives the HTTP request and forwards it to Kestrel running in a separate process (via the ASP.NET Core Module). In **in-process** mode (historically the default in .NET Core 2.x–5.x), the app ran directly inside the IIS worker process `w3wp.exe` using the IIS HTTP Server. In .NET 6+ out-of-process is supported but used less frequently; the in-process server is deprecated in .NET 8+ in favor of Kestrel. For most scenarios, Kestrel behind IIS/Nginx/an HTTPS terminator is the modern standard.

**The middleware pipeline.** After `builder.Build()` you get an `app` of type `WebApplication`. You register middleware on it: `app.UseRouting()`, `app.UseAuthentication()`, `app.UseAuthorization()`, `app.MapGet(...)`. Order matters: middleware runs in registration order, and any middleware can short-circuit the chain by not calling `next()`.

**Application lifetime.** `app.Run()` starts the host and blocks the calling thread until shutdown (Ctrl+C or `IHostApplicationLifetime.StopApplication()`). `await app.RunAsync()` is the async variant. `app.Services` exposes the root scope, but inside request handlers always resolve scoped services through DI, never through the root.

**Full analogy:** `CreateBuilder` — you rent the building and hire the staff (DI). `builder.Services.Add...` — you add departments (services). `app.Build()` — you open the restaurant. `app.Use...` — you set the order in which guests are served (middleware). `app.Run()` — the restaurant starts accepting customers.

#### Пример кода / Code Example

```csharp
// Program.cs — Минимальный хостинг-модель, .NET 8 / C# 12
// Minimal hosting model, .NET 8 / C# 12

using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

// 1) Создаём построитель хоста с преднастроенными умолчаниями
//    (конфигурация, логирование, DI, Kestrel по умолчанию).
// 1) Create the host builder with preconfigured defaults
//    (configuration, logging, DI, Kestrel by default).
var builder = WebApplication.CreateBuilder(args);

// 2) Регистрируем сервисы в DI-контейнере.
// 2) Register services in the DI container.
builder.Services.AddControllers();        // Поддержка контроллеров MVC / API
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();          // Swagger для документации API

// 3) Явная настройка Kestrel (необязательно — по умолчанию уже разумно).
//    Порт берётся из конфигурации (ASPNETCORE_URLS или appsettings).
// 3) Explicit Kestrel config (optional — defaults are sensible).
//    Port is taken from configuration (ASPNETCORE_URLS or appsettings).
builder.WebHost.ConfigureKestrel(options =>
{
    options.AddServerHeader = false;       // Не раскрываем версию сервера / Don't leak server version
});

// 4) Строим приложение — хост готов к запуску.
// 4) Build the application — the host is ready to run.
var app = builder.Build();

// 5) Конвейер middleware. ПОРЯДОК ВАЖЕН.
// 5) Middleware pipeline. ORDER MATTERS.
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();                 // Редирект HTTP -> HTTPS / Redirect HTTP to HTTPS
app.UseRouting();                          // Включает маршрутизацию / Enables routing
app.UseAuthentication();                   // Идентификация пользователя / Identifies the user
app.UseAuthorization();                    // Проверка прав / Checks permissions

// Минимальный endpoint — без контроллера.
// Minimal endpoint — no controller needed.
app.MapGet("/health", () =>
{
    // Возвращаем 200 OK с простым статусом.
    // Return 200 OK with a simple status.
    return Results.Ok(new { status = "healthy", env = app.Environment.EnvironmentName });
});

// Endpoint-маршруты для контроллеров.
// Attribute-routed controllers.
app.MapControllers();

// 6) Запускаем хост. Run() блокирует до завершения работы.
// 6) Start the host. Run() blocks until shutdown.
app.Run();

// Для graceful shutdown в тестах используйте:
// For graceful shutdown in tests use:
// await app.RunAsync(stoppingToken);
```

#### Best Practices

- Используйте `WebApplication.CreateBuilder(args)` в новых проектах на .NET 6+ — это минимальный хостинг-модель, меньше шаблонного кода и единая точка настройки.
- Разделяйте конфигурацию middleware по средам через `app.Environment.IsDevelopment()` / `IsProduction()`: в dev — Swagger и детальные ошибки, в prod — безопасные заглушки.
- Не раскрывайте заголовок сервера (`AddServerHeader = false`) и всегда настраивайте HTTPS-редирект в продакшене.
- В продакшене ставьте Kestrel за обратным прокси (IIS, Nginx, Yandex ALB, App Gateway) для TLS-терминации, балансировки и защиты от DDoS.
- Резолвьте scoped-сервисы только через DI в рамках запроса; никогда не вытаскивайте их из `app.Services` (root scope) внутри обработчиков.

- Use `WebApplication.CreateBuilder(args)` for new .NET 6+ projects — it is the minimal hosting model, less boilerplate and a single configuration point.
- Split middleware configuration by environment via `app.Environment.IsDevelopment()` / `IsProduction()`: Swagger and detailed errors in dev, safe stubs in prod.
- Do not expose the server header (`AddServerHeader = false`) and always configure HTTPS redirection in production.
- Put Kestrel behind a reverse proxy (IIS, Nginx, Yandex ALB, App Gateway) in production for TLS termination, load balancing, and DDoS protection.
- Resolve scoped services only through DI within a request; never pull them from `app.Services` (root scope) inside handlers.

#### Частые ошибки / Common Mistakes

- Неправильный порядок middleware (например, `UseAuthorization()` до `UseRouting()`) → права не применяются. → Всегда: `UseRouting` → `UseAuthentication` → `UseAuthorization` → endpoints.
- Использование `Host.CreateDefaultBuilder` вместо `WebApplication.CreateBuilder` для веб-приложения → нет сервера по умолчанию. → Для веба всегда `WebApplication.CreateBuilder(args)`.
- Резолв scoped-сервиса из `app.Services` в обработчике → `InvalidOperationException` про captive scope или утечка памяти. → Внедряйте через параметры метода/конструктор контроллера.
- Запуск Kestrel без HTTPS в продакшене, «потому что за прокси» → редиректы и cookie-флаги ломаются. → Терминируйте TLS на прокси и передавайте `X-Forwarded-Proto`, либо используйте HTTPS внутри.
- Правка портов прямо в коде `ListenAnyIP(80)` → негибко при деплое. → Используйте `ASPNETCORE_URLS` или `appsettings.json`, а код оставляйте порто-agnostic.

- Wrong middleware order (e.g., `UseAuthorization()` before `UseRouting()`) → authorization is not applied. → Always: `UseRouting` → `UseAuthentication` → `UseAuthorization` → endpoints.
- Using `Host.CreateDefaultBuilder` instead of `WebApplication.CreateBuilder` for a web app → no default server. → For web, always `WebApplication.CreateBuilder(args)`.
- Resolving a scoped service from `app.Services` inside a handler → `InvalidOperationException` about captive scope or a memory leak. → Inject via method parameters / controller constructor.
- Running Kestrel without HTTPS in production "because it's behind a proxy" → redirects and cookie flags break. → Terminate TLS on the proxy and forward `X-Forwarded-Proto`, or use HTTPS internally.
- Hardcoding ports in code (`ListenAnyIP(80)`) → inflexible at deployment. → Use `ASPNETCORE_URLS` or `appsettings.json` and keep the code port-agnostic.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Я создаю приложение через `WebApplication.CreateBuilder(args)`, а не через устаревший `WebHost.CreateDefaultBuilder`.
- [ ] Я понимаю, что `CreateBuilder` автоматически настраивает конфигурацию, логирование, DI и Kestrel.
- [ ] Я могу назвать роль Kestrel и объяснить, зачем перед ним ставят обратный прокси.
- [ ] Я знаю разницу между IIS in-process и out-of-process и что in-process устарел в .NET 8+.
- [ ] Middleware зарегистрированы в правильном порядке: Routing → Authentication → Authorization → endpoints.
- [ ] Порты и URL настраиваются через конфигурацию/переменные окружения, а не хардкодом.
- [ ] Scoped-сервисы я получаю только через DI в рамках запроса, не из root scope.

- [ ] I create the app via `WebApplication.CreateBuilder(args)`, not the legacy `WebHost.CreateDefaultBuilder`.
- [ ] I understand that `CreateBuilder` automatically wires up configuration, logging, DI, and Kestrel.
- [ ] I can describe Kestrel's role and explain why a reverse proxy sits in front of it.
- [ ] I know the difference between IIS in-process and out-of-process and that in-process is deprecated in .NET 8+.
- [ ] Middleware are registered in the correct order: Routing → Authentication → Authorization → endpoints.
- [ ] Ports and URLs are configured via configuration/environment variables, not hardcoded.
- [ ] Scoped services are resolved only through DI within a request, never from the root scope.

#### Ресурсы / Resources

- [Microsoft Learn — ASP.NET Core fundamentals](https://learn.microsoft.com/aspnet/core/fundamentals/)
- [Microsoft Learn — WebApplication.CreateBuilder / Minimal hosting model](https://learn.microsoft.com/aspnet/core/fundamentals/minimal-apis/overview)
- [Microsoft Learn — Kestrel web server](https://learn.microsoft.com/aspnet/core/fundamentals/servers/kestrel)
- [Microsoft Learn — Host ASP.NET Core on Windows with IIS](https://learn.microsoft.com/aspnet/core/host-and-deploy/iis/)
- [Microsoft Learn — Middleware in ASP.NET Core](https://learn.microsoft.com/aspnet/core/fundamentals/middleware/)

---

[⬆ К модулю M13](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
