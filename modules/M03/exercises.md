# Задания модуля M03 / Exercises: M03
# Управление потоком, методы / Control Flow & Methods

## Задания по урокам / Per-lesson exercises

---

### Задание M03-L01
**Задача / Task:**
Классификация числа. Пользователь вводит целое число. Программа определяет, является ли оно положительным, отрицательным или нулём, чётным или нечётным, и выводит сводку одним предложением. Использовать `if/else` и тернарный оператор.
*Classify a number entered by the user as positive/negative/zero and even/odd, printing one summary sentence. Use `if/else` and the ternary operator.*

**Требования / Requirements:**
- Ввод через `Console.ReadLine()` с проверкой через `int.TryParse`.
- Одна ветка `if / else if / else` для знака числа (положительное / отрицательное / ноль).
- Чётность определяется тернарным выражением, присваиваемым в `string parity`.
- Ноль считается чётным.
- Вывод имеет вид: `Число 5 — положительное, чётное.` (для нуля: `Число 0 — ноль, чётное.`).
- Input via `Console.ReadLine()` validated with `int.TryParse`.
- One `if / else if / else` chain for the sign.
- Parity computed by a ternary expression assigned to `string parity`.
- Zero is treated as even.
- Output format: `Number 5 — positive, even.` (for zero: `Number 0 — zero, even.`).

**Критерии приёмки / Acceptance criteria:**
- [ ] Программа не падает при нечисловом вводе, а просит повторить.
- [ ] Использован как `if/else`, так и тернарный оператор.
- [ ] Для нуля выводится корректная классификация.
- [ ] Negative input handled correctly.
- [ ] Program does not crash on non-numeric input.

**Время / Time:** 30 мин

**Пример решения / Sample solution (для менторов):**
```csharp
using System;

Console.Write("Введите целое число / Enter an integer: ");
while (!int.TryParse(Console.ReadLine(), out int n))
{
    Console.Write("Не число, попробуйте снова / Not a number, try again: ");
}

string sign = n switch
{
    > 0 => "положительное / positive",
    < 0 => "отрицательное / negative",
    _   => "ноль / zero"
};

string parity = (n % 2 == 0) ? "чётное / even" : "нечётное / odd";

Console.WriteLine($"Число {n} — {sign}, {parity}.");
Console.WriteLine($"Number {n} — {sign}, {parity}.");
```

---

### Задание M03-L02
**Задача / Task:**
Статус заказа. Дан перечислимый тип `OrderStatus { Pending, Paid, Shipped, Delivered, Cancelled }`. По статусу вернуть текстовое описание с помощью `switch` expression и pattern matching. Для `Paid` и `Shipped` учесть дополнительное поле `DaysInTransit` через property pattern.
*Given an `OrderStatus` enum, return a human-readable description via a `switch` expression with pattern matching. For `Paid`/`Shipped` also inspect `DaysInTransit` via a property pattern.*

**Требования / Requirements:**
- Тип `enum OrderStatus { Pending, Paid, Shipped, Delivered, Cancelled }`.
- Класс `Order` со свойствами `Status` и `int DaysInTransit`.
- Метод `string Describe(Order o)` использует `switch` expression с `when`-клаузами или property patterns.
- `Shipped` с `DaysInTransit > 5` → "в пути более 5 дней".
- `Shipped` иначе → "в пути".
- Отдельный `switch` для `Cancelled` с извлечением причины через tuple pattern при наличии `string? CancelReason`.
- Type `enum OrderStatus { Pending, Paid, Shipped, Delivered, Cancelled }`.
- Class `Order` with `Status` and `int DaysInTransit` properties.
- Method `string Describe(Order o)` uses a `switch` expression.
- `Shipped` with `DaysInTransit > 5` → "in transit over 5 days".
- Tuple pattern for `Cancelled` with optional `CancelReason`.

**Критерии приёмки / Acceptance criteria:**
- [ ] Использован `switch` expression (не `switch` statement).
- [ ] Применён property pattern или `when`-клауза.
- [ ] Покрыты все 5 значений enum (нет default-«заглушки» без смысла).
- [ ] `Cancelled` с причиной выводит причину, без причины — общее сообщение.
- [ ] Exhaustive switch (compiler can verify exhaustiveness).

**Время / Time:** 45 мин

