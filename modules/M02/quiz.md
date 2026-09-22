# Quiz модуля M02 / Quiz: M02
# Типы, переменные, операторы / Types, Variables, Operators

> 14 вопросов / 14 questions
> Вопросы на понимание, не запоминание / Understanding, not memorization
> Источник: уроки M02-L01…L08 / Source: lessons M02-L01…L08
> C# 12 / .NET 8

---

## Вопрос 1 / Question 1 (Multiple Choice / Множественный выбор)

**Вопрос / Question:**
Что выведет следующий код? `Money` — это `readonly struct` с полем `Amount`.
What will the following code print? `Money` is a `readonly struct` with an `Amount` field.

```csharp
public readonly struct Money(decimal amount, string currency)
{
    public decimal Amount { get; } = amount;
    public string Currency { get; } = currency;
}

var a = new Money(100m, "USD");
var b = a;                        // копия / copy
b = b with { Amount = 200m };     // with-expression: новая копия / new copy
Console.WriteLine($"a={a.Amount}, b={b.Amount}");
```

**Варианты / Options:**
- A) a=200, b=200
- B) a=100, b=200
- C) a=100, b=100
- D) a=200, b=100

**Правильный ответ / Correct:** B

**Объяснение / Explanation:**
`Money` — значимый тип (struct), поэтому `var b = a` создаёт **побитовую копию**: `a` и `b` — две независимые переменные. Выражение `b with { Amount = 200m }` строит новый `Money` и присваивает его `b`, не затрагивая `a`. Поэтому `a` остаётся 100, `b` становится 200.
`Money` is a value type (struct), so `var b = a` makes a **bitwise copy** — `a` and `b` are independent. The `with` expression builds a new `Money` assigned to `b`, leaving `a` untouched. So `a` stays 100 and `b` becomes 200.
- A неверно / wrong: это было бы ссылкой для `class` (reference copy + shared mutation), но не для `struct`.
- C неверно / wrong: `b` реально изменён через `with`.
- D неверно / wrong: изменение `b` не может отразиться на `a` у value-типа.

**Сложность / Difficulty:** 2/5

**Тип / Type:** Understanding

---

## Вопрос 2 / Question 2 (Multiple Choice / Множественный выбор)

**Вопрос / Question:**
Что выведет код? / What will the code print?

```csharp
double d = 0.1 + 0.2;
decimal m = 0.1m + 0.2m;
Console.WriteLine($"{d == 0.3} {m == 0.3m}");
```

**Варианты / Options:**
- A) True True
- B) False True
- C) False False
- D) True False

**Правильный ответ / Correct:** B

**Объяснение / Explanation:**
`double` хранит значения в двоичной форме (IEEE 754), поэтому `0.1 + 0.2` не равно в точности `0.3` — небольшая погрешность делает сравнение `False`. `decimal` — 128-битный тип с десятичным (base-10) представлением, `0.1m + 0.2m` даёт ровно `0.3m`, поэтому сравнение `True`.
`double` uses binary IEEE 754 storage, so `0.1 + 0.2` is not exactly `0.3` — the tiny drift makes the comparison `False`. `decimal` is a 128-bit base-10 type, so `0.1m + 0.2m` equals exactly `0.3m` and the comparison is `True`.
- A неверно / wrong: `double` даёт погрешность.
- C неверно / wrong: `decimal` точен для десятичных дробей.
- D неверно / wrong: наоборот — `double` «грязнее», `decimal` точнее.

**Сложность / Difficulty:** 3/5

**Тип / Type:** Understanding

---

## Вопрос 3 / Question 3 (Multiple Choice / Множественный выбор)

**Вопрос / Question:**
Какие типы выведет `var`? / What types does `var` infer?

```csharp
var x = 0;              // ?
var y = 1.5;            // ?
var z = 8_000_000_000L; // ?
```

**Варианты / Options:**
- A) x=`int`, y=`double`, z=`int`
- B) x=`int`, y=`float`, z=`long`
- C) x=`int`, y=`double`, z=`long`
- D) x=`long`, y=`double`, z=`long`

**Правильный ответ / Correct:** C

