---
[← К уроку M03-L03](lesson-M03-L03-loops.md) | [⬆ К модулю M03](../README.md) | [Следующее ДЗ →](homework-M03-L04-break-continue.md)
---

### Домашнее задание M03-L03: Циклы for, while, do-while, foreach / Homework M03-L03: Loops: for, while, do-while, foreach

**Урок / Lesson:** M03-L03
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) Научиться осознанно выбирать между `for`, `while`, `do-while` и `foreach` под задачу, правильно задавать границы счётчика, гарантировать завершение условных циклов и безопасно удалять элементы из коллекции без `InvalidOperationException`. (EN) Learn to choose consciously between `for`, `while`, `do-while` and `foreach` per task, set counter bounds correctly, guarantee termination of condition-driven loops, and remove elements from a collection safely without `InvalidOperationException`.

#### Связь с уроком / Connection to the lesson
(RU) ДЗ напрямую закрепляет все четыре вида циклов из урока: `for` применяется для генерации последовательности с известным диапазоном, `while` — для алгоритма Коллатца, где число итераций заранее неизвестно, `do-while` — для меню, которое должно показать пункт хотя бы один раз, а `foreach` — для обхода результатов. Отдельно отрабатывается критическое ограничение урока: нельзя структурно модифицировать коллекцию внутри `foreach`, поэтому фильтрация реализуется через `RemoveAll` и обратный `for`. (EN) The homework directly reinforces all four loop kinds from the lesson: `for` generates a sequence with a known range, `while` drives the Collatz algorithm where the iteration count is unknown up front, `do-while` powers a menu that must appear at least once, and `foreach` walks the results. It also drills the lesson's key restriction: you cannot structurally modify a collection inside `foreach`, so filtering is done with `RemoveAll` and a reverse `for`.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Представьте, что вы делаете учебную утилиту «Лаборатория последовательностей» для кафедры дискретной математики. Преподаватель хочет демонстрировать студентам, как ведут себя числа под действием рекуррентных правил, и просить утилиту сама генерировать ряды, считать «шаги схождения» и фильтровать результаты. Утилита консольная, работает локально, не использует баз данных — только чистый C# 12 / .NET 8. Важно, чтобы код был написан «по-взрослому»: каждый вид цикла применён ровно там, где он уместен по семантике, а не «потому что компилируется». Это значит, что генерация с известным числом элементов — это `for`; алгоритм, число шагов которого заранее неизвестно, — `while`; меню, которое нужно показать минимум один раз, — `do-while`; а обход готовой коллекции без индексов — `foreach`. Дополнительно требуется продемонстрировать безопасные приёмы удаления элементов из `List<T>`, потому что наивная попытка удалить чётные внутри `foreach` выбросит `InvalidOperationException` — ровно ту ошибку, которую урок называет классической. В качестве «мяса» задачи используется гипотеза Коллатца: для любого натурального `n` последовательность `n → n/2` (если чётно) или `3n+1` (если нечётно) сходится к 1. Число шагов до единицы заранее неизвестно — это идеальный полигон для `while`. Таким образом, одно небольшое приложение串联ит все четыре цикла и ключевые ошибки урока в один связный сценарий, а не разрозненные примеры.

#### Что нужно сделать (пошагово)
1. Создайте проект консольного приложения: `dotnet new console -n SequenceLab -o SequenceLab -f net8.0`, перейдите в каталог `SequenceLab` и замените содержимое `Program.cs` на решение ниже. Проверьте версию SDK: `dotnet --version` — ожидается 8.0.x.
2. В `Program.cs` (top-level statements) объявите константу `MaxN = 20` — верхнюю границу генерируемого ряда. Вынесите её в именованную константу, а не «магическое число» в условии цикла, как требует best practice урока.
3. Реализуйте генерацию последовательности первых `MaxN` треугольных чисел `T(i) = i*(i+1)/2` для `i` от `1` до `MaxN` включительно через цикл `for`. Верхняя граница включительна — consciously выберите `<=` и обоснуйте. Сохраните числа в `List<int> sequence`.
4. Реализуйте метод `int CollatzSteps(int n)`, который через `while` считает шаги до достижения `1`. Тело цикла обязано продвигать условие к `false`: на каждой итерации `n` уменьшается при чётном и строго меняется при нечётном. Добавьте предохранительный лимит итераций (например, 100 000), чтобы случайно переданное `0` или отрицательное число не привело к бесконечному циклу — урок прямо предостерегает от этого.
5. Постройте `Dictionary<int, int> stepsByValue`, где ключ — треугольное число, значение — шаги Коллатца. Заполняйте словарь через `foreach` по `sequence`. Выводите результат через `foreach (var kvp in stepsByValue)` — сразу с `kvp.Key` и `kvp.Value`, а не через `dict[k]` (урок называет эту ошибку частой).
6. Реализуйте меню через `do-while`: оно должно вывести пункты `(1) Показать все, (2) Удалить чётные, (3) Удалить кратные 5, (4) Выход` хотя бы один раз, затем прочитать выбор и обработать его. Используйте `switch` с pattern matching для разбора.
7. Команда «Удалить чётные» должна работать двумя способами для демонстрации: на копии списка через `RemoveAll(n => n % 2 == 0)` и на другой копии через обратный `for` с `RemoveAt(i)` от `Count-1` до `0`. Выведите оба результата и убедитесь, что они совпадают. Покажите в комментарии закомментированный неправильный вариант с `foreach`, который выбросил бы `InvalidOperationException`.
8. После каждой операции выводите обновлённую коллекцию через `foreach` с форматированием `string.Join(", ", ...)`.
9. Запустите: `dotnet run`. Подайте на вход последовательность `1` (показать), `2` (удалить чётные), `3` (удалить кратные 5), `4` (выход). Зафиксируйте ожидаемый вывод.
10. Добавьте модульные проверочные вызовы прямо в `Program.cs` (без отдельного тест-проекта): assert-проверки, что `CollatzSteps(1) == 0`, `CollatzSteps(2) == 1`, `CollatzSteps(6) == 8`, и что обратный `for` и `RemoveAll` дают одинаковый список. При несовпадении печатайте `FAIL`.
11. Проверьте edge-кейсы: пустая последовательность (`MaxN = 0`), `MaxN = 1`, передача `n = 1` в `CollatzSteps` (тело `while` не должно выполниться ни разу).

