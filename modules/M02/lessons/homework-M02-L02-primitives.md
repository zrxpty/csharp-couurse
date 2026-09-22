---
[← К уроку M02-L02](lesson-M02-L02-primitives.md) | [⬆ К модулю M02](../README.md) | [Следующее ДЗ →](homework-M02-L03-var-const.md)
---

### Домашнее задание M02-L02: Примитивы: int, double, decimal, bool, char, string / Homework M02-L02: Primitives: int, double, decimal, bool, char, string

**Урок / Lesson:** M02-L02
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) Научиться осознанно выбирать примитивные типы C# 12 под природу значения (счёт, физика, деньги, логика, символ, текст), применять суффиксы литералов, работать с `decimal` для финансов, понимать ограниченную точность `double`, отличать `char` от `string`, использовать сырые строковые литералы и pattern matching. (EN) Learn to consciously choose C# 12 primitive types to match the nature of the value (counting, physics, money, logic, symbol, text), apply literal suffixes, do financial math in `decimal`, understand the limited precision of `double`, tell `char` from `string`, and use raw string literals together with pattern matching.

#### Связь с уроком / Connection to the lesson
(RU) ДЗ закрепляет ключевые концепции урока: выбор целого типа по размеру значения и обязательные суффиксы `L`/`u`/`ul`, правило «деньги — в `decimal`» с суффиксом `m`, природа `double` по IEEE 754 и сравнение через epsilon, логический `bool` в условиях, `char` в одинарных кавычках как UTF-16-кодовую единицу, неизменяемый `string` и сырые строковые литералы `"""..."""`. Повторяются best practices и частые ошибки из урока (`double money`, `long` без `L`, `decimal` без `m`, путаница `char`/`string`, присваивание вместо сравнения).
(EN) The homework reinforces the lesson's key concepts: choosing an integer type by value size with mandatory `L`/`u`/`ul` suffixes, the "money goes in `decimal`" rule with the `m` suffix, the IEEE 754 nature of `double` and epsilon-based comparison, the `bool` type in conditions, `char` in single quotes as a UTF-16 code unit, immutable `string` and raw string literals `"""..."""`. It revisits the lesson's best practices and common mistakes (`double money`, `long` without `L`, `decimal` without `m`, `char`/`string` confusion, assignment instead of comparison).

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Вы присоединяетесь к команде, которая пишет backend для небольшого онлайн-магазина «Кофейная гуща». В репозитории уже есть каркас консольного приложения на .NET 8, но в модуле `Shop` разработчики-стажёры наспех использовали `double` для цен, путали `char` и `string`, забывали суффиксы литералов и не понимали, почему `0.1 + 0.2` не равно `0.3`. Технический лидер поручил вам: переписать финансовое ядро с `decimal`, навести порядок с целочисленными счётчиками и литералами, добавить корректную работу с символами валют, текстовыми шаблонами и проверками флагов состояния. Это не абстрактное упражнение — это типичный фрагмент реального кода, где неправильный выбор типа приводит к потерям копеек на налогах, переполнению литералов при компиляции и трудноуловимым ошибкам округления в отчётах. Вы пройдёте путь от создания проекта `dotnet new console` до полностью рабочего финансового модуля, который корректно считает цену с НДС, формирует чек, демонстрирует IEEE 754 на контрасте `double` vs `decimal` и использует pattern matching по типам для описания произвольных значений. Каждый шаг опирается на конкретные примеры и правила из урока M02-L02: суффиксы `L`, `f`, `m`, `u`, `ul`; правило «деньги — в `decimal`, физика — в `double`, счётчики — в `int`»; сравнение `double` через epsilon; неизменяемость `string` и сырые строковые литералы.

