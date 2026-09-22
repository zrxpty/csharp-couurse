---
[← К уроку M04-L06](lesson-M04-L06-this-indexers.md) | [⬆ К модулю M04](../README.md) | [Следующее ДЗ →](homework-M04-L07-record-basics.md)
---

### Домашнее задание M04-L06: this, индексаторы / Homework M04-L06: this, indexers

**Урок / Lesson:** M04-L06
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться применять ключевое слово `this` для различения полей и параметров, вызова одного конструктора из другого, построения fluent API и передачи экземпляра внешнему методу, а также проектировать индексаторы с проверкой границ, перегрузкой по типу параметра, асимметричными модификаторами доступа и несколькими параметрами. (EN) Learn to use the `this` keyword to distinguish fields from parameters, chain constructors, build fluent APIs and pass the instance to external methods, and design indexers with bounds validation, overloading by parameter type, asymmetric access modifiers and multiple parameters.

#### Связь с уроком / Connection to the lesson
(RU) Урок вводит `this` как ссылку на текущий экземпляр и показывает четыре сценария: устранение неоднозначности имён, передача экземпляра, возврат `this` для цепочек вызовов и объявление индексаторов. Отдельно разбирается делегирование конструкторов через `: this(...)` и правила для индексаторов: безымянность, обязательная принадлежность экземпляру, перегрузка по типу параметра, асимметричные модификаторы и многопараметровые варианты. Это ДЗ закрепляет все эти конструкции на одном связном примере — табличной сетке `Grid`.
(EN) The lesson introduces `this` as a reference to the current instance and shows four scenarios: resolving name ambiguity, passing the instance, returning `this` for call chains, and declaring indexers. It also covers constructor delegation via `: this(...)` and the indexer rules: namelessness, mandatory instance membership, overloading by parameter type, asymmetric modifiers, and multi-parameter variants. This homework consolidates all of these on a single cohesive example — a tabular `Grid`.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Вы разрабатываете небольшую библиотеку для работы с табличными данными, аналог мини-таблицы, в которой каждая ячейка хранит целое число. Такая структура нужна во многих учебных задачах: тепловые карты, игровые поля, простые растровые изображения, хранение результатов замеров. Важно, чтобы клиентский код обращался к ячейкам максимально естественно — так же, как к обычному двумерному массиву `arr[i, j]`, — но при этом получал безопасность: проверку границ, осмысленные исключения, защиту от несанкционированной записи и удобные способы адресации.

Параллельно вам нужно спроектировать API так, чтобы конфигурацию объекта можно было описывать цепочкой вызовов, как это принято в современных библиотеках: `new Grid(3, 3).SetName("A").Set(0, 0, 1).Set(1, 1, 2)`. Это требует грамотного использования `this` как возвращаемого значения. Наконец, в библиотеке появляется второй способ адресации — по строковому имени ячейки в стиле электронных таблиц (`"A1"`, `"B3"`), что демонстрирует перегрузку индексатора по типу параметра.

В этой задаче сходятся сразу все темы урока: `this` для различения поля и параметра, делегирование конструкторов через `this(...)`, возврат `this` для fluent API, передача `this` внешнему методу (например, сериализатору), целочисленный индексатор с проверкой границ, перегрузка индексатора строковым ключом, асимметричные модификаторы доступа (`get` публичный, `set` приватный) и многопараметровый индексатор `this[int row, int col]`. Вы построите всё это в одном классе и убедитесь, что каждая конструкция работает именно так, как описано в уроке.

#### Что нужно сделать (пошагово)
1. Создайте проект консольного приложения: `dotnet new console -n GridLab -o GridLab --framework net8.0`. Перейдите в каталог `GridLab` и откройте `Program.cs`. Удалите шаблонный код и оставьте только `using System;` и `using System.Globalization;`.
2. В файле `Grid.cs` (или прямо в `Program.cs` для компактности) объявите класс `Grid` с приватными полями: `private readonly int _rows;`, `private readonly int _cols;`, `private readonly int[,] _cells;` и `private string _name;`. Обратите внимание: для `readonly`-полей присвоение возможно только в конструкторе или в инициализаторе, поэтому в setter индексатора вы будете менять `_cells[i, j]`, а не сам массив.
3. Реализуйте три конструктора с делегированием через `this(...)`:
   - основной: `public Grid(int rows, int cols, string name)` — в нём проверьте `rows > 0 && cols > 0`, иначе бросьте `ArgumentOutOfRangeException`, присвойте поля, выделите `_cells = new int[rows, cols];`; используйте `this.` только если имя параметра совпадает с полем;
   - `public Grid(int rows, int cols) : this(rows, cols, "Untitled")` — делегирует основному;
   - `public Grid() : this(3, 3)` — дефолтная сетка 3×3. Помните: `: this(...)` обязан стоять в строке объявления, до тела.
4. Реализуйте fluent-методы, возвращающие `this`:
   - `public Grid SetName(string name) { _name = name; return this; }` — обратите внимание, что `_name` не `readonly`, поэтому менять можно;
   - `public Grid Set(int row, int col, int value)` — проверяет границы и записывает значение; возвращает `this` для цепочки;
   - `public Grid Fill(int value)` — заполняет все ячейки одним значением через двойной цикл, возвращает `this`.
