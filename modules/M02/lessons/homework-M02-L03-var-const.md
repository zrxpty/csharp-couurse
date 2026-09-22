---
[← К уроку M02-L03](lesson-M02-L03-var-const.md) | [⬆ К модулю M02](../README.md) | [Следующее ДЗ →](homework-M02-L04-operators.md)
---

### Домашнее задание M02-L03: Объявление переменных, var, константы / Homework M02-L03: Declaring variables, var, constants

**Урок / Lesson:** M02-L03
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) Научиться осознанно выбирать между явным типом и `var`, применять `const` для значений, известных на этапе компиляции, отличать `const` от `readonly`, соблюдать соглашения об именовании (camelCase / PascalCase) и использовать типизированные паттерны C# 12. (EN) Learn to choose deliberately between an explicit type and `var`, apply `const` for compile-time values, distinguish `const` from `readonly`, follow naming conventions (camelCase / PascalCase) and use C# 12 typed patterns.

#### Связь с уроком / Connection to the lesson
(RU) ДЗ напрямую закрепляет все ключевые темы урока: явное объявление типов, неявную типизацию через `var` (с акцентом, что `var` НЕ динамический), критерии уместности `var`, `const` для значений времени компиляции, отличие `const` от `readonly`, именование в camelCase/PascalCase и pattern matching с типизированным паттерном `is int value and > 10 and < 100`.
(EN) The homework directly reinforces every core topic of the lesson: explicit type declarations, implicit typing through `var` (with emphasis that `var` is NOT dynamic), the criteria for when `var` is appropriate, `const` for compile-time values, the distinction between `const` and `readonly`, camelCase/PascalCase naming and pattern matching with the typed pattern `is int value and > 10 and < 100`.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Вы присоединяетесь к команде, которая разрабатывает небольшой компонент «Агрегатор показаний датчиков» (SensorReadingsAggregator) для системы мониторинга температуры склада. Компонент принимает сырые показания, проверяет их на попадание в допустимый диапазон, агрегирует статистику и печатает краткий отчёт. Кодовая база молодая, и ревьюер уже несколько раз возвращал пулл-реквесты с замечаниями: кто-то использовал `var data = GetSomething();`, где тип непонятно; кто-то пытался объявить `const DateTime CreatedAt = DateTime.UtcNow;` и удивлялся ошибке компиляции; кто-то называл переменные `d`, `s`, `x1`, и никто не мог понять, что в них лежит.

Ваша задача — реализовать компонент так, чтобы он компилировался под C# 12 / .NET 8, работал корректно и при этом демонстрировал зрелое владение темами урока M02-L03: осознанный выбор между явным типом и `var`, грамотное применение `const` и `readonly`, корректное именование и использование типизированных паттернов. Это не абстрактное упражнение: именно такие решения вы будете принимать в каждом пулл-реквесте, и умение объяснить, почему здесь `var`, а здесь явный тип, ценится выше, чем слепое следование правилу «везде `var`» или «везде явный тип».

В уроке подчёркнуто: `var` — это НЕ динамическая типизация. Тип фиксируется один раз на этапе компиляции и больше не меняется. Значит, код вроде `var city = "Berlin"; city = 42;` не скомпилируется. Это ключевое заблуждение новичков, и в ДЗ вы столкнётесь с ситуацией, где его легко допустить. Также урок напоминает: `const` работает только с примитивами, строками и `null`-ссылками — никаких вычислений в рантайме. Поэтому `DateTime`, массивы и объекты требуют `readonly`, инициализируемого в конструкторе. Эти тонкости вы должны применить на практике, а не просто процитировать.

#### Что нужно сделать (пошагово)

1. **Создайте проект.** В терминале выполните:
   ```
   dotnet new console -n SensorReadingsAggregator -o SensorReadingsAggregator --framework net8.0
   cd SensorReadingsAggregator
   ```
   Убедитесь, что в `SensorReadingsAggregator.csproj` свойство `<TargetFramework>net8.0</TargetFramework>` присутствует и что язык по умолчанию — C# 12 (для .NET 8 это так). Откройте `Program.cs` и удалите шаблонный `Console.WriteLine("Hello, World!");`.

2. **Объявите константы времени компиляции.** В начале файла (top-level statements) объявите `const`-значения, которые действительно известны при компиляции: `const string AppName = "WarehouseMonitor";`, `const double AbsoluteZeroCelsius = -273.15;`, `const int MaxRetries = 3;`, `const double ValidMinCelsius = -40.0;`, `const double ValidMaxCelsius = 85.0;`. Имена — в PascalCase, как требует урок. Помните: `const` не подходит для `DateTime.UtcNow`, поэтому `CreatedAt` мы сделаем `readonly` позже.

3. **Объявите явные переменные для простых примитивов.** Для значений, тип которых читателю важно видеть сразу (например, пороги, флаги), используйте явный тип: `int readingCount = 0;`, `bool hasAlarm = false;`, `double currentReading = 0.0;`. Это соответствует best practice из урока: явный тип там, где он несёт смысл.

