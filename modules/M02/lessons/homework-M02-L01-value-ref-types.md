---
[← К уроку M02-L01](lesson-M02-L01-value-ref-types.md) | [⬆ К модулю M02](../README.md) | [Следующее ДЗ →](homework-M02-L02-primitives.md)
---

### Домашнее задание M02-L01: Значимые и ссылочные типы, stack vs heap / Homework M02-L01: Value vs Reference Types, Stack vs Heap

**Урок / Lesson:** M02-L01
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) На практике закрепить разницу между значимыми и ссылочными типами, научиться предсказывать, где окажутся данные (stack или heap), отличать копию значения от копии ссылки, осознанно избегать boxing, выбирать `struct`/`record struct`/`class` по характеристикам данных и корректно применять `in`/`ref`/`with`. (EN) Practice the distinction between value and reference types, predict where data lives (stack vs heap), tell a copy of a value from a copy of a reference, deliberately avoid boxing, choose `struct`/`record struct`/`class` based on data characteristics, and apply `in`/`ref`/`with` correctly.

#### Связь с уроком / Connection to the lesson
(RU) ДЗ напрямую опирается на теорию урока: побитовое копирование значимых типов против копирования ссылки для ссылочных, аналогию «ценная бумага vs адрес сейфа», поведение `string` как неизменяемого ссылочного типа, дороговизну boxing/unboxing и best practices (`readonly struct`, `record struct`, `in`-параметры, `with`-выражения). Оно также воспроизводит классические ошибки из раздела «Частые ошибки»: изменяемый `struct` в `List<T>`, неявный boxing в цикле, путаница `ReferenceEquals` vs `==` для строк.
(EN) The homework builds directly on the lesson: bitwise copying of value types vs reference copying for reference types, the "bearer bond vs safe address" analogy, `string` as an immutable reference type, the cost of boxing/unboxing, and best practices (`readonly struct`, `record struct`, `in` parameters, `with` expressions). It also reproduces the classic mistakes from the "Common Mistakes" section: a mutable `struct` inside `List<T>`, implicit boxing in a loop, and confusing `ReferenceEquals` with `==` for strings.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Вы присоединяетесь к команде, разрабатывающей учебный платёжный модуль на C# 12 / .NET 8. В коде уже есть наброски, но они содержат типичные ошибки, связанные с непониманием семантики значимых и ссылочных типов: где-то структура копируется при каждом присваивании и «теряет» изменения, где-то ссылочные объекты случайно разделяют состояние, где-то в горячем цикле происходит скрытый boxing, раздувающий нагрузку на GC, а где-то строки сравниваются через `ReferenceEquals` и логика ломается на интернированных литералах. Ваша задача — не просто переписать код, а сделать поведение памяти предсказуемым: для каждой переменной вы должны уметь ответить, лежат ли данные в стеке или в куче, копируется ли значение или ссылка, и почему именно так.

Платёжный модуль работает с двумя ключевыми сущностями. Первая — `Money` (деньги): сумма и валюта, небольшой и логически цельный кусок данных, который выгодно сделать значимым типом, чтобы копирование было дешёвым, а сравнение — по значению. Вторая — `BankAccount` (банковский счёт): объект с идентичностью, владельцем и изменяемым балансом, который естественно моделировать ссылочным типом, потому что несколько переменных могут ссылаться на один и тот же счёт и видеть общие изменения. На этой паре сущностей легко увидеть обе категории типов в действии и сравнить их поведение.

Дополнительно модуль должен уметь описывать произвольные значения через единый `Describe(object)` — а это прямой путь к boxing и pattern matching, где нужно отличать значимые типы (`int`, `Money`) от ссылочных (`string`, `BankAccount`). Наконец, модуль должен корректно работать с большой структурой `BigAmount` (например, фиксированная точка с 16 полями), для которой побитовое копирование в горячем методе недопустимо — здесь пригодится `in`-параметр. Урок даёт всю теорию; ДЗ превращает её в работающий, осмысленный код.

#### Что нужно сделать (пошагово)
1. Создайте консольный проект .NET 8 с именем `Payments.M02L01` командой `dotnet new console -n Payments.M02L01 -f net8.0`. Убедитесь, что в `Payments.M02L01.csproj` установлены `<LangVersion>latest</LangVersion>` и `<Nullable>enable</Nullable>` (последнее — обязательно, чтобы nullable-аннотации компилировались). Перейдите в папку проекта и запустите `dotnet build` — он должен пройти без ошибок.

2. В файле `Program.cs` (top-level statements) опишите значимый тип `Money` как `readonly record struct Money(decimal Amount, string Currency)` — именно `record struct`, чтобы получить авто-генерируемые `Equals`/`GetHashCode`, оператор `==` и поддержку `with`-выражений. Добавьте метод `Add(Money other)`, который складывает суммы только при совпадении валюты и через `with` возвращает новую копию, иначе бросает `InvalidOperationException` с понятным сообщением. Учтите, что `string Currency` — ссылочный тип внутри значимого: это допустимо, но при копировании `Money` копируется ссылка на строку, а не сама строка (строка неизменяема, поэтому безопасно).

3. Опишите ссылочный тип `BankAccount` как `class BankAccount` со свойствами `Owner` (строка) и `Balance` (тип `Money`, приватный сеттер). Конструктор принимает владельца и начальный баланс. Метод `Deposit(Money amount)` делегирует `Balance.Add(amount)`. Помните: `Balance` — значимое поле внутри ссылочного объекта, оно лежит inline в куче вместе с объектом, а не в стеке.