**Объяснение / Explanation:**
`var` фиксирует тип на этапе компиляции по правой части. Целочисленный литерал `0` по умолчанию — `int`. Дробный литерал `1.5` без суффикса — `double` (не `float`; для `float` нужен суффикс `f`). Литерал `8_000_000_000L` с суффиксом `L` — `long` (без `L` он не помещается в `int` и вызвал бы ошибку компиляции). Подчёркивание — лишь разделитель разрядов, на тип не влияет.
`var` fixes the type at compile time from the right-hand side. The integer literal `0` defaults to `int`. The fractional literal `1.5` without a suffix is `double` (not `float`; `float` needs the `f` suffix). The literal `8_000_000_000L` with the `L` suffix is `long` (without `L` it would not fit in `int` and would not compile). Underscores are only digit separators and do not affect the type.
- A неверно / wrong: `8_000_000_000L` — это `long`, а не `int`.
- B неверно / wrong: `1.5` — `double`, а не `float`.
- D неверно / wrong: `0` без суффикса — `int`, а не `long`.

**Сложность / Difficulty:** 2/5

**Тип / Type:** Understanding

---

## Вопрос 4 / Question 4 (Code Completion / Допиши код)

**Вопрос / Question:**
Заполни пропуск, чтобы безопасно разобрать ввод пользователя без исключений. / Fill the blank to safely parse user input without throwing exceptions.

```csharp
string? input = Console.ReadLine();
if (int._____(input, out int age))
{
    Console.WriteLine($"Возраст / Age: {age}");
}
else
{
    Console.WriteLine("Не число / Not a number");
}
```

**Варианты / Options:**
- A) `Parse`
- B) `TryParse`
- C) `Convert`
- D) `SafeParse`

**Правильный ответ / Correct:** B

**Объяснение / Explanation:**
`int.TryParse(input, out int age)` возвращает `bool` и никогда не бросает исключений: при неудче `age` получает `0`, а метод возвращает `false`. Это правильный выбор для ввода, формат которого неизвестен. Объявление `out int age` прямо в вызове доступно с C# 7.
`int.TryParse(input, out int age)` returns a `bool` and never throws: on failure `age` becomes `0` and the method returns `false`. This is the right choice for input of unknown format. Declaring `out int age` inline in the call is available since C# 7.
- A неверно / wrong: `int.Parse` бросает `FormatException`/`ArgumentNullException` на кривом вводе — приложение упадёт.
- C неверно / wrong: `Convert` — это класс (`Convert.ToInt32`), не метод `int`, и он бросает `FormatException` на нечисловой строке.
- D неверно / wrong: метода `SafeParse` в BCL не существует.

**Сложность / Difficulty:** 1/5

**Тип / Type:** Code Completion

---

## Вопрос 5 / Question 5 (Multiple Choice / Множественный выбор)

**Вопрос / Question:**
Что выведет код? / What will the code print?

```csharp
int a = 7, b = 2;
Console.WriteLine($"{a / b} {a % b} {7.0 / b}");
```

**Варианты / Options:**
- A) 3.5 1 3.5
- B) 3 1 3.5
- C) 3 1 3
- D) 3.5 0 3.5

**Правильный ответ / Correct:** B

**Объяснение / Explanation:**
Деление двух `int` всегда **целочисленное**: `7 / 2 == 3` (дробная часть отбрасывается, не округляется). Остаток `7 % 2 == 1`. А вот `7.0 / b`: один операнд `double`, поэтому `b` неявно приводится к `double` и результат `3.5`.
Division of two `int` values is always **integral**: `7 / 2 == 3` (the fractional part is truncated, not rounded). The remainder `7 % 2 == 1`. But `7.0 / b` has a `double` operand, so `b` is implicitly promoted to `double` and the result is `3.5`.
- A неверно / wrong: `7 / 2` — целочисленное, равно `3`, не `3.5`.
- C неверно / wrong: `7.0 / 2` даёт `3.5`, не `3`.
- D неверно / wrong: перепутаны и целочисленное деление, и остаток.

**Сложность / Difficulty:** 2/5

**Тип / Type:** Understanding

---

## Вопрос 6 / Question 6 (Debug / Найди баг)

**Вопрос / Question:**
В коде есть баг. Найди его. / There is a bug in the code. Find it.

```csharp
int? age = null;
Console.WriteLine($"Возраст / Age: {age.Value}");
```

**Варианты / Options:**
- A) Бага нет — всё работает / No bug — works fine
- B) `age.Value` бросает `InvalidOperationException`, когда `HasValue == false`; нужно проверить `HasValue` или использовать `age ?? 0`
- C) `int?` не может быть `null` / `int?` cannot be `null`
- D) Нужно вызвать `age.ToString()` вместо `age.Value`

**Правильный ответ / Correct:** B

