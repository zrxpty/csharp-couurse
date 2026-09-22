---
[← К уроку M03-L07](lesson-M03-L07-overloading-scope.md) | [⬆ К модулю M03](../README.md) | [Предыдущее ДЗ ←](homework-M03-L06-ref-out-in-params.md) | [Следующее ДЗ →](homework-M03-L08-recursion.md)
---

### Домашнее задание M03-L07: Перегрузка методов, область видимости / Homework M03-L07: Method overloading, scope

**Урок / Lesson:** M03-L07
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) Научиться объявлять перегрузки методов, различающиеся числом, типами и порядком параметров; понимать, как компилятор разрешает перегрузки и почему возвращаемый тип в этом не участвует; грамотно работать с областями видимости, избегать теневого перекрытия (shadowing) и применять локальные функции (включая `static`) для вспомогательной логики. (EN) Learn to declare method overloads that differ by parameter count, types, and order; understand how the compiler resolves overloads and why the return type does not participate; work competently with scopes, avoid shadowing, and apply local functions (including `static`) for helper logic.

#### Связь с уроком / Connection to the lesson
(RU) ДЗ закрепляет все ключевые концепции урока M03-L07: три способа различать перегрузки, правила разрешения и неоднозначности (особенно связки `int`/`long`/`double` и `params`), уровни области видимости, теневое перекрытие поля локальной переменной и доступ через `this.`, а также локальные функции с захватом и без него (`static`). Упражнение построено так, чтобы каждое правило из урока пришлось применить сознательно и «нащупать» хотя бы одну ошибку неоднозначности.
(EN) The homework reinforces every key concept of lesson M03-L07: the three ways to distinguish overloads, resolution rules and ambiguities (especially `int`/`long`/`double` combinations and `params`), scope levels, shadowing of a field by a local and access via `this.`, and local functions with and without capture (`static`). The exercise is designed so that each rule from the lesson must be applied deliberately and at least one ambiguity error is encountered.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Вы присоединились к команде, которая пишет небольшую библиотеку отчётности `FormatKit` для внутреннего портала. Библиотека должна превращать «сырые» значения, поступающие из разных источников, в строки, пригодные для вывода в консоль, лог-файл и HTML-виджет. Источники данных разные: счётчики отдаёт целые числа, датчики — дробные, справочник — текстовые метки, планировщик — даты, а агрегатор — произвольные наборы значений. Команде нужен единый, интуитивный API: одно имя метода `Format`, к которому можно обратиться с любым из этих значений, а библиотека сама разберётся, как именно его отформатировать.

Именно здесь пригождается перегрузка методов из урока M03-L07: одно имя `Format` превращается в несколько вариантов с разными списками параметров, и компилятор по типам аргументов выбирает нужный. Но на практике всё оказывается не так гладко: целые и дробные числа смешиваются, появляется соблазн перегрузить по возвращаемому типу (что запрещено), `params` начинает «поглощать» вызовы, а локальные переменные случайно перекрывают поля класса. Чтобы библиотека была надёжной, вам предстоит не просто написать перегрузки, но и спроектировать их так, чтобы разрешение было однозначным, а область видимости — прозрачной. Заодно вы примените локальные функции для вспомогательной логики нормализации строк, которая больше нигде не нужна, и сравните их с лямбда-выражениями.

В результате вы получите компактную, но реалистичную библиотеку, в которой каждое правило урока работает на вас, а не против вас: перегрузки осмысленны, неоднозначности устранены явными приведениями, поля не заслоняются локальными, а вспомогательные функции инкапсулированы внутри методов, где они используются.

#### Что нужно сделать (пошагово)

1. Создайте проект консольного приложения на C# 12 / .NET 8:
   ```bash
   dotnet new console -n FormatKit -o FormatKit --framework net8.0
   cd FormatKit
   ```
   Убедитесь, что в `FormatKit.csproj` указано `<TargetFramework>net8.0</TargetFramework>` и `<LangVersion>latest</LangVersion>` (или `preview`). Откройте `Program.cs` и удалите шаблонный `Console.WriteLine("Hello, World!");`.

2. Реализуйте класс `Formatter` с набором перегрузок метода `Format`, которые различаются **тремя способами** из урока: числом параметров, типами параметров и порядком типов:
   - `string Format(int value)` — форматирует целое с разделителем разрядов (инвариантная культура, например `12_345` → `"12,345"`).
   - `string Format(double value, int precision = 2)` — форматирует дробное с заданным числом знаков.
   - `string Format(string text)` — обрезает длинный текст до 40 символов и добавляет многоточие, если текст длиннее.
   - `string Format(DateTime date, string? pattern = null)` — форматирует дату; если `pattern` равен `null`, используется `"yyyy-MM-dd"`.
   - `string Format(params object[] values)` — «склеивает» произвольный набор значений через `"; "` (используйте существующие перегрузки внутри, чтобы каждое значение форматировалось по своему типу).

