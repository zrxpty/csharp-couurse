---
[← К уроку M07-L05](lesson-M07-L05-when-filters.md) | [⬆ К модулю M07](../README.md) | [Следующее ДЗ →](homework-M07-L06-inner-exception.md)
---

### Домашнее задание M07-L05: when-фильтры / Homework M07-L05: when filters

**Урок / Lesson:** M07-L05
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться применять фильтры исключений `catch ... when (...)` для фильтрации по состоянию исключения, безопасного логирования без «проглатывания» и отладочных точек останова; понять семантику вычисления фильтра до входа в `catch` и отличия от связки «catch + if + throw». (EN) Learn to apply exception filters `catch ... when (...)` for state-based filtering, safe non-swallowing logging, and debug-only breakpoints; understand that the filter is evaluated before entering `catch` and how this differs from the “catch + if + throw” idiom.

#### Связь с уроком / Connection to the lesson
(RU) Урок вводит конструкцию `when` как декларативный способ фильтрации исключений по условию до входа в обработчик, разбирает три главных сценария (фильтрация по типу и состоянию, безопасное логирование через метод, возвращающий `false`, и отладочные точки останова через `Debugger.IsAttached`). В домашнем задании вы построите HTTP-клиент, в котором те же сценарии применяются к реальным статус-кодам, и экспериментально подтвердите ключевые тонкости урока: «честную» трассировку стека, отсутствие входа в `catch` при `false`, всплывание исключения из `when`.
(EN) The lesson introduces `when` as a declarative way to filter exceptions by a condition before entering the handler, and covers three main scenarios (type-and-state filtering, safe logging via a method that returns `false`, and debug-only breakpoints through `Debugger.IsAttached`). In this homework you will build an HTTP client that applies the same scenarios to real status codes, and you will experimentally confirm the lesson’s key subtleties: an honest stack trace, no entry into `catch` when the filter returns `false`, and propagation of an exception thrown from inside `when`.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Вы разрабатываете мини-сервис `WeatherProbe`, который обращается к публичному HTTP-эндпоинту погоды и возвращает температуру. Эндпоинт капризный: часть ошибок — временные (серверные 5xx, сетевые сбои), часть — постоянные (клиентские 4xx, особенно `404 Not Found`), а часть — критические, которые нельзя «проглатывать», но нужно залогировать для последующего разбора инцидента. Команда устала от антипаттерна «поймал, записал в лог, бросил заново»: он искажает трассировку стека, добавляет лишние кадры и мешает диагностике в продакшене. Урок M07-L05 предлагает декларативную альтернативу — фильтры `catch ... when (...)`, которые позволяют фильтровать исключения по условию до входа в обработчик, а также выполнять побочные эффекты (например, логирование) без остановки всплывания.

Вам предстоит спроектировать обработку ошибок так, чтобы: серверные ошибки 5xx обрабатывались локально (повторная попытка через делегирование вызывающему коду), клиентские ошибки 4xx только логировались, но продолжали всплывать к глобальному обработчику, а любые «неожиданные» исключения попадали в отладочный фильтр под `Debugger.IsAttached`. Дополнительно вы экспериментально докажете несколько тонких свойств фильтров: что при `when (false)` блок `catch` не выполняется вообще, что исключение, брошенное внутри `when`, не ловится этим же `catch`, и что трассировка стека при использовании `when` остаётся «честной» по сравнению с вариантом `catch + if + throw`. Это задание не про HTTP как таковой, а про осознанное применение фильтров исключений — темы урока M07-L05.

#### Что нужно сделать (пошагово)

1. Создайте новый проект консольного приложения на C# 12 / .NET 8 с помощью команды `dotnet new console -n WeatherProbe -o WeatherProbe -f net8.0`. Перейдите в каталог проекта `cd WeatherProbe`. Откройте файл `Program.cs` и удалите шаблонный код — вы будете писать top-level statements.

2. В файле `Program.cs` определите класс `WeatherProbe` в пространстве имён `M07L05.WhenFilters`. Класс должен содержать асинхронный метод `public async Task<int?> ReadTemperatureAsync(HttpClient client, string city)` и статический вспомогательный метод `private static bool LogAndReturnFalse(Exception ex, string context)` (точная сигнатура — на ваше усмотрение, но смысл: записать в `Console` тип и сообщение исключения и вернуть `false`). Запустите проект командой `dotnet build` — он должен компилироваться без предупреждений.

3. Поднимите локальный «фальшивый» сервер прямо в `Program.cs` через `HttpListener` или используйте `HttpClient` с delegating handler, который эмулирует разные статус-коды по URL. Можно упростить: используйте `HttpClientHandler`/`SocketsHttpHandler` невозможно подменить статус без сервера, поэтому проще всего поднять `HttpListener` на `http://localhost:5137/`, который по пути `/ok` возвращает `200 OK` с JSON `{"temp": 17}`, по пути `/server` возвращает `500 Internal Server Error`, по пути `/client` возвращает `404 Not Found`, а по пути `/throw` выбрасывает исключение внутри обработчика запроса. Команды для проверки: `dotnet run` — ожидается вывод с температурой и логами.

4. Реализуйте в `ReadTemperatureAsync` обработку через фильтры `when`:
   - `catch (HttpRequestException ex) when (ex.StatusCode is >= HttpStatusCode.InternalServerError)` — серверная ошибка: выведите сообщение «Серверная ошибка — повторим позднее» и сделайте `throw;`, чтобы делегировать повтор вызывающему.
   - `catch (HttpRequestException ex) when (LogAndReturnFalse(ex, "client-error") && ex.StatusCode is >= HttpStatusCode.BadRequest and < HttpStatusCode.InternalServerError)` — клиентская ошибка: только логируется, блок никогда не выполняется (фильтр возвращает `false`).
   - `catch (Exception ex) when (Debugger.IsAttached && LogAndReturnFalse(ex, "debug-inspect"))` — отладочный фильтр: активен только под отладчиком.

