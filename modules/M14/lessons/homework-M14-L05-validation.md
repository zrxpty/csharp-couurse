---
[← К уроку M14-L05](lesson-M14-L05-validation.md) | [⬆ К модулю M14](../README.md) | [Следующее ДЗ →](homework-M14-L06-problemdetails-errors.md)
---

### Домашнее задание M14-L05: Валидация, DataAnnotations, FluentValidation (Optional) / Homework M14-L05: Validation, DataAnnotations, FluentValidation (Optional)

**Урок / Lesson:** M14-L05
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) На практике освоить двухуровневую валидацию входных данных в ASP.NET Core 8: декларативные правила через `DataAnnotations` для формата, явные условные правила через `FluentValidation` для бизнес-ограничений, и единый контракт ошибок на базе RFC 7807 Problem Details. / (EN) Gain hands-on mastery of two-level input validation in ASP.NET Core 8: declarative format rules with `DataAnnotations`, explicit conditional rules with `FluentValidation` for business constraints, and a uniform error contract based on RFC 7807 Problem Details.

#### Связь с уроком / Connection to the lesson

(RU) Урок вводит пропускную модель валидации: автоматическую через атрибуты `DataAnnotations` и явную через `FluentValidation`, с единым ответом `ValidationProblem` по RFC 7807. Это ДЗ заставит вас собрать оба уровня в одном эндпоинте, столкнуться с типичными ошибками из урока (`[Required]` на значимом типе, потеря `CancellationToken`, дублирование правил, возврат `BadRequest(string)`) и получить корректный `application/problem+json` во всех ветках.

(EN) The lesson introduces a checkpoint model of validation: automatic through `DataAnnotations` attributes and explicit through `FluentValidation`, with a single `ValidationProblem` response following RFC 7807. This homework makes you combine both levels in one endpoint, hit the typical mistakes from the lesson (`[Required]` on a value type, lost `CancellationToken`, duplicated rules, returning `BadRequest(string)`), and produce a correct `application/problem+json` payload in every branch.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Вы бэкенд-разработчик платформы конференций «DotNextHub». Спикеры подают заявки на доклады через публичный эндпоинт `POST /api/proposals`. Заявка содержит данные о спикере (email, имя, биография), о докладе (заголовок, аннотация, формат, длительность, теги) и опционально о со-спикере. Данные разнородны: часть — простой формат (email, длина строки), часть — условные бизнес-правила (длительность зависит от формата доклада, со-спикер не может совпадать с основным спикером, теги должны быть уникальны регистронезависимо). Кроме того, в домене есть инвариант: email спикера не должен числиться в «чёрном списке» заблокированных аккаунтов — это правило нельзя выразить атрибутами, оно проверяется в бизнес-логике и зависит от внешнего состояния.

Урок показал, что атрибуты `DataAnnotations` удобны для простых проверок формата, но захламляют доменную модель техническими деталями и не справляются с контекстом. `FluentValidation` решает сложные правила ценой отдельного класса-валидатора и регистрации в DI. Важно не дублировать одно и то же правило в обоих местах (иначе рассинхрон при изменении) и возвращать ошибки единым контрактом RFC 7807. В этом задании вы построите эндпоинт, где формат валидируется атрибутами, сложные и условные правила — `FluentValidation`, а доменный инвариант — в сервисе, и всё это сходится к одному `ValidationProblem`-ответу. Вы также столкнётесь с тем, что `[ApiController]` автоматически отклоняет невалидные по атрибутам запросы ещё до вашего кода, поэтому явный вызов валидатора нужно планировать осознанно.

#### Что нужно сделать (пошагово)

1. Создайте решение и веб-проект. Из корня репозитория выполните:
   ```
   dotnet new web -n DotNextHub.Api -o DotNextHub.Api
   cd DotNextHub.Api
   dotnet add package FluentValidation.AspNetCore --version 11.3.0
   dotnet add package Microsoft.AspNetCore.OpenApi --version 8.0.*
   ```
   Ожидаемый результат: каталог `DotNextHub.Api` с `Program.cs` и `DotNextHub.Api.csproj`, в котором есть `PackageReference` на `FluentValidation.AspNetCore`. Включите `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` и `<Nullable>enable</Nullable>` в `csproj`.

2. Определите enum и DTO. Создайте файл `Models/TalkProposalRequest.cs` с `record TalkProposalRequest`, перечислением `TalkFormat { Talk, Workshop, Panel }` и свойствами: `SpeakerEmail`, `SpeakerName`, `Bio`, `Title`, `Abstract`, `Format`, `Duration`, `Tags`, `CoSpeakerEmail`. Покройте простые правила атрибутами `DataAnnotations`: `SpeakerEmail` — `[Required][EmailAddress]`; `SpeakerName` — `[Required][StringLength(100, MinimumLength=2)]`; `Bio` — `[StringLength(500)]`; `Title` — `[Required][StringLength(120, MinimumLength=5)]`; `Abstract` — `[Required][StringLength(1000, MinimumLength=20)]`; `Format` — `[EnumDataType(typeof(TalkFormat))]`. КРИТИЧНО: `Duration` сделайте `int?`, а не `int`, — `[Required]` на значимом типе бесполезен (он не бывает `null`), это прямая ошибка из урока. `Tags` — `IReadOnlyList<string>?`.

3. Создайте `Validators/TalkProposalRequestValidator.cs`, унаследованный от `AbstractValidator<TalkProposalRequest>`. Опишите условные правила: `When(Format == Talk)` → `Duration` обязан быть `30 or 45`; `Workshop` → `120 or 180`; `Panel` → `60`. `CoSpeakerEmail`, если задан, должен быть валидным email и не равен `SpeakerEmail` (`NotEqual`). Каждый тег — 2–30 символов, тегов 1–5, регистронезависимо уникальны (`Distinct(StringComparer.OrdinalIgnoreCase)`). Не дублируйте email-формат `SpeakerEmail` — он уже покрыт атрибутом `[EmailAddress]`. Сообщения локализуйте RU/EN.

4. Зарегистрируйте валидаторы и настройте Problem Details в `Program.cs` (top-level statements): `AddControllers().ConfigureApiBehaviorOptions(...)` с кастомным `InvalidModelStateResponseFactory`, возвращающим `ValidationProblemDetails` с `Title` и `Status = 400`. Зарегистрируйте `AddValidatorsFromAssemblyContaining<TalkProposalRequestValidator>()`.