**Пример решения / Sample solution (для менторов):**
```csharp
using System;

public enum OrderStatus { Pending, Paid, Shipped, Delivered, Cancelled }

public sealed class Order
{
    public OrderStatus Status { get; init; }
    public int DaysInTransit { get; init; }
    public string? CancelReason { get; init; }
}

public static class OrderPrinter
{
    public static string Describe(Order o) => o switch
    {
        { Status: OrderStatus.Pending }                                  => "Заказ ожидает оплаты / Awaiting payment",
        { Status: OrderStatus.Paid }                                     => "Заказ оплачен, готовится / Paid, being prepared",
        { Status: OrderStatus.Shipped, DaysInTransit: > 5 }              => "В пути более 5 дней / In transit over 5 days",
        { Status: OrderStatus.Shipped }                                  => "В пути / In transit",
        { Status: OrderStatus.Delivered }                                => "Доставлен / Delivered",
        { Status: OrderStatus.Cancelled, CancelReason: string r }        => $"Отменён: {r} / Cancelled: {r}",
        { Status: OrderStatus.Cancelled }                                => "Отменён / Cancelled",
        _                                                                => throw new InvalidOperationException("Неизвестный статус / Unknown status")
    };
}

// Демонстрация / Demo
var orders = new[]
{
    new Order { Status = OrderStatus.Pending },
    new Order { Status = OrderStatus.Shipped, DaysInTransit = 7 },
    new Order { Status = OrderStatus.Shipped, DaysInTransit = 2 },
    new Order { Status = OrderStatus.Cancelled, CancelReason = "Не пришёл / No show" }
};

foreach (var o in orders)
    Console.WriteLine(OrderPrinter.Describe(o));
```

---

### Задание M03-L03
**Задача / Task:**
Таблица умножения. Вывести таблицу умножения 1..10 × 1..10 с выравниванием колонок по 4 символам, используя вложенные циклы `for`. Затем — второй блок: пройтись по массиву названий и вывести каждое `foreach`.
*Print a 10×10 multiplication table with 4-wide columns using nested `for` loops. Then iterate an array of names with `foreach` and print each.*

**Требования / Requirements:**
- Вложенный `for` по `i` и `j` от 1 до 10 включительно.
- Формат `{i*j,4}` для выравнивания.
- После таблицы — пустая строка.
- Массив `string[] names = { "Алиса", "Борис", "Вера", "Глеб" }`.
- Вывод каждого имени через `foreach` с индексом (использовать счётчик `int idx = 0; idx++`).
- Nested `for` over `i`, `j` in 1..10 inclusive.
- Format `{i*j,4}` for alignment.
- Blank line after the table.
- `foreach` over `names` with an external index counter.

**Критерии приёмки / Acceptance criteria:**
- [ ] Таблица ровно выровнена по колонкам.
- [ ] Использованы и `for`, и `foreach` (по назначению).
- [ ] Нет «магических чисел» — `const int Size = 10`.
- [ ] Вывод компактен (одна строка на ряд таблицы).
- [ ] Columns are aligned.
- [ ] Both `for` and `foreach` used appropriately.

**Время / Time:** 40 мин

**Пример решения / Sample solution (для менторов):**
```csharp
using System;

const int Size = 10;

// Таблица умножения / Multiplication table
for (int i = 1; i <= Size; i++)
{
    for (int j = 1; j <= Size; j++)
    {
        Console.Write($"{i * j,4}");
    }
    Console.WriteLine();
}

Console.WriteLine();

// foreach по массиву имён / foreach over names
string[] names = { "Алиса / Alice", "Борис / Boris", "Вера / Vera", "Глеб / Gleb" };
int idx = 0;
foreach (var name in names)
{
    Console.WriteLine($"[{idx}] {name}");
    idx++;
}
```

---

### Задание M03-L04
**Задача / Task:**
Поиск в массиве. Дан `int[] data`. Найти первый индекс элемента, кратного 7 и большего 20, используя `for` с `break`. Затем посчитать количество чётных чисел, пропуская отрицательные через `continue`. Реализовать метод поиска с ранним `return`.
*Given `int[] data`, find the first index of an element divisible by 7 and > 20 using `for` + `break`. Then count even numbers, skipping negatives with `continue`. Provide an early-return search method.*

