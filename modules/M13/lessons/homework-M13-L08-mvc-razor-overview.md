---
[← К уроку M13-L08](lesson-M13-L08-mvc-razor-overview.md) | [⬆ К модулю M13](../README.md) | [Следующее ДЗ →](homework-M13-L09-error-handling-environments.md)
---

### Домашнее задание M13-L08: MVC/Controllers (обзор) и Razor Pages (обзор) / Homework M13-L08: MVC/Controllers (overview) and Razor Pages (overview)

**Урок / Lesson:** M13-L08
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) На практике освоить обе модели построения веб-интерфейсов ASP.NET Core 8 — MVC-контроллеры с сильно типизированными представлениями и Razor Pages с `PageModel`. Научиться выбирать подход под задачу, применять Tag Helpers, маршрутизацию, валидацию `ModelState` и защиту от CSRF, держать контроллеры и PageModel «тонкими» за счёт внедрения зависимостей. (EN) Gain hands-on experience with both ASP.NET Core 8 web UI models — MVC controllers with strongly typed views and Razor Pages with `PageModel`. Learn to choose the right approach per task, apply Tag Helpers, routing, `ModelState` validation, and CSRF protection, and keep controllers and PageModels thin via dependency injection.

#### Связь с уроком / Connection to the lesson
(RU — 2–3 предложения) Урок вводит роли Model/View/Controller и page-first модель Razor Pages, синтаксис Razor (`@`), Tag Helpers и критерии выбора подхода. ДЗ закрепляет всё это в одном работающем приложении: вы построите каталог товаров на MVC и форму обратной связи на Razor Pages, чтобы на себе почувствовать разницу стилей. Особое внимание уделено лучшим практикам из урока: тонкие контроллеры, сильная типизация, `[ValidateAntiForgeryToken]`, проверка `ModelState`.
(EN — same) The lesson introduces the Model/View/Controller roles and the page-first Razor Pages model, the Razor (`@`) syntax, Tag Helpers, and the criteria for choosing between the approaches. This homework cements all of that in a single working app: you will build a product catalog with MVC and a contact form with Razor Pages to feel the stylistic difference first-hand. Special attention goes to the lesson's best practices: thin controllers, strong typing, `[ValidateAntiForgeryToken]`, and `ModelState` validation.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Вы — backend-разработчик небольшой команды, которая запускает внутренний портал «Мини-каталог» для отдела закупок. Портал должен показывать список товаров, открывать страницу с деталями товара и принимать новые записи через форму. Параллельно нужен раздел «Обратная связь» — простая контактная форма, которая пишет сообщение в лог и благодарит пользователя. Технический руководитель попросил вас не плодить одинаковый код, а осознанно применить обе модели, которые разбирались в уроке M13-L08: MVC для каталога (где много операций над одним ресурсом группируются в одном контроллере) и Razor Pages для контактной формы (где логика страницы самодостаточна). Это даст команде живой пример для будущих решений: «когда MVC, а когда Razor Pages».

Урок подчёркивает, что обе модели технически равноценны и работают поверх одного движка маршрутизации и Razor, поэтому выбор — вопрос стиля проекта, а не возможностей. Вы должны на практике убедиться в этом: одни и те же Tag Helpers, один и тот же синтаксис `@`, один и тот же механизм внедрения зависимостей. Заодно вы отработаете частые ошибки из урока: забытая директива `@page`, бизнес-логика прямо в контроллере, использование `ViewBag` вместо сильно типизированной модели, путаница `asp-controller/asp-action` и `asp-page/asp-page-handler`, игнорирование `ModelState.IsValid`. К концу задания у вас будет компилируемое приложение, готовое к запуску локально, и понимание, какой подход когда выбирать.

#### Что нужно сделать (пошагово)

1. **Создайте проект.** Из папки `modules/M13/lessons` выполните команду создания нового MVC-приложения, которое по умолчанию уже поддерживает и контроллеры, и Razor Pages:

   ```bash
   dotnet new mvc -n MiniCatalog -o MiniCatalog --framework net8.0
   cd MiniCatalog
   ```

   Убедитесь, что в `Program.cs` зарегистрированы `AddControllersWithViews()` и `AddRazorPages()` (шаблон `mvc` уже подключает и то, и другое, и добавляет `MapControllerRoute` с паттерном `{controller=Home}/{action=Index}/{id?}`, а также `MapRazorPages`). Запустите `dotnet build` — он должен пройти без ошибок и предупреждений.

2. **Опишите доменную модель и репозиторий.** В папке `Models` создайте файл `Product.cs` с `record Product(int Id, string Name, decimal Price, string Category)`. Затем добавьте интерфейс `IProductRepository` с методами `IReadOnlyList<Product> GetAll()`, `Product? Get(int id)` и `void Add(Product product)`, а также реализацию `InMemoryProductRepository`, которая хранит товары в `List<Product>` и инициализируется тремя-четырьмя тестовыми записями при запуске. В `Program.cs` зарегистрируйте репозиторий как singleton: `builder.Services.AddSingleton<IProductRepository, InMemoryProductRepository>();`. Это та самая зависимость, которую контроллер и PageModel получат через конструктор.

3. **Реализуйте MVC-контроллер каталога.** В `Controllers/ProductsController.cs` создайте класс `ProductsController : Controller`. Внедрите `IProductRepository` через конструктор и сохраните в поле только для чтения. Добавьте три action-метода: `Index()` — возвращает `View(_repo.GetAll())`; `Details(int id)` — ищет товар, возвращает `NotFound()` при отсутствии или `View(product)`; `Create()` (GET) — возвращает пустую форму; `Create(Product product)` (POST) — помечен `[HttpPost]` и `[ValidateAntiForgeryToken]`, проверяет `ModelState.IsValid`, при ошибке возвращает `View(product)`, иначе вычисляет новый идентификатор (`_repo.GetAll().Count + 1`), добавляет продукт через `product with { Id = nextId }` и делает `RedirectToAction(nameof(Index))`. Контроллер должен оставаться тонким координатором — никакой бизнес-логики, кроме делегирования репозиторию.

