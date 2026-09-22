---
[← К уроку M14-L07](lesson-M14-L07-api-versioning.md) | [⬆ К модулю M14](../README.md) | [Следующее ДЗ →](homework-M14-L08-swagger-openapi.md)
---

### Домашнее задание M14-L07: Версионирование API / Homework M14-L07: API versioning

**Урок / Lesson:** M14-L07
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться подключать библиотеку Asp.Versioning в ASP.NET Core API на .NET 8, выбирать и последовательно применять один способ передачи версии (URL, query, header), объявлять устаревшие версии через `Deprecated = true` и заголовки `Sunset`/`Deprecation`, а также генерировать отдельный OpenAPI-документ на каждую версию, не ломая контракт v1 при разработке v2. (EN) Learn to wire up the Asp.Versioning library in an ASP.NET Core API on .NET 8, pick and consistently apply one versioning channel (URL, query, header), declare deprecated versions via `Deprecated = true` and the `Sunset`/`Deprecation` headers, and produce a separate OpenAPI document per version without breaking the v1 contract while working on v2.

#### Связь с уроком / Connection to the lesson
(RU) Урок вводит три классических способа передачи версии (URL, query string, header), показывает библиотеку `Asp.Versioning` как de-facto стандарт в .NET и объясняет, как декларировать версии атрибутом `[ApiVersion]`, включать `ReportApiVersions`, помечать версии устаревшими и группировать документы OpenAPI через `IApiVersionDescriptionProvider`. Это ДЗ закрепляет все эти механизмы на реальном коде и заставляет пройти полный цикл: настройка → два контроллера разных версий → устаревание → документация → контрактные тесты.
(EN) The lesson introduces the three classic versioning channels (URL, query string, header), presents `Asp.Versioning` as the .NET de-facto standard, and shows how to declare versions with `[ApiVersion]`, enable `ReportApiVersions`, mark versions as deprecated, and group OpenAPI documents via `IApiVersionDescriptionProvider`. This homework cements all of those mechanisms in real code and walks the full cycle: configuration → two versioned controllers → deprecation → documentation → contract tests.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Представьте, что вы сопровождаете небольшое публичное API интернет-магазина `ProductsApi`. Первый релиз (`v1.0`) уже полгода крутится в продакшене: мобильное приложение и два партнёрских интегратора дёргают эндпоинт `GET /api/v1/products` и получают массив объектов с полями `id` и `name`. Бизнес приходит с требованием: добавить валюту (`currency`) и переименовать `name` в `title`, потому что поле теперь хранит не только название, но и краткое описание для витрины. Технически это несовместимое изменение контракта: старые клиенты, читающие `name`, сломаются, а новые клиенты хотят `title` и `currency`.

Удалить v1 «в лоб» нельзя — мобильное приложение обновляется медленно, а партнёрские интеграции вообще не под вашим контролем. Согласно уроку, правильный путь — параллельно держать v1 и v2, дать клиентам окно миграции, пометить v1 как `Deprecated = true`, в ответе v1 возвращать заголовок `Sunset` с датой вывода из эксплуатации и `Deprecation: true`, а в документации OpenAPI завести отдельную группу на каждую версию. Параллельно нужно выбрать и зафиксировать один основной способ передачи версии (вы выберете URL versioning как самый наглядный) и оставить query string как запасной канал, как рекомендует урок.

Цель задания — пройти этот цикл руками: поднять проект, подключить `Asp.Versioning.Mvc` и `Asp.Versioning.Mvc.ApiExplorer`, настроить конвейер, написать два контроллера, объявить устаревание, убедиться, что v1 и v2 независимы, что заголовки `Sunset`/`Deprecation` действительно приходят, и что `IApiVersionDescriptionProvider` формирует отдельные описания версий для будущей Swagger-генерации (она будет в уроке M14-L08). Заодно вы напишете контрактные тесты, которые защитят v1 от случайных поломок, пока вы развиваете v2 — это прямо перекликается с best practice из урока «покрывайте каждую версию контрактными тестами».

#### Что нужно сделать (пошагово)
1. Создайте пустой проект веб-API. В каталоге решения выполните команду, которая создаёт новый проект с минимальными зависимостями: `dotnet new web -n ProductsApi -o ProductsApi`. Перейдите в каталог проекта `cd ProductsApi` и добавьте контроллеры, потому что шаблон `web` по умолчанию их не регистрирует: в `Program.cs` вызовите `builder.Services.AddControllers()` и `app.MapControllers()`. Убедитесь, что `dotnet run` стартует сервер и слушает `http://localhost:5000` (или порт из `launchSettings.json`). Откройте другой терминал и сделайте `dotnet build` — должны получить `Build succeeded` без предупреждений.

2. Добавьте пакеты версионирования. Выполните `dotnet add package Asp.Versioning.Mvc` и `dotnet add package Asp.Versioning.Mvc.ApiExplorer`. Эти два пакета — ровно те, что упомянуты в примере кода урока: первый даёт конвейер `AddApiVersioning`, второй — `AddApiExplorer` и `IApiVersionDescriptionProvider` для генерации групп OpenAPI. После установки проверьте `dotnet list package`, чтобы убедиться, что версии подтянулись совместимые с .NET 8.