4. В `Main` (top-level) создайте переменную `var a = new Money(100m, "USD");`, затем `var b = a;`. Выведите `a.Amount` и `b.Amount` — ожидается `100, 100`. Выполните `b = b with { Amount = 200m };` и снова выведите — ожидается `a=100, b=200`. Поясните в комментарии, что `with` создаёт копию и не трогает `a`.

5. Создайте `var acc1 = new BankAccount("Alice", new Money(50m, "USD"));`, затем `var acc2 = acc1;`. Вызовите `acc2.Deposit(new Money(30m, "USD"));` и выведите `acc1.Balance.Amount` — ожидается `80`, потому что `acc1` и `acc2` — одна ссылка. Добавьте комментарий о том, что здесь проявляется ссылочная семантика: мутация через одну переменную видна через другую.

6. Демонстрация изменяемого `struct` в `List<T>`: создайте локальный `record struct Point(int X, int Y);`, заполните список `var pts = new List<Point> { new(1, 1) };` и попробуйте изменить `pts[0].X = 5;` — компилятор выдаст ошибку CS1612, потому что индексатор возвращает копию. Обойдите ошибку правильно: `pts[0] = pts[0] with { X = 5 };`. Выведите результат. Это воспроизводит классическую ошибку из урока.

7. Boxing: создайте `int n = 42;`, затем `object boxed = n;` (boxing) и `int unboxed = (int)boxed;` (unboxing). Выведите оба значения. Затем намеренно создайте неявный boxing в цикле: `foreach (object o in new[] { 1, 2, 3 }) { /* ... */ }` — и тут же исправьте его на `foreach (int i in new[] { 1, 2, 3 })`. В комментарии объясните, почему вторая версия не аллоцирует.

8. String: создайте `string s1 = "hello";`, затем `string s2 = s1;`, затем `s2 = s2.ToUpper();`. Выведите `s1` и `s2` — ожидается `hello` и `HELLO`. Покажите сравнение `ReferenceEquals(s1, "hello")` (часто `true` из-за интернирования) и `s1 == "hello"` (всегда `true` по содержимому). Поясните в комментарии, что `==` для `string` перегружен на сравнение значений, а `ReferenceEquals` — на сравнение ссылок.

9. Pattern matching: напишите `string Describe(object obj) => obj switch { ... }`, различающий `int i when i > 0`, `string s`, `Money m`, `BankAccount { Owner: var o }`, `null`, `_`. Вызовите `Describe` для `n`, `s1`, `a`, `acc1`, `null` и выведите результаты. Обратите внимание, что передача `a` (значимого `Money`) в `Describe(object)` — это boxing; отметьте это в комментарии.

10. `in`-параметр для большой структуры: опишите `readonly record struct BigAmount(long Units, long Nano, long Checksum, long Reserved, long Reserved2, long Reserved3, long Reserved4, long Reserved5);` и метод `long HashBig(in BigAmount ba) => ba.Units ^ ba.Nano ^ ba.Checksum;`. Вызовите его и поясните, что `in` передаёт ссылку только для чтения и не копирует структуру.

11. Запустите приложение командой `dotnet run` и убедитесь, что вывод совпадает с ожидаемым. Сохраните вывод в `output.txt` (можно просто перенаправлением `dotnet run > output.txt` или копированием).

#### Требования к решению
- Целевая платформа — .NET 8, язык C# 12 (`<LangVersion>latest</LangVersion>`), nullable-контекст включён. Используйте top-level statements в `Program.cs`.
- `Money` — `readonly record struct` с `with`-выражением в `Add`; `BankAccount` — `class` с приватным сеттером у `Balance`.
- В коде должны быть физически представлены все явления из урока: побитовое копирование значимого типа, копирование ссылки для ссылочного, boxing и unboxing, исправленный неявный boxing в цикле, immutable `string` с `ToUpper`, исправление ошибки CS1612 в `List<T>`, pattern matching в `Describe`, `in`-параметр для большой структуры.
- Каждое ключевое место должно сопровождаться комментарием на русском и английском (двуязычные комментарии, как в эталонном решении), объясняющим, какое поведение памяти здесь демонстрируется.
- Код должен компилироваться без предупреждений уровня error и запускаться. Ожидаемые числовые выводы должны совпадать: `a=100, b=200`, `acc1=80`, `s1=hello, s2=HELLO`, и т.д.
- Запрещено использовать `ArrayList` или нетипизированные коллекции там, где есть дженерики: `List<int>`, а не `ArrayList`. Это воспроизводит best practice из урока про избегание boxing.
- Нельзя «чинить» CS1612 через замену `struct` на `class` — нужно сохранить значимый тип и применить `with`, как учит урок.