#### Что нужно сделать (пошагово)
1. Создайте новый проект в подпапке `src/Shop`: выполните `dotnet new console -n Shop -o src/Shop -f net8.0` из корня репозитория. Убедитесь, что в `src/Shop/Shop.csproj` указано `<TargetFramework>net8.0</TargetFramework>` и что язык по умолчанию — C# 12 (для .NET 8 это так). Откройте `src/Shop/Program.cs` и удалите шаблонный `Console.WriteLine("Hello, World!");`.
2. Реализуйте блок целых чисел. Объявите: `int itemsInStock = 12_500;` (складской остаток), `long worldCustomers = 8_100_000_000L;` (гипотетическая глобальная база клиентов — обязательно с суффиксом `L`, иначе компилятор сочтёт литерал `int` и выдаст ошибку переполнения), `byte roastLevel = 3;` (степень обжарки, 0–10), `uint loyaltyPoints = 4_000_000_000u;` (баллы лояльности, физически неотрицательны — суффикс `u`), `ulong totalBeansSold = 10_000_000_000ul;` (суффикс `ul`). Выведите их через интерполяцию с форматом `:N0` для разрядности.
3. Реализуйте блок чисел с плавающей точкой. Объявите `double latitude = 55.7558;` и `double longitude = 37.6173;` (координаты кофейни — физика, `double` уместен). Объявите `float brewingTemp = 92.5f;` (суффикс `f`). Покажите ловушку IEEE 754: `double dirtySum = 0.1 + 0.2;` и выведите его — убедитесь, что получается не ровно `0.3`. Реализуйте метод `bool ApproxEquals(double a, double b, double eps = 1e-9)`, возвращающий `Math.Abs(a - b) < eps`, и продемонстрируйте, что `ApproxEquals(0.1 + 0.2, 0.3)` даёт `true`, тогда как прямой `==` — `false`.
4. Реализуйте финансовое ядро на `decimal`. Объявите `decimal price = 19.99m;` (цена за пачку кофе, суффикс `m` обязателен — без него литерал `19.99` трактуется как `double` и неявно не преобразуется в `decimal`). Реализуйте `decimal vatRate = 0.20m;` (НДС 20%) и вычислите `decimal vat = price * vatRate;`, `decimal total = price + vat;`. Выведите все три значения с форматом `:C` (валюта). Сравните с тем же расчётом в `double`: `double dPrice = 19.99; double dTotal = dPrice * 1.20;` и покажите, что `decimal` даёт точное `23.988`, а `double` — значение с хвостовой погрешностью. В комментарии объясните, почему деньги считают в `decimal`.
5. Реализуйте блок `bool`. Заведите `bool isPaid = true;`, `bool isMember = false;`, `bool hasDiscount = isMember && !isPaid;` и используйте `if (isPaid)` для вывода сообщения. В комментарии отметьте, что в C# `bool` не приводится к `int` автоматически — это защищает от `if (x = 5)`.
6. Реализуйте блок `char`. Объявите `char currency = '₽';` и `char euro = '\u20AC';` (€ через escape-последовательность Unicode, как в уроке). Выведите их и поясните, что `char` — это одна UTF-16-кодовая единица в одинарных кавычках, в отличие от `string` в двойных.
7. Реализуйте блок `string` с сырыми строковыми литералами. Создайте `string greeting = "Добро пожаловать в «Кофейную гущу»!";`. Сформируйте JSON-чек через raw string literal `"""..."""` с полями `order`, `price`, `vat`, `total`. Дополнительно соберите многострочный шаблон чека с интерполяцией — обратите внимание, что конкатенация и `+` создают новые строки, потому что `string` неизменяем.
8. Реализуйте pattern matching по типу. Объявите `object sample = 42;` (потом последовательно меняйте на `19.99m`, `55.7558`, `true`, `"Latte"`, `'€'`) и используйте `switch` с шаблонами `int i`, `decimal d`, `double dd`, `bool b`, `string s`, `char c` и `_`, как в примере урока. Для `int` добавьте защитное условие `when i < 0` для отрицательных чисел. Выведите описание каждого значения.
9. Соберите проект: `dotnet build src/Shop`. Запустите: `dotnet run --project src/Shop`. Зафиксируйте ожидаемый вывод: координаты, `0.1+0.2` с предупреждением, точный финансовый расчёт до копейки, символы валют, JSON-чек и результаты pattern matching. Убедитесь, что компилятор не выдаёт ошибок литералов и переполнений.

#### Требования к решению
- Целевая платформа — .NET 8, язык C# 12; используйте top-level statements в `Program.cs` (без явного `class Program` / `static void Main`).
- Все числовые литералы, выходящие за `int`/`double` по умолчанию, должны нести явные суффиксы: `L` для `long`, `f` для `float`, `m` для `decimal`, `u` для `uint`, `ul` для `ulong`. Подчёркивания `1_000_000` как разделитель разрядов — обязательны для больших чисел.
- Финансовые расчёты (цена, НДС, итог) выполняются строго в `decimal`; сравнение с `double`-вариантом приведено только как негативный пример в отдельном блоке с комментарием.
- Сравнение `double` на равенство реализовано через метод с epsilon (`1e-9` или меньше), а не через `==`. В коде должен быть наглядный контраст: `dirtySum == 0.3` → `false`, `ApproxEquals(dirtySum, 0.3)` → `true`.
- `char` пишется в одинарных кавычках и хотя бы один символ задаётся через Unicode-escape `\uXXXX`; `string` — в двойных кавычках, и хотя бы одна строка — сырой литерал `"""..."""`.
- `bool` используется в `if` без приведения к числу; в комментарии отмечена защита C# от `if (x = 5)`.
- Pattern matching по типу через `switch` covers не менее 6 вариантов (`int`, `decimal`, `double`, `bool`, `string`, `char`) плюс `_`; для `int` есть ветка с `when i < 0`.
- Код компилируется без предупреждений уровня `CS0168`/`CS0219`-типа (объявил, но не используешь) и без ошибок переполнения литералов; вывод соответствует описанию.
- Все комментарии в коде — двуязычные RU+EN, как в примере урока.

