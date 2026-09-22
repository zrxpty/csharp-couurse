# Задания модуля M02 / Exercises: M02
# Типы, переменные, операторы / Types, Variables, Operators

## Задания по урокам / Per-lesson exercises

---

### Задание M02-L01 — Значимые и ссылочные типы / Value & Reference types
**Задача / Task:**
Демонстрация разницы между значимыми и ссылочными типами при копировании. Создайте значимый тип `struct Money` и ссылочный тип `class Account`. Покажите, что изменение копии значимого типа не затрагивает оригинал, а изменение копии ссылочного типа — затрагивает.
Demonstrate the difference between value and reference types on copy. Create a value type `struct Money` and a reference type `class Account`. Show that mutating a value-type copy does not affect the original, while mutating a reference-type copy does.

**Требования / Requirements:**
- Объявить `public struct Money` с полями `decimal Amount` и `string Currency`.
- Объявить `public class Account` с полями `decimal Balance` и `string Owner`.
- В `Main` создать оригинал и копию каждого типа, изменить копию, вывести оба.
- Использовать `record struct` не нужно — достаточно обычного `struct`.
- Вывод должен наглядно показывать, где оригинал изменился, а где нет.

**Критерии приёмки / Acceptance criteria:**
- [ ] Копия `Money` не меняет оригинал; копия `Account` меняет оригинал.
- [ ] Вывод содержит подписи обоих случаев и оба состояния.
- [ ] Нет `unsafe`, нет `ref`, только присваивание `=` для копирования.
- [ ] Код компилируется как C# 12 / .NET 8 без предупреждений уровня error.

**Время / Time:** 30–45 мин

**Пример решения / Sample solution (для менторов):**
```csharp
// C# 12 / .NET 8
namespace M02.L01;

public struct Money
{
    public decimal Amount;
    public string Currency;
}

public class Account
{
    public decimal Balance;
    public string Owner = string.Empty;
}

internal static class Program
{
    private static void Main()
    {
        // Значимый тип: копия независима / Value type: copy is independent
        var wallet = new Money { Amount = 100m, Currency = "RUB" };
        var walletCopy = wallet;          // копирование по значению
        walletCopy.Amount = 999m;

        Console.WriteLine("Value type (struct Money):");
        Console.WriteLine($"  original : {wallet.Amount} {wallet.Currency}");
        Console.WriteLine($"  copy     : {walletCopy.Amount} {walletCopy.Currency}");

        // Ссылочный тип: копия разделяет данные / Reference type: copy shares data
        var acc = new Account { Balance = 100m, Owner = "Alice" };
        var accCopy = acc;               // копирование по ссылке
        accCopy.Balance = 999m;

        Console.WriteLine("Reference type (class Account):");
        Console.WriteLine($"  original : {acc.Balance} {acc.Owner}");
        Console.WriteLine($"  copy     : {accCopy.Balance} {accCopy.Owner}");
    }
}
```

---

### Задание M02-L02 — Примитивы и decimal для финансов / Primitives & decimal for finance
**Задача / Task:**
Реализуйте простой финансовый расчёт: сложить три позиции счёта, вычесть НДС 20% и округлить до копеек. Используйте `decimal`, не `double`. Сравните результат `decimal` и `double` для одного и того же набора сумм и объясните расхождение.
Implement a simple financial calculation: sum three invoice lines, subtract 20% VAT, round to cents. Use `decimal`, not `double`. Compare `decimal` and `double` results for the same amounts and explain the discrepancy.

**Требования / Requirements:**
- Входные данные: `19.99m`, `0.10m`, `5.00m` (три позиции).
- Сумма → вычесть 20% НДС → `decimal.Round(..., 2, MidpointRounding.ToEven)`.
- Повторить тот же расчёт с `double` и вывести разницу.
- В комментарии объяснить, почему `double` даёт погрешность.