#### Тонкости и подводные камни
- `string Currency` внутри `Money` — это ссылочный тип, вложенный в значимый. При копировании `Money` копируется ссылка на строку, а сама строка не дублируется. Это безопасно, потому что строки неизменяемы, но важно понимать: `Money` не «полностью» побайтово независим от копии — строковое поле у них общее.
- `with`-выражение создаёт **новую** копию; исходная переменная не меняется. Начинающие часто ожидают, что `b with { Amount = 200 }` обновит `b` — это не так, нужно присваивание `b = b with { ... }`.
- Индексатор `List<T>` для значимого типа возвращает **копию** элемента. Попытка `pts[0].X = 5;` меняет временную копию и поэтому запрещена компилятором (CS1612). Правильный путь — заменить элемент целиком через `with`.
- Неявный boxing в `foreach (object o in intArray)` аллоцирует объект-обёртку для каждого `int` — в горячем цикле это давит на GC. Типизация цикла как `foreach (int i in intArray)` убирает аллокации полностью. Это лучшая иллюстрация best practice «дженерики вместо boxing».
- `ReferenceEquals(s1, "hello")` может вернуть `true` из-за интернирования литералов, но это деталь реализации, на которую нельзя полагаться в логике. Сравнивать строки по содержимому нужно через `==` или `string.Equals`.
- Передача значимого `Money` в `Describe(object)` вызывает boxing — это плата за полиморфизм через `object`. В реальном коде для конкретных типов лучше дженерики или перегрузки; здесь boxing оставлен намеренно для демонстрации.
- `in`-параметр передаёт структуру по ссылке только для чтения: внутри метода нельзя мутировать поля, но и копия не создаётся. Это спасает для больших структур; для маленьких `readonly record struct` копирование дешевле, и `in` может даже замедлить код из-за лишней косвенности — выбирайте осознанно.
- `decimal` и `DateTime` — значимые типы; в кучу они попадают только при boxing или как поля class-объекта. Не путайте «ссылочный тип» с «живёт в куче» — стек/куча определяется контекстом размещения, а не только категорией типа.
- `record struct` по умолчанию не `readonly`; для иммутабельной семантики явно указывайте `readonly record struct`. Иначе можно случайно создать изменяемый value-тип со всеми вытекающими.

#### Критерии приёмки
- [ ] Проект `Payments.M02L01` создаётся командой `dotnet new console` и собирается без ошибок на .NET 8 / C# 12.
- [ ] `Money` объявлен как `readonly record struct` с методом `Add`, использующим `with`.
- [ ] `BankAccount` объявлен как `class` с приватным сеттером `Balance`.
- [ ] Вывод `a=100, b=200` воспроизведён и пояснён комментарием о копии значения.
- [ ] Вывод `acc1=80` воспроизведён и пояснён комментарием об общей ссылке.
- [ ] Демонстрируется ошибка CS1612 и её исправление через `pts[0] = pts[0] with { X = 5 };`.
- [ ] Boxing и unboxing показаны явно (`object boxed = n; (int)boxed`).
- [ ] Неявный boxing в `foreach (object ...)` показан и исправлен на `foreach (int ...)`.
- [ ] Демонстрируется immutable `string` (`ToUpper` создаёт новый объект).
- [ ] Различаются `ReferenceEquals` и `==` для строк, оба вызова выведены.
- [ ] `Describe(object)` реализован через pattern matching и покрывает `int`, `string`, `Money`, `BankAccount`, `null`, default.
- [ ] В комментариях отмечено, что передача `Money` в `Describe(object)` — это boxing.
- [ ] Большая структура `BigAmount` передаётся в метод через `in`, копирования нет.
- [ ] Все ключевые места снабжены двуязычными (RU+EN) комментариями.
- [ ] `dotnet run` выводит ожидаемые значения, вывод сохранён в `output.txt`.
- [ ] В коде нет `ArrayList` и нет «обхода» CS1612 через замену `struct` на `class`.

#### Подсказки (без прямого ответа)
- Вспомните аналогию из урока: «ценная бумага» копируется целиком, «адрес сейфа» разделяется. Какая из сущностей (`Money`, `BankAccount`) — какая?
- Для `with`-выражения тип должен быть `record` (включая `record struct`). Убина `readonly record struct` это поддерживает «из коробки».
- CS1612 — это не баг компилятора, а защита: индексатор возвращает копию value-типа. Подумайте, как вернуть новый элемент в список целиком.
- `object` — это ссылочный тип. Любой value-тип, приведённый к `object`, боксится. Где в `Describe` это происходит?
- `in` означает «ссылка только для чтения». Что произойдёт, если внутри метода попытаться изменить поле `in`-параметра?
- Строки интернируются: одинаковые литералы могут быть одним объектом. Почему `==` надёжнее `ReferenceEquals` для строк?

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 — Payments.M02L01 / Homework M02-L01
// Демонстрация значимых и ссылочных типов, stack vs heap, boxing, immutable string
// Demonstrates value vs reference types, stack vs heap, boxing, immutable string

using System;
using System.Collections.Generic;

// Значимый тип: record struct, with-expressions, авто-Equals/GetHashCode
// Value type: record struct, with-expressions, auto-Equals/GetHashCode
public readonly record struct Money(decimal Amount, string Currency)
{
    // with создаёт КОПИЮ с новым Amount — исходный Money не меняется
    // with creates a COPY with a new Amount — the original Money is untouched
    public Money Add(Money other) =>
        Currency == other.Currency
            ? this with { Amount = Amount + other.Amount }
            : throw new InvalidOperationException(
                $"Валюты не совпадают / Currency mismatch: {Currency} vs {other.Currency}");
}

// Ссылочный тип: несколько переменных могут указывать на один объект
// Reference type: several variables may point to the same object
public class BankAccount
{
    public string Owner { get; set; }
    // Balance — ЗНАЧИМОЕ поле внутри ссылочного объекта: лежит inline в куче
    // Balance is a VALUE field inside a reference object: stored inline on the heap
    public Money Balance { get; private set; }