#### Тонкости и подводные камни
- **Суффикс `L` обязателен** для литералов `long`, превышающих `int.MaxValue`. `long big = 5_000_000_000;` без `L` — ошибка компиляции: компилятор сначала пытается вместить литерал в `int` и падает с переполнением, ещё не дойдя до неявного преобразования в `long`. Пишите `5_000_000_000L`. Аналогично `uint` требует `u`, `ulong` — `ul`, `decimal` — `m`, `float` — `f`.
- **`decimal d = 1.5;` без `m` — ошибка**, потому что литерал `1.5` по умолчанию `double`, а неявного сужающего преобразования `double` → `decimal` нет (оно может терять точность). Всегда `1.5m`.
- **`0.1 + 0.2 != 0.3` в `double`** — это не баг, а свойство двоичной формы IEEE 754: `0.1` и `0.2` не представимы точно в двоичной дроби. Для денег это критично, поэтому `decimal` (десятичная base-10 форма) даёт ровно `0.3m`. Сравнивать `double` нужно через `Math.Abs(a - b) < eps`, а не `==`.
- **`char` vs `string`:** `char c = "A";` — ошибка компиляции; `char` в одинарных кавычках `'A'`, `string` в двойных `"A"`. Символ валюты можно задать напрямую (`'₽'`) или через escape (`'\u20BD'` — знак рубля, `'\u20AC'` — €).
- **`string` неизменяем:** любая «модификация» (`s += "x"`) создаёт новый объект. В горячих циклах используйте `StringBuilder` (бонус). Сырые строковые литералы `"""..."""` снимают необходимость экранировать кавычки и слэши — идеально для JSON/SQL/regex.
- **`bool` и `if (x = 5)`:** в C/C++ это присваивание, в C# — ошибка компиляции, потому что `if` требует `bool`, а `int` в `bool` не приводится. Тем не менее следите за `==` в условиях.
- **Беззнаковые типы (`uint`, `ulong`)** удобны для счётчиков, но в публичных API их избегают из-за проблем совместимости с другими языками CLS. Внутри модуля — допустимо.
- **Форматирование:** `:C` — валюта, `:N0` — число с разделителями разрядов без дробной части, `:F2` — два знака после запятой. Для `decimal` `:C` даёт корректное представление денег.

#### Критерии приёмки
- [ ] Проект `src/Shop` создан через `dotnet new console -f net8.0`, в `.csproj` указан `net8.0`.
- [ ] В `Program.cs` используются top-level statements C# 12, без явного `class Program`.
- [ ] Целочисленный блок содержит `int`, `long` (с `L`), `byte`, `uint` (с `u`), `ulong` (с `ul`), с разделителями `_`.
- [ ] Блок `double` демонстрирует `0.1 + 0.2 != 0.3` и метод `ApproxEquals` с epsilon.
- [ ] Финансовый блок использует `decimal` с суффиксом `m`; цена, НДС и итог выведены с `:C`.
- [ ] Есть явный негативный пример того же расчёта в `double` с комментарием о погрешности.
- [ ] `bool` используется в `if` без приведения к числу; есть комментарий про защиту от `if (x = 5)`.
- [ ] `char` в одинарных кавычках, минимум один символ через `\uXXXX`.
- [ ] `string` включает сырой строковый литерал `"""..."""` (JSON-чек).
- [ ] Pattern matching `switch` покрывает не менее 6 типов + `_`, с `when i < 0` для `int`.
- [ ] `dotnet build src/Shop` проходит без ошибок и без предупреждений о неиспользуемых переменных.
- [ ] `dotnet run --project src/Shop` выводит ожидаемые строки: координаты, предупреждение о `0.1+0.2`, точный финансовый итог, символы валют, JSON, описания типов.
- [ ] Все комментарии в коде двуязычные RU+EN.
- [ ] В файле нет TODO, заглушек, закомментированного нерабочего кода.

#### Подсказки (без прямого ответа)
- Вспомните правило из урока: «деньги — в `decimal`, физика — в `double`, счётчики — в `int`». Подумайте, к какой категории относится каждый шаг.
- Для epsilon используйте `Math.Abs(a - b) < 1e-9`. Почему `==` не работает с `double`? Потому что двоичная форма IEEE 754 не точно представляет `0.1`.
- Сырой строковый литерал начинается и заканчивается тремя двойными кавычками `"""`; внутри не нужно экранировать `"`.
- Для pattern matching по типу используйте синтаксис `sample switch { int i => ..., decimal d => ..., _ => ... }`. Защитное условие — `int i when i < 0 => ...`.
- Если компилятор ругается на литерал `long` — добавьте `L`. Если на `decimal` без `m` — добавьте `m`. Это самые частые ошибки стажёров из урока.

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 — top-level statements
// ДЗ M02-L02: Магазин «Кофейная гуща» / Homework: "Coffee Grounds" shop

using System;
using System.Globalization;
using System.Text;

// Культура invariant, чтобы :C не зависел от локали демо-машины
// Invariant culture so :C does not depend on the demo machine locale
CultureInfo.CurrentCulture = CultureInfo.InvariantCulture;

// --- 1. Целые / Integers ---
int itemsInStock = 12_500;              // остаток на складе / stock
long worldCustomers = 8_100_000_000L;   // суффикс L обязателен / L suffix required
byte roastLevel = 3;                    // 0..10, обжарка / roast
uint loyaltyPoints = 4_000_000_000u;    // суффикс u / u suffix
ulong totalBeansSold = 10_000_000_000ul;// суффикс ul / ul suffix

Console.WriteLine($"itemsInStock={itemsInStock:N0} (int)");
Console.WriteLine($"worldCustomers={worldCustomers:N0} (long, L-suffix)");
Console.WriteLine($"roastLevel={roastLevel} (byte)");
Console.WriteLine($"loyaltyPoints={loyaltyPoints:N0} (uint, u-suffix)");
Console.WriteLine($"totalBeansSold={totalBeansSold:N0} (ulong, ul-suffix)");

// --- 2. С плавающей точкой / Floating point ---
double latitude = 55.7558;              // физика — double / physics
double longitude = 37.6173;
float brewingTemp = 92.5f;              // суффикс f / f suffix