5. Реализуйте целочисленный индексатор `public int this[int row, int col]` с `get` и `set`. В обоих аксессорах проверяйте `row` и `col` и выбрасывайте `IndexOutOfRangeException` с понятным сообщением вида `"Bad index: row=5, col=2 (grid 3x3)"`. В `set` используйте неявный параметр `value` — его не нужно объявлять, компилятор предоставляет его сам. Внутри `get` не должно быть никаких побочных эффектов: только чтение.
6. Реализуйте перегрузку индексатора по строковому ключу `public int this[string cellRef]` — только `get` (только чтение). Принимайте ссылки вида `"A1"` (столбец буквой A–Z, строка числом с 1). Разбор строки: первая литера — столбец (`'A'` → 0, `'B'` → 1), остальные цифры — номер строки минус один. При некорректном формате бросайте `FormatException`, при выходе за границы — `IndexOutOfRangeException`. Это демонстрирует перегрузку индексатора по типу параметра, как в уроке с `this[int]` и `this[string]`.
7. Сделайте асимметричный модификатор доступа: публичный индексатор `this[int, int]` с `public get` и `private set` — внешние клиенты смогут читать ячейки через `grid[1, 2]`, но менять только через метод `Set`. Это повторяет приём `public int this[int i] { get; private set; }` из урока.
8. Реализуйте метод `public void PrintTo(TextWriter writer)`, который выводит сетку в переданный `writer`. Внутри вызовите внешний вспомогательный метод `GridSerializer.Serialize(this, writer);` — так вы продемонстрируете передачу `this` другому методу. Сам `GridSerializer` сделайте отдельным `static` классом с методом `public static void Serialize(Grid grid, TextWriter writer)`.
9. В `Program.cs` создайте сетку цепочкой, запишите значения через `Set` и через индексатор, прочитайте через целочисленный индексатор и через строковый ключ, вызовите `PrintTo(Console.Out)` и убедитесь, что вывод корректный. Добавьте блок `try/catch`, демонстрирующий реакцию на выход за границы.
10. Соберите и запустите: `dotnet build` затем `dotnet run --project GridLab`. Зафиксируйте ожидаемый вывод в комментарии в начале `Program.cs`.

#### Требования к решению
- Целевая платформа: .NET 8, язык C# 12. Можно использовать top-level statements в `Program.cs`, collection expressions и pattern matching там, где это уместно, но основная логика класса `Grid` должна быть классической и читаемой.
- Класс `Grid` должен содержать ровно три конструктора с корректным делегированием через `this(...)`. Основной конструктор — единственное место, где проверяются границы `rows`/`cols` и выделяется массив. Остальные делегируют ему.
- Все fluent-методы обязаны возвращать `this`, чтобы клиентский код мог строить цепочки любой длины. Метод `SetName` должен принимать параметр с тем же именем, что и поле `_name` (через параметр `name`), и присваивать `this._name = name;` — это задействует `this` для устранения неоднозначности.
- Целочисленный индексатор `this[int row, int col]` обязан проверять границы в обоих аксессорах и выбрасывать `IndexOutOfRangeException` с информативным сообщением, включающим размер сетки и значение индекса. `get` должен быть без побочных эффектов.
- Строковый индексатор `this[string cellRef]` должен быть только для чтения и корректно разбирать ссылки `"A1"`..`"Z99"`. Это иллюстрирует перегрузку индексатора по типу параметра.
- Асимметричный доступ `public get / private set` для целочисленного индексатора — обязателен: прямая запись `grid[1, 2] = 5;` из внешнего кода должна приводить к ошибке компиляции, а чтение `int x = grid[1, 2];` — работать.
- Передача `this` внешнему методу `GridSerializer.Serialize(this, writer)` обязательна внутри `PrintTo`. Сами данные `Grid` при этом не должны утечь наружу: сериализатор читает ячейки только через публичный индексатор.
- Код должен компилироваться без предупреждений (уровень `TreatWarningsAsErrors` приветствуется, но не обязателен) и проходить все проверки из критериев приёмки.

#### Тонкости и подводные камни
- **`this` в `static` недоступен.** Метод `GridSerializer.Serialize` статический, поэтому внутри него нельзя писать `this` — обращайтесь к переданному параметру `grid`. Обратное правило: внутри индексаторов и экземплярных методов `this` всегда доступен и ссылается на текущий экземпляр.
- **Делегирование конструктора — до тела, в строке объявления.** Конструкция `: this(...)` обязана стоять сразу после списка параметров и до открывающей скобки тела. Если поставить её внутри тела — будет ошибка компиляции. Тело текущего конструктора выполнится *после* вызванного.
- **Параметр `value` неявный.** В `set` индексатора и свойства `value` не объявляется — компилятор подставляет его сам с типом, совпадающим с типом индексатора. Объявлять `int value` вручную нельзя.
- **`get` без побочных эффектов.** В уроке подчёркнуто: `get` индексатора не должен ничего менять в состоянии. Любая модификация внутри `get` нарушает ожидание «чтение как у массива» и ломает отладку.
- **`readonly`-поля и setter.** `_rows`, `_cols` и сам массив `_cells` объявлены `readonly`. Вы можете менять содержимое массива (`_cells[i, j] = x`), но не можете переприсвоить саму ссылку `_cells = ...` вне конструктора. Это частая путаница: `readonly` замораживает ссылку, а не содержимое.
- **Перегрузка индексатора по типу параметра.** Компилятор различает `grid[1, 2]` и `grid["A1"]` по типам аргументов. Но перегрузки с близкими типами (например, `this[int]` и `this[long]`) запутывают читателя — урок предостерегает от этого. В нашем случае `int` и `string` радикально различны, что безопасно.
- **Асимметричный доступ и fluent-метод.** Если у индексатора `private set`, прямое присваивание `grid[1, 2] = 5` из `Program.cs` не скомпилируется — именно поэтому нужен метод `Set`, который живёт внутри класса и потому имеет доступ к `private set`.
- **Индексаторы не имеют имени и не могут быть `static`.** Их нельзя вызвать по имени вроде `grid.Item(1,2)` и нельзя объявить `static`. Это ограничение прямо из урока.

