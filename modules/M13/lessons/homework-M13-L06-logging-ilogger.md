---
[← К уроку M13-L06](lesson-M13-L06-logging-ilogger.md) | [⬆ К модулю M13](../README.md) | [Следующее ДЗ →](homework-M13-L07-static-files-wwwroot.md)
---

### Домашнее задание M13-L06: Logging, ILogger, провайдеры, structured logging / Homework M13-L06: Logging, ILogger, providers, structured logging

**Урок / Lesson:** M13-L06
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) Научиться строить полноценную инфраструктуру логирования в ASP.NET Core на .NET 8: инжектить `ILogger<T>` через DI, писать структурированные шаблоны с именованными свойствами в PascalCase, управлять уровнями через `appsettings.json`, группировать логи одной операции через `BeginScope`, корректно логировать исключения, добавлять провайдеры (`Console`, `Debug`) и понимать, когда подключать Serilog/Seq. (EN) Learn to build a real logging infrastructure in ASP.NET Core on .NET 8: inject `ILogger<T>` through DI, write structured templates with named PascalCase properties, control levels via `appsettings.json`, group logs of a single operation with `BeginScope`, log exceptions correctly, add providers (`Console`, `Debug`), and understand when to reach for Serilog/Seq.

#### Связь с уроком / Connection to the lesson
(RU) Урок вводит абстракцию `ILogger`, иерархию уровней, провайдеры и структурированное логирование. В этом задании вы построите мини-сервис обработки платежей, в котором каждое правило урока применяется на практике: шаблоны без интерполяции, scope с `CorrelationId`, передача `Exception` первым аргументом, пер-категорийная настройка уровней. Цель — закрепить muscle memory, чтобы в реальных проектах вы не склеивали строки и не теряли stack trace. (EN) The lesson introduces the `ILogger` abstraction, the level hierarchy, providers, and structured logging. In this task you will build a small payment-processing service where every rule from the lesson is applied hands-on: templates without interpolation, a scope carrying `CorrelationId`, passing `Exception` as the first argument, and per-category level configuration. The goal is to build muscle memory so that in real projects you never concatenate strings or lose stack traces.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Представьте, что вы присоединились к команде платёжного шлюза. Сервис обрабатывает входящие запросы на списание средств: проверяет лимиты, обращается к внешнему банковскому адаптеру, записывает результат. В первый же день инцидент: часть транзакций «теряется», support получает жалобы, а у дежурного инженера в консоли — месиво из `Console.WriteLine` без идентификаторов, без контекста операции, без связи между шагами. Воспроизвести проблему локально не получается, потому что логи не структурированы и по ним нельзя отфильтровать «покажи мне всё по `TransactionId == 77123`».

Ваша задача — заменить хаотичное логирование на инфраструктуру, построенную по правилам урока M13-L06. Вы будете использовать `ILogger<T>` через DI, именованные свойства в PascalCase, scopes для группировки логов одной транзакции, корректную передачу исключений и настройку уровней через `appsettings.json` отдельно для `Default` и для категории `PaymentService`. Дополнительно вы добавите провайдеры `Console` и `Debug`, а в бонусе — подключите Serilog с sink в Seq, чтобы убедиться, что структурированные свойства действительно попадают в приёмник и по ним можно строить запросы. В результате дежурный инженер должен по одному `TransactionId` видеть всю историю операции от входа до результата, включая отмену по таймауту или ошибку адаптера, без повторов одного сообщения на разных уровнях и без утечки чувствительных данных (номер карты, CVV).

