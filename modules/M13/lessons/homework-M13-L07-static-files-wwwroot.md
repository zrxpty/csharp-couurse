---
[← К уроку M13-L07](lesson-M13-L07-static-files-wwwroot.md) | [⬆ К модулю M13](../README.md) | [Следующее ДЗ →](homework-M13-L08-mvc-razor-overview.md)
---

### Домашнее задание M13-L07: Статические файлы, wwwroot / Homework M13-L07: Static files, wwwroot

**Урок / Lesson:** M13-L07
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) Научиться подключать и настраивать раздачу статических файлов из `wwwroot` в ASP.NET Core 8, корректно упорядочивать middleware (`UseDefaultFiles`, `UseStaticFiles`, `UseResponseCompression`, `UseHttpsRedirection`, `MapFallbackToFile`), управлять кэшированием через `Cache-Control` и `OnPrepareResponse`, применять версионирование (fingerprinting) для безопасного длинного кэша, отличать публичную статику от защищённых ресурсов и избегать типичных ошибок порядка middleware и утечки секретов. (EN) Learn how to enable and configure static file delivery from `wwwroot` in ASP.NET Core 8, order middleware correctly (`UseDefaultFiles`, `UseStaticFiles`, `UseResponseCompression`, `UseHttpsRedirection`, `MapFallbackToFile`), drive caching through `Cache-Control` and `OnPrepareResponse`, apply fingerprinting for safe long cache, separate public static assets from protected resources, and avoid common ordering mistakes and secret leakage.

#### Связь с уроком / Connection to the lesson
(RU) Урок объясняет концепцию `wwwroot` как «витрины» готовых ресурсов, порядок middleware в конвейере ASP.NET Core, работу `UseDefaultFiles`/`UseStaticFiles`/`UseFileServer`, кэш-заголовки и сжатие через Brotli/Gzip. Данное задание закрепляет все эти темы: вы соберёте мини-приложение с публичной статикой, защищённым эндпоинтом, кэшем и сжатием, повторив лучшие практики и наступив на частые грабли в контролируемой среде.
(EN) The lesson explains `wwwroot` as a "display case" of ready resources, the middleware order in the ASP.NET Core pipeline, the roles of `UseDefaultFiles`/`UseStaticFiles`/`UseFileServer`, cache headers and compression via Brotli/Gzip. This homework cements all of those topics: you will build a mini app with public static assets, a protected endpoint, cache and compression, replaying best practices and tripping the common pitfalls in a controlled environment.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Представьте, что вы присоединились к небольшой команде, которая разворачивает корпоративный «лендинг» для внутреннего продукта компании. Дизайн уже готов: есть несколько HTML-страниц, набор CSS-стилей, минимальный JavaScript для переключения темы и пара SVG-иконок. Команда хочет, чтобы эти ресурсы отдавались максимально быстро, корректно кэшировались в браузерах сотрудников и сжимались на лету. При этом часть файлов (например, отчёты в PDF) должна быть доступна только аутентифицированным пользователям, а часть эндпоинтов (API состояния) должна работать параллельно со статикой и не «поглощаться» ею.

ASP.NET Core предлагает для этого полноценный конвейер middleware: `UseStaticFiles`, `UseDefaultFiles`, `UseResponseCompression`, `UseHttpsRedirection`, `UseHsts` и `MapFallbackToFile`. Однако именно здесь разработчики чаще всего ошибаются в порядке вызовов: ставят `UseDefaultFiles` после `UseStaticFiles` и получают 404 на корневом URL; включают сжатие после статики и не получают сжатия вовсе; выставляют длинный `max-age` без версионирования и ломают обновление стилей у пользователей. Кроме того, острый вопрос — безопасность: в `wwwroot` по ошибке попадают `appsettings.json`, ключи и `.env`, что превращает «витрину» в дыру.

В этом задании вы соберёте проект с нуля, повторите все шаги из урока, столкнётесь с типичными ошибками и исправите их, а затем реализуете тонкую настройку кэша с регулярным распознаванием fingerprint-файлов. В результате вы получите рабочее приложение, корректно отдающее статику, защищённое от утечек и оптимизированное по скорости и трафику — готовый шаблон, который можно переносить в реальные проекты.

#### Что нужно сделать (пошагово)

