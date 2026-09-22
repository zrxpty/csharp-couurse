---
[← К уроку M13-L01](lesson-M13-L01-aspnet-architecture-host.md) | [⬆ К модулю M13](../README.md) | [Следующее ДЗ →](homework-M13-L02-middleware-pipeline.md)
---

### Домашнее задание M13-L01: Архитектура ASP.NET Core, хост, WebApplication / Homework M13-L01: ASP.NET Core architecture, host, WebApplication

**Урок / Lesson:** M13-L01
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) Самостоятельно собрать минимальное ASP.NET Core-приложение на .NET 8 / C# 12 через `WebApplication.CreateBuilder`, настроить Kestrel и конвейер middleware в правильном порядке, разделить конфигурацию по средам, корректно зарегистрировать и зарезолвить сервисы в DI и продемонстрировать понимание времени жизни приложения. (EN) Build a minimal ASP.NET Core application on .NET 8 / C# 12 using `WebApplication.CreateBuilder`, configure Kestrel and the middleware pipeline in the correct order, split configuration by environment, correctly register and resolve DI services, and demonstrate understanding of application lifetime.

#### Связь с уроком / Connection to the lesson

(RU) Урок вводит ключевые понятия архитектуры ASP.NET Core: хост как «здание» приложения, минимальную хостинг-модель с `WebApplication.CreateBuilder`, сервер Kestrel, конвейер middleware и внедрение зависимостей. ДЗ закрепляет все эти понятия на практике: вы пройдёте путь от пустого проекта до запущенного хоста с правильным порядком middleware и разделением по средам, опираясь на примеры кода, best practices и частые ошибки из урока.

(EN) The lesson introduces the core concepts of ASP.NET Core architecture: the host as the "building" of the application, the minimal hosting model with `WebApplication.CreateBuilder`, the Kestrel server, the middleware pipeline, and dependency injection. This homework reinforces all of these concepts in practice: you will go from an empty project to a running host with the correct middleware order and environment-based splitting, relying on the code examples, best practices, and common mistakes from the lesson.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Представьте, что вы присоединяетесь к небольшой команде, которая начинает новый внутренний сервис «HealthCheck API» для мониторинга состояния микросервисов компании. Технический лидер уже выбрал стек: C# 12 и .NET 8, модель минимального хостинга (`WebApplication.CreateBuilder`), Kestrel как сервер, в продакшене — за обратным прокси Nginx. Вам поручено собрать «каркас» приложения: пустой проект, правильно сконфигурированный хост, конвейер middleware в корректном порядке, разделение конфигурации по средам (Development/Staging/Production) и один демо-endpoint `/health`, который возвращает статус и имя текущей среды. Каркас должен быть готов к тому, чтобы следующие разработчики добавляли контроллеры, аутентификацию и интеграции, не переделывая фундамент.

Это задание моделирует ровно тот шаг, на котором чаще всего совершают ошибки начинающие: путают `WebApplication.CreateBuilder` и `Host.CreateDefaultBuilder`, нарушают порядок middleware (`UseAuthorization` до `UseRouting`), хардкодят порты, тянут scoped-сервисы из root scope и забывают отключить заголовок сервера. Пройдя задание, вы получите мышечную память для правильного «скелета» ASP.NET Core-приложения, который потом будете воспроизводить в каждом новом проекте. Особое внимание уделите тому, чтобы код оставался порто-агностичным, конфигурация бралась из `ASPNETCORE_URLS` и `appsettings.json`, а среда определялась через переменную `ASPNETCORE_ENVIRONMENT`.

#### Что нужно сделать (пошагово)

1. **Создайте пустой web-проект.** Из корня репозитория выполните:
   ```
   dotnet new web -n HealthCheckApi -o src/HealthCheckApi --framework net8.0
   cd src/HealthCheckApi
   ```
   Откройте `src/HealthCheckApi/HealthCheckApi.csproj` и убедитесь, что `TargetFramework` равен `net8.0`, а `ImplicitUsings` включён. Добавьте при необходимости `Nullable` в `enable`. Соберите проект командой `dotnet build` — он должен скомпилироваться без ошибок.

2. **Изучите сгенерированный `Program.cs`.** Шаблон `dotnet new web` уже использует `WebApplication.CreateBuilder(args)`. Убедитесь, что в файле нет ссылок на устаревший `WebHost.CreateDefaultBuilder` или класс `Startup`. Если они есть — удалите.

3. **Настройте конфигурацию по средам.** Добавьте в проект два файла: `appsettings.json` (базовый) и `appsettings.Development.json`. В базовом укажите секцию `"HealthCheck"` с параметрами, например `"TimeoutMs": 1000`. В Development-файле переопределите `"TimeoutMs": 250`. Убедитесь, что оба файла попадают в выходной каталог (`CopyToOutputDirectory` не нужен — ASP.NET Core сам подхватывает `appsettings.{Environment}.json`).

