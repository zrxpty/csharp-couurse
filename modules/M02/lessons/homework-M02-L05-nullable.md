---
[← Предыдущее ДЗ: M02-L04](homework-M02-L04-operators.md) | [← К уроку M02-L05](lesson-M02-L05-nullable.md) | [⬆ К модулю M02](../README.md) | [Следующее ДЗ: M02-L06 →](homework-M02-L06-type-conversion.md)
---

### Домашнее задание M02-L05: null и nullable-типы (int?) / Homework M02-L05: null and nullable types (int?)

**Урок / Lesson:** M02-L05
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) Научиться осознанно работать с отсутствием значения: применять `int?`/`Nullable<T>`, операторы `??`, `??=`, `?.`, включать nullable reference types и писать код, в котором `NullReferenceException` невозможен по построению. (EN) Learn to handle "no value" deliberately: use `int?`/`Nullable<T>`, the `??`, `??=`, `?.` operators, enable nullable reference types, and write code where `NullReferenceException` is impossible by construction.

#### Связь с уроком / Connection to the lesson
(RU) ДЗ закрепляет все ключевые темы урока: разницу между ссылочными и типами-значениями, синтаксис `int?` как сахар над `Nullable<int>`, безопасное чтение через `HasValue`/`Value`, три null-оператора и систему nullable reference types с предупреждениями CS86xx. Вы будете применять ровно те же приёмы, что в коде урока: pattern matching с `null`, raw string literal для отчёта, `?.` для обхода графа объектов и `??=` для ленивой инициализации.
(EN) The homework reinforces every key topic of the lesson: the difference between reference and value types, the `int?` sugar over `Nullable<int>`, safe reading through `HasValue`/`Value`, the three null-operators, and the nullable reference types system with CS86xx warnings. You will apply exactly the same techniques as the lesson code: pattern matching with `null`, a raw string literal for a report, `?.` for walking an object graph, and `??=` for lazy initialization.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Вы — разработчик мини-сервиса мониторинга температуры серверных комнат. Система опрашивает массив датчиков каждый час и сохраняет показания в массив `double?[]`, где `null` означает «датчик молчит» (нет связи, калибровка не выполнена, линия оборвана). Часть метаданных датчиков тоже может быть неизвестна на момент старта: дата калибровки `DateTime?`, ответственное лицо `string?`. При этом идентификатор и местоположение датчика — обязательны: без них запись не имеет смысла.

Бизнесу нужен ежедневный сводный отчёт: средняя температура по активным показаниям, количество «молчащих» датчиков, флаг самого проблемного датчика. Любой отчёт должен строиться даже тогда, когда вообще ни одного валидного показания нет — в этом случае вместо среднего выводится «нет данных», а не падает `InvalidOperationException`. Старая версия сервиса постоянно роняла прод из-за `NullReferenceException` в цепочках вида `sensor.CalibrationDate.Value.ToString(...)`, потому что разработчики забывали, что `Value` бросает исключение, а `CalibrationDate` мог быть `null`. Ваша задача — переписать модуль так, чтобы отсутствие значения стало легитимным, явно обработанным состоянием, а не миной замедленного действия.

Это типичный сценарий, ради которого в C# и существуют nullable-типы: «нет значения» — это не ошибка, а нормальное состояние домена. Урок показал, что `int?` — это `Nullable<int>` с битом «значение отсутствует», а три оператора (`??`, `??=`, `?.`) и система NRT позволяют описать контракт в сигнатурах и обойти граф объектов без ручных `if (x != null)`. В этом ДЗ вы соберёте все эти инструменты в один рабочий модуль и убедитесь, что компилятор сам ловит потенциальные NullReferenceException ещё на этапе сборки.