3. В `Program.cs` (top-level statements) создайте экземпляр `Formatter` и вызовите каждую перегрузку так, чтобы в выводе были видны все варианты. Вывод должен содержать строки вроде:
   ```
   int      : 12,345
   double   : 3.14159 (2)
   text     : Очень длинный заголовок который не влез...
   date     : 2024-05-01
   many     : 1; 2,5; 2024-05-01; hello
   ```

4. **Спровоцируйте и устраните неоднозначность.** Добавьте (временно) перегрузку `string Format(long value)` рядом с `Format(int)` и `Format(double, int)` и попытайтесь вызвать `fmt.Format(1, 2L)`. Убедитесь, что компилятор выдаёт ошибку неоднозначности (ambiguous call), как описано в уроке. Затем удалите `Format(long)` либо явно приведите аргументы, например `fmt.Format((double)1, (double)2L)`, и добейтесь компиляции. В комментарии в коде объясните, почему возникла неоднозначность и какое правило разрешения из урока здесь работает.

5. Реализуйте класс `ReportBuilder`, у которого есть поле `_counter` (целое) и метод `Append(string label, int delta)`. Внутри метода **объявите локальную переменную с тем же именем, что и поле** (например, `counter`), чтобы продемонстрировать shadowing, а затем исправьте ситуацию двумя способами, как в уроке: (а) переименуйте локальную, (б) обратитесь к полю через `this._counter`. В комментарии укажите, какой способ выбрали и почему.

6. В методе `ReportBuilder.Render()` реализуйте **локальную функцию** `string Pad(string s, int width)`, которая видна только внутри `Render` и дополняет строку пробелами слева. Убедитесь, что её можно объявить после `return` — она всё равно видна во всём теле метода (это частая ошибка из урока: студенты думают, что локальная функция после `return` недоступна).

7. Реализуйте статический метод `TextUtils.Normalize(string text)`, внутри которого объявите **`static` локальную функцию** `char Lower(char c)` (или аналогичную), которая ничего не захватывает из внешнего метода. В комментарии поясните, почему `static` здесь уместен и какой выигрыш даёт (отсутствие аллокаций замыкания, явный запрет на случайный захват). Для сравнения рядом реализуйте обычную локальную функцию (без `static`), которая **захватывает** локальную переменную внешнего метода, и в комментарии опишите разницу.

8. Соберите проект и запустите:
   ```bash
   dotnet build
   dotnet run
   ```
   Убедитесь, что сборка проходит без ошибок и предупреждений уровня error, а вывод соответствует ожидаемому.

#### Требования к решению

- Код должен компилироваться на C# 12 / .NET 8 без error-level предупреждений; использовать top-level statements в `Program.cs`.
- Все перегрузки метода `Format` должны иметь **одно имя**, но различаться числом, типами или порядком параметров. Ни одна пара перегрузок не должна отличаться **только** возвращаемым типом — это запрет из урока, за него компилятор выдаёт ошибку.
- Должна присутствовать хотя бы одна перегрузка с `params` и хотя бы одна с опциональным параметром (`precision = 2`, `pattern = null`). `params`-версия должна идти последней и не должна «поглощать» вызовы, предназначенные конкретным перегрузкам.
- В коде должна быть явно показана и устранена ситуация неоднозначности (например, связка `int`/`long`/`double`) с комментарием-объяснением, опирающимся на правила разрешения из урока (`int` → `int` лучше, чем `int` → `double`/`int` → `long`).
- Класс `ReportBuilder` должен содержать поле и демонстрацию shadowing с исправлением через переименование **или** `this.`; выбранный способ должен быть обоснован в комментарии.
- Должны быть применены локальные функции как минимум двух видов: одна с захватом переменных внешнего метода (замыкание) и одна `static` без захвата. Назначение `static` поясняется в комментарии.
- Имена полей должны использовать префикс `_` (соглашение из best practices урока), а локальные переменные — не перекрывать имена полей без явной необходимости.
- Код должен быть структурирован: классы `Formatter`, `ReportBuilder`, статический `TextUtils` разнесены по смыслу; `Program.cs` только демонстрирует API.

#### Тонкости и подводные камни

- **Возвращаемый тип не участвует в разрешении.** Если вы попытаетесь добавить `int Format(int)` рядом с `string Format(int)`, компилятор сообщит об ошибке уже на этапе объявления, потому что сигнатуры совпадают. Различайте перегрузки параметрами — это единственный легальный способ.
- **Неоднозначность `int` + `long` / `int` + `double`.** Когда есть перегрузки `(int, int)` и `(double, double)`, вызов `Format(1, 2L)` не имеет однозначного победителя: `int`→`double` и `long`→`double` оба допустимы, и ни одна перегрузка не «точнее». Решение — явное приведение обоих аргументов к `double` или добавление конкретной перегрузки `(long, long)`.
- **`params` «поглощает» вызовы.** Перегрузка `Format(params object[])` способна принять почти всё, поэтому конкретные перегрузки могут перестать вызываться. Помещайте `params`-версию последней и оставляйте конкретные перегрузки для частых случаев; иначе вы получите «универсальный» метод, который проигрывает в производительности и читаемости.
- **Опциональные параметры и совместимость.** Значения по умолчанию встраиваются в место вызова (call site), поэтому изменение дефолта ломает совместимость без перекомпиляции вызывающей стороны. Если дефолты могут меняться, предпочтительнее перегрузки.
- **Shadowing поля локальной.** Локальная переменная с именем поля «заслоняет» поле внутри метода; случайная мутация локальной вместо поля — классический баг. Префикс `_` у полей и `this.` для явного доступа — два защитных приёма из урока.
- **Локальная функция после `return`.** Вопреки интуиции, локальная функция видна во всём теле метода независимо от порядка объявления. Не считайте её недоступной только потому, что она стоит ниже `return`.
- **`static` локальная функция.** Запрещает захват переменных внешнего метода; компилятор генерирует более эффективный код (без аллокации замыкания) и защищает от случайных захватов. Используйте её для чистых помощников вроде `Lower(char)`.
- **Локальная функция против лямбды.** Для рекурсивной или выделенной вспомогательной логики внутри метода предпочтительнее локальная функция: она поддерживает рекурсию естественно и не требует делегата.