**Объяснение / Explanation:**
`int?` — это `Nullable<int>`. Чтение `.Value` при `HasValue == false` бросает `InvalidOperationException`. Безопасные варианты: `if (age.HasValue) { ... age.Value ... }`, `age ?? 0` (null-coalescing) или pattern matching `age is int a`. В интерполяции можно писать просто `{age}` — компилятор вызовет `ToString()`, который для null вернёт пустую строку, но `.Value` явно unsafe.
`int?` is `Nullable<int>`. Reading `.Value` when `HasValue == false` throws `InvalidOperationException`. Safe alternatives: `if (age.HasValue) { ... age.Value ... }`, `age ?? 0` (null-coalescing), or pattern matching `age is int a`. In interpolation `{age}` alone is safe (it calls `ToString()`, which returns "" for null), but explicit `.Value` is not.
- A неверно / wrong: код упадёт в рантайме.
- C неверно / wrong: `int?` как раз и создан, чтобы хранить null.
- D неверно / wrong: `ToString()` безопаснее, но это обход проблемы, а не исправление причины — корректный ответ описывает природу бага.

**Сложность / Difficulty:** 3/5

**Тип / Type:** Debug

---

## Вопрос 7 / Question 7 (Multiple Choice / Множественный выбор)

**Вопрос / Question:**
Что произойдёт при выполнении? / What happens at runtime?

```csharp
string? name = null;
bool ok = name != null && name.Length > 0;
Console.WriteLine(ok);
```

**Варианты / Options:**
- A) `NullReferenceException`
- B) Выведется `False`, без исключения / Prints `False`, no exception
- C) Ошибка компиляции / Compile error
- D) Выведется `True` / Prints `True`

**Правильный ответ / Correct:** B

**Объяснение / Explanation:**
Оператор `&&` **short-circuit**: если левый операнд `false`, правый не вычисляется. Здесь `name != null` даёт `false`, поэтому `name.Length` не вызывается — `NullReferenceException` не возникает, `ok = false`. Если бы использовали одиночное `&` (без short-circuit), правая часть вычислялась бы всегда и падала.
The `&&` operator **short-circuits**: if the left operand is `false`, the right is never evaluated. Here `name != null` is `false`, so `name.Length` is not called — no `NullReferenceException`, and `ok = false`. A single `&` (non-short-circuit) would evaluate the right side unconditionally and crash.
- A неверно / wrong: это случилось бы с `&` вместо `&&`.
- C неверно / wrong: код компилируется без ошибок.
- D неверно / wrong: `name` равен null, условие ложно.

**Сложность / Difficulty:** 3/5

**Тип / Type:** Understanding

---

## Вопрос 8 / Question 8 (Multiple Choice / Множественный выбор)

**Вопрос / Question:**
Что выведет код? / What will the code print?

```csharp
string s1 = "hello";
string s2 = s1;
s2 = s2.ToUpper();
Console.WriteLine($"{s1} {s2}");
```

**Варианты / Options:**
- A) HELLO HELLO
- B) hello HELLO
- C) hello hello
- D) HELLO hello

**Правильный ответ / Correct:** B

**Объяснение / Explanation:**
`string` — ссылочный тип, но **неизменяемый (immutable)**. `s2 = s1` копирует ссылку на тот же объект, однако `s2.ToUpper()` **не меняет** исходную строку, а создаёт новый объект `"HELLO"` и присваивает его `s2`. Ссылка `s1` по-прежнему указывает на `"hello"`.
`string` is a reference type but **immutable**. `s2 = s1` copies the reference to the same object, but `s2.ToUpper()` does **not** mutate the original string — it creates a new `"HELLO"` object and assigns it to `s2`. The `s1` reference still points to `"hello"`.
- A неверно / wrong: `s1` не изменился — `ToUpper` создаёт новый объект.
- C неверно / wrong: `s2` действительно переприсвоен на `"HELLO"`.
- D неверно / wrong: перепутаны стороны — `s1` остаётся `"hello"`, `s2` становится `"HELLO"`.

**Сложность / Difficulty:** 2/5

**Тип / Type:** Understanding

---

## Вопрос 9 / Question 9 (Multiple Choice / Множественный выбор)

**Вопрос / Question:**
Что выведет код? / What will the code print?

```csharp
[Flags]
enum Access { None = 0, Read = 1, Write = 2, Exec = 4 }

Access mine = Access.Read | Access.Write;
Console.WriteLine(mine);
Console.WriteLine((mine & Access.Write) == Access.Write);
```