**Критерии приёмки / Acceptance criteria:**
- [ ] Используется `decimal` для итогового расчёта.
- [ ] Округление до 2 знаков, банковское (`MidpointRounding.ToEven`).
- [ ] Выведены оба результата и их разница.
- [ ] Есть комментарий-объяснение расхождения `double` vs `decimal`.

**Время / Time:** 30–45 мин

**Пример решения / Sample solution (для менторов):**
```csharp
// C# 12 / .NET 8
namespace M02.L02;

internal static class Program
{
    private static void Main()
    {
        // decimal: точное представление денежных сумм, нет погрешности двоичных дробей.
        decimal d1 = 19.99m, d2 = 0.10m, d3 = 5.00m;
        decimal sumD = d1 + d2 + d3;
        decimal netD = sumD - sumD * 0.20m;
        decimal netDRounded = decimal.Round(netD, 2, MidpointRounding.ToEven);

        // double: двоичная мантисса не может точно представить 0.10 и 0.99 → накопление ошибки.
        double f1 = 19.99, f2 = 0.10, f3 = 5.00;
        double sumF = f1 + f2 + f3;
        double netF = sumF - sumF * 0.20;

        Console.WriteLine($"decimal net (rounded): {netDRounded}");
        Console.WriteLine($"double  net          : {netF:F17}");
        Console.WriteLine($"difference           : {(decimal)netF - netDRounded}");
    }
}
```

---

### Задание M02-L03 — var / const / readonly / var / const / readonly
**Задача / Task:**
Дан фрагмент кода с «плохими» объявлениями. Перепишите его, используя `var` для локальных выводимых типов, `const` для компиляционных констант и `readonly` для поля, устанавливаемого только в конструкторе. Уберите избыточные явные типы там, где `var` улучшает читаемость, но сохраните явные типы там, где тип неочевиден.
Given a code fragment with "poor" declarations. Rewrite it using `var` for inferred locals, `const` for compile-time constants, and `readonly` for a field set only in the constructor. Remove redundant explicit types where `var` improves readability, but keep explicit types where the type is non-obvious.

**Требования / Requirements:**
- Исходный код:
  ```csharp
  class Config {
      public double Pi = 3.14159;
      public string Name = "app";
      public Config(string n) { Name = n; }
  }
  class App {
      static void Main() {
          int x = 5;
          string s = "hello";
          List<int> nums = new List<int>();
          Dictionary<string,int> map = new Dictionary<string,int>();
      }
  }
  ```
- `Pi` должно стать `const`, `Name` — `readonly`.
- Локальные `x`, `s` — `var`; коллекции — `var` с target-typed `new()`.
- Сохранить явный тип там, где он улучшает ясность (например, возвращаемое значение метода-фабрики).

**Критерии приёмки / Acceptance criteria:**
- [ ] `Pi` объявлен как `public const double`.
- [ ] `Name` объявлен как `public readonly string`.
- [ ] Локальные используют `var` и target-typed `new()`.
- [ ] Код компилируется, поведение не изменилось.

**Время / Time:** 20–30 мин

**Пример решения / Sample solution (для менторов):**
```csharp
// C# 12 / .NET 8
using System.Collections.Generic;

namespace M02.L03;

public class Config
{
    public const double Pi = 3.14159;          // компиляционная константа
    public readonly string Name;               // задаётся только в конструкторе

    public Config(string name) => Name = name;
}

internal static class App
{
    private static void Main()
    {
        var x = 5;                             // int выводится из литерала
        var s = "hello";                        // string
        var nums = new List<int>();             // var + target-typed new
        var map = new Dictionary<string, int>();

        nums.Add(x);
        map[s] = x;

        // Явный тип сохранён: результат фабрики неочевиден читателю.
        IReadOnlyDictionary<string, int> readOnly = map;
        Console.WriteLine($"Pi={Config.Pi}, name={new Config("app").Name}, items={nums.Count}");
    }
}
```

---