3. Зарегистрируйте версионирование в `Program.cs`. В вызове `AddApiVersioning` задайте: `DefaultApiVersion = new ApiVersion(1, 0)`, `AssumeDefaultVersionWhenUnspecified = true` (чтобы клиенты, не указавшие версию, попадали в v1), `ReportApiVersions = true` (чтобы в ответе появлялся заголовок со списком поддерживаемых версий). В качестве `ApiVersionReader` используйте комбинированный ридер: `ApiVersionReader.Combine(new UrlSegmentApiVersionReader(), new QueryStringApiVersionReader("api-version"))` — это в точности повторяет рекомендацию урока «URL versioning как основной канал + query string как запасной». В `AddApiExplorer` задайте `GroupNameFormat = "'v'VVV"` и `SubstituteApiVersionInUrl = true`, чтобы группы назывались `v1`, `v2`, а сегмент версии подставлялся в URL автоматически.

4. Напишите контроллер `ProductsV1Controller`. Украсьте класс `[ApiController]`, `[ApiVersion("1.0", Deprecated = true)]` и маршрутом `[Route("api/v{version:apiVersion}/products")]`. Действие `GetAll` должно возвращать `Ok(new[] { new { id = 1, name = "Widget" } })` — старую форму ответа с полем `name`. Здесь `Deprecated = true` — флаг урока, который заставит библиотеку помечать версию устаревшей в метаданных.

5. Напишите контроллер `ProductsV2Controller` с тем же маршрутом, но `[ApiVersion("2.0")]`. Действие `GetAll` возвращает новую форму: `Ok(new[] { new { id = 1, title = "Widget", currency = "USD" } })`. Важно, что оба контроллера живут по одному маршруту `api/v{version:apiVersion}/products` — библиотека сама разнесёт запросы по версии из сегмента URL, и конфликта маршрутов не возникнет.

6. Добавьте заголовки `Sunset` и `Deprecation` к ответам v1. В `ProductsV1Controller` переопределите или используйте фильтр действия, который добавляет в `Response.Headers` значения `Sunset = "Sat, 31 Dec 2025 23:59:59 GMT"` (формат RFC 8594 — HTTP-date) и `Deprecation = "true"` (RFC 9745). Это закрывает требование урока: устаревшая версия должна явно сообщать клиенту о выводе из эксплуатации. Сделайте `dotnet run` и запросите `curl -i http://localhost:5000/api/v1/products` — в выводе `-i` вы должны увидеть оба заголовка.

7. Проверьте маршрутизацию версий. Выполните три запроса: `curl http://localhost:5000/api/v1/products` (должен вернуть `name`), `curl http://localhost:5000/api/v2/products` (должен вернуть `title` и `currency`), `curl "http://localhost:5000/api/v1/products?api-version=2.0"` (запасной канал query string — библиотека выберет v2, потому что явный query-параметр приоритетнее сегмента). Зафиксируйте выводы — это доказывает, что комбинированный ридер работает.

8. Получите описания версий. В `Program.cs` после `var app = builder.Build();` достаньте `IApiVersionDescriptionProvider` через `app.Services.GetRequiredService<IApiVersionDescriptionProvider>()` и выведите в консоль при старте количество описаний и их имена (`apiDescription.ApiVersionDescriptions`). Должно быть два описания — `v1` и `v2`. Это ровно тот хук, который в уроке M14-L08 будет использоваться для генерации Swagger-групп; здесь вы только убеждаетесь, что провайдер их формирует.

9. Напишите контрактные тесты. Добавьте тестовый проект `dotnet new xunit -o ProductsApi.Tests` и ссылку `dotnet add reference ../ProductsApi/ProductsApi.csproj`. В тесте используйте `WebApplicationFactory<Program>` из `Microsoft.AspNetCore.Mvc.Testing`, чтобы поднять приложение в памяти. Напишите три теста: (а) запрос `v1` возвращает поле `name` и заголовок `Deprecation`; (б) запрос `v2` возвращает `title` и `currency`; (в) запрос с `?api-version=2.0` к маршруту v1 действительно попадает в v2. Запустите `dotnet test` — все три должны быть зелёными.

10. Задокументируйте выбор способа версионирования. В файле `README.md` в корне проекта опишите: основной канал — URL (`/api/v{version:apiVersion}/...`), запасной — query string (`?api-version=`), дата вывода v1 — из заголовка `Sunset`, путь миграции — переезд с `name` на `title` и добавление `currency`. Это закрывает best practice урока «документируйте, как клиенту указывать версию и как читать заголовок Sunset».

#### Требования к решению
Решение должно быть оформлено как два проекта в одном solution: `ProductsApi` (основное приложение) и `ProductsApi.Tests` (xUnit). Целевая платформа — .NET 8, язык C# 12: используйте top-level statements в `Program.cs`, collection expressions там, где собираете списки, и file-scoped namespaces в тестах. Пакеты `Asp.Versioning.Mvc` и `Asp.Versioning.Mvc.ApiExplorer` должны быть явно указаны в `ProductsApi.csproj`. Регистрация версионирования обязана располагаться до `AddControllers` (порядок важен — иначе конвейер не подхватит атрибуты на контроллерах).

Каждый контроллер обязан иметь `[ApiController]` и явный `[ApiVersion(...)]`. V1 — `Deprecated = true`. Маршрут у обоих контроллеров — `api/v{version:apiVersion}/products`; библиотека разнесёт их по версиям, конфликта быть не должно. V1 возвращает объекты с `name`, v2 — с `title` и `currency`. Это несовместимое изменение контракта — ровно тот случай из урока, когда требуется новая мажорная версия.