5. Добавьте глобальный `try/finally` в `Main`, чтобы продемонстрировать всплывание исключения: в `finally` печатайте «Глобальный обработчик достигнут» и (для тестов) перехватывайте всплывшее исключение на верхнем уровне, печатая его тип.

6. Напишите отдельный метод `DemonstrateHonestStackTrace()`, в котором сравниваются два варианта: `WhenApproach()` с `catch (Exception ex) when (SomeCondition(ex))` и `NaiveApproach()` с `catch (Exception ex) { if (!SomeCondition(ex)) throw; ... }`. В каждом случае распечатайте `ex.StackTrace` и заметьте, что в наивном варианте кадр `NaiveApproach` появляется как место повторного броска, тогда как в варианте с `when` трассировка указывает на первоисточник.

7. Напишите метод `DemonstrateFilterThrow()` для доказательства тонкости «исключение в `when` не ловится этим же `catch`». Внутри метода вызовите код, который бросает `InvalidOperationException`, и оберните его в `try { ... } catch (InvalidOperationException ex) when (FilterThatThrows(ex)) { ... }` поверх ещё одного `catch (Exception ex) { ... }`, который поймает уже исключение из фильтра. Распечатайте тип пойманного исключения — ожидается `InvalidOperationException` из фильтра, а не оригинал.

8. Скомпилируйте и запустите: `dotnet build`, затем `dotnet run`. Зафиксируйте ожидаемый вывод: для `/ok` — температура 17; для `/server` — сообщение о серверной ошибке и всплывшее исключение в глобальном обработчике; для `/client` — строка лога `[LOG] client-error: ...` и всплывшее исключение; для `/throw` — в обычном режиме ничего особенного, в режиме с подключённым отладчиком — строка `[LOG] debug-inspect: ...`.

9. Добавьте модульные тесты (опционально, через `xunit`): `dotnet new xunit -n WeatherProbe.Tests -o WeatherProbe.Tests`, добавьте reference, проверьте что `ReadTemperatureAsync` возвращает `17` для `/ok`, выбрасывает для `/server` и `/client`, и что `DemonstrateFilterThrow` действительно ловит исключение из фильтра.

#### Требования к решению

Решение должно использовать C# 12 / .NET 8: top-level statements в `Program.cs`, pattern matching с `is >= ... and < ...`, коллекционные выражения и raw string literals там, где они уместны (например, для JSON-ответа сервера). Код должен компилироваться без предупреждений и без подавлений через `#pragma`. Все три варианта `catch ... when (...)` из урока должны присутствовать: фильтрация по состоянию (5xx/4xx через `StatusCode`), логирование через метод, возвращающий `false`, и отладочный фильтр с `Debugger.IsAttached`. Использовать антипаттерн «catch + if + throw» в основном решении запрещено — он допустим только в методе `DemonstrateHonestStackTrace` для сравнения.

Имена файлов: `Program.cs` (top-level statements + класс `WeatherProbe`), опционально `WeatherProbe.Tests/UnitTest1.cs`. Код должен быть рабочим: при `dotnet run` вывод должен соответствовать описанию. В комментариях кода чередуйте RU и EN строки, как в примерах урока. Не используйте `throw ex;` (это обнулит трассировку) — только `throw;`. Не помещайте тяжёлые операции или бросания внутрь `when` (кроме демонстрационного метода `DemonstrateFilterThrow`, где бросание намеренное). Все исключения в `when`-выражениях основного решения должны быть детерминированными и без побочных мутаций состояния (только чтение `ex` и запись в `Console`/лог через вспомогательный метод).

#### Тонкости и подводные камни

- **`when (false)` ≠ «войти и ничего не сделать».** Если фильтр возвращает `false`, CLR ведёт себя так, будто `catch` не существует — блок не выполняется, локальные переменные не инициализируются. Это ключевое отличие от `catch + if + throw`, где блок всё-таки входит и только потом бросает заново. Студенты часто путают это и удивляются, почему «лог внутри `catch` не печатается» — потому что `when` вернул `false` и входа не было.

- **Исключение внутри `when` не ловится этим же `catch`.** Если `FilterThatThrows(ex)` выбрасывает `InvalidOperationException`, то CLR считает фильтр «неприменимым», и уже это новое исключение всплывает как отдельная ошибка — его поймает внешний `catch (Exception)`, а не текущий. Поэтому в `when` пишут только чистые детерминированные проверки; бросание там — почти всегда баг.

- **Порядок `catch` с одинаковым типом.** Несколько `catch (HttpRequestException ex) when (...)` с разными условиями компилируются без ошибок (без `when` это была бы ошибка CS0160). Выбирается первый подходящий — поэтому порядок имеет значение: сначала специфичные условия, потом более общие.

- **Честная трассировка стека.** При `catch + if + throw;` трассировка добавляет кадр повторного броска и может терять информацию о первоисточнике в некоторых сценариях (хотя `throw;` сохраняет исходный стек, сам факт входа в обработчик «загрязняет» логику). С `when` блок не входит, поэтому трассировка указывает на первоисточник — это и есть «честность».

- **Логирование через метод, возвращающий `false`.** Это устраняет антипаттерн «поймал, записал, бросил заново»: исключение залогировано, но всплывает нетронутым. Однако не злоупотребляйте побочными эффектами в `when` — они затрудняют рассуждения о порядке выполнения.

- **`Debugger.IsAttached`** в фильтре позволяет ставить «отладочные» точки останова только под подключённым отладчиком, не влияя на продакшен-поведение. Удобно для инспекции исключений в_dev-среде.