#### Критерии приёмки

- [ ] Проект `FormatKit` создан на `net8.0`, собирается без error-level предупреждений.
- [ ] В `Program.cs` используются top-level statements.
- [ ] Класс `Formatter` содержит не менее пяти перегрузок `Format`, различающихся числом/типами/порядком параметров.
- [ ] Есть перегрузка с `params` и хотя бы одна с опциональным параметром.
- [ ] Ни одна пара перегрузок не отличается только возвращаемым типом.
- [ ] В коде продемонстрирована и устранена неоднозначность (например, `int`/`long`/`double`) с комментарием, ссылающимся на правило разрешения из урока.
- [ ] Класс `ReportBuilder` содержит поле с префиксом `_` и демонстрацию shadowing с исправлением (переименование или `this.`).
- [ ] В `ReportBuilder.Render()` есть локальная функция, объявленная после `return`, и она корректно вызывается.
- [ ] В `TextUtils.Normalize()` есть `static` локальная функция без захвата с поясняющим комментарием.
- [ ] Рядом есть обычная локальная функция с захватом переменной внешнего метода и комментарием о разнице.
- [ ] Вывод `dotnet run` содержит строки для всех пяти видов значений (int, double, text, date, many).
- [ ] Код структурирован: `Formatter`, `ReportBuilder`, `TextUtils` разделены по смыслу.
- [ ] Имена полей используют префикс `_`; локальные не перекрывают поля без необходимости.
- [ ] В комментариях (RU) объяснён выбор между перегрузками и опциональными параметрами хотя бы в одном месте.
- [ ] Чек-лист сдачи (ниже) заполнен.

#### Подсказки (без прямого ответа)

- Для форматирования целого с разделителем разрядов посмотрите на перегрузки `ToString` с `CultureInfo.InvariantCulture` и строкой формата `"N0"`.
- Для дробного с заданной точностью пригодится `"F"` + число знаков, но помните про опциональный параметр `precision`.
- Чтобы «склеить» набор значений в `params`-перегрузке, не дублируйте логику: вызывайте существующие перегрузки `Format` для каждого элемента. Подумайте, как отличить `int` от `double` от `DateTime` внутри `object[]` (pattern matching `is`).
- Для демонстрации shadowing достаточно объявить локальную с тем же именем, что и поле, и попытаться инкрементировать — посмотрите, что меняется. Потом исправьте через `this.`.
- Чтобы спровоцировать неоднозначность, достаточно добавить перегрузку с `long` и вызвать с `int` и `long` одновременно. Компилятор сам подскажет.
- Для `static` локальной функции выберите чистое преобразование, не зависящее от переменных метода (например, приведение регистра символа).

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8 — top-level statements
// Мини-библиотека FormatKit: перегрузки, область видимости, локальные функции
// Mini FormatKit library: overloads, scope, local functions

using System.Globalization;

Formatter fmt = new();

// Демонстрация всех перегрузок / Demonstrate every overload
Console.WriteLine($"int      : {fmt.Format(12_345)}");
Console.WriteLine($"double   : {fmt.Format(3.14159)}");          // precision = 2 по умолчанию / default precision
Console.WriteLine($"double   : {fmt.Format(3.14159, 4)}");       // явная точность / explicit precision
Console.WriteLine($"text     : {fmt.Format("Очень длинный заголовок который не влезает в строку")}");
Console.WriteLine($"date     : {fmt.Format(new DateTime(2024, 5, 1))}");
Console.WriteLine($"date     : {fmt.Format(new DateTime(2024, 5, 1), "dd.MM.yyyy")}");
Console.WriteLine($"many     : {fmt.Format(1, 2.5, new DateTime(2024, 5, 1), "hello")}");

// Неоднозначность: если бы существовала перегрузка Format(long), вызов ниже был бы неоднозначен.
// Ambiguity: had a Format(long) overload existed, the call below would be ambiguous.
// fmt.Format(1, 2L);  // ❌ потенциально неоднозначно / potentially ambiguous
Console.WriteLine($"resolved : {fmt.Format((double)1, (double)2L)}");  // явное приведение / explicit cast