V1 обязан отдавать заголовки `Sunset` (HTTP-date, дата в будущем) и `Deprecation: true` в каждом ответе. Заголовок `Sunset` должен иметь корректный формат RFC 8594 (`Day, DD Mon YYYY HH:MM:SS GMT`), иначе клиенты не смогут его распарсить. `ReportApiVersions = true` должен быть включён — тогда в ответе появляется заголовок со списком поддерживаемых версий, как требует урок.

Должен быть настроен `IApiVersionDescriptionProvider` (`AddApiExplorer` с `GroupNameFormat = "'v'VVV"` и `SubstituteApiVersionInUrl = true`). При старте приложение должно логировать две группы — `v1` и `v2`. Тесты должны покрывать три сценария: контракт v1, контракт v2 и приоритет query-параметра над сегментом URL. Все тесты — зелёные. В `README.md` зафиксированы выбранный стиль, дата вывода v1 и путь миграции.

#### Тонкости и подводные камни
Главная ловушка — порядок вызовов в `Program.cs`. Если вызвать `AddControllers()` до `AddApiVersioning()`, конвейер версионирования не увидит атрибуты `[ApiVersion]` на контроллерах, и все запросы будут падать в `404` или идти в первый попавшийся контроллер. Всегда регистрируйте версионирование первым. Вторая ловушка — конфликт маршрутов: оба контроллера нельзя вешать на один и тот же литеральный маршрут `api/v1/products` вручную; нужно использовать шаблон `api/v{version:apiVersion}/products` и дать библиотеке самой подставить версию, иначе ASP.NET Core на старте выбросит `AmbiguousMatchException`.

Третья тонкость касается `AssumeDefaultVersionWhenUnspecified`. Без этого флага запрос `GET /api/products` (без версии) вернёт ошибку, потому что библиотека не знает, какую версию выбрать. С флагом — попадёт в `DefaultApiVersion`. Но это не освобождает от явного указания версии в публичной документации: полагаться на «дефолт» опасно, потому что при будущем изменении дефолта поведение всех неявных клиентов молча поменяется. Поэтому в `README` фиксируйте основной канал как URL.

Четвёртая тонкость — формат заголовка `Sunset`. Это HTTP-date по RFC 7231/8594, например `Sat, 31 Dec 2025 23:59:59 GMT`. Если написать `31.12.2025` или `2025-12-31`, клиенты, умеющие читать `Sunset`, его проигнорируют или упадут. Заголовок `Deprecation` по RFC 9745 может быть либо `true`, либо HTTP-date, когда версия стала устаревшей; для v1 ставьте `Deprecation: true`. Пятая тонкость — `ReportApiVersions = true` добавляет заголовок `api-supported-versions` и `api-deprecated-versions`; в тестах можно проверять и их, но в задании достаточно `Deprecation`.

Шестая — комбинированный ридер и приоритет. Когда клиент шлёт `?api-version=2.0` на URL `/api/v1/products`, библиотека выбирает v2, потому что явный query-параметр приоритетнее сегмента. Это удобно для миграции, но может удивить: следите, чтобы в логах вы видели реально выбранную версию, а не ту, что в URL. Седьмая — `Deprecated = true` в атрибуте автоматически добавляет версию в `api-deprecated-versions`, но сам заголовок `Sunset` вы добавляете сами; библиотека его не генерирует. Восьмая — `IApiVersionDescriptionProvider` формирует описания на основе атрибутов, и если вы забыли `[ApiVersion]` на каком-то контроллере, этой версии в группах не появится.

#### Критерии приёмки
- [ ] Создан проект `ProductsApi` на .NET 8 с top-level statements в `Program.cs`.
- [ ] Установлены пакеты `Asp.Versioning.Mvc` и `Asp.Versioning.Mvc.ApiExplorer`.
- [ ] `AddApiVersioning` вызывается до `AddControllers` и настраивает `DefaultApiVersion`, `AssumeDefaultVersionWhenUnspecified`, `ReportApiVersions`.
- [ ] `ApiVersionReader` комбинирует `UrlSegmentApiVersionReader` и `QueryStringApiVersionReader("api-version")`.
- [ ] `AddApiExplorer` настроен с `GroupNameFormat = "'v'VVV"` и `SubstituteApiVersionInUrl = true`.
- [ ] `ProductsV1Controller` помечен `[ApiVersion("1.0", Deprecated = true)]` и возвращает `name`.
- [ ] `ProductsV2Controller` помечен `[ApiVersion("2.0")]` и возвращает `title` + `currency`.
- [ ] Оба контроллера используют маршрут `api/v{version:apiVersion}/products` без конфликтов.
- [ ] V1 добавляет заголовки `Sunset` (корректный HTTP-date) и `Deprecation: true` в каждый ответ.
- [ ] `curl /api/v1/products` возвращает старую форму, `curl /api/v2/products` — новую.
- [ ] `curl "/api/v1/products?api-version=2.0"` попадает в v2 (приоритет query над сегментом).
- [ ] При старте приложение логирует две группы версий от `IApiVersionDescriptionProvider`.
- [ ] Тестовый проект `ProductsApi.Tests` (xUnit) с тремя зелёными тестами: контракт v1, контракт v2, приоритет query.
- [ ] В `README.md` зафиксированы выбранный стиль, дата вывода v1 и путь миграции.
- [ ] `dotnet build` и `dotnet test` проходят без предупреждений и ошибок.