1. Создайте новый пустой web-проект .NET 8: `dotnet new web -n StaticFilesDemo -o StaticFilesDemo`, перейдите в каталог `cd StaticFilesDemo` и откройте `Program.cs`. Шаблон `web` минимален и не содержит MVC/Razor, что идеально для упражнений со статикой.
2. Создайте структуру папок `wwwroot` внутри проекта командой `mkdir wwwroot` (или через проводник), затем вложенные `wwwroot/css`, `wwwroot/js`, `wwwroot/img`. В каждую папку положите по одному файлу: `wwwroot/index.html`, `wwwroot/css/site.1a2b3c4d.css`, `wwwroot/js/app.js`, `wwwroot/img/logo.svg`. Файл `site.1a2b3c4d.css` намеренно имеет хеш в имени — он будет нужен для проверки fingerprinting-логики кэша.
3. Заполните `wwwroot/index.html` минимальной разметкой: `<!DOCTYPE html>`, заголовок `<title>Static Files Demo</title>`, подключение CSS через `<link rel="stylesheet" href="/css/site.1a2b3c4d.css">` и JS через `<script src="/js/app.js"></script>`. В `site.1a2b3c4d.css` добавьте несколько правил (например, `body { font-family: sans-serif; }`), в `app.js` — `console.log("hello from wwwroot");`, в `logo.svg` — простейший SVG.
4. Зарегистрируйте в `Program.cs` сжатие ответов через `builder.Services.AddResponseCompression(...)` с поставщиками `BrotliCompressionProvider` и `GzipCompressionProvider`, включите `EnableForHttps = true` и задайте список MIME-типов: `text/plain`, `text/css`, `text/html`, `application/javascript`, `application/json`, `image/svg+xml`, `application/wasm`. Это точно повторяет пример из урока.
5. В middleware-секции строго соблюдайте порядок: сначала `app.UseResponseCompression()`, затем в неразвивающей среде `app.UseHttpsRedirection()` и `app.UseHsts()`, затем `app.UseDefaultFiles()`, затем `app.UseStaticFiles(new StaticFileOptions { OnPrepareResponse = ... })`, и только потом `app.UseRouting()` и `app.UseAuthorization()`. Любая перестановка — баг.
6. В `OnPrepareResponse` реализуйте регулярное распознавание версионированных файлов: путь вида `/css/site.1a2b3c4d.css` должен получать `Cache-Control: public, max-age=31536000, immutable`, а все прочие пути — `public, max-age=300`. Используйте regex `@"\.[a-f0-9]{8,}\.(css|js)$"` как в уроке.
7. Добавьте защищённый minimal API эндпоинт `app.MapGet("/api/health", ...)`, возвращающий JSON `new { status = "ok", ts = DateTime.UtcNow }`. Этот эндпоинт не должен перекрываться статикой: убедитесь, что `/api/health` действительно отвечает, а `/css/site.1a2b3c4d.css` — отдаёт CSS.
8. Добавьте `app.MapFallbackToFile("index.html")` в конец — это SPA-фоллбэк: любой неизвестный маршрут должен вернуть `index.html`. Проверьте, что `/nonexistent-route` отдаёт именно HTML главной страницы, а не 404.
9. Запустите приложение `dotnet run`, откройте браузер на `https://localhost:<port>/` и проверьте: главная страница открывается (значит, `UseDefaultFiles` сработал), CSS подключается (200), JS подключается (200), SVG-логотип виден, `/api/health` возвращает JSON, `/nonexistent-route` отдаёт `index.html`.
10. Откройте DevTools → Network и проверьте заголовки: для `site.1a2b3c4d.css` должен быть `cache-control: public, max-age=31536000, immutable`, а для `app.js` — `public, max-age=300`. Включите фильтр «Response Headers» и убедитесь, что для CSS присутствует `content-encoding: br` (Brotli), если браузер поддерживает.
11. Намеренно сломайте порядок: временно переставьте `app.UseStaticFiles()` **до** `app.UseDefaultFiles()`, перезапустите и откройте `/` — вы должны получить 404, потому что `UseDefaultFiles` не успел переписать URL. Верните правильный порядок.
12. Намеренно поместите копию `appsettings.json` внутрь `wwwroot/appsettings.json`, запустите приложение и попробуйте открыть `/appsettings.json` в браузере — вы увидите содержимое конфига в открытом виде. Удалите файл и сделайте вывод о безопасности `wwwroot`.

#### Требования к решению

Решение должно быть оформлено как проект .NET 8 (target framework `net8.0`) с top-level statements в `Program.cs`, без класса `Startup`. Все middleware должны вызываться ровно в порядке, заданном уроком: сжатие → HTTPS/HSTS → default files → static files → routing → authorization → endpoints → fallback. Код должен использовать C# 12 (collection expressions, например `MimeTypes = [ "text/css", "text/html", ... ]` уместны, raw-string-literal для regex необязателен, но приветствуется). Кэш-логика должна корректно различать версионированные и неверсионированные файлы по regex. Должен присутствовать хотя бы один публичный эндпоинт (`/api/health`) и один SPA-фоллбэк (`MapFallbackToFile("index.html")`).

Структура файлов должна включать как минимум `wwwroot/index.html`, `wwwroot/css/site.1a2b3c4d.css` (с fingerprint), `wwwroot/js/app.js`, `wwwroot/img/logo.svg`. В репозитории не должно быть секретов внутри `wwwroot` — это проверяется отдельным шагом. Решение должно запускаться командой `dotnet run` без ошибок и отвечать на все описанные URL с ожидаемыми статус-кодами и заголовками. Запрещено использовать `UseFileServer` с включённым `UseDirectoryBrowser` в продакшен-секции. Код должен быть прокомментирован на двух языках (RU + EN), как в уроке.