- **`throw;` vs `throw ex;`.** Всегда используйте `throw;` — он сохраняет исходную трассировку. `throw ex;` обнуляет `StackTrace` в месте повторного броска, что ломает диагностику.

#### Критерии приёмки

- [ ] Проект `WeatherProbe` создан командой `dotnet new console -f net8.0` и собирается без ошибок и предупреждений.
- [ ] Используется C# 12: top-level statements, pattern matching `is >= ... and < ...`, при необходимости raw strings / коллекционные выражения.
- [ ] В `ReadTemperatureAsync` присутствуют три `catch ... when (...)`: по 5xx, по 4xx через `LogAndReturnFalse`, отладочный с `Debugger.IsAttached`.
- [ ] Метод `LogAndReturnFalse` выполняет побочный эффект (печать в `Console`) и возвращает `false`; блок `catch` с ним никогда не выполняется.
- [ ] Для серверной ошибки 5xx выводится сообщение «Серверная ошибка — повторим позднее» и делается `throw;` (не `throw ex;`).
- [ ] Для клиентской ошибки 4xx в лог попадает строка `[LOG] client-error: HttpRequestException: ...`, а само исключение всплывает к глобальному обработчику.
- [ ] Глобальный обработчик на верхнем уровне ловит всплывшие исключения и печатает их тип.
- [ ] Метод `DemonstrateHonestStackTrace` сравнивает `when` и `catch + if + throw`, печатает `StackTrace` в обоих случаях.
- [ ] Метод `DemonstrateFilterThrow` доказывает, что исключение, брошенное внутри `when`, ловится внешним `catch (Exception)`, а не текущим `catch`.
- [ ] В основном решении нет бросаний и тяжёлых операций внутри `when` (кроме демонстрационного метода).
- [ ] Не используется `throw ex;` нигде, кроме как с явным комментарием-объяснением, почему это нужно (в эталоне — не используется вообще).
- [ ] Несколько `catch` с одинаковым типом `HttpRequestException` компилируются благодаря разным `when`-условиям.
- [ ] `dotnet run` даёт вывод, соответствующий описанию для всех четырёх путей (`/ok`, `/server`, `/client`, `/throw`).
- [ ] Код содержит двуязычные комментарии RU + EN, как в примерах урока.
- [ ] (Бонус) Добавлены модульные тесты `xunit`, проверяющие ключевые сценарии.

#### Подсказки (без прямого ответа)

- Для эмуляции HTTP без реального внешнего сервера проще всего использовать `HttpListener` в отдельной фоновой задаче: он умеет возвращать любой статус-код и тело ответа.
- Вспомните из урока, что `ex.StatusCode` у `HttpRequestException` может быть `null` (например, при сетевой ошибке без ответа) — учитывайте это в pattern matching через `is { } code and >= ...`.
- Метод `LogAndReturnFalse` должен возвращать `bool`, а не `void`, иначе его нельзя использовать в `when`.
- Для доказательства «честной трассировки» достаточно распечатать `ex.StackTrace` — кадр повторного броска в наивном варианте будет виден как строка с именем метода.
- Для `DemonstrateFilterThrow` определите локальную функцию `bool FilterThatThrows(Exception ex) => throw new InvalidOperationException("filter blew up");` — но учтите, что компилятор предупредит о недостижимом `return`; используйте `throw`-выражение аккуратно.

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8 — фильтры исключений через when (эталон ДЗ M07-L05)
// C# 12 / .NET 8 — exception filters with when (reference for HW M07-L05)

using System.Net;
using System.Diagnostics;
using System.Text;
using System.Text.Json;
using System.Net.Http;

namespace M07L05.WhenFilters;

// Простой эмулятор HTTP-сервера на HttpListener.
// A simple HTTP server emulator built on HttpListener.
public sealed class FakeWeatherServer : IDisposable
{
    private readonly HttpListener _listener = new();
    private CancellationTokenSource? _cts;

    public string BaseUrl => "http://localhost:5137/";

    public void Start()
    {
        _listener.Prefixes.Add(BaseUrl);
        _listener.Start();
        _cts = new CancellationTokenSource();
        _ = ListenAsync(_cts.Token);
    }

    // Цикл обработки запросов: раздаём разные статус-коды по пути.
    // Request loop: serve different status codes depending on the path.
    private async Task ListenAsync(CancellationToken token)
    {
        while (!token.IsCancellationRequested)
        {
            HttpListenerContext ctx;
            try { ctx = await _listener.GetContextAsync(); }
            catch (HttpListenerException) { break; }

            var path = ctx.Request.Url?.AbsolutePath ?? "/";
            var (status, body) = path switch
            {
                "/ok"     => (HttpStatusCode.OK, """{"temp":17}"""),
                "/server" => (HttpStatusCode.InternalServerError, """{"err":"boom"}"""),
                "/client" => (HttpStatusCode.NotFound, """{"err":"no city"}"""),
                "/throw"  => throw new InvalidOperationException("handler blew up"),
                _         => (HttpStatusCode.NotFound, """{"err":"unknown"}"""),
            };

            ctx.Response.StatusCode = (int)status;
            var bytes = Encoding.UTF8.GetBytes(body);
            await ctx.Response.OutputStream.WriteAsync(bytes, token);
            ctx.Response.Close();
        }
    }

    public void Dispose()
    {
        _cts?.Cancel();
        if (_listener.IsListening) _listener.Stop();
    }
}

public static class WeatherProbe
{
    // Безопасное логирование без проглатывания: пишет в лог и возвращает false.
    // Safe non-swallowing logging: writes a log entry and returns false.
    private static bool LogAndReturnFalse(Exception ex, string context)
    {
        Console.WriteLine($"[LOG] {context}: {ex.GetType().Name}: {ex.Message}");
        return false;
    }