// ReportBuilder: shadowing и локальная функция после return
// ReportBuilder: shadowing and a local function after return
ReportBuilder rb = new();
rb.Append("orders", 3);
rb.Append("returns", 1);
Console.WriteLine(rb.Render());

// Normalize: static локальная функция без захвата
// Normalize: static local function without capture
Console.WriteLine(TextUtils.Normalize("  Привет, МИР!  "));

class Formatter
{
    // Перегрузки различаются типами и числом параметров; возвращаемый тип — один и тот же.
    // Overloads differ by types and count; return type is the same.
    public string Format(int value) =>
        value.ToString("N0", CultureInfo.InvariantCulture);

    public string Format(double value, int precision = 2) =>
        value.ToString($"F{precision}", CultureInfo.InvariantCulture);

    public string Format(string text) =>
        text.Length <= 40 ? text : string.Concat(text.AsSpan(0, 40), "...");

    public string Format(DateTime date, string? pattern = null) =>
        date.ToString(pattern ?? "yyyy-MM-dd", CultureInfo.InvariantCulture);

    // params-перегрузка идёт последней, чтобы не поглощать конкретные случаи.
    // The params overload goes last so it does not swallow specific cases.
    public string Format(params object[] values) =>
        string.Join("; ", values.Select(FormatOne));

    // Внутренний помощник: выбирает подходящую перегрузку по типу элемента.
    // Internal helper: picks the right overload by element type.
    private string FormatOne(object v) => v switch
    {
        int i      => Format(i),
        double d   => Format(d),
        DateTime dt => Format(dt),
        string s   => Format(s),
        null       => "<null>",
        _          => v.ToString() ?? "<null>"
    };
}

class ReportBuilder
{
    private int _counter;        // поле с префиксом _ / field with _ prefix
    private readonly StringBuilder _sb = new();

    public void Append(string label, int delta)
    {
        // Демонстрация shadowing: если назвать локальную _counter, она заслонит поле.
        // Shadowing demo: naming the local _counter would shadow the field.
        // Исправление — обращение к полю через this._counter, локальную назвали counter.
        // Fix: reach the field via this._counter; the local is named counter.
        int counter = delta;          // локальная / local
        this._counter += counter;     // явно к полю / explicitly to the field
        _sb.AppendLine($"{label}: +{counter}");
    }

    public string Render()
    {
        var lines = _sb.ToString().TrimEnd().Split(Environment.NewLine);
        int maxLen = lines.Max(l => l.Length);

        return string.Join(Environment.NewLine, lines.Select(Pad));

        // Локальная функция объявлена после return, но видна во всём теле метода.
        // Local function declared after return, yet visible across the whole method body.
        string Pad(string s) => s.PadLeft(maxLen);
    }
}