    public BankAccount(string owner, Money initial) =>
        (Owner, Balance) = (owner, initial);

    public void Deposit(Money amount) => Balance = Balance.Add(amount);
}

// Большая структура: копировать дорого — передаём по ссылке через in
// Large struct: expensive to copy — pass by reference via in
public readonly record struct BigAmount(
    long Units, long Nano, long Checksum,
    long Reserved1, long Reserved2, long Reserved3, long Reserved4, long Reserved5);

// Top-level Main
var a = new Money(100m, "USD");           // value-переменная / value var
var b = a;                                // КОПИЯ значения / COPY of value
b = b with { Amount = 200m };             // with: новая копия / new copy
Console.WriteLine($"a={a.Amount}, b={b.Amount}");   // a=100, b=200

var acc1 = new BankAccount("Alice", new Money(50m, "USD"));
var acc2 = acc1;                          // КОПИЯ ССЫЛКИ: тот же объект / COPY of reference: same object
acc2.Deposit(new Money(30m, "USD"));
Console.WriteLine($"acc1={acc1.Balance.Amount}");    // 80 — видно через acc1 / visible via acc1

// Ошибка CS1612 и её правильное исправление через with
// CS1612 error and its correct fix via with
var pts = new List<Point> { new(1, 1) };
// pts[0].X = 5;  // CS1612: нельзя мутировать копию, возвращённую индексатором
pts[0] = pts[0] with { X = 5 };           // заменяем элемент целиком / replace the element wholesale
Console.WriteLine($"pts[0]={pts[0]}");    // Point { X = 5, Y = 1 }

// Boxing / unboxing
int n = 42;
object boxed = n;                         // BOXING: аллокация в куче / heap allocation
int unboxed = (int)boxed;                 // UNBOXING: извлечение / extract value
Console.WriteLine($"boxed={boxed}, unboxed={unboxed}");

// Неявный boxing в цикле — исправляем типизацией foreach
// Implicit boxing in a loop — fix by typing foreach
// foreach (object o in new[] { 1, 2, 3 }) { ... }  // BAD: боксит каждый int
foreach (int i in new[] { 1, 2, 3 })      // GOOD: без аллокаций / no allocations
    Console.WriteLine($"i={i}");

// String: ссылочный, но immutable / reference-typed but immutable
string s1 = "hello";
string s2 = s1;                           // та же ссылка на интернированный литерал / shared interned ref
s2 = s2.ToUpper();                        // создаётся НОВЫЙ объект / creates a NEW object
Console.WriteLine($"s1={s1}, s2={s2}");   // s1=hello, s2=HELLO
Console.WriteLine($"ReferenceEquals(s1,\"hello\")={ReferenceEquals(s1, "hello")}"); // часто true
Console.WriteLine($"s1 == \"hello\" = {s1 == "hello"}");                              // всегда true

// Pattern matching: различаем value vs reference / distinguish value vs reference
string Describe(object obj) => obj switch
{
    int i when i > 0        => $"Положительное целое: {i} (boxed value)",
    string s                => $"Строка длины {s.Length}: «{s}»",
    Money m                 => $"{m.Amount} {m.Currency} (struct, boxed)",
    BankAccount { Owner: var o } => $"Счёт владельца {o}",
    null                    => "null",
    _                       => $"Другое: {obj}"
};

// Передача a (Money) в Describe(object) — это BOXING
// Passing a (Money) to Describe(object) is BOXING
Console.WriteLine(Describe(n));      // int
Console.WriteLine(Describe(s1));     // string
Console.WriteLine(Describe(a));      // Money (boxed)
Console.WriteLine(Describe(acc1));   // BankAccount
Console.WriteLine(Describe(null));   // null

// in-параметр для большой структуры: без копирования
// in-parameter for a large struct: no copy
long HashBig(in BigAmount ba) => ba.Units ^ ba.Nano ^ ba.Checksum;
var big = new BigAmount(1, 2, 3, 0, 0, 0, 0, 0);
Console.WriteLine($"HashBig={HashBig(in big)}");   // 0 (1^2^3 == 0)

