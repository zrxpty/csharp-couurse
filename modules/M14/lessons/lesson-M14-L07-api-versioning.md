[⬆ К модулю M14](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M14-L07: Версионирование API / API versioning

**Модуль / Module:** M14
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Версионирование API — это механизм, который позволяет изменять контракт вашего веб-API, не ломая уже работающих клиентов. Представьте городской автобусный маршрут: если вы внезапно поменяете номер маршрута и его остановки, пассажиры, привыкшие к старому расписанию, не доедут до работы. Версионирование даёт вам возможность запустить «новый маршрут» (v2), пока «старый маршрут» (v1) ещё некоторое время ездит параллельно, давая клиентам время спокойно перейти.

Зачем это нужно? Любое публичное API рано или поздно сталкивается с противоречием: бизнес хочет новых полей, новых эндпоинтов и изменений поведения, а мобильные приложения, интеграции партнёров и старые фронтенды уже стоят у пользователей на устройствах и не могут обновиться мгновенно. Без версионирования любое изменение становится хрупким: вы либо замораживаете развитие, либо ломаете совместимость. С версионированием вы явно объявляете: «это v1, а это v2», и клиенты сами решают, когда мигрировать.

Существует три классических способа передать версию клиентом.

**URL versioning** — версия вшита в путь: `/api/v1/products`, `/api/v2/products`. Это самый наглядный способ: версия видна в логах, в браузере, в документации, в кэше CDN. Его легко понять и новым разработчикам, и автоматическим инструментам. Минус — путь фактически становится частью контракта, и смена схемы URL требует маршрутизации на уровне приложения.

**Query string versioning** — версия передаётся как параметр: `/api/products?api-version=2.0`. Удобно, когда не хочется менять структуру URL, и легко добавить поверх существующего API. Минус — версия не видна в «чистом» пути, может теряться при кэшировании, а старые клиенты, формирующие URL вручную, могут про неё забыть.

**Header versioning** — версия передаётся в HTTP-заголовке, например `X-Api-Version: 2.0` или через медиа-тип `Accept: application/json;v=2`. Самый «чистый» с точки зрения REST: URL остаётся стабильным, версия — это метаданные запроса. Минус — плохо видно при ручном тестировании в браузере, сложнее документировать и объяснить новичкам.

В экосистеме .NET de-facto стандартом стала библиотека **Asp.Versioning** (ранее Microsoft.AspNetCore.Mvc.Versioning). Она добавляет единый конвейер: вы декларируете версию атрибутом `[ApiVersion("2.0")]`, настраиваете, как клиент сообщает версию (путь, query, заголовок или их комбинация), и фреймворк сам маршрутизирует запрос к нужному контроллеру. Библиотека также умеет сообщать клиентам об устаревших версиях: вы помечаете версию как deprecated, и в ответе появляется заголовок `Sunset` и/или `Deprecation`, а в OpenAPI появляется соответствующая пометка.

**Deprecation** (устаревание) — это мягкий вывод версии из эксплуатации. Вместо того чтобы резко удалить v1, вы помечаете её устаревшей, оставляете работающей, даёте клиентам окно миграции (например, 6 месяцев), параллельно публикуете v2, и только потом удаляете v1. Это уважение к потребителям API и страховка от падения продакшена. Хорошее правило: никогда не удаляйте версию без объявленного периода deprecation и задокументированного пути миграции.

На практике чаще всего выбирают URL versioning для публичных API (максимальная наглядность) и комбинируют его с query string как запасной канал. Header versioning уместен для внутренних API, где команда контролирует и клиента, и сервера. Какой бы способ вы ни выбрали, главное — последовательность: один стиль во всём API, явная документация и предсказуемый жизненный цикл версий.

#### Theory (EN)

API versioning is the mechanism that lets you evolve your web API contract without breaking clients that are already in production. Imagine a city bus route: if you suddenly change the route number and its stops, the passengers who memorised the old timetable will not get to work. Versioning lets you launch a "new route" (v2) while the "old route" (v1) keeps running in parallel for a while, giving clients time to migrate at their own pace.

Why does it matter? Every public API eventually hits a contradiction: the business wants new fields, new endpoints and behaviour changes, while mobile apps, partner integrations and legacy front-ends are already sitting on users' devices and cannot be updated instantly. Without versioning, every change becomes fragile: you either freeze evolution or break compatibility. With versioning, you declare explicitly: "this is v1, that is v2", and clients decide when to migrate.

There are three classic ways for a client to communicate the version it wants.

**URL versioning** embeds the version in the path: `/api/v1/products`, `/api/v2/products`. This is the most visible approach: the version shows up in logs, in the browser, in documentation and in CDN caches. It is easy for new developers and tooling to understand. The downside is that the path effectively becomes part of the contract, and changing the URL scheme requires routing support at the application level.

**Query string versioning** passes the version as a parameter: `/api/products?api-version=2.0`. Convenient when you do not want to change the URL structure, and easy to add on top of an existing API. The downside is that the version is not visible in the "clean" path, it can get lost during caching, and legacy clients that build URLs by hand may forget it.

**Header versioning** sends the version in an HTTP header, for example `X-Api-Version: 2.0` or through a media type `Accept: application/json;v=2`. The most "REST-pure" option: the URL stays stable and the version is request metadata. The downside is that it is hard to see when manually testing in a browser, and harder to document and explain to newcomers.

In the .NET ecosystem the de-facto standard is the **Asp.Versioning** library (formerly Microsoft.AspNetCore.Mvc.Versioning). It adds a unified pipeline: you declare a version with the `[ApiVersion("2.0")]` attribute, configure how the client reports the version (path, query, header, or a combination), and the framework routes the request to the right controller automatically. The library can also tell clients that a version is going away: you mark a version as deprecated, and the response carries a `Sunset` and/or `Deprecation` header, while OpenAPI reflects the deprecation in the docs.

**Deprecation** is the gentle retirement of a version. Instead of abruptly deleting v1, you mark it deprecated, keep it working, give clients a migration window (say, six months), publish v2 in parallel, and only then remove v1. This is respect for your API consumers and a safety net for production. A good rule: never delete a version without an announced deprecation period and a documented migration path.

In practice, most teams pick URL versioning for public APIs (maximum visibility) and combine it with query string as a fallback channel. Header versioning fits internal APIs where one team controls both client and server. Whichever you choose, the key is consistency: one style across the whole API, explicit documentation, and a predictable version lifecycle.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — Полный пример версионирования API через Asp.Versioning
// C# 12 / .NET 8 — Full API versioning example using Asp.Versioning
//
// Пакеты / Packages:
//   Asp.Versioning.Mvc
//   Asp.Versioning.Mvc.ApiExplorer
//
// Program.cs — регистрация версионирования / registration of versioning

using Asp.Versioning;
using Asp.Versioning.ApiExplorer;
using Microsoft.AspNetCore.Mvc;

var builder = WebApplication.CreateBuilder(args);

// Регистрируем API-версионирование / Register API versioning
// Здесь: версия в URL по умолчанию + запасной канал через query string.
// Here: version in URL by default + a fallback channel via query string.
builder.Services.AddApiVersioning(options =>
{
    // Версия по умолчанию, если клиент её не указал
    // Default version assumed when the client omits it
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;

    // Разрешаем читать версию из query string (?api-version=2.0)
    // Allow reading the version from the query string
    options.ApiVersionReader = ApiVersionReader.Combine(
        new UrlSegmentApiVersionReader(),   // /v1/, /v2/ в пути
        new QueryStringApiVersionReader("api-version"));

    // Сообщаем клиенту о поддерживаемых версиях в заголовке ответа
    // Report supported versions in a response header
    options.ReportApiVersions = true;
})
.AddApiExplorer(options =>
{
    // Группируем документы OpenAPI по версии
    // Group OpenAPI documents by version
    options.GroupNameFormat = "'v'VVV";
    options.SubstituteApiVersionInUrl = true;
});

builder.Services.AddControllers();

var app = builder.Build();

app.UseRouting();
app.MapControllers();

var apiDescription = app.Services.GetRequiredService<IApiVersionDescriptionProvider>();
// apiDescription.ApiVersionDescriptions можно использовать для генерации Swagger-документов
// apiDescription.ApiVersionDescriptions can be used to generate Swagger docs per version

app.Run();

// ---- Контроллеры / Controllers ----

// Версия 1 — устаревшая, но всё ещё работающая
// Version 1 — deprecated but still operational
[ApiController]
[ApiVersion("1.0", Deprecated = true)]
[Route("api/v{version:apiVersion}/products")]
public class ProductsV1Controller : ControllerBase
{
    [HttpGet]
    public IActionResult GetAll() =>
        // Старая форма ответа: поле "name"
        // Legacy response shape: a "name" field
        Ok(new[] { new { id = 1, name = "Widget" } });
}

// Версия 2 — актуальная, изменили контракт: "title" вместо "name"
// Version 2 — current, contract changed: "title" instead of "name"
[ApiController]
[ApiVersion("2.0")]
[Route("api/v{version:apiVersion}/products")]
public class ProductsV2Controller : ControllerBase
{
    [HttpGet]
    public IActionResult GetAll() =>
        // Новая форма ответа + поле currency
        // New response shape plus a currency field
        Ok(new[] { new { id = 1, title = "Widget", currency = "USD" } });
}

// Версионирование через заголовок (без сегмента /v1/ в пути)
// Header-based versioning (no /v1/ segment in the path)
[ApiController]
[ApiVersion("1.0")]
[ApiVersion("2.0")]
[Route("api/orders")]
public class OrdersController : ControllerBase
{
    // Версия определяется заголовком X-Api-Version
    // Version is selected by the X-Api-Version header
    [HttpGet]
    public IActionResult Get([FromServices] IApiVersioningInfo info) =>
        Ok(new
        {
            requestedVersion = info.RequestedApiVersion?.ToString(),
            current = "data"
        });
}

// Расширение для доступа к информации о версии запроса
// Helper to access the requested API version
public interface IApiVersioningInfo
{
    ApiVersion? RequestedApiVersion { get; }
}
```

#### Best Practices

- Выберите один основной способ версионирования и применяйте его последовательно во всём API. / Pick one primary versioning style and apply it consistently across the entire API.
- Всегда объявляйте окно deprecation перед удалением версии и документируйте путь миграции. / Always announce a deprecation window before removing a version and document the migration path.
- Используйте `ReportApiVersions = true`, чтобы клиенты видели поддерживаемые версии в заголовках ответа. / Use `ReportApiVersions = true` so clients can see supported versions in response headers.
- Нарастевайте мажорную версию только при несовместимых изменениях контракта; минорные изменения держите в той же версии. / Bump the major version only for breaking contract changes; keep additive, non-breaking changes in the same version.
- Генерируйте отдельный OpenAPI-документ на каждую версию, чтобы документация точно отражала контракт. / Generate a separate OpenAPI document per version so docs reflect the exact contract.
- Держите старую версию в работе достаточно долго — мобильные клиенты обновляются медленно. / Keep the old version alive long enough — mobile clients update slowly.
- Покрывайте каждую версию контрактными тестами, чтобы случайно не сломать v1 при работе над v2. / Cover each version with contract tests so v1 is not accidentally broken while working on v2.

#### Частые ошибки / Common Mistakes

- Удаление v1 без объявления deprecation → всегда помечайте версию `Deprecated = true` и публикуйте дату вывода из эксплуатации. (RU)
- Разные стили версионирования в разных модулях API → выберите один стиль в `AddApiVersioning` и применяйте его везде. (RU)
- Смена поведения v1 под предлогом «это багфикс» → любые несовместимые изменения — это новая мажорная версия. (RU)
- Игнорирование заголовка `Sunset` клиентами → документируйте, что заголовок означает, и мониторьте его в клиентском коде. (RU)
- Отсутствие OpenAPI-документов на каждую версию → используйте `IApiVersionDescriptionProvider` для генерации Swagger-групп. (RU)
- Deleting v1 without announcing deprecation → always mark the version `Deprecated = true` and publish a retirement date. (EN)
- Mixing versioning styles across API modules → pick one style in `AddApiVersioning` and apply it everywhere. (EN)
- Changing v1 behaviour under the excuse of "just a bugfix" → any breaking change is a new major version. (EN)
- Clients ignoring the `Sunset` header → document what the header means and monitor it in client code. (EN)
- Missing per-version OpenAPI documents → use `IApiVersionDescriptionProvider` to generate Swagger groups. (EN)

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Выбран один основной способ версионирования (URL / query / header) и задан в `AddApiVersioning`. (RU)
- [ ] Указана версия по умолчанию через `DefaultApiVersion` и `AssumeDefaultVersionWhenUnspecified`. (RU)
- [ ] Включён `ReportApiVersions = true` для передачи поддерживаемых версий в заголовке ответа. (RU)
- [ ] Старые версии помечены `Deprecated = true`, объявлено окно миграции. (RU)
- [ ] На каждую версию генерируется отдельный OpenAPI-документ через `IApiVersionDescriptionProvider`. (RU)
- [ ] Контракт каждой версии покрыт тестами, v1 не ломается при работе над v2. (RU)
- [ ] Документация явно описывает, как клиенту указывать версию и как читать заголовок `Sunset`. (RU)
- [ ] One primary versioning style (URL / query / header) is chosen and configured in `AddApiVersioning`. (EN)
- [ ] A default version is set via `DefaultApiVersion` and `AssumeDefaultVersionWhenUnspecified`. (EN)
- [ ] `ReportApiVersions = true` is enabled so supported versions are returned in a response header. (EN)
- [ ] Old versions are marked `Deprecated = true` with an announced migration window. (EN)
- [ ] A separate OpenAPI document is generated per version via `IApiVersionDescriptionProvider`. (EN)
- [ ] Each version's contract is covered by tests; v1 is not broken while working on v2. (EN)
- [ ] Documentation explicitly states how a client should send the version and read the `Sunset` header. (EN)

#### Ресурсы / Resources

- [Microsoft Learn — ASP.NET Core web API advanced topics](https://learn.microsoft.com/aspnet/core/web-api/advanced/)
- [Asp.Versioning на GitHub / Asp.Versioning on GitHub](https://github.com/dotnet/aspnet-api-versioning)
- [RFC 8594 — The Sunset HTTP Header Field](https://datatracker.ietf.org/doc/html/rfc8594)
- [RFC 9745 — The Deprecation HTTP Header Field](https://datatracker.ietf.org/doc/html/rfc9745)

---

[⬆ К модулю M14](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