**Требования / Requirements:**
- Метод `int IndexOfMultipleOfSeven(int[] arr)` возвращает индекс или `-1`, используя ранний `return`.
- В `Main` — найти первый такой элемент и выйти из цикла через `break`.
- Отдельный цикл считает чётные (`% 2 == 0`), пропуская отрицательные через `continue`.
- Вывод: индекс найденного элемента и количество чётных.
- `IndexOfMultipleOfSeven` returns index or `-1` via early `return`.
- `break` exits the main loop on first match.
- Separate loop counts evens, skipping negatives via `continue`.

**Критерии приёмки / Acceptance criteria:**
- [ ] Использованы `break`, `continue`, `return` — каждый по назначению.
- [ ] Метод не падает на пустом массиве (возвращает `-1`).
- [ ] Отрицательные числа не учитываются в счётчике чётных.
- [ ] `break`, `continue`, `return` each used purposefully.
- [ ] Empty array handled gracefully.

**Время / Time:** 45 мин

**Пример решения / Sample solution (для менторов):**
```csharp
using System;

int[] data = { 3, -4, 21, 14, 28, 35, -10, 42, 7, 63 };

// 1. Ранний return внутри метода / early return in a method
int found = IndexOfMultipleOfSeven(data);
Console.WriteLine($"Индекс / Index: {found}");

// 2. break — выходим при первом вхождении в Main / break on first match in Main
for (int i = 0; i < data.Length; i++)
{
    if (data[i] > 20 && data[i] % 7 == 0)
    {
        Console.WriteLine($"Первый в Main / First in Main: data[{i}] = {data[i]}");
        break;
    }
}

// 3. continue — пропускаем отрицательные / skip negatives
int evenCount = 0;
foreach (var x in data)
{
    if (x < 0) continue;
    if (x % 2 == 0) evenCount++;
}
Console.WriteLine($"Чётных положительных / Positive evens: {evenCount}");

static int IndexOfMultipleOfSeven(int[] arr)
{
    if (arr is null || arr.Length == 0) return -1;
    for (int i = 0; i < arr.Length; i++)
    {
        if (arr[i] % 7 == 0) return i;
    }
    return -1;
}
```

---

### Задание M03-L05
**Задача / Task:**
Декомпозиция калькулятора. Реализовать калькулятор двух чисел через набор небольших методов: `double Add`, `Sub`, `Mul`, `Div`, и `void PrintResult`. Метод `Div` бросает `DivideByZeroException` при делителе 0. В `Main` — выбор операции по символу и вывод.
*Decompose a two-number calculator into small methods: `Add`, `Sub`, `Mul`, `Div` returning `double`, and `void PrintResult`. `Div` throws on zero divisor. `Main` picks the op by symbol and prints.*

**Требования / Requirements:**
- Пять методов: `Add`, `Sub`, `Mul`, `Div`, `PrintResult(double, char)`.
- `Div` проверяет `b == 0` и кидает `DivideByZeroException`.
- `PrintResult` форматирует: `5 + 3 = 8`.
- `Main` читает `a`, `b`, символ `op` (`+ - * /`), вызывает нужный метод.
- Деление на ноль ловится в `try/catch` с дружелюбным сообщением.
- Five methods: `Add`, `Sub`, `Mul`, `Div`, `PrintResult`.
- `Div` throws `DivideByZeroException` when `b == 0`.
- `PrintResult` formats `5 + 3 = 8`.
- `try/catch` around division with a friendly message.

**Критерии приёмки / Acceptance criteria:**
- [ ] Каждый метод решает одну задачу (SRP).
- [ ] Деление на ноль обрабатывается, программа не падает.
- [ ] Использованы `static` local functions или обычные методы.
- [ ] Неизвестный символ выводит сообщение об ошибке.
- [ ] Single-responsibility per method.
- [ ] Zero division handled without crashing.

**Время / Time:** 50 мин