double dirtySum = 0.1 + 0.2;            // НЕ равно ровно 0.3 / NOT exactly 0.3
Console.WriteLine($"latitude={latitude}, longitude={longitude}, temp={brewingTemp}f");
Console.WriteLine($"0.1 + 0.2 = {dirtySum:G17}  (ожидается ~0.30000000000000004 / expected ~0.30000000000000004)");
Console.WriteLine($"  dirtySum == 0.3 ? {dirtySum == 0.3}        // bare == даёт false / bare == gives false");
Console.WriteLine($"  ApproxEquals(0.1+0.2, 0.3) ? {ApproxEquals(dirtySum, 0.3)}  // через epsilon / via epsilon");

// --- 3. decimal для финансов / decimal for money ---
decimal price = 19.99m;                 // суффикс m обязателен / m suffix required
decimal vatRate = 0.20m;                // НДС 20% / VAT 20%
decimal vat = price * vatRate;
decimal total = price + vat;            // ровно до копейки / exact to the cent

Console.WriteLine($"price={price:C}, vat={vat:C}, total={total:C}");

// Негативный пример: тот же расчёт в double / Negative example: same calc in double
double dPrice = 19.99;
double dTotal = dPrice * 1.20;
Console.WriteLine($"double total = {dTotal:G17}  (погрешность IEEE 754 / IEEE 754 drift)");

// --- 4. bool / logical ---
bool isPaid = true;
bool isMember = false;
bool hasDiscount = isMember && !isPaid; // false, т.к. isMember=false / false since isMember=false
if (isPaid)
{
    Console.WriteLine("Заказ оплачен / Order paid");
}
// В C# if требует bool, поэтому if (x = 5) не скомпилируется — защита от опечатки
// In C#, if requires bool, so if (x = 5) won't compile — typo protection

// --- 5. char ---
char currency = '₽';                    // Unicode UTF-16 напрямую / direct
char euro = '\u20AC';                   // € через escape / via escape
Console.WriteLine($"currency={currency}, euro={euro}");

// --- 6. string + raw string literal ---
string greeting = "Добро пожаловать в «Кофейную гущу»!";
string receiptJson = """
{
  "order": "Latte 1kg",
  "price": 19.99,
  "vat": 3.998,
  "total": 23.988
}
""";
Console.WriteLine(greeting);
Console.WriteLine(receiptJson);

// string неизменяем — конкатенация создаёт новые объекты / string is immutable
string receipt = "Чек / Receipt: ";
receipt += $"Итого {total:C}";          // новый объект / new object
Console.WriteLine(receipt);

// --- 7. Pattern matching по типу / type pattern matching ---
foreach (object sample in new object[] { 42, -7, 19.99m, 55.7558, true, "Latte", '€' })
{
    string description = sample switch
    {
        int i when i < 0 => $"Отрицательное целое / Negative integer: {i}",
        int i            => $"Целое / Integer: {i}",
        decimal d        => $"Деньги / Money: {d:C}",
        double dd        => $"Дробь / Double: {dd:G17}",
        bool b           => $"Логическое / Boolean: {b}",
        string s         => $"Строка / String: {s}",
        char c           => $"Символ / Char: {c}",
        _                => "Неизвестно / Unknown"
    };
    Console.WriteLine(description);
}

// Локальная функция: сравнение double через epsilon / local function for double equality
static bool ApproxEquals(double a, double b, double eps = 1e-9) =>
    Math.Abs(a - b) < eps;