4. **Примените `var` там, где тип очевиден.** Создайте список показаний: `var readings = new List<double> { 18.5, 22.1, -50.0, 90.0, 21.7 };`. Тип `List<double>` очевиден из `new`, поэтому `var` уместен — это прямо рекомендация урока. Затем отфильтруйте валидные показания через LINQ: `var validReadings = readings.Where(r => r >= ValidMinCelsius && r <= ValidMaxCelsius).ToList();`. Здесь `var` скрывает длинный дженерик-тип — тоже рекомендация урока.

5. **Покажите, что `var` НЕ динамический.** Добавьте закомментированную строку `// currentReading = "hot";` с комментарием-объяснением, что это ошибка компиляции, потому что `currentReading` имеет тип `double`, зафиксированный при объявлении. Это закрепляет ключевую мысль урока: `var` лишь сокращение, тип фиксируется один раз.

6. **Реализуйте класс `SensorConfig` с `const` и `readonly`.** Объявите класс (можно в том же файле ниже top-level statements):
   ```csharp
   class SensorConfig
   {
       public const int MaxConnections = 100;       // compile-time
       public readonly DateTime InstalledAt;        // runtime, set in ctor
       public SensorConfig() => InstalledAt = DateTime.UtcNow;
   }
   ```
   Создайте экземпляр `var config = new SensorConfig();` и выведите `SensorConfig.MaxConnections` (через имя класса, как `Math.PI`) и `config.InstalledAt`.

7. **Используйте типизированный паттерн C# 12.** Урок показывает паттерн `is int value and > 10 and < 100`. Реализуйте аналогичную проверку для `object box = 42;` — если `box is int value and > 0 and < 1000`, напечатайте значение. Обязательно объясните в комментарии, почему нельзя сразу применить relational-паттерн к `object` (он не сравним) и почему сначала нужен типизированный паттерн `is int value`.

8. **Примените `foreach` с `var`.** Пройдитесь по `validReadings` через `foreach (var r in validReadings)` — здесь `var` уместен, тип понятен из коллекции (рекомендация урока). Внутри цикла используйте pattern matching, чтобы классифицировать показание: ниже нуля, в норме, выше нормы.

9. **Соблюдайте именование.** Все локальные переменные и параметры — в camelCase: `readingCount`, `hasAlarm`, `validReadings`, `r`. Константы — в PascalCase: `AppName`, `MaxRetries`. Никаких `d`, `s`, `x1`. Булевы переменные — с осмысленным префиксом `is`/`has`: `hasAlarm`, `isInRange`.

10. **Запустите и проверьте вывод.** Выполните `dotnet build`, затем `dotnet run`. Ожидаемый вывод должен содержать: имя приложения, количество валидных показаний, среднее значение, флаг тревоги, значение из типизированного паттерна и `MaxConnections`/`InstalledAt` из конфига. Убедитесь, что невалидные показания (`-50.0`, `90.0`) отфильтрованы.

#### Требования к решению

Решение должно компилироваться под C# 12 / .NET 8 без предупреждений (уровень `dotnet build` — `0 Warning(s)`). Используйте top-level statements в `Program.cs` — это современный стиль, рекомендованный для .NET 8. Классы (`SensorConfig`) можно объявить в том же файле ниже top-level statements.

Каждое использование `var` должно быть обосновано: либо тип очевиден из правой части (`new`, LINQ, `foreach`), либо есть комментарий, объясняющий выбор. Использование `var` с числовыми литералами вроде `var x = 0;` (где тип несёт смысл и непонятно, `int`/`long`/`double`) запрещено — это явное «неуместно» из урока. Если тип не очевиден из правой части (например, результат вызова метода с непрозрачным именем), используйте явный тип.

Все значения, известные на этапе компиляции и не зависящие от конфигурации, должны быть `const` с именами в PascalCase. Значения, разрешаемые в рантайме (например, `DateTime.UtcNow` или значение из аргументов командной строки), должны быть `readonly`, инициализируемым в конструкторе. Категорически запрещено пытаться объявить `const DateTime` — это ошибка компиляции, прямо описанная в уроке.

Имена переменных должны быть осмысленными и самодокументирующими: `daysRemaining`, а не `d`. Локальные переменные и параметры — camelCase; константы и публичные поля классов — PascalCase. Булевы переменные — с префиксом `is`/`has`/`can`. Код должен содержать минимальные комментарии RU+EN, объясняющие нетривиальные решения (почему `var`, почему `readonly`), но не дублирующие очевидное.

Должен присутствовать хотя бы один пример типизированного паттерна C# 12 (`is int value and > … and < …`) с комментарием, почему сначала нужна проверка типа, а затем relational-паттерны. Хотя бы один закомментированный пример ошибки компиляции, демонстрирующий, что `var` не динамический (попытка присвоить значение другого типа).

#### Тонкости и подводные камни