**Варианты / Options:**
- A) `5` / `True`
- B) `Read, Write` / `True`
- C) `Read, Write` / `False`
- D) `3` / `True`

**Правильный ответ / Correct:** B

**Объяснение / Explanation:**
Атрибут `[Flags]` + значения степеней двойки позволяют комбинировать флаги через `|`. `Read | Write` = `1 | 2` = `3`. Благодаря `[Flags]` метод `ToString()` показывает **`Read, Write`**, а не число `3`. Побитовая проверка `(mine & Access.Write) == Access.Write` выделяет бит `Write` (он установлен) → `True`. Это правильный способ проверки флага (наряду с `mine.HasFlag(Access.Write)`).
The `[Flags]` attribute plus power-of-two values let flags combine via `|`. `Read | Write` = `1 | 2` = `3`. Thanks to `[Flags]`, `ToString()` shows **`Read, Write`** instead of the number `3`. The bitwise test `(mine & Access.Write) == Access.Write` isolates the `Write` bit (which is set) → `True`. This is the correct way to test a flag (alongside `mine.HasFlag(Access.Write)`).
- A неверно / wrong: без `[Flags]` `ToString()` показал бы число, но здесь атрибут есть — будет текст.
- C неверно / wrong: бит `Write` установлен, проверка даёт `True`.
- D неверно / wrong: числовое значение `3`, а не `5`, но строка всё равно `Read, Write`.

**Сложность / Difficulty:** 3/5

**Тип / Type:** Understanding

---

## Вопрос 10 / Question 10 (Code Completion / Допиши код)

**Вопрос / Question:**
Заполни пропуск, чтобы деконструировать кортеж в переменные `code` и `msg`. / Fill the blank to deconstruct the tuple into `code` and `msg`.

```csharp
(int Code, string Message) GetResult() => (404, "Not Found");

var (code, msg) = _____;
Console.WriteLine($"{code}: {msg}");
```

**Варианты / Options:**
- A) `GetResult()`
- B) `GetResult().Deconstruct()`
- C) `new GetResult()`
- D) `GetResult.Code, GetResult.Message`

**Правильный ответ / Correct:** A

**Объяснение / Explanation:**
Синтаксис деконструкции `var (code, msg) = GetResult();` разбирает возвращаемый кортеж на отдельные переменные одной строкой — это идиоматичный способ получить несколько значений из метода без `out`-параметров. Компилятор сам вызывает `Deconstruct` под капотом; писать его явно не нужно.
The deconstruction syntax `var (code, msg) = GetResult();` splits the returned tuple into separate variables in one line — the idiomatic way to get multiple values out of a method without `out` parameters. The compiler invokes `Deconstruct` for you; you do not write it explicitly.
- A — верно / correct: `GetResult()` возвращает кортеж, который деконструируется.
- B неверно / wrong: `Deconstruct` вызывается неявно; явный вызов здесь не синтаксис деконструкции.
- C неверно / wrong: кортеж — это не тип с конструктором `new GetResult()`.
- D неверно / wrong: синтаксис деконструкции требует `var (a, b) = выражение-кортеж`, а не перечисление полей.

**Сложность / Difficulty:** 2/5

**Тип / Type:** Code Completion

---

## Вопрос 11 / Question 11 (Multiple Choice / Множественный выбор)

**Вопрос / Question:**
Что выведет код? / What will the code print?

```csharp
double v = 3.5;
int a = (int)v;
int b = Convert.ToInt32(v);
Console.WriteLine($"{a} {b}");
```

**Варианты / Options:**
- A) `3 3`
- B) `4 4`
- C) `3 4`
- D) `4 3`

**Правильный ответ / Correct:** C

**Объяснение / Explanation:**
Cast `(int)v` **отбрасывает** дробную часть → `3` (не округляет!). А `Convert.ToInt32(v)` округляет по правилам банка (round half to even): `3.5` округляется до ближайшего чётного → `4`. Это ключевое отличие `cast` от `Convert`: cast всегда truncates, `Convert` — banker's rounding.
The cast `(int)v` **truncates** the fractional part → `3` (it does not round!). But `Convert.ToInt32(v)` uses banker's rounding (round half to even): `3.5` rounds to the nearest even → `4`. This is the key difference between a cast and `Convert`: a cast always truncates, `Convert` uses banker's rounding.
- A неверно / wrong: `Convert.ToInt32(3.5)` даёт `4`, а не `3`.
- B неверно / wrong: cast не округляет, он отбрасывает дробную часть → `3`.
- D неверно / wrong: стороны перепутаны.