#### Критерии приёмки
- [ ] Проект `GridLab` создаётся командой `dotnet new console` и собирается без ошибок под .NET 8 / C# 12.
- [ ] В классе `Grid` ровно три конструктора, и только основной содержит логику проверки и выделения массива.
- [ ] Делегирование реализовано через `: this(...)` в строке объявления, до тела конструктора.
- [ ] Метод `SetName(string name)` присваивает `this._name = name;`, демонстрируя использование `this` для устранения неоднозначности.
- [ ] Методы `SetName`, `Set`, `Fill` возвращают `this` и позволяют строить цепочки вызовов.
- [ ] Целочисленный индексатор `this[int row, int col]` проверяет границы и бросает `IndexOutOfRangeException` с информативным сообщением в обоих аксессорах.
- [ ] В `set` индексатора используется неявный параметр `value`, без ручного объявления.
- [ ] `get` целочисленного индексатора не имеет побочных эффектов.
- [ ] Целочисленный индексатор имеет `public get` и `private set` — прямая запись из `Program.cs` не компилируется.
- [ ] Реализована перегрузка `this[string cellRef]` — только `get`, корректно разбирает `"A1"`..`"Z99"`.
- [ ] Строковый индексатор бросает `FormatException` на некорректном формате и `IndexOutOfRangeException` на выходе за границы.
- [ ] Метод `PrintTo(TextWriter writer)` вызывает внешний статический метод `GridSerializer.Serialize(this, writer)`, передавая `this`.
- [ ] `GridSerializer` объявлен как отдельный `static` класс и не использует `this` внутри.
- [ ] `Program.cs` демонстрирует: цепочку fluent-вызовов, чтение через `int`-индексатор, чтение через `string`-индексатор, корректный вывод и `try/catch` для выхода за границы.
- [ ] `dotnet run` выводит ожидаемый результат, зафиксированный в комментарии в начале `Program.cs`.

#### Подсказки (без прямого ответа)
- Для разбора `"A1"`: столбец = `char.ToUpper(cellRef[0]) - 'A'`; строка = `int.Parse(cellRef[1..]) - 1`. Не забудьте проверить длину строки.
- Для проверки границ пригодится локальная функция `bool InBounds(int r, int c) => r >= 0 && r < _rows && c >= 0 && c < _cols;`.
- Чтобы сделать `private set` у индексатора, пишите модификатор прямо перед аксессором: `public int this[int r, int c] { get { ... } private set { ... } }`.
- Помните, что `return this;` в fluent-методе возвращает *тот же* экземпляр, а не копию — иначе цепочка сломается.
- Для передачи `this` внешнему методу достаточно `GridSerializer.Serialize(this, writer);` — здесь `this` выступает как аргумент.

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 — this, индексаторы / this, indexers
using System;
using System.Globalization;
using System.IO;

// Класс Grid: демонстрирует this для полей, this() делегирование,
// fluent API через return this, перегрузку индексаторов и асимметричные модификаторы.
// Class Grid: demonstrates this for fields, this() delegation,
// fluent API via return this, indexer overloads and asymmetric access modifiers.
public class Grid
{
    private readonly int _rows;          // число строк / row count
    private readonly int _cols;          // число столбцов / column count
    private readonly int[,] _cells;      // двумерный массив ячеек / 2D cell array
    private string _name;                // имя сетки, можно менять / grid name, mutable

    // Основной конструктор — единственное место с проверкой и выделением
    // Main constructor — the only place with validation and allocation
    public Grid(int rows, int cols, string name)
    {
        if (rows <= 0 || cols <= 0)
            throw new ArgumentOutOfRangeException(
                $"rows={rows}, cols={cols} — размер должен быть положительным / size must be positive");
        _rows = rows;                    // this не нужен: имена различаются / this not needed: names differ
        _cols = cols;
        _cells = new int[rows, cols];
        _name = name ?? "Untitled";
    }

    // Делегирование основному конструктору / Delegate to the main constructor
    public Grid(int rows, int cols) : this(rows, cols, "Untitled") { }

    // Дефолтная сетка 3x3 / Default 3x3 grid
    public Grid() : this(3, 3) { }

    // Fluent: имя параметра совпадает с полем — нужен this
    // Fluent: parameter name matches the field — this is required
    public Grid SetName(string name)
    {
        this._name = name;               // this устраняет неоднозначность / this resolves ambiguity
        return this;                     // возвращаем себя для цепочки / return self for chaining
    }

    // Fluent-метод с проверкой границ / Fluent method with bounds check
    public Grid Set(int row, int col, int value)
    {
        if (!InBounds(row, col))
            throw new IndexOutOfRangeException($"Bad index: row={row}, col={col} (grid {_rows}x{_cols})");
        _cells[row, col] = value;        // меняем содержимое, а не ссылку / mutate content, not reference
        return this;
    }

    public Grid Fill(int value)
    {
        for (int r = 0; r < _rows; r++)
            for (int c = 0; c < _cols; c++)
                _cells[r, c] = value;
        return this;
    }