5. Реализуйте `ProposalsController` с `[ApiController][Route("api/[controller]")]` и эндпоинтом `POST`. Внутри: явно вызовите `_validator.ValidateAsync(request, ct)`, при ошибках сложите их в `ModelState` через `AddModelError` и верните `ValidationProblem(ModelState)` (НЕ `BadRequest("...")`). Затем вызовите доменный сервис `IBannedSpeakerService.IsBannedAsync(email, ct)` — если заблокирован, верните `Problem(..., statusCode: 409)`. Иначе `Ok(...)`.

6. Реализуйте `IBannedSpeakerService` и `BannedSpeakerService` с захардкоженным множеством заблокированных email (например, `banned@dotnexthub.dev`), используя `HashSet<string>` с `StringComparer.OrdinalIgnoreCase`.

7. Запустите `dotnet run` и проверьте `curl`-запросами три сценария: (a) пустой `SpeakerEmail` → `400` с `errors.SpeakerEmail`; (b) `Format=Workshop, Duration=30` → `400` с `errors.Duration`; (c) корректная заявка → `200`.

8. Напишите unit-тесты xUnit на валидатор: `ShouldPass_WhenValidTalk`, `ShouldFail_WhenWorkshopDurationIs30`, `ShouldFail_WhenTagsAreDuplicates`. Ожидаемый вывод: 3 теста passed.

#### Требования к решению

- Целевой фреймворк `net8.0`, C# 12: разрешён top-level `Program.cs`, pattern matching (`is 30 or 45`), collection expressions (`["a","b"]`), `init`-свойства.
- Все входные DTO — `record` с позиционными параметрами; сложные и условные правила — только в `FluentValidation`, простые — в атрибутах. Не дублируйте одно и то же правило в обоих местах: email-формат `SpeakerEmail` задан атрибутом `[EmailAddress]`, а в валидаторе его НЕ повторяют — там только `NotEqual`/`Must` для `CoSpeakerEmail`.
- Контроллер помечен `[ApiController]`; ответ на ошибку валидации — `ValidationProblem` (не `BadRequest("string")`). Доменный инвариант (блокировка) возвращает `409 ProblemDetails` через `Problem(...)`.
- `FluentValidation`: используйте `When(...)`, `Must(...)`, `NotEqual(...)`, `WithMessage(...)`. Обязательно передавайте `CancellationToken` в `ValidateAsync` — иначе утечка ресурсов под нагрузкой (ошибка из урока).
- Локализация: хотя бы два сообщения явно двуязычные (`"RU / EN"`), как в примере урока.
- Покрытие unit-тестами валидатора: минимум 3 кейса (pass, fail, conditional).
- Код компилируется без warning благодаря `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>`.

#### Тонкости и подводные камни