4. **Явно сконфигурируйте Kestrel.** В `Program.cs` добавьте `builder.WebHost.ConfigureKestrel(options => { options.AddServerHeader = false; });`. Это отключит раскрытие версии сервера в HTTP-заголовке `Server`, что соответствует best practice из урока.

5. **Зарегистрируйте сервисы в DI.** Создайте класс `HealthSettings` со свойством `int TimeoutMs`. Зарегистрируйте его как `IOptions<HealthSettings>` через `builder.Services.Configure<HealthSettings>(builder.Configuration.GetSection("HealthCheck"))`. Добавьте также `builder.Services.AddEndpointsApiExplorer()` и `builder.Services.AddSwaggerGen()` — они пригодятся для углубления.

6. **Выстройте middleware в правильном порядке.** После `var app = builder.Build();` зарегистрируйте: в Development — `UseSwagger` + `UseSwaggerUI`; затем строго в таком порядке — `UseHttpsRedirection`, `UseRouting`, `UseAuthentication`, `UseAuthorization`. Помните из урока: нарушение порядка ломает применение прав.

7. **Добавьте endpoint `/health`.** Через `app.MapGet("/health", ...)` возвращайте `Results.Ok(new { status = "healthy", env = app.Environment.EnvironmentName, timeoutMs = ... })`, где `timeoutMs` берётся из зарегистрированного `IOptions<HealthSettings>`. Внедряйте `IOptions<HealthSettings>` через параметры лямбды — НЕ вытаскивайте из `app.Services`.

8. **Запустите и проверьте.** Установите переменные окружения:
   ```
   $env:ASPNETCORE_ENVIRONMENT="Development"
   $env:ASPNETCORE_URLS="http://localhost:5080"
   dotnet run
   ```
   В другом терминале выполните `curl http://localhost:5080/health` — ожидается JSON с `"env":"Development"` и `"timeoutMs":250`. Заголовок `Server` должен отсутствовать.

9. **Переключите среду и перепроверьте.** Установите `ASPNETCORE_ENVIRONMENT=Production` и `ASPNETCORE_URLS=http://localhost:5081`, перезапустите. Проверьте, что `/health` теперь возвращает `"env":"Production"` и `"timeoutMs":1000`, а Swagger UI по адресу `/swagger` недоступен (он включается только в Development).

10. **Проверьте заголовок сервера.** Выполните `curl -I http://localhost:5081/health` и убедитесь, что заголовка `Server: Kestrel` нет — это следствие `AddServerHeader = false`.

#### Требования к решению

- Проект `HealthCheckApi` на .NET 8, C# 12, с включёнными `ImplicitUsings` и `Nullable`. Сборка `dotnet build` без предупреждений.
- Используется именно `WebApplication.CreateBuilder(args)` — никакой ссылки на `Host.CreateDefaultBuilder` или `WebHost.CreateDefaultBuilder` быть не должно.
- Kestrel явно сконфигурирован: `AddServerHeader = false`. Порт берётся из `ASPNETCORE_URLS` (или `appsettings.json`), в коде нет хардкода портов (`ListenAnyIP(80)` и т.п.).
- Middleware зарегистрированы в порядке: `UseHttpsRedirection` → `UseRouting` → `UseAuthentication` → `UseAuthorization` → endpoints (`MapGet`, `MapControllers`). Swagger/подробные ошибки — только в Development.
- Сервисы зарегистрированы в DI: `AddEndpointsApiExplorer`, `AddSwaggerGen`, `Configure<HealthSettings>`. Никакого ручного `new HealthSettings()` в обработчике.
- Endpoint `/health` возвращает JSON с полями `status`, `env`, `timeoutMs`, где `env` совпадает с `ASPNETCORE_ENVIRONMENT`, а `timeoutMs` — со значением из соответствующего `appsettings.{Environment}.json`.
- Scoped/вычисляемые сервисы резолвятся только через параметры метода, не из `app.Services`.
- Код читаемый, с двуязычными комментариями (RU+EN) в ключевых местах: настройка Kestrel, порядок middleware, регистрация DI.
- Приложение запускается в обеих средах и корректно меняет поведение без перекомпиляции.

#### Тонкости и подводные камни

