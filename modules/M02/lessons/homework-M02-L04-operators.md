---
[← К уроку M02-L04](lesson-M02-L04-operators.md) | [⬆ К модулю M02](../README.md) | [Предыдущее ДЗ ←](homework-M02-L03-var-const.md) | [Следующее ДЗ →](homework-M02-L05-nullable.md)
---

### Домашнее задание M02-L04: Операторы: арифметика, сравнение, логика, побитовые / Homework M02-L04: Operators: arithmetic, comparison, logical, bitwise

**Урок / Lesson:** M02-L04
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) Научиться осознанно применять все основные группы операторов C# 12 — арифметические (с учётом целочисленного деления и префиксного/постфиксного инкремента), сравнения, логические с short-circuit, побитовые для флагов и масок, тернарный, null-coalescing и `checked`/`unchecked` — в едином практически осмысленном сценарии. (EN) Learn to apply every major operator family of C# 12 deliberately — arithmetic (with integer division and prefix/postfix increment), comparison, short-circuit logical, bitwise flags and masks, ternary, null-coalescing, and `checked`/`unchecked` — inside one practically meaningful scenario.

#### Связь с уроком / Connection to the lesson
(RU) ДЗ напрямую закрепляет все шесть блоков урока: целочисленное деление и `%`, различие `a++`/`++a`, short-circuit `&&`/`||` против небезопасных `&`/`|`, `[Flags]` enum с `&`, `|`, `^`, `~`, `<<`, `>>`, тернарный `?:` и `??`/`??=`, а также `checked` для переполнения и приоритет операторов со скобками. Те же «подводные камни» из урока — молчаливое переполнение, `NullReferenceException` без short-circuit, целочисленное `7 / 2 == 3` — здесь ловятся в коде студента.
(EN) The homework directly reinforces all six blocks of the lesson: integer division and `%`, the `a++`/`++a` distinction, short-circuit `&&`/`||` versus unsafe `&`/`|`, `[Flags]` enums with `&`, `|`, `^`, `~`, `<<`, `>>`, the ternary `?:` and `??`/`??=`, plus `checked` for overflow and precedence with parentheses. The same pitfalls from the lesson — silent overflow, `NullReferenceException` without short-circuit, integer `7 / 2 == 3` — are exercised here in the student's own code.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Представьте, что вы единственный backend-разработчик мини-сервиса «BookClub Loyalty» — системы лояльности для сети книжных магазинов. Сервис консольный (.NET 8, top-level statements), данных немного, но каждое вычисление должно быть безупречным, потому что ошибки в деньгах и правах доступа замечают сразу. В одном файле вам предстоит объединить все группы операторов, которые вы прошли в уроке M02-L04: арифметику с её целочисленным делением и инкрементом, сравнения, короткозамкнутую логику, побитовые маски для прав доступа, тернарный выбор уровня лояльности, null-coalescing для необязательных полей и `checked` для защищённого подсчёта баллов. Сценарий намеренно сквозной: одно действие вытекает из другого, чтобы операторы не жили изолированно, а работали вместе, как на реальной кухне ресторана из аналогии урока.

Зачем именно так? В уроке подчёркнуто, что операторы — это не набор значков, а инструменты с семантикой: `7 / 2` — это `3`, а не `3.5`; `&&` спасает от `NullReferenceException`; `int.MaxValue + 1` молча становится отрицательным; `[Flags]` enum заменяет пучок `bool`-полей. Если студент проработает все эти нюансы в одном задании, у него сформируется мышечная память: где ставить `checked`, где нужен `double`, где скобки, где `??`. Это и есть цель — не написать «что-то работающее», а написать код, в котором каждый оператор стоит на своём месте по осознанной причине, и уметь это объяснить.