4. **Создайте сильно типизированные представления.** В `Views/Products/` создайте `Index.cshtml` с директивой `@model IReadOnlyList<MiniCatalog.Models.Product>`, заголовком `<h1>Каталог товаров</h1>` и таблицей, построенной через `@foreach (var p in Model)`. Колонка «Детали» должна использовать Tag Helper `<a asp-controller="Products" asp-action="Details" asp-route-id="@p.Id">подробнее</a>`, а ссылка «Добавить» — `<a asp-controller="Products" asp-action="Create">`. Файл `Details.cshtml` примите `@model Product` и покажите все поля. Файл `Create.cshtml` примите `@model Product`, используйте `<form asp-controller="Products" asp-action="Create" method="post">` с полями `Name`, `Price`, `Category` и кнопкой submit; не забудьте `@Html.AntiForgeryToken()` или rely на автоматическую генерацию токена Tag Helper-ом формы.

5. **Добавьте Razor Page для обратной связи.** В папке `Pages/Contact/` создайте пару файлов: `Index.cshtml` и `Index.cshtml.cs`. В code-behind класс `ContactIndexModel : PageModel` должен внедрять `ILogger<ContactIndexModel>`, содержать свойства `[BindProperty] public string? Name { get; set; }` и `[BindProperty] public string? Message { get; set; }`, метод `OnGet()` без параметров и метод `OnPostAsync()`, который проверяет `ModelState.IsValid`, логирует сообщение через `_logger.LogInformation("Contact from {Name}: {Message}", Name, Message)`, устанавливает `TempData["Thanks"] = "Спасибо за обращение!"` и возвращает `RedirectToPage()`. В `.cshtml` обязательно первая строка `@page`, затем `@model ContactIndexModel`, форма `<form method="post">` с `asp-page-handler` (или просто с `method="post"`, тогда сработает `OnPostAsync`) и вывод `TempData["Thanks"]`, если он есть.

6. **Свяжите навигацию и проверьте.** В `Views/Shared/_Layout.cshtml` добавьте две ссылки в меню: на каталог (`asp-controller="Products" asp-action="Index"`) и на обратную связь (`asp-page="/Contact/Index"`). Запустите `dotnet run`, откройте `https://localhost:<port>` и убедитесь, что обе ветки работают: каталог показывает список, «подробнее» открывает детали, форма создания принимает POST и редиректит на список, контактная форма пишет в консоль лог и показывает благодарность.

7. **Объясните выбор.** В файле `CHOICE.md` в корне проекта напишите 5–7 предложений на русском, почему для каталога выбран MVC, а для контактов — Razor Pages, со ссылкой на критерии из урока.

#### Требования к решению

Решение должно компилироваться под .NET 8 с использованием возможностей C# 12: top-level statements в `Program.cs`, `record` для модели, выражения-члены (`=>`) для коротких методов, pattern matching (`is null`) при проверке товара, `with`-выражения для создания копии записи с новым идентификатором. Контроллер и PageModel обязаны получать зависимости через конструктор — никакого ручного `new InMemoryProductRepository()` внутри action-методов. Все POST-формы должны быть защищены `[ValidateAntiForgeryToken]` (для MVC — атрибут на action, для Razor Pages — он применяется автоматически к `OnPost*`). В каждой ветке POST обязана быть проверка `ModelState.IsValid`, и при невалидных данных форма должна показываться повторно с сохранённым вводом и ошибками.

Представления должны быть сильно типизированными через `@model T` — `ViewBag` запрещён. В Razor Pages директива `@page` обязана быть первой строкой `.cshtml`; нарушите это — маршрут не зарегистрируется, и страница вернёт 404, как предостерегает урок. Используйте Tag Helpers (`asp-controller`, `asp-action`, `asp-route-id`, `asp-page`, `asp-page-handler`), а не `@Html.ActionLink`/`@Url.Action` — это соответствует best practice из урока о читаемости для верстальщиков. Код должен быть чистым: без SQL и бизнес-правил в контроллере, без `async void`, без захвата `HttpContext` в полях. Финальная сборка `dotnet build` и запуск `dotnet run` обязаны проходить без предупреждений компилятора.

#### Тонкости и подводные камни