    // Читает температуру из эндпоинта погоды с фильтрами when.
    // Reads the temperature from the weather endpoint using when filters.
    public static async Task<int?> ReadTemperatureAsync(HttpClient client, string path)
    {
        try
        {
            // Может выбросить HttpRequestException с разными статус-кодами.
            // May throw HttpRequestException with various status codes.
            var json = await client.GetStringAsync($"http://localhost:5137{path}");
            using var doc = JsonDocument.Parse(json);
            return doc.RootElement.GetProperty("temp").GetInt32();
        }
        // Серверная ошибка 5xx: обработаем локально, но делегируем повтор вызывающему.
        // Server error 5xx: handle locally, but delegate retry to the caller.
        catch (HttpRequestException ex) when (ex.StatusCode is >= HttpStatusCode.InternalServerError)
        {
            Console.WriteLine("Серверная ошибка — повторим позднее / Server error — will retry later");
            throw; // честная трассировка сохраняется / honest stack trace preserved
        }
        // Клиентская ошибка 4xx: только логируем, исключение всплывает дальше.
        // Client error 4xx: log only, the exception keeps propagating.
        catch (HttpRequestException ex) when (
            LogAndReturnFalse(ex, "client-error") &&
            ex.StatusCode is >= HttpStatusCode.BadRequest and < HttpStatusCode.InternalServerError)
        {
            // Недостижимо: фильтр всегда возвращает false.
            // Unreachable: the filter always returns false.
            throw;
        }
        // Отладочный фильтр: активен только под подключённым отладчиком.
        // Debug-only filter: active only when the debugger is attached.
        catch (Exception ex) when (Debugger.IsAttached && LogAndReturnFalse(ex, "debug-inspect"))
        {
            throw;
        }
    }

    // Демонстрация: трассировка при when vs catch+if+throw.
    // Demo: stack trace with when vs catch+if+throw.
    public static void DemonstrateHonestStackTrace()
    {
        Console.WriteLine("--- Honest stack trace demo ---");
        try { WhenApproach(); }
        catch (Exception ex) { Console.WriteLine($"when:        {ex.StackTrace?.Split('\n')[0]}"); }

        try { NaiveApproach(); }
        catch (Exception ex) { Console.WriteLine($"catch+if:    {ex.StackTrace?.Split('\n')[0]}"); }
    }

    private static bool SomeCondition(Exception ex) => ex.Message.Contains("boom");

    // Вариант с when: блок catch не входит, если условие ложно.
    // The when variant: the catch block is not entered when the condition is false.
    private static void WhenApproach()
    {
        try { throw new InvalidOperationException("boom"); }
        catch (InvalidOperationException ex) when (SomeCondition(ex))
        {
            // сюда не зайдём при false / not entered when false
            throw;
        }
    }

    // Наивный вариант: блок входит, потом бросает заново.
    // Naive variant: the block is entered, then rethrows.
    private static void NaiveApproach()
    {
        try { throw new InvalidOperationException("boom"); }
        catch (InvalidOperationException ex)
        {
            if (!SomeCondition(ex)) throw;
            throw;
        }
    }

    // Доказательство: исключение в when не ловится этим же catch.
    // Proof: an exception thrown inside when is not caught by the same catch.
    public static void DemonstrateFilterThrow()
    {
        Console.WriteLine("--- Filter-throw demo ---");
        try
        {
            try
            {
                throw new InvalidOperationException("original");
            }
            catch (InvalidOperationException ex) when (FilterThatThrows(ex))
            {
                // сюда не зайдём: фильтр выбросил / not entered: the filter threw
                throw;
            }
        }
        // Сюда придёт исключение из фильтра, а не оригинал.
        // The exception from the filter lands here, not the original.
        catch (Exception ex)
        {
            Console.WriteLine($"Поймано / Caught: {ex.GetType().Name}: {ex.Message}");
        }
    }