#### Требования к решению
Решение должно компилироваться без предупреждений ( уровень `nullable` включён, `TreatWarningsAsErrors` опционально). Использовать только возможности C# 12 / .NET 8: top-level statements, pattern matching (`is < 1 or > 10`), collection expressions там, где это уместно, необработанные строковые литералы не нужны — достаточно интерполяции. Каждый из четырёх циклов должен быть применён ровно по своему семантическому назначению: `for` — генерация с известным диапазоном, `while` — Коллатц, `do-while` — меню, `foreach` — обход коллекции и словаря. Границы `for` должны быть именованы через константу или переменную, без «магических» чисел. В `while` обязателен предохранительный лимит итераций. Удаление по условию выполнено безопасно (`RemoveAll` + обратный `for`), и запрещённый вариант с `foreach` показан в комментарии с пояснением причины. Словарь обходится через `KeyValuePair`. Все выводы информативны и помечены префиксом операции. Код читается сверху вниз без «прыжков» и избыточных флагов. Имена переменных осмысленны (`stepsByValue`, не `d`). Решение не использует LINQ там, где урок рекомендует явный цикл (например, не заменяйте `CollatzSteps` на рекурсию или LINQ-агрегат — нужен именно `while`).

#### Тонкости и подводные камни
- **Off-by-one в `for`.** Урок подчёркивает: `<=` вместо `<` даёт ошибку на единицу. Для треугольных чисел `i` от `1` до `MaxN` включительно нужен `i <= MaxN`; если написать `i < MaxN`, потеряете последний элемент. Тестируйте границу `MaxN = 1`.
- **Бесконечный `while`.** В `CollatzSteps` тело обязано менять `n`. Если забыть присваивание `n = n / 2` или `n = 3 * n + 1`, цикл не завершится. Предохранительный счётчик итераций — страховка от логических ошибок и от передачи `0` (для `0` правило не определено, нужен явный возврат или `break`).
- **Мутация коллекции в `foreach`.** Удаление элемента из `List<T>` внутри `foreach` выбрасывает `InvalidOperationException`, потому что итератор отслеживает версию коллекции. Допустимо менять *значения* (свойства объектов), но не *структуру*.
- **Обратный `for` при удалении.** Идти нужно с конца (`Count-1` вниз до `0`), иначе после `RemoveAt(i)` индексы сдвигаются и вы пропускаете следующий элемент. Урок явно демонстрирует этот приём.
- **`RemoveAll` декларативнее.** Для простого предиката `RemoveAll` короче и не подвержен ошибке индексов; обратный `for` нужен, когда удаление сопровождается побочными действиями.
- **Словарь через `KeyValuePair`.** `foreach (var k in dict)` + `dict[k]` — лишний lookup. `foreach (var kvp in dict)` даёт и ключ, и значение сразу.
- **`do-while` vs `while`.** Меню обязано появиться хотя бы раз — это семантика `do-while`. Если бы ноль показов был допустим, нужен `while`.
- **Изменение счётчика `for` в теле.** Не меняйте `i` внутри тела — это «сбивает» шаг. Управляйте потоком через `break`/`continue` (тема следующего урока), а здесь просто не трогайте счётчик.
- **Nullable-ввод в меню.** `Console.ReadLine()` возвращает `string?`; разбор через `int.TryParse` с `out var v` и pattern matching `is < 1 or > 4` — устойчив к пустому и нечисловому вводу.