- **`var` — это НЕ `dynamic`.** Самая частая ошибка: новичок пишет `var x = 10;` и потом пытается `x = "hello";`, ожидая, что тип «подстроится». Он не подстроится — будет ошибка компиляции CS0029. Тип фиксируется один раз, при первом присваивании. В ДЗ специально оставьте закомментированную строку с такой попыткой и комментарием.

- **`var` с числовыми литералами.** `var x = 0;` даёт `int`, а не `long` или `double`. Если вам нужен `double`, пишите `var x = 0.0;` (суффикс не нужен, литерал с точкой — `double`) или явный тип `double x = 0;`. Для `long` — суффикс `L`: `var x = 0L;`. Для `float` — суффикс `f`: `var x = 0f;`. Урок явно предупреждает: не используйте `var` с числовыми литералами, где тип влияет на смысл.

- **`const` и `DateTime`.** `const DateTime CreatedAt = DateTime.UtcNow;` — ошибка компиляции CS0133, потому что значение `DateTime.UtcNow` вычисляется в рантайме. Решение — `readonly DateTime CreatedAt`, инициализируемый в конструкторе. Это классический пример из урока.

- **`const` «вшивается» в сборку.** Если библиотека экспортирует `public const int MaxConnections = 100;`, а вы меняете значение на `200`, все потребители библиотеки нужно перекомпилировать — иначе у них останется старое `100`, «вшитое» при прошлой компиляции. Для значений, которые могут меняться между релизами без перекомпиляции потребителей, используйте `readonly` или статические поля конфигурации. Это тонкость из раздела «Частые ошибки» урока.

- **`var` в `foreach`.** `foreach (var r in validReadings)` уместен: тип понятен из коллекции. Но если коллекция — `IEnumerable<object>` или `ArrayList`, тип `r` будет неочевиден, и лучше указать тип явно или привести.

- **PascalCase для констант, camelCase для локальных.** Константы `AppName`, `MaxRetries` — PascalCase. Локальные `readingCount`, `hasAlarm` — camelCase. Поля классов часто с подчёркиванием `_fieldName`, но публичные константы — без подчёркивания, в PascalCase. Смешивание стилей — частая придирка на ревью.

- **Relational-паттерны требуют сравнимого типа.** Нельзя написать `if (box is > 10)` для `object box` — компилятор не знает, что `box` сравним. Сначала типизированный паттерн `is int value`, потом relational `and > 10 and < 100`. Урок явно это разбирает.

#### Критерии приёмки

- [ ] Проект создаётся командой `dotnet new console` с `--framework net8.0` и собирается без ошибок.
- [ ] `dotnet build` выдаёт `0 Warning(s)` (или минимум без CS-предупреждений по теме урока).
- [ ] `dotnet run` выводит ожидаемые строки: имя приложения, статистику, значение паттерна, конфиг.
- [ ] Присутствует минимум 4 `const`-значения в PascalCase, известных на этапе компиляции.
- [ ] Присутствует класс `SensorConfig` с `public const` и `public readonly DateTime`, инициализируемым в конструкторе.
- [ ] `var` использован минимум 3 раза в уместных местах (`new`, LINQ, `foreach`) с обоснованием.
- [ ] Нет `var` с числовыми литералами в стиле `var x = 0;` без необходимости.
- [ ] Есть закомментированный пример ошибки компиляции, показывающий, что `var` не динамический.
- [ ] Есть типизированный паттерн C# 12 `is int value and > … and < …` с комментарием о необходимости проверки типа.
- [ ] Все локальные переменные и параметры — в camelCase, константы — в PascalCase.
- [ ] Нет однобуквенных или бессмысленных имён (`d`, `s`, `x1`), кроме счётчиков в коротких циклах.
- [ ] Булевы переменные имеют осмысленный префикс (`is`/`has`/`can`).
- [ ] Код содержит комментарии RU+EN для нетривиальных решений.
- [ ] Невалидные показания (`-50.0`, `90.0`) корректно отфильтрованы через LINQ и `const`-пороги.
- [ ] Решение запускается и завершается без исключений.

#### Подсказки (без прямого ответа)

- Вспомните правило урока: тип должен быть очевиден читателю из правой части. Если правая часть — `new List<double> {...}`, тип очевиден → `var`. Если правая часть — вызов метода с непрозрачным именем → явный тип.
- Для `DateTime.UtcNow` подойдёт `readonly`, а не `const`. Подумайте, почему: значение вычисляется в рантайме.
- Для фильтрации используйте `Where` и `ToList` из `System.Linq`. Тип результата — длинный дженерик, и это как раз случай, когда `var` улучшает читаемость.
- В pattern matching сначала «распакуйте» тип через `is int value`, затем применяйте `> ` и `<`.
- Чтобы продемонстрировать, что `var` не динамический, закомментируйте строку с попыткой присвоить строку числовой переменной и добавьте комментарий с номером ошибки компиляции (CS0029).

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8 — top-level statements
// Агрегатор показаний датчиков: var, const, readonly, типизированные паттерны
// Sensor readings aggregator: var, const, readonly, typed patterns