### Задание M02-L04 — checked/unchecked и приоритет / checked/unchecked & precedence
**Задача / Task:**
Покажите переполнение `int` при умножении и продемонстрируйте разницу между `checked` и `unchecked` блоками. Затем вычислите выражение со смешанными операторами и явно покажите порядок вычисления с помощью скобок, чтобы результат был предсказуем.
Show `int` overflow on multiplication and demonstrate the difference between `checked` and `unchecked` blocks. Then compute an expression with mixed operators and explicitly show evaluation order using parentheses so the result is predictable.

**Требования / Requirements:**
- В `checked` блоке умножение, приводящее к переполнению, должно бросать `OverflowException`.
- В `unchecked` блоке то же умножение должно тихо «обернуться» (wrap-around).
- Выражение для приоритета: `10 + 2 * 3 - 4 / 2`; выведите его «как есть» и со скобками, дающими тот же результат, но явно.
- Перехватить `OverflowException` и вывести понятное сообщение.

**Критерии приёмки / Acceptance criteria:**
- [ ] `checked` бросает `OverflowException`, программа не падает.
- [ ] `unchecked` возвращает wrap-around значение.
- [ ] Порядок операций в выражении объяснён скобками и комментарием.
- [ ] Вывод содержит оба результата и сообщение об ошибке.

**Время / Time:** 30–45 мин

**Пример решения / Sample solution (для менторов):**
```csharp
// C# 12 / .NET 8
namespace M02.L04;

internal static class Program
{
    private static void Main()
    {
        int big = int.MaxValue;

        // checked: переполнение детектируется → исключение
        try
        {
            int _ = checked(big * 2);
        }
        catch (OverflowException ex)
        {
            Console.WriteLine($"checked  : OverflowException — {ex.Message}");
        }

        // unchecked: тихий wrap-around
        int wrapped = unchecked(big * 2);
        Console.WriteLine($"unchecked: {wrapped} (wrap-around)");

        // Приоритет: * и / раньше + и - ; скобки делают порядок явным.
        int plain = 10 + 2 * 3 - 4 / 2;        // 10 + (2*3) - (4/2) = 10 + 6 - 2 = 14
        int bracketed = 10 + (2 * 3) - (4 / 2);
        Console.WriteLine($"plain={plain}, bracketed={bracketed}");
    }
}
```

---

### Задание M02-L05 — Nullable и безопасный ввод / Nullable & safe input
**Задача / Task:**
Запросите у пользователя возраст. Если ввод пустой или не парсится, переменная возраста должна остаться `null`, а программа — напечатать дружелюбное сообщение. Используйте `int?` и `TryParse` через pattern-matching (`is int parsed`). Не бросайте исключений при плохом вводе.
Ask the user for their age. If the input is empty or fails to parse, the age variable must remain `null` and the program must print a friendly message. Use `int?` and `TryParse` via pattern matching (`is int parsed`). Do not throw exceptions on bad input.

**Требования / Requirements:**
- Переменная возраста: `int? age`.
- Чтение из `Console.ReadLine()` с защитой от `null` (`?? string.Empty`).
- `int.TryParse(..., out var parsed)` + `is int parsed` или `if (TryParse)`.
- Если `age` равно `null` — сообщение «Возраст не указан / Age not provided», иначе — печатаем возраст.
- Отдельный путь: ввод пустой строки обрабатывается явно (ранний выход).

**Критерии приёмки / Acceptance criteria:**
- [ ] Пустой ввод → `age == null`, без исключений.
- [ ] Нечисловой ввод → `age == null`, без исключений.
- [ ] Корректный ввод → печатается число.
- [ ] Используется `int?` и `TryParse`.

**Время / Time:** 30 мин

**Пример решения / Sample solution (для менторов):**
```csharp
// C# 12 / .NET 8
namespace M02.L05;

internal static class Program
{
    private static void Main()
    {
        Console.Write("Введите возраст / Enter age: ");
        string raw = Console.ReadLine() ?? string.Empty;

        int? age = null;

        if (!string.IsNullOrWhiteSpace(raw) &&
            int.TryParse(raw, out int parsed))
        {
            age = parsed;
        }

        if (age is int value)
        {
            Console.WriteLine($"Возраст / Age: {value}");
        }
        else
        {
            Console.WriteLine("Возраст не указан / Age not provided.");
        }
    }
}
```