// Локальный record struct для демонстрации CS1612 / local record struct for CS1612
public readonly record struct Point(int X, int Y);
```

Разбор по строкам. `readonly record struct Money` — это значимый тип с иммутабельной семантикой: копирование побитовое, а `with`-выражение в `Add` создаёт новую копию с изменённым `Amount`, оставляя исходный `Money` нетронутым; это лучшая иллюстрация правила «значимый тип копируется целиком». `BankAccount` — ссылочный тип: `acc2 = acc1` копирует ссылку, а не объект, поэтому `Deposit` через `acc2` виден через `acc1` — это аналогия «адрес сейфа» из урока. Поле `Balance` значимого типа `Money` лежит inline в куче внутри объекта `BankAccount`, а не в стеке: это показывает, что стек/куча определяется контекстом, а не только категорией типа.

Блок с `List<Point>` воспроизводит классическую ошибку CS1612: индексатор возвращает копию value-типа, и мутировать её поле бессмысленно — компилятор это запрещает. Правильное решение из урока — заменить элемент целиком через `with`: `pts[0] = pts[0] with { X = 5 }`. Блок boxing показывает как явное (`object boxed = n`), так и неявное (`foreach (object ...)`) приведение value-типа к `object`; исправление на `foreach (int ...)` убирает аллокации, демонстрируя best practice «дженерики вместо boxing».

Строковый блок иллюстрирует immutable-природу `string`: `ToUpper` создаёт новый объект, `s1` остаётся `"hello"`. Различие `ReferenceEquals` и `==` — ключевая тонкость урока: `==` перегружен для строк на сравнение по содержимому, а `ReferenceEquals` сравнивает ссылки и может давать `true` лишь случайно из-за интернирования. `Describe(object)` через pattern matching различает value- и reference-типы; важно, что передача `Money` в `Describe(object)` — это boxing, что отмечено комментарием. Наконец, `in`-параметр для `BigAmount` показывает осознанное избегание копирования большой структуры — ровно тот best practice, которому учит урок. Таким образом, эталонное решение покрывает все ключевые темы: копирование значения, копирование ссылки, stack/heap, boxing, immutable string, изменяемый struct в коллекции, pattern matching и `in`.

#### Задания на углубление (бонус)
1. Добавьте метод `Transfer(BankAccount from, BankAccount to, Money amount)`, который перекидывает деньги между счетами, и убедитесь, что два счёта — разные объекты (не один и тот же). Подумайте, что произойдёт, если случайно передать один и тот же объект в оба аргумента.
2. Измерьте количество аллокаций boxing через `GC.GetAllocatedBytesForCurrentThread()` до и после цикла `foreach (object ...)` vs `foreach (int ...)`. Выведите разницу и объясните её.
3. Реализуйте `readonly struct MoneyClassic` (без `record`) с ручными `Equals`/`GetHashCode` и сравните объём кода и поведение с `readonly record struct Money`. Покажите, что `with` без `record` недоступен.
4. Исследуйте интернирование строк: сравните `ReferenceEquals("ab"+"c", "abc")` с `ReferenceEquals(string.Intern(BuildAbc()), "abc")`, где `BuildAbc()` возвращает `"ab"+"c"` во время выполнения. Объясните, почему результаты могут отличаться.

---

## Statement in English / Постановка на английском

#### Context & motivation
You are joining a team building a small payments module on C# 12 / .NET 8. The existing sketch code contains the usual mistakes that come from not understanding value vs reference semantics: somewhere a struct is copied on every assignment and silently loses mutations, somewhere reference objects accidentally share state, somewhere a hot loop hides boxing that inflates GC pressure, and somewhere strings are compared with `ReferenceEquals` and logic breaks on interned literals. Your job is not just to rewrite the code — it is to make memory behavior predictable: for every variable you should be able to answer whether the data lives on the stack or the heap, whether a value or a reference is being copied, and why exactly that is the case.

The payments module works with two core entities. The first is `Money` — an amount and a currency, a small and logically cohesive chunk of data that is best modeled as a value type so that copying is cheap and equality is by value. The second is `BankAccount` — an object with identity, an owner, and a mutable balance, which naturally calls for a reference type, because several variables may refer to the same account and observe shared changes. On this pair of entities it is easy to see both categories of types in action and compare their behavior side by side.

Additionally, the module must be able to describe arbitrary values through a single `Describe(object)` entry point — a direct route into boxing and pattern matching, where you must distinguish value types (`int`, `Money`) from reference types (`string`, `BankAccount`). Finally, the module must correctly handle a large struct `BigAmount` (say, a fixed-point value with eight `long` fields) for which a bitwise copy in a hot method is unacceptable — this is where an `in` parameter earns its place. The lesson supplies the theory; this homework turns it into working, meaningful code.

#### What to do step by step
1. Create a .NET 8 console project named `Payments.M02L01` with `dotnet new console -n Payments.M02L01 -f net8.0`. Make sure `Payments.M02L01.csproj` sets `<LangVersion>latest</LangVersion>` and `<Nullable>enable</Nullable>` (nullable is required so the annotations compile). From the project folder, run `dotnet build` — it must succeed with no errors.

2. In `Program.cs` (top-level statements) declare a value type `Money` as `readonly record struct Money(decimal Amount, string Currency)` — specifically a `record struct` so that you get auto-generated `Equals`/`GetHashCode`, the `==` operator, and `with` expressions. Add a method `Add(Money other)` that adds amounts only when currencies match and returns a new copy via `with`, otherwise throws `InvalidOperationException` with a clear message. Note that `string Currency` is a reference type nested inside a value type: that is fine, but when `Money` is copied, the reference to the string is copied, not the string itself (the string is immutable, so this is safe).

3. Declare a reference type `BankAccount` as `class BankAccount` with properties `Owner` (string) and `Balance` (of type `Money`, private setter). The constructor takes the owner and the initial balance. The `Deposit(Money amount)` method delegates to `Balance.Add(amount)`. Remember: `Balance` is a value field inside a reference object — it lives inline on the heap together with the object, not on the stack.

4. In `Main` (top-level) create `var a = new Money(100m, "USD");`, then `var b = a;`. Print `a.Amount` and `b.Amount` — expect `100, 100`. Execute `b = b with { Amount = 200m };` and print again — expect `a=100, b=200`. Add a comment explaining that `with` makes a copy and never touches `a`.

5. Create `var acc1 = new BankAccount("Alice", new Money(50m, "USD"));`, then `var acc2 = acc1;`. Call `acc2.Deposit(new Money(30m, "USD"));` and print `acc1.Balance.Amount` — expect `80`, because `acc1` and `acc2` are one reference. Add a comment that this is reference semantics in action: a mutation through one variable is visible through the other.

6. Mutable struct in `List<T>` demo: declare a local `record struct Point(int X, int Y);`, fill a list `var pts = new List<Point> { new(1, 1) };` and try `pts[0].X = 5;` — the compiler emits CS1612 because the indexer returns a copy. Fix it the right way: `pts[0] = pts[0] with { X = 5 };`. Print the result. This reproduces the classic mistake from the lesson.

7. Boxing: create `int n = 42;`, then `object boxed = n;` (boxing) and `int unboxed = (int)boxed;` (unboxing). Print both. Then deliberately create implicit boxing in a loop: `foreach (object o in new[] { 1, 2, 3 }) { /* ... */ }` — and immediately fix it to `foreach (int i in new[] { 1, 2, 3 })`. In a comment explain why the second version allocates nothing.

8. String: create `string s1 = "hello";`, then `string s2 = s1;`, then `s2 = s2.ToUpper();`. Print `s1` and `s2` — expect `hello` and `HELLO`. Show the comparison `ReferenceEquals(s1, "hello")` (often `true` due to interning) and `s1 == "hello"` (always `true` by content). In a comment, explain that `==` for `string` is overloaded to compare values, while `ReferenceEquals` compares references.

9. Pattern matching: write `string Describe(object obj) => obj switch { ... }` distinguishing `int i when i > 0`, `string s`, `Money m`, `BankAccount { Owner: var o }`, `null`, `_`. Call `Describe` for `n`, `s1`, `a`, `acc1`, `null` and print the results. Note that passing `a` (a value `Money`) to `Describe(object)` is boxing — mark this in a comment.

10. `in` parameter for a large struct: declare `readonly record struct BigAmount(long Units, long Nano, long Checksum, long Reserved, long Reserved2, long Reserved3, long Reserved4, long Reserved5);` and a method `long HashBig(in BigAmount ba) => ba.Units ^ ba.Nano ^ ba.Checksum;`. Call it and explain that `in` passes a read-only reference and does not copy the struct.

11. Run the app with `dotnet run` and confirm the output matches expectations. Save the output to `output.txt` (either by redirection `dotnet run > output.txt` or by copy-paste).

#### Requirements
- Target .NET 8, language C# 12 (`<LangVersion>latest</LangVersion>`), nullable context enabled. Use top-level statements in `Program.cs`.
- `Money` is a `readonly record struct` with a `with` expression in `Add`; `BankAccount` is a `class` with a private setter on `Balance`.
- The code must physically demonstrate every phenomenon from the lesson: bitwise copy of a value type, reference copy for a reference type, boxing and unboxing, the fixed implicit boxing in a loop, immutable `string` with `ToUpper`, the CS1612 fix in `List<T>`, pattern matching in `Describe`, and an `in` parameter for a large struct.
- Each key spot must carry a bilingual (RU+EN) comment explaining which memory behavior is being demonstrated.
- The code must compile without error-level warnings and run. Expected numeric outputs must match: `a=100, b=200`, `acc1=80`, `s1=hello, s2=HELLO`, and so on.
- Do not use `ArrayList` or non-generic collections where generics exist: `List<int>`, not `ArrayList`. This mirrors the lesson's best practice of avoiding boxing.
- Do not "fix" CS1612 by replacing the `struct` with a `class` — keep the value type and use `with`, exactly as the lesson teaches.

#### Pitfalls
- `string Currency` inside `Money` is a reference type nested inside a value type. When `Money` is copied, the reference to the string is copied, not the string itself. This is safe because strings are immutable, but understand that `Money` is not "fully" byte-independent from its copy — the string field is shared.
- The `with` expression creates a **new** copy; the source variable is untouched. Beginners often expect `b with { Amount = 200 }` to mutate `b` — it does not; you need the assignment `b = b with { ... }`.
- The `List<T>` indexer for a value type returns a **copy** of the element. Trying `pts[0].X = 5;` mutates a temporary copy and is therefore rejected by the compiler (CS1612). The correct path is to replace the element wholesale via `with`.
- Implicit boxing in `foreach (object o in intArray)` allocates a wrapper object for each `int` — in a hot loop this pressures the GC. Typing the loop as `foreach (int i in intArray)` removes allocations entirely. This is the best illustration of the "generics over boxing" best practice.
- `ReferenceEquals(s1, "hello")` may return `true` because literals are interned, but this is an implementation detail you must not rely on in logic. Compare strings by content using `==` or `string.Equals`.
- Passing a value `Money` to `Describe(object)` triggers boxing — the price of polymorphism through `object`. In real code, prefer generics or overloads for concrete types; here boxing is kept on purpose for demonstration.
- An `in` parameter passes the struct by read-only reference: the method cannot mutate fields, and no copy is made. This saves large structs; for tiny `readonly record struct`s copying is cheaper and `in` may even slow the code down through extra indirection — choose deliberately.
- `decimal` and `DateTime` are value types; they reach the heap only via boxing or as fields of a class object. Do not conflate "reference type" with "lives on the heap" — stack vs heap is decided by placement context, not just by the type category.
- A `record struct` is not `readonly` by default; for immutable semantics, explicitly write `readonly record struct`. Otherwise you can accidentally create a mutable value type with all the usual pitfalls.

#### Acceptance criteria
- [ ] The project `Payments.M02L01` is created via `dotnet new console` and builds cleanly on .NET 8 / C# 12.
- [ ] `Money` is declared as `readonly record struct` with an `Add` method using `with`.
- [ ] `BankAccount` is declared as `class` with a private setter on `Balance`.
- [ ] The output `a=100, b=200` is reproduced and explained by a comment on value copy.
- [ ] The output `acc1=80` is reproduced and explained by a comment on shared reference.
- [ ] The CS1612 error is demonstrated and fixed via `pts[0] = pts[0] with { X = 5 };`.
- [ ] Boxing and unboxing are shown explicitly (`object boxed = n; (int)boxed`).
- [ ] Implicit boxing in `foreach (object ...)` is shown and fixed to `foreach (int ...)`.
- [ ] Immutable `string` is demonstrated (`ToUpper` creates a new object).
- [ ] `ReferenceEquals` and `==` for strings are distinguished and both printed.
- [ ] `Describe(object)` is implemented with pattern matching covering `int`, `string`, `Money`, `BankAccount`, `null`, default.
- [ ] A comment notes that passing `Money` to `Describe(object)` is boxing.
- [ ] The large struct `BigAmount` is passed to a method via `in`, with no copy.
- [ ] All key spots carry bilingual (RU+EN) comments.
- [ ] `dotnet run` prints the expected values, and the output is saved to `output.txt`.
- [ ] There is no `ArrayList` and no CS1612 "fix" that swaps `struct` for `class`.

#### Hints (no direct answer)
- Recall the lesson analogy: a "bearer bond" is copied wholesale, a "safe address" is shared. Which of `Money` and `BankAccount` is which?
- For a `with` expression the type must be a `record` (including `record struct`). A `readonly record struct` supports this out of the box.
- CS1612 is not a compiler bug — it is a safeguard: the indexer returns a copy of a value type. Think about how to put a whole new element back into the list.
- `object` is a reference type. Any value type cast to `object` gets boxed. Where in `Describe` does that happen?
- `in` means "read-only reference". What happens if you try to mutate a field of an `in` parameter inside the method?
- Strings are interned: identical literals may be the same object. Why is `==` safer than `ReferenceEquals` for strings?

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8 — Payments.M02L01 / Homework M02-L01
// Demonstrates value vs reference types, stack vs heap, boxing, immutable string

using System;
using System.Collections.Generic;

// Value type: record struct, with-expressions, auto-Equals/GetHashCode
public readonly record struct Money(decimal Amount, string Currency)
{
    // with creates a COPY with a new Amount — the original Money is untouched
    public Money Add(Money other) =>
        Currency == other.Currency
            ? this with { Amount = Amount + other.Amount }
            : throw new InvalidOperationException(
                $"Currency mismatch: {Currency} vs {other.Currency}");
}

// Reference type: several variables may point to the same object
public class BankAccount
{
    public string Owner { get; set; }
    // Balance is a VALUE field inside a reference object: stored inline on the heap
    public Money Balance { get; private set; }

    public BankAccount(string owner, Money initial) =>
        (Owner, Balance) = (owner, initial);

    public void Deposit(Money amount) => Balance = Balance.Add(amount);
}

// Large struct: expensive to copy — pass by reference via in
public readonly record struct BigAmount(
    long Units, long Nano, long Checksum,
    long Reserved1, long Reserved2, long Reserved3, long Reserved4, long Reserved5);

public readonly record struct Point(int X, int Y);

// Top-level Main
var a = new Money(100m, "USD");           // value var
var b = a;                                // COPY of value
b = b with { Amount = 200m };             // with: new copy
Console.WriteLine($"a={a.Amount}, b={b.Amount}");   // a=100, b=200

var acc1 = new BankAccount("Alice", new Money(50m, "USD"));
var acc2 = acc1;                          // COPY of reference: same object
acc2.Deposit(new Money(30m, "USD"));
Console.WriteLine($"acc1={acc1.Balance.Amount}");    // 80 — visible via acc1

// CS1612 error and its correct fix via with
var pts = new List<Point> { new(1, 1) };
// pts[0].X = 5;  // CS1612: cannot mutate a copy returned by the indexer
pts[0] = pts[0] with { X = 5 };           // replace the element wholesale
Console.WriteLine($"pts[0]={pts[0]}");    // Point { X = 5, Y = 1 }

// Boxing / unboxing
int n = 42;
object boxed = n;                         // BOXING: heap allocation
int unboxed = (int)boxed;                 // UNBOXING: extract value
Console.WriteLine($"boxed={boxed}, unboxed={unboxed}");

// Implicit boxing in a loop — fix by typing foreach
// foreach (object o in new[] { 1, 2, 3 }) { ... }  // BAD: boxes every int
foreach (int i in new[] { 1, 2, 3 })      // GOOD: no allocations
    Console.WriteLine($"i={i}");

// String: reference-typed but immutable
string s1 = "hello";
string s2 = s1;                           // shared interned ref
s2 = s2.ToUpper();                        // creates a NEW object
Console.WriteLine($"s1={s1}, s2={s2}");   // s1=hello, s2=HELLO
Console.WriteLine($"ReferenceEquals(s1,\"hello\")={ReferenceEquals(s1, "hello")}"); // often true
Console.WriteLine($"s1 == \"hello\" = {s1 == "hello"}");                              // always true

// Pattern matching: distinguish value vs reference
string Describe(object obj) => obj switch
{
    int i when i > 0            => $"Positive int: {i} (boxed value)",
    string s                    => $"String of length {s.Length}: \"{s}\"",
    Money m                     => $"{m.Amount} {m.Currency} (struct, boxed)",
    BankAccount { Owner: var o } => $"Account of owner {o}",
    null                        => "null",
    _                           => $"Other: {obj}"
};

// Passing a (Money) to Describe(object) is BOXING
Console.WriteLine(Describe(n));      // int
Console.WriteLine(Describe(s1));     // string
Console.WriteLine(Describe(a));      // Money (boxed)
Console.WriteLine(Describe(acc1));   // BankAccount
Console.WriteLine(Describe(null));   // null

// in-parameter for a large struct: no copy
long HashBig(in BigAmount ba) => ba.Units ^ ba.Nano ^ ba.Checksum;
var big = new BigAmount(1, 2, 3, 0, 0, 0, 0, 0);
Console.WriteLine($"HashBig={HashBig(in big)}");   // 0 (1^2^3 == 0)
```