    // Многопараметровый индексатор с асимметричным доступом
    // Multi-parameter indexer with asymmetric access
    public int this[int row, int col]
    {
        get                              // без побочных эффектов / side-effect free
        {
            if (!InBounds(row, col))
                throw new IndexOutOfRangeException($"Bad index: row={row}, col={col} (grid {_rows}x{_cols})");
            return _cells[row, col];
        }
        private set                      // запись только изнутри класса / write only from inside
        {
            if (!InBounds(row, col))
                throw new IndexOutOfRangeException($"Bad index: row={row}, col={col} (grid {_rows}x{_cols})");
            _cells[row, col] = value;    // value — неявный параметр / value is the implicit param
        }
    }

    // Перегрузка по строковому ключу — только чтение
    // Overload by string key — read only
    public int this[string cellRef]
    {
        get
        {
            if (string.IsNullOrEmpty(cellRef) || cellRef.Length < 2)
                throw new FormatException($"Некорректная ссылка / Bad cell ref: '{cellRef}'");
            int col = char.ToUpper(cellRef[0], CultureInfo.InvariantCulture) - 'A';
            if (!int.TryParse(cellRef[1..], NumberStyles.None, CultureInfo.InvariantCulture, out int rowParsed))
                throw new FormatException($"Некорректная строка / Bad row part: '{cellRef[1..]}'");
            int row = rowParsed - 1;
            if (!InBounds(row, col))
                throw new IndexOutOfRangeException($"Bad index: row={row}, col={col} (grid {_rows}x{_cols})");
            return _cells[row, col];
        }
    }

    public int Rows => _rows;
    public int Cols => _cols;
    public string Name => _name;

    // Передаём this внешнему методу / Pass this to an external method
    public void PrintTo(TextWriter writer) => GridSerializer.Serialize(this, writer);

    private bool InBounds(int r, int c) => r >= 0 && r < _rows && c >= 0 && c < _cols;

    public override string ToString() => $"{_name}: {_rows}x{_cols}";
}

// Статический класс-сериализатор — внутри НЕЛЬЗЯ использовать this
// Static serializer class — this CANNOT be used inside
public static class GridSerializer
{
    public static void Serialize(Grid grid, TextWriter writer)
    {
        writer.WriteLine(grid.ToString());
        for (int r = 0; r < grid.Rows; r++)
        {
            for (int c = 0; c < grid.Cols; c++)
                writer.Write($"{grid[r, c],4}");   // читаем через публичный индексатор / read via public indexer
            writer.WriteLine();
        }
    }
}

// Точка входа через top-level statements / Entry point via top-level statements
var grid = new Grid(3, 3)
    .SetName("Heatmap")
    .Set(0, 0, 1)
    .Set(1, 1, 2)
    .Set(2, 2, 3);

grid.PrintTo(Console.Out);                  // передаём this внутри PrintTo / pass this inside PrintTo
Console.WriteLine(grid[1, 1]);              // 2 — чтение через int-индексатор / read via int indexer
Console.WriteLine(grid["B2"]);              // 2 — чтение через string-индексатор / read via string indexer
Console.WriteLine(grid);                    // Heatmap: 3x3