---

### Задание M02-L06 — TryParse и конвертация ввода / TryParse & input conversion
**Задача / Task:**
Напишите функцию `bool TryReadDecimal(string prompt, out decimal value)`, которая печатает приглашение, читает строку и через `decimal.TryParse` (с `NumberStyles.Currency` и `CultureInfo.InvariantCulture`) пытается её разобрать. В `Main` запросите две суммы и сложите их только если обе введены корректно.
Write a function `bool TryReadDecimal(string prompt, out decimal value)` that prints a prompt, reads a line, and uses `decimal.TryParse` (with `NumberStyles.Currency` and `CultureInfo.InvariantCulture`) to parse it. In `Main`, ask for two amounts and sum them only if both parse successfully.

**Требования / Requirements:**
- Сигнатура точно `bool TryReadDecimal(string prompt, out decimal value)`.
- Использовать `NumberStyles.Currency | NumberStyles.AllowDecimalPoint` и `InvariantCulture`.
- При неудаче `value = 0` и возвращается `false`.
- `Main` складывает суммы только при двух `true`.

**Критерии приёмки / Acceptance criteria:**
- [ ] Функция имеет требуемую сигнатуру.
- [ ] Принимает и `"12.50"`, и `"12,50"`-стиль через инвариантную культуру корректно для точки.
- [ ] Некорректный ввод одной из сумм → отказ, без исключений.
- [ ] Выводится итог или сообщение об ошибке.

**Время / Time:** 30–45 мин

**Пример решения / Sample solution (для менторов):**
```csharp
// C# 12 / .NET 8
using System.Globalization;

namespace M02.L06;

internal static class Program
{
    private static bool TryReadDecimal(string prompt, out decimal value)
    {
        Console.Write(prompt);
        string? raw = Console.ReadLine();
        return decimal.TryParse(
            raw,
            NumberStyles.Currency | NumberStyles.AllowDecimalPoint,
            CultureInfo.InvariantCulture,
            out value);
    }

    private static void Main()
    {
        if (TryReadDecimal("Сумма A / Amount A: ", out decimal a) &&
            TryReadDecimal("Сумма B / Amount B: ", out decimal b))
        {
            Console.WriteLine($"Итого / Total: {(a + b):F2}");
        }
        else
        {
            Console.WriteLine("Некорректный ввод / Invalid input.");
        }
    }
}
```

---

### Задание M02-L07 — string vs StringBuilder / string vs StringBuilder
**Задача / Task:**
Соберите строку из N слов (например, числительных `"item1,item2,..."`) двумя способами: конкатенацией `string` в цикле и через `StringBuilder`. Измерьте и сравните время для `N = 10_000`. Объясните, почему `StringBuilder` быстрее.
Build a string of N words (e.g. numerals `"item1,item2,..."`) two ways: by `string` concatenation in a loop and via `StringBuilder`. Measure and compare the time for `N = 10_000`. Explain why `StringBuilder` is faster.

**Требования / Requirements:**
- `const int N = 10_000;`
- Способ 1: `string s = ""; for (...) s += $"item{i},";`
- Способ 2: `var sb = new StringBuilder(); for (...) sb.Append($"item{i},");`
- Замер через `System.Diagnostics.Stopwatch`.
- Удалить лишнюю запятую в конце у обоих вариантов.
- В комментарии — объяснение про иммутабельность `string` и аллокации.

**Критерии приёмки / Acceptance criteria:**
- [ ] Оба способа дают строки одинаковой длины и содержания (минус хвостовая запятая).
- [ ] `Stopwatch` показывает, что `StringBuilder` быстрее.
- [ ] Есть комментарий с объяснением причины.
- [ ] Код компилируется как C# 12 / .NET 8.

**Время / Time:** 30–45 мин

