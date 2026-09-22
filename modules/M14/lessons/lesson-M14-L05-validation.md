[⬆ К модулю M14](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M14-L05: Валидация, DataAnnotations, FluentValidation (Optional) / Validation, DataAnnotations, FluentValidation (Optional)

**Модуль / Module:** M14
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Валидация в ASP.NET Core — это первая линия обороны между «доверием к пользователю» и «целостностью данных». Представьте пропускной пункт на заводе: данные приходят снаружи (от клиента, из формы, из JSON-тела), и прежде чем попасть в базу или бизнес-логику, они обязаны пройти проверку. Если пропуск недействителен — человека разворачивают на входе, а не ловят уже внутри цеха.

ASP.NET Core предлагает два уровня проверки: **автоматическую** через атрибуты `DataAnnotations` и **явную** через `FluentValidation`. Первый подход декларативный: вы украшаете свойства модели атрибутами `[Required]`, `[Range]`, `[RegularExpression]`, `[StringLength]` и подобными. Фреймворк сам прогоняет их во время привязки модели (model binding) и складывает ошибки в `ModelState`. Это удобно для простых правил и даёт «валидацию на лету» — атрибуты видны прямо в классе, как этикетки на товаре.

**`ModelState`** — это словарь состояния модели, который собирает все ошибки валидации и привязки. В контроллере вы проверяете `ModelState.IsValid`: если `false`, значит входные данные не соответствуют ожиданиям, и запрос нужно отклонить. В минимальных API (Minimal APIs) аналог — `TypedResults.ValidationProblem(...)`, а в контроллерах с `[ApiController]` фреймворк автоматически возвращает `400 Bad Request` с `ProblemDetails`, даже если вы не написали ни строчки проверки вручную. Это поведение называется **automatic problem details** и настраивается через `ApiBehaviorOptions` (`SuppressModelStateInvalidFilter = true` отключает его).

**Атрибуты `DataAnnotations`:**
- `[Required]` — поле обязательно, не может быть `null` или пустой строкой.
- `[Range(min, max)]` — значение должно лежать в диапазоне (для чисел, дат).
- `[RegularExpression(@"...")]` — строка обязана соответствовать шаблону (email-подобные проверки, телефоны, пароли).
- `[StringLength]` / `[MinLength]` / `[MaxLength]` — длина строки или коллекции.
- `[EmailAddress]`, `[Url]`, `[Phone]`, `[CreditCard]` — готовые семантические проверки.
- `[Compare]` — два свойства должны совпадать (пароль и подтверждение).
- `[CustomValidation]` — свой валидатор-метод.

Главный минус атрибутов — они «загрязняют» доменную модель техническими деталями и плохо выражают сложные правила, зависящие от контекста (например, «если тип пользователя — админ, то поле X обязательно, иначе запрещено»). Тут вступает **FluentValidation** — отдельная библиотека, где правила описываются кодом в отдельном классе-валидаторе через fluent-API: `RuleFor(x => x.Email).NotEmpty().EmailAddress().WithMessage("Укажите корректный email")`. Это даёт переиспользование, тестирование, условные правила (`When(...)`), композицию и понятные сообщения об ошибках без захламления модели. Она опциональна и обычно подключается через пакет `FluentValidation.AspNetCore` или ручную регистрацию `IValidator<T>` в DI.

**Best practice:** валидируйте на двух уровнях — на входе (атрибуты/FluentValidation для формата и базовых правил) и в домене/бизнес-логике (для инвариантов и правил, зависящих от состояния). Никогда не полагайтесь только на клиентскую валидацию в браузере — её легко обойти. Помните о **Problem Details (RFC 7807)**: возвращайте ошибки в едином стандарте `application/problem+json`, чтобы клиенты могли их единообразно разбирать. И избегайте «пере-валидации»: дублирование правил в атрибутах, FluentValidation и DTO ведёт к рассинхрону и багам.

#### Theory (EN)

Validation in ASP.NET Core is the first line of defense between “trust the user” and “data integrity.” Think of a factory checkpoint: data arrives from the outside (from a client, a form, a JSON body) and, before it can reach the database or business logic, it must pass inspection. If the badge is invalid, the person is turned away at the gate, not caught later inside the shop floor.

ASP.NET Core offers two layers of checking: **automatic** through `DataAnnotations` attributes and **explicit** through `FluentValidation`. The first is declarative: you decorate model properties with `[Required]`, `[Range]`, `[RegularExpression]`, `[StringLength]` and similar. The framework runs them during model binding and accumulates errors in `ModelState`. This is great for simple rules and gives you “inline validation” — the attributes are visible right on the class, like labels on a product.

**`ModelState`** is a model-state dictionary that collects all binding and validation errors. In a controller you check `ModelState.IsValid`: if it is `false`, the input does not meet expectations and the request must be rejected. In Minimal APIs the analogue is `TypedResults.ValidationProblem(...)`, and in controllers decorated with `[ApiController]` the framework automatically returns `400 Bad Request` with `ProblemDetails` even if you never wrote a single line of manual checking. This behavior is called **automatic problem details** and is configurable through `ApiBehaviorOptions` (`SuppressModelStateInvalidFilter = true` disables it).

**`DataAnnotations` attributes:**
- `[Required]` — the field is mandatory, cannot be `null` or an empty string.
- `[Range(min, max)]` — the value must fall within a range (numbers, dates).
- `[RegularExpression(@"...")]` — the string must match a pattern (email-like checks, phones, passwords).
- `[StringLength]` / `[MinLength]` / `[MaxLength]` — string or collection length.
- `[EmailAddress]`, `[Url]`, `[Phone]`, `[CreditCard]` — ready-made semantic checks.
- `[Compare]` — two properties must match (password and confirmation).
- `[CustomValidation]` — your own validator method.

The main drawback of attributes is that they “pollute” the domain model with technical details and poorly express complex context-dependent rules (for example, “if the user type is admin, field X is required; otherwise it is forbidden”). This is where **FluentValidation** comes in — a separate library where rules are described in code, in a dedicated validator class via a fluent API: `RuleFor(x => x.Email).NotEmpty().EmailAddress().WithMessage("Provide a valid email")`. This gives reuse, testability, conditional rules (`When(...)`), composition, and clear error messages without cluttering the model. It is optional and is usually wired in through the `FluentValidation.AspNetCore` package or by manually registering `IValidator<T>` in DI.

**Best practice:** validate at two levels — at the entry point (attributes/FluentValidation for format and basic rules) and in the domain/business logic (for invariants and state-dependent rules). Never rely solely on client-side validation in the browser — it is trivial to bypass. Remember **Problem Details (RFC 7807)**: return errors in a single `application/problem+json` standard so clients can parse them uniformly. And avoid “over-validation”: duplicating rules across attributes, FluentValidation, and DTOs leads to desynchronization and bugs.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — Валидация через DataAnnotations + FluentValidation + Problem Details
// Validation via DataAnnotations + FluentValidation + Problem Details

using System.ComponentModel.DataAnnotations;
using FluentValidation;
using Microsoft.AspNetCore.Mvc;

// 1) Модель с DataAnnotations / Model with DataAnnotations
public record RegisterRequest(
    [Required(ErrorMessage = "Email обязателен / Email is required")]
    [EmailAddress(ErrorMessage = "Некорректный email / Invalid email")]
    string Email,

    [Required(ErrorMessage = "Имя обязательно / Name is required")]
    [StringLength(50, MinimumLength = 2,
        ErrorMessage = "Имя 2–50 символов / Name must be 2–50 chars")]
    string Name,

    [Range(18, 120, ErrorMessage = "Возраст 18–120 / Age must be 18–120")]
    int Age,

    [RegularExpression(@"^(?=.*[A-Za-z])(?=.*\d)[A-Za-z\d]{8,}$",
        ErrorMessage = "Пароль: 8+ символов, буквы и цифры / " +
                       "Password: 8+ chars, letters and digits")]
    string Password,

    [Compare("Password",
        ErrorMessage = "Пароли не совпадают / Passwords do not match")]
    string ConfirmPassword
);

// 2) Тот же сценарий через FluentValidation / Same scenario via FluentValidation
public class RegisterRequestValidator : AbstractValidator<RegisterRequest>
{
    public RegisterRequestValidator()
    {
        RuleFor(x => x.Email)
            .NotEmpty().WithMessage("Email обязателен / Email is required")
            .EmailAddress().WithMessage("Некорректный email / Invalid email");

        RuleFor(x => x.Name)
            .Length(2, 50).WithMessage("Имя 2–50 символов / Name must be 2–50 chars");

        RuleFor(x => x.Age)
            .InclusiveBetween(18, 120)
            .WithMessage("Возраст 18–120 / Age must be 18–120");

        RuleFor(x => x.Password)
            .Matches(@"^(?=.*[A-Za-z])(?=.*\d)[A-Za-z\d]{8,}$")
            .WithMessage("Пароль: 8+ символов, буквы и цифры / " +
                         "Password: 8+ chars, letters and digits");

        RuleFor(x => x.ConfirmPassword)
            .Equal(x => x.Password)
            .WithMessage("Пароли не совпадают / Passwords do not match");
    }
}

// 3) Контроллер с автоматическим Problem Details
//    Controller with automatic Problem Details
[ApiController]            // автоматически возвращает ValidationProblem при ошибках
[Route("api/[controller]")]
public class AccountController : ControllerBase
{
    private readonly IValidator<RegisterRequest> _validator; // FluentValidation

    public AccountController(IValidator<RegisterRequest> validator) =>
        _validator = validator;

    [HttpPost("register")]
    public async Task<IActionResult> Register(RegisterRequest request,
                                              CancellationToken ct)
    {
        // (a) ModelState уже проверен фильтром [ApiController] автоматически.
        //     (a) ModelState is already checked by the [ApiController] filter.
        // (b) Дополнительная явная проверка через FluentValidation.
        //     (b) Additional explicit check via FluentValidation.
        var result = await _validator.ValidateAsync(request, ct);
        if (!result.IsValid)
        {
            // Складываем ошибки FluentValidation в ModelState,
            // чтобы ответ был единообразным ValidationProblem (RFC 7807).
            // Push FluentValidation errors into ModelState so the response
            // is a uniform ValidationProblem (RFC 7807).
            foreach (var error in result.Errors)
                ModelState.AddModelError(error.PropertyName, error.ErrorMessage);

            return ValidationProblem(ModelState);
        }

        // Бизнес-инвариант: email не должен быть уже занят.
        // Business invariant: email must not already be taken.
        // (здесь — заглушка для демонстрации второго уровня валидации)
        // (placeholder here to demonstrate the second validation level)
        return Ok(new { ok = true, request.Email });
    }
}

// 4) Регистрация FluentValidation в DI (Program.cs)
//    FluentValidation DI registration (Program.cs)
// builder.Services.AddValidatorsFromAssemblyContaining<RegisterRequestValidator>();
// builder.Services.AddControllers()
//     .ConfigureApiBehaviorOptions(opt =>
//     {
//         // Кастомизация Problem Details при желании.
//         // Customize Problem Details if needed.
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
```

#### Best Practices

- Валидируйте на двух уровнях: формат и базовые правила — на входе (атрибуты/FluentValidation), инварианты и контекстные правила — в домене.
- Используйте `[ApiController]` и `ValidationProblem`, чтобы получать единый RFC 7807-совместимый ответ автоматически.
- Не дублируйте правила в атрибутах, FluentValidation и DTO — выберите один источник правды для каждого типа проверки.
- Не доверяйте клиентской валидации: она лишь улучшает UX, но обходится за секунды.
- Локализуйте сообщения об ошибках через `IStringLocalizer` или ресурсные файлы для двуязычных интерфейсов.

- Validate at two levels: format and basic rules at the entry point (attributes/FluentValidation), invariants and context-sensitive rules in the domain.
- Use `[ApiController]` and `ValidationProblem` to get a uniform RFC 7807-compliant response automatically.
- Do not duplicate rules across attributes, FluentValidation, and DTOs — pick one source of truth per check type.
- Do not trust client-side validation: it only improves UX and can be bypassed in seconds.
- Localize error messages via `IStringLocalizer` or resource files for bilingual interfaces.

#### Частые ошибки / Common Mistakes

- `[Required]` на `int` (значимый тип) → `int` не бывает `null`, поэтому атрибут бесполезен; используйте `int?` или `[Range]`. (RU)
- Доверие только клиентской валидации → всегда проверяйте на сервере. (RU)
- Возврат `BadRequest("message")` вместо `ValidationProblem` → ломает единый контракт RFC 7807. (RU)
- Дублирование правил в атрибутах и FluentValidation → рассинхрон и баги при изменении. (RU)
- Игнорирование `CancellationToken` в `ValidateAsync` → утечка ресурсов под нагрузкой. (RU)

- `[Required]` on a non-nullable `int` → a value type is never `null`, so the attribute is useless; use `int?` or `[Range]`. (EN)
- Trusting client-side validation only → always validate on the server. (EN)
- Returning `BadRequest("message")` instead of `ValidationProblem` → breaks the RFC 7807 contract. (EN)
- Duplicating rules across attributes and FluentValidation → desync and bugs on change. (EN)
- Ignoring `CancellationToken` in `ValidateAsync` → resource leak under load. (EN)

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Каждый входной DTO покрыт атрибутами или валидатором FluentValidation. (RU)
- [ ] Контроллер помечен `[ApiController]`, ошибки возвращаются как `ValidationProblem`. (RU)
- [ ] Сложные правила вынесены в FluentValidation, а не в атрибуты. (RU)
- [ ] Есть второй уровень проверки инвариантов в домене/бизнес-логике. (RU)
- [ ] Сообщения об ошибках локализованы и понятны пользователю. (RU)
- [ ] Клиентская валидация лишь дублирует серверную, не заменяет её. (RU)

- [ ] Every input DTO is covered by attributes or a FluentValidation validator. (EN)
- [ ] Controller is decorated with `[ApiController]`, errors return as `ValidationProblem`. (EN)
- [ ] Complex rules live in FluentValidation, not in attributes. (EN)
- [ ] A second invariant-check level exists in the domain/business logic. (EN)
- [ ] Error messages are localized and user-friendly. (EN)
- [ ] Client-side validation only mirrors the server-side one, never replaces it. (EN)

#### Ресурсы / Resources

- [Microsoft Learn — https://learn.microsoft.com/aspnet/core/mvc/models/validation]
- [FluentValidation documentation — https://docs.fluentvalidation.net/]
- [RFC 7807 Problem Details — https://datatracker.ietf.org/doc/html/rfc7807]

---

[⬆ К модулю M14](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