try
{
    _ = grid[5, 0];
}
catch (IndexOutOfRangeException ex)
{
    Console.WriteLine($"Поймано / Caught: {ex.Message}");
}
```

Разбор по строкам и применённые концепции урока. Поле `_rows`, `_cols` и `_cells` объявлены `readonly`: это иллюстрирует важную тонкость урока — `readonly` замораживает ссылку, а не содержимое, поэтому в `Set` и в `set` индексатора мы меняем элементы `_cells[row, col]`, а не саму переменную `_cells`. Основной конструктор `Grid(int rows, int cols, string name)` — единственное место с проверкой `rows > 0 && cols > 0` и выделением `new int[rows, cols]`. Имена параметров здесь совпадают с именами полей только частично (`rows` vs `_rows`), поэтому `this` не требуется — это демонстрирует рекомендацию урока «не злоупотреблять `this.`». Два других конструктора делегируют основному через `: this(...)` в строке объявления, до тела, как требует урок. Конструкция `: this(rows, cols, "Untitled")` означает: сначала выполнится основной конструктор, потом — пустое тело текущего.

Метод `SetName(string name)` намеренно использует параметр с тем же семантическим именем, что и поле, и присваивает `this._name = name;` — классический приём из урока для устранения неоднозначности. Возврат `this` в `SetName`, `Set` и `Fill` включает fluent API: `new Grid(3,3).SetName(...).Set(...).Set(...)`. Это третья типовая роль `this` из урока — возврат объекта из метода.

Целочисленный индексатор `this[int row, int col]` — многопараметровый (четвёртая форма из урока) с проверкой границ через `InBounds`. В `set` используется неявный параметр `value` (его не нужно объявлять — урок подчёркивает это явно). Асимметричный модификатор `private set` повторяет приём `public int this[int i] { get; private set; }`: внешняя запись `grid[1, 1] = 5;` из `Program.cs` не скомпилируется, а метод `Set`, живущий внутри класса, имеет полный доступ. `get` намеренно лишён побочных эффектов — это требование урока: чтение `obj[i]` должно быть дёшево и предсказуемо.

Перегрузка `this[string cellRef]` с тем же именем `this`, но другим типом параметра показывает перегрузку индексатора по типу параметра — как `this[int]` и `this[string]` в `Bookshelf` из урока. Она сделана только для чтения (`get`-only), что соответствует рекомендации урока «предпочитайте `get`-only или `private set`, если внешняя мутация не входит в контракт». Разбор `"A1"`/`"B2"` через `char.ToUpper` и `int.TryParse` — практическое применение `System.Globalization`.

Метод `PrintTo(TextWriter writer)` вызывает `GridSerializer.Serialize(this, writer);` — это вторая роль `this` из урока: передача текущего экземпляра другому методу. `GridSerializer` объявлен `static`-классом: внутри него `this` недоступен (первая частая ошибка урока), и он читает ячейки только через публичный индексатор `grid[r, c]`, не получая доступа к приватным полям — это безопасная инкапсуляция. Точка входа использует top-level statements (C# 12 / .NET 8) и показывает обе формы чтения — через `int` и через `string` — плюс блок `try/catch`, демонстрирующий осмысленное исключение при выходе за границы. Каждая концепция урока задействована: `this` для полей, `this(...)` для делегирования, `return this` для fluent, передача `this` внешнему методу, индексатор `this[int, int]`, перегрузка `this[string]`, асимметричный `private set`, проверка границ, неявный `value`, запрет `static`-индексаторов и побочных эффектов в `get`.

#### Задания на углубление (бонус)
1. Добавьте третий индексатор `this[(int row, int col) tuple]`, принимающий кортеж, и поясните, почему компилятор различает его от `this[int, int]` (подсказка: тип параметра отличается).
2. Реализуйте `Span<int>`-представление строки через метод `RowSpan(int row)`, используя `System.Runtime.InteropServices.MemoryMarshal` или ручную проекцию, и обсудите, почему индексатор `get` должен оставаться без побочных эффектов для потокобезопасного чтения.
3. Сделайте `Grid` реализацией интерфейса `IMatrix` с явной реализацией индексатора (explicit interface implementation) и сравните с обычной публичной реализацией: когда явная реализация удобнее?
4. Добавьте кэширование результатов строкового индексатора в `Dictionary<string, int>` и оцените, не нарушает ли это правило «`get` без побочных эффектов» — аргументируйте, является ли ленивое кэширование допустимым исключением.

---

## Statement in English / English statement

#### Context & motivation
You are building a small library for tabular data — a miniature spreadsheet where each cell holds an integer. Such a structure is needed in many educational tasks: heat maps, game boards, simple raster images, storing measurement results. It is important that client code addresses cells as naturally as a regular two-dimensional array `arr[i, j]`, while at the same time gaining safety: bounds checking, meaningful exceptions, protection from unauthorized writes, and convenient addressing.

At the same time you need to design the API so that object configuration can be expressed as a call chain, as is customary in modern libraries: `new Grid(3, 3).SetName("A").Set(0, 0, 1).Set(1, 1, 2)`. This requires correct use of `this` as a return value. Finally, the library gains a second addressing mode — by a string cell name in the spreadsheet style (`"A1"`, `"B3"`), which demonstrates indexer overloading by parameter type.

This task brings together every topic of the lesson: `this` to distinguish a field from a parameter, constructor delegation via `this(...)`, returning `this` for a fluent API, passing `this` to an external method (for example, a serializer), an integer indexer with bounds checking, an indexer overload by a string key, asymmetric access modifiers (`public get`, `private set`), and a multi-parameter indexer `this[int row, int col]`. You will build all of this in a single class and verify that each construct behaves exactly as described in the lesson.

#### What to do step by step
1. Create a console application project: `dotnet new console -n GridLab -o GridLab --framework net8.0`. Move into the `GridLab` directory and open `Program.cs`. Remove the template code and keep only `using System;` and `using System.Globalization;`.
2. In `Grid.cs` (or directly in `Program.cs` for compactness) declare a `Grid` class with private fields: `private readonly int _rows;`, `private readonly int _cols;`, `private readonly int[,] _cells;` and `private string _name;`. Notice that `readonly` fields can be assigned only in a constructor or initializer, so in the indexer setter you will mutate `_cells[i, j]`, not the array reference itself.
3. Implement three constructors with delegation via `this(...)`:
   - the main one: `public Grid(int rows, int cols, string name)` — inside, check `rows > 0 && cols > 0`, otherwise throw `ArgumentOutOfRangeException`, assign the fields, allocate `_cells = new int[rows, cols];`; use `this.` only when the parameter name matches the field;
   - `public Grid(int rows, int cols) : this(rows, cols, "Untitled")` — delegates to the main one;
   - `public Grid() : this(3, 3)` — a default 3×3 grid. Remember: `: this(...)` must appear on the declaration line, before the body.
4. Implement fluent methods returning `this`:
   - `public Grid SetName(string name) { _name = name; return this; }` — note that `_name` is not `readonly`, so it can be changed;
   - `public Grid Set(int row, int col, int value)` — validates bounds, writes the value, returns `this` for chaining;
   - `public Grid Fill(int value)` — fills every cell with one value using a nested loop, returns `this`.
5. Implement the integer indexer `public int this[int row, int col]` with `get` and `set`. In both accessors validate `row` and `col` and throw `IndexOutOfRangeException` with a clear message such as `"Bad index: row=5, col=2 (grid 3x3)"`. In `set` use the implicit parameter `value` — you must not declare it; the compiler provides it. Inside `get` there must be no side effects: only a read.
6. Implement an indexer overload by string key `public int this[string cellRef]` — `get` only (read-only). Accept references like `"A1"` (column as a letter A–Z, row as a number starting from 1). Parsing: the first character is the column (`'A'` → 0, `'B'` → 1), the remaining digits are the row number minus one. On a malformed format throw `FormatException`, on out-of-range access throw `IndexOutOfRangeException`. This demonstrates indexer overloading by parameter type, exactly as in the lesson with `this[int]` and `this[string]`.
7. Apply an asymmetric access modifier: a public indexer `this[int, int]` with `public get` and `private set` — external clients can read cells via `grid[1, 2]`, but can mutate them only through the `Set` method. This mirrors the `public int this[int i] { get; private set; }` trick from the lesson.
8. Implement `public void PrintTo(TextWriter writer)` that prints the grid to the supplied `writer`. Inside, call an external helper `GridSerializer.Serialize(this, writer);` — this demonstrates passing `this` to another method. Make `GridSerializer` a separate `static` class with `public static void Serialize(Grid grid, TextWriter writer)`.
9. In `Program.cs` create a grid using a chain, write values through `Set` and through the indexer, read through the integer indexer and through the string key, call `PrintTo(Console.Out)`, and confirm the output is correct. Add a `try/catch` block demonstrating the reaction to an out-of-bounds access.
10. Build and run: `dotnet build` then `dotnet run --project GridLab`. Record the expected output in a comment at the top of `Program.cs`.

#### Requirements
- Target platform: .NET 8, language C# 12. You may use top-level statements in `Program.cs`, collection expressions, and pattern matching where appropriate, but the core logic of the `Grid` class should remain classical and readable.
- The `Grid` class must contain exactly three constructors with correct delegation via `this(...)`. The main constructor is the single place where `rows`/`cols` are validated and the array is allocated. The others delegate to it.
- All fluent methods must return `this` so that client code can build chains of arbitrary length. The `SetName` method must take a parameter named identically to the field (a parameter `name`) and assign `this._name = name;` — this activates `this` for disambiguation.
- The integer indexer `this[int row, int col]` must validate bounds in both accessors and throw `IndexOutOfRangeException` with an informative message that includes the grid size and the index value. `get` must be side-effect free.
- The string indexer `this[string cellRef]` must be read-only and correctly parse references `"A1"`..`"Z99"`. This illustrates indexer overloading by parameter type.
- The asymmetric access `public get / private set` for the integer indexer is mandatory: a direct write `grid[1, 2] = 5;` from external code must fail to compile, while a read `int x = grid[1, 2];` must work.
- Passing `this` to the external method `GridSerializer.Serialize(this, writer)` is mandatory inside `PrintTo`. The grid data must not leak: the serializer reads cells only through the public indexer.
- The code must compile without warnings (setting `TreatWarningsAsErrors` is welcome but optional) and pass every check in the acceptance criteria.

#### Pitfalls
- **`this` is unavailable in `static`.** The `GridSerializer.Serialize` method is static, so you cannot write `this` inside it — use the passed `grid` parameter. The reverse rule holds: inside indexers and instance methods `this` is always available and refers to the current instance.
- **Constructor delegation goes before the body, on the declaration line.** The `: this(...)` construct must appear immediately after the parameter list and before the opening brace of the body. Putting it inside the body is a compile error. The current constructor body runs *after* the delegated one.
- **The `value` parameter is implicit.** In an indexer or property `set`, `value` is not declared — the compiler injects it with the indexer's type. You cannot declare `int value` manually.
- **`get` must be side-effect free.** The lesson stresses that an indexer `get` must not mutate state. Any mutation inside `get` breaks the "reads like an array" expectation and complicates debugging.
- **`readonly` fields and the setter.** `_rows`, `_cols`, and the `_cells` array itself are declared `readonly`. You can change array contents (`_cells[i, j] = x`) but cannot reassign the reference `_cells = ...` outside a constructor. A common confusion: `readonly` freezes the reference, not the contents.
- **Indexer overload by parameter type.** The compiler distinguishes `grid[1, 2]` from `grid["A1"]` by argument types. But overloads with close types (e.g. `this[int]` and `this[long]`) confuse readers — the lesson warns against this. Here `int` and `string` are radically different, which is safe.
- **Asymmetric access and the fluent method.** If the indexer has a `private set`, a direct assignment `grid[1, 2] = 5` from `Program.cs` will not compile — that is exactly why the `Set` method exists, living inside the class and therefore having access to `private set`.
- **Indexers have no name and cannot be `static`.** You cannot call them by name like `grid.Item(1,2)`, and you cannot declare them `static`. This restriction comes straight from the lesson.

#### Acceptance criteria
- [ ] The `GridLab` project is created with `dotnet new console` and builds without errors on .NET 8 / C# 12.
- [ ] The `Grid` class has exactly three constructors, and only the main one contains validation and array allocation logic.
- [ ] Delegation is implemented via `: this(...)` on the declaration line, before the constructor body.
- [ ] The `SetName(string name)` method assigns `this._name = name;`, demonstrating `this` for disambiguation.
- [ ] The `SetName`, `Set`, and `Fill` methods return `this` and enable call chains.
- [ ] The integer indexer `this[int row, int col]` validates bounds and throws `IndexOutOfRangeException` with an informative message in both accessors.
- [ ] The indexer `set` uses the implicit `value` parameter, without a manual declaration.
- [ ] The integer indexer `get` has no side effects.
- [ ] The integer indexer has `public get` and `private set` — a direct write from `Program.cs` does not compile.
- [ ] An overload `this[string cellRef]` is implemented — `get` only, correctly parsing `"A1"`..`"Z99"`.
- [ ] The string indexer throws `FormatException` on a malformed format and `IndexOutOfRangeException` on out-of-bounds access.
- [ ] The `PrintTo(TextWriter writer)` method calls the external static method `GridSerializer.Serialize(this, writer)`, passing `this`.
- [ ] `GridSerializer` is declared as a separate `static` class and does not use `this` inside.
- [ ] `Program.cs` demonstrates: a fluent call chain, a read through the `int` indexer, a read through the `string` indexer, correct output, and a `try/catch` for out-of-bounds access.
- [ ] `dotnet run` prints the expected result recorded in a comment at the top of `Program.cs`.

#### Hints (no direct answer)
- To parse `"A1"`: column = `char.ToUpper(cellRef[0]) - 'A'`; row = `int.Parse(cellRef[1..]) - 1`. Do not forget to check the string length.
- For bounds checking, a local function helps: `bool InBounds(int r, int c) => r >= 0 && r < _rows && c >= 0 && c < _cols;`.
- To make the indexer setter `private`, place the modifier directly on the accessor: `public int this[int r, int c] { get { ... } private set { ... } }`.
- Remember that `return this;` in a fluent method returns *the same* instance, not a copy — otherwise the chain breaks.
- To pass `this` to an external method, simply write `GridSerializer.Serialize(this, writer);` — here `this` is an argument.

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8 — this, indexers
using System;
using System.Globalization;
using System.IO;

// Class Grid: demonstrates this for fields, this() delegation,
// fluent API via return this, indexer overloads and asymmetric access modifiers.
public class Grid
{
    private readonly int _rows;          // row count
    private readonly int _cols;          // column count
    private readonly int[,] _cells;      // 2D cell array
    private string _name;                // grid name, mutable

    // Main constructor — the only place with validation and allocation
    public Grid(int rows, int cols, string name)
    {
        if (rows <= 0 || cols <= 0)
            throw new ArgumentOutOfRangeException(
                $"rows={rows}, cols={cols} — size must be positive");
        _rows = rows;                    // this not needed: names differ
        _cols = cols;
        _cells = new int[rows, cols];
        _name = name ?? "Untitled";
    }

    // Delegate to the main constructor
    public Grid(int rows, int cols) : this(rows, cols, "Untitled") { }

    // Default 3x3 grid
    public Grid() : this(3, 3) { }

    // Fluent: parameter name matches the field — this is required
    public Grid SetName(string name)
    {
        this._name = name;               // this resolves ambiguity
        return this;                     // return self for chaining
    }

    // Fluent method with bounds check
    public Grid Set(int row, int col, int value)
    {
        if (!InBounds(row, col))
            throw new IndexOutOfRangeException($"Bad index: row={row}, col={col} (grid {_rows}x{_cols})");
        _cells[row, col] = value;        // mutate content, not reference
        return this;
    }

    public Grid Fill(int value)
    {
        for (int r = 0; r < _rows; r++)
            for (int c = 0; c < _cols; c++)
                _cells[r, c] = value;
        return this;
    }

    // Multi-parameter indexer with asymmetric access
    public int this[int row, int col]
    {
        get                              // side-effect free
        {
            if (!InBounds(row, col))
                throw new IndexOutOfRangeException($"Bad index: row={row}, col={col} (grid {_rows}x{_cols})");
            return _cells[row, col];
        }
        private set                      // write only from inside
        {
            if (!InBounds(row, col))
                throw new IndexOutOfRangeException($"Bad index: row={row}, col={col} (grid {_rows}x{_cols})");
            _cells[row, col] = value;    // value is the implicit param
        }
    }

    // Overload by string key — read only
    public int this[string cellRef]
    {
        get
        {
            if (string.IsNullOrEmpty(cellRef) || cellRef.Length < 2)
                throw new FormatException($"Bad cell ref: '{cellRef}'");
            int col = char.ToUpper(cellRef[0], CultureInfo.InvariantCulture) - 'A';
            if (!int.TryParse(cellRef[1..], NumberStyles.None, CultureInfo.InvariantCulture, out int rowParsed))
                throw new FormatException($"Bad row part: '{cellRef[1..]}'");
            int row = rowParsed - 1;
            if (!InBounds(row, col))
                throw new IndexOutOfRangeException($"Bad index: row={row}, col={col} (grid {_rows}x{_cols})");
            return _cells[row, col];
        }
    }

    public int Rows => _rows;
    public int Cols => _cols;
    public string Name => _name;

    // Pass this to an external method
    public void PrintTo(TextWriter writer) => GridSerializer.Serialize(this, writer);

    private bool InBounds(int r, int c) => r >= 0 && r < _rows && c >= 0 && c < _cols;

    public override string ToString() => $"{_name}: {_rows}x{_cols}";
}

// Static serializer class — this CANNOT be used inside
public static class GridSerializer
{
    public static void Serialize(Grid grid, TextWriter writer)
    {
        writer.WriteLine(grid.ToString());
        for (int r = 0; r < grid.Rows; r++)
        {
            for (int c = 0; c < grid.Cols; c++)
                writer.Write($"{grid[r, c],4}");   // read via public indexer
            writer.WriteLine();
        }
    }
}

// Entry point via top-level statements
var grid = new Grid(3, 3)
    .SetName("Heatmap")
    .Set(0, 0, 1)
    .Set(1, 1, 2)
    .Set(2, 2, 3);

grid.PrintTo(Console.Out);                  // pass this inside PrintTo
Console.WriteLine(grid[1, 1]);              // 2 — read via int indexer
Console.WriteLine(grid["B2"]);              // 2 — read via string indexer
Console.WriteLine(grid);                    // Heatmap: 3x3

try
{
    _ = grid[5, 0];
}
catch (IndexOutOfRangeException ex)
{
    Console.WriteLine($"Caught: {ex.Message}");
}
```