    // Фильтр, который сам выбрасывает — демонстрация тонкости урока.
    // A filter that itself throws — demonstrating the lesson's subtlety.
    private static bool FilterThatThrows(Exception ex) =>
        throw new InvalidOperationException("filter blew up");
}
```

Разбор по строкам. Класс `FakeWeatherServer` инкапсулирует `HttpListener` и по пути запроса отдаёт разные статус-коды через `switch`-выражение с pattern matching — это C# 12, raw string literals `"""..."""` для JSON. Метод `LogAndReturnFalse` — сердце второго сценария урока: побочный эффект (печать в лог) плюс возврат `false`, чтобы фильтр не сработал и исключение всплыло нетронутым. В `ReadTemperatureAsync` три блока `catch ... when (...)`: первый фильтрует по состоянию `StatusCode is >= HttpStatusCode.InternalServerError` (5xx), входит в блок и делегирует повтор через `throw;`; второй использует `LogAndReturnFalse` и поэтому никогда не входит — это и есть «безопасное логирование без проглатывания»; третий активен только под отладчиком (`Debugger.IsAttached`), реализуя третий сценарий урока. Обратите внимание: несколько `catch` с одинаковым типом `HttpRequestException` компилируются именно благодаря разным `when`-условиям — без них была бы ошибка CS0160. Используется только `throw;`, а не `throw ex;`, чтобы сохранить честную трассировку.

Метод `DemonstrateHonestStackTrace` сравнивает два подхода: `WhenApproach` использует `catch ... when (SomeCondition(ex))`, а `NaiveApproach` — `catch + if + throw`. В наивном варианте блок `catch` фактически входит (хотя условие ложно и сразу бросает), что добавляет шум в трассировку; в варианте с `when` блок не входит вовсе, и трассировка чище. Метод `DemonstrateFilterThrow` доказывает тонкость из урока: фильтр `FilterThatThrows` выбрасывает `InvalidOperationException`, и это новое исключение всплывает к внешнему `catch (Exception)` — вывод подтвердит, что пойман именно `filter blew up`, а не `original`. Эта тонкость объясняет, почему в `when` нельзя ставить бросания или тяжёлые операции: они ломают фильтрацию и превращаются в отдельную ошибку. Применённые концепции урока: фильтрация по состоянию, логирование через метод, возвращающий `false`, отладочный фильтр, честная трассировка, всплывание исключения из `when`.

#### Задания на углубление (бонус)

1. **HResult-фильтрация.** Добавьте сценарий, в котором `HttpRequestException` имеет `InnerException` с конкретным `HResult` (например, сетевой сбой `WSAECONNREFUSED`), и отфильтруйте его через `when (ex.InnerException?.HResult == 0x8007274D)`. Сравните с фильтрацией по `StatusCode`.
2. **Порядок catch.** Поменяйте местами блоки 5xx и 4xx и объясните, почему 5xx-блок перестанет срабатывать для некоторых запросов (подсказка: `LogAndReturnFalse` выполняется первым и возвращает `false`, но порядок вычисления условий в `&&` имеет значение). Покажите это тестом.
3. **Агрегация нескольких вызовов.** Реализуйте метод `ReadManyAsync`, который параллельно опрашивает несколько городов через `Task.WhenAll`, и обработайте `AggregateException` с фильтром `when (ex.InnerExceptions.Count > 1)` — логируйте только множественные сбои.
4. **Сравнение производительности.** Измерьте через `BenchmarkDotNet` разницу между `when` и `catch + if + throw` на 100 000 итераций. Объясните результат: CLR оптимизирует фильтры так, что при `false` не происходит входа в обработчик, тогда как наивный вариант всегда входит.

---

## Statement in English / Постановка на английском

#### Context & motivation

You are building a mini-service called `WeatherProbe` that calls a public HTTP weather endpoint and returns a temperature. The endpoint is temperamental: some failures are transient (server-side 5xx, network glitches), some are permanent (client-side 4xx, especially `404 Not Found`), and some are critical and must not be swallowed but must be logged for later incident review. Your team is tired of the “catch, log, rethrow” anti-pattern: it distorts the stack trace, adds extra frames, and complicates production diagnostics. Lesson M07-L05 offers a declarative alternative — `catch ... when (...)` filters that let you filter exceptions by a condition before entering the handler, and also let you perform side effects (such as logging) without stopping propagation.

You are to design the error handling so that: server-side 5xx errors are handled locally (retry is delegated to the caller), client-side 4xx errors are only logged but keep propagating to a global handler, and any “unexpected” exceptions land in a debug-only filter guarded by `Debugger.IsAttached`. In addition, you will experimentally prove several subtle properties of filters: that when `when (false)` returns false the `catch` block never executes at all, that an exception thrown inside `when` is not caught by the same `catch`, and that the stack trace under `when` stays “honest” compared to the `catch + if + throw` idiom. This assignment is not about HTTP per se — it is about the mindful application of exception filters, the topic of lesson M07-L05.

#### What to do step by step

1. Create a new C# 12 / .NET 8 console application with `dotnet new console -n WeatherProbe -o WeatherProbe -f net8.0`. Move into the project folder with `cd WeatherProbe`. Open `Program.cs` and delete the template code — you will write top-level statements.

2. In `Program.cs`, define a class `WeatherProbe` in the namespace `M07L05.WhenFilters`. The class must contain an async method `public async Task<int?> ReadTemperatureAsync(HttpClient client, string city)` and a static helper `private static bool LogAndReturnFalse(Exception ex, string context)` (the exact signature is up to you, but the intent is: write the exception’s type and message to `Console` and return `false`). Run `dotnet build` — the project should compile without warnings.

3. Stand up a local “fake” server right inside `Program.cs` using `HttpListener`, or use an `HttpClient` with a delegating handler that emulates various status codes by URL. Simplification: you cannot easily fake a status code without a server through `HttpClientHandler`/`SocketsHttpHandler`, so the easiest route is to start an `HttpListener` on `http://localhost:5137/` that returns `200 OK` with JSON `{"temp": 17}` for `/ok`, `500 Internal Server Error` for `/server`, `404 Not Found` for `/client`, and throws an exception inside the request handler for `/throw`. Verification command: `dotnet run` — you should see the temperature and the logs.

4. Implement `ReadTemperatureAsync` using `when` filters:
   - `catch (HttpRequestException ex) when (ex.StatusCode is >= HttpStatusCode.InternalServerError)` — server error: print “Server error — will retry later” and `throw;` to delegate retry to the caller.
   - `catch (HttpRequestException ex) when (LogAndReturnFalse(ex, "client-error") && ex.StatusCode is >= HttpStatusCode.BadRequest and < HttpStatusCode.InternalServerError)` — client error: log only, the block never executes (the filter returns `false`).
   - `catch (Exception ex) when (Debugger.IsAttached && LogAndReturnFalse(ex, "debug-inspect"))` — debug-only filter: active only under the debugger.

5. Add a global `try/finally` in `Main` to demonstrate propagation: in `finally`, print “Global handler reached” and (for the tests) catch the propagated exception at the top level, printing its type.

6. Write a separate method `DemonstrateHonestStackTrace()` that compares two variants: `WhenApproach()` with `catch (Exception ex) when (SomeCondition(ex))` and `NaiveApproach()` with `catch (Exception ex) { if (!SomeCondition(ex)) throw; ... }`. In each case print `ex.StackTrace` and notice that in the naive variant the `NaiveApproach` frame appears as the rethrow site, while in the `when` variant the trace points to the original source.

