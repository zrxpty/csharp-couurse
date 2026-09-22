[⬆ К модулю M13](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M13-L07: Статические файлы, wwwroot / Static files, wwwroot

**Модуль / Module:** M13
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Статические файлы — это ресурсы, которые сервер отдаёт «как есть», без обработки: HTML, CSS, JavaScript, изображения, шрифты, видео, документы. В ASP.NET Core для них существует специальная папка `wwwroot` (буквально «web root» — корень веба). Аналогия: представь ресторан. Кухня — это твои контроллеры и Razor-страницы, где блюда готовят под заказ. А `wwwroot` — это витрина с готовыми пирожными: клиент берёт их сразу, без ожидания и без повара.

По умолчанию шаблон ASP.NET Core уже подключает статику через `app.UseStaticFiles()` в `Program.cs`. Важно понимать порядок middleware: статические файлы обрабатываются **до** маршрутизации (`UseRouting`), маршрутизации MVC/Razor (`MapControllerRoute`/`MapRazorPages`) и обработки конечных точек (`UseEndpoints`). Если файл найден, конвейер прерывается — запрос не доходит до контроллеров. Это даёт максимальную скорость: IIS или Kestrel отдают файлы напрямую из памяти или с диска.

Путь к файлу формируется относительно `wwwroot`. Файл `wwwroot/css/site.css` доступен по URL `/css/site.css`. Корень веба (`wwwroot`) **не является** корнем приложения: файлы вне `wwwroot` по умолчанию недоступны из веба — это защита от случайной публикации исходников и конфигураций.

**Default files.** Когда пользователь заходит на `/`, сервер ищет один из файлов по умолчанию: `index.html`, `index.htm`, `default.html`, `default.htm`. За это отвечает `app.UseDefaultFiles()`. Важно: `UseDefaultFiles` только **переписывает URL**, он не отдаёт файл сам — поэтому его ставят **до** `UseStaticFiles`. Альтернатива — `app.UseFileServer()`, комбинирующий `UseDefaultFiles` + `UseStaticFiles` + `UseDirectoryBrowser` (просмотр каталогов обычно отключают в продакшене из соображений безопасности).

**FileServer** удобен для простых сайтов-визиток и SPA-приложений: одна строка заменяет три middleware. Однако для сложных проектов предпочитают раздельные вызовы — это даёт тонкий контроль над порядком и настройками.

**Cache headers** управляют кэшированием на стороне браузера и CDN. По умолчанию `UseStaticFiles` не выставляет строгий `Cache-Control`, и файлы переотдаются при каждом запросе. Для продакшена это расточительно. Решение — `StaticFileOptions.OnPrepareResponse`, где задают `Cache-Control: public, max-age=31536000` для версионированных ресурсов (с хешем в имени) и короткий `max-age` для неизменяемых. Ключевая практика: внедрить fingerprinting (`site.v1a2b3.css`) — тогда длинный `max-age` безопасен, так как новое имя заставит браузер запросить новый файл.

**Compression.** Текстовые ресурсы (CSS, JS, HTML, SVG, JSON) сжимают через `app.UseResponseCompression()` с поставщиками `Gzip` или `Brotli`. Brotli даёт на 15–20% лучшее сжатие для текста и поддерживается всеми современными браузерами. Сжатие ставят **до** `UseStaticFiles`, чтобы middleware успело обернуть ответ. Комбинирование сжатия с кэшированием радикально снижает объём трафика и ускоряет загрузку: один сжатый CSS в 40 КБ вместо 200 КБ, кэшированный на год, — это миллионы сэкономленных запросов.

Безопасность статики: никогда не клади в `wwwroot` секреты (`appsettings.json`, `.env`), конфиги БД, ключи. Используй `UseHsts` и `UseHttpsRedirection`, чтобы статику тоже отдавали по HTTPS. Для защищённых файлов (приватные документы) статику отдают через контроллер с атрибутом `[Authorize]`, а не через `wwwroot`.

#### Theory (EN)

Static files are resources the server delivers "as is", without processing: HTML, CSS, JavaScript, images, fonts, video, documents. In ASP.NET Core they live in a special folder called `wwwroot` (literally "web root"). Analogy: imagine a restaurant. The kitchen is your controllers and Razor pages, where dishes are cooked to order. `wwwroot` is the display case with ready-made pastries: a client takes one instantly, with no wait and no chef involved.

By default, the ASP.NET Core template already wires up static files through `app.UseStaticFiles()` in `Program.cs`. Order of middleware matters: static files are handled **before** routing (`UseRouting`), before MVC/Razor endpoint mapping (`MapControllerRoute`/`MapRazorPages`), and before endpoint execution. If a file is found, the pipeline short-circuits — the request never reaches your controllers. That yields maximum speed: IIS or Kestrel serves files straight from memory or disk.

The URL path is built relative to `wwwroot`. The file `wwwroot/css/site.css` is reachable at URL `/css/site.css`. The web root (`wwwroot`) is **not** the application root: files outside `wwwroot` are not web-accessible by default — a deliberate safeguard against accidentally publishing source code and configuration.

**Default files.** When a user navigates to `/`, the server looks for one of the default files: `index.html`, `index.htm`, `default.html`, `default.htm`. This is the job of `app.UseDefaultFiles()`. Crucially, `UseDefaultFiles` only **rewrites the URL** — it does not serve the file itself, so it must be registered **before** `UseStaticFiles`. The alternative is `app.UseFileServer()`, which combines `UseDefaultFiles` + `UseStaticFiles` + `UseDirectoryBrowser` (directory browsing is usually disabled in production for security).

**FileServer** is handy for simple brochure sites and SPAs: one line replaces three middleware calls. For complex projects, however, teams prefer separate calls — finer control over ordering and options.

**Cache headers** drive caching on the browser side and at CDN edges. Out of the box `UseStaticFiles` does not set a strict `Cache-Control`, so files are re-sent on every request. That is wasteful in production. The fix is `StaticFileOptions.OnPrepareResponse`, where you set `Cache-Control: public, max-age=31536000` for versioned assets (hash in the filename) and a short `max-age` for files that may change. The key practice is fingerprinting (`site.v1a2b3.css`) — then a long `max-age` is safe, because a new name forces the browser to fetch the new file.

**Compression.** Textual assets (CSS, JS, HTML, SVG, JSON) are compressed through `app.UseResponseCompression()` with `Gzip` or `Brotli` providers. Brotli gives 15–20% better compression for text and is supported by all modern browsers. Compression must be registered **before** `UseStaticFiles`, so the middleware can wrap the response. Pairing compression with caching dramatically reduces traffic and speeds up loading: a single compressed CSS at 40 KB instead of 200 KB, cached for a year, saves millions of redundant requests.

Security of static content: never put secrets in `wwwroot` (`appsettings.json`, `.env`), DB configs, keys. Use `UseHsts` and `UseHttpsRedirection` so static files are also served over HTTPS. For protected files (private documents), serve them through a controller with an `[Authorize]` attribute rather than through `wwwroot`.

#### Пример кода / Code Example

```csharp
// Program.cs — C# 12 / .NET 8
// Полная настройка статики, default files, кэша и сжатия
// Full setup: static files, default files, cache, compression

var builder = WebApplication.CreateBuilder(args);

// Регистрируем сжатие ответов / Register response compression
builder.Services.AddResponseCompression(options =>
{
    options.EnableForHttps = true; // Сжимать и по HTTPS / Compress over HTTPS too
    options.Providers.Add<BrotliCompressionProvider>(); // Brotli — лучший для текста
    options.Providers.Add<GzipCompressionProvider>();   // Fallback для старых клиентов
    // Сжимаем только текстовые MIME-типы / Compress only textual MIME types
    options.MimeTypes = new[]
    {
        "text/plain",
        "text/css",
        "text/html",
        "application/javascript",
        "application/json",
        "image/svg+xml",
        "application/wasm"
    };
});

var app = builder.Build();

// 1) Сжатие — ДО статики, чтобы обернуть ответ / Compression BEFORE static files
app.UseResponseCompression();

// 2) HTTPS-редирект и HSTS / HTTPS redirection and HSTS
if (!app.Environment.IsDevelopment())
{
    app.UseHttpsRedirection();
    app.UseHsts();
}

// 3) Default files — ДО StaticFiles (переписывает URL на index.html)
// Default files BEFORE StaticFiles (rewrites URL to index.html)
app.UseDefaultFiles();

// 4) Статические файлы с кэш-заголовками / Static files with cache headers
app.UseStaticFiles(new StaticFileOptions
{
    OnPrepareResponse = ctx =>
    {
        // Версионированные файлы (с хешем) кэшируем на год / Versioned files cached 1 year
        // Неверсионированные — на 5 минут / Non-versioned — 5 minutes
        var path = ctx.Context.Request.Path.Value ?? string.Empty;
        var hasFingerprint = System.Text.RegularExpressions.Regex.IsMatch(
            path, @"\.[a-f0-9]{8,}\.(css|js)$");

        ctx.Context.Response.Headers.CacheControl = hasFingerprint
            ? "public, max-age=31536000, immutable"
            : "public, max-age=300";
    }
});

app.UseRouting();
app.UseAuthorization();

// Minimal API-эндпоинт, который НЕ перехватывается статикой / Endpoint not shadowed by static files
app.MapGet("/api/health", () => Results.Ok(new { status = "ok", ts = DateTime.UtcNow }));

app.MapFallbackToFile("index.html"); // SPA-фоллбэк на index.html / SPA fallback to index.html

app.Run();
```

```xml
<!-- wwwroot/css/site.css — пример версионированного файла / example versioned file -->
<!-- В продакшене имя генерируется сборкой: site.1a2b3c4d.css -->
```

#### Best Practices
- Ставь `UseStaticFiles` до маршрутизации и контроллеров, чтобы запросы к статике не проходили лишний конвейер. (RU)
- Версионируй имена файлов (fingerprinting) и ставь длинный `max-age` с `immutable` — безопасно для кэша. (RU)
- Включай `UseResponseCompression` с Brotli **до** `UseStaticFiles`, чтобы сжимать статику. (RU)
- Никогда не храни секреты, `appsettings` и ключи внутри `wwwroot` — они публично доступны. (RU)
- Для защищённых файлов используй контроллер с `[Authorize]`, а не папку `wwwroot`. (RU)
- Register `UseStaticFiles` before routing and controllers so static requests skip the rest of the pipeline. (EN)
- Fingerprint filenames and set a long `max-age` with `immutable` — safe and cache-friendly. (EN)
- Enable `UseResponseCompression` with Brotli **before** `UseStaticFiles` to compress static assets. (EN)
- Never store secrets, `appsettings`, or keys inside `wwwroot` — it is publicly served. (EN)
- For protected files, use a controller with `[Authorize]` instead of the `wwwroot` folder. (EN)

#### Частые ошибки / Common Mistakes
- `UseDefaultFiles` стоит **после** `UseStaticFiles` → `/` отдаёт 404. Поставь `UseDefaultFiles` первым. (RU)
- Секреты лежат в `wwwroot/appsettings.json` → утечка. Выноси конфиги за пределы `wwwroot`. (RU)
- Длинный `max-age` без версионирования → браузер не видит обновлённый CSS. Внедрять fingerprint. (RU)
- `UseResponseCompression` после `UseStaticFiles` → статику не сжимает. Перенеси выше. (RU)
- `UseDirectoryBrowser` включён в продакшене → утечка структуры. Отключи через `FileServerOptions`. (RU)
- `UseDefaultFiles` is placed **after** `UseStaticFiles` → `/` returns 404. Put `UseDefaultFiles` first. (EN)
- Secrets placed in `wwwroot/appsettings.json` → leak. Move configs outside `wwwroot`. (EN)
- Long `max-age` without fingerprinting → browser never sees the updated CSS. Add fingerprinting. (EN)
- `UseResponseCompression` after `UseStaticFiles` → static assets are not compressed. Move it up. (EN)
- `UseDirectoryBrowser` enabled in production → structure leak. Disable via `FileServerOptions`. (EN)

#### Чек-лист самопроверки / Self-check Checklist
- [ ] `UseStaticFiles()` вызывается до `UseRouting()` и маппинга эндпоинтов. (RU)
- [ ] `UseDefaultFiles()` стоит **до** `UseStaticFiles()`. (RU)
- [ ] `wwwroot` не содержит секретов и конфигов. (RU)
- [ ] Включён `UseResponseCompression` с Brotli **до** статики. (RU)
- [ ] Кэш-заголовки: версионированные — `max-age=31536000, immutable`, прочие — короткий. (RU)
- [ ] `MapFallbackToFile("index.html")` настроен для SPA. (RU)
- [ ] `UseStaticFiles()` is called before `UseRouting()` and endpoint mapping. (EN)
- [ ] `UseDefaultFiles()` is registered **before** `UseStaticFiles()`. (EN)
- [ ] `wwwroot` contains no secrets or configs. (EN)
- [ ] `UseResponseCompression` with Brotli is enabled **before** static files. (EN)
- [ ] Cache headers: versioned — `max-age=31536000, immutable`; others — short. (EN)
- [ ] `MapFallbackToFile("index.html")` is configured for SPA. (EN)

#### Ресурсы / Resources
- [Microsoft Learn — https://learn.microsoft.com/aspnet/core/fundamentals/static-files](https://learn.microsoft.com/aspnet/core/fundamentals/static-files)

---

[⬆ К модулю M13](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