#### Что нужно сделать (пошагово)
1. Создайте новый консольный проект: `dotnet new console -n TempMonitor -o TempMonitor`, перейдите в него `cd TempMonitor`. Откройте `TempMonitor.csproj` и убедитесь, что `dotnet` создал `<Nullable>enable</Nullable>` и `<ImplicitUsings>enable</ImplicitUsings>`. Если `Nullable` отсутствует — добавьте его вручную внутрь `<PropertyGroup>`, потому что без него вся система статических контрактов NRT молчит и ДЗ теряет смысл.
2. В `Program.cs` замените шаблон на top-level statements с директивой `#nullable enable` в первой строке — это включит предупреждения даже для отдельных файлов. Объявите модель датчика как `record Sensor(int Id, string Location, DateTime? CalibrationDate, string? Owner);`. Заметьте: `Id` и `Location` — не-null по контракту, а `CalibrationDate` и `Owner` помечены `?`, потому что могут быть неизвестны.
3. Создайте массив датчиков `Sensor[]` с помощью collection expression `[ ... ]`. Включите туда至少 один датчик с `CalibrationDate: null` и один с `Owner: null`, чтобы проверить, как ваш код справляется с отсутствующими метаданными. Создайте массив показаний `double?[] readings = [22.5, null, 19.0, null, 21.0, 18.5, null];` — намеренно с `null` внутри.
4. Напишите локальную функцию `static double? ComputeAverage(double?[] values)`, которая возвращает `null`, если ни одного валидного показания нет, иначе среднее. Внутри фильтруйте через `Where(v => v.HasValue)`, затем безопасно доставайте значение. Подумайте, какой оператор здесь уместнее — `v!.Value` (с оправданным null-forgiving, ведь вы только что отфильтровали) или `v.Value` после `if (v.HasValue)` в `Select`. Реализуйте оба варианта и сравните.
5. Напишите функцию `static string Describe(double? value)`, использующую switch expression с pattern matching: `null`, `< 0`, `0`, диапазоны температуры («прохладно», «комфортно», «жарко», «критично»). Урок применял ровно такой приём — повторите его, но с диапазонами `> 0 and < 18` и т.п.
6. Напишите функцию поиска датчика по `Id`, возвращающую `Sensor?`. Обойдите граф объектов через `?.`: возьмите `found?.CalibrationDate?.ToString("yyyy-MM-dd") ?? "калибровка отсутствует"`. Убедитесь, что даже если `found` равен `null`, никакого исключения не возникает — вся цепочка тихо возвращает `null`, а `??` подставляет заглушку.
7. Реализуйте ленивый кэш отчёта через `??=`: объявите `string? reportCache = null;`, затем `reportCache ??= BuildReport(...);`. При повторном вызове отчёт не должен пересчитываться. Добавьте вывод в консоль дважды и убедитесь по времени/логу, что `BuildReport` вызвался один раз.
8. Соберите финальный отчёт в raw string literal `$$""" ... """`, включив: общее число датчиков, число показаний, активных, молчащих, среднее (или «нет данных»), местоположение самого проблемного датчика (того, у которого `CalibrationDate is null`). Используйте interpolation с ternary/pattern: `avg is { } a ? a.ToString("F2") : "нет данных"`.
9. Соберите проект: `dotnet build`. Цель — **ноль предупреждений CS86xx**. Если компилятор ругается на CS8602/CS8600 — не глушите `!`, а перепишите код: добавьте проверку, поменяйте тип на `?` или используйте `??`. Запустите `dotnet run` и сверьте вывод с ожидаемым: среднее по активным показаниям, счётчики, описание каждого показания, отчёт.
10. Проверьте краевые случаи вручную: временно замените `readings` на массив из одних `null` — `ComputeAverage` должен вернуть `null`, а отчёт показать «нет данных» и не упасть. Верните исходный массив обратно.

#### Требования к решению
- Проект — консольное приложение .NET 8, C# 12, top-level statements, `<Nullable>enable</Nullable>` в `.csproj` и `#nullable enable` в `Program.cs`.
- Модель датчика — `record` с явным nullable-контрактом: обязательные поля без `?`, опциональные — с `?`. Никаких `class` с изменяемыми полями без необходимости.
- Все обращения к `Value` у nullable-типов предваряются проверкой `HasValue`, либо значение достаётся через pattern matching `{ }`, либо через `??` с дефолтом. Чтение `Value` «вслепую» запрещено.
- Все цепочки по ссылочному графу (`Sensor` → `CalibrationDate` → `ToString`) используют `?.`, а не прямой доступ. Каждая потенциально-null ссылка защищена.
- Операторы `??`, `??=`, `?.` применены каждый минимум один раз и осмысленно (не ради галочки). Ленивая инициализация кэша — именно через `??=`.
- Pattern matching со switch expression покрывает `null` отдельной веткой. Диапазоны используют комбинатор `and`.
- Отчёт собран в raw string literal с интерполяцией. Среднее выводится через формат `F2` или «нет данных», без исключений при пустом массиве.
- Сборка `dotnet build` проходит без предупреждений CS8600/CS8602/CS8603/CS8625. Использование `!` (null-forgiving) допускается ровно в одном месте — там, где вы отфильтровали `HasValue` и анализатор не может это вывести; в комментарии объяснён инвариант.
- Код сопровождается двуязычными комментариями (RU+EN) в ключевых местах: у каждой функции и у каждого применения null-оператора — одна строка «что и почему».