**Пример решения / Sample solution (для менторов):**
```csharp
using System;

Console.Write("a = ");
double a = double.Parse(Console.ReadLine()!);
Console.Write("b = ");
double b = double.Parse(Console.ReadLine()!);
Console.Write("Операция / Op (+ - * /): ");
char op = Console.ReadLine()![0];

try
{
    double result = op switch
    {
        '+' => Add(a, b),
        '-' => Sub(a, b),
        '*' => Mul(a, b),
        '/' => Div(a, b),
        _   => throw new InvalidOperationException($"Неизвестная операция / Unknown op: {op}")
    };
    PrintResult(a, b, op, result);
}
catch (DivideByZeroException)
{
    Console.WriteLine("Ошибка: деление на ноль / Error: division by zero");
}
catch (InvalidOperationException ex)
{
    Console.WriteLine(ex.Message);
}

static double Add(double x, double y) => x + y;
static double Sub(double x, double y) => x - y;
static double Mul(double x, double y) => x * y;
static double Div(double x, double y)
{
    if (y == 0) throw new DivideByZeroException();
    return x / y;
}
static void PrintResult(double x, double y, char op, double r)
    => Console.WriteLine($"{x} {op} {y} = {r}");
```

---

### Задание M03-L06
**Задача / Task:**
Метод `TryDivide(out)`. Реализовать `bool TryDivide(double a, double b, out double result)`, возвращающий `false` при `b == 0` (без исключений). Дополнительно: `void Increment(ref int x)`, `void PrintAll(params string[] items)` и метод с `in`-параметром `double Dot(in double ax, in double ay, in double bx, in double by)`.
*Implement `bool TryDivide(double a, double b, out double result)` returning `false` on zero divisor (no exceptions). Also `void Increment(ref int x)`, `void PrintAll(params string[] items)`, and `double Dot(in ...)`.*

**Требования / Requirements:**
- `TryDivide` помечает `result` как `out`, присваивает `0` перед возвратом `false`.
- `Increment(ref int x)` увеличивает `x` на 1 (изменение видно вызывающему).
- `PrintAll(params string[] items)` выводит все элементы в одну строку через `, `.
- `Dot` принимает все параметры как `in` (не копирует, защищает от изменений).
- В `Main` продемонстрировать все четыре метода.
- `TryDivide` assigns `result = 0` before returning `false`.
- `Increment(ref int x)` mutates the caller's variable.
- `PrintAll(params string[])` joins with `, `.
- `Dot` takes all `in` parameters.

**Критерии приёмки / Acceptance criteria:**
- [ ] `out`-параметр инициализирован во всех путях возврата.
- [ ] `ref` действительно изменяет переменную вызывающего.
- [ ] `params` позволяет вызвать с произвольным числом аргументов.
- [ ] `in` применён корректно (попытка изменить вызывает ошибку компиляции — продемонстрировать в комментарии).
- [ ] All return paths initialize `out`.
- [ ] `ref` mutates caller's variable.

**Время / Time:** 60 мин

**Пример решения / Sample solution (для менторов):**
```csharp
using System;

// 1. out
if (TryDivide(10, 2, out double r))
    Console.WriteLine($"10 / 2 = {r}");
if (!TryDivide(10, 0, out _))
    Console.WriteLine("10 / 0 — нельзя / cannot divide");

// 2. ref
int counter = 5;
Increment(ref counter);
Console.WriteLine($"counter = {counter}"); // 6

// 3. params
PrintAll("один / one", "два / two", "три / three");

// 4. in
double d = Dot(in 1.0, in 0.0, in 0.0, in 1.0);
Console.WriteLine($"dot = {d}");

static bool TryDivide(double a, double b, out double result)
{
    if (b == 0)
    {
        result = 0;
        return false;
    }
    result = a / b;
    return true;
}

static void Increment(ref int x) => x++;

static void PrintAll(params string[] items)
    => Console.WriteLine(string.Join(", ", items));

static double Dot(in double ax, in double ay, in double bx, in double by)
{
    // ax = 0; // Ошибка компиляции: in-параметр неизменяем / compile error: in is readonly
    return ax * bx + ay * by;
}
```

---

### Задание M03-L07
**Задача / Task:**
Перегрузка и local functions — форматирование. Реализовать три перегруженных метода `string Format(double x)`, `string Format(double x, int digits)`, `string Format(double x, string prefix)`. Внутри третьего использовать local function `string Pad(string s)`, дополняющую строку пробелами до 10 символов.
*Overloading + local functions — formatting. Provide three overloads of `Format`. The third uses a local function `Pad` padding strings to 10 chars.*