**Сложность / Difficulty:** 3/5

**Тип / Type:** Understanding

---

## Вопрос 12 / Question 12 (Debug / Найди баг)

**Вопрос / Question:**
В коде ошибка компиляции. Найди причину и исправление. / The code has a compile error. Find the cause and the fix.

```csharp
public class Config
{
    public const DateTime CreatedAt = DateTime.UtcNow;
}
```

**Варианты / Options:**
- A) Бага нет — компилируется / No bug — compiles
- B) `const` работает только со значениями, известными на этапе компиляции (примитивы, строки, `null`); `DateTime.UtcNow` вычисляется в рантайме — нужно `readonly` с инициализацией в конструкторе
- C) Нужно добавить `static` / Add `static`
- D) `DateTime` не может быть полем класса / `DateTime` cannot be a class field

**Правильный ответ / Correct:** B

**Объяснение / Explanation:**
`const` требует, чтобы значение было известно **при компиляции** — оно «вшивается» в сборку. `DateTime.UtcNow` — рантайм-значение, поэтому компилятор отказывается его принять. Правильный путь — `public readonly DateTime CreatedAt;` с присвоением в конструкторе: `CreatedAt = DateTime.UtcNow;`. `readonly` фиксирует значение после конструктора, но позволяет задать его в рантайме.
`const` requires a value known **at compile time** — it is baked into the assembly. `DateTime.UtcNow` is a runtime value, so the compiler rejects it. The correct fix is `public readonly DateTime CreatedAt;` assigned in a constructor: `CreatedAt = DateTime.UtcNow;`. `readonly` freezes the value after construction but allows it to be set at runtime.
- A неверно / wrong: код не компилируется (CS0133 — константа не вычислима).
- C неверно / wrong: `static` не решает проблему — `DateTime.UtcNow` всё ещё рантайм.
- D неверно / wrong: `DateTime` (value type) прекрасно может быть полем, проблема в `const`, а не в типе.

**Сложность / Difficulty:** 3/5

**Тип / Type:** Debug

---

## Вопрос 13 / Question 13 (Best Practices / Как лучше)

**Вопрос / Question:**
Ты пишешь корзину интернет-магазина: суммируешь цены, считаешь налог и скидки. Какой тип выбрать для денежных значений? / You are writing a shopping cart: you sum prices and compute taxes and discounts. Which type should you use for money?

**Варианты / Options:**
- A) `double` — быстрый и удобный / fast and convenient
- B) `float` — занимает меньше памяти / uses less memory
- C) `decimal` — точное base-10 представление, без потери копеек при сложении/округлении
- D) `int` в копейках/центах — вручную масштабируешь / `int` in cents — manual scaling

**Правильный ответ / Correct:** C

**Объяснение / Explanation:**
Главное правило урока: **деньги считают в `decimal`**. `decimal` — 128-битный base-10 тип с 28–29 значащими цифрами: `0.1m + 0.2m == 0.3m` точно, налоги и скидки не «дрейфуют» на тысячах операций. `double` быстрее, но двоичный — копейки будут теряться. Для финансовых расчётов, где переполнение — ошибка, дополнительно включают `checked`.
The lesson's key rule: **do money math in `decimal`**. `decimal` is a 128-bit base-10 type with 28–29 significant digits: `0.1m + 0.2m == 0.3m` exactly, and taxes/discounts do not drift across thousands of operations. `double` is faster but binary — cents will leak. For finance where overflow is a bug, also enable `checked`.
- A неверно / wrong: `double` накапливает двоичную погрешность — цены «уплывут».
- B неверно / wrong: `float` ещё менее точен (≈7 цифр) — совсем не для денег.
- D неверно / wrong: `int`-в-копейках рабочий, но требует ручного масштабирования и не даёт преимуществ `decimal` перед округлением налогов/процентов; по уроку предпочтение — `decimal`.

**Сложность / Difficulty:** 2/5

**Тип / Type:** Best Practice

---

## Вопрос 14 / Question 14 (Matching / Сопоставь понятия)

**Вопрос / Question:**
Сопоставь оператор/концепцию и его описание. Какая расстановка верна? / Match each operator/concept to its description. Which pairing is correct?

| № | Оператор / Operator |  | Описание / Description |
|---|---|---|---|
| 1 | `??`  |  | A) Short-circuit AND: пропускает правый операнд, если левый `false` |
| 2 | `?.`  |  | B) Возвращает левый, если не null, иначе правый |
| 3 | `??=` |  | C) Вызывает член только если не null, иначе возвращает null |
| 4 | `&&`  |  | D) Присваивает правое только если левое равно null |