#### Тонкости и подводные камни
- Главная ловушка урока — чтение `.Value` без `.HasValue`. Даже если вы «знаете», что значение есть, компилятор вам не мешает, а в runtime прилетает `InvalidOperationException`. В `ComputeAverage` после `Where(v => v.HasValue)` анализатор всё ещё видит `v` как `double?`, поэтому `v.Value` формально безопасно, но компилятор не выдаёт гарантий — здесь уместен `v!.Value` с комментарием «фильтр по HasValue выше» либо переписывание через `Select(v => (double?)v)` после проверки. Не оставляйте `Value` «голым».
- Не путайте `int?` и `int`: присваивание `int? x = null; int y = x;` не компилируется без явного приведения — компилятор защищает от потери бита. Обратное (`int? z = 5;`) разрешено, потому что любой `int` легитимно оборачивается в `Nullable<int>`. Это частый источник сообщений CS0266 при невнимательности.
- Оператор `?.` меняет тип результата: `DateTime?.ToString()` возвращает `string?`, а не `string`. Поэтому `found?.CalibrationDate?.ToString(...)` имеет тип `string?` и обязан комбинироваться с `??`, чтобы получить не-null `string` для отчёта. Забыть `??` — получить CS8602 при использовании значения как не-null.
- `??=` не «мгновенный» сахар для `if (x == null) x = y`: он вычисляет правую часть только если левая равна null (короткое замыкание). Это важно для ленивого кэша: `reportCache ??= BuildReport(...)` не пересчитает отчёт, если кэш уже заполнен. Если написать `reportCache = reportCache ?? BuildReport(...)`, результат тот же, но `??=` читается лучше и идиоматичнее.
- NRT — статический контракт, а не runtime-защита. Включение `<Nullable>enable</Nullable>` не добавляет проверок в IL; оно лишь заставляет компилятор ругаться. Поэтому «отсутствие предупреждений» — это договор о дисциплине, а не гарантия отсутствия NRE. Особенно коварно взаимодействие с не-NRT-кодом (старые библиотеки): `string?` из чужого API может вернуть null без предупреждения. В таких точках ставьте явную проверку.
- Оператор `!` (null-forgiving) — не «затычка для CS8602», а заявление «я доказал не-null, а анализатор не видит». Бездумное `x!.Foo()` превращает статическое предупреждение в runtime-аварию. В ДЗ разрешён ровно один `!` — в фильтрованном `Select`, с комментарием-инвариантом.
- Никогда не возвращайте `null` из метода, возвращающего коллекцию, — возвращайте `Array.Empty<T>()`/`Enumerable.Empty<T>()`. В `ComputeAverage` возвращать `null` для «нет данных» легитимно, потому что `double` — тип-значение и у него нет «пустого» состояния; именно для этого и нужен `double?`.

#### Критерии приёмки
- [ ] Проект собирается `dotnet build` без ошибок и без предупреждений CS8600/CS8602/CS8603/CS8625.
- [ ] `dotnet run` выводит: среднее активных показаний, число активных/молчащих, описания показаний и итоговый отчёт.
- [ ] В `.csproj` присутствует `<Nullable>enable</Nullable>`, в `Program.cs` — `#nullable enable`.
- [ ] Модель `Sensor` — `record` с `string Location` (не-null) и `DateTime? CalibrationDate`, `string? Owner`.
- [ ] `ComputeAverage` возвращает `null` для пустого/полностью-null массива и не бросает исключений.
- [ ] `Describe` через switch expression покрывает `null` и диапазоны через `and`.
- [ ] Граф объектов обходится через `?.` (минимум одна цепочка из двух и более `?.`).
- [ ] Оператор `??` применён минимум дважды (дефолт и подстановка в отчёте).
- [ ] Оператор `??=` применён для ленивого кэша отчёта ровно один раз.
- [ ] Отчёт собран в raw string literal `$$""" ... """` с интерполяцией.
- [ ] `!` (null-forgiving) встречается не более одного раза и сопровождается комментарием-инвариантом.
- [ ] Тест «массив из одних null» проходит: отчёт показывает «нет данных», приложение не падает.
- [ ] Код содержит двуязычные комментарии у каждой функции и каждого null-оператора.
- [ ] В выводе присутствует информация о самом проблемном датчике (с `CalibrationDate is null`).
- [ ] Никаких ручных `if (x != null) return x.Foo()` там, где достаточно `?.`/`??`.

#### Подсказки (без прямого ответа)
- Если компилятор ругается CS8602 на `found.Name` после `FirstOrDefault` — это не баг, это подсказка: результат может быть null. Подумайте, какой pattern (`is not null`, `is { } s`) разрулит flow-анализ.
- Для среднего по `List<double>` годится LINQ `Average()`, но он бросает `InvalidOperationException` на пустом списке. Где это проверить?
- `DateTime?.ToString(format)` возвращает `string?`. Что нужно добавить, чтобы вставить результат в не-null поле отчёта?
- Raw string literal требует, чтобы закрывающие `"""` были на отдельной строке с отступом не меньше, чем у содержимого. Следите за выравниванием.

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 — top-level statements, Nullable enable
// Система учёта показаний датчиков температуры
// Temperature sensor readings accounting system

#nullable enable

using System.Globalization;
using System.Linq;

// --- Модель датчика: обязательные поля без ?, опциональные — с ? ---
// --- Sensor model: required fields without ?, optional — with ? ---
record Sensor(int Id, string Location, DateTime? CalibrationDate, string? Owner);

Sensor[] sensors =
[
    new(1, "Server room A", new DateTime(2024, 1, 15), "DevOps team"),
    new(2, "Server room B", null,                     null),          // не калиброван, владелец неизвестен / not calibrated, owner unknown
    new(3, "Corridor",      new DateTime(2023, 11, 1), "Facility"),
];