static class TextUtils
{
    public static string Normalize(string text)
    {
        string trimmed = text.Trim();

        // Обычная локальная функция ЗАХВАТЫВАЕТ trimmed — это замыкание.
        // Ordinary local function CAPTURES trimmed — a closure.
        string Join(char[] chars) => new string(chars);

        // static локальная функция ничего не захватывает — эффективнее и безопаснее.
        // static local function captures nothing — more efficient and safer.
        static char Lower(char c) =>
            char.IsUpper(c) ? char.ToLower(c, CultureInfo.InvariantCulture) : c;

        char[] lowered = trimmed.Select(Lower).ToArray();
        return Join(lowered);
    }
}
```

Разбор по строкам. Класс `Formatter` демонстрирует три способа различать перегрузки: `Format(int)` и `Format(double, int)` различаются типами, `Format(DateTime, string?)` добавляет второй параметр (число), а порядок типов виден в том, что `Format(string)` и `Format(DateTime, string?)` имеют разный «костяк» параметров. Возвращаемый тип у всех — `string`, и это сознательно: урок прямо запрещает различать перегрузки только возвращаемым типом, потому что компилятору неоткуда узнать, какую версию выбрать в месте вызова. Опциональные параметры `precision = 2` и `pattern = null` дают удобные дефолты, но мы помним из урока, что дефолты встраиваются в call site и ломают совместимость при изменении — поэтому для меняющихся значений лучше перегрузки.

`params`-перегрузка `Format(params object[])` помещена последней и опирается на внутренний помощник `FormatOne`, который через pattern matching `switch` выбирает конкретную перегрузку по типу элемента. Это решает проблему из урока: `params` не «поглощает» вызовы конкретных перегрузок, потому что для `int`, `double`, `DateTime` и `string` компилятор выберет точные варианты, а `params` сработает только когда аргументов несколько и типов смешаны.

Строка `fmt.Format((double)1, (double)2L)` иллюстрирует устранение неоднозначности. Если бы рядом стояла перегрузка `Format(long)`, вызов с `int` и `long` не имел бы однозначного победителя (правило из урока: `int`→`int` точнее `int`→`double`, но `long`→`long` против `long`→`double` даёт ничью). Явное приведение обоих аргументов к `double` подсказывает компилятору выбор.

В `ReportBuilder` поле названо `_counter` — это соглашение из best practices урока, снижающее риск shadowing. Внутри `Append` локальная `counter` могла бы заслонить поле, если бы имела то же имя; мы избегаем этого переименованием локальной и явным доступом `this._counter` к полю. Метод `Render` содержит локальную функцию `Pad`, объявленную **после** `return` — это осознанная демонстрация правила из урока: локальные функции видны во всём теле метода независимо от порядка.

В `TextUtils.Normalize` соседствуют две локальные функции: `Join` захватывает `trimmed` (замыкание), а `Lower` объявлена `static` и ничего не захватывает. Урок рекомендует `static` для чистых помощников: компилятор не создаёт замыкание, код эффективнее, а случайный захват переменной внешнего метода становится ошибкой компиляции. Так каждое правило урока работает на читаемость и надёжность кода.

#### Задания на углубление (бонус)

1. Добавьте перегрузку `Format(TimeSpan value)` и перегрузку `Format(DateTime date, TimeSpan time)`, различающуюся **порядком типов**. Убедитесь, что разрешение однозначно.
2. Замените внутренний помощник `FormatOne` на обобщённый метод `FormatOne<T>(T v)` и попробуйте вызвать `Format` с пользовательским типом — проследите, как дженерики порождают сюрпризы (как предупреждает урок).
3. Реализуйте рекурсивную локальную функцию (например, для вычисления факториала внутри метода) и сравните её с рекурсивной лямбдой `Func<int,int>` — убедитесь, что локальная функция поддерживает рекурсию естественно, как сказано в уроке.
4. Покройте `Formatter` модульными тестами на xUnit, проверяя, что каждая перегрузка выбирается корректно; в тесте на `params` убедитесь, что конкретные перегрузки имеют приоритет.

---

## Statement in English / Постановка на английском

#### Context & motivation

You have joined a team building a small reporting library called `FormatKit` for an internal portal. The library must turn "raw" values coming from different sources into strings suitable for the console, a log file, and an HTML widget. The sources vary: counters return integers, sensors return floating-point numbers, a reference table returns text labels, a scheduler returns dates, and an aggregator returns arbitrary bundles of values. The team wants a single, intuitive API: one method name, `Format`, that you can call with any of these values, while the library figures out how exactly to render it.

This is exactly where method overloading from lesson M03-L07 pays off: a single name `Format` becomes several variants with different parameter lists, and the compiler selects the right one from the argument types. In practice, though, things are not so smooth: integers and doubles mix, there is a temptation to overload by return type (which is forbidden), `params` starts "swallowing" calls, and local variables accidentally shadow class fields. To make the library robust, you must not merely write the overloads but design them so that resolution is unambiguous and the scope is transparent. Along the way you will apply local functions for helper string-normalization logic that is needed nowhere else, and compare them with lambda expressions.

The result is a compact but realistic library in which every rule from the lesson works for you rather than against you: overloads are meaningful, ambiguities are resolved with explicit casts, fields are not shadowed by locals, and helper functions are encapsulated inside the methods that use them.

#### What to do step by step

1. Create a C# 12 / .NET 8 console application:
   ```bash
   dotnet new console -n FormatKit -o FormatKit --framework net8.0
   cd FormatKit
   ```
   Make sure `FormatKit.csproj` contains `<TargetFramework>net8.0</TargetFramework>` and `<LangVersion>latest</LangVersion>` (or `preview`). Open `Program.cs` and remove the boilerplate `Console.WriteLine("Hello, World!");`.

2. Implement a `Formatter` class with a set of `Format` method overloads that differ in the **three ways** described in the lesson: by parameter count, by parameter types, and by the order of types:
   - `string Format(int value)` — formats an integer with a thousands separator (invariant culture, e.g. `12_345` → `"12,345"`).
   - `string Format(double value, int precision = 2)` — formats a double with the given number of digits.
   - `string Format(string text)` — truncates long text to 40 characters and appends an ellipsis if it is longer.
   - `string Format(DateTime date, string? pattern = null)` — formats a date; if `pattern` is `null`, use `"yyyy-MM-dd"`.
   - `string Format(params object[] values)` — joins an arbitrary set of values with `"; "` (reuse the existing overloads internally so each value is formatted by its own type).

3. In `Program.cs` (top-level statements), create a `Formatter` instance and call every overload so that all variants are visible in the output. The output should contain lines such as:
   ```
   int      : 12,345
   double   : 3.14159 (2)
   text     : A very long title that simply does not fit on o...
   date     : 2024-05-01
   many     : 1; 2.5; 2024-05-01; hello
   ```

4. **Provoke and resolve an ambiguity.** Temporarily add an overload `string Format(long value)` next to `Format(int)` and `Format(double, int)`, then try to call `fmt.Format(1, 2L)`. Confirm that the compiler raises an "ambiguous call" error, as described in the lesson. Then either remove `Format(long)` or cast the arguments explicitly, for example `fmt.Format((double)1, (double)2L)`, and make it compile. In a code comment, explain why the ambiguity arose and which resolution rule from the lesson applies.

5. Implement a `ReportBuilder` class that has a field `_counter` (integer) and a method `Append(string label, int delta)`. Inside the method, **declare a local variable with the same name as the field** (for instance `counter`) to demonstrate shadowing, then fix the situation in two ways as in the lesson: (a) rename the local, (b) reach the field through `this._counter`. In a comment, state which option you chose and why.

6. In `ReportBuilder.Render()`, implement a **local function** `string Pad(string s, int width)` that is visible only inside `Render` and left-pads a string with spaces. Verify that it can be declared after `return` and is still callable — this is the common mistake from the lesson: students assume a local function after `return` is unreachable.

7. Implement a static method `TextUtils.Normalize(string text)` that contains a **`static` local function** `char Lower(char c)` (or similar) which captures nothing from the outer method. In a comment, explain why `static` is appropriate here and what benefit it brings (no closure allocation, an explicit ban on accidental captures). For comparison, place next to it an ordinary local function (without `static`) that **captures** a local variable of the outer method, and comment on the difference.

8. Build and run the project:
   ```bash
   dotnet build
   dotnet run
   ```
   Confirm that the build succeeds with no error-level warnings and the output matches expectations.

#### Requirements

- The code must compile on C# 12 / .NET 8 with no error-level warnings, using top-level statements in `Program.cs`.
- All `Format` overloads share **one name** but differ by parameter count, types, or order. No pair of overloads may differ **only** by return type — this is the lesson's prohibition, for which the compiler raises an error.
- There must be at least one `params` overload and at least one overload with an optional parameter (`precision = 2`, `pattern = null`). The `params` version must come last and must not "swallow" calls intended for specific overloads.
- The code must explicitly show and resolve an ambiguity situation (for example an `int`/`long`/`double` combination) with an explanatory comment grounded in the lesson's resolution rules (`int` → `int` beats `int` → `double` and `int` → `long`).
- `ReportBuilder` must contain a field and a shadowing demonstration fixed by renaming **or** `this.`; the chosen option must be justified in a comment.
- Local functions of at least two kinds must be used: one that captures variables of the outer method (closure) and one `static` function without capture. The purpose of `static` must be explained in a comment.
- Field names must use the `_` prefix (the convention from the lesson's best practices), and local variables must not shadow field names unless necessary.
- The code must be structured: `Formatter`, `ReportBuilder`, and the static `TextUtils` are separated by intent; `Program.cs` only demonstrates the API.

#### Pitfalls

- **The return type does not participate in resolution.** If you try to add `int Format(int)` next to `string Format(int)`, the compiler errors out at declaration time because the signatures coincide. Differentiate overloads by parameters — that is the only legal way.
- **The `int` + `long` / `int` + `double` ambiguity.** With `(int, int)` and `(double, double)` overloads present, a call `Format(1, 2L)` has no clear winner: `int`→`double` and `long`→`double` are both admissible, and neither overload is "more precise". The fix is to cast both arguments to `double` explicitly or to add a concrete `(long, long)` overload.
- **`params` "swallows" calls.** The `Format(params object[])` overload can accept almost anything, so specific overloads may stop being selected. Put the `params` version last and keep concrete overloads for the common cases; otherwise you get a "universal" method that loses on performance and readability.
- **Optional parameters and compatibility.** Default values are baked into the call site, so changing a default breaks compatibility without recompiling the caller. When defaults may change, prefer overloads.
- **Shadowing a field with a local.** A local that shares a field's name shadows the field inside the method; mutating the local instead of the field is a classic bug. The `_` prefix on fields and `this.` for explicit access are the two protective techniques from the lesson.
- **A local function after `return`.** Contrary to intuition, a local function is visible across the entire method body regardless of declaration order. Do not assume it is unreachable just because it sits below `return`.
- **A `static` local function.** It forbids capturing the outer method's variables; the compiler emits more efficient code (no closure allocation) and guards against accidental captures. Use it for pure helpers such as `Lower(char)`.
- **Local function vs lambda.** For recursive or dedicated helper logic inside a method, a local function is preferable: it supports recursion naturally and needs no delegate.

#### Acceptance criteria

- [ ] The `FormatKit` project is created on `net8.0` and builds with no error-level warnings.
- [ ] `Program.cs` uses top-level statements.
- [ ] The `Formatter` class contains at least five `Format` overloads differing by count/types/order.
- [ ] There is a `params` overload and at least one overload with an optional parameter.
- [ ] No pair of overloads differs only by return type.
- [ ] An ambiguity (e.g. `int`/`long`/`double`) is demonstrated and resolved with a comment referencing the lesson's resolution rule.
- [ ] `ReportBuilder` contains a `_`-prefixed field and a shadowing demonstration fixed by renaming or `this.`.
- [ ] `ReportBuilder.Render()` contains a local function declared after `return` and called correctly.
- [ ] `TextUtils.Normalize()` contains a `static` local function without capture and an explanatory comment.
- [ ] Next to it is an ordinary local function that captures a variable of the outer method, with a comment on the difference.
- [ ] The `dotnet run` output contains lines for all five value kinds (int, double, text, date, many).
- [ ] The code is structured: `Formatter`, `ReportBuilder`, and `TextUtils` are separated by intent.
- [ ] Field names use the `_` prefix; locals do not shadow fields without need.
- [ ] A comment (EN) explains the choice between overloads and optional parameters in at least one place.
- [ ] The submission checklist (below) is filled in.

#### Hints (without giving the answer away)

- For an integer with a thousands separator, look at `ToString` overloads with `CultureInfo.InvariantCulture` and the `"N0"` format string.
- For a double with a given precision, the `"F"` format plus the digit count works, but mind the optional `precision` parameter.
- To join values in the `params` overload without duplicating logic, call the existing `Format` overloads for each element. Think about how to tell `int` from `double` from `DateTime` inside `object[]` (pattern matching with `is`).
- To demonstrate shadowing, declare a local with the same name as the field and try to increment it — see what changes. Then fix it with `this.`.
- To provoke an ambiguity, add a `long` overload and call with `int` and `long` at once. The compiler will tell you.
- For the `static` local function, pick a pure transformation that does not depend on the method's variables (for instance, changing a character's case).

#### Reference solution walk-through

```csharp
// C# 12 / .NET 8 — top-level statements
// Mini FormatKit library: overloads, scope, local functions