- **Не путайте билдеры.** `WebApplication.CreateBuilder(args)` — для веб-приложений, он сразу регистрирует Kestrel и middleware-инфраструктуру. `Host.CreateDefaultBuilder(args)` — это Generic Host без веб-специфики; если использовать его для веб-приложения, сервер по умолчанию не подключится. В этом ДЗ — только `WebApplication.CreateBuilder`.
- **Порядок middleware критичен.** `UseAuthorization` до `UseRouting` не применит атрибуты `[Authorize]`, потому что маршрутизация ещё не сопоставила запрос с endpoint. Канонический порядок из урока: Routing → Authentication → Authorization → endpoints. `UseHttpsRedirection` ставят раньше endpoint-маппинга, но после среды-зависимых middleware вроде Swagger.
- **Scoped из root scope — ошибка.** Не вызывайте `app.Services.GetService<IMyScoped>()` внутри обработчика. `app.Services` — это root scope, и резолв из него scoped-сервиса приводит к `InvalidOperationException` про captive scope либо к утечке. Внедряйте через параметры: `app.MapGet("/health", (IOptions<HealthSettings> opts) => ...)`.
- **Заголовок сервера.** По умолчанию Kestrel добавляет `Server: Kestrel` — это раскрывает технологию. `AddServerHeader = false` убирает его. Это мелочь, но часть безопасного продакшен-дефолта.
- **Порты через конфигурацию.** Не хардкодьте `ListenAnyIP(80)`. Используйте `ASPNETCORE_URLS` или секцию `Kestrel:Endpoints` в `appsettings.json`. Тогда один и тот же артефакт можно деплоить на разные порты без перекомпиляции.
- **Среда через переменную.** `ASPNETCORE_ENVIRONMENT` определяет `app.Environment.EnvironmentName` и то, какой `appsettings.{Environment}.json` загрузится. `Development`, `Staging`, `Production` — предопределённые имена, для них есть `IsDevelopment()`, `IsStaging()`, `IsProduction()`.
- **HTTPS за прокси.** Если Kestrel за Nginx с TLS-терминацией, нужно передавать `X-Forwarded-Proto` и вызывать `UseForwardedHeaders`, иначе редиректы и `Secure`-флаги cookie сломаются. В этом ДЗ это не требуется, но помните как частую ошибку из урока.

#### Критерии приёмки

- [ ] Проект `HealthCheckApi` создан через `dotnet new web`, TargetFramework = `net8.0`, C# 12.
- [ ] В `Program.cs` используется `WebApplication.CreateBuilder(args)`, без `Host.CreateDefaultBuilder`/`WebHost.CreateDefaultBuilder`.
- [ ] Kestrel сконфигурирован с `AddServerHeader = false`.
- [ ] Порт приложения берётся из `ASPNETCORE_URLS` (или `appsettings.json`), хардкода портов в коде нет.
- [ ] Есть `appsettings.json` и `appsettings.Development.json` с секцией `HealthCheck.TimeoutMs`.
- [ ] `HealthSettings` зарегистрирован через `Configure<HealthSettings>` из секции `HealthCheck`.
- [ ] `AddEndpointsApiExplorer` и `AddSwaggerGen` зарегистрированы.
- [ ] Middleware идут в порядке: `UseHttpsRedirection` → `UseRouting` → `UseAuthentication` → `UseAuthorization`.
- [ ] Swagger и SwaggerUI включены только в Development.
- [ ] Endpoint `/health` возвращает `status`, `env`, `timeoutMs` в формате JSON.
- [ ] `env` в ответе совпадает с `ASPNETCORE_ENVIRONMENT`.
- [ ] `timeoutMs` в ответе равен 250 в Development и 1000 в Production.
- [ ] В ответе отсутствует заголовок `Server` (`curl -I`).
- [ ] В Development доступен `/swagger`, в Production — нет.
- [ ] В коде нет резолва scoped-сервисов из `app.Services`; все внедрения — через параметры.
- [ ] Код содержит двуязычные комментарии RU+EN в ключевых местах.

#### Подсказки (без прямого ответа)

- Перечитайте раздел «Theory» урока про аналогию с рестораном — она подсказывает, в каком порядке идут шаги: билдер → сервисы → Build → middleware → Run.
- Для `IOptions<T>` нужна `using Microsoft.Extensions.Options;`. Зарегистрируйте через `builder.Services.Configure<T>(builder.Configuration.GetSection("..."))`.
- В лямбде `app.MapGet("/health", (...) => ...)` параметры резолвятся из DI автоматически — это и есть правильный способ.
- Чтобы проверить отсутствие заголовка `Server`, используйте `curl -I` (только заголовки) или PowerShell `Invoke-WebRequest -Method Head`.
- Если Swagger не запускается — проверьте, что `UseSwagger` и `UseSwaggerUI` вызываются внутри `if (app.Environment.IsDevelopment())`.

#### Эталонное решение (разбор)