**Пример решения / Sample solution (для менторов):**
```csharp
// C# 12 / .NET 8
using System.Diagnostics;
using System.Text;

namespace M02.L07;

internal static class Program
{
    private const int N = 10_000;

    private static void Main()
    {
        // string иммутабельна: каждое += создаёт новую строку и копирует все символы → O(N^2).
        var sw = Stopwatch.StartNew();
        string s = string.Empty;
        for (int i = 0; i < N; i++)
            s += $"item{i},";
        if (s.EndsWith(','))
            s = s[..^1];
        sw.Stop();
        Console.WriteLine($"string concat: {sw.ElapsedMilliseconds} ms, len={s.Length}");

        // StringBuilder поддерживает изменяемый буфер → амортизированно O(N) аллокаций.
        sw.Restart();
        var sb = new StringBuilder();
        for (int i = 0; i < N; i++)
            sb.Append($"item{i},");
        if (sb.Length > 0 && sb[^1] == ',')
            sb.Length--;             // удалить хвостовую запятую без новой аллокации
        string built = sb.ToString();
        sw.Stop();
        Console.WriteLine($"StringBuilder : {sw.ElapsedMilliseconds} ms, len={built.Length}");

        Console.WriteLine($"identical content: {s == built}");
    }
}
```

---

### Задание M02-L08 — enum и кортежи: статус заказа / enum & tuples: order status
**Задача / Task:**
Определите `enum OrderStatus { Pending, Paid, Shipped, Delivered, Cancelled }`. Напишите метод `(OrderStatus Status, string Label, bool IsFinal) Describe(OrderStatus s)`, возвращающий кортеж с человекочитаемой меткой и признаком финального состояния. В `Main` прогоните все значения `Enum.GetValues<OrderStatus>()` и выведите таблицу.
Define `enum OrderStatus { Pending, Paid, Shipped, Delivered, Cancelled }`. Write a method `(OrderStatus Status, string Label, bool IsFinal) Describe(OrderStatus s)` returning a tuple with a human-readable label and a final-state flag. In `Main`, iterate all values via `Enum.GetValues<OrderStatus>()` and print a table.

**Требования / Requirements:**
- `enum OrderStatus` с ровно пятью значениями.
- Метод возвращает именованный кортеж из трёх элементов.
- `IsFinal = true` для `Delivered` и `Cancelled`.
- Использовать `switch` expression.
- Вывод в виде выровненной таблицы.

**Критерии приёмки / Acceptance criteria:**
- [ ] `enum` содержит ровно 5 значений в указанном порядке.
- [ ] Метод возвращает кортеж с именованными полями.
- [ ] `IsFinal` корректен для всех статусов.
- [ ] Используется `switch` expression и `Enum.GetValues<OrderStatus>()`.

**Время / Time:** 30–45 мин

**Пример решения / Sample solution (для менторов):**
```csharp
// C# 12 / .NET 8
namespace M02.L08;

public enum OrderStatus { Pending, Paid, Shipped, Delivered, Cancelled }

internal static class Program
{
    private static (OrderStatus Status, string Label, bool IsFinal) Describe(OrderStatus s) => s switch
    {
        OrderStatus.Pending   => (s, "Ожидает оплаты / Awaiting payment", false),
        OrderStatus.Paid      => (s, "Оплачен / Paid",                false),
        OrderStatus.Shipped   => (s, "Отправлен / Shipped",            false),
        OrderStatus.Delivered => (s, "Доставлен / Delivered",          true),
        OrderStatus.Cancelled => (s, "Отменён / Cancelled",            true),
        _ => throw new ArgumentOutOfRangeException(nameof(s), s, "Unknown status")
    };

    private static void Main()
    {
        Console.WriteLine($"{"Status",-12} {"Final",-7} Label");
        Console.WriteLine(new string('-', 50));
        foreach (OrderStatus s in Enum.GetValues<OrderStatus>())
        {
            var d = Describe(s);
            Console.WriteLine($"{d.Status,-12} {d.IsFinal,-7} {d.Label}");
        }
    }
}
```