using System.Globalization;

Formatter fmt = new();

// Demonstrate every overload
Console.WriteLine($"int      : {fmt.Format(12_345)}");
Console.WriteLine($"double   : {fmt.Format(3.14159)}");          // default precision
Console.WriteLine($"double   : {fmt.Format(3.14159, 4)}");       // explicit precision
Console.WriteLine($"text     : {fmt.Format("A very long title that simply does not fit on one line")}");
Console.WriteLine($"date     : {fmt.Format(new DateTime(2024, 5, 1))}");
Console.WriteLine($"date     : {fmt.Format(new DateTime(2024, 5, 1), "dd.MM.yyyy")}");
Console.WriteLine($"many     : {fmt.Format(1, 2.5, new DateTime(2024, 5, 1), "hello")}");

// Ambiguity: had a Format(long) overload existed, the call below would be ambiguous.
// fmt.Format(1, 2L);  // ❌ potentially ambiguous
Console.WriteLine($"resolved : {fmt.Format((double)1, (double)2L)}");  // explicit cast

// ReportBuilder: shadowing and a local function after return
ReportBuilder rb = new();
rb.Append("orders", 3);
rb.Append("returns", 1);
Console.WriteLine(rb.Render());

// Normalize: static local function without capture
Console.WriteLine(TextUtils.Normalize("  Hello, WORLD!  "));