Line-by-line walk-through and lesson concepts applied. The fields `_rows`, `_cols`, and `_cells` are declared `readonly`: this illustrates an important lesson subtlety — `readonly` freezes the reference, not the contents, so in `Set` and in the indexer setter we mutate elements `_cells[row, col]` rather than the variable `_cells` itself. The main constructor `Grid(int rows, int cols, string name)` is the only place with the `rows > 0 && cols > 0` check and the `new int[rows, cols]` allocation. The parameter names only partially overlap with the field names (`rows` vs `_rows`), so `this` is not required — this demonstrates the lesson recommendation "do not overuse `this.`". The other two constructors delegate to the main one via `: this(...)` on the declaration line, before the body, exactly as the lesson requires. The construct `: this(rows, cols, "Untitled")` means: the main constructor runs first, then the empty body of the current one.

The `SetName(string name)` method deliberately uses a parameter with the same semantic name as the field and assigns `this._name = name;` — the classic lesson technique for disambiguation. Returning `this` in `SetName`, `Set`, and `Fill` enables the fluent API: `new Grid(3,3).SetName(...).Set(...).Set(...)`. This is the third typical role of `this` from the lesson — returning the object from a method.

The integer indexer `this[int row, int col]` is multi-parameter (the fourth form from the lesson) with bounds validation through `InBounds`. In `set` the implicit parameter `value` is used (it must not be declared — the lesson stresses this explicitly). The asymmetric modifier `private set` mirrors the trick `public int this[int i] { get; private set; }`: an external write `grid[1, 1] = 5;` from `Program.cs` will not compile, while the `Set` method, living inside the class, has full access. `get` is intentionally side-effect free — a lesson requirement: reading `obj[i]` must be cheap and predictable.