**Варианты / Options:**
- A) 1-B, 2-C, 3-D, 4-A
- B) 1-C, 2-B, 3-D, 4-A
- C) 1-B, 2-D, 3-C, 4-A
- D) 1-D, 2-C, 3-B, 4-A

**Правильный ответ / Correct:** A

**Объяснение / Explanation:**
- `??` (null-coalescing): `a ?? b` возвращает `a`, если оно не null, иначе `b` → **1-B**.
- `?.` (null-conditional): `obj?.Method()` вызывает метод только при не-null `obj`, иначе null → **2-C**.
- `??=` (null-coalescing assignment): `x ??= y` присваивает `y` только если `x` равно null → **3-D**.
- `&&` (short-circuit AND): если левый операнд `false`, правый не вычисляется → **4-A**.

- `??` (null-coalescing): `a ?? b` returns `a` if not null, otherwise `b` → **1-B**.
- `?.` (null-conditional): `obj?.Method()` calls the method only when `obj` is non-null, otherwise yields null → **2-C**.
- `??=` (null-coalescing assignment): `x ??= y` assigns `y` only if `x` is null → **3-D**.
- `&&` (short-circuit AND): if the left operand is `false`, the right is not evaluated → **4-A**.

- B неверно / wrong: перепутаны `??` и `?.`.
- C неверно / wrong: перепутаны `?.` и `??=`.
- D неверно / wrong: перепутаны `??` и `??=`.

**Сложность / Difficulty:** 2/5

**Тип / Type:** Matching

---

## Ключ ответов / Answer key

| № | Ответ / Answer | Сложность / Difficulty | Тип / Type | Тема / Topic |
|---|---|---|---|---|
| 1  | B | 2/5 | Understanding    | Значимые vs ссылочные / Value vs reference (L01) |
| 2  | B | 3/5 | Understanding    | decimal vs double (L02) |
| 3  | C | 2/5 | Understanding    | var, суффиксы литералов / var, literal suffixes (L02, L03) |
| 4  | B | 1/5 | Code Completion  | TryParse безопасный ввод / TryParse safe input (L06) |
| 5  | B | 2/5 | Understanding    | Целочисленное деление, `%` / Integer division, `%` (L04) |
| 6  | B | 3/5 | Debug            | nullable `.Value` без `HasValue` (L05) |
| 7  | B | 3/5 | Understanding    | short-circuit `&&` (L04) |
| 8  | B | 2/5 | Understanding    | string immutability (L07) |
| 9  | B | 3/5 | Understanding    | `[Flags]` enum, побитовая проверка (L08) |
| 10 | A | 2/5 | Code Completion  | Деконструкция кортежа / Tuple deconstruction (L08) |
| 11 | C | 3/5 | Understanding    | cast vs `Convert.ToInt32` (округление) (L06) |
| 12 | B | 3/5 | Debug            | `const` vs `readonly` (L03) |
| 13 | C | 2/5 | Best Practice    | Выбор типа для денег / Money type choice (L02) |
| 14 | A | 2/5 | Matching         | Операторы `??` `?.` `??=` `&&` (L04, L05) |

---

## Покрытие тем / Topic coverage

| Урок / Lesson | Тема / Topic | Вопросы / Questions |
|---|---|---|
| L01 | Значимые/ссылочные, stack/heap, boxing | 1 |
| L02 | Примитивы int/double/decimal, суффиксы | 2, 3, 13 |
| L03 | var, const, readonly | 3, 12 |
| L04 | Операторы: арифметика, логика, побитовые, short-circuit | 5, 7, 14 |
| L05 | null, nullable, `??` `?.` `??=`, NRT | 6, 14 |
| L06 | Преобразования: cast/Convert/Parse/TryParse, checked | 4, 11 |
| L07 | string, интерполяция, StringBuilder, immutable | 8 |
| L08 | enum `[Flags]`, кортежи, деконструкция | 9, 10 |

## Разнообразие типов / Question-type variety

| Тип / Type | Кол-во / Count | Вопросы / Questions |
|---|---|---|
| Multiple Choice (Understanding) | 8 | 1, 2, 3, 5, 7, 8, 9, 11 |
| Code Completion | 2 | 4, 10 |
| Debug | 2 | 6, 12 |
| Best Practice | 1 | 13 |
| Matching | 1 | 14 |
| **Итого / Total** | **14** | |