**Требования / Requirements:**
- Три перегрузки `Format`, отличающиеся сигнатурой.
- `Format(double)` → 2 знака после запятой.
- `Format(double, int digits)` → `digits` знаков.
- `Format(double, string prefix)` → `prefix + значение` с padded prefix через local function.
- Local function `Pad` определена внутри `Format(double, string prefix)`.
- Три перегрузки с разными сигнатурами.
- `Format(double)` → 2 decimals.
- `Format(double, int digits)` → `digits` decimals.
- `Format(double, string prefix)` → `prefix + value` with padded prefix.
- Local function `Pad` defined inside the third overload.

**Критерии приёмки / Acceptance criteria:**
- [ ] Все три перегрузки вызываются по имени `Format` с разным набором аргументов.
- [ ] Local function определена внутри метода и не видна снаружи.
- [ ] Вывод корректно использует `F` / `F{n}` форматирование.
- [ ] All three overloads callable as `Format`.
- [ ] Local function scoped to its containing method.

**Время / Time:** 50 мин

**Пример решения / Sample solution (для менторов):**
```csharp
using System;

double v = 3.14159265;
Console.WriteLine(Format(v));                 // 3.14
Console.WriteLine(Format(v, 4));              // 3.1416
Console.WriteLine(Format(v, "Pi"));           // Pi        = 3.14

static string Format(double x) => x.ToString("F2");

static string Format(double x, int digits) => x.ToString($"F{digits}");

static string Format(double x, string prefix)
{
    string Pad(string s) => s.PadRight(10);
    return $"{Pad(prefix)}= {x:F2}";
}
```

---

### Задание M03-L08
**Задача / Task:**
Рекурсия — факториал и Фибоначчи. Реализовать `long Factorial(int n)` и `long Fib(int n)` рекурсивно. Защитить от отрицательных аргументов через `ArgumentOutOfRangeException`. Добавить мемоизацию для `Fib` через `Dictionary<int, long>`.
*Recursion — factorial & Fibonacci. Implement `long Factorial(int n)` and `long Fib(int n)` recursively. Guard against negative args. Memoize `Fib` with a `Dictionary`.*

**Требования / Requirements:**
- `Factorial(0) == 1`, `Factorial(n) == n * Factorial(n - 1)`.
- `Fib(0) == 0`, `Fib(1) == 1`, `Fib(n) == Fib(n-1) + Fib(n-2)`.
- Отрицательные `n` кидают `ArgumentOutOfRangeException`.
- Мемоизация через `static Dictionary<int, long> FibCache`.
- В `Main` вывести `Factorial(10)` и `Fib(20)`.
- `Factorial(0) == 1`, recursive definition.
- `Fib(0) == 0`, `Fib(1) == 1`, recursive definition.
- Negative `n` throws `ArgumentOutOfRangeException`.
- Memoization via `static Dictionary<int, long>`.

**Критерии приёмки / Acceptance criteria:**
- [ ] Рекурсия есть и корректно завершается (базовый случай).
- [ ] Отрицательный вход вызывает исключение, а не зацикливание.
- [ ] Мемоизация ускоряет `Fib` для `n = 30` (время ощутимо меньше без неё).
- [ ] Без tail-рекурсии — но глубина не превышает разумного для `n <= 30`.
- [ ] Base case present; recursion terminates.
- [ ] Negative input throws, doesn't loop.

**Время / Time:** 60 мин

**Пример решения / Sample solution (для менторов):**
```csharp
using System;
using System.Collections.Generic;

Console.WriteLine($"10! = {Factorial(10)}");
Console.WriteLine($"Fib(20) = {Fib(20)}");
Console.WriteLine($"Fib(30) = {Fib(30)}");

static long Factorial(int n)
{
    if (n < 0) throw new ArgumentOutOfRangeException(nameof(n), "Факториал не определён для отрицательных / undefined for negatives");
    return n <= 1 ? 1 : n * Factorial(n - 1);
}

static readonly Dictionary<int, long> FibCache = new() { [0] = 0, [1] = 1 };

static long Fib(int n)
{
    if (n < 0) throw new ArgumentOutOfRangeException(nameof(n), "Фибоначчи не определён для отрицательных / undefined for negatives");
    if (FibCache.TryGetValue(n, out long v)) return v;
    long f = Fib(n - 1) + Fib(n - 2);
    FibCache[n] = f;
    return f;
}
```