#### Тонкости и подводные камни

Главная тонкость, на которой спотыкаются новички — `UseDefaultFiles` **не отдаёт** файл, а только переписывает URL. Если поставить его после `UseStaticFiles`, то к моменту, когда `UseStaticFiles` ищет файл по пути `/`, никакого файла с именем `/` (без `index.html`) нет — и клиент получает 404. Поэтому `UseDefaultFiles` всегда первым. Вторая тонкость — `UseResponseCompression` должен стоять **до** `UseStaticFiles`, потому что middleware оборачивает `Response.Body` при входе запроса в конвейер; если статику уже отдали и закрыли поток, сжать нечего. Третья — длинный `max-age` без fingerprinting опасен: браузер закэширует CSS на год и не увидит обновление. Поэтому в `OnPrepareResponse` регуляркой проверяют, есть ли в имени 8+ шестнадцатеричных символов перед расширением; если есть — ставим `immutable`, иначе короткий TTL.

Четвёртая тонкость — `wwwroot` публично доступен **целиком**. Любой файл внутри него, включая `appsettings.json`, `.env`, `web.config`, ключи и резервные копии, будет отдан по прямому URL. Это не «ошибка» ASP.NET Core, а его дизайн: «витрина» должна быть витриной. Поэтому конфиги и секреты всегда живут вне `wwwroot`, а защищённые файлы отдаются через контроллер с `[Authorize]`. Пятая тонкость — `MapFallbackToFile` должен быть последним: он перехватывает любой маршрут, не сопоставленный с эндпоинтом, и возвращает указанный файл; если поставить его раньше, он «съест» ваши API-маршруты. Шестая — `EnableForHttps = true` у сжатия: по умолчанию сжатие отключено для HTTPS из-за атак BREACH, но для статики (не динамических ответов с секретами) оно безопасно. Седьмая — `UseHsts` нельзя включать в Development, иначе браузер «запомнит» HSTS на localhost и разработка станет болезненной; поэтому HSTS всегда под условием `!IsDevelopment()`.

#### Критерии приёмки

- [ ] Создан проект `StaticFilesDemo` на .NET 8 (`net8.0`) с top-level statements.
- [ ] В `Program.cs` зарегистрировано `AddResponseCompression` с Brotli и Gzip.
- [ ] `MimeTypes` содержит минимум 6 текстовых типов, включая `image/svg+xml`.
- [ ] `EnableForHttps = true` выставлен у сжатия.
- [ ] Порядок middleware строго: `UseResponseCompression` → (`UseHttpsRedirection` + `UseHsts` вне Dev) → `UseDefaultFiles` → `UseStaticFiles` → `UseRouting` → `UseAuthorization` → endpoints → `MapFallbackToFile`.
- [ ] В `OnPrepareResponse` реализовано regex-распознавание fingerprint с корректными `Cache-Control` для версионированных и неверсионированных файлов.
- [ ] Версионированный файл получает `max-age=31536000, immutable`, прочие — `max-age=300`.
- [ ] `/` отдаёт `index.html` (значит, `UseDefaultFiles` сработал).
- [ ] `/css/site.1a2b3c4d.css` отдаёт 200 и CSS-содержимое.
- [ ] `/api/health` отдаёт JSON и **не** перехватывается статикой или фоллбэком.
- [ ] `/nonexistent-route` отдаёт `index.html` через `MapFallbackToFile`.
- [ ] В DevTools виден `content-encoding: br` (или `gzip`) для CSS.
- [ ] `wwwroot` не содержит секретов и `appsettings.json`.
- [ ] Шаг с намеренной поломкой порядка задокументирован скриншотом/выводом 404.
- [ ] Шаг с `wwwroot/appsettings.json` задокументирован и файл удалён.
- [ ] Код прокомментирован на двух языках (RU + EN).

#### Подсказки (без прямого ответа)

- Вспомните ресторанную аналогию из урока: «витрина» (`wwwroot`) и «кухня» (контроллеры). Что должно быть доступно сразу, а что — под заказ?
- Подумайте, в какой момент конвейера `Response.Body` уже обёрнут сжимающим потоком. Где должна стоять ститика относительно сжатия?
- `UseDefaultFiles` «переписывает URL» — что это значит технически? Почему файл не отдаётся?
- Если `max-age=31536000` без `immutable` — что произойдёт при обновлении файла с тем же именем?
- Где физически должен лежать `appsettings.json`, чтобы его нельзя было запросить по URL?
- Почему `MapFallbackToFile` должен идти **после** `MapGet("/api/health", ...)`?
- Можно ли использовать `UseFileServer` вместо трёх отдельных middleware? Когда это уместно, а когда — нет?

#### Эталонное решение (разбор)