---

## Мини-проект модуля / Module mini-project

### Базовая версия / Base version
**Задача / Task:**
Калькулятор скидок: ввести цену (`decimal`) и процент скидки (`decimal`), вычислить и вывести итоговую цену со скидкой. Все суммы — в `decimal`, ввод защищён `TryParse`, процент ограничен диапазоном `0..100`.
Discount calculator: input a price (`decimal`) and a discount percentage (`decimal`), compute and print the final price after discount. All amounts use `decimal`, input is guarded by `TryParse`, percentage is bounded to `0..100`.

**Требования / Requirements:**
- Запрос цены и процента через `TryReadDecimal` (можно повторно использовать из L06).
- Проверка `0 <= percent <= 100`; иначе сообщение и выход.
- `discount = price * percent / 100m`; `final = price - discount`.
- Вывод: исходная цена, скидка, итог — с 2 знаками, `InvariantCulture`.
- Без исключений при любом вводе.

**Критерии приёмки / Acceptance criteria:**
- [ ] Цена `100`, процент `10` → итог `90.00`.
- [ ] Процент `0` → итог равен цене; процент `100` → итог `0.00`.
- [ ] Процент вне `[0;100]` → отказ, без исключений.
- [ ] Нечисловой ввод → отказ, без исключений.
- [ ] Все вычисления в `decimal`.

**Время / Time:** 45–60 мин

**Пример решения / Sample solution:**
```csharp
// C# 12 / .NET 8
using System.Globalization;

namespace M02.MiniProject.Base;

internal static class Program
{
    private static bool TryReadDecimal(string prompt, out decimal value)
    {
        Console.Write(prompt);
        string? raw = Console.ReadLine();
        return decimal.TryParse(
            raw,
            NumberStyles.Currency | NumberStyles.AllowDecimalPoint,
            CultureInfo.InvariantCulture,
            out value);
    }

    private static void Main()
    {
        if (!TryReadDecimal("Цена / Price: ", out decimal price))
        {
            Console.WriteLine("Некорректная цена / Invalid price.");
            return;
        }

        if (!TryReadDecimal("Скидка % / Discount %: ", out decimal percent))
        {
            Console.WriteLine("Некорректный процент / Invalid percent.");
            return;
        }

        if (percent < 0m || percent > 100m)
        {
            Console.WriteLine("Процент должен быть в диапазоне 0..100 / Percent must be 0..100.");
            return;
        }

        decimal discount = price * percent / 100m;
        decimal final = price - discount;

        Console.WriteLine($"Price   : {price.ToString("F2", CultureInfo.InvariantCulture)}");
        Console.WriteLine($"Discount: {discount.ToString("F2", CultureInfo.InvariantCulture)}");
        Console.WriteLine($"Final   : {final.ToString("F2", CultureInfo.InvariantCulture)}");
    }
}
```

### Pro-версия / Pro version
**Задача / Task:**
Расширить калькулятор: добавить `enum CustomerType { Regular, Loyalty, Vip }`, который задаёт базовую скидку; кортеж `(decimal Price, decimal Discount, decimal Final)` как результат метода; `string? promoCode` — опциональный промокод, дающий дополнительную фиксированную скидку при совпадении с известным кодом. Итоговая скидка не может превышать цену.
Extend the calculator: add `enum CustomerType { Regular, Loyalty, Vip }` providing a base discount; a tuple `(decimal Price, decimal Discount, decimal Final)` as the method result; `string? promoCode` — an optional promo code granting an extra fixed discount when it matches a known code. The total discount cannot exceed the price.

