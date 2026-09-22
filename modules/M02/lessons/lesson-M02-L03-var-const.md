---
[← Предыдущий: M02-L02](lesson-M02-L02-primitives.md) | [⬆ К модулю M02](../README.md) | [Следующий: M02-L04 →](lesson-M02-L04-operators.md)
---

### Урок M02-L03: Объявление переменных, var, константы / Declaring variables, var, constants

**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

В C# каждая переменная имеет тип, имя и значение. Прежде чем использовать переменную, её нужно **объявить** — то есть сказать компилятору: «выдели мне память под данные такого вида и назови это так-то». Самый явный способ — указать тип напрямую:

```csharp
int age = 30;
string name = "Анна";
bool isActive = true;
```

Это **явная типизация**. Компилятор и читатель сразу видят, какой тип данных хранится. Аналогия: табличка на ящике с подписью «Здесь лежат целые числа».

С версии C# 3.0 появилось ключевое слово `var` — **неявная типизация**. Пишете `var`, а компилятор сам выводит тип из правой части:

```csharp
var age = 30;        // int
var name = "Анна";   // string
var pi = 3.14;       // double
```

Важно понимать: `var` **не делает переменную динамической**. Тип фиксируется один раз на этапе компиляции и больше не меняется. Это просто сокращение — компилятор «подставляет» настоящий тип за вас. Аналогия: вы говорите «положи это в подходящий ящик», а складской робот сам выбирает размер ящика по содержимому. После этого ящик остаётся того же размера навсегда.

**Когда `var` уместен?** Главное правило — читатель должен без труда понимать тип из контекста.

- ✅ Уместно: `var customer = new Customer();` — тип очевиден из `new`.
- ✅ Уместно: `var files = Directory.GetFiles(path);` — LINQ и сложные дженерик-типы (`IEnumerable<KeyValuePair<string,int>>`) лучше спрятать в `var`, иначе строка становится нечитаемой.
- ❌ Неуместно: `var data = GetData();` — непонятно, что вернётся; лучше указать тип явно или переименовать метод.
- ❌ Неуместно: числовые литералы, где тип влияет на смысл — `var x = 0;` это `int`, а не `double` или `long`.

**Константы.** Ключевое слово `const` создаёт значение, известное **на этапе компиляции**:

```csharp
const double Pi = 3.14159;
const string AppName = "MyApp";
```

Такие значения «вшиваются» в сборку: каждое использование `Pi` компилятор заменяет самим числом. Поэтому `const` работает только с примитивами, строками и `null`-ссылками — никаких вычислений в рантайме. Имена констант принято писать в **PascalCase**: `MaxRetryCount`, `DefaultTimeout`.

**`const` vs `readonly`.** Это частая путаница.

- `const` — значение известно при компиляции, неизменно, только примитивы/строки, доступно через имя класса (`Math.PI`).
- `readonly` — значение задаётся при объявлении или **в конструкторе** в рантайме, после чего не меняется. Подходит для полей классов со сложными типами, а также для «констант», зависящих от конфигурации.

```csharp
public class Configuration
{
    public const int MaxConnections = 100;          // compile-time
    public readonly DateTime CreatedAt;             // runtime, задаётся в конструкторе
    public Configuration() => CreatedAt = DateTime.UtcNow;
}
```

Аналогия: `const` — это цифра, выгравированная на табличке на заводе; `readonly` — это надпись, которую делают один раз при установке, но потом уже не трогают.

**Именование.** Локальные переменные и параметры в C# принято называть в **camelCase**: `firstName`, `totalCount`, `isReady`. Первое слово с маленькой буквы, каждое следующее — с большой. Поля классов — тоже camelCase (часто с подчёркиванием: `_firstName`). Имена должны быть осмысленными: `d` плохое имя, `daysRemaining` — хорошее. Хорошие имена заменяют комментарии и делают код самодокументируемым.

#### Theory (EN)

In C#, every variable has a **type**, a **name**, and a **value**. Before you can use a variable, you must **declare** it — that is, tell the compiler: “set aside memory for this kind of data and call it by this name.” The most explicit way is to write the type directly:

```csharp
int age = 30;
string name = "Anna";
bool isActive = true;
```

This is **explicit typing**. Both the compiler and the reader immediately see what kind of data lives in the box. Analogy: a label on a drawer that says “integers only.”

Since C# 3.0 you can use the `var` keyword for **implicit typing**. You write `var`, and the compiler infers the real type from the right-hand side:

```csharp
var age = 30;        // int
var name = "Anna";   // string
var pi = 3.14;       // double
```