// --- Показания: null = датчик молчит / Readings: null = sensor silent ---
double?[] readings = [22.5, null, 19.0, null, 21.0, 18.5, null];

// 1. Среднее по активным показаниям (null, если активных нет)
// 1. Average over active readings (null when none are active)
double? average = ComputeAverage(readings);
Console.WriteLine(
    $"Среднее / Average: {(average is { } a ? a.ToString("F2", CultureInfo.InvariantCulture) : "нет данных / no data")}");

// 2. Счётчики через HasValue / Counters via HasValue
int active = readings.Count(r => r.HasValue);
Console.WriteLine($"Активных / Active: {active} из / of {readings.Length}");

// 3. Описание показания через pattern matching с null
// 3. Reading description via pattern matching with null
foreach (double? r in readings)
    Console.WriteLine(Describe(r));

// 4. Безопасный обход графа объектов через ?.
// 4. Safe object-graph walk via ?.
Sensor? found = sensors.FirstOrDefault(s => s.Id == 2);
string calInfo = found?.CalibrationDate?.ToString("yyyy-MM-dd") ?? "калибровка отсутствует / no calibration";
string ownerInfo = found?.Owner ?? "владелец неизвестен / owner unknown";
Console.WriteLine($"Датчик 2 / Sensor 2: {found?.Location ?? "—"}, калибровка: {calInfo}, владелец: {ownerInfo}");

// 5. Ленивый кэш через ??= / Lazy cache via ??=
string? reportCache = null;
reportCache ??= BuildReport(sensors, readings);
Console.WriteLine(reportCache);

// --- Локальные функции ---
// --- Local functions ---
static double? ComputeAverage(double?[] values)
{
    // Только непустые показания / only non-null readings
    List<double> active = values.Where(v => v.HasValue)
                                .Select(v => v!.Value) // инвариант: HasValue гарантирован фильтром / invariant: HasValue guaranteed by the filter
                                .ToList();
    return active.Count == 0 ? null : active.Average(); // пусто => null, иначе среднее / empty => null, else average
}

static string Describe(double? value) => value switch
{
    null           => "показание отсутствует / reading missing",
    < 0            => $"отрицательное: {value} (ошибка датчика?) / negative: {value} (sensor error?)",
    0              => "ноль / zero",
    > 0 and < 18   => $"прохладно: {value:F1}°C / cool: {value:F1}°C",
    >= 18 and < 26 => $"комфортно: {value:F1}°C / comfortable: {value:F1}°C",
    >= 26 and < 35 => $"жарко: {value:F1}°C / hot: {value:F1}°C",
    _              => $"критично: {value:F1}°C / critical: {value:F1}°C"
};

static string BuildReport(Sensor[] sensors, double?[] readings)
{
    double? avg = ComputeAverage(readings);
    int active = readings.Count(r => r.HasValue);
    int silent = readings.Length - active;
    // Самый проблемный датчик: CalibrationDate is null; найдём через pattern
    // Worst sensor: CalibrationDate is null; locate via pattern
    Sensor? worst = sensors.FirstOrDefault(s => s.CalibrationDate is null);
    return $"""
        ===== Отчёт по температуре / Temperature report =====
        Всего датчиков / total sensors : {sensors.Length}
        Показаний / readings           : {readings.Length}
        Активных / active              : {active}
        Молчащих / silent              : {silent}
        Среднее / average              : {(avg is { } a ? a.ToString("F2", CultureInfo.InvariantCulture) : "нет данных / no data")}
        Худший датчик / worst sensor   : {worst?.Location ?? "все калиброваны / all calibrated"}
        """;
}
```

Разбор по строкам. `record Sensor` объявляет контракт: `Id` и `Location` — не-null, `CalibrationDate` и `Owner` — `?`. Это ровно правило урока: помечать nullable-поля явно. Массив `sensors` через collection expression `[ ... ]` — синтаксис C# 12. Второй датчик намеренно с `null` в обоих опциональных полях, чтобы проверить граничный случай.

`ComputeAverage` возвращает `double?`, потому что «нет данных» — легитимное состояние, а у `double` его нет. Внутри — `Where(v => v.HasValue)`, затем `Select(v => v!.Value)`. Здесь единственный оправданный `!`: после фильтра по `HasValue` значение точно есть, но анализатор не распространяет этот факт через LINQ, поэтому мы заявляем инвариант комментарием. Это применение правила «`!` только при доказанном инварианте». Если `active.Count == 0` — возвращаем `null`, иначе `Average()`; мы избегаем исключения «пустая последовательность», которое бросает сам `Average()`.

`Describe` — switch expression с pattern matching, как в уроке: ветка `null` отдельная, диапазоны через комбинатор `and`. Это идиоматичная замена каскаду `if (value.HasValue) { ... } else { ... }`.

Обход графа `found?.CalibrationDate?.ToString(...) ?? "..."` — три приема урока в одной строке: `?.` прерывает цепочку на первом null, `ToString` вызывается только на не-null, а `??` подставляет дефолт. Тип всего выражения — не-null `string`, поэтому CS8602 не возникает. `found?.Owner ?? "..."` — то же для строкового поля.

Ленивый кэш `reportCache ??= BuildReport(...)` — оператор урока `??=`: правая часть вычисляется только если `reportCache` равен null. При повторном вызове отчёт не пересчитывается, что и требовалось.

`BuildReport` собирает отчёт в raw string literal `$$""" ... """` с интерполяцией. Строка среднего использует `avg is { } a ? ... : "нет данных"` — pattern `{ }` проверяет не-null и одновременно вводит переменную `a` типа `double`, так что `a.ToString(...)` безопасно и без `!`. Поиск худшего датчика — `FirstOrDefault(s => s.CalibrationDate is null)`, результат `Sensor?`, вставляем через `worst?.Location ?? "..."`. Никаких голых `Value`, никаких ручных `if (x != null)`, никаких CS86xx — контракт выдержан от сигнатур до вывода.

#### Задания на углубление (бонус)
1. Добавьте поле `string? Notes` и метод `string Summarize(Sensor? s)`, который через единственное switch expression возвращает сводку: «датчик не найден» для null, «нет калибровки» для `CalibrationDate is null`, иначе дату и владельца.
2. Реализуйте `double? Median(double?[] values)` — медиану по активным показаниям с чётным/нечётным числом элементов; разберитесь, что вернуть для пустого массива.
3. Переведите `ComputeAverage` на `Span<double?>`/`stackalloc` для коротких массивов и сравните производительность; объясните, почему `Nullable<T>` в span остаётся безопасным.
4. Подключите XML-комментарии `/// <summary>` к каждой функции с описанием nullable-контракта (`<returns>` уточняет, когда возможен null) и сгенерируйте документацию через `dotnet build -p:GenerateDocumentationFile=true`.