- **Директива `@page` обязана быть первой строкой**. Если поставить перед ней комментарий `@* ... *@` или пустую строку с HTML, Razor Pages не зарегистрирует маршрут, и обращение к `/Contact/Index` вернёт 404. Это одна из самых частых ошибок из урока.
- **`@` переключает HTML↔C#, поэтому пробелы и переносы влияют на вывод**. `@p.Name` работает, а `@ p.Name` (с пробелом) — нет; сложные выражения оборачивайте в `@( ... )`, например `@(p.Price.ToString("C"))`.
- **Не путайте пары Tag Helpers**. В MVC используются `asp-controller`/`asp-action`/`asp-route-id`; в Razor Pages — `asp-page`/`asp-page-handler`. Скрестить их — получить сгенерированный пустой или неверный URL.
- **`[Bind]` vs `[BindProperty]`**. В MVC-контроллере уместно `Create([Bind("Name,Price,Category")] Product product)` или просто принять `Product` и проверять `ModelState`; в Razor Pages для автоматической привязки POST-данных к свойствам PageModel нужен `[BindProperty]` над свойством (привязка по умолчанию работает только для GET, если не указать `SupportsGet = true`).
- **Бизнес-логика не в контроллере**. Вычисление `nextId` через `_repo.GetAll().Count + 1` допустимо как минимум, но в реальном проекте идентификаторы должен назначать репозиторий или сервис. Не разрастайте action-метод логикой правил — выносите в домен.
- **`RedirectToAction(nameof(Index))` надёжнее строки `"Index"`**: переименование метода сломает сборку, а не runtime-маршрут. В Razor Pages аналогично — `RedirectToPage()` без аргумента перезагружает текущую страницу, `RedirectToPage("Index")` указывает имя явно.
- **`TempData` живёт один редирект**. Поэтому в `OnPostAsync` вы кладёте «Спасибо», делаете редирект, а в `OnGet`/разметке читаете и показываете — паттерн Post/Redirect/Get защищает от повторной отправки формы по F5.
- **Антифоргери-токен в Razor Pages добавляется автоматически**, но в MVC его надо явно: либо атрибут `[ValidateAntiForgeryToken]` на POST-action, либо автоматическая глобальная фильтра. Форма, сгенерированная Tag Helper `<form asp-controller=...>`, сама вставит скрытое поле `__RequestVerificationToken`, но атрибут всё равно нужен на методе.

#### Критерии приёмки

- [ ] Проект `MiniCatalog` создаётся командой `dotnet new mvc` и собирается без ошибок и предупреждений.
- [ ] В `Program.cs` зарегистрированы `AddControllersWithViews()`, `AddRazorPages()` и DI для `IProductRepository`.
- [ ] `Product` — это `record` с полями `Id`, `Name`, `Price`, `Category`.
- [ ] `IProductRepository` и `InMemoryProductRepository` реализованы; репозиторий зарегистрирован как singleton с тестовыми данными.
- [ ] `ProductsController` наследует `Controller`, внедряет репозиторий через конструктор и содержит `Index`, `Details`, `Create` (GET) и `Create` (POST).
- [ ] POST `Create` помечен `[HttpPost]` и `[ValidateAntiForgeryToken]`, проверяет `ModelState.IsValid`, при ошибке возвращает `View(product)`.
- [ ] Новый идентификатор вычисляется через `_repo.GetAll().Count + 1`, продукт добавляется через `product with { Id = nextId }`.
- [ ] `Views/Products/Index.cshtml`, `Details.cshtml`, `Create.cshtml` сильно типизированы через `@model` и используют `@foreach`, `@if`, Tag Helpers.
- [ ] В `Index.cshtml` ссылка «подробнее» использует `asp-route-id`, а «Добавить» — `asp-controller="Products" asp-action="Create"`.
- [ ] Razor Page `Pages/Contact/Index.cshtml` начинается с `@page` (первая строка), имеет `@model ContactIndexModel` и форму `method="post"`.
- [ ] `ContactIndexModel` содержит `[BindProperty]`-свойства, `OnGet` и `OnPostAsync`, логирует сообщение, использует `TempData` и `RedirectToPage()`.
- [ ] Обе навигационные ссылки добавлены в `_Layout.cshtml`: на каталог (MVC) и на контакты (Razor Pages).
- [ ] Контактная форма показывает сообщение «Спасибо за обращение!» после редиректа (Post/Redirect/Get).
- [ ] Файл `CHOICE.md` содержит обоснование выбора MVC для каталога и Razor Pages для контактов со ссылкой на критерии урока.
- [ ] `dotnet run` запускает приложение; каталог, детали, форма создания и контактная форма работают в браузере.

#### Подсказки (без прямого ответа)

- Вспомните аналогию урока: MVC — «шкаф с ящиками» (один контроллер, много actions), Razor Pages — «коробочки по одной на страницу». Подумайте, какая структура лучше для CRUD одной сущности и какая — для автономной формы.
- Если страница контактов возвращает 404 — проверьте, что `@page` действительно первая строка файла, без пробела или комментария перед ней.
- Если `ModelState` всегда невалиден — убедитесь, что имена полей формы совпадают со свойствами модели (включая регистр), и что для `decimal Price` локаль парсит запятую или точку.
- Для Tag Helper `asp-route-id` значение подставится в шаблон маршрута `{id?}`; без него ссылка на `Details` будет вести к `Details` без параметра и вернёт 404.
- `TempData` использует cookie или сессию; если после редиректа сообщение не появляется — проверьте, что `AddRazorPages()` и middleware `UseRouting()`/`MapRazorPages()` подключены.

#### Эталонное решение (разбор)