- `[Required]` на `int` бесполезен: значимый тип не бывает `null`. Делайте `Duration` типа `int?` и проверяйте в `FluentValidation` через `NotNull().Must(...)`. Урок явно предупреждает об этой ошибке.
- `[ApiController]` автоматически возвращает `ValidationProblem` по атрибутам ещё до вашего кода: если атрибуты отклонили запрос, до явного вызова `FluentValidation` дело не дойдёт. Решение: не пересекайте правила — формат в атрибутах, условную логику в валидаторе.
- Игнорирование `CancellationToken` в `ValidateAsync` — частая утечка ресурсов под нагрузкой; всегда пробрасывайте `ct` из сигнатуры действия.
- Дублирование правил (email в атрибуте И в `FluentValidation`) — источник рассинхрона и багов при изменении; выберите один источник правды для каждой проверки.
- Не возвращайте `BadRequest("text")` — это ломает контракт RFC 7807 и заставляет клиента парсить разные форматы ошибок.
- `Must(...)` в `FluentValidation` получает весь объект — удобно для перекрёстных правил (`CoSpeakerEmail != SpeakerEmail`), но не забудьте `WithMessage`, иначе клиент получит дефолтное непонятное сообщение.
- `When(...)` по умолчанию пропускает правила, если условие ложно — это правильное поведение для опциональных полей (`CoSpeakerEmail`, `Tags`), но следите, чтобы обязательные проверки не оказались «проглочены» условием.
- Collection expressions (C# 12) `Tags = ["architecture","testing"]` удобны в тестах, но в DTO оставьте `IReadOnlyList<string>?`.
- Уникальность тегов регистронезависимо — `StringComparer.OrdinalIgnoreCase`; `Distinct` без компарера даст ложное неуникальное поведение для `dotnet`/`DotNet`.
- `ValidationProblemDetails` (из `Microsoft.AspNetCore.Mvc`) имеет словарь `Errors`, который заполняется из `ModelState`; `Title` и `Status` задаются вручную в фабрике.

#### Критерии приёмки

- [ ] Проект `DotNextHub.Api` собирается `dotnet build` без ошибок и warning.
- [ ] `TalkProposalRequest` — `record` с позиционными параметрами и атрибутами `DataAnnotations` для простых правил.
- [ ] `Duration` имеет тип `int?` (не `int`), валидируется в `FluentValidation` через `NotNull().Must(...)`.
- [ ] `TalkProposalRequestValidator` содержит условные правила через `When`/`Must` для формата доклада.
- [ ] `CoSpeakerEmail` проверяется на валидность и неравенство `SpeakerEmail` (`NotEqual`), только если задан.
- [ ] Теги проверяются на длину каждого тега, количество 1–5 и регистронезависимую уникальность.
- [ ] Валидаторы зарегистрированы через `AddValidatorsFromAssemblyContaining<TalkProposalRequestValidator>()`.
- [ ] `ConfigureApiBehaviorOptions` задаёт кастомный `InvalidModelStateResponseFactory` с `ValidationProblemDetails`.
- [ ] Контроллер помечен `[ApiController]`; ошибки валидации — `ValidationProblem`, не `BadRequest(string)`.
- [ ] Доменный инвариант (блокировка спикера) возвращает `409` через `Problem(...)`.
- [ ] `CancellationToken` пробрасывается в `ValidateAsync` и доменный сервис.
- [ ] Минимум 3 unit-теста xUnit на валидатор (pass/fail/conditional) — все зелёные.
- [ ] `curl`-сценарии (a)(b)(c) возвращают ожидаемые статусы и JSON в формате `application/problem+json`.
- [ ] Сообщения об ошибках локализованы (RU/EN) — минимум два сообщения.
- [ ] Включено `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` и `<Nullable>enable</Nullable>`.

#### Подсказки (без прямого ответа)

- Для проверки «`Duration ∈ {30,45}` когда `Talk`» используйте `Must` с switch-выражением и pattern matching: `d => d is 30 or 45`.
- Для уникальности тегов: `Must(t => t.Distinct(StringComparer.OrdinalIgnoreCase).Count() == t.Count)`.
- Для перекрёстного правила `CoSpeakerEmail != SpeakerEmail` оберните в `When(x => !string.IsNullOrWhiteSpace(x.CoSpeakerEmail), ...)` и используйте `NotEqual(x => x.SpeakerEmail)`.
- `ValidationProblemDetails` — класс из `Microsoft.AspNetCore.Mvc`; его словарь `Errors` заполняется из `ModelState`.
- Не забудьте `using FluentValidation;`, `using FluentValidation.AspNetCore;`, `using System.ComponentModel.DataAnnotations;`, `using Microsoft.AspNetCore.Mvc;`.
- Для `Problem(...)` с `409` используйте перегрузку с `statusCode:` и `title:`.

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8 — DotNextHub: двухуровневая валидация заявок на доклад
// DotNextHub: two-level validation of talk proposals

using System.ComponentModel.DataAnnotations;
using FluentValidation;
using FluentValidation.AspNetCore;
using Microsoft.AspNetCore.Mvc;

// 1) Enum формата доклада / Talk format enum
public enum TalkFormat { Talk, Workshop, Panel }

// 2) DTO с простыми правилами через DataAnnotations / DTO with simple DataAnnotations rules
public record TalkProposalRequest(
    [Required(ErrorMessage = "SpeakerEmail обязателен / Speaker email is required")]
    [EmailAddress(ErrorMessage = "Некорректный email / Invalid email")]
    string SpeakerEmail,

    [Required(ErrorMessage = "Имя обязательно / Speaker name is required")]
    [StringLength(100, MinimumLength = 2,
        ErrorMessage = "Имя 2–100 символов / Name must be 2–100 chars")]
    string SpeakerName,

    [StringLength(500, ErrorMessage = "Био ≤ 500 символов / Bio must be ≤ 500 chars")]
    string Bio,

    [Required(ErrorMessage = "Заголовок обязателен / Title is required")]
    [StringLength(120, MinimumLength = 5,
        ErrorMessage = "Заголовок 5–120 символов / Title must be 5–120 chars")]
    string Title,

    [Required(ErrorMessage = "Аннотация обязательна / Abstract is required")]
    [StringLength(1000, MinimumLength = 20,
        ErrorMessage = "Аннотация 20–1000 символов / Abstract must be 20–1000 chars")]
    string Abstract,

    [EnumDataType(typeof(TalkFormat),
        ErrorMessage = "Неизвестный формат / Unknown format")]
    TalkFormat Format,

    // int? — значимый тип не бывает null, [Required] бесполезен; проверяем в FV.
    // int? — value type is never null, [Required] is useless; validate in FV.
    int? Duration,

    IReadOnlyList<string>? Tags,

    string? CoSpeakerEmail
);

// 3) FluentValidation: условные и перекрёстные правила / conditional & cross-cutting rules
public class TalkProposalRequestValidator : AbstractValidator<TalkProposalRequest>
{
    public TalkProposalRequestValidator()
    {
        // Duration зависит от Format (pattern matching, C# 12).
        RuleFor(x => x.Duration)
            .NotNull().WithMessage("Duration обязателен / Duration is required")
            .Must((req, d) => req.Format switch
            {
                TalkFormat.Talk     => d is 30 or 45,
                TalkFormat.Workshop => d is 120 or 180,
                TalkFormat.Panel    => d is 60,
                _ => false
            })
            .WithMessage((req, _) => req.Format switch
            {
                TalkFormat.Talk     => "Talk: 30 или 45 мин / Talk: 30 or 45 min",
                TalkFormat.Workshop => "Workshop: 120 или 180 мин / Workshop: 120 or 180 min",
                TalkFormat.Panel    => "Panel: 60 мин / Panel: 60 min",
                _ => "Неизвестный формат / Unknown format"
            });

        // Теги: количество, длина каждого, регистронезависимая уникальность.
        When(x => x.Tags is not null, () =>
        {
            RuleFor(x => x.Tags!)
                .Must(t => t.Count is >= 1 and <= 5)
                .WithMessage("Тегов 1–5 / 1 to 5 tags required")
                .Must(t => t.All(s => s.Length is >= 2 and <= 30))
                .WithMessage("Каждый тег 2–30 символов / Each tag must be 2–30 chars")
                .Must(t => t.Distinct(StringComparer.OrdinalIgnoreCase).Count() == t.Count)
                .WithMessage("Теги должны быть уникальны / Tags must be unique");
        });

        // Со-спикер: опционален, но если задан — валиден и не равен основному.
        When(x => !string.IsNullOrWhiteSpace(x.CoSpeakerEmail), () =>
        {
            RuleFor(x => x.CoSpeakerEmail!)
                .EmailAddress().WithMessage("Co-speaker email некорректен / Invalid co-speaker email")
                .NotEqual(x => x.SpeakerEmail)
                .WithMessage("Со-спикер не может быть основным / Co-speaker must differ from speaker");
        });
    }
}

// 4) Доменный инвариант: заблокированные спикеры / domain invariant: banned speakers
public interface IBannedSpeakerService
{
    Task<bool> IsBannedAsync(string email, CancellationToken ct);
}

public sealed class BannedSpeakerService : IBannedSpeakerService
{
    private static readonly HashSet<string> _banned = new(StringComparer.OrdinalIgnoreCase)
    {
        "banned@dotnexthub.dev"
    };

    public Task<bool> IsBannedAsync(string email, CancellationToken ct) =>
        Task.FromResult(_banned.Contains(email));
}

// 5) Контроллер: [ApiController] + явный FV + доменная проверка / controller
[ApiController]
[Route("api/[controller]")]
public class ProposalsController : ControllerBase
{
    private readonly IValidator<TalkProposalRequest> _validator;
    private readonly IBannedSpeakerService _banned;

    public ProposalsController(IValidator<TalkProposalRequest> validator, IBannedSpeakerService banned)
    {
        _validator = validator;
        _banned = banned;
    }

    [HttpPost]
    public async Task<IActionResult> Submit(TalkProposalRequest request, CancellationToken ct)
    {
        // ModelState уже проверен фильтром [ApiController] по атрибутам.
        // ModelState is already checked by the [ApiController] filter against attributes.
        var result = await _validator.ValidateAsync(request, ct);
        if (!result.IsValid)
        {
            // Складываем ошибки FV в ModelState → единый ValidationProblem (RFC 7807).
            foreach (var e in result.Errors)
                ModelState.AddModelError(e.PropertyName, e.ErrorMessage);

            return ValidationProblem(ModelState);
        }

        // Доменный инвариант: спикер не в чёрном списке → 409 ProblemDetails.
        if (await _banned.IsBannedAsync(request.SpeakerEmail, ct))
        {
            return Problem(
                title: "Спикер заблокирован / Speaker is banned",
                statusCode: StatusCodes.Status409Conflict,
                detail: $"Email {request.SpeakerEmail} в чёрном списке / email is banned");
        }

        return Ok(new { ok = true, request.Title, request.Format, request.Duration });
    }
}

// 6) Program.cs (top-level statements, .NET 8)
// var builder = WebApplication.CreateBuilder(args);
// builder.Services.AddControllers()
//     .ConfigureApiBehaviorOptions(opt =>
//     {
//         opt.InvalidModelStateResponseFactory = ctx =>
//         {
//             var problem = new ValidationProblemDetails(ctx.ModelState)
//             {
//                 Title = "Ошибка валидации / Validation error",
//                 Status = StatusCodes.Status400BadRequest
//             };
//             return new BadRequestObjectResult(problem);
//         };
//     });
// builder.Services.AddValidatorsFromAssemblyContaining<TalkProposalRequestValidator>();
// builder.Services.AddSingleton<IBannedSpeakerService, BannedSpeakerService>();
// var app = builder.Build();
// app.MapControllers();
// app.Run();

// 7) Unit-тесты (xUnit + FluentAssertions)
public class TalkProposalRequestValidatorTests
{
    private readonly TalkProposalRequestValidator _sut = new();

    private static TalkProposalRequest Valid(int? duration = 45, TalkFormat f = TalkFormat.Talk) =>
        new("sp@dotnexthub.dev", "Anna", "bio", "Scaling .NET Apps",
            new string('x', 50), f, duration, ["architecture", "testing"], null);

    [Fact]
    public void ShouldPass_WhenValidTalk() =>
        _sut.Validate(Valid()).IsValid.Should().BeTrue();

    [Fact]
    public void ShouldFail_WhenWorkshopDurationIs30() =>
        _sut.Validate(Valid(duration: 30, f: TalkFormat.Workshop)).IsValid
            .Should().BeFalse();

    [Fact]
    public void ShouldFail_WhenTagsAreDuplicates() =>
        _sut.Validate(Valid() with { Tags = ["dotnet", "DotNet"] }).Errors
            .Should().Contain(e => e.PropertyName == nameof(TalkProposalRequest.Tags));
}
```

**Разбор по строкам.** Блок (1) — перечисление `TalkFormat`: простой тип-источник правды для формата доклада, на него опирается switch в валидаторе. Блок (2) — `record` с позиционными параметрами: простые правила формата (email, длины строк, `EnumDataType`) живут в атрибутах `DataAnnotations`, как в примере урока с `RegisterRequest`. КРИТИЧНО `Duration` сделан `int?` — это прямая защита от ошибки урока «`[Required]` на `int` бесполезен». `Tags` — `IReadOnlyList<string>?`, чтобы применять collection expressions в тестах. Блок (3) — `TalkProposalRequestValidator`: `RuleFor(x => x.Duration).NotNull().Must(...)` с switch-выражением и C# 12 pattern matching `d is 30 or 45` выражает условное правило «длительность зависит от формата», которое атрибутами выразить нельзя. `WithMessage` с лямбдой даёт контекстное сообщение. `When(x => x.Tags is not null, ...)` применяет правила к тегам только если они заданы — это правильное поведение `When` для опциональных полей, упомянутое в уроке. `Distinct(StringComparer.OrdinalIgnoreCase)` обеспечивает регистронезависимую уникальность. `NotEqual(x => x.SpeakerEmail)` — перекрёстное правило, главное преимущество `FluentValidation` над атрибутами. Заметьте: email-формат `SpeakerEmail` НЕ повторяется в валидаторе — он уже в атрибуте `[EmailAddress]`, что следует best practice «не дублировать правила». Блок (4) — `BannedSpeakerService` с `HashSet` и `OrdinalIgnoreCase`: это доменный инвариант, второй уровень валидации из урока, который нельзя выразить атрибутами. Блок (5) — контроллер: `[ApiController]` уже отклонил невалидные по атрибутам запросы; явный `ValidateAsync(request, ct)` проверяет сложные правила, `ct` пробрасывается (защита от утечки ресурсов из урока). Ошибки складываются в `ModelState` и возвращаются через `ValidationProblem(ModelState)` — единый RFC 7807-контракт, а НЕ `BadRequest("...")`. Доменный инвариант — `Problem(..., 409)`. Блок (6) — `Program.cs` с `ConfigureApiBehaviorOptions` и кастомным `InvalidModelStateResponseFactory`, возвращающим `ValidationProblemDetails` с `Title` и `Status`, ровно как в коде урока. Блок (7) — unit-тесты с collection expressions `["architecture", "testing"]` и `with`-выражением для модификации записи. Применённые концепции урока: двухуровневая валидация, `ModelState`, `[ApiController]` automatic problem details, `ValidationProblem`, `ApiBehaviorOptions`, `When`/`Must`/`NotEqual`, локализация сообщений, `CancellationToken`, отказ от дублирования правил, RFC 7807.

#### Задания на углубление (бонус)

1. **Локализация через `IStringLocalizer`.** Вынесите все строки сообщений в ресурсные файлы `Resources/Messages.ru.resx` и `Resources/Messages.en.resx`, подключите `IStringLocalizer<TalkProposalRequestValidator>` и переключайте язык по заголовку `Accept-Language`. Проверьте, что `curl` с `Accept-Language: ru` возвращает русские сообщения, а с `en` — английские.
2. **Асинхронный валидатор с обращением к БД.** Добавьте правило «email спикера не должен уже иметь принятую заявку в этом году» через `MustAsync` с обращением к in-memory `DbContext`. Не забудьте пробросить `CancellationToken` и покрыть правило тестом с моком контекста.
3. **Minimal API-вариант.** Перепишите эндпоинт на Minimal API с `TypedResults.ValidationProblem(...)` вместо контроллера; сохраните единый контракт ошибок. Сравните, где валидация читается легче.
4. **Кастомный `ValidationAttribute`.** Реализуйте `[UniqueTags(StringComparison.OrdinalIgnoreCase)]` как `ValidationAttribute` с `IsValid`, чтобы дублировать проверку уникальности на уровне атрибута, и объясните в комментариях, почему это нарушает best practice «один источник правды» из урока.

---

## Statement in English / Постановка на английском

#### Context & motivation

You are a backend engineer on the “DotNextHub” conference platform. Speakers submit talk proposals through a public endpoint `POST /api/proposals`. A proposal carries speaker data (email, name, bio), talk data (title, abstract, format, duration, tags), and optionally a co-speaker. The data is heterogeneous: some of it is simple format (email, string length), some is conditional business logic (duration depends on the talk format, the co-speaker cannot be the same person as the main speaker, tags must be case-insensitively unique). On top of that, the domain has an invariant: the speaker’s email must not appear on a “black list” of banned accounts. That rule cannot be expressed with attributes — it depends on external state and must be checked in business logic.

The lesson showed that `DataAnnotations` attributes are convenient for simple format checks, but they pollute the domain model with technical details and cannot express context. `FluentValidation` handles complex rules at the cost of a separate validator class and DI registration. The critical discipline is not to duplicate the same rule in both places (otherwise you get desync and bugs on change) and to return errors through a single RFC 7807 contract. In this assignment you will build an endpoint where format is validated by attributes, complex and conditional rules by `FluentValidation`, and a domain invariant by a service — and all of it converges on one `ValidationProblem` response. You will also face the fact that `[ApiController]` automatically rejects attribute-invalid requests before your code runs, so the explicit validator call must be planned deliberately, not bolted on blindly.

#### What to do step by step

1. Create the solution and the web project. From the repository root run:
   ```
   dotnet new web -n DotNextHub.Api -o DotNextHub.Api
   cd DotNextHub.Api
   dotnet add package FluentValidation.AspNetCore --version 11.3.0
   dotnet add package Microsoft.AspNetCore.OpenApi --version 8.0.*
   ```
   Expected result: a `DotNextHub.Api` folder containing `Program.cs` and `DotNextHub.Api.csproj` with a `PackageReference` to `FluentValidation.AspNetCore`. Enable `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` and `<Nullable>enable</Nullable>` in the `csproj`.

2. Define the enum and the DTO. Create `Models/TalkProposalRequest.cs` with a `record TalkProposalRequest`, an enum `TalkFormat { Talk, Workshop, Panel }`, and properties: `SpeakerEmail`, `SpeakerName`, `Bio`, `Title`, `Abstract`, `Format`, `Duration`, `Tags`, `CoSpeakerEmail`. Cover simple rules with `DataAnnotations` attributes: `SpeakerEmail` — `[Required][EmailAddress]`; `SpeakerName` — `[Required][StringLength(100, MinimumLength=2)]`; `Bio` — `[StringLength(500)]`; `Title` — `[Required][StringLength(120, MinimumLength=5)]`; `Abstract` — `[Required][StringLength(1000, MinimumLength=20)]`; `Format` — `[EnumDataType(typeof(TalkFormat))]`. CRITICAL: make `Duration` an `int?`, not an `int` — `[Required]` on a value type is useless (it is never `null`); this is a direct mistake called out in the lesson. `Tags` should be `IReadOnlyList<string>?`.

3. Create `Validators/TalkProposalRequestValidator.cs` inheriting from `AbstractValidator<TalkProposalRequest>`. Describe conditional rules: `When(Format == Talk)` → `Duration` must be `30 or 45`; `Workshop` → `120 or 180`; `Panel` → `60`. `CoSpeakerEmail`, when present, must be a valid email and not equal to `SpeakerEmail` (`NotEqual`). Each tag must be 2–30 chars, there must be 1–5 tags, and they must be case-insensitively unique (`Distinct(StringComparer.OrdinalIgnoreCase)`). Do NOT duplicate the `SpeakerEmail` email format — it is already covered by the `[EmailAddress]` attribute. Localize messages RU/EN.

4. Register the validators and configure Problem Details in `Program.cs` (top-level statements): `AddControllers().ConfigureApiBehaviorOptions(...)` with a custom `InvalidModelStateResponseFactory` returning a `ValidationProblemDetails` with `Title` and `Status = 400`. Register `AddValidatorsFromAssemblyContaining<TalkProposalRequestValidator>()`.

5. Implement `ProposalsController` with `[ApiController][Route("api/[controller]")]` and a `POST` endpoint. Inside: explicitly call `_validator.ValidateAsync(request, ct)`, on errors push them into `ModelState` via `AddModelError` and return `ValidationProblem(ModelState)` (NOT `BadRequest("...")`). Then call the domain service `IBannedSpeakerService.IsBannedAsync(email, ct)` — if banned, return `Problem(..., statusCode: 409)`. Otherwise `Ok(...)`.

6. Implement `IBannedSpeakerService` and `BannedSpeakerService` with a hard-coded set of banned emails (e.g. `banned@dotnexthub.dev`) using `HashSet<string>` with `StringComparer.OrdinalIgnoreCase`.

7. Run `dotnet run` and verify with `curl` three scenarios: (a) empty `SpeakerEmail` → `400` with `errors.SpeakerEmail`; (b) `Format=Workshop, Duration=30` → `400` with `errors.Duration`; (c) a valid proposal → `200`.

8. Write xUnit unit tests for the validator: `ShouldPass_WhenValidTalk`, `ShouldFail_WhenWorkshopDurationIs30`, `ShouldFail_WhenTagsAreDuplicates`. Expected output: 3 tests passed.

#### Requirements

- Target framework `net8.0`, C# 12: top-level `Program.cs`, pattern matching (`is 30 or 45`), collection expressions (`["a","b"]`), `init` properties are all allowed.
- All input DTOs are `record` with positional parameters; complex and conditional rules live only in `FluentValidation`, simple ones in attributes. Do not duplicate the same rule in both places: the `SpeakerEmail` email format is set by the `[EmailAddress]` attribute and is NOT repeated in the validator — the validator only has `NotEqual`/`Must` for `CoSpeakerEmail`.
- The controller is decorated with `[ApiController]`; the validation-error response is `ValidationProblem` (not `BadRequest("string")`). The domain invariant (banned speaker) returns `409 ProblemDetails` via `Problem(...)`.
- `FluentValidation`: use `When(...)`, `Must(...)`, `NotEqual(...)`, `WithMessage(...)`. You must pass the `CancellationToken` into `ValidateAsync` — otherwise a resource leak under load (a mistake from the lesson).
- Localization: at least two messages explicitly bilingual (`"RU / EN"`), like the lesson example.
- Unit-test coverage of the validator: at least 3 cases (pass, fail, conditional).
- The code compiles without warnings thanks to `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>`.

#### Pitfalls

- `[Required]` on an `int` is useless: a value type is never `null`. Make `Duration` an `int?` and validate it in `FluentValidation` with `NotNull().Must(...)`. The lesson explicitly warns about this mistake.
- `[ApiController]` automatically returns `ValidationProblem` from attributes before your code runs: if the attributes reject the request, the explicit `FluentValidation` call is never reached. The fix: do not overlap rules — format in attributes, conditional logic in the validator.
- Ignoring the `CancellationToken` in `ValidateAsync` is a common resource leak under load; always forward `ct` from the action signature.
- Duplicating rules (email in the attribute AND in `FluentValidation`) is a source of desync and bugs on change; pick one source of truth per check.
- Do not return `BadRequest("text")` — it breaks the RFC 7807 contract and forces the client to parse different error shapes.
- `Must(...)` in `FluentValidation` receives the whole object — handy for cross-cutting rules (`CoSpeakerEmail != SpeakerEmail`), but do not forget `WithMessage`, otherwise the client gets a cryptic default message.
- `When(...)` skips rules when the condition is false by default — correct behavior for optional fields (`CoSpeakerEmail`, `Tags`), but make sure required checks are not accidentally swallowed by a condition.
- Collection expressions (C# 12) `Tags = ["architecture","testing"]` are handy in tests, but keep `IReadOnlyList<string>?` in the DTO.
- Case-insensitive tag uniqueness requires `StringComparer.OrdinalIgnoreCase`; `Distinct` without a comparer gives a false non-unique result for `dotnet`/`DotNet`.
- `ValidationProblemDetails` (from `Microsoft.AspNetCore.Mvc`) has an `Errors` dictionary populated from `ModelState`; `Title` and `Status` are set manually in the factory.

#### Acceptance criteria

- [ ] The `DotNextHub.Api` project builds with `dotnet build` without errors or warnings.
- [ ] `TalkProposalRequest` is a `record` with positional parameters and `DataAnnotations` attributes for simple rules.
- [ ] `Duration` is `int?` (not `int`), validated in `FluentValidation` via `NotNull().Must(...)`.
- [ ] `TalkProposalRequestValidator` contains conditional rules via `When`/`Must` keyed on the talk format.
- [ ] `CoSpeakerEmail` is validated for validity and inequality with `SpeakerEmail` (`NotEqual`), only when present.
- [ ] Tags are validated for per-tag length, count 1–5, and case-insensitive uniqueness.
- [ ] Validators are registered via `AddValidatorsFromAssemblyContaining<TalkProposalRequestValidator>()`.
- [ ] `ConfigureApiBehaviorOptions` sets a custom `InvalidModelStateResponseFactory` with `ValidationProblemDetails`.
- [ ] The controller is decorated with `[ApiController]`; validation errors are `ValidationProblem`, not `BadRequest(string)`.
- [ ] The domain invariant (banned speaker) returns `409` via `Problem(...)`.
- [ ] The `CancellationToken` is forwarded to `ValidateAsync` and the domain service.
- [ ] At least 3 xUnit tests on the validator (pass/fail/conditional) — all green.
- [ ] `curl` scenarios (a)(b)(c) return the expected statuses and JSON in `application/problem+json`.
- [ ] Error messages are localized (RU/EN) — at least two messages.
- [ ] `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` and `<Nullable>enable</Nullable>` are enabled.

#### Hints (no direct answer)

- For “`Duration ∈ {30,45}` when `Talk`” use `Must` with a switch expression and pattern matching: `d => d is 30 or 45`.
- For tag uniqueness: `Must(t => t.Distinct(StringComparer.OrdinalIgnoreCase).Count() == t.Count)`.
- For the cross-cutting rule `CoSpeakerEmail != SpeakerEmail`, wrap in `When(x => !string.IsNullOrWhiteSpace(x.CoSpeakerEmail), ...)` and use `NotEqual(x => x.SpeakerEmail)`.
- `ValidationProblemDetails` is a class from `Microsoft.AspNetCore.Mvc`; its `Errors` dictionary is filled from `ModelState`.
- Do not forget `using FluentValidation;`, `using FluentValidation.AspNetCore;`, `using System.ComponentModel.DataAnnotations;`, `using Microsoft.AspNetCore.Mvc;`.
- For `Problem(...)` with `409` use the overload with `statusCode:` and `title:`.

#### Reference solution walk-through

```csharp
// C# 12 / .NET 8 — DotNextHub: two-level validation of talk proposals

using System.ComponentModel.DataAnnotations;
using FluentValidation;
using FluentValidation.AspNetCore;
using Microsoft.AspNetCore.Mvc;

// 1) Talk format enum
public enum TalkFormat { Talk, Workshop, Panel }

// 2) DTO with simple DataAnnotations rules
public record TalkProposalRequest(
    [Required(ErrorMessage = "SpeakerEmail is required / SpeakerEmail обязателен")]
    [EmailAddress(ErrorMessage = "Invalid email / Некорректный email")]
    string SpeakerEmail,

    [Required(ErrorMessage = "Speaker name is required / Имя обязательно")]
    [StringLength(100, MinimumLength = 2,
        ErrorMessage = "Name must be 2–100 chars / Имя 2–100 символов")]
    string SpeakerName,

    [StringLength(500, ErrorMessage = "Bio must be ≤ 500 chars / Био ≤ 500 символов")]
    string Bio,

    [Required(ErrorMessage = "Title is required / Заголовок обязателен")]
    [StringLength(120, MinimumLength = 5,
        ErrorMessage = "Title must be 5–120 chars / Заголовок 5–120 символов")]
    string Title,

    [Required(ErrorMessage = "Abstract is required / Аннотация обязательна")]
    [StringLength(1000, MinimumLength = 20,
        ErrorMessage = "Abstract must be 20–1000 chars / Аннотация 20–1000 символов")]
    string Abstract,

    [EnumDataType(typeof(TalkFormat),
        ErrorMessage = "Unknown format / Неизвестный формат")]
    TalkFormat Format,

    // int? — value type is never null, [Required] is useless; validate in FV.
    int? Duration,

    IReadOnlyList<string>? Tags,

    string? CoSpeakerEmail
);

// 3) FluentValidation: conditional & cross-cutting rules
public class TalkProposalRequestValidator : AbstractValidator<TalkProposalRequest>
{
    public TalkProposalRequestValidator()
    {
        // Duration depends on Format (pattern matching, C# 12).
        RuleFor(x => x.Duration)
            .NotNull().WithMessage("Duration is required / Duration обязателен")
            .Must((req, d) => req.Format switch
            {
                TalkFormat.Talk     => d is 30 or 45,
                TalkFormat.Workshop => d is 120 or 180,
                TalkFormat.Panel    => d is 60,
                _ => false
            })
            .WithMessage((req, _) => req.Format switch
            {
                TalkFormat.Talk     => "Talk: 30 or 45 min / Talk: 30 или 45 мин",
                TalkFormat.Workshop => "Workshop: 120 or 180 min / Workshop: 120 или 180 мин",
                TalkFormat.Panel    => "Panel: 60 min / Panel: 60 мин",
                _ => "Unknown format / Неизвестный формат"
            });

        // Tags: count, per-tag length, case-insensitive uniqueness.
        When(x => x.Tags is not null, () =>
        {
            RuleFor(x => x.Tags!)
                .Must(t => t.Count is >= 1 and <= 5)
                .WithMessage("1 to 5 tags required / Тегов 1–5")
                .Must(t => t.All(s => s.Length is >= 2 and <= 30))
                .WithMessage("Each tag must be 2–30 chars / Каждый тег 2–30 символов")
                .Must(t => t.Distinct(StringComparer.OrdinalIgnoreCase).Count() == t.Count)
                .WithMessage("Tags must be unique / Теги должны быть уникальны");
        });

        // Co-speaker: optional, but if present must be valid and differ from the speaker.
        When(x => !string.IsNullOrWhiteSpace(x.CoSpeakerEmail), () =>
        {
            RuleFor(x => x.CoSpeakerEmail!)
                .EmailAddress().WithMessage("Invalid co-speaker email / Co-speaker email некорректен")
                .NotEqual(x => x.SpeakerEmail)
                .WithMessage("Co-speaker must differ from speaker / Со-спикер не может быть основным");
        });
    }
}

// 4) Domain invariant: banned speakers
public interface IBannedSpeakerService
{
    Task<bool> IsBannedAsync(string email, CancellationToken ct);
}

public sealed class BannedSpeakerService : IBannedSpeakerService
{
    private static readonly HashSet<string> _banned = new(StringComparer.OrdinalIgnoreCase)
    {
        "banned@dotnexthub.dev"
    };

    public Task<bool> IsBannedAsync(string email, CancellationToken ct) =>
        Task.FromResult(_banned.Contains(email));
}

// 5) Controller: [ApiController] + explicit FV + domain check
[ApiController]
[Route("api/[controller]")]
public class ProposalsController : ControllerBase
{
    private readonly IValidator<TalkProposalRequest> _validator;
    private readonly IBannedSpeakerService _banned;

    public ProposalsController(IValidator<TalkProposalRequest> validator, IBannedSpeakerService banned)
    {
        _validator = validator;
        _banned = banned;
    }

    [HttpPost]
    public async Task<IActionResult> Submit(TalkProposalRequest request, CancellationToken ct)
    {
        // ModelState is already checked by the [ApiController] filter against attributes.
        var result = await _validator.ValidateAsync(request, ct);
        if (!result.IsValid)
        {
            // Push FV errors into ModelState → uniform ValidationProblem (RFC 7807).
            foreach (var e in result.Errors)
                ModelState.AddModelError(e.PropertyName, e.ErrorMessage);

            return ValidationProblem(ModelState);
        }

        // Domain invariant: speaker not banned → 409 ProblemDetails.
        if (await _banned.IsBannedAsync(request.SpeakerEmail, ct))
        {
            return Problem(
                title: "Speaker is banned / Спикер заблокирован",
                statusCode: StatusCodes.Status409Conflict,
                detail: $"Email {request.SpeakerEmail} is banned / email в чёрном списке");
        }

        return Ok(new { ok = true, request.Title, request.Format, request.Duration });
    }
}

// 6) Program.cs (top-level statements, .NET 8)
// var builder = WebApplication.CreateBuilder(args);
// builder.Services.AddControllers()
//     .ConfigureApiBehaviorOptions(opt =>
//     {
//         opt.InvalidModelStateResponseFactory = ctx =>
//         {
//             var problem = new ValidationProblemDetails(ctx.ModelState)
//             {
//                 Title = "Validation error / Ошибка валидации",
//                 Status = StatusCodes.Status400BadRequest
//             };
//             return new BadRequestObjectResult(problem);
//         };
//     });
// builder.Services.AddValidatorsFromAssemblyContaining<TalkProposalRequestValidator>();
// builder.Services.AddSingleton<IBannedSpeakerService, BannedSpeakerService>();
// var app = builder.Build();
// app.MapControllers();
// app.Run();

// 7) Unit tests (xUnit + FluentAssertions)
public class TalkProposalRequestValidatorTests
{
    private readonly TalkProposalRequestValidator _sut = new();

    private static TalkProposalRequest Valid(int? duration = 45, TalkFormat f = TalkFormat.Talk) =>
        new("sp@dotnexthub.dev", "Anna", "bio", "Scaling .NET Apps",
            new string('x', 50), f, duration, ["architecture", "testing"], null);

    [Fact]
    public void ShouldPass_WhenValidTalk() =>
        _sut.Validate(Valid()).IsValid.Should().BeTrue();

    [Fact]
    public void ShouldFail_WhenWorkshopDurationIs30() =>
        _sut.Validate(Valid(duration: 30, f: TalkFormat.Workshop)).IsValid
            .Should().BeFalse();

    [Fact]
    public void ShouldFail_WhenTagsAreDuplicates() =>
        _sut.Validate(Valid() with { Tags = ["dotnet", "DotNet"] }).Errors
            .Should().Contain(e => e.PropertyName == nameof(TalkProposalRequest.Tags));
}
```

**Line-by-line walk-through.** Block (1) is the `TalkFormat` enum: a single source of truth for the format that the validator switch relies on. Block (2) is the positional `record`: simple format rules (email, string lengths, `EnumDataType`) live in `DataAnnotations` attributes, exactly like the lesson’s `RegisterRequest`. Critically, `Duration` is `int?` — a direct defense against the lesson’s mistake “`[Required]` on `int` is useless”. `Tags` is `IReadOnlyList<string>?` so collection expressions work in tests. Block (3) is `TalkProposalRequestValidator`: `RuleFor(x => x.Duration).NotNull().Must(...)` with a switch expression and C# 12 pattern matching `d is 30 or 45` expresses the conditional rule “duration depends on format”, which attributes cannot express. `WithMessage` with a lambda gives a context-aware message. `When(x => x.Tags is not null, ...)` applies tag rules only when tags are present — the correct `When` behavior for optional fields mentioned in the lesson. `Distinct(StringComparer.OrdinalIgnoreCase)` enforces case-insensitive uniqueness. `NotEqual(x => x.SpeakerEmail)` is a cross-cutting rule, the main advantage of `FluentValidation` over attributes. Note that the `SpeakerEmail` email format is NOT repeated in the validator — it is already in the `[EmailAddress]` attribute, following the best practice “do not duplicate rules”. Block (4) is `BannedSpeakerService` with a `HashSet` and `OrdinalIgnoreCase`: this is the domain invariant, the second validation level from the lesson, which cannot be expressed by attributes. Block (5) is the controller: `[ApiController]` has already rejected attribute-invalid requests; the explicit `ValidateAsync(request, ct)` checks complex rules, `ct` is forwarded (protection against the lesson’s resource leak). Errors are pushed into `ModelState` and returned via `ValidationProblem(ModelState)` — a single RFC 7807 contract, NOT `BadRequest("...")`. The domain invariant returns `Problem(..., 409)`. Block (6) is `Program.cs` with `ConfigureApiBehaviorOptions` and a custom `InvalidModelStateResponseFactory` returning `ValidationProblemDetails` with `Title` and `Status`, exactly like the lesson code. Block (7) is unit tests with collection expressions `["architecture", "testing"]` and a `with` expression to mutate the record. Lesson concepts applied: two-level validation, `ModelState`, `[ApiController]` automatic problem details, `ValidationProblem`, `ApiBehaviorOptions`, `When`/`Must`/`NotEqual`, message localization, `CancellationToken`, no rule duplication, RFC 7807.

#### Going deeper (bonus)

1. **Localization via `IStringLocalizer`.** Move every message string into resource files `Resources/Messages.ru.resx` and `Resources/Messages.en.resx`, inject `IStringLocalizer<TalkProposalRequestValidator>`, and switch language by the `Accept-Language` header. Verify that `curl` with `Accept-Language: ru` returns Russian messages and with `en` returns English ones.
2. **Async validator with a DB call.** Add a rule “the speaker email must not already have an accepted proposal this year” using `MustAsync` against an in-memory `DbContext`. Forward the `CancellationToken` and cover the rule with a test using a mocked context.
3. **Minimal API variant.** Reimplement the endpoint as a Minimal API with `TypedResults.ValidationProblem(...)` instead of a controller; keep the error contract identical. Compare where validation reads more naturally.
4. **Custom `ValidationAttribute`.** Implement `[UniqueTags(StringComparison.OrdinalIgnoreCase)]` as a `ValidationAttribute` with `IsValid` to duplicate the uniqueness check at the attribute level, and explain in comments why this violates the lesson’s “single source of truth” best practice.

---

#### Чек-лист сдачи / Submission checklist

- [ ] (RU) Файл `homework-M14-L05-validation.md` создан в `modules/M14/lessons/`.
- [ ] (RU) Проект `DotNextHub.Api` собирается без ошибок и warning.
- [ ] (RU) DTO и валидатор реализованы согласно требованиям.
- [ ] (RU) Контроллер возвращает `ValidationProblem` и `Problem(409)`.
- [ ] (RU) Минимум 3 unit-теста зелёные.
- [ ] (RU) `curl`-сценарии (a)(b)(c) подтверждаются скриншотами/выводом.
- [ ] (EN) The `homework-M14-L05-validation.md` file is created under `modules/M14/lessons/`.
- [ ] (EN) The `DotNextHub.Api` project builds without errors or warnings.
- [ ] (EN) DTO and validator are implemented per the requirements.
- [ ] (EN) The controller returns `ValidationProblem` and `Problem(409)`.
- [ ] (EN) At least 3 unit tests are green.
- [ ] (EN) `curl` scenarios (a)(b)(c) are confirmed with output/screenshots.

#### Ресурсы / Resources

- [Microsoft Learn — Model validation in ASP.NET Core](https://learn.microsoft.com/aspnet/core/mvc/models/validation)
- [FluentValidation documentation](https://docs.fluentvalidation.net/)
- [RFC 7807 Problem Details for HTTP APIs](https://datatracker.ietf.org/doc/html/rfc7807)
- [Microsoft Learn — Problem Details in ASP.NET Core](https://learn.microsoft.com/aspnet/core/web-api/handle-errors)
- [Microsoft Learn — `ValidationProblemDetails` class](https://learn.microsoft.com/dotnet/api/microsoft.aspnetcore.mvc.validationproblemdetails)

---

[← К уроку M14-L05](lesson-M14-L05-validation.md) | [⬆ К модулю M14](../README.md) | [Следующее ДЗ →](homework-M14-L06-problemdetails-errors.md)