```csharp
// Program.cs — C# 12 / .NET 8
// Полная настройка статики, default files, кэша, сжатия, фоллбэка
// Full setup: static files, default files, cache, compression, fallback

using Microsoft.AspNetCore.StaticFiles;
using System.Text.RegularExpressions;

var builder = WebApplication.CreateBuilder(args);

// 1) Регистрация сжатия ответов / Register response compression
// Brotli — лучший для текста, Gzip — fallback для старых клиентов
// Brotli is best for text, Gzip is a fallback for older clients
builder.Services.AddResponseCompression(options =>
{
    options.EnableForHttps = true; // Сжимать и по HTTPS / Compress over HTTPS too
    options.Providers.Add<BrotliCompressionProvider>();
    options.Providers.Add<GzipCompressionProvider>();
    // Collection expression C# 12 / C# 12 collection expression
    options.MimeTypes =
    [
        "text/plain",
        "text/css",
        "text/html",
        "application/javascript",
        "application/json",
        "image/svg+xml",
        "application/wasm"
    ];
});

var app = builder.Build();

// 2) Сжатие — ДО статики, чтобы обернуть ответ / Compression BEFORE static files
app.UseResponseCompression();

// 3) HTTPS-редирект и HSTS только в продакшене / HTTPS redirect + HSTS only in production
if (!app.Environment.IsDevelopment())
{
    app.UseHttpsRedirection();
    app.UseHsts();
}

// 4) Default files — ДО StaticFiles (переписывает URL на index.html)
// Default files BEFORE StaticFiles (rewrites URL to index.html)
app.UseDefaultFiles();

// 5) Статические файлы с кэш-заголовками / Static files with cache headers
app.UseStaticFiles(new StaticFileOptions
{
    OnPrepareResponse = ctx =>
    {
        var path = ctx.Context.Request.Path.Value ?? string.Empty;
        // Regex ищет fingerprint: 8+ hex-символов перед .css/.js
        // Regex looks for a fingerprint: 8+ hex chars before .css/.js
        var hasFingerprint = Regex.IsMatch(path, @"\.[a-f0-9]{8,}\.(css|js)$");
        ctx.Context.Response.Headers.CacheControl = hasFingerprint
            ? "public, max-age=31536000, immutable"
            : "public, max-age=300";
    }
});

app.UseRouting();
app.UseAuthorization();

// 6) Публичный эндпоинт, который НЕ перекрывается статикой / Endpoint not shadowed by static files
app.MapGet("/api/health", () => Results.Ok(new { status = "ok", ts = DateTime.UtcNow }));

// 7) SPA-фоллбэк на index.html — последний шаг / SPA fallback to index.html — last step
app.MapFallbackToFile("index.html");

app.Run();
```

Разбор по строкам. Строка `using Microsoft.AspNetCore.StaticFiles;` подключает тип `StaticFileOptions`; `using System.Text.RegularExpressions;` — тип `Regex` для распознавания fingerprint. `WebApplication.CreateBuilder(args)` — стандартная точка входа .NET 8 с top-level statements. Блок `AddResponseCompression` регистрирует сервис сжатия в DI-контейнере: `EnableForHttps = true` включает сжатие для HTTPS-ответов (по умолчанию оно выключено из-за риска BREACH, но для статики безопасно). Порядок провайдеров важен: Brotli предпочтительнее, Gzip остаётся запасным для старых клиентов. `MimeTypes` задан коллекционным выражением C# 12 — это компактная форма `new[] { ... }`.

Далее идёт критически важный порядок middleware. `app.UseResponseCompression()` ставится **первым** в pipeline, чтобы обернуть `Response.Body` сжимающим потоком до того, как статика начнёт в него писать. Условие `!IsDevelopment()` защищает разработчика от HSTS на localhost — браузер запомнил бы HSTS и постоянно редиректил на HTTPS, ломая локальную отладку. `UseDefaultFiles()` идёт **до** `UseStaticFiles()`: он не отдаёт файл, а переписывает URL `/` на `/index.html`, после чего `UseStaticFiles` находит и отдаёт его; если переставить — 404.

`UseStaticFiles(new StaticFileOptions { OnPrepareResponse = ... })` — точка тонкой настройки кэша. `OnPrepareResponse` — колбэк, выполняемый перед отправкой каждого файла; здесь мы читаем `Request.Path` и regex'ом проверяем, есть ли в имени хеш. Паттерн `\.[a-f0-9]{8,}\.(css|js)$` соответствует именам вида `site.1a2b3c4d.css`: точка, 8+ шестнадцатеричных символов, точка, расширение. Если совпало — `immutable` на год (браузер даже не будет делать условный запрос); иначе — 5 минут. `app.UseRouting()` и `app.UseAuthorization()` идут после статики, потому что маршрутизация нужна только для эндпоинтов, а статику мы уже обработали. `app.MapGet("/api/health", ...)` — minimal API, который не перекрыт статикой (нет файла `wwwroot/api/health`) и не «съедается» фоллбэком, потому что `MapFallbackToFile` работает только для unmatched-маршрутов. `app.MapFallbackToFile("index.html")` — последний шаг, SPA-фоллбэк: любой неизвестный маршрут возвращает главную страницу, чтобы клиентский роутер мог взять управление.