---

## Statement in English / Постановка на английском

#### Context & motivation
You are the developer of a small server-room temperature monitoring service. The system polls an array of sensors every hour and stores readings in a `double?[]`, where `null` means "sensor silent" (no link, calibration missing, line broken). Some sensor metadata may also be unknown at start time: the calibration date `DateTime?`, the responsible person `string?`. The sensor identifier and location, however, are mandatory: without them a record is meaningless.

The business needs a daily summary report: average temperature over active readings, the count of "silent" sensors, and a flag for the most problematic sensor. Any report must build even when not a single valid reading exists — in that case the average line shows "no data" instead of crashing with `InvalidOperationException`. The old version of the service kept dropping production with `NullReferenceException` in chains like `sensor.CalibrationDate.Value.ToString(...)`, because developers forgot that `Value` throws and that `CalibrationDate` could be `null`. Your task is to rewrite the module so that "no value" becomes a legitimate, explicitly handled state rather than a slow-burning mine.

This is exactly the scenario nullable types exist for in C#: "no value" is not an error, it is a normal domain state. The lesson showed that `int?` is `Nullable<int>` with an extra "value is missing" bit, and that the three operators (`??`, `??=`, `?.`) plus the NRT system let you express the contract in signatures and walk an object graph without manual `if (x != null)`. In this homework you will gather all of these tools into one working module and verify that the compiler itself catches potential NullReferenceExceptions at build time.

#### What to do step by step
1. Create a new console project: `dotnet new console -n TempMonitor -o TempMonitor`, then `cd TempMonitor`. Open `TempMonitor.csproj` and confirm that `dotnet` generated `<Nullable>enable</Nullable>` and `<ImplicitUsings>enable</ImplicitUsings>`. If `Nullable` is missing, add it manually inside `<PropertyGroup>` — without it the whole NRT static-contract system stays silent and the homework loses its point.
2. In `Program.cs` replace the template with top-level statements and a `#nullable enable` directive on the first line — this turns on warnings even at the file level. Declare the sensor model as `record Sensor(int Id, string Location, DateTime? CalibrationDate, string? Owner);`. Note that `Id` and `Location` are non-null by contract, while `CalibrationDate` and `Owner` are marked `?` because they may be unknown.
3. Build the sensors array `Sensor[]` with a collection expression `[ ... ]`. Include at least one sensor with `CalibrationDate: null` and one with `Owner: null`, to test how your code copes with missing metadata. Create the readings array `double?[] readings = [22.5, null, 19.0, null, 21.0, 18.5, null];` — intentionally with `null` inside.
4. Write a local function `static double? ComputeAverage(double?[] values)` that returns `null` when no valid reading exists, otherwise the average. Inside, filter with `Where(v => v.HasValue)`, then safely extract the value. Think about which operator fits best here — `v!.Value` (a justified null-forgiving, since you just filtered) or `v.Value` after `if (v.HasValue)` inside `Select`. Implement both and compare.
5. Write a function `static string Describe(double? value)` using a switch expression with pattern matching: `null`, `< 0`, `0`, temperature ranges ("cool", "comfortable", "hot", "critical"). The lesson used exactly this technique — reproduce it, but with ranges like `> 0 and < 18`.
6. Write a function that finds a sensor by `Id` and returns `Sensor?`. Walk the object graph with `?.`: take `found?.CalibrationDate?.ToString("yyyy-MM-dd") ?? "no calibration"`. Make sure that even when `found` is `null` no exception is raised — the whole chain quietly yields `null`, and `??` substitutes the fallback.
7. Implement a lazy report cache with `??=`: declare `string? reportCache = null;`, then `reportCache ??= BuildReport(...);`. On a second call the report must not be recomputed. Print it twice and verify by timing/logging that `BuildReport` ran only once.
8. Assemble the final report in a raw string literal `$$""" ... """`, including: total sensors, number of readings, active, silent, the average (or "no data"), and the location of the most problematic sensor (the one whose `CalibrationDate is null`). Use interpolation with a ternary/pattern: `avg is { } a ? a.ToString("F2") : "no data"`.
9. Build the project: `dotnet build`. The goal is **zero CS86xx warnings**. If the compiler complains about CS8602/CS8600, do not silence with `!` — rewrite the code: add a check, change the type to `?`, or use `??`. Run `dotnet run` and compare the output with the expected: the average of active readings, the counters, each reading description, the report.
10. Check edge cases manually: temporarily replace `readings` with an array of all `null` — `ComputeAverage` must return `null` and the report must show "no data" without crashing. Restore the original array afterward.