#### Что нужно сделать (пошагово)
1. Создайте новый проект: `dotnet new web -n PaymentGateway -o PaymentGateway` и перейдите в него: `cd PaymentGateway`. Убедитесь, что целевой фреймворк — `net8.0` (`dotnet --version` должен показать SDK 8.x). Откройте `PaymentGateway.csproj` и при необходимости добавьте `<Nullable>enable</Nullable>` и `<ImplicitUsings>enable</ImplicitUsings>`.
2. Добавьте пакеты (только то, что нужно для основной части): `dotnet add package Microsoft.Extensions.Logging.Console` и `dotnet add package Microsoft.Extensions.Logging.Debug`. Для бонуса позже понадобятся `Serilog.AspNetCore` и `Serilog.Sinks.Seq`, но ставьте их только когда дойдёте до раздела «Углубление».
3. Создайте файл `PaymentService.cs` с классом `PaymentService`, который инжектит `ILogger<PaymentService>` через первичный конструктор (C# 12). Метод `ChargeAsync(int transactionId, string userId, decimal amount, CancellationToken ct)` должен: писать старт на `Information` со свойствами `{TransactionId}`, `{UserId}`, `{Amount}`; открывать scope `using (_logger.BeginScope("Transaction {TransactionId} by {UserId}", transactionId, userId))`; имитировать вызов адаптера через `await Task.Delay(80, ct)`; если `amount > 50_000m`, писать `Warning` с `{Amount}`; при `OperationCanceledException` с `ct.IsCancellationRequested` писать `Information` и перебрасывать; в общем `catch (Exception ex)` писать `LogError(ex, ...)` с передачей `ex` первым аргументом и перебрасывать.
4. В `Program.cs` (top-level statements) зарегистрируйте `PaymentService` как `AddScoped`, добавьте провайдеры `builder.Logging.AddConsole()` и `builder.Logging.AddDebug()`, а также точечный фильтр `builder.Logging.AddFilter<PaymentService>(LogLevel.Debug)`. Сопоставьте Minimal API `POST /payments/{transactionId:int}` с query-параметром `userId` и телом `amount`, который вызывает `ChargeAsync` и возвращает `Results.Ok(new { transactionId, userId, amount, status })`.
5. В `appsettings.json` настройте секцию `Logging` так: `Default` = `Information`, `Microsoft.AspNetCore` = `Warning`, `PaymentService` = `Debug`. Убедитесь, что после запуска `dotnet run` в консоли видны сообщения от `PaymentService` на уровне `Debug`, а спам от framework-категорий подавлен.
6. Прогоните сценарии через `curl` (или REST-клиент): `curl -X POST "http://localhost:5000/payments/77123?userId=u42" -H "Content-Type: application/json" -d "{\"amount\": 1234.5}"` — ожидаемый ответ `{"transactionId":77123,"userId":"u42","amount":1234.5,"status":"ok"}`. Затем повторите с `amount: 75000` — в логе должна появиться строка уровня `Warning` с `{Amount}`. Затем отправьте запрос и прервите его (Ctrl+C или таймаут на стороне клиента) — должна появиться строка `Information` про отмену. Наконец, временно бросьте `throw new InvalidOperationException("adapter down")` внутри `ChargeAsync` и убедитесь, что в логе уровня `Error` виден полный stack trace, а не просто текст `"Ошибка: ..."`.

#### Требования к решению
- Используется только `ILogger<T>` через DI; `Console.WriteLine` и `new ...Logger()` запрещены. Категория логгера должна быть осмысленной (`PaymentService`), чтобы пер-категорийный фильтр работал.
- Все шаблоны сообщений — структурированные, с именованными плейсхолдерами в PascalCase (`{TransactionId}`, `{UserId}`, `{Amount}`). Строковая интерполяция `$"...{x}"` в шаблоне недопустима: она разрушает свойства и фильтрацию в Seq/Elasticsearch.
- Каждая длительная операция обёрнута в `BeginScope`, который связывает логи одной транзакции через `{TransactionId}` и `{UserId}`. Scope Dispose корректно закрывается через `using`.
- Исключения логируются через перегрузку `LogError(Exception ex, string template, params object[] args)` — объект исключения передаётся первым аргументом, чтобы сохранить stack trace. Шаблон при этом остаётся структурированным.
- Уровни подобраны по смыслу: старт/успех — `Information`, крупная сумма — `Warning`, обработанный сбой — `Error`, отмена по токену — `Information` (штатная ситуация), детали отладки — `Debug`. Дублирование одного сообщения на нескольких уровнях запрещено.
- Чувствительные данные (номер карты, CVV, токены, пароли) не логируются вообще. Если в запросе есть карта, в лог попадает только маска или последние 4 цифры через отдельное свойство `{CardLast4}`.
- Конфигурация уровней и провайдеров — в `appsettings.json` и/или в `Program.cs`, но не хардкод секретов и URL sink-ов в коде (для Seq URL берётся из конфигурации).
- Код компилируется под C# 12 / .NET 8, использует primary constructors, top-level statements и (где уместно) collection expressions и pattern matching.

#### Тонкости и подводные камни
- Главная ловушка — привычка написать `logger.LogInformation($"Начали {transactionId}")`. Интерполяция вычисляется в строку до того, как провайдер увидит шаблон, поэтому Seq получит плоский текст без свойства `TransactionId`. Правильно: `logger.LogInformation("Начали {TransactionId}", transactionId)`. Проверить себя можно так: если в Seq по фильтру `TransactionId == 77123` ничего не находится — вы интерполировали.
- Вторая ловушка — `logger.LogError("Ошибка: " + ex)` или `logger.LogError(ex.ToString())`. Без передачи `ex` как объекта теряется stack trace в структурированном виде, а some провайдеры не смогут извлечь `ExceptionDispatchInfo`. Всегда передавайте `ex` первым аргументом перегрузки `LogError(Exception, string, params object[])`.
- Третья — пер-категорийный фильтр не сработает, если вы создали логгер через `ILoggerFactory.CreateLogger("foo")` с произвольной строкой категории, не совпадающей с именем класса. Инжектируйте `ILogger<PaymentService>`, тогда категория — `PaymentService`, и `AddFilter<PaymentService>(LogLevel.Debug)` применяется именно к ней.
- `BeginScope` возвращает `IDisposable`, который нужно Dispose. Используйте `using` или `using var`. Если забудете — scope «залипнет» и последующие логи чужих операций получат чужой `TransactionId`.
- Отмена по `CancellationToken` — это `OperationCanceledException`. Не логируйте её как `Error`: клиент сам отменил запрос, это штатный сценарий. Используйте фильтр `when (ct.IsCancellationRequested)`, чтобы отличить реальный таймаут адаптера от отмены пользователем.
- `AddDebug` пишет в окно Output Visual Studio / VS Code Debug Console; в `dotnet run` без отладчика вы его не увидите. Не считайте это «не работает» — просто проверяйте Console-провайдер.
- Уровень `Debug` по умолчанию suppressed в Release-сборке, если вы явно не включите его в `appsettings.json`. Если ваши `LogDebug` не видны — проверьте `LogLevel.Default` и категорию.

#### Критерии приёмки
- [ ] Проект `PaymentGateway` собирается: `dotnet build` без ошибок и warning-ов, связанных с логированием.
- [ ] `ILogger<PaymentService>` инжектируется через DI (primary constructor), `Console.WriteLine` отсутствует.
- [ ] Все шаблоны — структурированные, PascalCase, без интерполяции; проверено grep-ом по файлу (нет `$"...{` рядом с `Log`).
- [ ] `ChargeAsync` открывает `BeginScope` с `{TransactionId}` и `{UserId}` и закрывает его через `using`.
- [ ] Старт и успешное завершение — `Information`, крупная сумма — `Warning`, сбой адаптера — `Error` с `ex`.
- [ ] Отмена по `CancellationToken` логируется как `Information`, а не `Error`, и перебрасывается `throw`.
- [ ] `appsettings.json` содержит секцию `Logging` с `Default`, `Microsoft.AspNetCore` и `PaymentService`.
- [ ] В `Program.cs` вызваны `AddConsole()` и `AddDebug()`, а также `AddFilter<PaymentService>(LogLevel.Debug)`.
- [ ] Minimal API `POST /payments/{transactionId:int}` работает: `curl` возвращает JSON со статусом.
- [ ] При `amount > 50000` в логе есть `Warning` со свойством `{Amount}`.
- [ ] При искусственном `throw new InvalidOperationException` в логе `Error` виден полный stack trace.
- [ ] В логах нет номеров карт, CVV, токенов; чувствительные поля маскируются или отсутствуют.
- [ ] Нет дублирования одного сообщения на нескольких уровнях подряд.
- [ ] (Бонус) Serilog + Seq: свойства видны в Seq, фильтр `TransactionId == 77123` работает.
- [ ] Код использует C# 12 / .NET 8 (primary constructors, top-level statements).

#### Подсказки (без прямого ответа)
- Вспомните правило из урока: «шаблон НЕ интерполируем — иначе свойства потеряются». Если хочется `$`, значит, вы пишете не шаблон, а конкатенацию.
- Для scope подойдёт `using (_logger.BeginScope(...))` — это `IDisposable`, он сам закроется на выходе из блока.
- Чтобы передать исключение, используйте перегрузку с `Exception` первым параметром, а шаблон и аргументы — дальше.
- `AddFilter<TCategoryName>(LogLevel)` — типобезопасный способ включить `Debug` только для одного сервиса.
- Для маскирования карты заведите отдельное свойство `{CardLast4}` и пишите только последние 4 цифры; полное число в лог не попадает.
- Запустите Seq локально (`docker run --rm -p 5341:80 datalust/seq`) только если делаете бонус; URL берите из конфигурации.

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8. PaymentGateway: ILogger + structured logging + scopes.
// C# 12 / .NET 8. PaymentGateway: ILogger + structured logging + scopes.

using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Logging;

// Сервис списания средств / Charge service.
// Primary constructor инжектит ILogger<PaymentService> через DI.
// Primary constructor injects ILogger<PaymentService> through DI.
public sealed class PaymentService(ILogger<PaymentService> logger)
{
    public async Task<ChargeResult> ChargeAsync(
        int transactionId, string userId, decimal amount,
        string? cardLast4, CancellationToken ct = default)
    {
        // Шаблон без интерполяции: имена свойств в PascalCase.
        // Template without interpolation: PascalCase property names.
        logger.LogInformation(
            "Начинаем списание {TransactionId} для {UserId} на сумму {Amount} / " +
            "Starting charge {TransactionId} for {UserId} amount {Amount}",
            transactionId, userId, amount);

        // Scope связывает все логи одной операции correlation-идентификатором.
        // A scope ties all logs of one operation with a correlation id.
        using (logger.BeginScope("Transaction {TransactionId} by {UserId}", transactionId, userId))
        {
            try
            {
                // Имитация вызова банковского адаптера / Simulated adapter call.
                await Task.Delay(80, ct);

                // Warning для крупной суммы — это аномалия, но не ошибка.
                // Warning for a large amount — anomaly, not a failure.
                if (amount > 50_000m)
                {
                    logger.LogWarning(
                        "Крупная транзакция {TransactionId} сумма {Amount} / " +
                        "Large transaction {TransactionId} amount {Amount}",
                        transactionId, amount);
                }

                logger.LogInformation(
                    "Транзакция {TransactionId} завершена, карта *{CardLast4} / " +
                    "Transaction {TransactionId} completed, card *{CardLast4}",
                    transactionId, cardLast4 ?? "0000");

                return new ChargeResult(transactionId, userId, amount, "ok");
            }
            catch (OperationCanceledException) when (ct.IsCancellationRequested)
            {
                // Отмена — штатно, не ошибка / Cancellation is expected, not an error.
                logger.LogInformation(
                    "Транзакция {TransactionId} отменена клиентом / " +
                    "Transaction {TransactionId} cancelled by client",
                    transactionId);
                throw;
            }
            catch (Exception ex)
            {
                // Исключение передаётся первым аргументом — сохраняется stack trace.
                // Exception passed as the first argument — stack trace is preserved.
                logger.LogError(ex,
                    "Сбой списания {TransactionId} для {UserId} / " +
                    "Charge failed for {TransactionId} by {UserId}",
                    transactionId, userId);
                throw;
            }
        }
    }
}

public sealed record ChargeResult(int TransactionId, string UserId, decimal Amount, string Status);

// Minimal API с DI и настройкой провайдеров / Minimal API with DI and provider setup.
// Program.cs (top-level statements):
// var builder = WebApplication.CreateBuilder(args);
// builder.Logging.AddConsole();
// builder.Logging.AddDebug();
// builder.Logging.AddFilter<PaymentService>(LogLevel.Debug);
// builder.Services.AddScoped<PaymentService>();
// var app = builder.Build();
// app.MapPost("/payments/{transactionId:int}", async (int transactionId, [FromQuery] string userId,
//     PaymentService svc, CancellationToken ct, [FromBody] ChargeRequest req) =>
// {
//     var res = await svc.ChargeAsync(transactionId, userId, req.Amount, req.CardLast4, ct);
//     return Results.Ok(res);
// });
// app.Run();
// public sealed record ChargeRequest(decimal Amount, string? CardLast4);
```

Разбор по строкам. Класс `PaymentService` использует primary constructor C# 12 — параметр `ILogger<PaymentService> logger` автоматически становится инжектируемым полем, категория логгера равна `PaymentService`, что совпадает с фильтром `AddFilter<PaymentService>(LogLevel.Debug)`. Первое `LogInformation` содержит шаблон с тремя именованными свойствами в PascalCase (`{TransactionId}`, `{UserId}`, `{Amount}`) — без `$`, без интерполяции; именно так провайдер получает структурированные поля, по которым в Seq можно отфильтровать `TransactionId == 77123`. Блок `using (logger.BeginScope(...))` открывает scope: все логи внутри, включая те, что будут выброшены из вложенных вызовов, получат контекст операции, и scope автоматически закроется при выходе. Имитация `Task.Delay(80, ct)` — это точка, где реально был бы вызов адаптера; токен отмены пробрасывается, поэтому отмена клиента немедленно поднимает `OperationCanceledException`. Ветка `when (ct.IsCancellationRequested)` ловит именно отмену и логирует её как `Information` — потому что это штатный сценарий, а не сбой; `throw` перебрасывает исключение, чтобы клиент тоже узнал. Общий `catch (Exception ex)` передаёт `ex` первым аргументом перегрузки `LogError(Exception, string, params object[])` — это ключевое правило урока: так сохраняется stack trace и провайдер может извлечь структурированную информацию об исключении. Шаблон при этом остаётся структурированным (`{TransactionId}`, `{UserId}`), не склеивается с `ex.ToString()`. Маска карты передаётся как `CardLast4` — полное число никогда не попадает в лог, что закрывает требование по PII. В `Program.cs` (показан как комментарий для компактности) регистрируются провайдеры `Console` и `Debug`, точечный фильтр для `PaymentService`, сам сервис как `Scoped`, и Minimal API, который пробрасывает `CancellationToken` из инфраструктуры ASP.NET Core — поэтому таймаут клиента корректно отражается как `OperationCanceledException`. Концепции урока, применённые здесь: DI для `ILogger<T>`, структурированные шаблоны, `BeginScope`, корректная передача исключения, пер-категорийная настройка уровней, разделение `Information`/`Warning`/`Error`/`Debug`, маскирование PII.

#### Задания на углубление (бонус)
1. Подключите Serilog через `Serilog.AspNetCore`: `builder.Host.UseSerilog((ctx, lc) => lc.ReadFrom.Configuration(ctx.Configuration).WriteTo.Console().WriteTo.Seq("http://localhost:5341"))`. Запустите Seq в Docker, отправьте несколько запросов и убедитесь, что в Seq видны свойства `TransactionId`, `UserId`, `Amount` и по ним работает фильтр.
2. Реализуйте собственный `ILoggerProvider`, который пишет логи уровня `Error` и выше в отдельный файл `errors.jsonl` (по одной JSON-строке на запись, с полями `timestamp`, `level`, `category`, `message`, `TransactionId`). Зарегистрируйте его через `builder.Logging.AddProvider(new JsonErrorLoggerProvider())`.
3. Добавьте middleware, который генерирует `CorrelationId` (GUID) для каждого запроса и кладёт его в scope через `BeginScope("Correlation {CorrelationId}", correlationId)`, чтобы все логи одного HTTP-запроса — включая логи из `PaymentService` — были связаны единым идентификатором.
4. Настройте разные `appsettings.Development.json` и `appsettings.Production.json`: в dev — `Debug` для `PaymentService` и sink в локальный Seq, в prod — `Information` и подавление `Microsoft.*` на `Warning`. Переключайте через переменную `ASPNETCORE_ENVIRONMENT` и проверяйте, что уровни меняются.

---

## Statement in English / Постановка на английском

#### Context & motivation
Imagine you just joined a payment-gateway team. The service handles incoming charge requests: it checks limits, calls an external banking adapter, records the result. On your very first day an incident hits: some transactions "disappear", support gets complaints, and the on-call engineer stares at a console full of `Console.WriteLine` garbage — no identifiers, no operation context, no link between steps. The problem cannot be reproduced locally, because the logs are not structured and you cannot filter them with "show me everything for `TransactionId == 77123`".

Your job is to replace the chaotic logging with a proper infrastructure built along the rules of lesson M13-L06. You will use `ILogger<T>` through DI, named PascalCase properties, scopes to group the logs of a single transaction, correct exception passing, and level configuration through `appsettings.json` separately for `Default` and for the `PaymentService` category. You will additionally add the `Console` and `Debug` providers, and as a bonus you will plug in Serilog with a Seq sink to confirm that the structured properties really reach the destination and can be queried. By the end, the on-call engineer should be able to take a single `TransactionId` and see the whole story of that operation — from entry to result, including cancellation by timeout or an adapter error — without duplicate messages at multiple levels and without leaking sensitive data (card number, CVV).

#### What to do step by step
1. Create a new project: `dotnet new web -n PaymentGateway -o PaymentGateway` and enter it: `cd PaymentGateway`. Verify the target framework is `net8.0` (`dotnet --version` should show SDK 8.x). Open `PaymentGateway.csproj` and, if needed, add `<Nullable>enable</Nullable>` and `<ImplicitUsings>enable</ImplicitUsings>`.
2. Add the packages required for the core part: `dotnet add package Microsoft.Extensions.Logging.Console` and `dotnet add package Microsoft.Extensions.Logging.Debug`. For the bonus you will later need `Serilog.AspNetCore` and `Serilog.Sinks.Seq`, but install them only when you reach the "Going deeper" section.
3. Create `PaymentService.cs` with a `PaymentService` class that injects `ILogger<PaymentService>` through a primary constructor (C# 12). The method `ChargeAsync(int transactionId, string userId, decimal amount, CancellationToken ct)` must: log the start at `Information` with properties `{TransactionId}`, `{UserId}`, `{Amount}`; open a scope `using (_logger.BeginScope("Transaction {TransactionId} by {UserId}", transactionId, userId))`; simulate the adapter call via `await Task.Delay(80, ct)`; if `amount > 50_000m`, emit a `Warning` with `{Amount}`; on `OperationCanceledException` with `ct.IsCancellationRequested`, log `Information` and rethrow; in a general `catch (Exception ex)`, call `LogError(ex, ...)` passing `ex` as the first argument and rethrow.
4. In `Program.cs` (top-level statements) register `PaymentService` as `AddScoped`, add the providers `builder.Logging.AddConsole()` and `builder.Logging.AddDebug()`, and a targeted filter `builder.Logging.AddFilter<PaymentService>(LogLevel.Debug)`. Map a Minimal API `POST /payments/{transactionId:int}` with a `userId` query parameter and an `amount` body that calls `ChargeAsync` and returns `Results.Ok(new { transactionId, userId, amount, status })`.
5. In `appsettings.json` configure the `Logging` section so that `Default` = `Information`, `Microsoft.AspNetCore` = `Warning`, `PaymentService` = `Debug`. After `dotnet run`, confirm that `PaymentService` `Debug` messages appear in the console while framework-category spam is suppressed.
6. Run the scenarios with `curl` (or a REST client): `curl -X POST "http://localhost:5000/payments/77123?userId=u42" -H "Content-Type: application/json" -d "{\"amount\": 1234.5}"` — expected response `{"transactionId":77123,"userId":"u42","amount":1234.5,"status":"ok"}`. Then repeat with `amount: 75000` — a `Warning` line with `{Amount}` must appear in the log. Then send a request and abort it (Ctrl+C or a client-side timeout) — an `Information` line about cancellation must appear. Finally, temporarily `throw new InvalidOperationException("adapter down")` inside `ChargeAsync` and confirm that the `Error` log line shows the full stack trace rather than a bare `"Error: ..."` text.

#### Requirements
- Only `ILogger<T>` through DI is allowed; `Console.WriteLine` and `new ...Logger()` are forbidden. The logger category must be meaningful (`PaymentService`) so the per-category filter actually applies.
- All message templates are structured, with named PascalCase placeholders (`{TransactionId}`, `{UserId}`, `{Amount}`). String interpolation `$"...{x}"` in the template is unacceptable: it destroys properties and breaks filtering in Seq/Elasticsearch.
- Every long operation is wrapped in `BeginScope` that ties the logs of one transaction through `{TransactionId}` and `{UserId}`. The scope is correctly disposed via `using`.
- Exceptions are logged through the `LogError(Exception ex, string template, params object[] args)` overload — the exception object is passed as the first argument so the stack trace is preserved. The template itself stays structured.
- Levels match intent: start/success — `Information`, large amount — `Warning`, handled failure — `Error`, token cancellation — `Information` (expected), debug detail — `Debug`. Duplicating the same message at multiple levels is forbidden.
- Sensitive data (card number, CVV, tokens, passwords) is never logged. If a card is present in the request, only a mask or the last four digits reach the log via a separate `{CardLast4}` property.
- Level and provider configuration lives in `appsettings.json` and/or `Program.cs`, but no secrets or sink URLs are hard-coded (Seq URL is read from configuration).
- The code compiles under C# 12 / .NET 8 and uses primary constructors, top-level statements, and (where appropriate) collection expressions and pattern matching.

#### Pitfalls
- The main trap is the habit of writing `logger.LogInformation($"Started {transactionId}")`. Interpolation is evaluated into a flat string before the provider ever sees the template, so Seq receives text without a `TransactionId` property. Correct: `logger.LogInformation("Started {TransactionId}", transactionId)`. Self-check: if filtering by `TransactionId == 77123` in Seq returns nothing — you interpolated.
- The second trap is `logger.LogError("Error: " + ex)` or `logger.LogError(ex.ToString())`. Without passing `ex` as an object you lose the structured stack trace, and some providers cannot extract `ExceptionDispatchInfo`. Always pass `ex` as the first argument of the `LogError(Exception, string, params object[])` overload.
- The third trap: a per-category filter will not fire if you created the logger through `ILoggerFactory.CreateLogger("foo")` with an arbitrary category string that does not match the class name. Inject `ILogger<PaymentService>` so the category is `PaymentService`, and `AddFilter<PaymentService>(LogLevel.Debug)` applies precisely to it.
- `BeginScope` returns an `IDisposable` that must be disposed. Use `using` or `using var`. If you forget, the scope "sticks" and logs of unrelated operations inherit a foreign `TransactionId`.
- Cancellation through `CancellationToken` raises `OperationCanceledException`. Do not log it as `Error`: the client cancelled on its own, that is expected. Use the `when (ct.IsCancellationRequested)` filter to distinguish a real adapter timeout from a user cancellation.
- `AddDebug` writes to the Output window of Visual Studio / the VS Code Debug Console; under plain `dotnet run` without a debugger you will not see it. That is not "broken" — just verify against the Console provider.
- The `Debug` level is suppressed by default in Release builds unless you explicitly enable it in `appsettings.json`. If your `LogDebug` calls are invisible, check `LogLevel.Default` and the category name.

#### Acceptance criteria
- [ ] The `PaymentGateway` project builds: `dotnet build` with no errors and no logging-related warnings.
- [ ] `ILogger<PaymentService>` is injected through DI (primary constructor); no `Console.WriteLine` anywhere.
- [ ] All templates are structured, PascalCase, without interpolation; verified by grepping the file (no `$"...{` near a `Log` call).
- [ ] `ChargeAsync` opens a `BeginScope` with `{TransactionId}` and `{UserId}` and disposes it via `using`.
- [ ] Start and success are `Information`, large amount is `Warning`, adapter failure is `Error` with `ex`.
- [ ] Cancellation via `CancellationToken` is logged as `Information`, not `Error`, and rethrown with `throw`.
- [ ] `appsettings.json` contains a `Logging` section with `Default`, `Microsoft.AspNetCore`, and `PaymentService`.
- [ ] `Program.cs` calls `AddConsole()` and `AddDebug()` plus `AddFilter<PaymentService>(LogLevel.Debug)`.
- [ ] The Minimal API `POST /payments/{transactionId:int}` works: `curl` returns JSON with a status.
- [ ] With `amount > 50000` the log contains a `Warning` line carrying the `{Amount}` property.
- [ ] With an artificial `throw new InvalidOperationException` the `Error` log line shows the full stack trace.
- [ ] No card numbers, CVVs, or tokens appear in the logs; sensitive fields are masked or absent.
- [ ] No duplication of the same message at consecutive levels.
- [ ] (Bonus) Serilog + Seq: properties are visible in Seq and the filter `TransactionId == 77123` works.
- [ ] The code uses C# 12 / .NET 8 (primary constructors, top-level statements).

#### Hints (no direct answer)
- Recall the lesson rule: "do NOT interpolate the template — properties are lost". If you feel like using `$`, you are writing concatenation, not a template.
- For a scope, `using (_logger.BeginScope(...))` works — it returns an `IDisposable` that closes the scope on block exit.
- To pass an exception, use the overload whose first parameter is `Exception`; the template and arguments come after.
- `AddFilter<TCategoryName>(LogLevel)` is the type-safe way to enable `Debug` for exactly one service.
- For masking the card, introduce a separate `{CardLast4}` property and log only the last four digits; the full number never reaches the log.
- Run Seq locally (`docker run --rm -p 5341:80 datalust/seq`) only for the bonus; take the URL from configuration.

#### Reference solution walk-through

```csharp
// C# 12 / .NET 8. PaymentGateway: ILogger + structured logging + scopes.
// C# 12 / .NET 8. PaymentGateway: ILogger + structured logging + scopes.

using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Logging;

// Charge service.
// Primary constructor injects ILogger<PaymentService> through DI.
public sealed class PaymentService(ILogger<PaymentService> logger)
{
    public async Task<ChargeResult> ChargeAsync(
        int transactionId, string userId, decimal amount,
        string? cardLast4, CancellationToken ct = default)
    {
        // Template without interpolation: PascalCase property names.
        logger.LogInformation(
            "Starting charge {TransactionId} for {UserId} amount {Amount} / " +
            "Starting charge {TransactionId} for {UserId} amount {Amount}",
            transactionId, userId, amount);

        // A scope ties all logs of one operation with a correlation id.
        using (logger.BeginScope("Transaction {TransactionId} by {UserId}", transactionId, userId))
        {
            try
            {
                // Simulated adapter call.
                await Task.Delay(80, ct);

                // Warning for a large amount — anomaly, not a failure.
                if (amount > 50_000m)
                {
                    logger.LogWarning(
                        "Large transaction {TransactionId} amount {Amount} / " +
                        "Large transaction {TransactionId} amount {Amount}",
                        transactionId, amount);
                }

                logger.LogInformation(
                    "Transaction {TransactionId} completed, card *{CardLast4} / " +
                    "Transaction {TransactionId} completed, card *{CardLast4}",
                    transactionId, cardLast4 ?? "0000");

                return new ChargeResult(transactionId, userId, amount, "ok");
            }
            catch (OperationCanceledException) when (ct.IsCancellationRequested)
            {
                // Cancellation is expected, not an error.
                logger.LogInformation(
                    "Transaction {TransactionId} cancelled by client / " +
                    "Transaction {TransactionId} cancelled by client",
                    transactionId);
                throw;
            }
            catch (Exception ex)
            {
                // Exception passed as the first argument — stack trace is preserved.
                logger.LogError(ex,
                    "Charge failed for {TransactionId} by {UserId} / " +
                    "Charge failed for {TransactionId} by {UserId}",
                    transactionId, userId);
                throw;
            }
        }
    }
}

public sealed record ChargeResult(int TransactionId, string UserId, decimal Amount, string Status);

// Minimal API with DI and provider setup.
// Program.cs (top-level statements):
// var builder = WebApplication.CreateBuilder(args);
// builder.Logging.AddConsole();
// builder.Logging.AddDebug();
// builder.Logging.AddFilter<PaymentService>(LogLevel.Debug);
// builder.Services.AddScoped<PaymentService>();
// var app = builder.Build();
// app.MapPost("/payments/{transactionId:int}", async (int transactionId, [FromQuery] string userId,
//     PaymentService svc, CancellationToken ct, [FromBody] ChargeRequest req) =>
// {
//     var res = await svc.ChargeAsync(transactionId, userId, req.Amount, req.CardLast4, ct);
//     return Results.Ok(res);
// });
// app.Run();
// public sealed record ChargeRequest(decimal Amount, string? CardLast4);
```

Line-by-line walk-through. The `PaymentService` class uses a C# 12 primary constructor — the `ILogger<PaymentService> logger` parameter automatically becomes an injected field, the logger category equals `PaymentService`, which matches the `AddFilter<PaymentService>(LogLevel.Debug)` filter. The first `LogInformation` carries a template with three named PascalCase properties (`{TransactionId}`, `{UserId}`, `{Amount}`) — no `$`, no interpolation; that is exactly how the provider receives structured fields that Seq can later filter with `TransactionId == 77123`. The `using (logger.BeginScope(...))` block opens a scope: every log emitted inside it, including ones thrown from nested calls, inherits the operation context, and the scope auto-closes on exit. The `Task.Delay(80, ct)` simulation stands in for a real adapter call; the cancellation token is forwarded, so a client abort immediately raises `OperationCanceledException`. The `when (ct.IsCancellationRequested)` branch catches precisely a cancellation and logs it as `Information` — because it is an expected scenario, not a failure — and `throw` rethrows so the client learns too. The general `catch (Exception ex)` passes `ex` as the first argument of the `LogError(Exception, string, params object[])` overload — this is the key lesson rule: the stack trace is preserved and the provider can extract structured exception information. The template itself stays structured (`{TransactionId}`, `{UserId}`), never concatenated with `ex.ToString()`. The card mask is passed as `CardLast4` — the full number never reaches the log, satisfying the PII requirement. In `Program.cs` (shown as comments for compactness) the providers `Console` and `Debug` are registered, the targeted filter for `PaymentService` is set, the service itself is registered as `Scoped`, and the Minimal API forwards the `CancellationToken` from the ASP.NET Core infrastructure — so a client timeout correctly surfaces as `OperationCanceledException`. Lesson concepts applied here: DI for `ILogger<T>`, structured templates, `BeginScope`, correct exception passing, per-category level configuration, the `Information`/`Warning`/`Error`/`Debug` split, and PII masking.

#### Going deeper (bonus)
1. Plug in Serilog through `Serilog.AspNetCore`: `builder.Host.UseSerilog((ctx, lc) => lc.ReadFrom.Configuration(ctx.Configuration).WriteTo.Console().WriteTo.Seq("http://localhost:5341"))`. Run Seq in Docker, send a few requests, and confirm that Seq shows the `TransactionId`, `UserId`, and `Amount` properties and that filtering by them works.
2. Implement your own `ILoggerProvider` that writes `Error`-and-above logs to a separate `errors.jsonl` file (one JSON line per entry, with `timestamp`, `level`, `category`, `message`, `TransactionId` fields). Register it via `builder.Logging.AddProvider(new JsonErrorLoggerProvider())`.
3. Add middleware that generates a `CorrelationId` (a GUID) for each request and pushes it into a scope via `BeginScope("Correlation {CorrelationId}", correlationId)`, so every log of a single HTTP request — including logs from `PaymentService` — is tied by one identifier.
4. Set up distinct `appsettings.Development.json` and `appsettings.Production.json`: in dev — `Debug` for `PaymentService` and a sink to local Seq, in prod — `Information` and suppression of `Microsoft.*` at `Warning`. Switch with the `ASPNETCORE_ENVIRONMENT` variable and verify that levels change.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Проект собирается, `dotnet build` без ошибок.
- [ ] (RU) `ILogger<PaymentService>` инжектируется через DI, `Console.WriteLine` нет.
- [ ] (RU) Шаблоны структурированные, PascalCase, без интерполяции.
- [ ] (RU) `BeginScope` с `{TransactionId}`/`{UserId}` и `using`.
- [ ] (RU) Уровни: `Information`/`Warning`/`Error`/`Debug`, отмена — `Information`.
- [ ] (RU) `LogError(ex, ...)` с передачей `ex` первым аргументом.
- [ ] (RU) `appsettings.json` с секцией `Logging`, `PaymentService` = `Debug`.
- [ ] (RU) `AddConsole()`, `AddDebug()`, `AddFilter<PaymentService>(LogLevel.Debug)`.
- [ ] (RU) Minimal API работает, `curl` возвращает JSON.
- [ ] (RU) PII маскируются, дублей сообщений нет.
- [ ] (EN) Project builds, `dotnet build` with no errors.
- [ ] (EN) `ILogger<PaymentService>` injected via DI, no `Console.WriteLine`.
- [ ] (EN) Templates structured, PascalCase, no interpolation.
- [ ] (EN) `BeginScope` with `{TransactionId}`/`{UserId}` and `using`.
- [ ] (EN) Levels: `Information`/`Warning`/`Error`/`Debug`, cancellation as `Information`.
- [ ] (EN) `LogError(ex, ...)` passes `ex` as the first argument.
- [ ] (EN) `appsettings.json` with a `Logging` section, `PaymentService` = `Debug`.
- [ ] (EN) `AddConsole()`, `AddDebug()`, `AddFilter<PaymentService>(LogLevel.Debug)`.
- [ ] (EN) Minimal API works, `curl` returns JSON.
- [ ] (EN) PII masked, no duplicate messages.

#### Ресурсы / Resources
- [Microsoft Learn — Logging in ASP.NET Core — https://learn.microsoft.com/aspnet/core/fundamentals/logging/](https://learn.microsoft.com/aspnet/core/fundamentals/logging/)
- [Microsoft Learn — Structured logging — https://learn.microsoft.com/dotnet/core/extensions/logging](https://learn.microsoft.com/dotnet/core/extensions/logging)
- [Microsoft Learn — Log levels — https://learn.microsoft.com/aspnet/core/fundamentals/logging/#log-level](https://learn.microsoft.com/aspnet/core/fundamentals/logging/)
- [Serilog homepage — https://serilog.net/](https://serilog.net/)
- [Serilog.AspNetCore — https://github.com/serilog/serilog-aspnetcore](https://github.com/serilog/serilog-aspnetcore)
- [Seq (Datalust) — https://datalust.co/seq](https://datalust.co/seq)
- [OpenTelemetry .NET Logging — https://opentelemetry.io/docs/instrumentation/net/logs/](https://opentelemetry.io/docs/instrumentation/net/logs/)