#### Задания на углубление (бонус)

1. Реализуйте отдельную папку `wwwroot/private` с PDF-документом, но сделайте так, чтобы она **не** отдавалась напрямую через `UseStaticFiles`. Вместо этого создайте контроллер `ReportsController` с действием `[Authorize]` и `PhysicalFile(...)` для отдачи PDF после проверки прав. Сравните с публичной статикой.
2. Добавьте версионирование через сборку: напишите MSBuild-target, который на `dotnet publish` переименовывает `site.css` в `site.<hash>.css` и подменяет ссылки в `index.html`. Проверьте, что regex-логика кэша корректно распознаёт новое имя.
3. Включите `UseFileServer` вместо трёх middleware в отдельной ветке и измерьте разницу в производительности через `dotnet-counters`. Объясните, почему для продакшена предпочитают раздельные вызовы.
4. Добавьте CDN-заголовок `Access-Control-Allow-Origin` для шрифтов из `wwwroot/fonts` через `OnPrepareResponse`, реализовав логику «если путь начинается с `/fonts/` — добавляем CORS».

---

## Statement in English / Постановка на английском

#### Context & motivation

Imagine you have joined a small team that is launching an internal corporate landing page for a company product. The design is already finished: several HTML pages, a set of CSS stylesheets, a tiny JavaScript snippet to toggle dark mode, and a couple of SVG icons. The team wants these assets to be served as fast as possible, cached correctly inside employee browsers, and compressed on the fly. At the same time, a subset of files (for example, PDF reports) must be reachable only by authenticated users, and a subset of endpoints (the health API) must run alongside the static assets and must not be "swallowed" by them.

ASP.NET Core offers a complete middleware pipeline for this: `UseStaticFiles`, `UseDefaultFiles`, `UseResponseCompression`, `UseHttpsRedirection`, `UseHsts`, and `MapFallbackToFile`. This is exactly the place where developers most often get the order wrong: they put `UseDefaultFiles` after `UseStaticFiles` and get a 404 on the root URL; they enable compression after static files and get no compression at all; they set a long `max-age` without versioning and break stylesheet updates for users. A separate, sharp concern is security: `appsettings.json`, keys, and `.env` files accidentally land inside `wwwroot`, turning the "display case" into a wide open hole.

In this homework you will build a project from scratch, replay every step from the lesson, hit the typical mistakes, fix them, and then implement a fine-grained cache policy that uses a regular expression to recognise fingerprinted files. The end result is a working application that serves static assets correctly, is protected against secret leakage, and is optimised for both speed and traffic — a ready-to-reuse template you can carry into real projects.

#### What to do step by step

1. Create a fresh empty .NET 8 web project: `dotnet new web -n StaticFilesDemo -o StaticFilesDemo`, navigate into it with `cd StaticFilesDemo`, and open `Program.cs`. The `web` template is minimal and does not include MVC/Razor, which is ideal for static-files exercises.
2. Create the `wwwroot` folder structure inside the project with `mkdir wwwroot` (or via your file explorer), then the nested folders `wwwroot/css`, `wwwroot/js`, `wwwroot/img`. Put one file in each: `wwwroot/index.html`, `wwwroot/css/site.1a2b3c4d.css`, `wwwroot/js/app.js`, `wwwroot/img/logo.svg`. The file `site.1a2b3c4d.css` deliberately has a hash in its name — it will be used to verify the fingerprinting cache logic.
3. Fill `wwwroot/index.html` with minimal markup: `<!DOCTYPE html>`, a `<title>Static Files Demo</title>`, a stylesheet link `<link rel="stylesheet" href="/css/site.1a2b3c4d.css">`, and a script tag `<script src="/js/app.js"></script>`. In `site.1a2b3c4d.css` add a few rules (for instance, `body { font-family: sans-serif; }`); in `app.js` add `console.log("hello from wwwroot");`; in `logo.svg` add a minimal SVG.
4. Register response compression in `Program.cs` via `builder.Services.AddResponseCompression(...)` with the `BrotliCompressionProvider` and `GzipCompressionProvider`, set `EnableForHttps = true`, and provide a MIME-type list: `text/plain`, `text/css`, `text/html`, `application/javascript`, `application/json`, `image/svg+xml`, `application/wasm`. This mirrors the lesson exactly.
5. In the middleware section, follow the order strictly: first `app.UseResponseCompression()`, then in non-development environments `app.UseHttpsRedirection()` and `app.UseHsts()`, then `app.UseDefaultFiles()`, then `app.UseStaticFiles(new StaticFileOptions { OnPrepareResponse = ... })`, and only after that `app.UseRouting()` and `app.UseAuthorization()`. Any permutation is a bug.
6. In `OnPrepareResponse` implement a regular-expression check for versioned files: a path like `/css/site.1a2b3c4d.css` must receive `Cache-Control: public, max-age=31536000, immutable`, and every other path must receive `public, max-age=300`. Use the regex `@"\.[a-f0-9]{8,}\.(css|js)$"` as in the lesson.
7. Add a protected minimal API endpoint `app.MapGet("/api/health", ...)` returning JSON `new { status = "ok", ts = DateTime.UtcNow }`. This endpoint must not be shadowed by static files: verify that `/api/health` actually answers, and `/css/site.1a2b3c4d.css` actually serves the CSS.
8. Add `app.MapFallbackToFile("index.html")` at the end — this is the SPA fallback: any unknown route must return `index.html`. Verify that `/nonexistent-route` serves the home HTML, not a 404.
9. Run the application with `dotnet run`, open the browser at `https://localhost:<port>/`, and check: the home page opens (so `UseDefaultFiles` worked), the CSS loads (200), the JS loads (200), the SVG logo is visible, `/api/health` returns JSON, and `/nonexistent-route` returns `index.html`.
10. Open DevTools → Network and inspect the headers: for `site.1a2b3c4d.css` the response must carry `cache-control: public, max-age=31536000, immutable`, and for `app.js` it must carry `public, max-age=300`. Use the "Response Headers" filter and confirm that for the CSS you see `content-encoding: br` (Brotli) when the browser supports it.
11. Intentionally break the order: temporarily move `app.UseStaticFiles()` **before** `app.UseDefaultFiles()`, restart, and open `/` — you should get a 404 because `UseDefaultFiles` did not have a chance to rewrite the URL. Restore the correct order.
12. Intentionally place a copy of `appsettings.json` inside `wwwroot/appsettings.json`, start the application, and try to open `/appsettings.json` in the browser — you will see the configuration in plain text. Remove the file and draw a conclusion about `wwwroot` security.