#### Что нужно сделать (пошагово)
1. Создайте консольный проект .NET 8 с именем `BookClubLoyalty`: `dotnet new console -n BookClubLoyalty -o BookClubLoyalty --framework net8.0`. Откройте `Program.cs` и удалите шаблонный код — будете писать на top-level statements.
2. Объявите `[Flags] enum Permission { None = 0, Read = 1, Write = 2, Delete = 4, Admin = 8 }`. В уроке показан именно такой приём для набора переключаемых прав.
3. Объявите запись-пользователя через `record` или класс: поля `string? DisplayName`, `int Age`, `int Points`, `Permission Rights`, `string? PromoCode`. `DisplayName` и `PromoCode` намеренно nullable — на них вы потренируете `??` и `??=`.
4. Реализуйте метод `Permission ApplyDefaults(Permission current, bool isStaff)` через побитовые операции: если `isStaff`, добавьте (`|`) `Read | Write`; если не staff и прав нет — верните `None`. Затем через `^` «переключите» флаг `Delete` (инвертируйте его наличие).
5. Реализуйте `int SafeAddPoints(int current, int add)` в блоке `checked`: при переполнении должно броситься `OverflowException`, а в `Main` его нужно поймать и вывести понятное сообщение. Продемонстрируйте также `unchecked`, где то же сложение молча переполняется (выведите результат, чтобы увидеть отрицательное число).
6. Реализуйте `string LoyaltyTier(int points)` через каскад тернарных операторов: `≥1000` → `"Platinum"`, `≥500` → `"Gold"`, `≥100` → `"Silver"`, иначе `"Bronze"`. Не вкладывайте больше двух уровней без скобок.
7. Реализуйте `bool CanAccess(Permission rights, Permission required)` через `(rights & required) == required` — это правильная проверка «есть ли все нужные флаги» (а не `!= 0`, который проверяет «хотя бы один»).
8. Реализуйте `double AveragePointsPerYear(int totalPoints, int years)` так, чтобы избежать целочисленного деления: один операнд должен стать `double` (как `7.0 / 2` из урока). Выведите результат с двумя знаками.
9. В `Main` создайте двух пользователей: одного с `Rights = Permission.Read | Permission.Write`, другого с `null` `DisplayName` и `PromoCode`. Примените `ApplyDefaults`, посчитайте баллы через `SafeAddPoints`, определите уровень через `LoyaltyTier`, проверьте доступ через `CanAccess`, посчитайте среднее через `AveragePointsPerYear`.
10. Покажите короткое замыкание: вызовите `if (user.PromoCode != null && user.PromoCode.Length >= 5)` и убедитесь, что для null-промокода `Length` не вычисляется. Отдельно покажите, что `&` (без short-circuit) для того же условия бросает `NullReferenceException` — оберните в `try/catch` и выведите текст.
11. В одном месте намеренно используйте постфиксный `x++` и префиксный `++x` на отдельных строках (не внутри сложного выражения, как требует best practice урока) и выведите значения до и после, чтобы продемонстрировать разницу.
12. Запустите `dotnet run` и убедитесь, что вывод содержит: применённые права, уровень лояльности, среднее с двумя знаками, сообщение о переполнении из `checked`, отрицательное число из `unchecked` и сообщение о пойманном `NullReferenceException`.

#### Требования к решению
- Целевая платформа — .NET 8, язык C# 12, файл `Program.cs` использует top-level statements без явного `class Program`/`Main`.
- Все операторы из урока должны быть задействованы осознанно и хотя бы раз: `+ - * / % ++ --`, `== != < > <= >=`, `&& || !` (и контрастно `&` без short-circuit), `& | ^ ~ << >>`, `?:`, `??`, `??=`, `checked`, `unchecked`.
- Для `[Flags]` enum используйте именно побитовые операции, а не наборы `bool`. Проверка прав — через `(rights & required) == required`.
- Логические условия должны использовать `&&`/`||`; единственное место с одиночным `&` — намеренная демонстрация отсутствия short-circuit и ловли `NullReferenceException`.
- Сложения, где переполнение семантически недопустимо (баллы), оборачиваются в `checked`; рядом показывается `unchecked` для контраста.
- Деление баллов на годы должно давать `double`, а не терять дробь; выведите результат с форматом `F2`.
- Код должен быть читаемым: скобки расставлены даже там, где приоритет и так сработает, как требует best practice урока.
- Названия методов и полей — на английском, осмысленные. Вывод в консоль — с поясняющими строками на русском (или двуязычно).

#### Тонкости и подводные камни
- **Целочисленное деление.** Если написать `totalPoints / years` для двух `int`, дробь исчезнет (`7 / 2 == 3`). Приводите один операнд: `(double)totalPoints / years` или `totalPoints / (double)years`. Умножение на `1.0` тоже работает, но явный cast читается понятнее.
- **Инкремент в выражениях.** Урок прямо предостерегает: не используйте `a++`/`++a` внутри сложных выражений. Выносите на отдельную строку. Разница между постфиксом и префиксом критична: `int b = a++` даёт старое значение, `int b = ++a` — новое.
- **Short-circuit.** `&&` не вычисляет правый операнд, если левый `false`; `||` — если левый `true`. Одиночные `&`/`|` вычисляют оба всегда — отсюда `NullReferenceException` на `null.Length`. В «боевых» условиях всегда `&&`/`||`.
- **Молчаливое переполнение.** По умолчанию `int.MaxValue + 1` даёт отрицательное число без исключения. Если баллы «ушли в минус» — это баг, а не фича. `checked { ... }` ловит; используйте его для финансов и лояльности.
- **Проверка флагов.** `(rights & required) != 0` отвечает «есть хотя бы один из требуемых», а `(rights & required) == required` — «есть все требуемые». Для прав доступа обычно нужно второе. Не перепутайте.
- **Строковое сравнение.** `==` для `string` сравнивает значения, не ссылки. Не падайте в ловушку `ReferenceEquals` для проверки равенства содержимого.
- **`??=` vs `??`.** `name ?? "Гость"` возвращает значение, не меняя `name`; `name ??= "Гость"` ещё и присваивает. Выбирайте по смыслу: нужна ли мутация.
- **Приоритет.** `!` сильнее `&&` сильнее `||`. `<<` и `>>` сильнее `&` сильнее `^` сильнее `|`. Скобки бесплатны — ставьте, чтобы документировать намерение.

