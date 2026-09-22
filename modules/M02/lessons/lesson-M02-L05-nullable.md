---
[← Предыдущий: M02-L04](lesson-M02-L04-operators.md) | [⬆ К модулю M02](../README.md) | [Следующий: M02-L06 →](lesson-M02-L06-type-conversion.md)
---

### Урок M02-L05: null и nullable-типы (int?) / null and nullable types (int?)

**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

В C# типы делятся на две большие семьи: ссылочные типы (class, string, массивы) и типы-значения (int, bool, DateTime, struct). Переменная ссылочного типа хранит адрес объекта в куче, поэтому она может указывать «в никуда» — это и есть `null`. Переменная типа-значения хранит само значение прямо в памяти и по своей природе не может быть пустой: `int x` всегда содержит какое-то целое число, даже если вы его не инициализировали (тогда там `0`).

Но в реальных задачах «отсутствие значения» встречается повсеместно: возраст пациента неизвестен, дата окончания подписки ещё не задана, ответ от сервера не пришёл. Для таких случаев придумали **nullable value types** — обёртку `Nullable<T>`, которая добавляет к значению типа `T` ещё один бит «значение отсутствует». Сахарный синтаксис `int?` означает ровно `Nullable<int>`. У такой переменной есть свойство `HasValue` (true, если значение есть) и свойство `Value` (само значение; обращение к `Value`, когда `HasValue == false`, бросает `InvalidOperationException`). Проверяйте `HasValue` перед чтением `Value` либо используйте безопасные операторы.

Аналогия: представьте почтовую ячейку, в которую кладут записки с числами. Обычный `int` — это ячейка, в которой всегда лежит хотя бы нулевая записка. `int?` — это ячейка с флажком «пусто»: флажок поднят, если записки нет, и опущен, если она там. `HasValue` проверяет флажок, `Value` достаёт записку (и хлопает дверью, если флажок поднят).