using System;
using System.Collections.Generic;
using System.Linq;

// 1) const — значения, известные на этапе компиляции / compile-time constants
// Имена в PascalCase, как требует урок / PascalCase names per the lesson
const string AppName = "WarehouseMonitor";
const double AbsoluteZeroCelsius = -273.15;
const int MaxRetries = 3;
const double ValidMinCelsius = -40.0;   // const OK: литерал double / OK: double literal
const double ValidMaxCelsius = 85.0;

// const DateTime CreatedAt = DateTime.UtcNow; // ❌ CS0133: значение вычисляется в рантайме
// → используем readonly в классе SensorConfig ниже / use readonly in SensorConfig below

// 2) Явные типы там, где тип несёт смысл / Explicit types where type carries meaning
int readingCount = 0;          // счетчик — важен тип int
bool hasAlarm = false;         // флаг — важен тип bool, префикс has
double currentReading = 0.0;   // показание — важен тип double

// 3) var уместен: тип очевиден из new / var OK: type obvious from new
var readings = new List<double> { 18.5, 22.1, -50.0, 90.0, 21.7 };

// var уместен: LINQ даёт длинный дженерик-тип / var OK: LINQ yields verbose generic
var validReadings = readings
    .Where(r => r >= ValidMinCelsius && r <= ValidMaxCelsius)
    .ToList();

readingCount = validReadings.Count;

// 4) var НЕ динамический: тип фиксируется один раз / var is NOT dynamic
// currentReading = "hot"; // ❌ CS0029: нельзя string присвоить double
currentReading = validReadings.Count > 0 ? validReadings.Average() : 0.0;
hasAlarm = currentReading > 30.0; // тревога, если среднее выше 30°C

Console.WriteLine($"{AppName}: readings={readingCount}, avg={currentReading:F2}°C, alarm={hasAlarm}");

// 5) foreach с var — тип понятен из коллекции / foreach with var: type clear from collection
foreach (var r in validReadings)
{
    // Pattern matching для классификации / classification via pattern matching
    string category = r switch
    {
        < 0.0   => "below zero",
        < 25.0  => "normal",
        _       => "above normal"
    };
    Console.WriteLine($"  reading {r:F1}°C — {category}");
}

// 6) Класс с const и readonly / Class with const and readonly
var config = new SensorConfig();
// const доступен через имя класса, как Math.PI / const accessed via class name, like Math.PI
Console.WriteLine($"MaxConnections={SensorConfig.MaxConnections}, InstalledAt={config.InstalledAt:O}");

// 7) Типизированный паттерн C# 12 / C# 12 typed pattern
// ВАЖНО: object не сравним, поэтому сначала is int value, затем relational-паттерны
// NOTE: object is not comparable, so first is int value, then relational patterns
object box = 42;
if (box is int value and > 0 and < 1000)
{
    Console.WriteLine($"Boxed int in range: {value}");
}

class SensorConfig
{
    public const int MaxConnections = 100;       // compile-time, PascalCase
    public readonly DateTime InstalledAt;        // runtime, immutable after ctor
    public SensorConfig() => InstalledAt = DateTime.UtcNow;
}
```

Разбор по строкам. Строки с `const` (блок 1) демонстрируют ключевую мысль урока: `const` подходит только для значений, известных при компиляции — строковых и числовых литералов. Имена в PascalCase (`AppName`, `MaxRetries`) соответствуют соглашению. Закомментированная попытка `const DateTime CreatedAt = DateTime.UtcNow;` — это классическая ошибка CS0133 из раздела «Частые ошибки»: значение `DateTime.UtcNow` вычисляется в рантайме, поэтому `const` неприменим; вместо него используем `readonly` в классе `SensorConfig`.

Блок 2 с явными типами (`int readingCount`, `bool hasAlarm`, `double currentReading`) иллюстрирует best practice «явный тип там, где он несёт смысл». Блок 3 с `var readings = new List<double> {...}` и `var validReadings = readings.Where(...).ToList();` — это прямая рекомендация урока: `var` уместен, когда тип очевиден из `new`, а также для LINQ-результатов с длинными дженерик-сигнатурами. Строка `readingCount = validReadings.Count` показывает, что явная переменная может быть переназначена значением того же типа.

Закомментированная строка `// currentReading = "hot";` с комментарием про CS0029 — это закрепление главного заблуждения: `var` (и вообще любая типизированная переменная) НЕ динамическая; тип фиксируется один раз. Строка `currentReading = validReadings.Count > 0 ? validReadings.Average() : 0.0;` корректна, потому что справа — `double`.

Цикл `foreach (var r in validReadings)` — рекомендация урока: `var` в `foreach` уместен, тип понятен из коллекции. Внутри используется switch-выражение с relational-паттернами по `double` — здесь `r` уже типизирован как `double`, поэтому relational-паттерны валидны (в отличие от `object`).