```csharp
// =============================================================
// M13-L08: Эталонное решение ДЗ — MiniCatalog
// Reference solution for the homework — MiniCatalog
// C# 12 / .NET 8
// =============================================================
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.RazorPages;

// --- Доменная модель / Domain model ---------------------------
public record Product(int Id, string Name, decimal Price, string Category);

// --- Репозиторий / Repository --------------------------------
public interface IProductRepository
{
    IReadOnlyList<Product> GetAll();
    Product? Get(int id);
    void Add(Product product);
}

public sealed class InMemoryProductRepository : IProductRepository
{
    private readonly List<Product> _items =
    [
        new(1, "Ноутбук X1", 95_000m, "Электроника"),
        new(2, "Кофемашина Pro", 28_500m, "Бытовая техника"),
        new(3, "Кресло офисное", 12_900m, "Мебель"),
    ]; // collection expression (C# 12)

    public IReadOnlyList<Product> GetAll() => _items;
    public Product? Get(int id) => _items.FirstOrDefault(p => p.Id == id);
    public void Add(Product product) => _items.Add(product);
}

// =============================================================
// 1) MVC-КОНТРОЛЛЕР / MVC CONTROLLER
//    Тонкий координатор: принимает запрос, делегирует репозиторию.
//    Thin coordinator: receives the request, delegates to the repo.
// =============================================================
public class ProductsController : Controller
{
    private readonly IProductRepository _repo;
    public ProductsController(IProductRepository repo) => _repo = repo;

    // GET /Products
    public IActionResult Index() => View(_repo.GetAll());

    // GET /Products/Details/5
    public IActionResult Details(int id)
    {
        var product = _repo.Get(id);
        // pattern matching: product is null
        if (product is null) return NotFound();
        return View(product);
    }

    // GET /Products/Create
    public IActionResult Create() => View();

    // POST /Products/Create
    [HttpPost]
    [ValidateAntiForgeryToken] // CSRF-защита / CSRF protection
    public IActionResult Create(Product product)
    {
        if (!ModelState.IsValid) return View(product); // повтор формы
        var nextId = _repo.GetAll().Count + 1;
        _repo.Add(product with { Id = nextId }); // with-выражение / with-expression
        return RedirectToAction(nameof(Index));
    }
}

// =============================================================
// 2) RAZOR PAGES — PageModel для контактов
//    PageModel for the contact page
// =============================================================
public class ContactIndexModel : PageModel
{
    private readonly ILogger<ContactIndexModel> _logger;
    public ContactIndexModel(ILogger<ContactIndexModel> logger) => _logger = logger;

    [BindProperty] public string? Name { get; set; }
    [BindProperty] public string? Message { get; set; }

    public void OnGet() { }

    public IActionResult OnPostAsync()
    {
        if (!ModelState.IsValid) return Page();
        _logger.LogInformation("Contact from {Name}: {Message}", Name, Message);
        TempData["Thanks"] = "Спасибо за обращение! / Thank you for your message!";
        return RedirectToPage(); // Post/Redirect/Get
    }
}
```

```html
@* Views/Products/Index.cshtml — MVC-представление / MVC view *@
@model IReadOnlyList<Product>

<h1>Каталог товаров / Product catalog</h1>
<table>
    <thead><tr><th>Название</th><th>Цена</th><th>Категория</th><th></th></tr></thead>
    <tbody>
        @foreach (var p in Model)
        {
            <tr>
                <td>@p.Name</td>
                <td>@p.Price.ToString("C")</td>
                <td>@p.Category</td>
                <td><a asp-controller="Products" asp-action="Details" asp-route-id="@p.Id">подробнее</a></td>
            </tr>
        }
    </tbody>
</table>
<a asp-controller="Products" asp-action="Create">Добавить товар</a>
```

```html
@* Pages/Contact/Index.cshtml — Razor Page *@
@page
@model ContactIndexModel

<h1>Обратная связь / Contact</h1>
@if (TempData["Thanks"] is string thanks)
{
    <p style="color:green">@thanks</p>
}
<form method="post">
    <input name="Name" placeholder="Ваше имя" />
    <textarea name="Message" placeholder="Сообщение"></textarea>
    <button type="submit">Отправить</button>
</form>
```

Разбор по строкам. `record Product` использует позиционную запись C# 12 — это даёт иммутабельность и авто-сгенерированные `Equals`/`GetHashCode`, что идеально для доменной модели и удобно для привязки формы. `InMemoryProductRepository` инициализирует список через collection expression `[ ... ]` (новинка C# 12), а методы оформлены выражениями-членами `=>` — коротко и читаемо. `Get(int id)` возвращает `Product?` и использует `FirstOrDefault`, что соответствует nullable-семантике: вызывающая сторона обязана проверить результат (здесь — pattern matching `is null` в `Details`). Контроллер `ProductsController : Controller` — это канонический MVC-контроллер из урока: он наследует `Controller`, получает репозиторий через конструктор (DI), и каждый action-метод тонок — только координация. `Index` сразу возвращает `View(_repo.GetAll())`, передавая модель в представление; `Details` проверяет `product is null` и возвращает `NotFound()` (HTTP 404), что демонстрирует обработку отсутствующего ресурса. POST `Create` помечен `[HttpPost]` (ограничивает HTTP-метод) и `[ValidateAntiForgeryToken]` (CSRF-защита из best practices урока); проверка `!ModelState.IsValid` возвращает `View(product)`, чтобы показать форму повторно с ошибками — это та самая частая ошибка «игнорирование `ModelState`», которую урок просит избегать. `product with { Id = nextId }` использует `with`-выражение для иммутабельной записи: создаётся копия с назначенным идентификатором, исходный объект не меняется. `RedirectToAction(nameof(Index))` реализует Post/Redirect/Get и защищает от повторной отправки по F5. В `ContactIndexModel : PageModel` — page-first подход: свойства помечены `[BindProperty]`, поэтому привязка из POST работает автоматически (без этого свойства остались бы `null`). `OnGet` пустой — страница просто рендерится; `OnPostAsync` проверяет `ModelState`, логирует через внедрённый `ILogger` (DI), кладёт сообщение в `TempData` и делает `RedirectToPage()`. `TempData` переживает ровно один редирект, поэтому благодарность показывается после перехода, а не сразу на POST — это и есть паттерн Post/Redirect/Get. В `.cshtml` первой строкой идёт `@page` — без неё маршрут не зарегистрируется, как подчёркивает урок. `@model ContactIndexModel` даёт сильную типизацию. Условие `@if (TempData["Thanks"] is string thanks)` иллюстрирует pattern matching с объявлением переменной прямо в разметке. Таким образом, в одном проекте сосуществуют обе модели урока: MVC (controller-first) для каталога со многими действиями над ресурсом и Razor Pages (page-first) для автономной формы, и обе пользуются общим Razor-движком, Tag Helpers и DI.

#### Задания на углубление (бонус)