```csharp
// Program.cs — HealthCheckApi, .NET 8 / C# 12
// Minimal hosting model, правильный порядок middleware, разделение по средам.

using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Options;

// 1) Создаём построитель хоста с преднастроенными умолчаниями.
//    CreateBuilder сам подключает конфигурацию, логирование, DI и Kestrel.
// 1) Create the host builder with preconfigured defaults.
//    CreateBuilder wires up configuration, logging, DI, and Kestrel itself.
var builder = WebApplication.CreateBuilder(args);

// 2) Регистрируем сервисы в DI.
// 2) Register services in the DI container.
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// Привязываем секцию HealthCheck к strongly-typed объекту.
// Bind the HealthCheck section to a strongly-typed options object.
builder.Services.Configure<HealthSettings>(
    builder.Configuration.GetSection("HealthCheck"));

// 3) Явная настройка Kestrel: не раскрываем версию сервера.
// 3) Explicit Kestrel config: do not leak the server version.
builder.WebHost.ConfigureKestrel(options =>
{
    options.AddServerHeader = false;
});

// 4) Строим приложение.
// 4) Build the application.
var app = builder.Build();

// 5) Middleware, зависящие от среды — только в Development.
// 5) Environment-specific middleware — Development only.
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

// 6) Конвейер middleware. ПОРЯДОК ВАЖЕН: HTTPS → Routing → Auth → AuthZ.
// 6) Middleware pipeline. ORDER MATTERS: HTTPS → Routing → Auth → AuthZ.
app.UseHttpsRedirection();
app.UseRouting();
app.UseAuthentication();
app.UseAuthorization();

// 7) Endpoint /health. Параметры резолвятся из DI — НЕ из app.Services.
// 7) /health endpoint. Parameters are resolved from DI — NOT from app.Services.
app.MapGet("/health", (IOptions<HealthSettings> opts) =>
{
    var settings = opts.Value;
    return Results.Ok(new
    {
        status = "healthy",
        env = app.Environment.EnvironmentName,
        timeoutMs = settings.TimeoutMs
    });
});

// 8) Запускаем хост. Run() блокирует до завершения работы.
// 8) Start the host. Run() blocks until shutdown.
app.Run();

// Класс параметров (можно вынести в отдельный файл).
// Options class (can be moved to a separate file).
public class HealthSettings
{
    public int TimeoutMs { get; init; } = 1000;
}
```

**Разбор по строкам.** Строки с `WebApplication.CreateBuilder(args)` — это вход в минимальную хостинг-модель: один вызов заменяет десятки строк шаблонного кода из старой модели с `WebHostBuilder` и `Startup`. Концепция из урока: «хост — это здание приложения», и `CreateBuilder` арендует это здание с уже подведёнными коммуникациями (конфигурация из `appsettings.json`, переменные окружения, логирование, DI, Kestrel).

`AddEndpointsApiExplorer` + `AddSwaggerGen` регистрируют инфраструктуру Swagger как singleton-сервисы — это best practice: инструменты разработки подключаются через DI, а не тащатся вручную. `Configure<HealthSettings>` применяет паттерн Options: конфигурация из секции `HealthCheck` маппится на strongly-typed объект, что избавляет от строковых ключей и даёт валидацию на старте при желании.

`ConfigureKestrel` с `AddServerHeader = false` реализует рекомендацию урока «не раскрывайте заголовок сервера». Это часть безопасного дефолта: в продакшене мы не хотим сообщать версию Kestrel/.NET потенциальным злоумышленникам. Порт при этом НЕ хардкодится — он берётся из `ASPNETCORE_URLS`, что соответствует best practice «код остаётся порто-agnostic».

`builder.Build()` «открывает ресторан» — хост готов к запуску. Дальше идёт конвейер middleware, и здесь критичен порядок из урока. `UseHttpsRedirection` стоит первым после среды-зависимых компонент: он перенаправляет HTTP на HTTPS до того, как запрос пойдёт по маршрутам. `UseRouting` включает сопоставление запроса с endpoint — без него `UseAuthorization` не будет знать, к какому endpoint применяется `[Authorize]`. `UseAuthentication` идёт до `UseAuthorization`: сначала определяем, кто пользователь, потом — что ему можно. Нарушение этого порядка — частая ошибка из урока, ведущая к «права не применяются».

Блок `if (app.Environment.IsDevelopment())` разделяет конфигурацию по средам — это best practice: Swagger и подробные ошибки в dev, безопасные заглушки в prod. Среда определяется переменной `ASPNETCORE_ENVIRONMENT` и доступна через `app.Environment.EnvironmentName`.

Endpoint `/health` демонстрирует правильный резолв зависимостей: `IOptions<HealthSettings>` берётся через параметр лямбды, а не из `app.Services`. Это защищает от captive scope — ошибки, описанной в уроке. Ответ содержит `env` (имя среды) и `timeoutMs` (значение из конфигурации), что позволяет проверить, что нужный `appsettings.{Environment}.json` подгрузился.

`app.Run()` запускает хост и блокирует поток до завершения (Ctrl+C или `StopApplication`). В тестах использовали бы `await app.RunAsync(stoppingToken)` для graceful shutdown, но для приложения достаточно синхронного `Run()`. Класс `HealthSettings` объявлен с `init`-сеттером — идиоматичный C# 12 для immutable конфигурационных объектов.