#### Критерии приёмки
- [ ] Проект `BookClubLoyalty` собирается командой `dotnet build` без предупреждений (warnings as errors желательно).
- [ ] Файл `Program.cs` использует top-level statements, без явного `class Program`.
- [ ] Объявлен `[Flags] enum Permission` с `None=0, Read=1, Write=2, Delete=4, Admin=8`.
- [ ] `ApplyDefaults` использует `|` для добавления и `^` для переключения `Delete`.
- [ ] `SafeAddPoints` обёрнут в `checked` и при переполнении бросает `OverflowException`, пойманный в `Main`.
- [ ] Рядом продемонстрирован `unchecked`, выводящий отрицательное число при переполнении.
- [ ] `LoyaltyTier` реализован каскадом тернарных операторов со скобками.
- [ ] `CanAccess` использует `(rights & required) == required`, а не `!= 0`.
- [ ] `AveragePointsPerYear` возвращает `double`; вывод отформатирован как `F2`.
- [ ] В коде есть осознанная демонстрация short-circuit `&&` и контрастного `&` с пойманным `NullReferenceException`.
- [ ] Постфиксный и префиксный инкремент использованы на отдельных строках с выводом до/после.
- [ ] `??` и `??=` применены к nullable-полям `DisplayName` и `PromoCode`.
- [ ] Хотя бы раз использованы `~`, `<<` или `>>` с выводом результата.
- [ ] Вывод `dotnet run` содержит все ожидаемые строки: права, уровень, среднее, переполнение, NRE.
- [ ] В комментариях (RU+EN) пояснено, почему выбран именно этот оператор в каждой ключевой точке.

#### Подсказки (без прямого ответа)
- Чтобы избежать целочисленного деления, посмотрите на типы операндов: пока оба `int`, результат `int`. Сделайте один `double`.
- Для переключения одного бита используйте оператор, который даёт `0` на одинаковых битах и `1` на разных — это XOR.
- Помните: «проверить, что все требуемые флаги есть» ≠ «проверить, что хоть один есть». Подумайте, что должно получиться после `&`.
- `checked` можно применить и к блоку, и к одному выражению: `checked(max + 1)`.
- Если боитесь забыть про short-circuit — спросите себя: «что произойдёт, если левый операнд `false`, а правый обращается к `null`?»

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 — top-level statements / top-level инструкции
using System;
using System.Collections.Generic;

// [Flags] enum для прав доступа / [Flags] enum for access rights
[Flags] enum Permission { None = 0, Read = 1, Write = 2, Delete = 4, Admin = 8 }

// Пользователь системы лояльности / loyalty user
class User
{
    public string? DisplayName { get; set; }   // nullable — потренируем ?? / nullable for ?? drill
    public int Age { get; set; }
    public int Points { get; set; }
    public Permission Rights { get; set; }
    public string? PromoCode { get; set; }     // nullable — потренируем ??= / nullable for ??=
}

// Добавляем права через | (OR) и переключаем Delete через ^ (XOR)
// Add rights via | (OR) and toggle Delete via ^ (XOR)
Permission ApplyDefaults(Permission current, bool isStaff) =>
    isStaff
        ? current | Permission.Read | Permission.Write
        : current == Permission.None ? Permission.None : current ^ Permission.Delete;

// checked ловит переполнение баллов / checked catches points overflow
int SafeAddPoints(int current, int add) => checked(current + add);

// Каскад тернарных операторов со скобками / ternary cascade with parentheses
string LoyaltyTier(int points) =>
    points >= 1000 ? "Platinum"
    : points >= 500 ? "Gold"
    : points >= 100 ? "Silver"
    : "Bronze";

// Все требуемые флаги должны быть — поэтому == required, а не != 0
// All required flags must be present — so == required, not != 0
bool CanAccess(Permission rights, Permission required) =>
    (rights & required) == required;

// Избегаем целочисленного деления: один операнд — double / avoid integer division
double AveragePointsPerYear(int totalPoints, int years) =>
    (double)totalPoints / years;

// --- Main (top-level) ---
User staff = new()
{
    DisplayName = "Анна / Anna",
    Age = 30,
    Points = 750,
    Rights = Permission.Read | Permission.Write,
    PromoCode = "BOOK2024"
};

User guest = new()
{
    DisplayName = null,          // потренируем ?? / drill ??
    Age = 17,
    Points = 50,
    Rights = Permission.None,
    PromoCode = null             // потренируем ??= / drill ??=
};