#### Критерии приёмки
- [ ] Проект создаётся командой `dotnet new console -n SequenceLab -f net8.0` и собирается без ошибок и предупреждений.
- [ ] `Program.cs` использует top-level statements C# 12.
- [ ] Генерация треугольных чисел реализована циклом `for` с границей `i <= MaxN`.
- [ ] `MaxN` — именованная константа, а не «магическое» число в условии.
- [ ] `CollatzSteps` реализован через `while` и содержит предохранительный лимит итераций.
- [ ] `CollatzSteps(1) == 0`, `CollatzSteps(2) == 1`, `CollatzSteps(6) == 8` проверяются и проходят.
- [ ] Меню реализовано через `do-while` и показывается хотя бы один раз.
- [ ] Разбор выбора меню использует pattern matching (`switch` или `is`-паттерны).
- [ ] Словарь `stepsByValue` заполняется и обходится через `foreach (var kvp in ...)`.
- [ ] «Удалить чётные» реализовано двумя способами: `RemoveAll` и обратный `for` — результаты совпадают.
- [ ] В коде присутствует закомментированный неправильный вариант удаления внутри `foreach` с пояснением причины `InvalidOperationException`.
- [ ] Edge-кейсы `MaxN = 0` и `MaxN = 1` не вызывают исключений.
- [ ] Передача `n = 0` в `CollatzSteps` не приводит к бесконечному циклу.
- [ ] Вывод каждой операции помечен префиксом и читаем.
- [ ] Код не содержит «магических» чисел в условиях циклов.

#### Подсказки (без прямого ответа)
- Треугольное число: `T(i) = i*(i+1)/2`. Подумайте, почему для `i` от `1` уместен именно `<= MaxN`.
- В `CollatzSteps` сначала обработайте крайний случай `n <= 0` отдельно — для него правило не определено.
- Для предохранителя заведите локальный счётчик `int guard = 0` и условие `while (n != 1 && guard++ < 100_000)`.
- Обратный `for`: `for (int i = list.Count - 1; i >= 0; i--)`.
- Чтобы меню не «зависало» на неверном вводе, в `default`-ветви `switch` просто напечатайте подсказку и дайте `do-while` повторить.
- Сравнить два списка на равенство содержимого можно построчно через `string.Join` или цикл `for` по индексам.

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 — top-level statements
// Лаборатория последовательностей: for + while + do-while + foreach / Sequence lab
using System;
using System.Collections.Generic;

const int MaxN = 20;                                  // именованная граница, не «магическое число» / named bound

// 1) for: генерация треугольных чисел, известный диапазон / for: known range
var sequence = new List<int>(MaxN);
for (int i = 1; i <= MaxN; i++)                       // 1..MaxN включительно / inclusive
{
    sequence.Add(i * (i + 1) / 2);                    // T(i) = i*(i+1)/2
}
Console.WriteLine($"for: sequence = [{string.Join(", ", sequence)}]");

// 2) while: шаги Коллатца, число итераций неизвестно / while: unknown count
static int CollatzSteps(int n)
{
    if (n < 1) return -1;                             // правило не определено / undefined
    int steps = 0;
    int guard = 0;                                    // предохранитель / safety counter
    while (n != 1 && guard++ < 100_000)               // тело продвигает условие / body advances
    {
        n = (n % 2 == 0) ? n / 2 : 3 * n + 1;
        steps++;
    }
    return n == 1 ? steps : -1;                       // -1 если не сошлось / -1 if not converged
}

// 3) foreach: заполнение и обход словаря / foreach: fill and walk dictionary
var stepsByValue = new Dictionary<int, int>();
foreach (var value in sequence)                       // обход коллекции / walk collection
{
    stepsByValue[value] = CollatzSteps(value);
}
Console.WriteLine("foreach: steps by value:");
foreach (var kvp in stepsByValue)                     // сразу ключ+значение / key+value together
{
    Console.WriteLine($"  T={kvp.Key,4} -> steps={kvp.Value}");
}

// Проверочные assert-ы / asserts
static void Assert(bool cond, string msg)
{
    if (!cond) Console.WriteLine($"FAIL: {msg}");
}
Assert(CollatzSteps(1) == 0, "Collatz(1)==0");
Assert(CollatzSteps(2) == 1, "Collatz(2)==1");
Assert(CollatzSteps(6) == 8, "Collatz(6)==8");
Assert(CollatzSteps(0) == -1, "Collatz(0) undefined");
Assert(CollatzSteps(-5) == -1, "Collatz(-5) undefined");