class Formatter
{
    // Overloads differ by types and count; the return type is the same.
    public string Format(int value) =>
        value.ToString("N0", CultureInfo.InvariantCulture);

    public string Format(double value, int precision = 2) =>
        value.ToString($"F{precision}", CultureInfo.InvariantCulture);

    public string Format(string text) =>
        text.Length <= 40 ? text : string.Concat(text.AsSpan(0, 40), "...");

    public string Format(DateTime date, string? pattern = null) =>
        date.ToString(pattern ?? "yyyy-MM-dd", CultureInfo.InvariantCulture);

    // The params overload goes last so it does not swallow specific cases.
    public string Format(params object[] values) =>
        string.Join("; ", values.Select(FormatOne));

    // Internal helper: picks the right overload by element type.
    private string FormatOne(object v) => v switch
    {
        int i      => Format(i),
        double d   => Format(d),
        DateTime dt => Format(dt),
        string s   => Format(s),
        null       => "<null>",
        _          => v.ToString() ?? "<null>"
    };
}

class ReportBuilder
{
    private int _counter;        // field with _ prefix
    private readonly StringBuilder _sb = new();

    public void Append(string label, int delta)
    {
        // Shadowing demo: naming the local _counter would shadow the field.
        // Fix: reach the field via this._counter; the local is named counter.
        int counter = delta;          // local
        this._counter += counter;     // explicitly to the field
        _sb.AppendLine($"{label}: +{counter}");
    }

    public string Render()
    {
        var lines = _sb.ToString().TrimEnd().Split(Environment.NewLine);
        int maxLen = lines.Max(l => l.Length);

        return string.Join(Environment.NewLine, lines.Select(Pad));

        // Local function declared after return, yet visible across the whole method body.
        string Pad(string s) => s.PadLeft(maxLen);
    }
}