#### Requirements
- The project is a .NET 8 console application, C# 12, top-level statements, with `<Nullable>enable</Nullable>` in `.csproj` and `#nullable enable` in `Program.cs`.
- The sensor model is a `record` with an explicit nullable contract: required fields without `?`, optional ones with `?`. No mutable `class` fields unless necessary.
- Every access to `Value` on a nullable type is preceded by a `HasValue` check, or the value is extracted via the `{ }` pattern, or via `??` with a default. Reading `Value` "blind" is forbidden.
- Every chain over the reference graph (`Sensor` → `CalibrationDate` → `ToString`) uses `?.`, not direct access. Each potentially-null reference is guarded.
- The operators `??`, `??=`, `?.` are each used at least once and meaningfully (not for a checkbox). Lazy cache initialization must be via `??=`.
- Pattern matching in the switch expression covers `null` as a separate arm. Ranges use the `and` combinator.
- The report is built in a raw string literal with interpolation. The average is printed with `F2` or "no data", with no exceptions on an empty array.
- `dotnet build` passes with no CS8600/CS8602/CS8603/CS8625 warnings. The `!` (null-forgiving) operator is allowed in exactly one place — where you filtered by `HasValue` and the analyzer cannot infer it; a comment explains the invariant.
- The code carries bilingual comments (RU+EN) at key spots: at each function and at each null-operator use, one line of "what and why".

#### Pitfalls
- The lesson's main trap is reading `.Value` without `.HasValue`. Even if you "know" the value is there, the compiler does not stop you, and at runtime an `InvalidOperationException` lands. In `ComputeAverage`, after `Where(v => v.HasValue)` the analyzer still sees `v` as `double?`, so `v.Value` is formally safe but the compiler gives no guarantee — here `v!.Value` with a comment "filter by HasValue above" is appropriate, or a rewrite through `Select(v => (double?)v)` after a check. Do not leave `Value` bare.
- Do not confuse `int?` and `int`: assigning `int? x = null; int y = x;` does not compile without an explicit cast — the compiler guards against losing the bit. The reverse (`int? z = 5;`) is allowed, because any `int` legitimately wraps into `Nullable<int>`. This is a common source of CS0266 when you are not careful.
- The `?.` operator changes the result type: `DateTime?.ToString()` returns `string?`, not `string`. So `found?.CalibrationDate?.ToString(...)` has type `string?` and must be combined with `??` to get a non-null `string` for the report. Forgetting `??` yields CS8602 when the value is used as non-null.
- `??=` is not "instant sugar" for `if (x == null) x = y`: it evaluates the right side only if the left is null (short-circuit). This matters for a lazy cache: `reportCache ??= BuildReport(...)` will not recompute the report if the cache is already filled. Writing `reportCache = reportCache ?? BuildReport(...)` gives the same result, but `??=` reads better and is idiomatic.
- NRT is a static contract, not runtime protection. Enabling `<Nullable>enable</Nullable>` adds no checks to the IL; it only makes the compiler complain. So "no warnings" is a discipline agreement, not a guarantee of no NRE. The interaction with non-NRT code (legacy libraries) is especially sneaky: a `string?` from a foreign API may return null without a warning. Put an explicit check at such boundaries.
- The `!` (null-forgiving) operator is not a "plug for CS8602" but a statement "I have proven non-null and the analyzer cannot see it". A careless `x!.Foo()` turns a static warning into a runtime crash. In the homework exactly one `!` is allowed — in the filtered `Select`, with an invariant comment.
- Never return `null` from a collection-returning method — return `Array.Empty<T>()`/`Enumerable.Empty<T>()`. In `ComputeAverage`, returning `null` for "no data" is legitimate because `double` is a value type with no "empty" state; that is precisely what `double?` is for.