Класс `SensorConfig` объединяет `const` (`MaxConnections`, время компиляции, доступ через `SensorConfig.MaxConnections`, как `Math.PI`) и `readonly DateTime InstalledAt`, инициализируемый в конструкторе. Это иллюстрирует ключевое отличие из урока: `const` — заводская гравировка, `readonly` — надпись при установке.

Блок 7 с типизированным паттерном `box is int value and > 0 and < 1000` — прямой пример из урока: `object` не сравним, поэтому сначала типизированный паттерн `is int value` «распаковывает» тип, и только потом relational-паттерны `> 0` и `< 1000` становятся валидными. Комментарий явно объясняет эту последовательность.

#### Задания на углубление (бонус)

1. Добавьте поддержку аргументов командной строки: если передан аргумент, используйте его как имя приложения вместо `const AppName`. Подумайте, почему `AppName` уже не может быть `const` в этом сценарии и какой механизм (`readonly`, статическое поле) подойдёт.
2. Расширьте pattern matching: обрабатывайте `object box`, который может содержать `int`, `double` или `string`. Используйте несколько типизированных веток в switch-выражении.
3. Замените `List<double>` на `Span<double>` или `ReadOnlySpan<double>` и исследуйте, можно ли использовать `var` для результата `span.ToArray()`. Сравните читаемость.
4. Добавьте юнит-тест (через `dotnet test`) на метод фильтрации, проверяющий, что пороги `ValidMinCelsius`/`ValidMaxCelsius` работают корректно, включая граничные значения.

---

## Statement in English / Постановка на английском

#### Context & motivation

You are joining a team building a small "Sensor Readings Aggregator" component for a warehouse temperature monitoring system. The component takes raw readings, validates them against an acceptable range, aggregates statistics and prints a short report. The codebase is young, and the reviewer has already bounced several pull requests for exactly the kinds of mistakes the lesson warns about: someone wrote `var data = GetSomething();` where nobody could tell the type; someone tried `const DateTime CreatedAt = DateTime.UtcNow;` and was surprised by a compiler error; someone named variables `d`, `s`, `x1` and nobody understood what lived inside them.

Your task is to implement the component so that it compiles under C# 12 / .NET 8, runs correctly, and demonstrates mature command of every topic from lesson M02-L03: a deliberate choice between an explicit type and `var`, correct use of `const` and `readonly`, proper naming and the typed pattern `is int value and > 10 and < 100`. This is not an abstract drill: these are the decisions you will make in every pull request, and the ability to explain why `var` here and an explicit type there is valued far more than blindly following "always `var`" or "always explicit".

The lesson stresses that `var` is NOT dynamic typing. The type is fixed once, at compile time, and never changes afterward. So code like `var city = "Berlin"; city = 42;` simply will not compile. That is the central beginner misconception, and in this homework you will meet a situation where it is easy to fall into it. The lesson also reminds you that `const` works only with primitives, strings and `null` references — no runtime computation is allowed. Therefore `DateTime`, arrays and objects require `readonly`, initialized in a constructor. You must apply these subtleties in practice, not merely quote them.

#### What to do step by step

1. **Create the project.** In a terminal run:
   ```
   dotnet new console -n SensorReadingsAggregator -o SensorReadingsAggregator --framework net8.0
   cd SensorReadingsAggregator
   ```
   Confirm that `SensorReadingsAggregator.csproj` contains `<TargetFramework>net8.0</TargetFramework>` and that the default language is C# 12 (it is, on .NET 8). Open `Program.cs` and delete the template `Console.WriteLine("Hello, World!");`.

2. **Declare compile-time constants.** At the top of the file (top-level statements) declare `const` values that are genuinely known at compile time: `const string AppName = "WarehouseMonitor";`, `const double AbsoluteZeroCelsius = -273.15;`, `const int MaxRetries = 3;`, `const double ValidMinCelsius = -40.0;`, `const double ValidMaxCelsius = 85.0;`. Names are PascalCase, as the lesson requires. Remember that `const` does not work for `DateTime.UtcNow`, so `CreatedAt` will become `readonly` later.

3. **Declare explicit variables for simple primitives.** For values whose type a reader should see immediately (thresholds, flags), use an explicit type: `int readingCount = 0;`, `bool hasAlarm = false;`, `double currentReading = 0.0;`. This matches the lesson's best practice: explicit type where the type carries meaning.

4. **Apply `var` where the type is obvious.** Create the readings list: `var readings = new List<double> { 18.5, 22.1, -50.0, 90.0, 21.7 };`. The `List<double>` type is obvious from `new`, so `var` is appropriate — this is a direct recommendation of the lesson. Then filter the valid readings with LINQ: `var validReadings = readings.Where(r => r >= ValidMinCelsius && r <= ValidMaxCelsius).ToList();`. Here `var` hides a verbose generic type — again, a lesson recommendation.