7. Write a method `DemonstrateFilterThrow()` to prove the subtlety “an exception thrown inside `when` is not caught by the same `catch`”. Inside the method, call code that throws `InvalidOperationException`, and wrap it in `try { ... } catch (InvalidOperationException ex) when (FilterThatThrows(ex)) { ... }` with an outer `catch (Exception ex) { ... }` that will catch the exception coming from the filter. Print the type of the caught exception — it should be the `InvalidOperationException` from the filter, not the original one.

8. Build and run: `dotnet build`, then `dotnet run`. Record the expected output: for `/ok` — temperature 17; for `/server` — the server-error message and the propagated exception at the global handler; for `/client` — a log line `[LOG] client-error: ...` and the propagated exception; for `/throw` — nothing special in normal mode, but under an attached debugger a line `[LOG] debug-inspect: ...`.

9. Add unit tests (optional, via `xunit`): `dotnet new xunit -n WeatherProbe.Tests -o WeatherProbe.Tests`, add a project reference, and assert that `ReadTemperatureAsync` returns `17` for `/ok`, throws for `/server` and `/client`, and that `DemonstrateFilterThrow` truly catches the exception from the filter.

#### Requirements

The solution must use C# 12 / .NET 8: top-level statements in `Program.cs`, pattern matching with `is >= ... and < ...`, collection expressions and raw string literals where appropriate (for example, for the JSON response body). The code must compile without warnings and without `#pragma` suppressions. All three `catch ... when (...)` variants from the lesson must be present: state-based filtering (5xx/4xx via `StatusCode`), logging through a method that returns `false`, and the debug-only filter with `Debugger.IsAttached`. The “catch + if + throw” anti-pattern is forbidden in the main solution — it is only allowed inside `DemonstrateHonestStackTrace` for comparison.

File names: `Program.cs` (top-level statements + the `WeatherProbe` class), optionally `WeatherProbe.Tests/UnitTest1.cs`. The code must run: `dotnet run` output must match the description. In the code comments, interleave RU and EN lines as in the lesson examples. Never use `throw ex;` (it resets the stack trace) — only `throw;`. Do not put heavy work or throws inside `when` (except in the demo method `DemonstrateFilterThrow`, where the throw is intentional). All expressions in `when` clauses of the main solution must be deterministic and free of mutating side effects (only reading `ex` and writing to `Console`/log via the helper).

#### Pitfalls

- **`when (false)` is not “enter and do nothing”.** When the filter returns `false`, the CLR behaves as if the `catch` clause did not exist — the block is not entered, locals are not initialized. This is the key difference from `catch + if + throw`, where the block is still entered and only then rethrows. Students often confuse this and wonder why “the log inside `catch` is not printed” — because `when` returned `false` and entry never happened.

- **An exception inside `when` is not caught by the same `catch`.** If `FilterThatThrows(ex)` throws `InvalidOperationException`, the CLR treats the filter as “not applicable”, and this new exception propagates as a separate error — it will be caught by an outer `catch (Exception)`, not the current one. That is why `when` should only contain pure deterministic checks; throwing there is almost always a bug.

- **Order of same-type `catch` clauses.** Multiple `catch (HttpRequestException ex) when (...)` with different conditions compile cleanly (without `when` this would be error CS0160). The first matching one wins — so order matters: put specific conditions first, then more general ones.

- **Honest stack trace.** With `catch + if + throw;` the trace gains a rethrow frame and may lose information about the original source in some scenarios (although `throw;` preserves the original stack, the very fact of entering the handler “pollutes” the logic). With `when` the block is not entered, so the trace points to the source — that is the “honesty”.

- **Logging through a method that returns `false`.** This eliminates the “catch, log, rethrow” anti-pattern: the exception is logged but propagates untouched. Still, do not abuse side effects in `when` — they make reasoning about execution order harder.

- **`Debugger.IsAttached`** in a filter lets you set “debug” breakpoints only under an attached debugger, without affecting production behavior. Handy for inspecting exceptions in a dev environment.

- **`throw;` vs `throw ex;`.** Always use `throw;` — it preserves the original stack trace. `throw ex;` resets `StackTrace` at the rethrow site, which breaks diagnostics.

#### Acceptance criteria

- [ ] The `WeatherProbe` project was created with `dotnet new console -f net8.0` and builds without errors or warnings.
- [ ] C# 12 is used: top-level statements, pattern matching `is >= ... and < ...`, raw strings / collection expressions where needed.
- [ ] `ReadTemperatureAsync` contains three `catch ... when (...)` clauses: 5xx, 4xx via `LogAndReturnFalse`, and a debug one with `Debugger.IsAttached`.
- [ ] The `LogAndReturnFalse` method performs a side effect (printing to `Console`) and returns `false`; the `catch` block that uses it never executes.
- [ ] For a 5xx server error, the message “Server error — will retry later” is printed and `throw;` is used (not `throw ex;`).
- [ ] For a 4xx client error, a log line `[LOG] client-error: HttpRequestException: ...` appears and the exception propagates to the global handler.
- [ ] The top-level global handler catches the propagated exceptions and prints their type.
- [ ] The method `DemonstrateHonestStackTrace` compares `when` and `catch + if + throw`, printing `StackTrace` in both cases.
- [ ] The method `DemonstrateFilterThrow` proves that an exception thrown inside `when` is caught by the outer `catch (Exception)`, not by the current `catch`.
- [ ] The main solution has no throws or heavy work inside `when` (except in the demo method).
- [ ] `throw ex;` is not used anywhere, except with an explicit comment explaining why it is needed (in the reference it is not used at all).
- [ ] Multiple `catch` clauses with the same `HttpRequestException` type compile thanks to distinct `when` conditions.
- [ ] `dotnet run` produces output matching the description for all four paths (`/ok`, `/server`, `/client`, `/throw`).
- [ ] The code contains bilingual RU + EN comments, as in the lesson examples.
- [ ] (Bonus) `xunit` unit tests are added covering the key scenarios.