#### Задания на углубление (бонус)

1. **Добавьте `IHostApplicationLifetime` в endpoint.** Сделайте endpoint `/shutdown`, который вызывает `lifetime.StopApplication()` и корректно завершает хост. Убедитесь, что `app.Run()` завершается без исключения.
2. **Forwarded Headers.** Добавьте `UseForwardedHeaders` так, чтобы за прокси (имитируйте через заголовок `X-Forwarded-Proto=https`) корректно работали редиректы. Сравните поведение с и без.
3. **Связь с Generic Host.** В отдельной ветке попробуйте собрать то же приложение через `Host.CreateDefaultBuilder` + `ConfigureWebHostDefaults`. Сравните объём кода и поведение, напишите вывод, почему минимальная модель предпочтительнее.
4. **Валидация Options.** Подключите `AddOptions().Bind(...).Validate(...)` так, чтобы `TimeoutMs` обязан был быть в диапазоне 100–5000, и при нарушении приложение падало на старте с понятной ошибкой.

---

## Statement in English / Постановка на английском

#### Context & motivation

Imagine you are joining a small team that is starting a new internal service, "HealthCheck API", to monitor the health of the company's microservices. The tech lead has already chosen the stack: C# 12 and .NET 8, the minimal hosting model (`WebApplication.CreateBuilder`), Kestrel as the server, and in production Kestrel will sit behind an Nginx reverse proxy. You are tasked with building the "skeleton" of the application: an empty project, a correctly configured host, a middleware pipeline in the correct order, configuration split by environment (Development/Staging/Production), and a single demo endpoint `/health` that returns status and the current environment name. The skeleton must be ready for the next developers to add controllers, authentication, and integrations without rebuilding the foundation.

This assignment models exactly the step where beginners most often make mistakes: they confuse `WebApplication.CreateBuilder` with `Host.CreateDefaultBuilder`, break the middleware order (`UseAuthorization` before `UseRouting`), hardcode ports, pull scoped services from the root scope, and forget to disable the server header. By completing the assignment you will build muscle memory for the correct ASP.NET Core "skeleton" that you will then reproduce in every new project. Pay special attention to keeping the code port-agnostic, pulling configuration from `ASPNETCORE_URLS` and `appsettings.json`, and letting the environment be defined through the `ASPNETCORE_ENVIRONMENT` variable.

#### What to do step by step

1. **Create an empty web project.** From the repository root run:
   ```
   dotnet new web -n HealthCheckApi -o src/HealthCheckApi --framework net8.0
   cd src/HealthCheckApi
   ```
   Open `src/HealthCheckApi/HealthCheckApi.csproj` and verify that `TargetFramework` is `net8.0` and `ImplicitUsings` is enabled. Add `Nullable` set to `enable` if missing. Build the project with `dotnet build` — it must compile without errors.

2. **Inspect the generated `Program.cs`.** The `dotnet new web` template already uses `WebApplication.CreateBuilder(args)`. Make sure there are no references to the legacy `WebHost.CreateDefaultBuilder` or to a `Startup` class. If any exist — remove them.

3. **Configure settings per environment.** Add two files to the project: `appsettings.json` (base) and `appsettings.Development.json`. In the base file add a `"HealthCheck"` section with, for example, `"TimeoutMs": 1000`. In the Development file override `"TimeoutMs": 250`. No `CopyToOutputDirectory` is needed — ASP.NET Core automatically picks up `appsettings.{Environment}.json`.

4. **Configure Kestrel explicitly.** In `Program.cs` add `builder.WebHost.ConfigureKestrel(options => { options.AddServerHeader = false; });`. This disables leaking the server version in the `Server` HTTP header, matching the best practice from the lesson.

5. **Register services in DI.** Create a `HealthSettings` class with an `int TimeoutMs` property. Register it as `IOptions<HealthSettings>` via `builder.Services.Configure<HealthSettings>(builder.Configuration.GetSection("HealthCheck"))`. Also add `builder.Services.AddEndpointsApiExplorer()` and `builder.Services.AddSwaggerGen()` — they will be useful for the going-deeper tasks.

6. **Order the middleware correctly.** After `var app = builder.Build();` register the following: in Development — `UseSwagger` + `UseSwaggerUI`; then strictly in this order — `UseHttpsRedirection`, `UseRouting`, `UseAuthentication`, `UseAuthorization`. Remember from the lesson: a wrong order breaks authorization.

7. **Add the `/health` endpoint.** Through `app.MapGet("/health", ...)` return `Results.Ok(new { status = "healthy", env = app.Environment.EnvironmentName, timeoutMs = ... })`, where `timeoutMs` comes from the registered `IOptions<HealthSettings>`. Inject `IOptions<HealthSettings>` through the lambda parameters — do NOT pull it from `app.Services`.