#### Подсказки (без прямого ответа)
- Если получаете `AmbiguousMatchException` на старте — проверьте, что у обоих контроллеров маршрут содержит сегмент `{version:apiVersion}`, а не захардкожен `v1`/`v2`.
- Если `IApiVersionDescriptionProvider` отдаёт пустой список — проверьте, что `AddApiExplorer` действительно вызван, и что на каждом контроллере есть атрибут `[ApiVersion]`.
- Заголовок `Sunset` добавляйте в фильтре действия или прямо в действии через `Response.Headers["Sunset"] = ...`; помните про формат HTTP-date.
- Для тестов используйте `WebApplicationFactory<Program>` и `HttpClient` — не нужно поднимать реальный порт.
- Чтобы проверить приоритет query над сегментом, шлите запрос на URL с v1 в пути, но с `?api-version=2.0` в query, и проверьте тело ответа.

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 — Эталонное решение ДЗ M14-L07
// C# 12 / .NET 8 — Reference solution for homework M14-L07
//
// Пакеты / Packages:
//   Asp.Versioning.Mvc
//   Asp.Versioning.Mvc.ApiExplorer

using Asp.Versioning;
using Asp.Versioning.ApiExplorer;
using Microsoft.AspNetCore.Mvc;

var builder = WebApplication.CreateBuilder(args);

// Регистрируем версионирование ПЕРВЫМ — до AddControllers.
// Register versioning FIRST — before AddControllers.
builder.Services.AddApiVersioning(options =>
{
    // Версия по умолчанию, если клиент её не указал.
    // Default version assumed when the client omits it.
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;

    // Комбинированный ридер: URL-сегмент + query string.
    // Combined reader: URL segment + query string.
    options.ApiVersionReader = ApiVersionReader.Combine(
        new UrlSegmentApiVersionReader(),
        new QueryStringApiVersionReader("api-version"));

    // Передаём поддерживаемые версии в заголовке ответа.
    // Report supported versions in a response header.
    options.ReportApiVersions = true;
})
.AddApiExplorer(options =>
{
    // Имена групп: v1, v2.
    // Group names: v1, v2.
    options.GroupNameFormat = "'v'VVV";
    options.SubstituteApiVersionInUrl = true;
});

builder.Services.AddControllers();

var app = builder.Build();

app.UseRouting();
app.MapControllers();

// Логируем описания версий при старте.
// Log version descriptions at startup.
var apiDescription = app.Services.GetRequiredService<IApiVersionDescriptionProvider>();
foreach (var desc in apiDescription.ApiVersionDescriptions)
{
    Console.WriteLine($"Version group: {desc.GroupName}, deprecated: {desc.IsDeprecated}");
}

app.Run();

// ---- Контроллер v1 — устаревший ----
// ---- v1 controller — deprecated ----
[ApiController]
[ApiVersion("1.0", Deprecated = true)]
[Route("api/v{version:apiVersion}/products")]
public class ProductsV1Controller : ControllerBase
{
    [HttpGet]
    public IActionResult GetAll()
    {
        // Заголовок Sunset — RFC 8594, HTTP-date в будущем.
        // Sunset header — RFC 8594, HTTP-date in the future.
        Response.Headers["Sunset"] = "Sat, 31 Dec 2025 23:59:59 GMT";
        // Заголовок Deprecation — RFC 9745.
        // Deprecation header — RFC 9745.
        Response.Headers["Deprecation"] = "true";

        // Старая форма контракта: поле name.
        // Legacy contract shape: name field.
        return Ok(new[] { new { id = 1, name = "Widget" } });
    }
}

// ---- Контроллер v2 — актуальный ----
// ---- v2 controller — current ----
[ApiController]
[ApiVersion("2.0")]
[Route("api/v{version:apiVersion}/products")]
public class ProductsV2Controller : ControllerBase
{
    [HttpGet]
    public IActionResult GetAll() =>
        // Новая форма: title вместо name + поле currency.
        // New shape: title instead of name + currency field.
        Ok(new[] { new { id = 1, title = "Widget", currency = "USD" } });
}