5. **Show that `var` is NOT dynamic.** Add a commented-out line `// currentReading = "hot";` with an explanatory comment that this is a compile error because `currentReading` has type `double`, fixed at declaration. This reinforces the lesson's central point: `var` is only shorthand, the type is fixed once.

6. **Implement a `SensorConfig` class with `const` and `readonly`.** Declare the class (you may put it in the same file below the top-level statements):
   ```csharp
   class SensorConfig
   {
       public const int MaxConnections = 100;       // compile-time
       public readonly DateTime InstalledAt;        // runtime, set in ctor
       public SensorConfig() => InstalledAt = DateTime.UtcNow;
   }
   ```
   Create an instance `var config = new SensorConfig();` and print `SensorConfig.MaxConnections` (through the class name, like `Math.PI`) and `config.InstalledAt`.

7. **Use the C# 12 typed pattern.** The lesson shows the pattern `is int value and > 10 and < 100`. Implement the same check for `object box = 42;` — if `box is int value and > 0 and < 1000`, print the value. Be sure to explain in a comment why you cannot apply a relational pattern directly to `object` (it is not comparable) and why the typed pattern `is int value` must come first.

8. **Use `foreach` with `var`.** Iterate `validReadings` with `foreach (var r in validReadings)` — here `var` is appropriate, the type is clear from the collection (a lesson recommendation). Inside the loop use pattern matching to classify the reading: below zero, normal, above normal.

9. **Follow naming.** All locals and parameters are camelCase: `readingCount`, `hasAlarm`, `validReadings`, `r`. Constants are PascalCase: `AppName`, `MaxRetries`. No `d`, `s`, `x1`. Boolean variables carry a meaningful prefix `is`/`has`: `hasAlarm`, `isInRange`.

10. **Run and verify output.** Run `dotnet build`, then `dotnet run`. The expected output must contain: the application name, the count of valid readings, the average value, the alarm flag, the value from the typed pattern and `MaxConnections`/`InstalledAt` from the config. Confirm that the invalid readings (`-50.0`, `90.0`) are filtered out.

#### Requirements

The solution must compile under C# 12 / .NET 8 with no warnings (the `dotnet build` level is `0 Warning(s)`). Use top-level statements in `Program.cs` — the modern style recommended for .NET 8. Classes (`SensorConfig`) may be declared in the same file below the top-level statements.

Every use of `var` must be justified: either the type is obvious from the right-hand side (`new`, LINQ, `foreach`), or there is a comment explaining the choice. Using `var` with numeric literals such as `var x = 0;` (where the type carries meaning and it is unclear whether it is `int`/`long`/`double`) is forbidden — this is an explicit "not appropriate" from the lesson. When the type is not obvious from the right-hand side (for example, the result of a method call with an opaque name), use an explicit type.

All values known at compile time and independent of configuration must be `const` with PascalCase names. Values resolved at runtime (for example `DateTime.UtcNow` or a value from command-line arguments) must be `readonly`, initialized in a constructor. Trying to declare `const DateTime` is strictly forbidden — it is a compile error, exactly as described in the lesson.

Variable names must be meaningful and self-documenting: `daysRemaining`, not `d`. Locals and parameters are camelCase; constants and public class fields are PascalCase. Boolean variables carry a prefix `is`/`has`/`can`. The code must contain minimal RU+EN comments explaining non-trivial decisions (why `var`, why `readonly`) but not restating the obvious.

There must be at least one example of the C# 12 typed pattern (`is int value and > … and < …`) with a comment explaining why the type check must come before the relational patterns. At least one commented-out example of a compile error demonstrating that `var` is not dynamic (an attempt to assign a value of a different type).

#### Pitfalls

- **`var` is NOT `dynamic`.** The most frequent mistake: a beginner writes `var x = 10;` and then tries `x = "hello";`, expecting the type to "adapt". It will not — you get compile error CS0029. The type is fixed once, at the first assignment. In the homework deliberately leave a commented line with such an attempt and a comment.

- **`var` with numeric literals.** `var x = 0;` yields `int`, not `long` or `double`. If you need `double`, write `var x = 0.0;` (no suffix needed, a literal with a decimal point is `double`) or an explicit type `double x = 0;`. For `long` use the `L` suffix: `var x = 0L;`. For `float` use `f`: `var x = 0f;`. The lesson explicitly warns: do not use `var` with numeric literals where the type carries meaning.

- **`const` and `DateTime`.** `const DateTime CreatedAt = DateTime.UtcNow;` is compile error CS0133, because `DateTime.UtcNow` is computed at runtime. The fix is `readonly DateTime CreatedAt`, initialized in a constructor. This is the classic example from the lesson.

- **`const` is baked into the assembly.** If a library exposes `public const int MaxConnections = 100;` and you change it to `200`, every consumer of the library must be recompiled — otherwise they keep the old `100` baked in at their previous compilation. For values that may change between releases without recompiling consumers, use `readonly` or static configuration fields. This subtlety comes straight from the lesson's "Common Mistakes" section.