The key insight: `var` does **not** make a variable dynamic. The type is fixed once, at compile time, and never changes afterward. It is just shorthand — the compiler fills in the real type for you. Analogy: you say “put this in a fitting drawer,” and the warehouse robot picks the drawer size based on the contents. After that, the drawer keeps that size forever.

**When is `var` appropriate?** The guiding rule is that the reader should be able to tell the type without effort.

- ✅ Appropriate: `var customer = new Customer();` — the type is obvious from `new`.
- ✅ Appropriate: `var files = Directory.GetFiles(path);` — LINQ results and verbose generics (`IEnumerable<KeyValuePair<string,int>>`) read better with `var`.
- ❌ Not appropriate: `var data = GetData();` — unclear what comes back; prefer an explicit type or a more descriptive method name.
- ❌ Not appropriate for numeric literals where type carries meaning — `var x = 0;` is `int`, not `double` or `long`.

**Constants.** The `const` keyword produces a value known **at compile time**:

```csharp
const double Pi = 3.14159;
const string AppName = "MyApp";
```

Such values are baked into the assembly: every use of `Pi` is replaced by the literal number. That is why `const` only works with primitives, strings, and `null` references — no runtime computation is allowed. By convention, constant names use **PascalCase**: `MaxRetryCount`, `DefaultTimeout`.

**`const` vs `readonly`.** This is a common point of confusion.

- `const` — value known at compile time, immutable, primitives/strings only, accessed through the class name (`Math.PI`).
- `readonly` — value set at declaration or **in a constructor** at runtime, and immutable afterward. Good for class fields of complex types, and for “constants” that depend on configuration.

```csharp
public class Configuration
{
    public const int MaxConnections = 100;          // compile-time
    public readonly DateTime CreatedAt;             // runtime, set in constructor
    public Configuration() => CreatedAt = DateTime.UtcNow;
}
```

Analogy: `const` is a number engraved on a plate at the factory; `readonly` is a label written once during installation and then never touched again.

**Naming.** Local variables and parameters in C# are written in **camelCase**: `firstName`, `totalCount`, `isReady`. The first word starts lowercase, every following word is capitalized. Class fields are also camelCase (often prefixed with an underscore: `_firstName`). Names should be meaningful: `d` is a poor name, `daysRemaining` is a good one. Good names replace comments and make code self-documenting.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — top-level statements
// Демонстрация объявления переменных, var, const и readonly
// Demonstration of variable declarations, var, const and readonly

using System;
using System.Collections.Generic;
using System.IO;

// 1) Явное объявление с типом / Explicit typed declaration
int age = 30;
string name = "Anna";
bool isActive = true;
Console.WriteLine($"{name}, {age}, active={isActive}");

// 2) var — неявная типизация, тип выводится один раз / var: type inferred once
var city = "Berlin";          // string
var temperature = 36.6;       // double
var scores = new List<int> { 7, 8, 9 };   // List<int>

// var НЕ динамическая: переназначить другим типом нельзя / var is NOT dynamic
// city = 42;  // ❌ compile error: cannot convert int to string
city = "Paris";               // ✅ same type, OK
Console.WriteLine($"{city}, {temperature}°C, scores count={scores.Count}");

// 3) Когда var уместен / When var is appropriate
var files = Directory.GetFiles(Directory.GetCurrentDirectory()); // тип очевиден
foreach (var file in files) // var уместен в foreach — тип понятен из коллекции
{
    Console.WriteLine(file);
}

// 4) const — значение на этапе компиляции / const: compile-time value
const double Pi = 3.14159;
const string AppName = "MyApp";
const int MaxRetryCount = 3;

// const нельзя вычислять в рантайме / const cannot be computed at runtime
// const DateTime CreatedAt = DateTime.UtcNow; // ❌ compile error
// Pi = 3.0; // ❌ compile error: const is immutable

double area = Pi * 5 * 5;
Console.WriteLine($"{AppName}: area={area:F2}, retries={MaxRetryCount}");

// 5) readonly vs const в классе / readonly vs const in a class
var config = new Configuration();
Console.WriteLine($"MaxConnections={Configuration.MaxConnections}, CreatedAt={config.CreatedAt:O}");

// 6) Pattern matching с типизированным паттерном (C# 12) / Typed pattern matching
// ВАЖНО: relational-паттерны (> <) требуют сравнимого типа. object не сравним,
// поэтому сначала проверяем тип через `is int value`, затем relational.
// NOTE: relational patterns need a comparable type. object is not comparable,
// so we first match the type with `is int value`, then apply relational patterns.
object box = 42;
if (box is int value and > 10 and < 100) // value — int, relational-паттерны валидны
{
    Console.WriteLine($"Boxed value in range: {value}");
}