Walk-through. `readonly record struct Money` is a value type with immutable semantics: copying is bitwise, and the `with` expression in `Add` produces a new copy with a changed `Amount`, leaving the original `Money` intact — the clearest illustration of the rule "a value type is copied wholesale." `BankAccount` is a reference type: `acc2 = acc1` copies the reference, not the object, so a `Deposit` through `acc2` is visible through `acc1` — the "safe address" analogy from the lesson. The `Balance` field of value type `Money` lives inline on the heap inside the `BankAccount` object, not on the stack: this shows that stack vs heap is decided by placement, not just by type category.

The `List<Point>` block reproduces the classic CS1612 mistake: the indexer returns a copy of a value type, so mutating its field is meaningless and the compiler forbids it. The lesson's correct fix is to replace the element wholesale with `with`: `pts[0] = pts[0] with { X = 5 }`. The boxing block shows both explicit (`object boxed = n`) and implicit (`foreach (object ...)`) casts of a value type to `object`; fixing the loop to `foreach (int ...)` removes allocations, demonstrating the "generics over boxing" best practice.

The string block illustrates the immutable nature of `string`: `ToUpper` creates a new object and `s1` stays `"hello"`. The distinction between `ReferenceEquals` and `==` is the key subtlety of the lesson: `==` is overloaded for strings to compare by content, while `ReferenceEquals` compares references and may return `true` only coincidentally because of interning. `Describe(object)` with pattern matching distinguishes value and reference types; importantly, passing `Money` to `Describe(object)` is boxing, which is called out in a comment. Finally, the `in` parameter for `BigAmount` shows deliberate avoidance of copying a large struct — precisely the best practice the lesson teaches. Thus the reference solution covers all the key topics: value copy, reference copy, stack/heap, boxing, immutable string, mutable struct in a collection, pattern matching, and `in`.