- **`var` in `foreach`.** `foreach (var r in validReadings)` is fine: the type is clear from the collection. But if the collection is `IEnumerable<object>` or `ArrayList`, the type of `r` is unclear, and an explicit type or a cast is preferable.

- **PascalCase for constants, camelCase for locals.** Constants `AppName`, `MaxRetries` are PascalCase. Locals `readingCount`, `hasAlarm` are camelCase. Class fields are often prefixed with an underscore `_fieldName`, but public constants have no underscore and use PascalCase. Mixing styles is a common review nitpick.

- **Relational patterns need a comparable type.** You cannot write `if (box is > 10)` for `object box` — the compiler does not know `box` is comparable. First the typed pattern `is int value`, then the relational `and > 10 and < 100`. The lesson walks through exactly this.

#### Acceptance criteria

- [ ] The project is created with `dotnet new console --framework net8.0` and builds without errors.
- [ ] `dotnet build` reports `0 Warning(s)` (or at least no CS warnings related to the lesson).
- [ ] `dotnet run` prints the expected lines: app name, statistics, pattern value, config.
- [ ] At least 4 `const` values in PascalCase known at compile time are present.
- [ ] A `SensorConfig` class exists with `public const` and `public readonly DateTime` initialized in the constructor.
- [ ] `var` is used at least 3 times in appropriate places (`new`, LINQ, `foreach`) with justification.
- [ ] There is no `var` with numeric literals in the `var x = 0;` style without a reason.
- [ ] A commented-out compile error demonstrates that `var` is not dynamic.
- [ ] A C# 12 typed pattern `is int value and > … and < …` is present with a comment on the type check.
- [ ] All locals and parameters are camelCase; constants are PascalCase.
- [ ] No single-letter or meaningless names (`d`, `s`, `x1`), except short loop counters.
- [ ] Boolean variables carry a meaningful prefix (`is`/`has`/`can`).
- [ ] The code contains RU+EN comments for non-trivial decisions.
- [ ] Invalid readings (`-50.0`, `90.0`) are correctly filtered out via LINQ and `const` thresholds.
- [ ] The program runs and exits without exceptions.

#### Hints (no direct answer)

- Recall the lesson's rule: the type must be obvious to the reader from the right-hand side. If the right-hand side is `new List<double> {...}`, the type is obvious → `var`. If it is a call to a method with an opaque name → explicit type.
- For `DateTime.UtcNow`, `readonly` fits, not `const`. Think about why: the value is computed at runtime.
- For filtering use `Where` and `ToList` from `System.Linq`. The result type is a verbose generic, and this is precisely when `var` improves readability.
- In pattern matching, first "unwrap" the type with `is int value`, then apply `>` and `<`.
- To demonstrate that `var` is not dynamic, comment out a line that tries to assign a string to a numeric variable and add a comment with the compile error code (CS0029).

#### Reference solution walk-through

```csharp
// C# 12 / .NET 8 — top-level statements
// Sensor readings aggregator: var, const, readonly, typed patterns

using System;
using System.Collections.Generic;
using System.Linq;

// 1) const — values known at compile time / compile-time constants
// PascalCase names, as the lesson requires
const string AppName = "WarehouseMonitor";
const double AbsoluteZeroCelsius = -273.15;
const int MaxRetries = 3;
const double ValidMinCelsius = -40.0;   // const OK: double literal
const double ValidMaxCelsius = 85.0;

// const DateTime CreatedAt = DateTime.UtcNow; // ❌ CS0133: runtime-computed value
// → use readonly in the SensorConfig class below

// 2) Explicit types where the type carries meaning
int readingCount = 0;          // counter — int type matters
bool hasAlarm = false;         // flag — bool type matters, has prefix
double currentReading = 0.0;   // reading — double type matters

// 3) var is appropriate: type obvious from new
var readings = new List<double> { 18.5, 22.1, -50.0, 90.0, 21.7 };

// var is appropriate: LINQ yields a verbose generic type
var validReadings = readings
    .Where(r => r >= ValidMinCelsius && r <= ValidMaxCelsius)
    .ToList();

readingCount = validReadings.Count;

// 4) var is NOT dynamic: the type is fixed once
// currentReading = "hot"; // ❌ CS0029: cannot assign string to double
currentReading = validReadings.Count > 0 ? validReadings.Average() : 0.0;
hasAlarm = currentReading > 30.0; // alarm if the average exceeds 30°C

Console.WriteLine($"{AppName}: readings={readingCount}, avg={currentReading:F2}°C, alarm={hasAlarm}");

// 5) foreach with var — type clear from the collection
foreach (var r in validReadings)
{
    // Pattern matching for classification
    string category = r switch
    {
        < 0.0   => "below zero",
        < 25.0  => "normal",
        _       => "above normal"
    };
    Console.WriteLine($"  reading {r:F1}°C — {category}");
}

// 6) Class with const and readonly
var config = new SensorConfig();
// const is accessed through the class name, like Math.PI
Console.WriteLine($"MaxConnections={SensorConfig.MaxConnections}, InstalledAt={config.InstalledAt:O}");

// 7) C# 12 typed pattern
// NOTE: object is not comparable, so first is int value, then relational patterns
object box = 42;
if (box is int value and > 0 and < 1000)
{
    Console.WriteLine($"Boxed int in range: {value}");
}

class SensorConfig
{
    public const int MaxConnections = 100;       // compile-time, PascalCase
    public readonly DateTime InstalledAt;        // runtime, immutable after ctor
    public SensorConfig() => InstalledAt = DateTime.UtcNow;
}
```