1. **Валидация аннотациями.** Добавьте на свойства `Product` атрибуты `[Required]`, `[Range(0.01, 1_000_000)]` для `Price` и `[StringLength(100)]` для `Name`, и убедитесь, что `ModelState` ловит нарушения. Сравните поведение с ручной проверкой в action.
2. **Асинхронный репозиторий.** Преобразуйте `IProductRepository` в асинхронный (`Task<IReadOnlyList<Product>> GetAllAsync()` и т. п.), имитируя задержку `Task.Delay`. Контроллер и PageModel станут `async`, action — `Task<IActionResult>`, а `OnPostAsync` уже соответствует. Проследите, чтобы не было `async void`.
3. **Маршрутизация атрибутами.** Перепишите `ProductsController` на attribute routing: `[Route("products")]` на классе и `[Route("")]`, `[Route("{id:int}")]` на методах. Сравните читаемость с conventional routing из шаблона.
4. **Смешивание моделей.** Добавьте ещё одну Razor Page `Pages/Products/Index.cshtml`, которая показывает тот же каталог, но через PageModel, и сравните объём кода с MVC-вариантом. Запишите наблюдения в `CHOICE.md`.

---

## Statement in English / Постановка на английском

#### Context & motivation

You are a backend developer in a small team rolling out an internal "Mini-Catalog" portal for the procurement department. The portal must show a list of products, open a page with product details, and accept new entries through a form. In parallel you need a "Contact" section — a simple contact form that logs a message and thanks the user. Your tech lead asked you not to duplicate the same code, but to consciously apply both models covered in lesson M13-L08: MVC for the catalog (where many operations over a single resource group inside one controller) and Razor Pages for the contact form (where the page's logic is self-contained). This gives the team a living example for future decisions: "when MVC, and when Razor Pages".

The lesson emphasizes that both models are technically peers and run on the same routing engine and Razor view engine, so the choice is mostly about project style rather than capabilities. You must verify this in practice: the same Tag Helpers, the same `@` syntax, the same dependency injection mechanism. Along the way you will practice the common mistakes from the lesson: a forgotten `@page` directive, business logic directly in the controller, using `ViewBag` instead of a strongly typed model, mixing up `asp-controller/asp-action` and `asp-page/asp-page-handler`, ignoring `ModelState.IsValid`. By the end you will have a compilable application ready to run locally and a clear understanding of which approach to choose when.

#### What to do step by step

1. **Create the project.** From the `modules/M13/lessons` folder, run the command to create a new MVC application that by default already supports both controllers and Razor Pages:

   ```bash
   dotnet new mvc -n MiniCatalog -o MiniCatalog --framework net8.0
   cd MiniCatalog
   ```

   Make sure `Program.cs` registers `AddControllersWithViews()` and `AddRazorPages()` (the `mvc` template already wires both, adds `MapControllerRoute` with the `{controller=Home}/{action=Index}/{id?}` pattern, and adds `MapRazorPages`). Run `dotnet build` — it must succeed without errors or warnings.

2. **Describe the domain model and repository.** In the `Models` folder create `Product.cs` with `record Product(int Id, string Name, decimal Price, string Category)`. Then add the `IProductRepository` interface with methods `IReadOnlyList<Product> GetAll()`, `Product? Get(int id)`, and `void Add(Product product)`, plus an `InMemoryProductRepository` implementation that keeps products in a `List<Product>` and seeds three or four test records at startup. In `Program.cs` register the repository as a singleton: `builder.Services.AddSingleton<IProductRepository, InMemoryProductRepository>();`. This is exactly the dependency that the controller and PageModel will receive through their constructors.

3. **Implement the MVC catalog controller.** In `Controllers/ProductsController.cs` create a class `ProductsController : Controller`. Inject `IProductRepository` through the constructor and store it in a read-only field. Add three action methods: `Index()` — returns `View(_repo.GetAll())`; `Details(int id)` — looks up the product, returns `NotFound()` if missing or `View(product)`; `Create()` (GET) — returns an empty form; `Create(Product product)` (POST) — decorated with `[HttpPost]` and `[ValidateAntiForgeryToken]`, checks `ModelState.IsValid`, returns `View(product)` on error, otherwise computes the new id (`_repo.GetAll().Count + 1`), adds the product via `product with { Id = nextId }`, and does `RedirectToAction(nameof(Index))`. The controller must stay a thin coordinator — no business logic beyond delegating to the repository.

4. **Create strongly typed views.** In `Views/Products/` create `Index.cshtml` with the directive `@model IReadOnlyList<MiniCatalog.Models.Product>`, a heading `<h1>Product catalog</h1>`, and a table built with `@foreach (var p in Model)`. The "Details" column must use the Tag Helper `<a asp-controller="Products" asp-action="Details" asp-route-id="@p.Id">details</a>`, and the "Add" link must use `<a asp-controller="Products" asp-action="Create">`. The `Details.cshtml` file accepts `@model Product` and shows all fields. The `Create.cshtml` file accepts `@model Product`, uses `<form asp-controller="Products" asp-action="Create" method="post">` with `Name`, `Price`, `Category` fields and a submit button; do not forget `@Html.AntiForgeryToken()` or rely on the form Tag Helper to emit the token automatically.

5. **Add a Razor Page for contact.** In the `Pages/Contact/` folder create a pair of files: `Index.cshtml` and `Index.cshtml.cs`. In the code-behind, the `ContactIndexModel : PageModel` class must inject `ILogger<ContactIndexModel>`, hold properties `[BindProperty] public string? Name { get; set; }` and `[BindProperty] public string? Message { get; set; }`, a parameterless `OnGet()` method, and an `OnPostAsync()` method that checks `ModelState.IsValid`, logs the message via `_logger.LogInformation("Contact from {Name}: {Message}", Name, Message)`, sets `TempData["Thanks"] = "Thank you for your message!"`, and returns `RedirectToPage()`. In `.cshtml` the first line must be `@page`, then `@model ContactIndexModel`, a `<form method="post">` with `asp-page-handler` (or just `method="post"`, in which case `OnPostAsync` fires), and a render of `TempData["Thanks"]` when present.

6. **Wire navigation and verify.** In `Views/Shared/_Layout.cshtml` add two menu links: to the catalog (`asp-controller="Products" asp-action="Index"`) and to the contact page (`asp-page="/Contact/Index"`). Run `dotnet run`, open `https://localhost:<port>`, and confirm both branches work: the catalog shows the list, "details" opens a product, the create form accepts a POST and redirects back to the list, and the contact form writes to the console log and shows a thank-you message.

7. **Explain the choice.** In a `CHOICE.md` file at the project root write 5–7 sentences in English on why MVC was chosen for the catalog and Razor Pages for the contact form, referencing the lesson's criteria.

#### Requirements

The solution must compile under .NET 8 using C# 12 features: top-level statements in `Program.cs`, `record` for the model, expression-bodied members (`=>`) for short methods, pattern matching (`is null`) when checking the product, and `with`-expressions to create a copy of the record with a new id. The controller and PageModel must receive dependencies through the constructor — no manual `new InMemoryProductRepository()` inside action methods. All POST forms must be protected with `[ValidateAntiForgeryToken]` (for MVC — the attribute on the action; for Razor Pages — it is applied automatically to `OnPost*`). Every POST branch must check `ModelState.IsValid`, and on invalid data the form must be re-shown with the preserved input and errors.

Views must be strongly typed through `@model T` — `ViewBag` is forbidden. In Razor Pages the `@page` directive must be the first line of the `.cshtml` file; violate this and the route will not register and the page will return 404, as the lesson warns. Use Tag Helpers (`asp-controller`, `asp-action`, `asp-route-id`, `asp-page`, `asp-page-handler`) rather than `@Html.ActionLink`/`@Url.Action` — this matches the lesson's best practice about readability for designers. The code must be clean: no SQL or business rules in the controller, no `async void`, no capturing `HttpContext` in fields. The final `dotnet build` and `dotnet run` must pass without compiler warnings.

#### Pitfalls

- **The `@page` directive must be the very first line.** If you put a `@* ... *@` comment or an HTML line before it, Razor Pages will not register the route and `/Contact/Index` will return 404. This is one of the most frequent mistakes from the lesson.
- **`@` toggles HTML↔C#, so whitespace and line breaks affect output.** `@p.Name` works, `@ p.Name` (with a space) does not; wrap complex expressions in `@( ... )`, for example `@(p.Price.ToString("C"))`.
- **Do not mix up Tag Helper pairs.** MVC uses `asp-controller`/`asp-action`/`asp-route-id`; Razor Pages uses `asp-page`/`asp-page-handler`. Cross them and you get an empty or wrong URL.
- **`[Bind]` vs `[BindProperty]`.** In an MVC controller `Create([Bind("Name,Price,Category")] Product product)` is appropriate, or simply accept `Product` and check `ModelState`; in Razor Pages, for automatic binding of POST data to PageModel properties you need `[BindProperty]` on the property (binding works for GET only unless you set `SupportsGet = true`).
- **No business logic in the controller.** Computing `nextId` via `_repo.GetAll().Count + 1` is acceptable as a minimum, but in a real project ids should be assigned by the repository or a service. Do not grow the action method with rule logic — move it to the domain.
- **`RedirectToAction(nameof(Index))` is safer than the string `"Index"`**: renaming the method will break the build, not a runtime route. In Razor Pages `RedirectToPage()` without an argument reloads the current page, `RedirectToPage("Index")` names it explicitly.
- **`TempData` lives for exactly one redirect.** So in `OnPostAsync` you store "Thank you", redirect, and in `OnGet`/markup you read and display it — the Post/Redirect/Get pattern protects against re-submitting the form on F5.
- **The anti-forgery token is added automatically in Razor Pages**, but in MVC you need it explicitly: either the `[ValidateAntiForgeryToken]` attribute on the POST action, or an automatic global filter. A form generated by the `<form asp-controller=...>` Tag Helper will itself emit the hidden `__RequestVerificationToken` field, but the attribute is still required on the method.

#### Acceptance criteria

- [ ] The `MiniCatalog` project is created with `dotnet new mvc` and builds without errors or warnings.
- [ ] `Program.cs` registers `AddControllersWithViews()`, `AddRazorPages()`, and DI for `IProductRepository`.
- [ ] `Product` is a `record` with fields `Id`, `Name`, `Price`, `Category`.
- [ ] `IProductRepository` and `InMemoryProductRepository` are implemented; the repository is registered as a singleton with seed data.
- [ ] `ProductsController` derives from `Controller`, injects the repository through the constructor, and contains `Index`, `Details`, `Create` (GET), and `Create` (POST).
- [ ] POST `Create` is decorated with `[HttpPost]` and `[ValidateAntiForgeryToken]`, checks `ModelState.IsValid`, and returns `View(product)` on error.
- [ ] The new id is computed via `_repo.GetAll().Count + 1`, and the product is added via `product with { Id = nextId }`.
- [ ] `Views/Products/Index.cshtml`, `Details.cshtml`, `Create.cshtml` are strongly typed through `@model` and use `@foreach`, `@if`, and Tag Helpers.
- [ ] In `Index.cshtml` the "details" link uses `asp-route-id`, and the "Add" link uses `asp-controller="Products" asp-action="Create"`.
- [ ] The Razor Page `Pages/Contact/Index.cshtml` starts with `@page` (first line), has `@model ContactIndexModel`, and a `method="post"` form.
- [ ] `ContactIndexModel` contains `[BindProperty]` properties, `OnGet`, and `OnPostAsync`, logs the message, uses `TempData`, and `RedirectToPage()`.
- [ ] Both navigation links are added to `_Layout.cshtml`: to the catalog (MVC) and to the contact page (Razor Pages).
- [ ] The contact form shows a "Thank you for your message!" line after the redirect (Post/Redirect/Get).
- [ ] The `CHOICE.md` file explains the choice of MVC for the catalog and Razor Pages for the contact form, referencing the lesson's criteria.
- [ ] `dotnet run` starts the app; the catalog, details, the create form, and the contact form all work in the browser.

#### Hints (no direct answer)

- Recall the lesson's analogy: MVC is a "cabinet with drawers" (one controller, many actions), Razor Pages is "a box per page". Think about which structure fits CRUD on a single entity and which fits an autonomous form.
- If the contact page returns 404, check that `@page` is truly the first line of the file, with no leading space or comment.
- If `ModelState` is always invalid, make sure the form field names match the model properties (including case), and that the `decimal Price` is parsed with the right decimal separator for your locale.
- For the `asp-route-id` Tag Helper, the value is substituted into the `{id?}` route template; without it the `Details` link will point to `Details` without a parameter and return 404.
- `TempData` relies on a cookie or session; if the message does not appear after the redirect, verify that `AddRazorPages()` and the `UseRouting()`/`MapRazorPages()` middleware are wired up.

#### Reference solution walk-through

```csharp
// =============================================================
// M13-L08: Reference solution for the homework — MiniCatalog
// C# 12 / .NET 8
// =============================================================
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.RazorPages;

// --- Domain model -------------------------------------------
public record Product(int Id, string Name, decimal Price, string Category);

// --- Repository ---------------------------------------------
public interface IProductRepository
{
    IReadOnlyList<Product> GetAll();
    Product? Get(int id);
    void Add(Product product);
}

public sealed class InMemoryProductRepository : IProductRepository
{
    private readonly List<Product> _items =
    [
        new(1, "Laptop X1", 95_000m, "Electronics"),
        new(2, "Coffee Machine Pro", 28_500m, "Appliances"),
        new(3, "Office Chair", 12_900m, "Furniture"),
    ]; // collection expression (C# 12)

    public IReadOnlyList<Product> GetAll() => _items;
    public Product? Get(int id) => _items.FirstOrDefault(p => p.Id == id);
    public void Add(Product product) => _items.Add(product);
}

// =============================================================
// 1) MVC CONTROLLER
//    Thin coordinator: receives the request, delegates to the repo.
// =============================================================
public class ProductsController : Controller
{
    private readonly IProductRepository _repo;
    public ProductsController(IProductRepository repo) => _repo = repo;

    // GET /Products
    public IActionResult Index() => View(_repo.GetAll());

    // GET /Products/Details/5
    public IActionResult Details(int id)
    {
        var product = _repo.Get(id);
        if (product is null) return NotFound(); // pattern matching
        return View(product);
    }

    // GET /Products/Create
    public IActionResult Create() => View();

    // POST /Products/Create
    [HttpPost]
    [ValidateAntiForgeryToken] // CSRF protection
    public IActionResult Create(Product product)
    {
        if (!ModelState.IsValid) return View(product); // re-show form
        var nextId = _repo.GetAll().Count + 1;
        _repo.Add(product with { Id = nextId }); // with-expression
        return RedirectToAction(nameof(Index));
    }
}

// =============================================================
// 2) RAZOR PAGES — PageModel for the contact page
// =============================================================
public class ContactIndexModel : PageModel
{
    private readonly ILogger<ContactIndexModel> _logger;
    public ContactIndexModel(ILogger<ContactIndexModel> logger) => _logger = logger;

    [BindProperty] public string? Name { get; set; }
    [BindProperty] public string? Message { get; set; }

    public void OnGet() { }

    public IActionResult OnPostAsync()
    {
        if (!ModelState.IsValid) return Page();
        _logger.LogInformation("Contact from {Name}: {Message}", Name, Message);
        TempData["Thanks"] = "Thank you for your message!";
        return RedirectToPage(); // Post/Redirect/Get
    }
}
```

```html
@* Views/Products/Index.cshtml — MVC view *@
@model IReadOnlyList<Product>

<h1>Product catalog</h1>
<table>
    <thead><tr><th>Name</th><th>Price</th><th>Category</th><th></th></tr></thead>
    <tbody>
        @foreach (var p in Model)
        {
            <tr>
                <td>@p.Name</td>
                <td>@p.Price.ToString("C")</td>
                <td>@p.Category</td>
                <td><a asp-controller="Products" asp-action="Details" asp-route-id="@p.Id">details</a></td>
            </tr>
        }
    </tbody>
</table>
<a asp-controller="Products" asp-action="Create">Add product</a>
```

```html
@* Pages/Contact/Index.cshtml — Razor Page *@
@page
@model ContactIndexModel

<h1>Contact</h1>
@if (TempData["Thanks"] is string thanks)
{
    <p style="color:green">@thanks</p>
}
<form method="post">
    <input name="Name" placeholder="Your name" />
    <textarea name="Message" placeholder="Message"></textarea>
    <button type="submit">Send</button>
</form>
```

Line-by-line walk-through. The `record Product` uses a positional record in C# 12 — this gives immutability and auto-generated `Equals`/`GetHashCode`, ideal for a domain model and convenient for form binding. `InMemoryProductRepository` initializes the list with a collection expression `[ ... ]` (a C# 12 feature), and the methods are expression-bodied `=>` — short and readable. `Get(int id)` returns `Product?` and uses `FirstOrDefault`, matching nullable semantics: the caller must check the result (here — the `is null` pattern in `Details`). The `ProductsController : Controller` is the canonical MVC controller from the lesson: it derives from `Controller`, receives the repository through the constructor (DI), and every action method is thin — only coordination. `Index` immediately returns `View(_repo.GetAll())`, passing the model to the view; `Details` checks `product is null` and returns `NotFound()` (HTTP 404), demonstrating missing-resource handling. The POST `Create` is decorated with `[HttpPost]` (restricting the HTTP method) and `[ValidateAntiForgeryToken]` (CSRF protection from the lesson's best practices); the `!ModelState.IsValid` check returns `View(product)` to re-show the form with errors — exactly the "ignoring `ModelState`" mistake the lesson tells you to avoid. The `product with { Id = nextId }` line uses a `with`-expression on an immutable record: a copy is created with the assigned id, and the original object is untouched. `RedirectToAction(nameof(Index))` implements Post/Redirect/Get and guards against re-submission on F5. In `ContactIndexModel : PageModel` we see the page-first approach: the properties are decorated with `[BindProperty]`, so binding from POST works automatically (without it the properties would stay `null`). `OnGet` is empty — the page just renders; `OnPostAsync` checks `ModelState`, logs through the injected `ILogger` (DI), puts the message into `TempData`, and calls `RedirectToPage()`. `TempData` survives exactly one redirect, so the thank-you note appears after the transition rather than immediately on the POST — that is the Post/Redirect/Get pattern. In the `.cshtml` file the first line is `@page` — without it the route does not register, as the lesson stresses. `@model ContactIndexModel` gives strong typing. The condition `@if (TempData["Thanks"] is string thanks)` shows pattern matching with a variable declaration right in the markup. Thus both lesson models coexist in one project: MVC (controller-first) for the catalog with many actions over a resource, and Razor Pages (page-first) for an autonomous form — and both share the same Razor engine, Tag Helpers, and DI.

#### Going deeper (bonus)

1. **Validation with attributes.** Add `[Required]`, `[Range(0.01, 1_000_000)]` for `Price`, and `[StringLength(100)]` for `Name` on the `Product` properties, and confirm that `ModelState` catches violations. Compare with manual checks inside the action.
2. **Async repository.** Turn `IProductRepository` into an async version (`Task<IReadOnlyList<Product>> GetAllAsync()` and so on), simulating a delay with `Task.Delay`. The controller and PageModel become `async`, the action becomes `Task<IActionResult>`, and `OnPostAsync` already fits. Make sure there is no `async void`.
3. **Attribute routing.** Rewrite `ProductsController` with attribute routing: `[Route("products")]` on the class and `[Route("")]`, `[Route("{id:int}")]` on methods. Compare readability with the conventional routing from the template.
4. **Mixing the models.** Add another Razor Page `Pages/Products/Index.cshtml` that shows the same catalog through a PageModel, and compare the amount of code with the MVC variant. Record your observations in `CHOICE.md`.

---

#### Чек-лист сдачи / Submission checklist

- [ ] Проект `MiniCatalog` создан и собирается без предупреждений.
- [ ] `Program.cs` регистрирует MVC, Razor Pages и `IProductRepository`.
- [ ] Модель `Product` — `record`; репозиторий — singleton с тестовыми данными.
- [ ] `ProductsController` содержит `Index`, `Details`, `Create` (GET/POST), POST защищён и проверяет `ModelState`.
- [ ] Представления сильно типизированы, используют Tag Helpers и `@foreach`/`@if`.
- [ ] Razor Page `Contact/Index` начинается с `@page`, имеет `OnGet`/`OnPostAsync`, `TempData` и `RedirectToPage()`.
- [ ] Навигация в `_Layout.cshtml` ведёт и на MVC-каталог, и на Razor-страницу.
- [ ] Файл `CHOICE.md` обосновывает выбор моделей по критериям урока.
- [ ] `dotnet run` запускается; все ветки работают в браузере.
- [ ] The `MiniCatalog` project is created and builds without warnings.
- [ ] `Program.cs` registers MVC, Razor Pages, and `IProductRepository`.
- [ ] The `Product` model is a `record`; the repository is a singleton with seed data.
- [ ] `ProductsController` contains `Index`, `Details`, `Create` (GET/POST); POST is protected and checks `ModelState`.
- [ ] Views are strongly typed, use Tag Helpers and `@foreach`/`@if`.
- [ ] The `Contact/Index` Razor Page starts with `@page`, has `OnGet`/`OnPostAsync`, `TempData`, and `RedirectToPage()`.
- [ ] Navigation in `_Layout.cshtml` links to both the MVC catalog and the Razor page.
- [ ] `CHOICE.md` justifies the model choices per the lesson's criteria.
- [ ] `dotnet run` starts; all branches work in the browser.

#### Ресурсы / Resources

- [Microsoft Learn — Overview of ASP.NET Core MVC](https://learn.microsoft.com/aspnet/core/mvc/overview)
- [Microsoft Learn — Introduction to Razor Pages in ASP.NET Core](https://learn.microsoft.com/aspnet/core/razor-pages/)
- [Microsoft Learn — Tag Helpers in ASP.NET Core](https://learn.microsoft.com/aspnet/core/mvc/views/tag-helpers/intro)
- [Microsoft Learn — Routing in ASP.NET Core](https://learn.microsoft.com/aspnet/core/fundamentals/routing)
- [Microsoft Learn — Prevent Cross-Site Request Forgery (XSRF/CSRF)](https://learn.microsoft.com/aspnet/core/security/anti-request-forgery)