// ?? — запасное значение без мутации / ?? — fallback without mutation
string staffName = staff.DisplayName ?? "Гость / Guest";
string guestName = guest.DisplayName ?? "Гость / Guest";

// ??= — присваиваем только если null / ??= — assign only if null
guest.PromoCode ??= "DEFAULT";

// Применяем права: staff получает Read|Write, у guest Delete переключается
staff.Rights = ApplyDefaults(staff.Rights, isStaff: true);
guest.Rights = ApplyDefaults(guest.Rights, isStaff: false);

// Уровень лояльности через тернар / loyalty tier via ternary
Console.WriteLine($"{staffName}: {LoyaltyTier(staff.Points)}");
Console.WriteLine($"{guestName}: {LoyaltyTier(guest.Points)}");

// Среднее с F2 — избежали целочисленного деления / average F2 — no integer division
Console.WriteLine($"staff avg/year: {AveragePointsPerYear(staff.Points, 3):F2}");

// checked: переполнение ловится / checked: overflow caught
try
{
    int big = SafeAddPoints(int.MaxValue, 10);
    Console.WriteLine($"big = {big}");
}
catch (OverflowException)
{
    Console.WriteLine("checked: переполнение баллов поймано / overflow caught");
}

// unchecked: молчаливое переполнение / unchecked: silent overflow
int silent = unchecked(int.MaxValue + 1);
Console.WriteLine($"unchecked overflow = {silent}");   // отрицательное / negative

// Побитовые: сдвиг и инверсия / bitwise: shift and invert
int one = 1;
Console.WriteLine($"1 << 3 = {one << 3}");   // 8 = умножение на 2^3
Console.WriteLine($"~1 = {~one}");           // -2

// Доступ: все требуемые флаги / access: all required flags
Console.WriteLine($"staff can Write+Delete: {CanAccess(staff.Rights, Permission.Write | Permission.Delete)}");

// Short-circuit && безопасен для null / short-circuit && is null-safe
if (guest.PromoCode != null && guest.PromoCode.Length >= 5)
    Console.WriteLine("guest promo ok");
else
    Console.WriteLine("guest promo short or null");

// Контраст: одиночный & без short-circuit бросает NRE / single & without short-circuit throws NRE
try
{
    User ghost = new() { PromoCode = null };
    bool bad = ghost.PromoCode != null & ghost.PromoCode.Length >= 5;  // правый операнд вычисляется!
}
catch (NullReferenceException)
{
    Console.WriteLine("& без short-circuit: поймано NullReferenceException / caught");
}

// Инкремент на отдельных строках / increment on its own lines
int counter = 5;
int post = counter++;   // post = 5, counter = 6
int pre = ++counter;    // pre = 7, counter = 7
Console.WriteLine($"post={post}, pre={pre}, counter={counter}");