---

## Мини-проект модуля / Module mini-project

### Базовая версия / Base version
**Задача / Task:**
Мини-калькулятор с меню в консоли, объединяющий поток управления и методы. Пользователь выбирает операцию из меню, вводит два числа, получает результат; цикл повторяется до выбора «Выход».
*A console mini-calculator with a menu, combining control flow and methods. User picks an operation from a menu, enters two numbers, gets the result; loops until "Exit".*

**Требования / Requirements:**
- Меню: `1) +  2) -  3) *  4) /  0) Выход / Exit`.
- Чтение выбора через `Console.ReadLine()` + `switch` (statement или expression).
- Чтение чисел — метод `double ReadNumber(string prompt)`.
- Методы операций: `Add`, `Sub`, `Mul`, `Div` (деление на 0 → сообщение).
- Основной цикл `while (true)` с `break` по выбору 0.
- Чистый, хорошо именованный код; каждый метод — одна ответственность.
- Menu: `1) +  2) -  3) *  4) /  0) Exit`.
- Read choice via `Console.ReadLine()` + `switch`.
- Numbers via `ReadNumber(string prompt)`.
- Operation methods `Add`/`Sub`/`Mul`/`Div` (division by 0 → message).
- `while (true)` loop with `break` on 0.

**Критерии приёмки / Acceptance criteria:**
- [ ] Меню отображается перед каждым действием.
- [ ] Неизвестный выбор не ломает программу — выводится подсказка.
- [ ] Деление на ноль обрабатывается без исключения на верхнем уровне.
- [ ] Программа завершается только при выборе 0.
- [ ] Все арифметические операции в отдельных методах.
- [ ] Unknown choice shows a hint, doesn't crash.
- [ ] Zero division handled.
- [ ] Exit only on 0.

**Время / Time:** 90 мин

**Пример решения / Sample solution:**
```csharp
using System;

while (true)
{
    Console.WriteLine();
    Console.WriteLine("=== Мини-калькулятор / Mini-calculator ===");
    Console.WriteLine("1) +   2) -   3) *   4) /   0) Выход / Exit");
    Console.Write("Выбор / Choice: ");
    var choice = Console.ReadLine();

    if (choice == "0") break;

    double a = ReadNumber("a = ");
    double b = ReadNumber("b = ");

    double result;
    switch (choice)
    {
        case "1": result = Add(a, b); break;
        case "2": result = Sub(a, b); break;
        case "3": result = Mul(a, b); break;
        case "4":
            if (b == 0) { Console.WriteLine("Деление на ноль / Division by zero"); continue; }
            result = Div(a, b); break;
        default:
            Console.WriteLine("Неизвестный выбор / Unknown choice");
            continue;
    }
    Console.WriteLine($"Результат / Result = {result:F4}");
}

static double ReadNumber(string prompt)
{
    while (true)
    {
        Console.Write(prompt);
        if (double.TryParse(Console.ReadLine(), out double v)) return v;
        Console.WriteLine("Не число / Not a number");
    }
}

static double Add(double x, double y) => x + y;
static double Sub(double x, double y) => x - y;
static double Mul(double x, double y) => x * y;
static double Div(double x, double y) => x / y;
```

### Pro-версия / Pro version
**Задача / Task:**
Расширить базовую версию: история операций, обработка ошибок ввода через `TryParse`, рекурсивное вычисление (поддержка факториала одной операндой), `switch`-expression для разбора команд.
*Extend the base version: operation history, input validation via `TryParse`, recursive computation (single-operand factorial), `switch` expression for command parsing.*

**Доп. требования / Дополнительные / Additional requirements:**
- Меню расширено: `5) ! (факториал, один операнд) / factorial`.
- История хранится в `List<string>` и выводится по команде `h` (или пункту `6) История / History`).
- Все строки ввода проходят `TryParse`-валидацию; ошибочный ввод — повтор.
- `switch` expression используется для разбора выбора команды.
- `Factorial` реализован рекурсивно (из L08), вызывается для одного числа.
- Все ошибки (деление на 0, отрицательный факториал) обрабатываются внутри `switch` через `when`-клаузы или try/catch.
- Вывод истории в виде таблицы с нумерацией.
- Menu adds `5) ! (factorial, one operand)` and `6) History`.
- History in `List<string>`, printed on `6`.
- All inputs validated with `TryParse`.
- `switch` expression for command parsing.
- `Factorial` recursive (from L08).
- Errors handled inside `switch` with `when` clauses or try/catch.
- History printed as a numbered table.