// 4) do-while: меню, минимум один показ / do-while: menu, at least once
int choice;
do
{
    Console.WriteLine("Меню / Menu:");
    Console.WriteLine("  1) Показать все / Show all");
    Console.WriteLine("  2) Удалить чётные / Remove even (RemoveAll)");
    Console.WriteLine("  3) Удалить кратные 5 / Remove multiples of 5 (reverse for)");
    Console.WriteLine("  4) Выход / Exit");
    Console.Write("> ");
    choice = int.TryParse(Console.ReadLine(), out var v) ? v : 0;

    switch (choice)
    {
        case 1:
            Console.WriteLine($"  current = [{string.Join(", ", sequence)}]");
            break;
        case 2:
            // ✅ RemoveAll — декларативно / declarative
            var copyA = new List<int>(sequence);
            copyA.RemoveAll(n => n % 2 == 0);
            // ✅ обратный for — индексы не сбиваются / reverse for keeps indices valid
            var copyB = new List<int>(sequence);
            for (int i = copyB.Count - 1; i >= 0; i--)
            {
                if (copyB[i] % 2 == 0) copyB.RemoveAt(i);
            }
            // ❌ Запрещено: структурная мутация в foreach → InvalidOperationException
            // foreach (var n in sequence) if (n % 2 == 0) sequence.Remove(n);
            Assert(copyA.SequenceEqual(copyB), "RemoveAll == reverse for");
            Console.WriteLine($"  RemoveAll     = [{string.Join(", ", copyA)}]");
            Console.WriteLine($"  reverse for   = [{string.Join(", ", copyB)}]");
            break;
        case 3:
            var copyC = new List<int>(sequence);
            for (int i = copyC.Count - 1; i >= 0; i--)   // только обратный / reverse only
            {
                if (copyC[i] % 5 == 0) copyC.RemoveAt(i);
            }
            Console.WriteLine($"  no mult5      = [{string.Join(", ", copyC)}]");
            break;
        case 4:
            Console.WriteLine("  bye.");
            break;
        default:
            Console.WriteLine("  неизвестный пункт / unknown option");
            break;
    }
} while (choice is not 4);                             // пока не выбран выход / until exit