8. **Run and verify.** Set the environment variables:
   ```
   $env:ASPNETCORE_ENVIRONMENT="Development"
   $env:ASPNETCORE_URLS="http://localhost:5080"
   dotnet run
   ```
   In another terminal run `curl http://localhost:5080/health` — expect JSON with `"env":"Development"` and `"timeoutMs":250`. The `Server` header must be absent.

9. **Switch the environment and re-check.** Set `ASPNETCORE_ENVIRONMENT=Production` and `ASPNETCORE_URLS=http://localhost:5081`, restart. Verify that `/health` now returns `"env":"Production"` and `"timeoutMs":1000`, and that the Swagger UI at `/swagger` is not available (it is enabled only in Development).

10. **Verify the server header.** Run `curl -I http://localhost:5081/health` and confirm there is no `Server: Kestrel` header — this is the effect of `AddServerHeader = false`.

#### Requirements

- The `HealthCheckApi` project targets .NET 8, C# 12, with `ImplicitUsings` and `Nullable` enabled. `dotnet build` produces no warnings.
- `WebApplication.CreateBuilder(args)` is used — there must be no reference to `Host.CreateDefaultBuilder` or `WebHost.CreateDefaultBuilder`.
- Kestrel is explicitly configured with `AddServerHeader = false`. The port comes from `ASPNETCORE_URLS` (or `appsettings.json`); there is no hardcoded port (`ListenAnyIP(80)`, etc.) in the code.
- Middleware are registered in the order: `UseHttpsRedirection` → `UseRouting` → `UseAuthentication` → `UseAuthorization` → endpoints (`MapGet`, `MapControllers`). Swagger and detailed errors are Development-only.
- Services are registered in DI: `AddEndpointsApiExplorer`, `AddSwaggerGen`, `Configure<HealthSettings>`. No manual `new HealthSettings()` inside a handler.
- The `/health` endpoint returns JSON with fields `status`, `env`, `timeoutMs`, where `env` matches `ASPNETCORE_ENVIRONMENT` and `timeoutMs` matches the value from the corresponding `appsettings.{Environment}.json`.
- Scoped/computed services are resolved only through method parameters, never from `app.Services`.
- The code is readable, with bilingual RU+EN comments at the key points: Kestrel configuration, middleware order, DI registration.
- The application runs in both environments and changes behavior correctly without recompilation.

#### Pitfalls

- **Do not confuse the builders.** `WebApplication.CreateBuilder(args)` is for web applications — it immediately registers Kestrel and the middleware infrastructure. `Host.CreateDefaultBuilder(args)` is the Generic Host without web specifics; if you use it for a web app, no default server is attached. In this homework — only `WebApplication.CreateBuilder`.
- **Middleware order is critical.** `UseAuthorization` before `UseRouting` will not apply `[Authorize]` attributes, because routing has not yet matched the request to an endpoint. The canonical order from the lesson is: Routing → Authentication → Authorization → endpoints. `UseHttpsRedirection` goes before endpoint mapping, but after environment-specific middleware such as Swagger.
- **Scoped from the root scope is a bug.** Do not call `app.Services.GetService<IMyScoped>()` inside a handler. `app.Services` is the root scope, and resolving a scoped service from it leads to an `InvalidOperationException` about a captive scope or to a leak. Inject through parameters: `app.MapGet("/health", (IOptions<HealthSettings> opts) => ...)`.
- **The server header.** By default Kestrel adds `Server: Kestrel`, which discloses the technology. `AddServerHeader = false` removes it. It is a small thing, but part of a safe production default.
- **Ports via configuration.** Do not hardcode `ListenAnyIP(80)`. Use `ASPNETCORE_URLS` or the `Kestrel:Endpoints` section in `appsettings.json`. Then the same artifact can be deployed to different ports without recompilation.
- **Environment via a variable.** `ASPNETCORE_ENVIRONMENT` defines `app.Environment.EnvironmentName` and which `appsettings.{Environment}.json` is loaded. `Development`, `Staging`, `Production` are predefined names, with matching `IsDevelopment()`, `IsStaging()`, `IsProduction()` helpers.
- **HTTPS behind a proxy.** If Kestrel sits behind Nginx with TLS termination, you need to forward `X-Forwarded-Proto` and call `UseForwardedHeaders`, otherwise redirects and the `Secure` cookie flag break. Not required in this homework, but remember it as a common mistake from the lesson.

#### Acceptance criteria