public partial class Program { }
```

Разбор по строкам. `AddApiVersioning` вызывается строго до `AddControllers` — это та самая частая ошибка из урока: если порядок нарушен, атрибуты `[ApiVersion]` не подхватываются. `DefaultApiVersion` и `AssumeDefaultVersionWhenUnspecified` дают мягкое поведение для клиентов без версии, как требует урок. `ApiVersionReader.Combine` с `UrlSegmentApiVersionReader` и `QueryStringApiVersionReader` реализует рекомендацию «URL как основной канал, query как запасной». `ReportApiVersions = true` включает заголовки `api-supported-versions` и `api-deprecated-versions` — это best practice из урока. `AddApiExplorer` с `GroupNameFormat = "'v'VVV"` формирует имена `v1` и `v2`, которые в уроке M14-L08 станут группами Swagger; `SubstituteApiVersionInUrl = true` позволяет OpenAPI-генератору подставлять версию в URL автоматически. `IApiVersionDescriptionProvider` достаётся после `Build()` и используется только для логирования групп — в реальном Swagger-сценарии по нему итерирует конфигурация Swashbuckle.

В контроллере v1 флаг `Deprecated = true` помечает версию устаревшей в метаданных библиотеки; это автоматически добавляет v1 в `api-deprecated-versions`, но не генерирует заголовок `Sunset` — поэтому мы добавляем его вручную в действии, с корректным HTTP-date по RFC 8594. Заголовок `Deprecation: true` соответствует RFC 9745. Возврат `name` — старый контракт; v2 возвращает `title` и `currency` — новый, несовместимый контракт, что по уроку требует именно новой мажорной версии, а не «багфикса» v1. Одинаковый маршрут `api/v{version:apiVersion}/products` не конфликтует, потому что сегмент `{version:apiVersion}` — это ограничение маршрута, которое библиотека использует для выбора контроллера; ручное дублирование литерала `api/v1/products` в обоих классах вызвало бы `AmbiguousMatchException`. `public partial class Program { }` делает класс `Program` доступным для `WebApplicationFactory<Program>` в тестах — это идиома .NET 6+ для интеграционного тестирования minimal API / top-level programs.

#### Задания на углубление (бонус)
1. Добавьте header-based версионирование для эндпоинта `GET /api/orders`: один контроллер с двумя атрибутами `[ApiVersion("1.0")]` и `[ApiVersion("2.0")]`, версия выбирается заголовком `X-Api-Version`. Используйте `HeaderApiVersionReader("X-Api-Version")` в комбинации с существующими ридерами.
2. Реализуйте собственный `IAppliesToApiController` или фильтр, который для любой устаревшей версии автоматически проставляет заголовок `Sunset` и `Deprecation`, чтобы не дублировать код в каждом v1-контроллере.
3. Покройте контракт каждой версии тестами на схему (например, через JSON Schema или `Shouldly` + `JToken`), чтобы любое изменение полей `name`/`title`/`currency` падало в CI.
4. Сэмулируйте окно миграции: сделайте v1 возвращать в теле ответа подсказку `migrationHint: "use v2; rename name → title, add currency"`, и напишите тест, проверяющий наличие этого поля только в v1.

---

## Statement in English / Постановка на английском

#### Context & motivation
Imagine you maintain a small public API for an online shop called `ProductsApi`. The first release (`v1.0`) has been running in production for six months: a mobile app and two partner integrations call `GET /api/v1/products` and receive an array of objects with `id` and `name` fields. The business comes with a requirement: add a `currency` field and rename `name` to `title`, because the field now stores not only the product name but also a short storefront description. Technically this is a breaking contract change: old clients reading `name` will break, while new clients want `title` and `currency`.

You cannot delete v1 outright — the mobile app updates slowly, and partner integrations are not under your control. Following the lesson, the correct path is to keep v1 and v2 running in parallel, give clients a migration window, mark v1 as `Deprecated = true`, return a `Sunset` header with the retirement date and `Deprecation: true` in v1 responses, and create a separate OpenAPI group per version in the documentation. At the same time you need to pick and fix one primary versioning channel (you will choose URL versioning for maximum visibility) and keep query string as a fallback channel, exactly as the lesson recommends.

The goal of this homework is to walk this cycle hands-on: create the project, add `Asp.Versioning.Mvc` and `Asp.Versioning.Mvc.ApiExplorer`, configure the pipeline, write two controllers, declare deprecation, verify that v1 and v2 are independent, that the `Sunset`/`Deprecation` headers actually arrive, and that `IApiVersionDescriptionProvider` produces separate version descriptions for future Swagger generation (which will be covered in lesson M14-L08). Along the way you will write contract tests that protect v1 from accidental breakage while you evolve v2 — this directly mirrors the lesson's best practice "cover each version with contract tests so v1 is not accidentally broken while working on v2".

#### What to do step by step
1. Create an empty web API project. From your solution folder run the command that creates a new project with minimal dependencies: `dotnet new web -n ProductsApi -o ProductsApi`. Move into the project folder `cd ProductsApi` and add controllers, because the `web` template does not register them by default: in `Program.cs` call `builder.Services.AddControllers()` and `app.MapControllers()`. Confirm that `dotnet run` starts the server and listens on `http://localhost:5000` (or the port from `launchSettings.json`). Open another terminal and run `dotnet build` — you should get `Build succeeded` with no warnings.

2. Add the versioning packages. Run `dotnet add package Asp.Versioning.Mvc` and `dotnet add package Asp.Versioning.Mvc.ApiExplorer`. These two packages are exactly the ones referenced in the lesson's code sample: the first provides the `AddApiVersioning` pipeline, the second provides `AddApiExplorer` and `IApiVersionDescriptionProvider` for OpenAPI group generation. After installation run `dotnet list package` to confirm that the resolved versions are compatible with .NET 8.

3. Register versioning in `Program.cs`. Inside the `AddApiVersioning` call set: `DefaultApiVersion = new ApiVersion(1, 0)`, `AssumeDefaultVersionWhenUnspecified = true` (so clients that omit the version land on v1), and `ReportApiVersions = true` (so a response header with the list of supported versions appears). For `ApiVersionReader` use a combined reader: `ApiVersionReader.Combine(new UrlSegmentApiVersionReader(), new QueryStringApiVersionReader("api-version"))` — this reproduces the lesson's recommendation "URL versioning as the primary channel plus query string as a fallback". In `AddApiExplorer` set `GroupNameFormat = "'v'VVV"` and `SubstituteApiVersionInUrl = true`, so groups are named `v1`, `v2`, and the version segment is substituted into the URL automatically.

4. Write the `ProductsV1Controller`. Decorate the class with `[ApiController]`, `[ApiVersion("1.0", Deprecated = true)]`, and the route `[Route("api/v{version:apiVersion}/products")]`. The `GetAll` action should return `Ok(new[] { new { id = 1, name = "Widget" } })` — the legacy response shape with the `name` field. The `Deprecated = true` flag is the lesson's signal that makes the library mark the version as deprecated in metadata.

5. Write the `ProductsV2Controller` with the same route but `[ApiVersion("2.0")]`. The `GetAll` action returns the new shape: `Ok(new[] { new { id = 1, title = "Widget", currency = "USD" } })`. Note that both controllers live under the same route `api/v{version:apiVersion}/products` — the library itself dispatches requests based on the version segment, and no route conflict occurs.