#### Hints (no direct answer)

- To emulate HTTP without a real external server, the simplest approach is `HttpListener` in a background task: it can return any status code and body.
- Recall from the lesson that `ex.StatusCode` on `HttpRequestException` can be `null` (for example, on a network error with no response) — account for this in pattern matching with `is { } code and >= ...`.
- The `LogAndReturnFalse` method must return `bool`, not `void`, otherwise it cannot be used inside `when`.
- To prove the “honest stack trace”, it is enough to print `ex.StackTrace` — the rethrow frame in the naive variant will appear as a line with the method name.
- For `DemonstrateFilterThrow`, define a local function `bool FilterThatThrows(Exception ex) => throw new InvalidOperationException("filter blew up");` — but note the compiler will warn about unreachable `return`; use the throw expression carefully.

#### Reference solution walk-through

```csharp
// C# 12 / .NET 8 — exception filters with when (reference for HW M07-L05)

using System.Net;
using System.Diagnostics;
using System.Text;
using System.Text.Json;
using System.Net.Http;

namespace M07L05.WhenFilters;

// A simple HTTP server emulator built on HttpListener.
public sealed class FakeWeatherServer : IDisposable
{
    private readonly HttpListener _listener = new();
    private CancellationTokenSource? _cts;

    public string BaseUrl => "http://localhost:5137/";

    public void Start()
    {
        _listener.Prefixes.Add(BaseUrl);
        _listener.Start();
        _cts = new CancellationTokenSource();
        _ = ListenAsync(_cts.Token);
    }

    // Request loop: serve different status codes depending on the path.
    private async Task ListenAsync(CancellationToken token)
    {
        while (!token.IsCancellationRequested)
        {
            HttpListenerContext ctx;
            try { ctx = await _listener.GetContextAsync(); }
            catch (HttpListenerException) { break; }

            var path = ctx.Request.Url?.AbsolutePath ?? "/";
            var (status, body) = path switch
            {
                "/ok"     => (HttpStatusCode.OK, """{"temp":17}"""),
                "/server" => (HttpStatusCode.InternalServerError, """{"err":"boom"}"""),
                "/client" => (HttpStatusCode.NotFound, """{"err":"no city"}"""),
                "/throw"  => throw new InvalidOperationException("handler blew up"),
                _         => (HttpStatusCode.NotFound, """{"err":"unknown"}"""),
            };

            ctx.Response.StatusCode = (int)status;
            var bytes = Encoding.UTF8.GetBytes(body);
            await ctx.Response.OutputStream.WriteAsync(bytes, token);
            ctx.Response.Close();
        }
    }

    public void Dispose()
    {
        _cts?.Cancel();
        if (_listener.IsListening) _listener.Stop();
    }
}

public static class WeatherProbe
{
    // Safe non-swallowing logging: writes a log entry and returns false.
    private static bool LogAndReturnFalse(Exception ex, string context)
    {
        Console.WriteLine($"[LOG] {context}: {ex.GetType().Name}: {ex.Message}");
        return false;
    }

    // Reads the temperature from the weather endpoint using when filters.
    public static async Task<int?> ReadTemperatureAsync(HttpClient client, string path)
    {
        try
        {
            // May throw HttpRequestException with various status codes.
            var json = await client.GetStringAsync($"http://localhost:5137{path}");
            using var doc = JsonDocument.Parse(json);
            return doc.RootElement.GetProperty("temp").GetInt32();
        }
        // Server error 5xx: handle locally, but delegate retry to the caller.
        catch (HttpRequestException ex) when (ex.StatusCode is >= HttpStatusCode.InternalServerError)
        {
            Console.WriteLine("Server error — will retry later");
            throw; // honest stack trace preserved
        }
        // Client error 4xx: log only, the exception keeps propagating.
        catch (HttpRequestException ex) when (
            LogAndReturnFalse(ex, "client-error") &&
            ex.StatusCode is >= HttpStatusCode.BadRequest and < HttpStatusCode.InternalServerError)
        {
            // Unreachable: the filter always returns false.
            throw;
        }
        // Debug-only filter: active only when the debugger is attached.
        catch (Exception ex) when (Debugger.IsAttached && LogAndReturnFalse(ex, "debug-inspect"))
        {
            throw;
        }
    }

    // Demo: stack trace with when vs catch+if+throw.
    public static void DemonstrateHonestStackTrace()
    {
        Console.WriteLine("--- Honest stack trace demo ---");
        try { WhenApproach(); }
        catch (Exception ex) { Console.WriteLine($"when:        {ex.StackTrace?.Split('\n')[0]}"); }

        try { NaiveApproach(); }
        catch (Exception ex) { Console.WriteLine($"catch+if:    {ex.StackTrace?.Split('\n')[0]}"); }
    }

    private static bool SomeCondition(Exception ex) => ex.Message.Contains("boom");

    // The when variant: the catch block is not entered when the condition is false.
    private static void WhenApproach()
    {
        try { throw new InvalidOperationException("boom"); }
        catch (InvalidOperationException ex) when (SomeCondition(ex))
        {
            // not entered when false
            throw;
        }
    }

    // Naive variant: the block is entered, then rethrows.
    private static void NaiveApproach()
    {
        try { throw new InvalidOperationException("boom"); }
        catch (InvalidOperationException ex)
        {
            if (!SomeCondition(ex)) throw;
            throw;
        }
    }

    // Proof: an exception thrown inside when is not caught by the same catch.
    public static void DemonstrateFilterThrow()
    {
        Console.WriteLine("--- Filter-throw demo ---");
        try
        {
            try
            {
                throw new InvalidOperationException("original");
            }
            catch (InvalidOperationException ex) when (FilterThatThrows(ex))
            {
                // not entered: the filter threw
                throw;
            }
        }
        // The exception from the filter lands here, not the original.
        catch (Exception ex)
        {
            Console.WriteLine($"Caught: {ex.GetType().Name}: {ex.Message}");
        }
    }

    // A filter that itself throws — demonstrating the lesson's subtlety.
    private static bool FilterThatThrows(Exception ex) =>
        throw new InvalidOperationException("filter blew up");
}
```