#### Requirements

The solution must be a .NET 8 project (target framework `net8.0`) with top-level statements in `Program.cs`, without a `Startup` class. All middleware must be called in exactly the order prescribed by the lesson: compression → HTTPS/HSTS → default files → static files → routing → authorization → endpoints → fallback. The code should use C# 12 features (collection expressions, e.g. `MimeTypes = [ "text/css", "text/html", ... ]` are appropriate; a raw-string-literal for the regex is optional but welcome). The cache logic must correctly distinguish versioned and non-versioned files using the regex. There must be at least one public endpoint (`/api/health`) and one SPA fallback (`MapFallbackToFile("index.html")`).

The file structure must include at least `wwwroot/index.html`, `wwwroot/css/site.1a2b3c4d.css` (with a fingerprint), `wwwroot/js/app.js`, and `wwwroot/img/logo.svg`. There must be no secrets inside `wwwroot` in the repository — this is verified in a separate step. The solution must run with `dotnet run` without errors and respond to every described URL with the expected status codes and headers. Using `UseFileServer` with `UseDirectoryBrowser` enabled in the production section is forbidden. The code must be commented in both languages (RU + EN), as in the lesson.

#### Pitfalls

The main pitfall that trips up newcomers is that `UseDefaultFiles` does **not** serve a file — it only rewrites the URL. If you put it after `UseStaticFiles`, then by the time `UseStaticFiles` looks for a file at path `/`, there is no file literally named `/` (without `index.html`), and the client gets a 404. That is why `UseDefaultFiles` always goes first. The second pitfall is that `UseResponseCompression` must come **before** `UseStaticFiles`, because the middleware wraps `Response.Body` when the request enters the pipeline; once the static file has already been written and the stream closed, there is nothing left to compress. The third is that a long `max-age` without fingerprinting is dangerous: the browser caches the CSS for a year and never sees the update. That is why `OnPrepareResponse` uses a regex to check whether the file name contains eight or more hexadecimal characters before the extension; if it does, we set `immutable`, otherwise a short TTL.

The fourth pitfall is that `wwwroot` is publicly accessible **in full**. Any file inside it, including `appsettings.json`, `.env`, `web.config`, keys, and backups, will be served by a direct URL. This is not an "error" of ASP.NET Core — it is its design: a "display case" must be a display case. That is why configs and secrets always live outside `wwwroot`, and protected files are served through a controller with `[Authorize]`. The fifth pitfall is that `MapFallbackToFile` must be the last step: it intercepts any route that was not matched to an endpoint and returns the specified file; if you put it earlier, it will "eat" your API routes. The sixth is `EnableForHttps = true` on compression: by default compression is disabled for HTTPS due to BREACH attacks, but for static files (not dynamic responses carrying secrets) it is safe. The seventh is that `UseHsts` must not be enabled in Development, otherwise the browser will "remember" HSTS for localhost and development becomes painful; that is why HSTS is always guarded by `!IsDevelopment()`.

#### Acceptance criteria