```

Разбор по строкам. Строка `using System.Globalization;` и установка `CultureInfo.CurrentCulture = CultureInfo.InvariantCulture;` нужна, чтобы формат `:C` выводил валюту предсказуемо, а не зависел от локали демо-машины — это частый подводный камень при проверке ДЗ на разных ОС. Блок целых чисел повторяет правило урока: `int` для счётчиков, `long` с суффиксом `L` для больших значений (без `L` литерал `8_100_000_000` не помещается в `int` и вызывает ошибку компиляции), `byte` для маленьких диапазонов, `uint` с `u` и `ulong` с `ul` для беззнаковых счётчиков. Подчёркивания `12_500` — легальный разделитель разрядов из урока, улучшающий читаемость. Блок `double` показывает суть IEEE 754: `0.1 + 0.2` даёт `0.30000000000000004`, поэтому прямой `==` возвращает `false`, а метод `ApproxEquals` с epsilon `1e-9` — `true`; формат `:G17` выводит все значащие цифры, чтобы погрешность была видна. Финансовый блок — ядро ДЗ: `decimal price = 19.99m;` с обязательным суффиксом `m` (без него литерал `double` и неявного преобразования нет), `decimal vat = price * vatRate;` и `decimal total = price + vat;` дают точный результат без накопления погрешности, потому что `decimal` хранится в base-10 форме. Негативный пример в `double` рядом — чтобы студент видел разницу. Блок `bool` использует `if (isPaid)` без приведения к числу; комментарий фиксирует, что C# запрещает `if (x = 5)`, в отличие от C/C++. Блок `char` показывает и прямой символ `'₽'`, и escape `\u20AC` для € — обе формы из урока. Блок `string` включает сырой литерал `"""..."""` для JSON без экранирования кавычек и конкатенацию с комментарием о неизменяемости. Pattern matching `switch` покрывает 6 типов плюс `_`, с защитным условием `when i < 0` для отрицательных `int` — точно как в примере урока. Локальная функция `ApproxEquals` объявлена внизу, что допустимо в top-level statements. Применённые концепции урока: выбор типа по природе значения, суффиксы литералов, IEEE 754 и epsilon, `decimal` для финансов, `char` vs `string`, неизменяемость `string`, сырые строковые литералы, pattern matching по типу.

#### Задания на углубление (бонус)
1. Замените конкатенацию строк в цикле (соберите чек из 1000 строк) на `StringBuilder` и измерьте разницу во времени через `Stopwatch`. Объясните, почему `string +=` в горячем цикле медленный из-за неизменяемости.
2. Добавьте тип `float` с суффиксом `f` и покажите, что `0.1f + 0.2f` тоже не равно `0.3f`, но погрешность больше, чем у `double`. Сравните количество значащих цифр `float` (≈7) и `double` (≈15–17).
3. Реализуйте метод `decimal RoundToCents(decimal value)`, округляющий до копеек через `decimal.Round(value, 2, MidpointRounding.ToEven)`, и объясните, почему банковское округление (`ToEven`) снижает систематическую ошибку в финансовой отчётности.
4. Используйте collection expressions C# 12 (`int[] xs = [1, 2, 3];`) для списка заказов и примените pattern matching по типу к элементам `object[]`, собранным через collection expression.

---

## Statement in English / Постановка на английском

#### Context & motivation
You are joining a team that writes the backend for a small online coffee shop called "Coffee Grounds". The repository already has the skeleton of a .NET 8 console application, but in the `Shop` module junior developers carelessly used `double` for prices, confused `char` and `string`, forgot literal suffixes, and did not understand why `0.1 + 0.2` is not `0.3`. The tech lead assigned you to rewrite the financial core with `decimal`, tidy up the integral counters and literals, add correct handling of currency symbols and text templates, and wire up state-flag checks. This is not an abstract exercise — it is a typical slice of real code where the wrong type choice causes lost cents on taxes, compile-time literal overflows, and hard-to-spot rounding drift in reports. You will go from creating the project with `dotnet new console` to a fully working financial module that correctly computes a VAT-inclusive price, prints a receipt, demonstrates IEEE 754 by contrasting `double` vs `decimal`, and uses pattern matching by type to describe arbitrary values. Every step leans on concrete examples and rules from lesson M02-L02: suffixes `L`, `f`, `m`, `u`, `ul`; the rule "money in `decimal`, physics in `double`, counters in `int`"; epsilon comparison for `double`; `string` immutability and raw string literals.

#### What to do step by step
1. Create a new project in the `src/Shop` subfolder: from the repository root run `dotnet new console -n Shop -o src/Shop -f net8.0`. Confirm that `src/Shop/Shop.csproj` contains `<TargetFramework>net8.0</TargetFramework>` and that the default language is C# 12 (on .NET 8 it is). Open `src/Shop/Program.cs` and delete the template `Console.WriteLine("Hello, World!");`.
2. Implement the integer block. Declare `int itemsInStock = 12_500;` (warehouse stock), `long worldCustomers = 8_100_000_000L;` (a hypothetical global customer base — the `L` suffix is mandatory, otherwise the compiler reads the literal as `int` and reports an overflow), `byte roastLevel = 3;` (roast level, 0–10), `uint loyaltyPoints = 4_000_000_000u;` (loyalty points, physically never negative — `u` suffix), `ulong totalBeansSold = 10_000_000_000ul;` (`ul` suffix). Print them with interpolation and the `:N0` format for digit grouping.
3. Implement the floating-point block. Declare `double latitude = 55.7558;` and `double longitude = 37.6173;` (coffee shop coordinates — physics, `double` is appropriate). Declare `float brewingTemp = 92.5f;` (`f` suffix). Demonstrate the IEEE 754 trap: `double dirtySum = 0.1 + 0.2;` and print it — you should see it is not exactly `0.3`. Implement a method `bool ApproxEquals(double a, double b, double eps = 1e-9)` returning `Math.Abs(a - b) < eps`, and show that `ApproxEquals(0.1 + 0.2, 0.3)` yields `true` while a bare `==` yields `false`.
4. Implement the financial core in `decimal`. Declare `decimal price = 19.99m;` (price per bag of coffee — the `m` suffix is mandatory; without it the literal `19.99` is `double` and does not implicitly convert to `decimal`). Define `decimal vatRate = 0.20m;` (20% VAT) and compute `decimal vat = price * vatRate;`, `decimal total = price + vat;`. Print all three with the `:C` (currency) format. Compare with the same calculation in `double`: `double dPrice = 19.99; double dTotal = dPrice * 1.20;` and show that `decimal` gives the exact `23.988` while `double` carries a trailing drift. In a comment explain why money is computed in `decimal`.
5. Implement the `bool` block. Introduce `bool isPaid = true;`, `bool isMember = false;`, `bool hasDiscount = isMember && !isPaid;` and use `if (isPaid)` to print a message. In a comment note that in C# `bool` does not auto-convert to `int`, which guards against `if (x = 5)`.
6. Implement the `char` block. Declare `char currency = '₽';` and `char euro = '\u20AC';` (€ via a Unicode escape, as in the lesson). Print them and explain that `char` is a single UTF-16 code unit in single quotes, unlike `string` in double quotes.
7. Implement the `string` block with raw string literals. Create `string greeting = "Welcome to Coffee Grounds!";`. Build a JSON receipt using the raw string literal `"""..."""` with fields `order`, `price`, `vat`, `total`. Additionally build a multiline receipt template with interpolation — note that concatenation and `+` create new strings because `string` is immutable.
8. Implement pattern matching by type. Declare `object sample = 42;` (then change it in turn to `19.99m`, `55.7558`, `true`, `"Latte"`, `'€'`) and use a `switch` with patterns `int i`, `decimal d`, `double dd`, `bool b`, `string s`, `char c`, and `_`, as in the lesson example. For `int` add a guard `when i < 0` for negative numbers. Print a description for each value.
9. Build the project: `dotnet build src/Shop`. Run it: `dotnet run --project src/Shop`. Record the expected output: coordinates, the `0.1+0.2` warning, the exact financial result to the cent, currency symbols, the JSON receipt, and the pattern-matching results. Confirm the compiler reports no literal or overflow errors.

#### Requirements
- Target platform is .NET 8, language C# 12; use top-level statements in `Program.cs` (no explicit `class Program` / `static void Main`).
- Every numeric literal that exceeds the default `int`/`double` range must carry an explicit suffix: `L` for `long`, `f` for `float`, `m` for `decimal`, `u` for `uint`, `ul` for `ulong`. Underscores `1_000_000` as a digit-grouping separator are mandatory for large numbers.
- Financial calculations (price, VAT, total) are performed strictly in `decimal`; the `double` comparison is shown only as a negative example in a separate block with a comment.
- `double` equality is implemented via an epsilon method (`1e-9` or smaller), not via `==`. The code must contrast `dirtySum == 0.3` → `false` with `ApproxEquals(dirtySum, 0.3)` → `true`.
- `char` is written in single quotes and at least one character is given via the Unicode escape `\uXXXX`; `string` is in double quotes and at least one string is a raw literal `"""..."""`.
- `bool` is used in `if` without any numeric cast; the comment notes C#'s protection against `if (x = 5)`.
- Type-based pattern matching in `switch` covers at least 6 variants (`int`, `decimal`, `double`, `bool`, `string`, `char`) plus `_`; the `int` branch has a `when i < 0` guard.
- The code compiles without warnings like unused locals (CS0168/CS0219 family) and without literal-overflow errors; the output matches the description.
- All code comments are bilingual RU+EN, as in the lesson example.

#### Pitfalls
- **The `L` suffix is mandatory** for `long` literals exceeding `int.MaxValue`. `long big = 5_000_000_000;` without `L` is a compile error: the compiler first tries to fit the literal into `int` and fails with an overflow before it ever reaches the implicit widening to `long`. Write `5_000_000_000L`. Likewise `uint` needs `u`, `ulong` needs `ul`, `decimal` needs `m`, `float` needs `f`.
- **`decimal d = 1.5;` without `m` is an error**, because the literal `1.5` defaults to `double`, and there is no implicit narrowing conversion `double` → `decimal` (it could lose precision). Always write `1.5m`.
- **`0.1 + 0.2 != 0.3` in `double`** is not a bug but a property of the binary IEEE 754 form: `0.1` and `0.2` are not exactly representable as binary fractions. For money this is critical, so `decimal` (decimal base-10 form) yields exactly `0.3m`. Compare `double` with `Math.Abs(a - b) < eps`, never with bare `==`.
- **`char` vs `string`:** `char c = "A";` is a compile error; `char` uses single quotes `'A'`, `string` uses double quotes `"A"`. A currency symbol can be written directly (`'₽'`) or via an escape (`'\u20BD'` — ruble sign, `'\u20AC'` — €).
- **`string` is immutable:** any "modification" (`s += "x"`) creates a new object. In hot loops use `StringBuilder` (bonus). Raw string literals `"""..."""` remove the need to escape quotes and backslashes — ideal for JSON/SQL/regex.
- **`bool` and `if (x = 5)`:** in C/C++ this is assignment; in C# it is a compile error, because `if` requires `bool` and `int` does not convert to `bool`. Still, watch `==` in conditions.
- **Unsigned types (`uint`, `ulong`)** are handy for counters but avoided in public APIs because of CLS cross-language interop headaches. Inside a module they are fine.
- **Formatting:** `:C` is currency, `:N0` is a number with group separators and no fraction, `:F2` is two decimal places. For `decimal`, `:C` gives a correct money representation.

#### Acceptance criteria
- [ ] Project `src/Shop` created via `dotnet new console -f net8.0`; `.csproj` specifies `net8.0`.
- [ ] `Program.cs` uses C# 12 top-level statements, with no explicit `class Program`.
- [ ] The integer block contains `int`, `long` (with `L`), `byte`, `uint` (with `u`), `ulong` (with `ul`), with `_` separators.
- [ ] The `double` block demonstrates `0.1 + 0.2 != 0.3` and an `ApproxEquals` method with epsilon.
- [ ] The financial block uses `decimal` with the `m` suffix; price, VAT, and total are printed with `:C`.
- [ ] There is an explicit negative example of the same calculation in `double` with a comment about drift.
- [ ] `bool` is used in `if` without a numeric cast; there is a comment about the `if (x = 5)` protection.
- [ ] `char` is in single quotes, with at least one character via `\uXXXX`.
- [ ] `string` includes a raw string literal `"""..."""` (JSON receipt).
- [ ] Pattern matching `switch` covers at least 6 types + `_`, with `when i < 0` for `int`.
- [ ] `dotnet build src/Shop` succeeds with no errors and no unused-variable warnings.
- [ ] `dotnet run --project src/Shop` prints the expected lines: coordinates, the `0.1+0.2` warning, the exact financial total, currency symbols, JSON, type descriptions.
- [ ] All code comments are bilingual RU+EN.
- [ ] The file has no TODOs, stubs, or commented-out dead code.

#### Hints (no direct answer)
- Recall the lesson rule: "money in `decimal`, physics in `double`, counters in `int`". Decide which category each step belongs to.
- For epsilon use `Math.Abs(a - b) < 1e-9`. Why does `==` fail with `double`? Because the binary IEEE 754 form does not represent `0.1` exactly.
- A raw string literal starts and ends with three double quotes `"""`; inside, you do not need to escape `"`.
- For pattern matching by type use the syntax `sample switch { int i => ..., decimal d => ..., _ => ... }`. A guard is `int i when i < 0 => ...`.
- If the compiler complains about a `long` literal — add `L`. If it complains about `decimal` without `m` — add `m`. These are the most frequent junior mistakes from the lesson.

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8 — top-level statements
// Homework M02-L02: "Coffee Grounds" shop

using System;
using System.Globalization;
using System.Text;

// Invariant culture so :C does not depend on the demo machine locale
CultureInfo.CurrentCulture = CultureInfo.InvariantCulture;

// --- 1. Integers ---
int itemsInStock = 12_500;              // warehouse stock
long worldCustomers = 8_100_000_000L;   // L suffix required
byte roastLevel = 3;                    // 0..10, roast
uint loyaltyPoints = 4_000_000_000u;    // u suffix
ulong totalBeansSold = 10_000_000_000ul;// ul suffix

Console.WriteLine($"itemsInStock={itemsInStock:N0} (int)");
Console.WriteLine($"worldCustomers={worldCustomers:N0} (long, L-suffix)");
Console.WriteLine($"roastLevel={roastLevel} (byte)");
Console.WriteLine($"loyaltyPoints={loyaltyPoints:N0} (uint, u-suffix)");
Console.WriteLine($"totalBeansSold={totalBeansSold:N0} (ulong, ul-suffix)");

// --- 2. Floating point ---
double latitude = 55.7558;              // physics — double
double longitude = 37.6173;
float brewingTemp = 92.5f;              // f suffix

double dirtySum = 0.1 + 0.2;            // NOT exactly 0.3
Console.WriteLine($"latitude={latitude}, longitude={longitude}, temp={brewingTemp}f");
Console.WriteLine($"0.1 + 0.2 = {dirtySum:G17}  (expected ~0.30000000000000004)");
Console.WriteLine($"  dirtySum == 0.3 ? {dirtySum == 0.3}        // bare == gives false");
Console.WriteLine($"  ApproxEquals(0.1+0.2, 0.3) ? {ApproxEquals(dirtySum, 0.3)}  // via epsilon");

// --- 3. decimal for money ---
decimal price = 19.99m;                 // m suffix required
decimal vatRate = 0.20m;                // VAT 20%
decimal vat = price * vatRate;
decimal total = price + vat;            // exact to the cent

Console.WriteLine($"price={price:C}, vat={vat:C}, total={total:C}");

// Negative example: same calc in double
double dPrice = 19.99;
double dTotal = dPrice * 1.20;
Console.WriteLine($"double total = {dTotal:G17}  (IEEE 754 drift)");

// --- 4. bool / logical ---
bool isPaid = true;
bool isMember = false;
bool hasDiscount = isMember && !isPaid; // false since isMember=false
if (isPaid)
{
    Console.WriteLine("Order paid");
}
// In C#, if requires bool, so if (x = 5) won't compile — typo protection

// --- 5. char ---
char currency = '₽';                    // Unicode UTF-16 direct
char euro = '\u20AC';                   // € via escape
Console.WriteLine($"currency={currency}, euro={euro}");

// --- 6. string + raw string literal ---
string greeting = "Welcome to Coffee Grounds!";
string receiptJson = """
{
  "order": "Latte 1kg",
  "price": 19.99,
  "vat": 3.998,
  "total": 23.988
}
""";
Console.WriteLine(greeting);
Console.WriteLine(receiptJson);

// string is immutable — concatenation creates new objects
string receipt = "Receipt: ";
receipt += $"Total {total:C}";          // new object
Console.WriteLine(receipt);

// --- 7. Type pattern matching ---
foreach (object sample in new object[] { 42, -7, 19.99m, 55.7558, true, "Latte", '€' })
{
    string description = sample switch
    {
        int i when i < 0 => $"Negative integer: {i}",
        int i            => $"Integer: {i}",
        decimal d        => $"Money: {d:C}",
        double dd        => $"Double: {dd:G17}",
        bool b           => $"Boolean: {b}",
        string s         => $"String: {s}",
        char c           => $"Char: {c}",
        _                => "Unknown"
    };
    Console.WriteLine(description);
}

// Local function: double equality via epsilon
static bool ApproxEquals(double a, double b, double eps = 1e-9) =>
    Math.Abs(a - b) < eps;
```

Line-by-line walk-through. The `using System.Globalization;` line and the `CultureInfo.CurrentCulture = CultureInfo.InvariantCulture;` assignment are needed so the `:C` format prints currency predictably instead of depending on the demo machine's locale — a common pitfall when grading homework across different operating systems. The integer block restates the lesson rule: `int` for counters, `long` with the `L` suffix for large values (without `L` the literal `8_100_000_000` does not fit in `int` and causes a compile error), `byte` for small ranges, `uint` with `u` and `ulong` with `ul` for unsigned counters. The underscores in `12_500` are the legal digit-grouping separator from the lesson, improving readability. The `double` block shows the essence of IEEE 754: `0.1 + 0.2` yields `0.30000000000000004`, so a bare `==` returns `false` while the `ApproxEquals` method with epsilon `1e-9` returns `true`; the `:G17` format prints all significant digits so the drift is visible. The financial block is the core of the homework: `decimal price = 19.99m;` with the mandatory `m` suffix (without it the literal is `double` and there is no implicit conversion), `decimal vat = price * vatRate;` and `decimal total = price + vat;` give an exact result with no drift accumulation because `decimal` is stored in base-10 form. The negative `double` example sits next to it so the student sees the difference. The `bool` block uses `if (isPaid)` with no numeric cast; the comment records that C# forbids `if (x = 5)`, unlike C/C++. The `char` block shows both a direct symbol `'₽'` and the `\u20AC` escape for € — both forms from the lesson. The `string` block includes a raw literal `"""..."""` for JSON without escaping quotes, plus a concatenation with a comment about immutability. The pattern-matching `switch` covers six types plus `_`, with a `when i < 0` guard for negative `int` — exactly like the lesson example. The local function `ApproxEquals` is declared at the bottom, which is allowed in top-level statements. Applied lesson concepts: choosing the type by the nature of the value, literal suffixes, IEEE 754 and epsilon, `decimal` for finance, `char` vs `string`, `string` immutability, raw string literals, type pattern matching.

#### Going deeper (bonus)
1. Replace string concatenation in a loop (build a 1000-line receipt) with `StringBuilder` and measure the time difference with `Stopwatch`. Explain why `string +=` is slow in a hot loop because of immutability.
2. Add a `float` with the `f` suffix and show that `0.1f + 0.2f` is also not equal to `0.3f`, but the drift is larger than for `double`. Compare the number of significant digits of `float` (≈7) and `double` (≈15–17).
3. Implement a method `decimal RoundToCents(decimal value)` that rounds to cents via `decimal.Round(value, 2, MidpointRounding.ToEven)`, and explain why banker's rounding (`ToEven`) reduces systematic error in financial reporting.
4. Use C# 12 collection expressions (`int[] xs = [1, 2, 3];`) for a list of orders and apply type pattern matching to `object[]` elements assembled via a collection expression.

---

#### Чек-лист сдачи / Submission checklist
- [ ] Проект `src/Shop` создан, собирается и запускается на .NET 8 / C# 12. / Project `src/Shop` created, builds, and runs on .NET 8 / C# 12.
- [ ] Использованы top-level statements, без `class Program`. / Top-level statements used, no `class Program`.
- [ ] Все числовые литералы несут нужные суффиксы (`L`, `f`, `m`, `u`, `ul`). / All numeric literals carry the required suffixes (`L`, `f`, `m`, `u`, `ul`).
- [ ] Финансы в `decimal`, есть негативный пример в `double`. / Finance in `decimal`, with a `double` negative example.
- [ ] `double` сравнивается через epsilon, не через `==`. / `double` compared via epsilon, not `==`.
- [ ] `char` в одинарных кавычках, минимум один через `\uXXXX`. / `char` in single quotes, at least one via `\uXXXX`.
- [ ] Есть сырой строковый литерал `"""..."""`. / A raw string literal `"""..."""` is present.
- [ ] Pattern matching покрывает ≥6 типов + `_`, с `when i < 0`. / Pattern matching covers ≥6 types + `_`, with `when i < 0`.
- [ ] `dotnet build` без ошибок и предупреждений о неиспользуемых переменных. / `dotnet build` succeeds with no errors or unused-variable warnings.
- [ ] Вывод соответствует описанию; комментарии двуязычные. / Output matches the description; comments are bilingual.
- [ ] Нет TODO, заглушек, закомментированного нерабочего кода. / No TODOs, stubs, or commented-out dead code.

#### Ресурсы / Resources
- [Microsoft Learn — Встроенные типы C# / C# built-in types — https://learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/built-in-types](https://learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/built-in-types)
- [Microsoft Learn — System.Decimal для финансовых расчётов / System.Decimal for financial calculations — https://learn.microsoft.com/dotnet/api/system.decimal](https://learn.microsoft.com/dotnet/api/system.decimal)
- [Microsoft Learn — Числовые типы с плавающей запятой / Floating-point numeric types — https://learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/floating-point-numeric-types](https://learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/floating-point-numeric-types)
- [Microsoft Learn — Сырые строковые литералы / Raw string literals — https://learn.microsoft.com/dotnet/csharp/programming-guide/strings/#raw-string-literals](https://learn.microsoft.com/dotnet/csharp/programming-guide/strings/)
- [Microsoft Learn — Pattern matching / Сопоставление шаблонов — https://learn.microsoft.com/dotnet/csharp/fundamentals/functional/pattern-matching](https://learn.microsoft.com/dotnet/csharp/fundamentals/functional/pattern-matching)
- [Microsoft Learn — Collection expressions (C# 12) / Выражения коллекций — https://learn.microsoft.com/dotnet/csharp/language-reference/operators/collection-expressions](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/collection-expressions)

---
[← К уроку M02-L02](lesson-M02-L02-primitives.md) | [⬆ К модулю M02](../README.md) | [Следующее ДЗ →](homework-M02-L03-var-const.md)