Walk-through line by line. The `FakeWeatherServer` class wraps an `HttpListener` and returns different status codes depending on the request path through a `switch` expression with pattern matching — this is C# 12, with raw string literals `"""..."""` for JSON. The `LogAndReturnFalse` method is the heart of the lesson’s second scenario: a side effect (writing to the log) plus returning `false`, so the filter does not match and the exception propagates untouched. In `ReadTemperatureAsync` there are three `catch ... when (...)` blocks: the first filters by the state `StatusCode is >= HttpStatusCode.InternalServerError` (5xx), enters the block, and delegates retry via `throw;`; the second uses `LogAndReturnFalse` and therefore never enters — this is “safe logging without swallowing”; the third is active only under the debugger (`Debugger.IsAttached`), realizing the lesson’s third scenario. Note that several `catch` clauses with the same `HttpRequestException` type compile precisely because of the distinct `when` conditions — without them this would be error CS0160. Only `throw;` is used, never `throw ex;`, to preserve an honest stack trace.

The method `DemonstrateHonestStackTrace` compares the two approaches: `WhenApproach` uses `catch ... when (SomeCondition(ex))`, while `NaiveApproach` uses `catch + if + throw`. In the naive variant the `catch` block is actually entered (even though the condition is false and it immediately rethrows), which adds noise to the trace; in the `when` variant the block is never entered, and the trace is cleaner. The method `DemonstrateFilterThrow` proves the lesson’s subtlety: the filter `FilterThatThrows` throws `InvalidOperationException`, and this new exception propagates to the outer `catch (Exception)` — the output will confirm that what was caught is `filter blew up`, not `original`. This subtlety explains why `when` must not contain throws or heavy operations: they break filtering and turn into a separate error. Applied lesson concepts: state-based filtering, logging through a method that returns `false`, debug-only filter, honest stack trace, propagation of an exception thrown from `when`.

#### Going deeper (bonus)

1. **HResult filtering.** Add a scenario where `HttpRequestException` has an `InnerException` with a specific `HResult` (for example, a network failure `WSAECONNREFUSED`), and filter it with `when (ex.InnerException?.HResult == 0x8007274D)`. Compare this with filtering by `StatusCode`.
2. **Order of catch clauses.** Swap the 5xx and 4xx blocks and explain why the 5xx block stops firing for some requests (hint: `LogAndReturnFalse` is evaluated first and returns `false`, but the order of evaluation in `&&` matters). Demonstrate this with a test.
3. **Aggregating multiple calls.** Implement `ReadManyAsync` that polls several cities in parallel via `Task.WhenAll`, and handle `AggregateException` with a filter `when (ex.InnerExceptions.Count > 1)` — log only the multi-failure cases.
4. **Performance comparison.** Measure the difference between `when` and `catch + if + throw` over 100 000 iterations with `BenchmarkDotNet`. Explain the result: the CLR optimizes filters so that on `false` the handler is never entered, whereas the naive variant always enters.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Проект собирается без ошибок и предупреждений через `dotnet build`.
- [ ] (RU) Использованы top-level statements, pattern matching, raw strings — C# 12 / .NET 8.
- [ ] (RU) В решении есть три `catch ... when (...)`: по 5xx, по 4xx через `LogAndReturnFalse`, отладочный.
- [ ] (RU) Метод `LogAndReturnFalse` возвращает `false`, блок с ним не выполняется.
- [ ] (RU) Используется только `throw;`, нигде нет `throw ex;`.
- [ ] (RU) Методы `DemonstrateHonestStackTrace` и `DemonstrateFilterThrow` реализованы и работают.
- [ ] (RU) `dotnet run` выводит ожидаемый результат для всех четырёх путей.
- [ ] (RU) Комментарии в коде двуязычные RU + EN.
- [ ] (EN) The project builds without errors or warnings via `dotnet build`.
- [ ] (EN) Top-level statements, pattern matching, raw strings are used — C# 12 / .NET 8.
- [ ] (EN) The solution has three `catch ... when (...)` clauses: 5xx, 4xx via `LogAndReturnFalse`, debug.
- [ ] (EN) `LogAndReturnFalse` returns `false`; the block using it never executes.
- [ ] (EN) Only `throw;` is used; `throw ex;` appears nowhere.
- [ ] (EN) `DemonstrateHonestStackTrace` and `DemonstrateFilterThrow` are implemented and working.
- [ ] (EN) `dotnet run` produces the expected output for all four paths.
- [ ] (EN) Code comments are bilingual RU + EN.

#### Ресурсы / Resources
- [Microsoft Learn — `when` keyword](https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/when)
- [Microsoft Learn — Exception filters (C# 6+)](https://learn.microsoft.com/dotnet/csharp/language-reference/proposals/csharp-6.0/exception-filters)
- [Microsoft Learn — `HttpRequestException` class](https://learn.microsoft.com/dotnet/api/system.net.http.httprequestexception)
- [Microsoft Learn — `HttpListener` class](https://learn.microsoft.com/dotnet/api/system.net.httplistener)
- [Microsoft Learn — `Debugger.IsAttached` property](https://learn.microsoft.com/dotnet/api/system.diagnostics.debugger.isattached)
- [Lesson M07-L05: when-фильтры / when filters](lesson-M07-L05-when-filters.md)