- [ ] A `StaticFilesDemo` project on .NET 8 (`net8.0`) with top-level statements exists.
- [ ] `AddResponseCompression` with Brotli and Gzip is registered in `Program.cs`.
- [ ] `MimeTypes` contains at least 6 textual types, including `image/svg+xml`.
- [ ] `EnableForHttps = true` is set on compression.
- [ ] Middleware order is strict: `UseResponseCompression` → (`UseHttpsRedirection` + `UseHsts` outside Dev) → `UseDefaultFiles` → `UseStaticFiles` → `UseRouting` → `UseAuthorization` → endpoints → `MapFallbackToFile`.
- [ ] `OnPrepareResponse` implements regex-based fingerprint detection with correct `Cache-Control` for versioned and non-versioned files.
- [ ] The versioned file gets `max-age=31536000, immutable`; the others get `max-age=300`.
- [ ] `/` serves `index.html` (so `UseDefaultFiles` worked).
- [ ] `/css/site.1a2b3c4d.css` returns 200 and CSS content.
- [ ] `/api/health` returns JSON and is **not** shadowed by static files or by the fallback.
- [ ] `/nonexistent-route` returns `index.html` via `MapFallbackToFile`.
- [ ] DevTools shows `content-encoding: br` (or `gzip`) for the CSS.
- [ ] `wwwroot` contains no secrets and no `appsettings.json`.
- [ ] The intentional order-break step is documented with a screenshot/log of the 404.
- [ ] The `wwwroot/appsettings.json` step is documented and the file is removed.
- [ ] The code is commented in both languages (RU + EN).

#### Hints (no direct answer)

- Recall the restaurant analogy from the lesson: the "display case" (`wwwroot`) and the "kitchen" (controllers). What should be available instantly, and what should be made to order?
- Think about the moment in the pipeline when `Response.Body` has already been wrapped by the compressing stream. Where should static files sit relative to compression?
- `UseDefaultFiles` "rewrites the URL" — what does that mean technically? Why is the file not served?
- If `max-age=31536000` is set without `immutable`, what happens when the file is updated under the same name?
- Where should `appsettings.json` physically live so that it cannot be requested by URL?
- Why must `MapFallbackToFile` come **after** `MapGet("/api/health", ...)`?
- Can `UseFileServer` replace the three separate middleware? When is it appropriate, and when is it not?

#### Reference solution walk-through

```csharp
// Program.cs — C# 12 / .NET 8
// Full setup: static files, default files, cache, compression, fallback

using Microsoft.AspNetCore.StaticFiles;
using System.Text.RegularExpressions;

var builder = WebApplication.CreateBuilder(args);

// 1) Register response compression: Brotli for text, Gzip as a fallback
builder.Services.AddResponseCompression(options =>
{
    options.EnableForHttps = true; // Compress over HTTPS too
    options.Providers.Add<BrotliCompressionProvider>();
    options.Providers.Add<GzipCompressionProvider>();
    // C# 12 collection expression
    options.MimeTypes =
    [
        "text/plain",
        "text/css",
        "text/html",
        "application/javascript",
        "application/json",
        "image/svg+xml",
        "application/wasm"
    ];
});

var app = builder.Build();

// 2) Compression BEFORE static files so the response can be wrapped
app.UseResponseCompression();

// 3) HTTPS redirect + HSTS only in production
if (!app.Environment.IsDevelopment())
{
    app.UseHttpsRedirection();
    app.UseHsts();
}

// 4) Default files BEFORE StaticFiles (rewrites URL to index.html)
app.UseDefaultFiles();

// 5) Static files with cache headers
app.UseStaticFiles(new StaticFileOptions
{
    OnPrepareResponse = ctx =>
    {
        var path = ctx.Context.Request.Path.Value ?? string.Empty;
        // Regex looks for a fingerprint: 8+ hex chars before .css/.js
        var hasFingerprint = Regex.IsMatch(path, @"\.[a-f0-9]{8,}\.(css|js)$");
        ctx.Context.Response.Headers.CacheControl = hasFingerprint
            ? "public, max-age=31536000, immutable"
            : "public, max-age=300";
    }
});

app.UseRouting();
app.UseAuthorization();

// 6) Public endpoint that is NOT shadowed by static files
app.MapGet("/api/health", () => Results.Ok(new { status = "ok", ts = DateTime.UtcNow }));

// 7) SPA fallback to index.html — must be the last step
app.MapFallbackToFile("index.html");

app.Run();
```

Line-by-line walk-through. The line `using Microsoft.AspNetCore.StaticFiles;` brings in the `StaticFileOptions` type; `using System.Text.RegularExpressions;` brings in `Regex` for fingerprint detection. `WebApplication.CreateBuilder(args)` is the standard .NET 8 entry point with top-level statements. The `AddResponseCompression` block registers the compression service in the DI container: `EnableForHttps = true` enables compression for HTTPS responses (off by default because of the BREACH risk, but safe for static assets). Provider order matters: Brotli is preferred, Gzip stays as a fallback for older clients. `MimeTypes` is set with a C# 12 collection expression — a compact form of `new[] { ... }`.