// небольшое расширение для сравнения списков / small helper for list equality
static class SeqExt
{
    public static bool SequenceEqual(List<int> a, List<int> b)
    {
        if (a.Count != b.Count) return false;
        for (int i = 0; i < a.Count; i++) if (a[i] != b[i]) return false; // индекс нужен — for
        return true;
    }
}
```

**Разбор по строкам.** `const int MaxN = 20;` — граница вынесена в именованную константу, чтобы убрать «магическое число» из условия цикла (best practice урока). Цикл `for (int i = 1; i <= MaxN; i++)` использует `<=`, потому что верхняя граница включительна: треугольные числа считаются для `i = 1..20`, и `i < MaxN` потерял бы последний элемент — это та самая off-by-one ошибка, которую урок выделяет первой. Тело цикла не меняет `i`, что соответствует правилу «считайте счётчик неизменяемым». Метод `CollatzSteps` начинается с `if (n < 1) return -1;` — для нуля и отрицательных правило Коллатца не определено, и без этой проверки `while` для `0` стал бы бесконечным (`0 → 0`), что урок прямо называет классической ошибкой бесконечного `while`. Внутри `while (n != 1 && guard++ < 100_000)` совмещены два условия: основное (`n != 1`) и предохранительный счётчик `guard`, который гарантирует завершение даже при логическом баге — урок требует «гарантируйте продвижение условия и добавьте предохранительный счётчик итераций». Тело `n = (n % 2 == 0) ? n / 2 : 3 * n + 1;` обязательно переприсваивает `n`, продвигая условие к `false`. Словарь заполняется через `foreach (var value in sequence)` — типичный обход коллекции без индекса, семантика `foreach` из урока. Вывод идёт через `foreach (var kvp in stepsByValue)` с `kvp.Key` и `kvp.Value` — это правильный способ, в отличие от `foreach (var k in dict)` + `dict[k]`, который урок относит к частым ошибкам (лишний lookup, медленнее и грязнее). Меню реализовано `do-while`, потому что оно обязано показаться хотя бы один раз даже при первом вводе «4» — это в точности сценарий «диалог с пользователем, сначала показать меню» из теории урока. Разбор `switch (choice)` с `default` обрабатывает неверный ввод и позволяет `do-while` повторить, что демонстрирует постусловие. Удаление чётных показано двумя способами: `RemoveAll(n => n % 2 == 0)` — декларативный вариант, рекомендованный уроком как самый чистый; и обратный `for (int i = copyB.Count - 1; i >= 0; i--)` с `RemoveAt(i)` — урок явно приводит этот приём, потому что при удалении с конца индексы ещё необработанных элементов не сдвигаются. Закомментированный блок `foreach (var n in sequence) if (...) sequence.Remove(n);` снабжён комментарием про `InvalidOperationException` — это демонстрация запрещённого паттерна, который урок выделяет как «ключевое ограничение `foreach`». Важно: мутация здесь структурная (удаляется элемент), а не значения — именно она запрещена. `Assert(copyA.SequenceEqual(copyB), ...)` через вспомогательный `for` по индексам доказывает эквивалентность двух безопасных способов. Edge-кейсы `MaxN = 0` (пустой список, `for` не выполнится ни разу) и `MaxN = 1` (один элемент) обрабатываются естественно: `for` с `i <= 0` не выполняет тело, а `foreach` по пустой коллекции тоже безопасен. Таким образом, в одном файле последовательно применены все четыре цикла по их семантическому назначению, а каждое «тонкое место» урока — off-by-one, бесконечный `while`, мутация в `foreach`, обратный `for`, `KeyValuePair`, `do-while` для меню — присутствует и явно прокомментировано.

#### Задания на углубление (бонус)
1. Замените словарь `Dictionary<int,int>` на `SortedDictionary<int,int>` или отсортированный список пар и сравните порядок вывода. Объясните, почему `foreach` по словарю не гарантирует порядок.
2. Реализуйте «удаление в два прохода»: первым `foreach` соберите индексы удаляемых элементов в `List<int>`, затем удалите их обратным `for` по этому списку. Сравните читаемость с `RemoveAll`.
3. Добавьте команду меню «5) Свернуть до 1 за минимальное число шагов среди всех T(i)» — используйте `foreach` для поиска минимума и `for` для генерации. Подумайте, уместен ли тут LINQ `MinBy` вместо ручного `foreach` (урок рекомендует LINQ для линейного поиска).
4. Превратите `CollatzSteps` в итератор, который через `yield return` отдаёт промежуточные значения `n`, и обойдите его `foreach`. Сравните с `while`-версией по читаемости.

---

## Statement in English / Постановка на английском

#### Context & motivation
Imagine you are building a small utility, the "Sequence Lab", for a discrete-mathematics department. The instructor wants to show students how numbers behave under recurrent rules and to ask the utility to generate series, count "convergence steps", and filter the results. The utility is a console app, runs locally, uses no databases — pure C# 12 / .NET 8. The important thing is that the code is written "the grown-up way": each loop kind is used exactly where it fits semantically, not "because it compiles". That means: generation with a known element count is a `for`; an algorithm whose step count is unknown up front is a `while`; a menu that must appear at least once is a `do-while`; and traversal of a ready collection without indices is a `foreach`. Additionally, you must demonstrate safe techniques for removing elements from a `List<T>`, because the naive attempt to delete even numbers inside a `foreach` will throw `InvalidOperationException` — exactly the error the lesson calls classic. The "meat" of the task is the Collatz conjecture: for any natural `n`, the sequence `n → n/2` (if even) or `3n+1` (if odd) converges to 1. The number of steps to reach 1 is unknown in advance — an ideal proving ground for `while`. So one small app ties all four loops and the lesson's key mistakes into a single coherent scenario rather than disjoint examples.

#### What to do step by step
1. Create the console project: `dotnet new console -n SequenceLab -o SequenceLab -f net8.0`, enter the `SequenceLab` directory, and replace the contents of `Program.cs` with the solution below. Verify the SDK with `dotnet --version` — expect 8.0.x.
2. In `Program.cs` (top-level statements), declare a constant `MaxN = 20` — the upper bound of the generated series. Put it in a named constant, not a "magic number" in the loop condition, as the lesson's best practice requires.
3. Generate the first `MaxN` triangular numbers `T(i) = i*(i+1)/2` for `i` from `1` to `MaxN` inclusive using a `for` loop. The upper bound is inclusive — consciously choose `<=` and justify it. Store the numbers in a `List<int> sequence`.
4. Implement `int CollatzSteps(int n)`, which counts steps to reach `1` using a `while`. The body must move the condition toward `false`: on each iteration `n` shrinks when even and strictly changes when odd. Add a safety iteration cap (say, 100 000) so that an accidental `0` or a negative input cannot cause an infinite loop — the lesson warns about this directly.
5. Build a `Dictionary<int, int> stepsByValue` where the key is the triangular number and the value is the Collatz step count. Fill the dictionary via `foreach` over `sequence`. Print the result with `foreach (var kvp in stepsByValue)` — using `kvp.Key` and `kvp.Value` directly, not `dict[k]` (the lesson names this a common mistake).
6. Implement the menu with `do-while`: it must print the items `(1) Show all, (2) Remove even, (3) Remove multiples of 5, (4) Exit` at least once, then read the choice and handle it. Use a `switch` with pattern matching.
7. The "Remove even" command must work two ways for demonstration: on a copy of the list via `RemoveAll(n => n % 2 == 0)`, and on another copy via a reverse `for` with `RemoveAt(i)` from `Count-1` down to `0`. Print both results and make sure they match. Show the wrong `foreach` variant commented out, with a note that it would throw `InvalidOperationException`.
8. After each operation, print the updated collection via `foreach` with `string.Join(", ", ...)` formatting.
9. Run: `dotnet run`. Feed the inputs `1` (show), `2` (remove even), `3` (remove multiples of 5), `4` (exit). Record the expected output.
10. Add in-place check calls right in `Program.cs` (no separate test project): assertions that `CollatzSteps(1) == 0`, `CollatzSteps(2) == 1`, `CollatzSteps(6) == 8`, and that the reverse `for` and `RemoveAll` produce the same list. On mismatch, print `FAIL`.
11. Test edge cases: an empty sequence (`MaxN = 0`), `MaxN = 1`, and passing `n = 1` to `CollatzSteps` (the `while` body must not run at all).

#### Requirements
The solution must compile without warnings (nullable enabled; `TreatWarningsAsErrors` optional). Use only C# 12 / .NET 8 features: top-level statements, pattern matching (`is < 1 or > 10`), collection expressions where appropriate; raw string literals are not required — interpolation is enough. Each of the four loops must be used exactly for its semantic purpose: `for` — generation with a known range; `while` — Collatz; `do-while` — menu; `foreach` — collection and dictionary traversal. The `for` bounds must be named through a constant or variable, with no magic numbers. The `while` must have a safety iteration cap. Conditional removal is done safely (`RemoveAll` + reverse `for`), and the forbidden `foreach` variant is shown in a comment with an explanation. The dictionary is walked through `KeyValuePair`. All output is informative and prefixed with the operation name. The code reads top to bottom without jumps and without redundant flags. Variable names are meaningful (`stepsByValue`, not `d`). The solution must not replace an explicit loop with LINQ where the lesson recommends a loop (for example, do not swap `CollatzSteps` for recursion or a LINQ aggregate — a `while` is required).

#### Pitfalls
- **Off-by-one in `for`.** The lesson stresses that `<=` instead of `<` causes an off-by-one error. For triangular numbers with `i` from `1` to `MaxN` inclusive, you need `i <= MaxN`; writing `i < MaxN` drops the last element. Test the `MaxN = 1` boundary.
- **Infinite `while`.** In `CollatzSteps` the body must change `n`. Forgetting the assignment `n = n / 2` or `n = 3 * n + 1` means the loop never ends. The safety counter is insurance against logic bugs and against passing `0` (the rule is undefined for `0` — you need an explicit return or `break`).
- **Mutating a collection inside `foreach`.** Removing an element from a `List<T>` inside `foreach` throws `InvalidOperationException` because the iterator tracks the collection version. Mutating element *values* (object properties) is allowed; structural change is not.
- **Reverse `for` for removal.** Walk from the end (`Count-1` down to `0`), otherwise after `RemoveAt(i)` indices shift and you skip the next element. The lesson shows this technique explicitly.
- **`RemoveAll` is more declarative.** For a simple predicate `RemoveAll` is shorter and not subject to index errors; the reverse `for` is needed when removal comes with side effects.
- **Dictionary via `KeyValuePair`.** `foreach (var k in dict)` + `dict[k]` is a redundant lookup. `foreach (var kvp in dict)` gives key and value at once.
- **`do-while` vs `while`.** The menu must appear at least once — that is `do-while` semantics. If zero iterations were valid, you would need `while`.
- **Changing the `for` counter in the body.** Do not modify `i` inside the body — it throws off the step. Steer flow with `break`/`continue` (the next lesson's topic); here, simply leave the counter alone.
- **Nullable input in the menu.** `Console.ReadLine()` returns `string?`; parsing via `int.TryParse` with `out var v` and a pattern `is < 1 or > 4` handles empty and non-numeric input gracefully.

#### Acceptance criteria
- [ ] The project is created with `dotnet new console -n SequenceLab -f net8.0` and builds with no errors or warnings.
- [ ] `Program.cs` uses C# 12 top-level statements.
- [ ] Triangular-number generation uses a `for` loop with bound `i <= MaxN`.
- [ ] `MaxN` is a named constant, not a magic number in the condition.
- [ ] `CollatzSteps` uses `while` and contains a safety iteration cap.
- [ ] `CollatzSteps(1) == 0`, `CollatzSteps(2) == 1`, `CollatzSteps(6) == 8` are checked and pass.
- [ ] The menu is implemented with `do-while` and shown at least once.
- [ ] Menu choice is parsed with pattern matching (`switch` or `is` patterns).
- [ ] `stepsByValue` is filled and walked via `foreach (var kvp in ...)`.
- [ ] "Remove even" is implemented two ways: `RemoveAll` and reverse `for` — results match.
- [ ] The wrong `foreach` removal variant is present as a comment with an explanation of `InvalidOperationException`.
- [ ] Edge cases `MaxN = 0` and `MaxN = 1` do not throw.
- [ ] Passing `n = 0` to `CollatzSteps` does not cause an infinite loop.
- [ ] Each operation's output is prefixed and readable.
- [ ] No magic numbers appear in loop conditions.

#### Hints (no direct answer)
- Triangular number: `T(i) = i*(i+1)/2`. Think about why `i` from `1` calls for `<= MaxN`.
- In `CollatzSteps`, handle the edge case `n <= 0` separately — the rule is undefined there.
- For the safety guard, use a local `int guard = 0` and a condition `while (n != 1 && guard++ < 100_000)`.
- Reverse `for`: `for (int i = list.Count - 1; i >= 0; i--)`.
- To keep the menu from hanging on bad input, print a hint in the `default` branch and let `do-while` repeat.
- To compare two lists for content equality, print `string.Join` of each or run a `for` over indices.

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8 — top-level statements
// Sequence lab: for + while + do-while + foreach
using System;
using System.Collections.Generic;

const int MaxN = 20;                                  // named bound, no magic number

// 1) for: generate triangular numbers, known range
var sequence = new List<int>(MaxN);
for (int i = 1; i <= MaxN; i++)                       // 1..MaxN inclusive
{
    sequence.Add(i * (i + 1) / 2);                    // T(i) = i*(i+1)/2
}
Console.WriteLine($"for: sequence = [{string.Join(", ", sequence)}]");

// 2) while: Collatz steps, iteration count unknown
static int CollatzSteps(int n)
{
    if (n < 1) return -1;                             // rule undefined
    int steps = 0;
    int guard = 0;                                    // safety counter
    while (n != 1 && guard++ < 100_000)               // body advances condition
    {
        n = (n % 2 == 0) ? n / 2 : 3 * n + 1;
        steps++;
    }
    return n == 1 ? steps : -1;                       // -1 if not converged
}

// 3) foreach: fill and walk the dictionary
var stepsByValue = new Dictionary<int, int>();
foreach (var value in sequence)                       // collection traversal
{
    stepsByValue[value] = CollatzSteps(value);
}
Console.WriteLine("foreach: steps by value:");
foreach (var kvp in stepsByValue)                     // key+value together
{
    Console.WriteLine($"  T={kvp.Key,4} -> steps={kvp.Value}");
}

// In-place asserts
static void Assert(bool cond, string msg)
{
    if (!cond) Console.WriteLine($"FAIL: {msg}");
}
Assert(CollatzSteps(1) == 0, "Collatz(1)==0");
Assert(CollatzSteps(2) == 1, "Collatz(2)==1");
Assert(CollatzSteps(6) == 8, "Collatz(6)==8");
Assert(CollatzSteps(0) == -1, "Collatz(0) undefined");
Assert(CollatzSteps(-5) == -1, "Collatz(-5) undefined");

// 4) do-while: menu, shown at least once
int choice;
do
{
    Console.WriteLine("Menu:");
    Console.WriteLine("  1) Show all");
    Console.WriteLine("  2) Remove even (RemoveAll)");
    Console.WriteLine("  3) Remove multiples of 5 (reverse for)");
    Console.WriteLine("  4) Exit");
    Console.Write("> ");
    choice = int.TryParse(Console.ReadLine(), out var v) ? v : 0;

    switch (choice)
    {
        case 1:
            Console.WriteLine($"  current = [{string.Join(", ", sequence)}]");
            break;
        case 2:
            // ✅ RemoveAll — declarative
            var copyA = new List<int>(sequence);
            copyA.RemoveAll(n => n % 2 == 0);
            // ✅ reverse for — indices stay valid
            var copyB = new List<int>(sequence);
            for (int i = copyB.Count - 1; i >= 0; i--)
            {
                if (copyB[i] % 2 == 0) copyB.RemoveAt(i);
            }
            // ❌ Forbidden: structural mutation in foreach → InvalidOperationException
            // foreach (var n in sequence) if (n % 2 == 0) sequence.Remove(n);
            Assert(copyA.SequenceEqual(copyB), "RemoveAll == reverse for");
            Console.WriteLine($"  RemoveAll     = [{string.Join(", ", copyA)}]");
            Console.WriteLine($"  reverse for   = [{string.Join(", ", copyB)}]");
            break;
        case 3:
            var copyC = new List<int>(sequence);
            for (int i = copyC.Count - 1; i >= 0; i--)   // reverse only
            {
                if (copyC[i] % 5 == 0) copyC.RemoveAt(i);
            }
            Console.WriteLine($"  no mult5      = [{string.Join(", ", copyC)}]");
            break;
        case 4:
            Console.WriteLine("  bye.");
            break;
        default:
            Console.WriteLine("  unknown option");
            break;
    }
} while (choice is not 4);                             // until exit chosen

// small helper for list equality
static class SeqExt
{
    public static bool SequenceEqual(List<int> a, List<int> b)
    {
        if (a.Count != b.Count) return false;
        for (int i = 0; i < a.Count; i++) if (a[i] != b[i]) return false; // index needed → for
        return true;
    }
}
```