// 7) Именование camelCase / camelCase naming
int totalCount = 0;        // ✅ осмысленное имя
bool isReady = false;      // ✅ булевы — с префиксом is/has
string firstName = "Anna"; // ✅ camelCase
// var d = 0;              // ❌ слишком короткое, непонятно

class Configuration
{
    public const int MaxConnections = 100;        // compile-time constant, PascalCase
    public readonly DateTime CreatedAt;           // runtime, immutable after construction

    public Configuration() => CreatedAt = DateTime.UtcNow;
}
```

#### Best Practices

- Используйте `var`, когда тип очевиден из правой части (`new`, LINQ, `foreach`), и явный тип в остальных случаях — читаемость важнее краткости.
- Предпочитайте `const` для истинных неизменных значений, известных на этапе компиляции; используйте `readonly`, когда значение определяется в рантайме (например, в конструкторе).
- Давайте переменным осмысленные имена в camelCase: `daysRemaining`, а не `d`. Хорошие имена заменяют комментарии.
- Не используйте `var` с числовыми литералами, где тип влияет на смысл: `var x = 0;` даёт `int`, а не `long` или `double`.

- Use `var` when the type is obvious from the right-hand side (`new`, LINQ, `foreach`), and an explicit type otherwise — readability beats brevity.
- Prefer `const` for true compile-time immutable values; use `readonly` when the value is resolved at runtime (for example in a constructor).
- Give variables meaningful camelCase names: `daysRemaining`, not `d`. Good names replace comments.
- Do not use `var` with numeric literals where type carries meaning: `var x = 0;` yields `int`, not `long` or `double`.

#### Частые ошибки / Common Mistakes

- Считать `var` динамической типизацией → помнить, что тип фиксируется на компиляции и не меняется; `var` лишь сокращение. (RU)
- Использовать `var` там, где тип непонятен (`var data = GetData();`) → указывать тип явно или переименовывать метод/переменную так, чтобы тип читался из контекста. (RU)
- Применять `const` к типам, требующим рантайма (`DateTime`, массивы, объекты) → использовать `readonly` и инициализировать в конструкторе. (RU)
- Путать неизменность поля и константу класса: `const` «вшит» в сборку и при изменении требует перекомпиляции всех потребителей → для значений, которые могут меняться между релизами, использовать `readonly` или статическую конфигурацию. (RU)
- Имена вроде `d`, `s`, `x1` без контекста → давать описательные имена в camelCase, отражающие смысл. (RU)

- Treating `var` as dynamic typing → remember the type is fixed at compile time and never changes; `var` is only shorthand. (EN)
- Using `var` where the type is unclear (`var data = GetData();`) → write the type explicitly or rename the method/variable so the type reads from context. (EN)
- Applying `const` to runtime types (`DateTime`, arrays, objects) → use `readonly` and initialize it in a constructor. (EN)
- Confusing field immutability with a class constant: `const` is baked into the assembly, so changing it forces recompilation of all consumers → for values that may change between releases, use `readonly` or static configuration. (EN)
- Names like `d`, `s`, `x1` with no context → use descriptive camelCase names that convey meaning. (EN)

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Я могу объяснить разницу между явным объявлением типа и `var`. (RU)
- [ ] Я понимаю, что `var` фиксирует тип на этапе компиляции и не делает переменную динамической. (RU)
- [ ] Я знаю, когда `var` уместен, а когда лучше указать тип явно. (RU)
- [ ] Я могу объяснить отличие `const` (compile-time) от `readonly` (runtime). (RU)
- [ ] Я называю локальные переменные в camelCase осмысленными именами. (RU)

- [ ] I can explain the difference between an explicit type declaration and `var`. (EN)
- [ ] I understand that `var` fixes the type at compile time and does not make a variable dynamic. (EN)
- [ ] I know when `var` is appropriate and when an explicit type is better. (EN)
- [ ] I can explain the difference between `const` (compile-time) and `readonly` (runtime). (EN)
- [ ] I name local variables in camelCase with meaningful names. (EN)

#### Ресурсы / Resources

- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/implicitly-typed-local-variables — Неявно типизированные локальные переменные / Implicitly typed local variables
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/const — Ключевое слово const (справочник по C#) / const keyword (C# reference)

---
[← Предыдущий: M02-L02](lesson-M02-L02-primitives.md) | [⬆ К модулю M02](../README.md) | [Следующий: M02-L04 →](lesson-M02-L04-operators.md)
---