**Доп. требования / Additional requirements:**
- `enum CustomerType { Regular, Loyalty, Vip }`.
- Базовая скидка по типу: `Regular = 0%`, `Loyalty = 5%`, `Vip = 10%`.
- Вводимый пользователем процент — это «вручную заданная» скидка, которая складывается с базовой (но суммарный процент ≤ 100).
- `promoCode` — `string?`; если равно `"SAVE10"` — дополнительная скидка `10` (валюта, не проценты), иначе — без бонуса.
- Метод `Compute(decimal price, CustomerType type, decimal extraPercent, string? promo)` возвращает кортеж `(decimal Price, decimal Discount, decimal Final)`.
- Защита: `discount = min(discount, price)`; цена неотрицательна.
- Использовать `TryReadDecimal` и `Enum.TryParse<CustomerType>` для ввода типа.

**Критерии приёмки / Acceptance criteria:**
- [ ] `enum CustomerType` содержит ровно `Regular, Loyalty, Vip`.
- [ ] Метод возвращает именованный кортеж `(Price, Discount, Final)`.
- [ ] Промокод `null` или пустой → бонуса нет; `"SAVE10"` → +10 к скидке.
- [ ] Скидка никогда не превышает цену (`Final >= 0`).
- [ ] Базовая + ручная скидка суммируются, но ≤ 100%.
- [ ] Все расчёты в `decimal`, без исключений при любом вводе.

**Время:** 60–90 мин

**Пример решения / Sample solution:**
```csharp
// C# 12 / .NET 8
using System.Globalization;

namespace M02.MiniProject.Pro;

public enum CustomerType { Regular, Loyalty, Vip }

internal static class Program
{
    private const string KnownPromo = "SAVE10";
    private const decimal PromoBonus = 10m;

    private static bool TryReadDecimal(string prompt, out decimal value)
    {
        Console.Write(prompt);
        string? raw = Console.ReadLine();
        return decimal.TryParse(
            raw,
            NumberStyles.Currency | NumberStyles.AllowDecimalPoint,
            CultureInfo.InvariantCulture,
            out value);
    }

    private static decimal BasePercent(CustomerType type) => type switch
    {
        CustomerType.Regular => 0m,
        CustomerType.Loyalty => 5m,
        CustomerType.Vip     => 10m,
        _ => 0m
    };

    private static (decimal Price, decimal Discount, decimal Final) Compute(
        decimal price, CustomerType type, decimal extraPercent, string? promo)
    {
        if (price < 0m) price = 0m;

        decimal totalPercent = BasePercent(type) + extraPercent;
        if (totalPercent < 0m) totalPercent = 0m;
        if (totalPercent > 100m) totalPercent = 100m;

        decimal discount = price * totalPercent / 100m;

        if (!string.IsNullOrWhiteSpace(promo) &&
            promo.Trim().Equals(KnownPromo, StringComparison.OrdinalIgnoreCase))
        {
            discount += PromoBonus;
        }

        // Скидка не может превышать цену.
        if (discount > price) discount = price;

        decimal final = price - discount;
        return (price, discount, final);
    }

    private static void Main()
    {
        if (!TryReadDecimal("Цена / Price: ", out decimal price))
        {
            Console.WriteLine("Некорректная цена / Invalid price.");
            return;
        }

        Console.Write("Тип клиента (Regular/Loyalty/Vip) / Customer type: ");
        string? typeRaw = Console.ReadLine();
        if (!Enum.TryParse<CustomerType>(typeRaw, ignoreCase: true, out CustomerType type))
        {
            Console.WriteLine("Некорректный тип / Invalid customer type.");
            return;
        }

        if (!TryReadDecimal("Доп. скидка % / Extra discount %: ", out decimal extraPercent))
        {
            extraPercent = 0m;
        }

        Console.Write("Промокод (необязательно) / Promo code (optional): ");
        string? promo = Console.ReadLine();

        var r = Compute(price, type, extraPercent, promo);

        Console.WriteLine($"Customer : {type}");
        Console.WriteLine($"Price    : {r.Price.ToString("F2", CultureInfo.InvariantCulture)}");
        Console.WriteLine($"Discount : {r.Discount.ToString("F2", CultureInfo.InvariantCulture)}");
        Console.WriteLine($"Final    : {r.Final.ToString("F2", CultureInfo.InvariantCulture)}");
    }
}
```