**Line-by-line walk-through.** `const int MaxN = 20;` puts the bound into a named constant, removing a "magic number" from the loop condition — a lesson best practice. The loop `for (int i = 1; i <= MaxN; i++)` uses `<=` because the upper bound is inclusive: triangular numbers are computed for `i = 1..20`, and `i < MaxN` would drop the last element — exactly the off-by-one error the lesson puts first. The body never modifies `i`, honoring the rule "treat the counter as immutable". `CollatzSteps` begins with `if (n < 1) return -1;` — the Collatz rule is undefined for zero and negatives, and without this guard the `while` for `0` would be infinite (`0 → 0`), the exact "infinite `while`" mistake the lesson highlights. Inside, `while (n != 1 && guard++ < 100_000)` combines the main condition with a safety counter `guard` that guarantees termination even under a logic bug — the lesson demands "guarantee progress and add a safety iteration counter". The body `n = (n % 2 == 0) ? n / 2 : 3 * n + 1;` reassigns `n`, moving the condition toward `false`. The dictionary is filled by `foreach (var value in sequence)` — a textbook collection traversal with no index, the `foreach` semantics from the lesson. Output uses `foreach (var kvp in stepsByValue)` with `kvp.Key` and `kvp.Value` — the correct way, unlike `foreach (var k in dict)` + `dict[k]`, which the lesson lists as a common mistake (extra lookup, slower and messier). The menu uses `do-while` because it must appear at least once even if the first input is "4" — precisely the "user dialog, show the menu first" scenario from the theory. The `switch (choice)` with `default` handles bad input and lets `do-while` repeat, demonstrating the post-condition. Removing evens is shown two ways: `RemoveAll(n => n % 2 == 0)` — the declarative option the lesson recommends as cleanest; and a reverse `for (int i = copyB.Count - 1; i >= 0; i--)` with `RemoveAt(i)` — the lesson shows this explicitly because removing from the end keeps the indices of not-yet-processed elements valid. The commented block `foreach (var n in sequence) if (...) sequence.Remove(n);` carries a note about `InvalidOperationException` — a demonstration of the forbidden pattern the lesson calls the "key restriction of `foreach`". Note the mutation here is structural (an element is removed), not a value mutation — that is what is forbidden. `Assert(copyA.SequenceEqual(copyB), ...)` via a helper `for` over indices proves the two safe approaches are equivalent. Edge cases `MaxN = 0` (empty list, `for` body never runs) and `MaxN = 1` (one element) are handled naturally: a `for` with `i <= 0` skips the body, and `foreach` over an empty collection is also safe. So a single file applies all four loops for their semantic purposes, and every subtle point of the lesson — off-by-one, infinite `while`, mutation in `foreach`, reverse `for`, `KeyValuePair`, `do-while` for a menu — is present and explicitly commented.