Line-by-line walk-through. The `const` lines (block 1) demonstrate the lesson's central point: `const` suits only values known at compile time — string and numeric literals. The PascalCase names (`AppName`, `MaxRetries`) follow the convention. The commented-out attempt `const DateTime CreatedAt = DateTime.UtcNow;` is the classic CS0133 error from the "Common Mistakes" section: `DateTime.UtcNow` is computed at runtime, so `const` does not apply; we use `readonly` in the `SensorConfig` class instead.

Block 2 with explicit types (`int readingCount`, `bool hasAlarm`, `double currentReading`) illustrates the best practice "explicit type where the type carries meaning". Block 3 with `var readings = new List<double> {...}` and `var validReadings = readings.Where(...).ToList();` is a direct lesson recommendation: `var` is appropriate when the type is obvious from `new`, and also for LINQ results with verbose generic signatures. The line `readingCount = validReadings.Count` shows that an explicitly typed variable may be reassigned a value of the same type.

The commented line `// currentReading = "hot";` with the CS0029 note reinforces the chief misconception: `var` (and indeed any typed variable) is NOT dynamic; the type is fixed once. The line `currentReading = validReadings.Count > 0 ? validReadings.Average() : 0.0;` is valid because the right-hand side is `double`.

The loop `foreach (var r in validReadings)` is a lesson recommendation: `var` in `foreach` is appropriate, the type is clear from the collection. Inside, a switch expression with relational patterns on `double` is used — here `r` is already typed as `double`, so relational patterns are valid (unlike for `object`).

The `SensorConfig` class combines `const` (`MaxConnections`, compile time, accessed as `SensorConfig.MaxConnections`, like `Math.PI`) and `readonly DateTime InstalledAt`, initialized in the constructor. This illustrates the lesson's key distinction: `const` is a factory engraving, `readonly` is a label written at installation.

Block 7 with the typed pattern `box is int value and > 0 and < 1000` is a direct example from the lesson: `object` is not comparable, so the typed pattern `is int value` first "unwraps" the type, and only then do the relational patterns `> 0` and `< 1000` become valid. The comment spells out this ordering.

#### Going deeper (bonus)

1. Add command-line argument support: if an argument is passed, use it as the application name instead of `const AppName`. Think about why `AppName` can no longer be `const` in this scenario and which mechanism (`readonly`, a static field) fits.
2. Extend pattern matching: handle an `object box` that may hold an `int`, a `double` or a `string`. Use several typed branches in a switch expression.
3. Replace `List<double>` with `Span<double>` or `ReadOnlySpan<double>` and investigate whether `var` can be used for the result of `span.ToArray()`. Compare readability.
4. Add a unit test (via `dotnet test`) for the filtering method, verifying that the `ValidMinCelsius`/`ValidMaxCelsius` thresholds work correctly, including boundary values.

---

#### Чек-лист сдачи / Submission checklist
- [ ] Проект собирается под .NET 8 без ошибок и предупреждений (RU)
- [ ] Присутствуют `const`, `readonly`, уместный `var`, явные типы, pattern matching (RU)
- [ ] Имена в camelCase/PascalCase, нет мусорных имён (RU)
- [ ] Есть закомментированная ошибка CS0029 и комментарий про CS0133 (RU)
- [ ] Вывод `dotnet run` соответствует ожидаемому (RU)
- [ ] The project builds under .NET 8 with no errors or warnings (EN)
- [ ] `const`, `readonly`, appropriate `var`, explicit types, pattern matching are present (EN)
- [ ] Names follow camelCase/PascalCase, no junk names (EN)
- [ ] A commented CS0029 error and a CS0133 comment are present (EN)
- [ ] The `dotnet run` output matches the expected (EN)

#### Ресурсы / Resources
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/implicitly-typed-local-variables — Implicitly typed local variables / Неявно типизированные локальные переменные
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/const — const keyword (C# reference) / Ключевое слово const
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/readonly — readonly (C# reference) / Ключевое слово readonly
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/fundamentals/coding-style/coding-conventions — C# coding conventions / Соглашения о кодировании C#
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/operators/patterns — Patterns (C# reference) / Паттерны

---
[← К уроку M02-L03](lesson-M02-L03-var-const.md) | [⬆ К модулю M02](../README.md) | [Следующее ДЗ →](homework-M02-L04-operators.md)
---