// Приоритет: скобки документируют намерение / precedence: parentheses document intent
bool verdict = !(staff.Points > guest.Points && guest.Points > 0) || (staff.Age + guest.Age == 47);
Console.WriteLine($"verdict={verdict}");
```

Разбор по строкам. `[Flags] enum Permission` — это ровно та конструкция из урока, что заменяет набор `bool`-полей: значения `1, 2, 4, 8` — степени двойки, чтобы каждый флаг занимал свой бит. `ApplyDefaults` использует `|` для объединения прав (как `Permissions.Read | Permissions.Write` из урока) и `^` для переключения `Delete` — XOR именно потому, что на одинаковых битах даёт `0`, на разных `1`, то есть снимает флаг, если он был, и ставит, если не было. `SafeAddPoints` обёрнут в `checked`: баллы — это «деньги» лояльности, переполнение тут недопустимо, поэтому мы хотим `OverflowException`, а не молчаливый переход через `int.MaxValue`. Рядом `unchecked(int.MaxValue + 1)` показывает то самое «молчаливое переполнение» из урока — отрицательный результат без исключения. `LoyaltyTier` — каскад тернарных операторов; урок разрешает умеренную вложенность, но требует скобок и читаемости, поэтому каждый уровень на своей строке. `CanAccess` построен на `(rights & required) == required` — это ключевая тонкость: `!= 0` означало бы «есть хотя бы один требуемый флаг», а нам нужно «есть все»; маска `&` оставляет только биты `required`, и сравнение с `required` проверяет, что ни один не потерян. `AveragePointsPerYear` приводит `totalPoints` к `double` до деления — иначе сработало бы целочисленное деление `7 / 2 == 3` и дробь исчезла; формат `F2` выводит два знака. `??` даёт запасное имя без мутации поля, `??=` — присваивает `PromoCode` только если он `null`, ровно как в уроке. Блок с `&&` безопасен: если `PromoCode` равен `null`, `Length` не вычисляется благодаря short-circuit; контрастный `&` намеренно вычисляет оба операнда и бросает `NullReferenceException`, который мы ловим, — это наглядная иллюстрация предостережения из урока. Инкременты вынесены на отдельные строки (`post = counter++` даёт старое значение, `pre = ++counter` — новое), как требует best practice «не смешивайте в сложных выражениях». Финальная строка с `!(...) || (...)` демонстрирует приоритет: `!` сильнее `&&` сильнее `||`, но скобки делают намерение явным — это и есть рекомендация урока про «бесплатные скобки».

#### Задания на углубление (бонус)
1. Добавьте метод `Permission RevokeAllExcept(Permission current, Permission keep)`, который через одну побитовую операцию оставляет только указанные флаги. Подсказка: подумайте, какая маска обнуляет всё, кроме `keep`.
2. Замените каскад тернарных операторов в `LoyaltyTier` на `switch` expression с pattern matching (`>= 1000 => "Platinum", ...`) и сравните читаемость. Объясните, когда тернар уместнее `switch`.
3. Реализуйте подсчёт «хороших» промокодов через побитовый счётчик: считайте, у скольких пользователей установлен хотя бы один из флагов `Read|Write`, используя только `&` и сравнение, без LINQ.
4. Добавьте «горячий путь» с `unchecked` для быстрого хэширования баллов (`hash = (points * 31) & 0x7FFFFFFF`) и объясните, почему здесь `unchecked` уместен, а в `SafeAddPoints` — нет.

---

## Statement in English / Постановка на английском

### Homework M02-L04: Operators: arithmetic, comparison, logical, bitwise

#### Context & motivation
Imagine you are the sole backend developer of a small service called “BookClub Loyalty” — a loyalty system for a chain of bookstores. The service is a console application (.NET 8, top-level statements); there is not much data, but every computation must be flawless, because mistakes in money and access rights are noticed immediately. In a single file you will combine every operator family you studied in lesson M02-L04: arithmetic with its integer division and increment, comparison, short-circuit logic, bitwise masks for access rights, the ternary for loyalty tiers, null-coalescing for optional fields, and `checked` for protected point arithmetic. The scenario is intentionally end-to-end: one action flows into the next, so the operators do not live in isolation but work together, exactly like the restaurant-kitchen analogy from the lesson.

Why this shape? The lesson stresses that operators are not a grab-bag of symbols but tools with semantics: `7 / 2` is `3`, not `3.5`; `&&` rescues you from `NullReferenceException`; `int.MaxValue + 1` silently becomes negative; a `[Flags]` enum replaces a fistful of `bool` fields. If a student exercises all these nuances in one assignment, muscle memory forms: where to put `checked`, where a `double` is needed, where parentheses belong, where `??` applies. That is the goal — not to write “something that compiles,” but to write code where every operator is in its place for a conscious reason, and to be able to explain it.

#### What to do step by step
1. Create a .NET 8 console project named `BookClubLoyalty`: `dotnet new console -n BookClubLoyalty -o BookClubLoyalty --framework net8.0`. Open `Program.cs` and clear the template — you will write top-level statements.
2. Declare `[Flags] enum Permission { None = 0, Read = 1, Write = 2, Delete = 4, Admin = 8 }`. The lesson shows exactly this pattern for a set of toggleable rights.
3. Declare a user type via `record` or class: fields `string? DisplayName`, `int Age`, `int Points`, `Permission Rights`, `string? PromoCode`. `DisplayName` and `PromoCode` are intentionally nullable — on them you will drill `??` and `??=`.
4. Implement `Permission ApplyDefaults(Permission current, bool isStaff)` with bitwise ops: if `isStaff`, add (`|`) `Read | Write`; if not staff and no rights, return `None`. Then toggle the `Delete` flag with `^` (invert its presence).
5. Implement `int SafeAddPoints(int current, int add)` inside a `checked` context: on overflow it must throw `OverflowException`, and in `Main` you must catch it and print a clear message. Also demonstrate `unchecked`, where the same addition silently overflows (print the result to see the negative number).
6. Implement `string LoyaltyTier(int points)` as a cascade of ternary operators: `>=1000` → `"Platinum"`, `>=500` → `"Gold"`, `>=100` → `"Silver"`, otherwise `"Bronze"`. Do not nest more than two levels without parentheses.
7. Implement `bool CanAccess(Permission rights, Permission required)` as `(rights & required) == required` — the correct “all required flags present” check (not `!= 0`, which means “at least one”).
8. Implement `double AveragePointsPerYear(int totalPoints, int years)` so that integer division is avoided: one operand must become `double` (like `7.0 / 2` from the lesson). Print the result with two decimals.
9. In `Main`, create two users: one with `Rights = Permission.Read | Permission.Write`, the other with `null` `DisplayName` and `PromoCode`. Apply `ApplyDefaults`, compute points via `SafeAddPoints`, determine the tier via `LoyaltyTier`, check access via `CanAccess`, compute the average via `AveragePointsPerYear`.
10. Demonstrate short-circuiting: call `if (user.PromoCode != null && user.PromoCode.Length >= 5)` and confirm that for a null promo code `Length` is not evaluated. Separately show that `&` (without short-circuit) for the same condition throws `NullReferenceException` — wrap it in `try/catch` and print the message.
11. In one place deliberately use postfix `x++` and prefix `++x` on separate lines (not inside a complex expression, as the lesson’s best practice requires) and print the values before and after to show the difference.
12. Run `dotnet run` and confirm the output contains: applied rights, loyalty tier, average with two decimals, the overflow message from `checked`, the negative number from `unchecked`, and the caught `NullReferenceException` message.

#### Requirements
- Target platform is .NET 8, language C# 12, and `Program.cs` uses top-level statements with no explicit `class Program`/`Main`.
- Every operator from the lesson must be used deliberately and at least once: `+ - * / % ++ --`, `== != < > <= >=`, `&& || !` (and, contrastingly, `&` without short-circuit), `& | ^ ~ << >>`, `?:`, `??`, `??=`, `checked`, `unchecked`.
- For the `[Flags]` enum use bitwise operations, not a bunch of `bool`s. Access checking uses `(rights & required) == required`.
- Logical conditions use `&&`/`||`; the only place with a single `&` is the deliberate demonstration of missing short-circuit and the caught `NullReferenceException`.
- Additions where overflow is semantically unacceptable (points) are wrapped in `checked`; next to it, `unchecked` is shown for contrast.
- Dividing points by years must yield a `double`, not lose the fraction; print the result with the `F2` format.
- The code must be readable: parentheses are placed even where precedence would already work, as the lesson’s best practice demands.
- Method and field names are in English and meaningful. Console output uses explanatory strings (bilingual is fine).

#### Pitfalls
- **Integer division.** If you write `totalPoints / years` for two `int`s, the fraction vanishes (`7 / 2 == 3`). Cast one operand: `(double)totalPoints / years` or `totalPoints / (double)years`. Multiplying by `1.0` also works, but an explicit cast reads more clearly.
- **Increment in expressions.** The lesson explicitly warns: do not use `a++`/`++a` inside complex expressions. Put them on their own line. The difference between postfix and prefix is critical: `int b = a++` yields the old value, `int b = ++a` the new one.
- **Short-circuit.** `&&` does not evaluate the right operand if the left is `false`; `||` does not if the left is `true`. Single `&`/`|` always evaluate both — hence `NullReferenceException` on `null.Length`. In production, always use `&&`/`||`.
- **Silent overflow.** By default `int.MaxValue + 1` yields a negative number with no exception. If points “go negative,” that is a bug, not a feature. `checked { ... }` catches it; use it for money and loyalty.
- **Flag checks.** `(rights & required) != 0` answers “at least one required flag is present,” while `(rights & required) == required` answers “all required flags are present.” For access rights you usually want the latter. Do not mix them up.
- **String comparison.** `==` for `string` compares values, not references. Do not fall into the trap of `ReferenceEquals` for content equality.
- **`??=` vs `??`.** `name ?? "Guest"` returns a value without mutating `name`; `name ??= "Guest"` also assigns. Choose by meaning: do you need the mutation?
- **Precedence.** `!` binds tighter than `&&`, which binds tighter than `||`. `<<` and `>>` bind tighter than `&`, which binds tighter than `^`, which binds tighter than `|`. Parentheses are free — use them to document intent.

#### Acceptance criteria
- [ ] The `BookClubLoyalty` project builds with `dotnet build` without warnings (warnings-as-errors preferred).
- [ ] `Program.cs` uses top-level statements, with no explicit `class Program`.
- [ ] A `[Flags] enum Permission` is declared with `None=0, Read=1, Write=2, Delete=4, Admin=8`.
- [ ] `ApplyDefaults` uses `|` to add and `^` to toggle `Delete`.
- [ ] `SafeAddPoints` is wrapped in `checked` and throws `OverflowException` on overflow, caught in `Main`.
- [ ] Next to it, `unchecked` is demonstrated, printing a negative overflow result.
- [ ] `LoyaltyTier` is implemented as a ternary cascade with parentheses.
- [ ] `CanAccess` uses `(rights & required) == required`, not `!= 0`.
- [ ] `AveragePointsPerYear` returns `double`; output is formatted as `F2`.
- [ ] The code deliberately demonstrates short-circuit `&&` and the contrasting `&` with a caught `NullReferenceException`.
- [ ] Postfix and prefix increment are used on separate lines with before/after output.
- [ ] `??` and `??=` are applied to the nullable `DisplayName` and `PromoCode` fields.
- [ ] `~`, `<<`, or `>>` is used at least once with output.
- [ ] The `dotnet run` output contains all expected lines: rights, tier, average, overflow, NRE.
- [ ] Comments (RU+EN) explain why each operator was chosen at each key point.

#### Hints (no direct answer)
- To avoid integer division, look at operand types: while both are `int`, the result is `int`. Make one a `double`.
- To toggle a single bit, use the operator that yields `0` on equal bits and `1` on differing ones — that is XOR.
- Remember: “check that all required flags are present” is not the same as “check that at least one is.” Think about what `&` must yield.
- `checked` applies to both a block and a single expression: `checked(max + 1)`.
- If you worry about forgetting short-circuit, ask yourself: “what happens if the left operand is `false` and the right one touches `null`?”

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8 — top-level statements
using System;
using System.Collections.Generic;

// [Flags] enum for access rights
[Flags] enum Permission { None = 0, Read = 1, Write = 2, Delete = 4, Admin = 8 }

// Loyalty user
class User
{
    public string? DisplayName { get; set; }   // nullable for ?? drill
    public int Age { get; set; }
    public int Points { get; set; }
    public Permission Rights { get; set; }
    public string? PromoCode { get; set; }     // nullable for ??=
}

// Add rights via | (OR) and toggle Delete via ^ (XOR)
Permission ApplyDefaults(Permission current, bool isStaff) =>
    isStaff
        ? current | Permission.Read | Permission.Write
        : current == Permission.None ? Permission.None : current ^ Permission.Delete;

// checked catches points overflow
int SafeAddPoints(int current, int add) => checked(current + add);

// Ternary cascade with parentheses
string LoyaltyTier(int points) =>
    points >= 1000 ? "Platinum"
    : points >= 500 ? "Gold"
    : points >= 100 ? "Silver"
    : "Bronze";

// All required flags must be present — so == required, not != 0
bool CanAccess(Permission rights, Permission required) =>
    (rights & required) == required;

// Avoid integer division: one operand is double
double AveragePointsPerYear(int totalPoints, int years) =>
    (double)totalPoints / years;

// --- Main (top-level) ---
User staff = new()
{
    DisplayName = "Anna",
    Age = 30,
    Points = 750,
    Rights = Permission.Read | Permission.Write,
    PromoCode = "BOOK2024"
};

User guest = new()
{
    DisplayName = null,          // drill ??
    Age = 17,
    Points = 50,
    Rights = Permission.None,
    PromoCode = null             // drill ??=
};

// ?? — fallback without mutation
string staffName = staff.DisplayName ?? "Guest";
string guestName = guest.DisplayName ?? "Guest";

// ??= — assign only if null
guest.PromoCode ??= "DEFAULT";

// Apply rights: staff gets Read|Write, guest toggles Delete
staff.Rights = ApplyDefaults(staff.Rights, isStaff: true);
guest.Rights = ApplyDefaults(guest.Rights, isStaff: false);

// Loyalty tier via ternary
Console.WriteLine($"{staffName}: {LoyaltyTier(staff.Points)}");
Console.WriteLine($"{guestName}: {LoyaltyTier(guest.Points)}");

// Average F2 — no integer division
Console.WriteLine($"staff avg/year: {AveragePointsPerYear(staff.Points, 3):F2}");

// checked: overflow caught
try
{
    int big = SafeAddPoints(int.MaxValue, 10);
    Console.WriteLine($"big = {big}");
}
catch (OverflowException)
{
    Console.WriteLine("checked: points overflow caught");
}

// unchecked: silent overflow
int silent = unchecked(int.MaxValue + 1);
Console.WriteLine($"unchecked overflow = {silent}");   // negative

// Bitwise: shift and invert
int one = 1;
Console.WriteLine($"1 << 3 = {one << 3}");   // 8 = multiply by 2^3
Console.WriteLine($"~1 = {~one}");           // -2

// Access: all required flags
Console.WriteLine($"staff can Write+Delete: {CanAccess(staff.Rights, Permission.Write | Permission.Delete)}");

// Short-circuit && is null-safe
if (guest.PromoCode != null && guest.PromoCode.Length >= 5)
    Console.WriteLine("guest promo ok");
else
    Console.WriteLine("guest promo short or null");

// Contrast: single & without short-circuit throws NRE
try
{
    User ghost = new() { PromoCode = null };
    bool bad = ghost.PromoCode != null & ghost.PromoCode.Length >= 5;  // right operand evaluated!
}
catch (NullReferenceException)
{
    Console.WriteLine("& without short-circuit: caught NullReferenceException");
}

// Increment on its own lines
int counter = 5;
int post = counter++;   // post = 5, counter = 6
int pre = ++counter;    // pre = 7, counter = 7
Console.WriteLine($"post={post}, pre={pre}, counter={counter}");

// Precedence: parentheses document intent
bool verdict = !(staff.Points > guest.Points && guest.Points > 0) || (staff.Age + guest.Age == 47);
Console.WriteLine($"verdict={verdict}");
```