- [ ] The `HealthCheckApi` project is created via `dotnet new web`, TargetFramework = `net8.0`, C# 12.
- [ ] `Program.cs` uses `WebApplication.CreateBuilder(args)`, with no `Host.CreateDefaultBuilder`/`WebHost.CreateDefaultBuilder`.
- [ ] Kestrel is configured with `AddServerHeader = false`.
- [ ] The application port comes from `ASPNETCORE_URLS` (or `appsettings.json`); no port is hardcoded in code.
- [ ] Both `appsettings.json` and `appsettings.Development.json` exist with a `HealthCheck.TimeoutMs` section.
- [ ] `HealthSettings` is registered via `Configure<HealthSettings>` from the `HealthCheck` section.
- [ ] `AddEndpointsApiExplorer` and `AddSwaggerGen` are registered.
- [ ] Middleware are ordered: `UseHttpsRedirection` → `UseRouting` → `UseAuthentication` → `UseAuthorization`.
- [ ] Swagger and SwaggerUI are enabled only in Development.
- [ ] The `/health` endpoint returns `status`, `env`, `timeoutMs` as JSON.
- [ ] `env` in the response matches `ASPNETCORE_ENVIRONMENT`.
- [ ] `timeoutMs` in the response equals 250 in Development and 1000 in Production.
- [ ] The response has no `Server` header (`curl -I`).
- [ ] `/swagger` is reachable in Development and unreachable in Production.
- [ ] The code never resolves scoped services from `app.Services`; all injections go through parameters.
- [ ] The code has bilingual RU+EN comments at the key points.

#### Hints (no direct answer)

- Re-read the "Theory" section of the lesson with the restaurant analogy — it suggests the order of steps: builder → services → Build → middleware → Run.
- `IOptions<T>` needs `using Microsoft.Extensions.Options;`. Register it via `builder.Services.Configure<T>(builder.Configuration.GetSection("..."))`.
- In the `app.MapGet("/health", (...) => ...)` lambda, parameters are resolved from DI automatically — that is the correct way.
- To check the absence of the `Server` header, use `curl -I` (headers only) or PowerShell `Invoke-WebRequest -Method Head`.
- If Swagger does not start — make sure `UseSwagger` and `UseSwaggerUI` are called inside `if (app.Environment.IsDevelopment())`.

#### Reference solution walk-through

```csharp
// Program.cs — HealthCheckApi, .NET 8 / C# 12
// Minimal hosting model, correct middleware order, environment split.

using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Options;

// 1) Create the host builder with preconfigured defaults.
//    CreateBuilder wires up configuration, logging, DI, and Kestrel itself.
var builder = WebApplication.CreateBuilder(args);

// 2) Register services in the DI container.
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// Bind the HealthCheck section to a strongly-typed options object.
builder.Services.Configure<HealthSettings>(
    builder.Configuration.GetSection("HealthCheck"));

// 3) Explicit Kestrel config: do not leak the server version.
builder.WebHost.ConfigureKestrel(options =>
{
    options.AddServerHeader = false;
});

// 4) Build the application.
var app = builder.Build();

// 5) Environment-specific middleware — Development only.
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

// 6) Middleware pipeline. ORDER MATTERS: HTTPS → Routing → Auth → AuthZ.
app.UseHttpsRedirection();
app.UseRouting();
app.UseAuthentication();
app.UseAuthorization();

// 7) /health endpoint. Parameters are resolved from DI — NOT from app.Services.
app.MapGet("/health", (IOptions<HealthSettings> opts) =>
{
    var settings = opts.Value;
    return Results.Ok(new
    {
        status = "healthy",
        env = app.Environment.EnvironmentName,
        timeoutMs = settings.TimeoutMs
    });
});

// 8) Start the host. Run() blocks until shutdown.
app.Run();

// Options class (can be moved to a separate file).
public class HealthSettings
{
    public int TimeoutMs { get; init; } = 1000;
}
```

**Line-by-line walk-through.** The `WebApplication.CreateBuilder(args)` lines are the entry point into the minimal hosting model: a single call replaces dozens of boilerplate lines from the old model with `WebHostBuilder` and `Startup`. The lesson's concept is "the host is the building of the application", and `CreateBuilder` rents that building with utilities already connected (configuration from `appsettings.json`, environment variables, logging, DI, Kestrel).

`AddEndpointsApiExplorer` and `AddSwaggerGen` register the Swagger infrastructure as singleton services — a best practice: developer tooling is wired through DI instead of being pulled in manually. `Configure<HealthSettings>` applies the Options pattern: configuration from the `HealthCheck` section is mapped to a strongly-typed object, which removes stringly-typed keys and allows startup validation if desired.

`ConfigureKestrel` with `AddServerHeader = false` implements the lesson's recommendation "do not leak the server header". It is part of a safe default: in production we do not want to advertise the Kestrel/.NET version to potential attackers. The port is NOT hardcoded here — it comes from `ASPNETCORE_URLS`, matching the best practice "keep the code port-agnostic".