6. Add `Sunset` and `Deprecation` headers to v1 responses. In `ProductsV1Controller` override the action or use an action filter that adds to `Response.Headers` the values `Sunset = "Sat, 31 Dec 2025 23:59:59 GMT"` (RFC 8594 HTTP-date format) and `Deprecation = "true"` (RFC 9745). This closes the lesson's requirement: a deprecated version must explicitly tell clients it is being retired. Run `dotnet run` and request `curl -i http://localhost:5000/api/v1/products` — the `-i` output must show both headers.

7. Verify version routing. Run three requests: `curl http://localhost:5000/api/v1/products` (should return `name`), `curl http://localhost:5000/api/v2/products` (should return `title` and `currency`), `curl "http://localhost:5000/api/v1/products?api-version=2.0"` (the query string fallback channel — the library will select v2 because the explicit query parameter takes priority over the segment). Record the outputs — this proves the combined reader works.

8. Obtain version descriptions. In `Program.cs` after `var app = builder.Build();` resolve `IApiVersionDescriptionProvider` via `app.Services.GetRequiredService<IApiVersionDescriptionProvider>()` and log the count and names of descriptions (`apiDescription.ApiVersionDescriptions`) at startup. There should be two descriptions — `v1` and `v2`. This is exactly the hook that lesson M14-L08 will use to generate Swagger groups; here you only confirm the provider produces them.

9. Write contract tests. Add a test project `dotnet new xunit -o ProductsApi.Tests` and a reference `dotnet add reference ../ProductsApi/ProductsApi.csproj`. In the test use `WebApplicationFactory<Program>` from `Microsoft.AspNetCore.Mvc.Testing` to boot the app in memory. Write three tests: (a) a `v1` request returns the `name` field and the `Deprecation` header; (b) a `v2` request returns `title` and `currency`; (c) a request with `?api-version=2.0` against the v1 route actually lands in v2. Run `dotnet test` — all three must be green.

10. Document the versioning choice. In a `README.md` at the project root describe: the primary channel is URL (`/api/v{version:apiVersion}/...`), the fallback is query string (`?api-version=`), the v1 retirement date comes from the `Sunset` header, and the migration path is moving from `name` to `title` and adopting `currency`. This closes the lesson's best practice "document how a client should send the version and read the Sunset header".

#### Requirements
The solution must be structured as two projects in one solution: `ProductsApi` (the application) and `ProductsApi.Tests` (xUnit). The target platform is .NET 8, language C# 12: use top-level statements in `Program.cs`, collection expressions where you build lists, and file-scoped namespaces in tests. The packages `Asp.Versioning.Mvc` and `Asp.Versioning.Mvc.ApiExplorer` must be explicitly listed in `ProductsApi.csproj`. The versioning registration must come before `AddControllers` (order matters — otherwise the pipeline will not pick up the attributes on the controllers).

Each controller must have `[ApiController]` and an explicit `[ApiVersion(...)]`. V1 is `Deprecated = true`. The route for both controllers is `api/v{version:apiVersion}/products`; the library will dispatch them by version, and there must be no conflict. V1 returns objects with `name`, v2 returns objects with `title` and `currency`. This is a breaking contract change — exactly the case from the lesson where a new major version is required.

V1 must emit a `Sunset` header (HTTP-date, a future date) and `Deprecation: true` on every response. The `Sunset` header must follow the RFC 8594 format (`Day, DD Mon YYYY HH:MM:SS GMT`), otherwise clients will not be able to parse it. `ReportApiVersions = true` must be enabled — then the response carries a header listing the supported versions, as the lesson requires.

`IApiVersionDescriptionProvider` must be configured (`AddApiExplorer` with `GroupNameFormat = "'v'VVV"` and `SubstituteApiVersionInUrl = true`). At startup the application must log two groups — `v1` and `v2`. Tests must cover three scenarios: the v1 contract, the v2 contract, and the priority of the query parameter over the URL segment. All tests must be green. The `README.md` must record the chosen style, the v1 retirement date, and the migration path.

#### Pitfalls
The main trap is the call order in `Program.cs`. If you call `AddControllers()` before `AddApiVersioning()`, the versioning pipeline will not see the `[ApiVersion]` attributes on the controllers, and every request will fall through to `404` or hit the first controller. Always register versioning first. The second trap is route conflicts: you cannot hang both controllers on the same literal route `api/v1/products` manually; you must use the template `api/v{version:apiVersion}/products` and let the library substitute the version, otherwise ASP.NET Core will throw `AmbiguousMatchException` at startup.

The third pitfall concerns `AssumeDefaultVersionWhenUnspecified`. Without this flag a `GET /api/products` request (no version) will fail, because the library does not know which version to pick. With the flag it falls back to `DefaultApiVersion`. But this does not free you from documenting the version explicitly in public docs: relying on the default is dangerous, because a future change to the default will silently change the behaviour of every implicit client. That is why the `README` must fix the primary channel as URL.

The fourth pitfall is the `Sunset` header format. It is an HTTP-date per RFC 7231/8594, for example `Sat, 31 Dec 2025 23:59:59 GMT`. If you write `31.12.2025` or `2025-12-31`, clients that understand `Sunset` will ignore it or crash. The `Deprecation` header per RFC 9745 may be either `true` or an HTTP-date marking when the version became deprecated; for v1 use `Deprecation: true`. The fifth pitfall — `ReportApiVersions = true` adds the `api-supported-versions` and `api-deprecated-versions` headers; you can assert on them in tests too, but the assignment only requires `Deprecation`.