#### Going deeper (bonus)
1. Replace `Dictionary<int,int>` with `SortedDictionary<int,int>` or a sorted list of pairs and compare the output order. Explain why `foreach` over a dictionary does not guarantee order.
2. Implement "two-pass removal": first `foreach` collects the indices of the elements to delete into a `List<int>`, then a reverse `for` deletes them by those indices. Compare readability with `RemoveAll`.
3. Add a menu item "5) Find the T(i) that reaches 1 in the fewest steps" — use `foreach` to find the minimum and `for` to generate. Consider whether LINQ `MinBy` fits better than a hand-rolled `foreach` (the lesson recommends LINQ for linear search).
4. Turn `CollatzSteps` into an iterator that uses `yield return` to emit the intermediate values of `n`, and walk it with `foreach`. Compare readability with the `while` version.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Проект `SequenceLab` собирается без ошибок и предупреждений на .NET 8.
- [ ] (RU) Применены все четыре цикла по семантическому назначению: `for`, `while`, `do-while`, `foreach`.
- [ ] (RU) `CollatzSteps` содержит предохранительный лимит итераций и проходит assert-проверки.
- [ ] (RU) Удаление по условию реализовано безопасно (`RemoveAll` + обратный `for`); запрещённый `foreach`-вариант показан комментарием.
- [ ] (RU) Словарь обходится через `KeyValuePair`.
- [ ] (RU) Edge-кейсы `MaxN = 0`, `MaxN = 1`, `n = 0` не вызывают сбоев.
- [ ] (EN) The `SequenceLab` project builds without errors or warnings on .NET 8.
- [ ] (EN) All four loops are used for their semantic purpose: `for`, `while`, `do-while`, `foreach`.
- [ ] (EN) `CollatzSteps` has a safety iteration cap and passes the assertions.
- [ ] (EN) Conditional removal is safe (`RemoveAll` + reverse `for`); the forbidden `foreach` variant is shown as a comment.
- [ ] (EN) The dictionary is walked via `KeyValuePair`.
- [ ] (EN) Edge cases `MaxN = 0`, `MaxN = 1`, `n = 0` do not crash.

#### Ресурсы / Resources
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/statements/iteration-statements — Iteration statements (for, while, do-while, foreach) / Операторы итерации
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/foreach-in — The foreach, in keyword / Ключевое слово foreach, in
- Microsoft Learn — https://learn.microsoft.com/dotnet/api/system.collections.generic.list-1.removeall — List<T>.RemoveAll
- Wikipedia — https://en.wikipedia.org/wiki/Collatz_conjecture — Collatz conjecture / Гипотеза Коллатца

---
[← К уроку M03-L03](lesson-M03-L03-loops.md) | [⬆ К модулю M03](../README.md) | [Следующее ДЗ →](homework-M03-L04-break-continue.md)
---