Line-by-line walk-through. `[Flags] enum Permission` is exactly the construct from the lesson that replaces a set of `bool` fields: the values `1, 2, 4, 8` are powers of two so each flag owns its bit. `ApplyDefaults` uses `|` to combine rights (as in `Permissions.Read | Permissions.Write` from the lesson) and `^` to toggle `Delete` — XOR precisely because it yields `0` on equal bits and `1` on differing ones, removing the flag if present and setting it if absent. `SafeAddPoints` is wrapped in `checked`: points are the “money” of loyalty, overflow is unacceptable here, so we want an `OverflowException` rather than a silent wrap past `int.MaxValue`. Next to it, `unchecked(int.MaxValue + 1)` shows the “silent overflow” from the lesson — a negative result with no exception. `LoyaltyTier` is a cascade of ternary operators; the lesson allows moderate nesting but demands parentheses and readability, so each level sits on its own line. `CanAccess` is built on `(rights & required) == required` — the key nuance: `!= 0` would mean “at least one required flag is set,” but we need “all are set”; the `&` mask keeps only the bits of `required`, and the comparison with `required` verifies none was lost. `AveragePointsPerYear` casts `totalPoints` to `double` before division — otherwise integer division `7 / 2 == 3` would erase the fraction; the `F2` format prints two decimals. `??` provides a fallback name without mutating the field, `??=` assigns `PromoCode` only if it is `null`, exactly as in the lesson. The `&&` block is safe: if `PromoCode` is `null`, `Length` is not evaluated thanks to short-circuit; the contrasting `&` deliberately evaluates both operands and throws `NullReferenceException`, which we catch — a vivid illustration of the lesson’s warning. Increments are placed on separate lines (`post = counter++` gives the old value, `pre = ++counter` the new one), following the best practice “do not mix them inside complex expressions.” The final line with `!(...) || (...)` demonstrates precedence: `!` binds tighter than `&&`, which binds tighter than `||`, but the parentheses make intent explicit — this is the lesson’s “parentheses are free” recommendation in action.