#### Acceptance criteria
- [ ] The project builds with `dotnet build` with no errors and no CS8600/CS8602/CS8603/CS8625 warnings.
- [ ] `dotnet run` prints: the average of active readings, active/silent counts, reading descriptions, and the final report.
- [ ] `.csproj` contains `<Nullable>enable</Nullable>`; `Program.cs` has `#nullable enable`.
- [ ] The `Sensor` model is a `record` with `string Location` (non-null) and `DateTime? CalibrationDate`, `string? Owner`.
- [ ] `ComputeAverage` returns `null` for an empty / all-null array and throws no exceptions.
- [ ] `Describe` uses a switch expression that covers `null` and ranges via `and`.
- [ ] The object graph is walked with `?.` (at least one chain with two or more `?.`).
- [ ] The `??` operator is used at least twice (a default and a substitution in the report).
- [ ] The `??=` operator is used for the lazy report cache exactly once.
- [ ] The report is built in a raw string literal `$$""" ... """` with interpolation.
- [ ] The `!` (null-forgiving) operator appears at most once and carries an invariant comment.
- [ ] The "all-null array" test passes: the report shows "no data" and the app does not crash.
- [ ] The code has bilingual comments at each function and each null-operator use.
- [ ] The output mentions the most problematic sensor (the one with `CalibrationDate is null`).
- [ ] No manual `if (x != null) return x.Foo()` where `?.`/`??` would suffice.

#### Hints (no direct answer)
- If the compiler complains CS8602 about `found.Name` after `FirstOrDefault`, that is not a bug but a hint: the result may be null. Which pattern (`is not null`, `is { } s`) would resolve the flow analysis?
- `Average()` on a `List<double>` works, but throws `InvalidOperationException` on an empty list. Where should you check that?
- `DateTime?.ToString(format)` returns `string?`. What do you add to insert the result into a non-null field of the report?
- A raw string literal requires the closing `"""` on its own line, indented no less than the content. Watch the alignment.

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8 — top-level statements, Nullable enable
// Temperature sensor readings accounting system

#nullable enable

using System.Globalization;
using System.Linq;

// --- Sensor model: required fields without ?, optional — with ? ---
record Sensor(int Id, string Location, DateTime? CalibrationDate, string? Owner);

Sensor[] sensors =
[
    new(1, "Server room A", new DateTime(2024, 1, 15), "DevOps team"),
    new(2, "Server room B", null,                     null),          // not calibrated, owner unknown
    new(3, "Corridor",      new DateTime(2023, 11, 1), "Facility"),
];

// --- Readings: null = sensor silent ---
double?[] readings = [22.5, null, 19.0, null, 21.0, 18.5, null];

// 1. Average over active readings (null when none are active)
double? average = ComputeAverage(readings);
Console.WriteLine(
    $"Average: {(average is { } a ? a.ToString("F2", CultureInfo.InvariantCulture) : "no data")}");

// 2. Counters via HasValue
int active = readings.Count(r => r.HasValue);
Console.WriteLine($"Active: {active} of {readings.Length}");

// 3. Reading description via pattern matching with null
foreach (double? r in readings)
    Console.WriteLine(Describe(r));

// 4. Safe object-graph walk via ?.
Sensor? found = sensors.FirstOrDefault(s => s.Id == 2);
string calInfo = found?.CalibrationDate?.ToString("yyyy-MM-dd") ?? "no calibration";
string ownerInfo = found?.Owner ?? "owner unknown";
Console.WriteLine($"Sensor 2: {found?.Location ?? "—"}, calibration: {calInfo}, owner: {ownerInfo}");

// 5. Lazy cache via ??=
string? reportCache = null;
reportCache ??= BuildReport(sensors, readings);
Console.WriteLine(reportCache);

// --- Local functions ---
static double? ComputeAverage(double?[] values)
{
    // only non-null readings
    List<double> active = values.Where(v => v.HasValue)
                                .Select(v => v!.Value) // invariant: HasValue guaranteed by the filter
                                .ToList();
    return active.Count == 0 ? null : active.Average(); // empty => null, else average
}

static string Describe(double? value) => value switch
{
    null           => "reading missing",
    < 0            => $"negative: {value} (sensor error?)",
    0              => "zero",
    > 0 and < 18   => $"cool: {value:F1}°C",
    >= 18 and < 26 => $"comfortable: {value:F1}°C",
    >= 26 and < 35 => $"hot: {value:F1}°C",
    _              => $"critical: {value:F1}°C"
};