The sixth pitfall — the combined reader and priority. When a client sends `?api-version=2.0` to the URL `/api/v1/products`, the library picks v2, because the explicit query parameter takes priority over the segment. This is convenient for migration but can surprise you: make sure your logs show the actually selected version, not the one in the URL. The seventh — `Deprecated = true` on the attribute automatically adds the version to `api-deprecated-versions`, but you add the `Sunset` header yourself; the library does not generate it. The eighth — `IApiVersionDescriptionProvider` builds descriptions from the attributes, and if you forgot `[ApiVersion]` on some controller, that version will not appear in the groups.

#### Acceptance criteria
- [ ] A `ProductsApi` project on .NET 8 with top-level statements in `Program.cs` is created.
- [ ] Packages `Asp.Versioning.Mvc` and `Asp.Versioning.Mvc.ApiExplorer` are installed.
- [ ] `AddApiVersioning` is called before `AddControllers` and configures `DefaultApiVersion`, `AssumeDefaultVersionWhenUnspecified`, `ReportApiVersions`.
- [ ] `ApiVersionReader` combines `UrlSegmentApiVersionReader` and `QueryStringApiVersionReader("api-version")`.
- [ ] `AddApiExplorer` is configured with `GroupNameFormat = "'v'VVV"` and `SubstituteApiVersionInUrl = true`.
- [ ] `ProductsV1Controller` is marked `[ApiVersion("1.0", Deprecated = true)]` and returns `name`.
- [ ] `ProductsV2Controller` is marked `[ApiVersion("2.0")]` and returns `title` + `currency`.
- [ ] Both controllers use the route `api/v{version:apiVersion}/products` with no conflict.
- [ ] V1 adds a `Sunset` header (correct HTTP-date) and `Deprecation: true` to every response.
- [ ] `curl /api/v1/products` returns the legacy shape, `curl /api/v2/products` returns the new one.
- [ ] `curl "/api/v1/products?api-version=2.0"` lands in v2 (query priority over segment).
- [ ] At startup the application logs two version groups from `IApiVersionDescriptionProvider`.
- [ ] A test project `ProductsApi.Tests` (xUnit) with three green tests: v1 contract, v2 contract, query priority.
- [ ] The `README.md` records the chosen style, the v1 retirement date, and the migration path.
- [ ] `dotnet build` and `dotnet test` pass with no warnings or errors.

#### Hints
- If you get `AmbiguousMatchException` at startup — check that both controllers use a route with the `{version:apiVersion}` segment, not a hardcoded `v1`/`v2`.
- If `IApiVersionDescriptionProvider` returns an empty list — confirm `AddApiExplorer` is actually called and that every controller has the `[ApiVersion]` attribute.
- Add the `Sunset` header in an action filter or directly in the action via `Response.Headers["Sunset"] = ...`; remember the HTTP-date format.
- For tests use `WebApplicationFactory<Program>` and `HttpClient` — no need to spin up a real port.
- To check query priority over the segment, send a request to a URL with v1 in the path but `?api-version=2.0` in the query, and inspect the response body.

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8 — Reference solution for homework M14-L07
// C# 12 / .NET 8 — Reference solution for homework M14-L07
//
// Packages:
//   Asp.Versioning.Mvc
//   Asp.Versioning.Mvc.ApiExplorer

using Asp.Versioning;
using Asp.Versioning.ApiExplorer;
using Microsoft.AspNetCore.Mvc;

var builder = WebApplication.CreateBuilder(args);

// Register versioning FIRST — before AddControllers.
builder.Services.AddApiVersioning(options =>
{
    // Default version assumed when the client omits it.
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;

    // Combined reader: URL segment + query string.
    options.ApiVersionReader = ApiVersionReader.Combine(
        new UrlSegmentApiVersionReader(),
        new QueryStringApiVersionReader("api-version"));

    // Report supported versions in a response header.
    options.ReportApiVersions = true;
})
.AddApiExplorer(options =>
{
    // Group names: v1, v2.
    options.GroupNameFormat = "'v'VVV";
    options.SubstituteApiVersionInUrl = true;
});

builder.Services.AddControllers();

var app = builder.Build();

app.UseRouting();
app.MapControllers();

// Log version descriptions at startup.
var apiDescription = app.Services.GetRequiredService<IApiVersionDescriptionProvider>();
foreach (var desc in apiDescription.ApiVersionDescriptions)
{
    Console.WriteLine($"Version group: {desc.GroupName}, deprecated: {desc.IsDeprecated}");
}

app.Run();

// ---- v1 controller — deprecated ----
[ApiController]
[ApiVersion("1.0", Deprecated = true)]
[Route("api/v{version:apiVersion}/products")]
public class ProductsV1Controller : ControllerBase
{
    [HttpGet]
    public IActionResult GetAll()
    {
        // Sunset header — RFC 8594, HTTP-date in the future.
        Response.Headers["Sunset"] = "Sat, 31 Dec 2025 23:59:59 GMT";
        // Deprecation header — RFC 9745.
        Response.Headers["Deprecation"] = "true";

        // Legacy contract shape: name field.
        return Ok(new[] { new { id = 1, name = "Widget" } });
    }
}

// ---- v2 controller — current ----
[ApiController]
[ApiVersion("2.0")]
[Route("api/v{version:apiVersion}/products")]
public class ProductsV2Controller : ControllerBase
{
    [HttpGet]
    public IActionResult GetAll() =>
        // New shape: title instead of name + currency field.
        Ok(new[] { new { id = 1, title = "Widget", currency = "USD" } });
}