Then comes the critically important middleware order. `app.UseResponseCompression()` is placed **first** in the pipeline so it can wrap `Response.Body` with a compressing stream before static-files middleware starts writing into it. The `!IsDevelopment()` guard protects the developer from HSTS on localhost — the browser would otherwise remember HSTS and keep redirecting to HTTPS, breaking local debugging. `UseDefaultFiles()` goes **before** `UseStaticFiles()`: it does not serve a file, it rewrites the URL `/` to `/index.html`, after which `UseStaticFiles` finds and serves it; swap the order and you get a 404.

`UseStaticFiles(new StaticFileOptions { OnPrepareResponse = ... })` is the fine-grained cache hook. `OnPrepareResponse` is a callback executed before each file is sent; here we read `Request.Path` and run a regex to check whether the name carries a hash. The pattern `\.[a-f0-9]{8,}\.(css|js)$` matches names like `site.1a2b3c4d.css`: a dot, eight or more hexadecimal characters, a dot, the extension. On a match we set `immutable` for one year (the browser will not even make a conditional request); otherwise we set a five-minute TTL. `app.UseRouting()` and `app.UseAuthorization()` come after static files because routing is only needed for endpoints, and the static asset has already been handled. `app.MapGet("/api/health", ...)` is a minimal API that is not shadowed by static files (there is no `wwwroot/api/health` file) and is not "eaten" by the fallback because `MapFallbackToFile` only triggers for unmatched routes. `app.MapFallbackToFile("index.html")` is the final step, the SPA fallback: any unknown route returns the home page so the client-side router can take over.

#### Going deeper (bonus)

1. Implement a separate `wwwroot/private` folder with a PDF document, but make sure it is **not** served directly through `UseStaticFiles`. Instead, create a `ReportsController` with an `[Authorize]` action and `PhysicalFile(...)` to serve the PDF after an authorization check. Compare with the public static path.
2. Add build-time versioning: write an MSBuild target that, on `dotnet publish`, renames `site.css` to `site.<hash>.css` and rewrites the references in `index.html`. Verify that the cache regex correctly recognises the new name.
3. Enable `UseFileServer` instead of the three separate middleware in a separate branch and measure the performance difference with `dotnet-counters`. Explain why production teams prefer separate calls.
4. Add a CDN header `Access-Control-Allow-Origin` for fonts served from `wwwroot/fonts` via `OnPrepareResponse`, implementing the rule "if the path starts with `/fonts/`, add CORS".

---

#### Чек-лист сдачи / Submission checklist
- [ ] Проект `StaticFilesDemo` создан на .NET 8 и запускается `dotnet run`.
- [ ] Порядок middleware соответствует уроку (сжатие → HTTPS/HSTS → default → static → routing → auth → endpoints → fallback).
- [ ] `OnPrepareResponse` корректно различает fingerprint и обычные файлы.
- [ ] Проверены все URL: `/`, `/css/site.1a2b3c4d.css`, `/js/app.js`, `/img/logo.svg`, `/api/health`, `/nonexistent-route`.
- [ ] Скриншоты DevTools с `content-encoding` и `cache-control` приложены.
- [ ] Документирован шаг с поломкой порядка (404 на `/`) и удалённый `wwwroot/appsettings.json`.
- [ ] Код прокомментирован на двух языках.
- [ ] The `StaticFilesDemo` project is created on .NET 8 and runs via `dotnet run`.
- [ ] Middleware order matches the lesson (compression → HTTPS/HSTS → default → static → routing → auth → endpoints → fallback).
- [ ] `OnPrepareResponse` correctly distinguishes fingerprinted and plain files.
- [ ] All URLs are verified: `/`, `/css/site.1a2b3c4d.css`, `/js/app.js`, `/img/logo.svg`, `/api/health`, `/nonexistent-route`.
- [ ] DevTools screenshots with `content-encoding` and `cache-control` are attached.
- [ ] The order-break step (404 on `/`) and the removed `wwwroot/appsettings.json` are documented.
- [ ] The code is commented in both languages.

#### Ресурсы / Resources
- [Microsoft Learn — Static files in ASP.NET Core](https://learn.microsoft.com/aspnet/core/fundamentals/static-files)
- [Microsoft Learn — Response compression in ASP.NET Core](https://learn.microsoft.com/aspnet/core/performance/response-compression)
- [Microsoft Learn — Default files in ASP.NET Core](https://learn.microsoft.com/aspnet/core/fundamentals/static-files#serve-a-default-document)
- [Microsoft Learn — StaticFileOptions.OnPrepareResponse](https://learn.microsoft.com/dotnet/api/microsoft.aspnetcore.builder.staticfileoptions.onprepareresponse)
- [MDN — Cache-Control](https://developer.mozilla.org/docs/Web/HTTP/Headers/Cache-Control)
- [MDN — Immutable cache responses](https://developer.mozilla.org/docs/Web/HTTP/Headers/Cache-Control#immutable)