**Критерии приёмки / Acceptance criteria:**
- [ ] История аккумулирует все успешные операции с временнóй меткой.
- [ ] `TryParse` используется для всех числовых вводов; нечисловой ввод не падает.
- [ ] Факториал вычисляется рекурсивно; отрицательный аргумент выводит ошибку.
- [ ] `switch` expression используется для выбора команды (не statement).
- [ ] Деление на ноль не роняет приложение и не попадает в историю.
- [ ] История выводится в виде нумерованного списка.
- [ ] History accumulates successful ops with timestamps.
- [ ] All numeric inputs use `TryParse`.
- [ ] Factorial recursive; negatives handled.
- [ ] `switch` expression used for command parsing.

**Время:** 150 мин

**Пример решения / Sample solution:**
```csharp
using System;
using System.Collections.Generic;

var history = new List<string>();

while (true)
{
    Console.WriteLine();
    Console.WriteLine("=== Мини-калькулятор PRO ===");
    Console.WriteLine("1) +   2) -   3) *   4) /   5) ! (факториал / factorial)");
    Console.WriteLine("6) История / History   0) Выход / Exit");
    Console.Write("Выбор / Choice: ");
    var choice = Console.ReadLine()?.Trim();

    if (choice is null or "0") break;

    switch (choice)
    {
        case "6":
            PrintHistory();
            continue;

        case "5":
        {
            int n = ReadInt("n = ");
            try
            {
                long f = Factorial(n);
                var entry = $"{DateTime.Now:HH:mm:ss}  {n}! = {f}";
                history.Add(entry);
                Console.WriteLine(entry);
            }
            catch (ArgumentOutOfRangeException ex)
            {
                Console.WriteLine(ex.Message);
            }
            continue;
        }

        case "1" or "2" or "3" or "4":
        {
            double a = ReadDouble("a = ");
            double b = ReadDouble("b = ");
            string? error = null;
            double result = 0;

            try
            {
                result = choice switch
                {
                    "1" => Add(a, b),
                    "2" => Sub(a, b),
                    "3" => Mul(a, b),
                    "4" => b == 0
                        ? throw new DivideByZeroException()
                        : Div(a, b),
                    _  => 0
                };
            }
            catch (DivideByZeroException)
            {
                error = "Деление на ноль / Division by zero";
            }

            if (error is not null)
            {
                Console.WriteLine(error);
                continue;
            }

            char op = choice[0] switch { '1' => '+', '2' => '-', '3' => '*', '4' => '/', _ => '?' };
            var entry = $"{DateTime.Now:HH:mm:ss}  {a} {op} {b} = {result:F4}";
            history.Add(entry);
            Console.WriteLine(entry);
            continue;
        }

        default:
            Console.WriteLine("Неизвестная команда / Unknown command");
            continue;
    }
}

void PrintHistory()
{
    if (history.Count == 0)
    {
        Console.WriteLine("История пуста / History is empty");
        return;
    }
    for (int i = 0; i < history.Count; i++)
        Console.WriteLine($"#{i + 1,3}  {history[i]}");
}

static int ReadInt(string prompt)
{
    while (true)
    {
        Console.Write(prompt);
        if (int.TryParse(Console.ReadLine(), out int v)) return v;
        Console.WriteLine("Не число / Not a number");
    }
}

static double ReadDouble(string prompt)
{
    while (true)
    {
        Console.Write(prompt);
        if (double.TryParse(Console.ReadLine(), out double v)) return v;
        Console.WriteLine("Не число / Not a number");
    }
}

static long Factorial(int n) =>
    n < 0
        ? throw new ArgumentOutOfRangeException(nameof(n), "Факториал отрицательного не определён / undefined for negatives")
        : n <= 1 ? 1 : n * Factorial(n - 1);

static double Add(double x, double y) => x + y;
static double Sub(double x, double y) => x - y;
static double Mul(double x, double y) => x * y;
static double Div(double x, double y) => x / y;
```
