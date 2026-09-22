[⬆ К модулю M13](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M13-L08: MVC/Controllers (обзор) и Razor Pages (обзор) / MVC/Controllers (overview) and Razor Pages (overview)

**Модуль / Module:** M13
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

ASP.NET Core предлагает две основные модели для построения веб-интерфейсов: **MVC** (Model-View-Controller) и **Razor Pages**. Обе работают поверх одного и того же движка маршрутизации и Razor, поэтому выбирать приходится скорее по стилю проекта, а не по возможностям — технически они равноценны.

**Паттерн MVC** разделяет приложение на три роли. *Model* — это ваши данные и бизнес-логика: классы сущностей, DTO, сервисы домена. *View* — это Razor-представление (`.cshtml`), которое превращает модель в HTML. *Controller* — класс, помеченный атрибутом `[ApiController]` или унаследованный от `Controller`, который принимает HTTP-запрос, вызывает сервисы и возвращает результат (`View`, `Json`, `Redirect`). Аналогия: ресторан. Клиент (HTTP-запрос) делает заказ официанту (Controller), официант передаёт заказ на кухню (Model/сервисы), а потом блюдо выносит через красивую подачу (View).

**Контроллер** — это «точка входа» для запроса. В ASP.NET Core маршруты связывают URL с методами контроллера (actions). Например, `GET /products/list` вызывает `ProductsController.List()`. Метод читает параметры из маршрута, запроса и тела, обращается к сервисам и формирует ответ. Контроллер не должен содержать SQL и бизнес-правил — он тонкий координатор.

**Views** — это Razor-файлы, обычно лежащие в `Views/<ControllerName>/<Action>.cshtml`. Razor смешивает HTML и C#: всё, что начинается с `@`, выполняется как код. Конструкции вроде `@model`, `@{ }`, `@if`, `@foreach`, `@Html.ActionLink` позволяют динамически формировать разметку. Сильная типизация через `@model Product` даёт IntelliSense и проверку на этапе компиляции.

**Синтаксис Razor (`@`)** — сердце обеих моделей. `@DateTime.Now` вставит текущую дату. `@{ var x = 1; }` — блок кода. `@@` экранирует сам символ `@`. Выражения и блоки можно вкладывать, но важно помнить: `@` переключает HTML↔C#, поэтому пробелы и перенос строк влияют на результат.

**Razor Pages** — это «страницо-ориентированный» подход (page-first). Здесь единицей работы является не контроллер с кучей actions, а отдельная страница `Index.cshtml` с файлом-«спутником» `Index.cshtml.cs`, где живёт `PageModel`. Каждая страница сама отвечает за свой URL (по умолчанию путь к файлу = маршрут). Аналогия: MVC — это большой шкаф с ящиками (контроллер), где каждый ящик — action; Razor Pages — это отдельные коробочки, по одной на каждую страницу, со своей логикой внутри.

**Tag Helpers** — встроенные расширения Razor, которые позволяют писать серверную логику прямо в HTML-тегах. Например, `<a asp-controller="Products" asp-action="List">` автоматически сгенерирует правильный URL, а `<form asp-page-handler="Save">` в Razor Pages привяжет отправку формы к методу `OnPostSaveAsync`. Tag Helpers понятнее верстальщикам, потому что выглядят как обычный HTML, а не как вызовы `@Html.*`.

**Когда MVC, а когда Razor Pages?** Простое правило: если логика страницы самодостаточна (форма контактов, профиль пользователя, CRUD одной сущности) — берите Razor Pages: меньше файлов, понятнее поток. Если много операций над одним ресурсом группируются в API или UI с общей доменной моделью (админка каталога, dashboard с десятками действий) — MVC даёт удобную централизацию. Оба подхода можно смешивать в одном приложении, поэтому выбор не необратим.

---

#### Theory (EN)

ASP.NET Core offers two main models for building web UIs: **MVC** (Model-View-Controller) and **Razor Pages**. Both run on the same routing engine and Razor view engine, so the choice is mostly about project style rather than capabilities — technically they are peers.

**The MVC pattern** splits an application into three roles. *Model* is your data and business logic: entity classes, DTOs, domain services. *View* is a Razor file (`.cshtml`) that turns a model into HTML. *Controller* is a class decorated with `[ApiController]` or deriving from `Controller` that receives the HTTP request, calls services, and returns a result (`View`, `Json`, `Redirect`). Analogy: a restaurant. The customer (HTTP request) orders from the waiter (Controller), the waiter passes the order to the kitchen (Model/services), and the dish is brought out with nice plating (View).

**A controller** is the entry point for a request. In ASP.NET Core, routes map URLs to controller methods (actions). For example, `GET /products/list` calls `ProductsController.List()`. The method reads parameters from the route, query string, and body, talks to services, and shapes the response. A controller should not contain SQL or business rules — it is a thin coordinator.

**Views** are Razor files, usually stored under `Views/<ControllerName>/<Action>.cshtml`. Razor mixes HTML and C#: anything starting with `@` runs as code. Constructs like `@model`, `@{ }`, `@if`, `@foreach`, and `@Html.ActionLink` let you build markup dynamically. Strong typing via `@model Product` gives IntelliSense and compile-time checks.

**Razor syntax (`@`)** is the heart of both models. `@DateTime.Now` renders the current date. `@{ var x = 1; }` is a code block. `@@` escapes the `@` character itself. Expressions and blocks can nest, but remember: `@` toggles between HTML and C#, so whitespace and line breaks affect output.

**Razor Pages** is a page-first approach. Instead of a controller with many actions, the unit of work is a single page `Index.cshtml` paired with a code-behind file `Index.cshtml.cs` containing a `PageModel`. Each page owns its URL (by default the file path equals the route). Analogy: MVC is a big cabinet with drawers (one controller), where each drawer is an action; Razor Pages is a set of separate boxes, one per page, each with its own logic inside.

**Tag Helpers** are built-in Razor extensions that put server-side logic directly into HTML tags. For example, `<a asp-controller="Products" asp-action="List">` generates the correct URL automatically, and `<form asp-page-handler="Save">` in Razor Pages binds the form submission to the `OnPostSaveAsync` method. Tag Helpers are friendlier to designers because they look like ordinary HTML rather than `@Html.*` calls.

**When MVC vs Razor Pages?** A simple rule: if a page's logic is self-contained (a contact form, a user profile, CRUD on a single entity) — pick Razor Pages: fewer files, clearer flow. If many operations on a resource group into an API or UI with a shared domain model (a catalog admin panel, a dashboard with dozens of actions) — MVC gives convenient centralization. Both approaches can be mixed in one application, so the choice is not irreversible.

---

#### Пример кода / Code Example

```csharp
// =============================================================
// M13-L08: обзор MVC-контроллера и Razor Pages PageModel
// Overview of an MVC controller and a Razor Pages PageModel
// C# 12 / .NET 8
// =============================================================

using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.RazorPages;

// --- Доменная модель / Domain model ---------------------------
public record Product(int Id, string Name, decimal Price);

// --- Репозиторий-заглушка / Stub repository ------------------
public interface IProductRepository
{
    IReadOnlyList<Product> GetAll();
    Product? Get(int id);
    void Add(Product product);
}

public sealed class InMemoryProductRepository : IProductRepository
{
    private readonly List<Product> _items = new();

    public IReadOnlyList<Product> GetAll() => _items;
    public Product? Get(int id) => _items.FirstOrDefault(p => p.Id == id);
    public void Add(Product product) => _items.Add(product);
}

// =============================================================
// 1) MVC-КОНТРОЛЛЕР / MVC CONTROLLER
//    Один класс группирует несколько действий над ресурсом.
//    One class groups several actions over a resource.
// =============================================================
public class ProductsController : Controller
{
    private readonly IProductRepository _repo;

    // Внедрение зависимостей через конструктор / Constructor DI
    public ProductsController(IProductRepository repo) => _repo = repo;

    // GET /Products — список товаров / list of products
    public IActionResult Index()
    {
        var products = _repo.GetAll();
        return View(products); // вернёт Views/Products/Index.cshtml
                                // returns Views/Products/Index.cshtml
    }

    // GET /Products/Details/5 — один товар / single product
    public IActionResult Details(int id)
    {
        var product = _repo.Get(id);
        if (product is null) return NotFound(); // 404
        return View(product);
    }

    // POST /Products/Create — приём формы / form submit
    [HttpPost]
    [ValidateAntiForgeryToken] // защита от CSRF / CSRF protection
    public IActionResult Create([Bind(nameof(Product.Name), nameof(Product.Price))] Product product)
    {
        if (!ModelState.IsValid) return View(product); // повтор формы с ошибками
                                                        // re-show form with errors
        var nextId = _repo.GetAll().Count + 1;
        _repo.Add(product with { Id = nextId });
        return RedirectToAction(nameof(Index));
    }
}

// =============================================================
// 2) RAZOR PAGES — PageModel
//    Одна страница = один файл .cshtml + один .cshtml.cs.
//    One page = one .cshtml file + one .cshtml.cs file.
// =============================================================
public class IndexModel : PageModel
{
    private readonly IProductRepository _repo;

    public IndexModel(IProductRepository repo) => _repo = repo;

    // Данные для отображения на странице / Data for the page
    public IReadOnlyList<Product> Products { get; private set; } = Array.Empty<Product>();

    // GET — загрузка страницы / page load
    public void OnGet() => Products = _repo.GetAll();

    // POST — обработка кнопки "Добавить" / handle "Add" button.
    // Имя обработчика = Save => asp-page-handler="Save".
    // Handler name = Save => asp-page-handler="Save".
    public IActionResult OnPostSave(string name, decimal price)
    {
        if (string.IsNullOrWhiteSpace(name) || price <= 0)
        {
            ModelState.AddModelError(string.Empty, "Некорректные данные / Invalid data");
            Products = _repo.GetAll();
            return Page();
        }

        var nextId = _repo.GetAll().Count + 1;
        _repo.Add(new Product(nextId, name, price));
        return RedirectToPage(); // перезагрузка страницы / reload page
    }
}
```

```html
@* Views/Products/Index.cshtml — MVC-представление / MVC view *@
@model IReadOnlyList<Product>

<h1>Товары / Products</h1>
<ul>
    @foreach (var p in Model)
    {
        <li>@p.Name — @p.Price.ToString("C")</li>
    }
</ul>

@* Tag Helper генерирует URL автоматически / Tag Helper builds the URL *@
<a asp-controller="Products" asp-action="Create">Добавить / Add</a>
```

```html
@* Pages/Index.cshtml — Razor Pages страница / Razor Page *@
@page
@model IndexModel

<h1>Товары / Products</h1>
<ul>
    @foreach (var p in Model.Products)
    {
        <li>@p.Name — @p.Price.ToString("C")</li>
    }
</ul>

<form method="post">
    <input name="name" placeholder="Название / Name" />
    <input name="price" type="number" step="0.01" />
    @* asp-page-handler связывает форму с OnPostSaveAsync *@
    @* asp-page-handler binds the form to OnPostSaveAsync *@
    <button asp-page-handler="Save">Добавить / Add</button>
</form>
```

#### Best Practices

- Держите контроллеры и PageModel «тонкими»: бизнес-логику выносите в сервисы и доменные типы.
- Keep controllers and PageModels thin: move business logic into services and domain types.
- Используйте сильную типизацию (`@model T`) в представлениях — это даёт IntelliSense и защиту от опечаток.
- Use strongly typed views (`@model T`) for IntelliSense and compile-time safety.
- Группируйте связанные действия в одном контроллере; для автономных страниц выбирайте Razor Pages.
- Group related actions in one controller; pick Razor Pages for self-contained pages.
- Применяйте Tag Helpers вместо `@Html.*` — разметка остаётся читаемой для верстальщиков.
- Prefer Tag Helpers over `@Html.*` so the markup stays readable for designers.
- Всегда добавляйте `[ValidateAntiForgeryToken]` для POST-форм и валидируйте `ModelState`.
- Always add `[ValidateAntiForgeryToken]` for POST forms and validate `ModelState`.

#### Частые ошибки / Common Mistakes

- Бизнес-логика и доступ к данным прямо в контроллере → вынести в сервисы и внедрять через DI.
- Business logic and data access directly in the controller → move into services and inject via DI.
- Забытая директива `@page` в начале Razor Page → маршрут не регистрируется; `@page` обязана быть первой строкой.
- Missing `@page` directive at the top of a Razor Page → the route is not registered; `@page` must be the first line.
- Использование `ViewBag` вместо сильно типизированной модели → переключиться на `@model T` и `return View(model)`.
- Using `ViewBag` instead of a strongly typed model → switch to `@model T` and `return View(model)`.
- Путаница `asp-controller`/`asp-action` (MVC) и `asp-page`/`asp-page-handler` (Razor Pages) → проверять, к какой модели относится страница.
- Mixing up `asp-controller`/`asp-action` (MVC) with `asp-page`/`asp-page-handler` (Razor Pages) → check which model the page belongs to.
- Случайный пробел после `@`, ломающий выражение → писать `@item.Name` без пробела или оборачивать в `@( ... )`.
- Accidental space after `@` breaking the expression → write `@item.Name` without a space or wrap in `@( ... )`.
- Игнорирование `ModelState.IsValid` в POST → всегда проверять и возвращать форму с ошибками при невалидных данных.
- Ignoring `ModelState.IsValid` in POST → always check and re-show the form with errors on invalid data.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Я могу объяснить роли Model, View, Controller своими словами.
- [ ] I can explain the roles of Model, View, Controller in my own words.
- [ ] Я знаю разницу между MVC (controller-first) и Razor Pages (page-first).
- [ ] I know the difference between MVC (controller-first) and Razor Pages (page-first).
- [ ] Я могу написать action-метод контроллера и вернуть `View(model)`.
- [ ] I can write a controller action method and return `View(model)`.
- [ ] Я понимаю синтаксис `@`: выражения, блоки кода, экранирование `@@`.
- [ ] I understand `@` syntax: expressions, code blocks, escaping with `@@`.
- [ ] Я могу создать Razor Page с `@page` и методом `OnGet`/`OnPost*`.
- [ ] I can create a Razor Page with `@page` and an `OnGet`/`OnPost*` method.
- [ ] Я использую Tag Helpers (`asp-controller`, `asp-action`, `asp-page-handler`).
- [ ] I use Tag Helpers (`asp-controller`, `asp-action`, `asp-page-handler`).
- [ ] Я могу обосновать выбор MVC или Razor Pages для конкретной задачи.
- [ ] I can justify choosing MVC or Razor Pages for a given task.

#### Ресурсы / Resources

- [Microsoft Learn — https://learn.microsoft.com/aspnet/core/mvc/overview](https://learn.microsoft.com/aspnet/core/mvc/overview)

---

[⬆ К модулю M13](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