static string BuildReport(Sensor[] sensors, double?[] readings)
{
    double? avg = ComputeAverage(readings);
    int active = readings.Count(r => r.HasValue);
    int silent = readings.Length - active;
    // Worst sensor: CalibrationDate is null; locate via pattern
    Sensor? worst = sensors.FirstOrDefault(s => s.CalibrationDate is null);
    return $"""
        ===== Temperature report =====
        Total sensors : {sensors.Length}
        Readings      : {readings.Length}
        Active        : {active}
        Silent        : {silent}
        Average       : {(avg is { } a ? a.ToString("F2", CultureInfo.InvariantCulture) : "no data")}
        Worst sensor  : {worst?.Location ?? "all calibrated"}
        """;
}
```

Line-by-line walk-through. The `record Sensor` declares the contract: `Id` and `Location` are non-null, `CalibrationDate` and `Owner` are `?`. This is exactly the lesson rule: mark nullable fields explicitly. The `sensors` array uses a C# 12 collection expression `[ ... ]`. The second sensor intentionally has `null` in both optional fields, to exercise the edge case.

`ComputeAverage` returns `double?` because "no data" is a legitimate state and `double` has none. Inside there is `Where(v => v.HasValue)` followed by `Select(v => v!.Value)`. Here is the single justified `!`: after filtering by `HasValue` the value is definitely present, but the analyzer does not propagate that fact through LINQ, so we assert the invariant in a comment. This applies the rule "`!` only with a proven invariant". If `active.Count == 0`, return `null`; otherwise `Average()`. We avoid the "empty sequence" exception that `Average()` itself throws.

`Describe` is a switch expression with pattern matching, like in the lesson: a separate `null` arm, ranges via the `and` combinator. This is the idiomatic replacement for a cascade of `if (value.HasValue) { ... } else { ... }`.

The graph walk `found?.CalibrationDate?.ToString(...) ?? "..."` packs three lesson techniques into one line: `?.` stops the chain at the first null, `ToString` runs only on a non-null, and `??` supplies the fallback. The whole expression has type non-null `string`, so CS8602 never arises. `found?.Owner ?? "..."` does the same for the string field.

The lazy cache `reportCache ??= BuildReport(...)` is the lesson's `??=` operator: the right side is evaluated only when `reportCache` is null. A second call does not recompute the report, which is what was required.

`BuildReport` assembles the report in a raw string literal `$$""" ... """` with interpolation. The average line uses `avg is { } a ? ... : "no data"` — the `{ }` pattern checks for non-null and simultaneously introduces a variable `a` of type `double`, so `a.ToString(...)` is safe without `!`. The worst sensor is found with `FirstOrDefault(s => s.CalibrationDate is null)`, the result is `Sensor?`, and we insert it via `worst?.Location ?? "..."`. No bare `Value`, no manual `if (x != null)`, no CS86xx — the contract holds from signatures to output.

#### Going deeper (bonus)
1. Add a `string? Notes` field and a method `string Summarize(Sensor? s)` that, in a single switch expression, returns a summary: "sensor not found" for null, "no calibration" for `CalibrationDate is null`, otherwise the date and owner.
2. Implement `double? Median(double?[] values)` — the median over active readings with even/odd counts; decide what to return for an empty array.
3. Move `ComputeAverage` onto `Span<double?>`/`stackalloc` for short arrays and compare performance; explain why `Nullable<T>` in a span stays safe.
4. Add XML `/// <summary>` comments to every function describing the nullable contract (`<returns>` clarifies when null is possible) and generate docs via `dotnet build -p:GenerateDocumentationFile=true`.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Проект `TempMonitor` собирается без предупреждений CS86xx.
- [ ] (RU) `dotnet run` выводит среднее, счётчики, описания и отчёт.
- [ ] (RU) Включён `<Nullable>enable</Nullable>` и `#nullable enable`.
- [ ] (RU) Модель — `record` с корректным nullable-контрактом.
- [ ] (RU) Операторы `??`, `??=`, `?.` применены осмысленно.
- [ ] (RU) Тест «массив из null» проходит без падения.
- [ ] (RU) Двуязычные комментарии у функций и null-операторов.
- [ ] (EN) The `TempMonitor` project builds with no CS86xx warnings.
- [ ] (EN) `dotnet run` prints the average, counters, descriptions, and report.
- [ ] (EN) `<Nullable>enable</Nullable>` and `#nullable enable` are on.
- [ ] (EN) The model is a `record` with a correct nullable contract.
- [ ] (EN) The `??`, `??=`, `?.` operators are used meaningfully.
- [ ] (EN) The "all-null array" test passes without crashing.
- [ ] (EN) Bilingual comments at functions and null-operator uses.

#### Ресурсы / Resources
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/nullable-value-types — Nullable value types
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/nullable-references — Nullable reference types
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/operators/null-coalescing-operator — `??` and `??=` operators
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/operators/member-access-operators — `?.` and `!` operators
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-12#collection-expressions — Collection expressions (C# 12)

---
[← Предыдущее ДЗ: M02-L04](homework-M02-L04-operators.md) | [← К уроку M02-L05](lesson-M02-L05-nullable.md) | [⬆ К модулю M02](../README.md) | [Следующее ДЗ: M02-L06 →](homework-M02-L06-type-conversion.md)
---