`builder.Build()` "opens the restaurant" — the host is ready to run. Next comes the middleware pipeline, where the order from the lesson is critical. `UseHttpsRedirection` goes first, after environment-specific components: it redirects HTTP to HTTPS before the request reaches routing. `UseRouting` enables matching the request to an endpoint — without it, `UseAuthorization` would not know which endpoint `[Authorize]` applies to. `UseAuthentication` comes before `UseAuthorization`: first we identify who the user is, then what they are allowed to do. Breaking this order is a common mistake from the lesson that leads to "authorization is not applied".

The `if (app.Environment.IsDevelopment())` block splits configuration by environment — a best practice: Swagger and detailed errors in dev, safe stubs in prod. The environment is defined by the `ASPNETCORE_ENVIRONMENT` variable and is available through `app.Environment.EnvironmentName`.

The `/health` endpoint demonstrates the correct way to resolve dependencies: `IOptions<HealthSettings>` is taken through a lambda parameter, not from `app.Services`. This protects against the captive scope error described in the lesson. The response contains `env` (the environment name) and `timeoutMs` (the value from configuration), which lets you verify that the right `appsettings.{Environment}.json` was loaded.

`app.Run()` starts the host and blocks the thread until shutdown (Ctrl+C or `StopApplication`). In tests we would use `await app.RunAsync(stoppingToken)` for a graceful shutdown, but for a regular application the synchronous `Run()` is enough. The `HealthSettings` class is declared with an `init` setter — the idiomatic C# 12 way for immutable configuration objects.

#### Going deeper (bonus)

1. **Add `IHostApplicationLifetime` to an endpoint.** Build a `/shutdown` endpoint that calls `lifetime.StopApplication()` and gracefully stops the host. Verify that `app.Run()` exits without throwing.
2. **Forwarded Headers.** Add `UseForwardedHeaders` so that behind a proxy (simulate it with the `X-Forwarded-Proto=https` header) redirects work correctly. Compare behavior with and without.
3. **Connection to the Generic Host.** In a separate branch try building the same app through `Host.CreateDefaultBuilder` + `ConfigureWebHostDefaults`. Compare the amount of code and behavior, and write a conclusion on why the minimal model is preferable.
4. **Options validation.** Wire up `AddOptions().Bind(...).Validate(...)` so that `TimeoutMs` must be in the 100–5000 range, and on violation the app fails fast at startup with a clear error.

---

#### Чек-лист сдачи / Submission checklist

- [ ] (RU) Проект `HealthCheckApi` создан и собирается без предупреждений.
- [ ] (RU) `Program.cs` использует `WebApplication.CreateBuilder(args)`.
- [ ] (RU) Kestrel: `AddServerHeader = false`, порт из конфигурации.
- [ ] (RU) `appsettings.json` и `appsettings.Development.json` с `HealthCheck.TimeoutMs`.
- [ ] (RU) Middleware в порядке HTTPS → Routing → Auth → AuthZ; Swagger только в Dev.
- [ ] (RU) `/health` возвращает `env` и `timeoutMs`, заголовка `Server` нет.
- [ ] (RU) Нет резолва scoped-сервисов из `app.Services`; комментарии RU+EN.
- [ ] (EN) The `HealthCheckApi` project is created and builds without warnings.
- [ ] (EN) `Program.cs` uses `WebApplication.CreateBuilder(args)`.
- [ ] (EN) Kestrel: `AddServerHeader = false`, port from configuration.
- [ ] (EN) `appsettings.json` and `appsettings.Development.json` with `HealthCheck.TimeoutMs`.
- [ ] (EN) Middleware ordered HTTPS → Routing → Auth → AuthZ; Swagger only in Dev.
- [ ] (EN) `/health` returns `env` and `timeoutMs`, no `Server` header.
- [ ] (EN) No scoped-service resolution from `app.Services`; RU+EN comments present.

#### Ресурсы / Resources

- [Microsoft Learn — ASP.NET Core fundamentals](https://learn.microsoft.com/aspnet/core/fundamentals/)
- [Microsoft Learn — Minimal hosting model / WebApplication.CreateBuilder](https://learn.microsoft.com/aspnet/core/fundamentals/minimal-apis/overview)
- [Microsoft Learn — Kestrel web server](https://learn.microsoft.com/aspnet/core/fundamentals/servers/kestrel)
- [Microsoft Learn — Host ASP.NET Core on Windows with IIS](https://learn.microsoft.com/aspnet/core/host-and-deploy/iis/)
- [Microsoft Learn — Middleware in ASP.NET Core](https://learn.microsoft.com/aspnet/core/fundamentals/middleware/)
- [Microsoft Learn — Options pattern in .NET](https://learn.microsoft.com/dotnet/core/extensions/options)
- [Microsoft Learn — Dependency injection in ASP.NET Core](https://learn.microsoft.com/aspnet/core/fundamentals/dependency-injection)