public partial class Program { }
```

Walk-through, line by line. `AddApiVersioning` is called strictly before `AddControllers` — this is the very common mistake from the lesson: if the order is wrong, the `[ApiVersion]` attributes are not picked up. `DefaultApiVersion` and `AssumeDefaultVersionWhenUnspecified` give soft behaviour for clients without a version, as the lesson requires. `ApiVersionReader.Combine` with `UrlSegmentApiVersionReader` and `QueryStringApiVersionReader` implements the recommendation "URL as the primary channel, query as a fallback". `ReportApiVersions = true` turns on the `api-supported-versions` and `api-deprecated-versions` headers — a best practice from the lesson. `AddApiExplorer` with `GroupNameFormat = "'v'VVV"` produces the names `v1` and `v2`, which will become Swagger groups in lesson M14-L08; `SubstituteApiVersionInUrl = true` lets the OpenAPI generator substitute the version into the URL automatically. `IApiVersionDescriptionProvider` is resolved after `Build()` and used only for logging groups — in a real Swagger scenario Swashbuckle iterates over it.

In the v1 controller the `Deprecated = true` flag marks the version as deprecated in the library's metadata; this automatically adds v1 to `api-deprecated-versions`, but it does not generate the `Sunset` header — that is why we add it manually in the action, with a correct HTTP-date per RFC 8594. The `Deprecation: true` header matches RFC 9745. Returning `name` is the legacy contract; v2 returns `title` and `currency` — a new, incompatible contract, which per the lesson requires a new major version, not a "bugfix" of v1. The identical route `api/v{version:apiVersion}/products` does not conflict, because the `{version:apiVersion}` segment is a route constraint the library uses to pick the controller; manually duplicating the literal `api/v1/products` in both classes would raise `AmbiguousMatchException`. `public partial class Program { }` makes the `Program` class available to `WebApplicationFactory<Program>` in tests — the .NET 6+ idiom for integration testing minimal API / top-level programs.

#### Going deeper (bonus)
1. Add header-based versioning for the `GET /api/orders` endpoint: a single controller with two attributes `[ApiVersion("1.0")]` and `[ApiVersion("2.0")]`, where the version is selected by the `X-Api-Version` header. Use `HeaderApiVersionReader("X-Api-Version")` combined with the existing readers.
2. Implement a custom `IAppliesToApiController` or an action filter that automatically sets the `Sunset` and `Deprecation` headers for any deprecated version, so you do not duplicate the code in every v1 controller.
3. Cover each version's contract with schema tests (for example via JSON Schema or `Shouldly` + `JToken`), so that any change to the `name`/`title`/`currency` fields fails in CI.
4. Simulate the migration window: make v1 return a `migrationHint: "use v2; rename name → title, add currency"` field in the response body, and write a test asserting that this field is present only in v1.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Проект `ProductsApi` на .NET 8 с top-level statements создан.
- [ ] (RU) Пакеты `Asp.Versioning.Mvc` и `Asp.Versioning.Mvc.ApiExplorer` установлены.
- [ ] (RU) `AddApiVersioning` вызван до `AddControllers`, настроены `DefaultApiVersion`, `AssumeDefaultVersionWhenUnspecified`, `ReportApiVersions`.
- [ ] (RU) `ApiVersionReader` комбинирует URL-сегмент и query string.
- [ ] (RU) `ProductsV1Controller` помечен `Deprecated = true` и возвращает `name`.
- [ ] (RU) `ProductsV2Controller` возвращает `title` и `currency`.
- [ ] (RU) V1 отдаёт заголовки `Sunset` и `Deprecation: true`.
- [ ] (RU) Тесты xUnit (3 сценария) зелёные.
- [ ] (RU) В `README.md` зафиксированы стиль, дата вывода v1 и путь миграции.
- [ ] (EN) A `ProductsApi` project on .NET 8 with top-level statements is created.
- [ ] (EN) Packages `Asp.Versioning.Mvc` and `Asp.Versioning.Mvc.ApiExplorer` are installed.
- [ ] (EN) `AddApiVersioning` is called before `AddControllers`; `DefaultApiVersion`, `AssumeDefaultVersionWhenUnspecified`, `ReportApiVersions` are set.
- [ ] (EN) `ApiVersionReader` combines URL segment and query string.
- [ ] (EN) `ProductsV1Controller` is marked `Deprecated = true` and returns `name`.
- [ ] (EN) `ProductsV2Controller` returns `title` and `currency`.
- [ ] (EN) V1 emits `Sunset` and `Deprecation: true` headers.
- [ ] (EN) xUnit tests (3 scenarios) are green.
- [ ] (EN) `README.md` records the style, v1 retirement date, and migration path.

#### Ресурсы / Resources
- [Microsoft Learn — ASP.NET Core web API advanced topics](https://learn.microsoft.com/aspnet/core/web-api/advanced/)
- [Asp.Versioning on GitHub](https://github.com/dotnet/aspnet-api-versioning)
- [RFC 8594 — The Sunset HTTP Header Field](https://datatracker.ietf.org/doc/html/rfc8594)
- [RFC 9745 — The Deprecation HTTP Header Field](https://datatracker.ietf.org/doc/html/rfc9745)
- [Microsoft Learn — Versioning ASP.NET Core web APIs](https://learn.microsoft.com/aspnet/core/web-api/advanced/versioning)