#### Going deeper (bonus)
1. Add a method `Permission RevokeAllExcept(Permission current, Permission keep)` that, in a single bitwise operation, leaves only the specified flags. Hint: think about which mask zeroes everything except `keep`.
2. Replace the ternary cascade in `LoyaltyTier` with a `switch` expression using pattern matching (`>= 1000 => "Platinum", ...`) and compare readability. Explain when a ternary is preferable to a `switch`.
3. Implement a “good promo” counter through a bitwise approach: count how many users have at least one of `Read|Write` set, using only `&` and comparison, no LINQ.
4. Add a hot path with `unchecked` for fast point hashing (`hash = (points * 31) & 0x7FFFFFFF`) and explain why `unchecked` is appropriate here but not in `SafeAddPoints`.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Проект `BookClubLoyalty` создан и собирается без ошибок и предупреждений.
- [ ] (RU) Все операторы урока использованы осознанно и хотя бы раз.
- [ ] (RU) `[Flags]` enum и побитовые операции применены для прав доступа.
- [ ] (RU) `checked` ловит переполнение баллов, `unchecked` показан для контраста.
- [ ] (RU) Целочисленное деление избегается в `AveragePointsPerYear`.
- [ ] (RU) Демонстрируется short-circuit `&&` и контрастный `&` с пойманным NRE.
- [ ] (RU) Инкременты на отдельных строках, `??`/`??=` применены к nullable-полям.
- [ ] (RU) Вывод `dotnet run` содержит все ожидаемые строки.
- [ ] (EN) The `BookClubLoyalty` project is created and builds without errors or warnings.
- [ ] (EN) Every operator from the lesson is used deliberately and at least once.
- [ ] (EN) A `[Flags]` enum and bitwise operations are used for access rights.
- [ ] (EN) `checked` catches points overflow; `unchecked` is shown for contrast.
- [ ] (EN) Integer division is avoided in `AveragePointsPerYear`.
- [ ] (EN) Short-circuit `&&` and the contrasting `&` with a caught NRE are demonstrated.
- [ ] (EN) Increments are on separate lines; `??`/`??=` are applied to nullable fields.
- [ ] (EN) The `dotnet run` output contains all expected lines.

#### Ресурсы / Resources
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/operators/ — C# operators reference / Справочник по операторам C#
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/operators/arithmetic-operators — Arithmetic operators / Арифметические операторы
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/operators/boolean-logical-operators — Boolean logical operators / Логические операторы
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/operators/bitwise-and-shift-operators — Bitwise and shift operators / Побитовые операторы и операторы сдвига
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/operators/null-coalescing-operator — Null-coalescing operators / Операторы null-объединения
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/checked-and-unchecked — checked and unchecked / checked и unchecked
- Урок / Lesson: [M02-L04](lesson-M02-L04-operators.md)

---
[← К уроку M02-L04](lesson-M02-L04-operators.md) | [⬆ К модулю M02](../README.md) | [Предыдущее ДЗ ←](homework-M02-L03-var-const.md) | [Следующее ДЗ →](homework-M02-L05-nullable.md)
---