Чтобы удобно работать с `null`, в языке есть три оператора. **Null-coalescing** `??`: `a ?? b` вернёт `a`, если оно не null, иначе `b`. Можно chaining: `a ?? b ?? c ?? 0`. **Null-coalescing assignment** `??=` (C# 8): `x ??= 5;` присвоит `5` только если `x` равен null. **Null-conditional** `?.`: `obj?.Method()` вызовет метод, только если `obj` не null, иначе вернёт null. В цепочке `a?.b?.c` вычисление прерывается на первом null. В сочетании с `??` это самый безопасный способ пройти по графу объектов: `person?.Address?.City ?? "неизвестно"`.

Начиная с **C# 8** появились **nullable reference types (NRT)**. Это не новая runtime-сущность, а система статических предупреждений компилятора. Включив `<Nullable>enable</Nullable>`, вы говорите компилятору: считай, что обычный `string` не должен быть null, а `string?` — может. Если вы присвоите null обычной `string`, получите предупреждение CS8602/CS8600. Если разыменуете `string?` без проверки — CS8602. Это статический контракт, который ловит NullReferenceException ещё на этапе компиляции. Компилятор применяет flow-анализ: после `if (s != null)` переменная считается «не-null» в этой ветке.

Частые предупреждения, которые нужно понимать: CS8600 (приведение возможно-null к не-null типу), CS8602 (разыменование возможно-null значения), CS8603 (возможный возврат null-ссылки), CS8625 (присвоение null не-null типу). Не заглушайте их `!` (null-forgiving operator) бездумно: `x!.Foo()` говорит компилятору «я уверен, что x не null», но не добавляет реальной проверки. Используйте `!` только когда у вас есть инвариант, недоступный анализу.

Правило хорошего тона: для типов-значений используйте `T?`, когда «нет значения» — легитимное состояние; для ссылочных типов включайте NRT во всём проекте и явно помечайте nullable-поля через `?`. Никогда не возвращайте null из метода, если есть разумный default (пустая коллекция, `string.Empty`).

#### Theory (EN)

In C# types split into two big families: reference types (class, string, arrays) and value types (int, bool, DateTime, struct). A reference-type variable holds the address of an object on the heap, so it can point "nowhere" — that is `null`. A value-type variable stores the value itself in memory and by nature cannot be empty: an `int x` always holds some integer, even when uninitialized (then it is `0`).

But "no value" is everywhere in real code: a patient's age is unknown, a subscription end date is not set yet, a server response did not arrive. For these cases C# offers **nullable value types** — a wrapper `Nullable<T>` that adds to the value of `T` one extra bit meaning "value is missing". The sugared syntax `int?` means exactly `Nullable<int>`. Such a variable has a `HasValue` property (true when a value is present) and a `Value` property (the value itself; reading `Value` when `HasValue == false` throws `InvalidOperationException`). Always check `HasValue` before `Value`, or use the safe operators.

Analogy: imagine a mailbox that holds notes with numbers. A plain `int` is a mailbox where at least a zero note always sits. An `int?` is a mailbox with an "empty" flag: the flag is up when there is no note, down when there is one. `HasValue` reads the flag, `Value` takes the note out (and slams the door if the flag was up).

Three operators make `null` ergonomic. **Null-coalescing** `??`: `a ?? b` returns `a` when it is not null, otherwise `b`. Chaining works: `a ?? b ?? c ?? 0`. **Null-coalescing assignment** `??=` (C# 8): `x ??= 5;` assigns `5` only if `x` is null. **Null-conditional** `?.`: `obj?.Method()` calls the method only when `obj` is not null, otherwise it yields null. In a chain `a?.b?.c` evaluation stops at the first null. Combined with `??` this is the safest way to walk an object graph: `person?.Address?.City ?? "unknown"`.

Starting with **C# 8** we have **nullable reference types (NRT)**. This is not a new runtime entity — it is a system of static compiler warnings. By enabling `<Nullable>enable</Nullable>` you tell the compiler: treat a plain `string` as one that must not be null, and `string?` as one that may be. Assigning null to a plain `string` gives warnings CS8602/CS8600. Dereferencing a `string?` without a check gives CS8602. It is a static contract that catches NullReferenceException at compile time. The compiler uses flow analysis: after `if (s != null)` the variable is treated as non-null in that branch.

Warnings to understand: CS8600 (converting a maybe-null value to a non-null type), CS8602 (dereferencing a maybe-null value), CS8603 (possible null reference return), CS8625 (turning a null literal into a non-null type). Do not silence them blindly with `!` (the null-forgiving operator): `x!.Foo()` tells the compiler "I am sure x is not null" but adds no real check. Use `!` only when you hold an invariant the analyzer cannot see.

Good-taste rule: use `T?` for value types only when "no value" is a legitimate state; for reference types enable NRT across the whole project and explicitly mark nullable fields with `?`. Never return null from a method when a reasonable default exists (empty collection, `string.Empty`).

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — top-level statements, Nullable enable
// Демонстрация nullable value types, операторов ??, ?., ?.= и NRT
// Demo of nullable value types, the ??, ?., ??= operators and NRT

#nullable enable

using System;
using System.Collections.Generic;
using System.Linq;

// --- 1. Nullable value types: int? это Nullable<int> ---
// --- 1. Nullable value types: int? is Nullable<int> ---
int? age = null;            // «возраст неизвестен» / "age unknown"
int? score = 42;            // значение есть / value present

// Безопасное чтение через HasValue / Value
// Safe reading via HasValue / Value
if (age.HasValue)
{
    Console.WriteLine($"Возраст / Age: {age.Value}");
}
else
{
    Console.WriteLine("Возраст не задан / Age is not set");
}

// --- 2. Оператор ?? — значение по умолчанию ---
// --- 2. The ?? operator — a default value ---
int effectiveAge = age ?? 0;            // null превращается в 0 / null becomes 0
string city = "Saint-Petersburg";
string display = city ?? "неизвестно";  // "неизвестно" / "unknown" if city were null
Console.WriteLine($"effectiveAge={effectiveAge}, display={display}");

// --- 3. Оператор ??= — присвоить, если null ---
// --- 3. The ??= operator — assign only if null ---
string? name = null;
name ??= "Гость";                        // теперь name == "Гость" / now name == "Guest"
name ??= "Игнор";                        // не сработает, name уже не null / no-op, name is not null now
Console.WriteLine($"name={name}");

// --- 4. Оператор ?. — безопасный обход графа объектов ---
// --- 4. The ?. operator — safely walk an object graph ---
Person? p = new Person { Name = "Alice", Address = new Address { City = "Berlin" } };
Person? nobody = null;

string? pCity = p?.Address?.City;       // "Berlin"
string? nCity = nobody?.Address?.City;  // null, без исключения / null, no exception
string safe = nobody?.Address?.City ?? "нет города";   // "нет города" / "no city"
Console.WriteLine($"pCity={pCity}, nCity={nCity}, safe={safe}");

// --- 5. Pattern matching с nullable ---
// --- 5. Pattern matching with nullable ---
string Describe(int? value) => value switch
{
    null      => "значение отсутствует / missing",
    0         => "ноль / zero",
    < 0       => $"отрицательное: {value}",
    > 0 and < 100 => $"малое положительное: {value}",
    _         => $"большое значение: {value}"
};

Console.WriteLine(Describe(null));
Console.WriteLine(Describe(0));
Console.WriteLine(Describe(-7));
Console.WriteLine(Describe(42));
Console.WriteLine(Describe(999));

// --- 6. Nullable reference types: контракты и предупреждения ---
// --- 6. Nullable reference types: contracts and warnings ---
// FindById возвращает Person? — вызывающий обязан проверить на null
// FindById returns Person? — the caller must null-check
Person? FindById(IEnumerable<Person> people, string name) =>
    people.FirstOrDefault(x => x.Name == name);

Person? found = FindById(new[] { p }, "Alice");
// После проверки компилятор считает found не-null в этой ветке
// After the check the compiler treats found as non-null in this branch
if (found is not null)
{
    Console.WriteLine($"Найден / Found: {found.Name}");
}

// --- 7. Raw string literal для безопасного форматирования отчёта ---
// --- 7. Raw string literal to safely format a report ---
int?[] readings = [null, 10, 22, null, 30];
string report = $"""
    Отчёт по показаниям / Sensor report
    Всего датчиков / total sensors : {readings.Length}
    Активных / active            : {readings.Count(r => r.HasValue)}
    Среднее (без null) / average : {readings.Where(r => r.HasValue).Select(r => r!.Value).Average():F1}
    """;
Console.WriteLine(report);

// --- 8. Метод с nullable-контрактом: что компилятор проверяет ---
// --- 8. A method with a nullable contract: what the compiler checks ---
// Параметр 'label' не должен быть null; 'suffix' может быть null
// 'label' must not be null; 'suffix' may be null
string Build(string label, string? suffix)
{
    // label.ToUpper() — без предупреждения: label не-null по контракту
    // label.ToUpper() — no warning: label is non-null by contract
    string head = label.ToUpper();

    // suffix!.ToUpper() было бы грубо; лучше проверить через pattern
    // suffix!.ToUpper() would be blunt; better to check with a pattern
    string tail = suffix is { } s ? $":{s.ToUpper()}" : string.Empty;
    return $"{head}{tail}";
}

Console.WriteLine(Build("hello", "world"));
Console.WriteLine(Build("hello", null));

// --- Типы для примера ---
// --- Types used above ---
class Person
{
    public required string Name { get; init; }
    public Address? Address { get; init; }
}

class Address
{
    public required string City { get; init; }
}
```

#### Best Practices
- Используйте `T?` для типов-значений только когда «нет значения» — легитимное состояние домена, а не как способ отложить инициализацию.
- Включайте `<Nullable>enable</Nullable>` во всём проекте с первого дня; помечайте nullable-поля и параметры явно через `?`.
- Предпочитайте `x ?? defaultValue` и `obj?.Member` ручным `if (x != null)` проверкам — код короче и читаемее.
- Не возвращайте null из метода, если существует осмысленный default: пустая коллекция, `string.Empty`, `Array.Empty<T>()`.
- Документируйте nullable-контракт метода в XML-комментариях: `<param>`/`<returns>` должны уточнять, когда возможен null.

- Use `T?` for value types only when "no value" is a legitimate domain state, not as a way to defer initialization.
- Enable `<Nullable>enable</Nullable>` project-wide from day one; mark nullable fields and parameters explicitly with `?`.
- Prefer `x ?? defaultValue` and `obj?.Member` over manual `if (x != null)` checks — shorter and more readable code.
- Do not return null from a method when a meaningful default exists: an empty collection, `string.Empty`, `Array.Empty<T>()`.
- Document the nullable contract of a method in XML comments: `<param>`/`<returns>` should clarify when null is possible.

#### Частые ошибки / Common Mistakes
- Чтение `.Value` без проверки `.HasValue` → `InvalidOperationException`. Всегда проверяйте `HasValue` или используйте `??`/pattern matching.
- Использование `x!` (null-forgiving) как «затычки» для предупреждения CS8602 → runtime `NullReferenceException`. Применяйте `!` только при доказанном инварианте, недоступном анализу.
- Присвоение `null` обычной `string` при включённом NRT → CS8625. Пометьте тип как `string?`, если null действительно возможен.
- Цепочка `a.b.c.d` без `?.` на ссылочных типах → NullReferenceException на любом звене. Замените на `a?.b?.c?.d ?? fallback`.
- Возврат `null` из метода, возвращающего коллекцию, вместо `Enumerable.Empty<T>()` → заставляет каждого вызывающего писать null-check.

- Reading `.Value` without checking `.HasValue` → `InvalidOperationException`. Always check `HasValue` or use `??`/pattern matching.
- Using `x!` (null-forgiving) as a "plug" for warning CS8602 → a runtime `NullReferenceException`. Apply `!` only with a proven invariant the analyzer cannot see.
- Assigning `null` to a plain `string` with NRT enabled → CS8625. Mark the type `string?` if null is genuinely possible.
- A chain `a.b.c.d` with no `?.` on reference types → NullReferenceException at any link. Replace with `a?.b?.c?.d ?? fallback`.
- Returning `null` from a collection-returning method instead of `Enumerable.Empty<T>()` → forces every caller to null-check.

#### Чек-лист самопроверки / Self-check Checklist
- [ ] Я понимаю разницу между ссылочными типами (могут быть null по природе) и типами-значениями (не могут без `T?`).
- [ ] Я могу объяснить, что `int?` — это синтаксический сахар для `Nullable<int>`.
- [ ] Я всегда проверяю `HasValue` перед чтением `Value` либо использую `??`/pattern matching.
- [ ] Я знаю три оператора: `??`, `??=`, `?.` — и применяю их для безопасной работы с null.
- [ ] Я включил NRT в проекте и понимаю, что `string` и `string?` — разные статические контракты.
- [ ] Я не заглушаю предупреждения CS86xx оператором `!` без доказанного инварианта.
- [ ] Я предпочитаю возвращать пустые коллекции/`string.Empty` вместо null.

- [ ] I understand the difference between reference types (null by nature) and value types (not null without `T?`).
- [ ] I can explain that `int?` is syntactic sugar for `Nullable<int>`.
- [ ] I always check `HasValue` before reading `Value`, or use `??`/pattern matching.
- [ ] I know the three operators: `??`, `??=`, `?.` — and use them to work with null safely.
- [ ] I enabled NRT in the project and understand that `string` and `string?` are different static contracts.
- [ ] I do not silence CS86xx warnings with `!` without a proven invariant.
- [ ] I prefer returning empty collections/`string.Empty` instead of null.

#### Ресурсы / Resources
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/nullable-value-types — Nullable value types (Типы значений, допускающие значение NULL)
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/nullable-references — Nullable reference types (Ссылочные типы, допускающие значение NULL)

---
[← Предыдущий: M02-L04](lesson-M02-L04-operators.md) | [⬆ К модулю M02](../README.md) | [Следующий: M02-L06 →](lesson-M02-L06-type-conversion.md)
---