The overload `this[string cellRef]` with the same name `this` but a different parameter type shows indexer overloading by parameter type — like `this[int]` and `this[string]` in the lesson's `Bookshelf`. It is read-only (`get`-only), which follows the lesson recommendation "prefer `get`-only or `private set` when external mutation is not part of the contract." Parsing `"A1"`/`"B2"` via `char.ToUpper` and `int.TryParse` is a practical application of `System.Globalization`.

The method `PrintTo(TextWriter writer)` calls `GridSerializer.Serialize(this, writer);` — this is the second role of `this` from the lesson: passing the current instance to another method. `GridSerializer` is declared as a `static` class: inside it `this` is unavailable (the first common mistake of the lesson), and it reads cells only through the public indexer `grid[r, c]`, without access to private fields — safe encapsulation. The entry point uses top-level statements (C# 12 / .NET 8) and shows both read forms — via `int` and via `string` — plus a `try/catch` block demonstrating a meaningful exception on out-of-bounds access. Every lesson concept is exercised: `this` for fields, `this(...)` for delegation, `return this` for fluent APIs, passing `this` to an external method, the `this[int, int]` indexer, the `this[string]` overload, the asymmetric `private set`, bounds validation, the implicit `value`, the ban on `static` indexers, and the ban on side effects in `get`.

#### Going deeper (bonus)
1. Add a third indexer `this[(int row, int col) tuple]` that accepts a tuple, and explain why the compiler distinguishes it from `this[int, int]` (hint: the parameter type differs).
2. Implement a `Span<int>` row view via a `RowSpan(int row)` method, using `System.Runtime.InteropServices.MemoryMarshal` or a manual projection, and discuss why the indexer `get` must remain side-effect free for thread-safe reads.
3. Make `Grid` implement an `IMatrix` interface with explicit indexer implementation and compare it to a regular public implementation: when is explicit implementation preferable?
4. Add caching of the string indexer results in a `Dictionary<string, int>` and assess whether this violates the "side-effect free `get`" rule — argue whether lazy caching is an acceptable exception.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Проект `GridLab` собирается под .NET 8 / C# 12 без ошибок.
- [ ] (RU) Три конструктора с делегированием `: this(...)` в строке объявления.
- [ ] (RU) `SetName` использует `this._name = name;` для устранения неоднозначности.
- [ ] (RU) Fluent-методы `SetName`, `Set`, `Fill` возвращают `this`.
- [ ] (RU) Целочисленный индексатор `this[int, int]` проверяет границы и бросает `IndexOutOfRangeException`.
- [ ] (RU) `set` использует неявный `value`; `get` без побочных эффектов.
- [ ] (RU) `public get` / `private set` для целочисленного индексатора.
- [ ] (RU) Перегрузка `this[string]` только для чтения, корректный разбор `"A1"`..`"Z99"`.
- [ ] (RU) `PrintTo` вызывает `GridSerializer.Serialize(this, writer)`.
- [ ] (RU) `GridSerializer` — `static`, без `this` внутри.
- [ ] (EN) The `GridLab` project builds on .NET 8 / C# 12 without errors.
- [ ] (EN) Three constructors with `: this(...)` delegation on the declaration line.
- [ ] (EN) `SetName` uses `this._name = name;` for disambiguation.
- [ ] (EN) Fluent methods `SetName`, `Set`, `Fill` return `this`.
- [ ] (EN) The integer indexer `this[int, int]` validates bounds and throws `IndexOutOfRangeException`.
- [ ] (EN) `set` uses the implicit `value`; `get` is side-effect free.
- [ ] (EN) `public get` / `private set` for the integer indexer.
- [ ] (EN) The `this[string]` overload is read-only and correctly parses `"A1"`..`"Z99"`.
- [ ] (EN) `PrintTo` calls `GridSerializer.Serialize(this, writer)`.
- [ ] (EN) `GridSerializer` is `static`, with no `this` inside.

#### Ресурсы / Resources
- Microsoft Learn — Indexers (C# Programming Guide): https://learn.microsoft.com/dotnet/csharp/indexers
- Microsoft Learn — `this` keyword: https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/this
- Microsoft Learn — Constructors (using `this` for delegation): https://learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/constructors
- Microsoft Learn — Properties (asymmetric access modifiers): https://learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/using-properties