#### Going deeper (bonus)
1. Add a `Transfer(BankAccount from, BankAccount to, Money amount)` method that moves money between two accounts, and verify that the two accounts are distinct objects (not the same one). Think about what happens if you accidentally pass the same object as both arguments.
2. Measure boxing allocations using `GC.GetAllocatedBytesForCurrentThread()` before and after the `foreach (object ...)` vs `foreach (int ...)` loops. Print the difference and explain it.
3. Implement a `readonly struct MoneyClassic` (without `record`) with hand-written `Equals`/`GetHashCode` and compare the amount of code and behavior with `readonly record struct Money`. Show that `with` is not available without `record`.
4. Investigate string interning: compare `ReferenceEquals("ab"+"c", "abc")` with `ReferenceEquals(string.Intern(BuildAbc()), "abc")`, where `BuildAbc()` returns `"ab"+"c"` at runtime. Explain why the results may differ.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Проект `Payments.M02L01` собирается на .NET 8 / C# 12.
- [ ] (RU) Воспроизведены все выводы: `a=100, b=200`, `acc1=80`, `s1=hello, s2=HELLO`.
- [ ] (RU) Демонстрируется и исправляется CS1612 через `with`.
- [ ] (RU) Boxing/unboxing и неявный boxing в цикле показаны и исправлены.
- [ ] (RU) `Describe(object)` через pattern matching реализован; boxing `Money` отмечен.
- [ ] (RU) `in`-параметр для `BigAmount` применён.
- [ ] (RU) Двуязычные комментарии в ключевых местах; вывод сохранён в `output.txt`.
- [ ] (EN) Project `Payments.M02L01` builds on .NET 8 / C# 12.
- [ ] (EN) All outputs reproduced: `a=100, b=200`, `acc1=80`, `s1=hello, s2=HELLO`.
- [ ] (EN) CS1612 demonstrated and fixed via `with`.
- [ ] (EN) Boxing/unboxing and implicit loop boxing shown and fixed.
- [ ] (EN) `Describe(object)` with pattern matching implemented; `Money` boxing noted.
- [ ] (EN) `in` parameter for `BigAmount` applied.
- [ ] (EN) Bilingual comments at key spots; output saved to `output.txt`.

#### Ресурсы / Resources
- [Microsoft Learn — Value types](https://learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/value-types)
- [Microsoft Learn — Reference types](https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/reference-types)
- [Microsoft Learn — Type Design Guidelines (class vs struct)](https://learn.microsoft.com/dotnet/standard/design-guidelines/type)
- [Microsoft Learn — Boxing and Unboxing](https://learn.microsoft.com/dotnet/csharp/programming-guide/types/boxing-and-unboxing)
- [Microsoft Learn — records (record struct)](https://learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/record)
- [Microsoft Learn — Method parameters (`in`/`ref`/`out`)](https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/method-parameters)
