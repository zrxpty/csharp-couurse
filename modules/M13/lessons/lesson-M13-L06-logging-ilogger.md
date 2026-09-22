[⬆ К модулю M13](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M13-L06: Logging, ILogger, провайдеры, structured logging / Logging, ILogger, providers, structured logging

**Модуль / Module:** M13
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Логирование в ASP.NET Core — это не просто `Console.WriteLine`, а полноценная инфраструктура с единой абстракцией `ILogger`, набором поставщиков (providers) и поддержкой структурированного логирования (structured logging). Представьте себе современный офис, где все сотрудники пишут заметки в разные блокноты: один — в бумажный, другой — в электронный, третий — на доску. Руководителю всё равно, куда именно они пишут, — ему важен сам факт, что запись появилась и её можно прочитать. `ILogger` — это «единый язык» записи, а провайдеры — это разные «блокноты»: консоль, отладочное окно, файл, Seq, Elasticsearch и т. д.

**`ILogger<T>`** — основной интерфейс, который вы получаете через DI. Категория категории (category) берётся из типа `T` (обычно это класс, в который вы инжектите логгер). Это помогает фильтровать логи по источнику. Метод расширения `LogInformation`, `LogWarning`, `LogError` и другие — это удобные обёртки над `Log(logLevel, eventId, state, exception, formatter)`.

**Уровни логирования (log levels)** упорядочены по важности: `Trace` и `Debug` — детали для разработчика, `Information` — ключевые события жизненного цикла, `Warning` — что-то необычное, но не ошибка, `Error` — сбой, который был обработан, `Critical` — катастрофа, приложение не может работать. Уровень можно настраивать глобально и по категории в `appsettings.json`:

```json
"Logging": { "LogLevel": { "Default": "Information", "Microsoft.AspNetCore": "Warning" } }
```

**Провайдеры** решают, куда физически отправляется сообщение. Из коробки доступны `Console`, `Debug`, `EventSource`, `EventLog` (Windows). Сторонние пакеты добавляют `Serilog`, `NLog`, `log4net`. Популярный в dev-окружении **Seq** (от Datalust) позволяет искать логи через SQL-подобные запросы и строить графики. Провайдеры добавляются через `builder.Logging.AddConsole()` и аналогичные методы, а `ILoggerProvider` — точка расширения, если вы пишете свой приёмник.

**Structured logging** — главное отличие современного подхода от «склейки строк». Вместо `$"User {userId} bought {count} items"` вы пишете `logger.LogInformation("User {UserId} bought {Count} items", userId, count)`. Поставщик получает не плоскую строку, а именованные свойства `UserId` и `Count`. В Seq, Elasticsearch или Application Insights вы сможете фильтровать по `UserId == 42`, не разбирая регулярками текст. Аналогия: вместо фотографий чеков, где цифры размыты, вы получаете электронную таблицу с колонками. Несколько правил: имена свойств — PascalCase, не используйте интерполяцию строк для шаблона (она разрушает структуру), чувствительные данные (пароли, токены) никогда не логируйте.

**Scopes** позволяют группировать логи одной логической операции. Например, обработка HTTP-запроса оборачивается в scope с `RequestId` — все логи внутри получат этот идентификатор. Включаются через `using (_logger.BeginScope("Processing order {OrderId}", orderId))`. Провайдеры, поддерживающие scopes (например, Seq, Console с настройками), показывают группировку визуально.

**Serilog** — популярная альтернатива/дополнение к встроенному логированию. Он известен «синками» (sinks) в десятки destinations, богатым форматированием и стабильным structured logging. Часто используется как провайдер поверх `ILogger` через `Serilog.AspNetCore`, чтобы приложение работало со стандартным `ILogger<T>`, а под капотом — Serilog с sinks в Console, File, Seq, Elasticsearch. Обзорно: выбираем Serilog, когда нужен файловый sink, сложная фильтрация или интеграция с экосистемой, выходящая за рамки встроенного логирования.

Важно понимать разницу между логированием и телеметрией: логи — дискретные события для диагностики, метрики — агрегаты (rate, latency p99), трейсы — распределённые вызовы. В этом уроке мы сфокусированы на логах, но в реальных системах они работают вместе (OpenTelemetry, Application Insights).

#### Theory (EN)

Logging in ASP.NET Core is not a glorified `Console.WriteLine` — it is a real infrastructure built around a single abstraction, `ILogger`, a pluggable set of providers, and first-class support for structured logging. Picture a modern office where every employee writes notes into different notebooks: one paper, one digital, one on a shared whiteboard. The manager does not care which notebook was used — what matters is that the note exists and can be read later. `ILogger` is the common language for writing, and providers are the different notebooks: console, debug window, file, Seq, Elasticsearch, and so on.

**`ILogger<T>`** is the main interface you consume through dependency injection. The category is derived from the type `T` — typically the class into which you inject the logger — and it lets you filter logs by source. The `LogInformation`, `LogWarning`, `LogError` extension methods are convenient wrappers over the lower-level `Log(logLevel, eventId, state, exception, formatter)` call.

**Log levels** are ordered by severity: `Trace` and `Debug` are developer-only details, `Information` marks lifecycle milestones, `Warning` flags something unusual but not a failure, `Error` is a handled failure, `Critical` means the application cannot keep running. Levels can be configured globally and per category in `appsettings.json`:

```json
"Logging": { "LogLevel": { "Default": "Information", "Microsoft.AspNetCore": "Warning" } }
```

**Providers** decide where a message physically lands. Built-in providers include `Console`, `Debug`, `EventSource`, and `EventLog` on Windows. Third-party packages add `Serilog`, `NLog`, and `log4net`. A common dev-time destination is **Seq** (from Datalust): it lets you query logs with SQL-like expressions and chart trends. Providers are registered through `builder.Logging.AddConsole()` and similar calls, and `ILoggerProvider` is the extension point if you write your own sink.

**Structured logging** is the key difference from old-school string concatenation. Instead of `$"User {userId} bought {count} items"` you write `logger.LogInformation("User {UserId} bought {Count} items", userId, count)`. The provider receives not a flat string but named properties `UserId` and `Count`. In Seq, Elasticsearch, or Application Insights you can then filter by `UserId == 42` instead of regex-parsing text. Analogy: instead of blurry photos of receipts you get a spreadsheet with real columns. A few rules: property names are PascalCase, never use string interpolation for the template (it destroys structure), and never log secrets such as passwords or tokens.

**Scopes** group logs of a single logical operation. For example, an HTTP request is wrapped in a scope carrying a `RequestId`, and every log emitted inside inherits it. You start one with `using (_logger.BeginScope("Processing order {OrderId}", orderId))`. Scope-aware providers (Seq, Console with the right options) render the grouping visually.

**Serilog** is a popular alternative or addition to the built-in logging. It is known for dozens of sinks, rich formatting, and rock-solid structured logging. It is most often plugged in as a provider on top of `ILogger` through `Serilog.AspNetCore`, so the app keeps using the standard `ILogger<T>` while Serilog fans out to Console, File, Seq, or Elasticsearch underneath. In short: reach for Serilog when you need a file sink, sophisticated filtering, or integrations beyond what the built-in logger offers.

Finally, distinguish logging from broader telemetry: logs are discrete diagnostic events, metrics are aggregates (rate, p99 latency), traces are distributed call graphs. This lesson focuses on logs, but in production they run together with metrics and traces under tools like OpenTelemetry or Application Insights.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8. Полный, рабочий пример: IConfiguration + ILogger + scopes + structured logging.
// Full, working example: IConfiguration + ILogger + scopes + structured logging.

using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Logging;

// Простой сервис, который что-то делает и пишет логи / Simple service that does work and logs.
public sealed class OrderProcessor(ILogger<OrderProcessor> logger)
{
    // Структурированный шаблон: имена свойств в PascalCase / Structured template: PascalCase property names.
    public async Task<decimal> ProcessAsync(int orderId, int userId, CancellationToken ct = default)
    {
        // Шаблон НЕ интерполируем — иначе свойства потеряются / Do NOT interpolate the template.
        logger.LogInformation("Начинаем обработку заказа {OrderId} для пользователя {UserId} / " +
                              "Starting order {OrderId} for user {UserId}", orderId, userId);

        // Scope группирует все логи одной операции / A scope groups all logs of one operation.
        using (logger.BeginScope("Order {OrderId}, User {UserId}", orderId, userId))
        {
            try
            {
                var total = await ComputeTotalAsync(orderId, ct);

                // Предупреждение, а не ошибка — бизнес-кейс / Warning, not error — business case.
                if (total > 10_000m)
                {
                    logger.LogWarning("Большой заказ {OrderId} сумма {Total} / " +
                                      "Large order {OrderId} total {Total}", orderId, total);
                }

                logger.LogInformation("Заказ {OrderId} обработан, сумма {Total} / " +
                                      "Order {OrderId} completed, total {Total}", orderId, total);
                return total;
            }
            catch (OperationCanceledException) when (ct.IsCancellationRequested)
            {
                // Отмена — штатная ситуация, пишем Information / Cancellation is expected, log as Information.
                logger.LogInformation("Заказ {OrderId} отменён клиентом / Order {OrderId} cancelled", orderId);
                throw;
            }
            catch (Exception ex)
            {
                // Полная ошибка с exception / Full error with exception object.
                logger.LogError(ex, "Сбой обработки заказа {OrderId} / Failed to process order {OrderId}", orderId);
                throw;
            }
        }
    }

    private static async Task<decimal> ComputeTotalAsync(int orderId, CancellationToken ct)
    {
        await Task.Delay(50, ct); // имитация I/O / simulated I/O
        return orderId * 12.5m;
    }
}

// Минимальный API с DI и конфигурацией уровней / Minimal API with DI and level configuration.
public static class OrderEndpoints
{
    public static void MapOrders(this WebApplication app)
    {
        app.MapPost("/orders/{orderId:int}", async (
            int orderId,
            [FromQuery] int userId,
            OrderProcessor processor,
            CancellationToken ct) =>
        {
            var total = await processor.ProcessAsync(orderId, userId, ct);
            return Results.Ok(new { orderId, userId, total });
        });
    }
}

// Точка входа: регистрация сервисов и провайдеров / Entry point: register services and providers.
public class Program
{
    public static void Main(string[] args)
    {
        var builder = WebApplication.CreateBuilder(args);

        // Встроенные провайдеры / Built-in providers.
        builder.Logging.AddConsole();
        builder.Logging.AddDebug();

        // (Опционально) Seq как sink для локальной разработки / Optional Seq sink for local dev.
        // NuGet: Serilog.AspNetCore + Serilog.Sinks.Seq
        // NuGet: Serilog.AspNetCore + Serilog.Sinks.Seq
        // builder.Host.UseSerilog((ctx, lc) => lc
        //     .ReadFrom.Configuration(ctx.Configuration)
        //     .WriteTo.Console()
        //     .WriteTo.Seq("http://localhost:5341"));

        // Уровень по категории можно переопределить в коде / Override level per category in code.
        builder.Logging.AddFilter<OrderProcessor>(LogLevel.Debug);

        builder.Services.AddSingleton<OrderProcessor>();

        var app = builder.Build();
        app.MapOrders();
        app.Run();
    }
}
```

Пример `appsettings.json` для управления уровнями / Example `appsettings.json` for level control:

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "OrderProcessor": "Debug"
    }
  },
  "Serilog": {
    "MinimumLevel": { "Default": "Information" },
    "WriteTo": [
      { "Name": "Console" },
      { "Name": "Seq", "Args": { "serverUrl": "http://localhost:5341" } }
    ]
  }
}
```

#### Best Practices
- Используйте структурированные шаблоны (`{PropertyName}`) и никогда не интерполируйте строку-шаблон; давайте свойствам осмысленные имена в PascalCase.
- Логируйте на подходящем уровне: детали — `Debug`/`Trace`, ключевые события — `Information`, аномалии — `Warning`, обработанные сбои — `Error`, фатальные — `Critical`.
- Инжектируйте `ILogger<T>`, где `T` — класс-владелец; это даёт категорию и позволяет точечно настраивать уровни.
- Оборачивайте длительные операции в `BeginScope`, чтобы связывать логи одним `RequestId`/`OrderId`/`CorrelationId`.
- Никогда не пишите в лог пароли, токены, номера карт и другие PII — используйте redaction или отдельные безопасные sink-и.
- Centralize logging configuration in `appsettings.json` per environment; never hard-code provider URLs or secrets in code.
- Prefer named structured placeholders (`{PropertyName}`) over string interpolation for the template, and use PascalCase, meaningful names.
- Match level to intent: `Debug`/`Trace` for detail, `Information` for milestones, `Warning` for anomalies, `Error` for handled failures, `Critical` for fatal ones.
- Inject `ILogger<T>` where `T` is the owning class so the category is meaningful and per-category filters work.
- Wrap long operations in `BeginScope` to tie logs together with a `RequestId`/`OrderId`/`CorrelationId`.
- Never log passwords, tokens, card numbers, or other PII — use redaction or dedicated secure sinks.
- Keep logging configuration in `appsettings.json` per environment; never hard-code provider URLs or secrets in code.

#### Частые ошибки / Common Mistakes
- Использование интерполяции `$"...{x}"` в шаблоне → теряются свойства и фильтрация в Seq/Elasticsearch; используйте именованные плейсхолдеры `{X}`.
- Логирование одного и того же на нескольких уровнях подряд (`LogInformation` + `LogDebug` с тем же текстом) → шум; выберите один уровень.
- Перехват исключения и `logger.LogError("Ошибка: " + ex)` без передачи `ex` → теряется stack trace; передавайте `ex` первым аргументом.
- Логирование чувствительных данных (пароли, токены) в `Information` → утечка; не пишите их вообще или redact.
- Создание логгера через `new` вместо DI → теряются провайдеры и конфигурация; всегда инжектируйте `ILogger<T>`.
- Using string interpolation `$"...{x}"` in the template → properties are lost and Seq/Elasticsearch filtering breaks; use named placeholders `{X}`.
- Logging the same message at multiple levels (`LogInformation` + `LogDebug` with the same text) → noise; pick one level.
- Catching an exception and calling `logger.LogError("Error: " + ex)` without passing `ex` → stack trace is lost; pass `ex` as the first argument.
- Logging sensitive data (passwords, tokens) at `Information` → leakage; do not log them at all or redact.
- Creating the logger with `new` instead of DI → providers and configuration are lost; always inject `ILogger<T>`.

#### Чек-лист самопроверки / Self-check Checklist
- [ ] Я использую `ILogger<T>` через DI, а не `Console.WriteLine` или `new Logger()`.
- [ ] Все шаблоны сообщений — структурированные, с именованными плейсхолдерами в PascalCase, без интерполяции.
- [ ] Уровни логов настроены в `appsettings.json` отдельно для `Default` и ключевых категорий.
- [ ] Длительные операции обёрнуты в `BeginScope` с correlation-идентификатором.
- [ ] При логировании ошибок я передаю объект `Exception` первым аргументом.
- [ ] В логах нет паролей, токенов и PII; проверил содержимое сообщений.
- [ ] Я знаю, как добавить провайдер (`AddConsole`, `AddDebug`, Serilog sink) и чем отличается `ILoggerProvider` от `ILogger`.
- [ ] I inject `ILogger<T>` through DI instead of `Console.WriteLine` or `new Logger()`.
- [ ] All message templates are structured, with named PascalCase placeholders and no string interpolation.
- [ ] Log levels are configured in `appsettings.json` separately for `Default` and key categories.
- [ ] Long operations are wrapped in `BeginScope` with a correlation identifier.
- [ ] When logging errors, I pass the `Exception` object as the first argument.
- [ ] No passwords, tokens, or PII appear in logs; I reviewed the message contents.
- [ ] I know how to add a provider (`AddConsole`, `AddDebug`, a Serilog sink) and the difference between `ILoggerProvider` and `ILogger`.

#### Ресурсы / Resources
- [Microsoft Learn — Logging in ASP.NET Core — https://learn.microsoft.com/aspnet/core/fundamentals/logging/](https://learn.microsoft.com/aspnet/core/fundamentals/logging/)
- [Microsoft Learn — Structured logging — https://learn.microsoft.com/dotnet/core/extensions/logging](https://learn.microsoft.com/dotnet/core/extensions/logging)
- [Serilog homepage — https://serilog.net/](https://serilog.net/)
- [Seq (Datalust) — https://datalust.co/seq](https://datalust.co/seq)

---

[⬆ К модулю M13](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