static class TextUtils
{
    public static string Normalize(string text)
    {
        string trimmed = text.Trim();

        // Ordinary local function CAPTURES trimmed — a closure.
        string Join(char[] chars) => new string(chars);

        // static local function captures nothing — more efficient and safer.
        static char Lower(char c) =>
            char.IsUpper(c) ? char.ToLower(c, CultureInfo.InvariantCulture) : c;

        char[] lowered = trimmed.Select(Lower).ToArray();
        return Join(lowered);
    }
}
```

Walk-through, line by line. The `Formatter` class demonstrates the three ways to distinguish overloads: `Format(int)` and `Format(double, int)` differ by types, `Format(DateTime, string?)` adds a second parameter (count), and the order of types is visible because `Format(string)` and `Format(DateTime, string?)` have different parameter "skeletons". The return type is `string` for all of them — deliberately, because the lesson forbids differentiating overloads by return type alone: the compiler would have no basis to pick a version at the call site. Optional parameters `precision = 2` and `pattern = null` provide convenient defaults, but we keep in mind from the lesson that defaults are baked into the call site and break compatibility when changed — so for values that may change, overloads are preferable.

The `params` overload `Format(params object[])` is placed last and leans on the internal helper `FormatOne`, which uses a `switch` pattern to select the concrete overload by element type. This solves the lesson's problem: `params` does not "swallow" calls to concrete overloads, because for `int`, `double`, `DateTime`, and `string` the compiler picks the exact variants, and `params` only fires when there are several arguments of mixed types.

The line `fmt.Format((double)1, (double)2L)` illustrates ambiguity resolution. Had an overload `Format(long)` stood nearby, a call with `int` and `long` would have no clear winner (the lesson's rule: `int`→`int` is more precise than `int`→`double`, but `long`→`long` against `long`→`double` is a tie). Casting both arguments to `double` guides the compiler's choice.

In `ReportBuilder`, the field is named `_counter` — the convention from the lesson's best practices that lowers the risk of shadowing. Inside `Append`, the local `counter` could shadow the field if it shared the name; we avoid that by renaming the local and reaching the field explicitly via `this._counter`. The `Render` method contains a local function `Pad` declared **after** `return` — an intentional demonstration of the lesson's rule: local functions are visible across the entire method body regardless of order.

In `TextUtils.Normalize`, two local functions sit side by side: `Join` captures `trimmed` (a closure), while `Lower` is declared `static` and captures nothing. The lesson recommends `static` for pure helpers: the compiler creates no closure, the code is more efficient, and an accidental capture of an outer-method variable becomes a compile error. Thus every rule from the lesson contributes to readability and reliability.

#### Going deeper (bonus)

1. Add a `Format(TimeSpan value)` overload and a `Format(DateTime date, TimeSpan time)` overload that differ by the **order of types**. Confirm the resolution is unambiguous.
2. Replace the internal helper `FormatOne` with a generic method `FormatOne<T>(T v)` and try calling `Format` with a custom type — observe how generics breed surprises, as the lesson warns.
3. Implement a recursive local function (for example, factorial inside a method) and compare it with a recursive `Func<int,int>` lambda — confirm that the local function supports recursion naturally, as the lesson states.
4. Cover `Formatter` with xUnit unit tests verifying that each overload is selected correctly; in the `params` test, ensure concrete overloads take priority.

---

#### Чек-лист сдачи / Submission checklist
- [ ] Проект `FormatKit` собирается на `net8.0` без error-level предупреждений.
- [ ] `Program.cs` использует top-level statements.
- [ ] Класс `Formatter` содержит ≥5 перегрузок `Format`, различающихся числом/типами/порядком.
- [ ] Есть `params`-перегрузка и перегрузка с опциональным параметром.
- [ ] Нет пары перегрузок, различающейся только возвращаемым типом.
- [ ] Неоднозначность (`int`/`long`/`double`) продемонстрирована и устранена с комментарием.
- [ ] `ReportBuilder` содержит поле с префиксом `_` и демонстрацию shadowing с исправлением.
- [ ] В `Render()` есть локальная функция после `return`, корректно вызываемая.
- [ ] В `TextUtils.Normalize()` есть `static` локальная функция без захвата с комментарием.
- [ ] Рядом есть локальная функция с захватом и комментарием о разнице.
- [ ] Вывод `dotnet run` содержит строки для всех пяти видов значений.
- [ ] The `FormatKit` project builds on `net8.0` with no error-level warnings.
- [ ] `Program.cs` uses top-level statements.
- [ ] `Formatter` contains ≥5 `Format` overloads differing by count/types/order.
- [ ] A `params` overload and an overload with an optional parameter are present.
- [ ] No pair of overloads differs only by return type.
- [ ] An ambiguity (`int`/`long`/`double`) is demonstrated and resolved with a comment.
- [ ] `ReportBuilder` contains a `_`-prefixed field and a shadowing demonstration fixed.
- [ ] `Render()` contains a local function after `return` that is called correctly.
- [ ] `TextUtils.Normalize()` contains a `static` local function without capture, commented.
- [ ] A capturing local function sits next to it with a comment on the difference.
- [ ] The `dotnet run` output contains lines for all five value kinds.

#### Ресурсы / Resources
- [Microsoft Learn — Methods: overloading](https://learn.microsoft.com/dotnet/csharp/methods#overloading)
- [Microsoft Learn — Local functions](https://learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/local-functions)
- [Microsoft Learn — Scopes (C# reference)](https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/scope)
- [Microsoft Learn — `params` modifier](https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/params)
- [Microsoft Learn — Optional parameters / Named and optional arguments](https://learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/named-and-optional-arguments)

---
[← К уроку M03-L07](lesson-M03-L07-overloading-scope.md) | [⬆ К модулю M03](../README.md) | [Предыдущее ДЗ ←](homework-M03-L06-ref-out-in-params.md) | [Следующее ДЗ →](homework-M03-L08-recursion.md)
---
