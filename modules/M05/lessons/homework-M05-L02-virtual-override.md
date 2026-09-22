---
[← К уроку M05-L02](lesson-M05-L02-virtual-override.md) | [⬆ К модулю M05](../README.md) | [Следующее ДЗ →](homework-M05-L03-abstract.md)
---

### Домашнее задание M05-L02: virtual/override/new / Homework M05-L02: virtual/override/new

**Урок / Lesson:** M05-L02
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться осознанно применять `virtual`, `override`, `new` и `sealed override` в иерархии классов C# 12 / .NET 8; закрепить разницу между динамическим связыванием (полиморфизмом) и сокрытием методов; научиться писать тесты, которые доказывают, какая версия метода вызывается через ссылку базового типа. (EN) Learn to deliberately apply `virtual`, `override`, `new`, and `sealed override` in a C# 12 / .NET 8 class hierarchy; internalize the difference between dynamic dispatch (polymorphism) and method hiding; write tests that prove which method version is invoked through a base-typed reference.

#### Связь с уроком / Connection to the lesson
(RU) Домашнее задание напрямую опирается на пример урока с классами `Employee` / `Developer` / `Manager` / `LegacyEmployee`, но расширяет его до полноценной системы расчёта бонусов, где каждый тип сотрудника имеет свою логику. Вы повторите все четыре модификатора (`virtual`, `override`, `new`, `sealed override`), столкнётесь с предупреждением о сокрытии метода и научитесь отличать полиморфный вызов от неполиморфного. (EN) The homework directly builds on the lesson's example with `Employee` / `Developer` / `Manager` / `LegacyEmployee`, but extends it into a full bonus-calculation system where each employee type has its own logic. You will reuse all four modifiers (`virtual`, `override`, `new`, `sealed override`), face the method-hiding warning, and learn to tell a polymorphic call from a non-polymorphic one.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Вы пришли в небольшую IT-компанию «Полиморф Лабс», которая быстро растёт и теперь employs сотрудников нескольких категорий: обычные сотрудники (`Employee`), разработчики (`Developer`), менеджеры (`Manager`) и торговые представители (`SalesPerson`). Бухгалтерия устала писать `switch` по типам сотрудников для расчёта годового бонуса — каждое добавление новой категории требует правок в пяти местах, тесты ломаются, а в продакшене периодически всплывают баги, когда менеджеру начисляют бонус как обычному сотруднику. Руководство просит вас спроектировать единую объектную модель, в которой расчёт бонуса и описание обязанностей полиморфны: один и тот же вызов `employee.CalculateBonus()` через ссылку типа `Employee` должен вернуть корректную сумму в зависимости от фактического типа объекта.

Параллельно в компании есть легаси-подрядчики (`LegacyContractor`), которые не вписываются в новую модель: их отчётность считается по старому формату, и менять их поведение нельзя, потому что от него зависит интеграция с внешней бухгалтерской системой 2017 года. Вам нужно показать, что произойдёт, если попытаться «переопределить» отчёт подрядчика через `new` вместо `override`, и объяснить бухгалтерии, почему через ссылку базового класса будет вызвана старая версия. Это классический случай, который разбирается в уроке: `new` скрывает, но не переопределяет.

Наконец, для роли `Developer` руководство хочет навсегда зафиксировать формат строкового представления (чтобы лог-парсеры не сломались при рефакторинге) и помочь JIT-оптимизациям. Для этого вы примените `sealed override`. Вся работа должна сопровождаться модульными тестами на xUnit, которые доказывают каждое поведение: какой метод вызывается через какую ссылку. Это научит вас не доверять интуиции, а проверять диспетчеризацию тестами — именно так, как требует чек-лист самопроверки из урока.

#### Что нужно сделать (пошагово)

1. Создайте новый проект консольного приложения и проект тестов. Выполните команды:
   ```
   dotnet new sln -n PolymorphLabs
   dotnet new console -n PolymorphLabs.App -o src/PolymorphLabs.App
   dotnet new xunit -n PolymorphLabs.Tests -o tests/PolymorphLabs.Tests
   dotnet sln add src/PolymorphLabs.App tests/PolymorphLabs.Tests
   dotnet add tests/PolymorphLabs.Tests reference src/PolymorphLabs.App
   ```
   Убедитесь, что в обоих `.csproj` указан `<TargetFramework>net8.0</TargetFramework>` и `<LangVersion>latest</LangVersion>`, чтобы был доступен C# 12 (collection expressions, primary-конструкторы классов, `init`-свойства).

2. В файле `src/PolymorphLabs.App/Employee.cs` опишите базовый класс `Employee` со свойствами `Name` (string, `init`) и `BaseSalary` (decimal, `init`). Добавьте виртуальный метод `public virtual decimal CalculateBonus()`, который возвращает 10% от `BaseSalary`. Добавьте виртуальный метод `public virtual string DescribeDuties()`, возвращающий строку вида `"{Name}: общие обязанности"`. Добавьте НЕ виртуальный метод `public string FormatReport()`, возвращающий `"{Name}: стандартный отчёт"` — он понадобится для демонстрации `new`.

3. В `Developer.cs` создайте класс `Developer : Employee` со свойством `Stack` (string, `init`, по умолчанию `"C#"`). Переопределите `CalculateBonus()` так, чтобы бонус разработчика был 15% от `BaseSalary` плюс 2000 за каждую технологию в `Stack` (если в `Stack` несколько технологий через запятую — считайте количество). Переопределите `DescribeDuties()`. Переопределите `ToString()` с модификатором `sealed override`, чтобы зафиксировать формат `"Developer[{Name}, {Stack}]"` и помочь JIT-оптимизациям.

4. В `Manager.cs` создайте `Manager : Employee` со свойством `TeamSize` (int, `init`). Переопределите `CalculateBonus()`: 12% от `BaseSalary` плюс `TeamSize * 1000`. В переопределении ОБЯЗАТЕЛЬНО вызовите `base.DescribeDuties()` внутри `DescribeDuties()`, дополнив его информацией о команде, — чтобы продемонстрировать сохранение родительской логики.

5. В `SalesPerson.cs` создайте `SalesPerson : Employee` со свойством `DealsClosed` (int, `init`). Бонус: 8% от `BaseSalary` плюс `DealsClosed * 500`. Переопределите оба виртуальных метода.

6. В `LegacyContractor.cs` создайте `LegacyContractor : Employee`. Намеренно используйте `new` для сокрытия `FormatReport()`: `public new string FormatReport() => $"{Name}: устаревший отчёт подрядчика";`. Не переопределяйте `CalculateBonus()` — пусть подрядчик получает стандартные 10%. Цель — продемонстрировать разницу `override` vs `new`.

7. В `Program.cs` (top-level statements) соберите `List<Employee>` с одним объектом каждого типа, используя collection expressions C# 12: `List<Employee> staff = [new Employee{...}, new Developer{...}, ...];`. В цикле `foreach` выведите `e.CalculateBonus()` и `e.DescribeDuties()`. Затем создайте переменную `Employee legacy = new LegacyContractor{...};` и выведите `legacy.FormatReport()` и `((LegacyContractor)legacy).FormatReport()`, чтобы наглядно показать две разные версии. Запустите `dotnet run --project src/PolymorphLabs.App` и сохраните вывод.

8. В тестовом проекте напишите тесты на xUnit: `CalculateBonus_ReturnsExpected_ForEachType`, `PolymorphicDispatch_ThroughBaseReference`, `NewHidesButDoesNotOverride_ThroughBaseReference`, `SealedOverride_PreventsFurtherOverride` (последний — через попытку компиляции производного класса, можно оформить как комментарий-объяснение или через `Assert.Throws` если удастся спровоцировать). Запустите `dotnet test` и добейтесь зелёного.

#### Требования к решению

Решение должно компилироваться без предупреждений (в частности, предупреждение CS0108 о сокрытии метода должно быть либо устранено через явный `new`, либо объяснено). Все публичные методы базового класса, которые планируется переопределять, должны быть помечены `virtual`; все переопределения — `override` с сохранением сигнатуры и контракта (правило Лисков: постусловия не должны ослабевать). Использование `new` допускается только для `LegacyContractor.FormatReport()` и должно быть явно задокументировано комментарием, почему выбрано сокрытие, а не переопределение.

`sealed override` должен применяться к `Developer.ToString()` с комментарием о фиксации формата и JIT-оптимизации. В `Manager.DescribeDuties()` обязателен вызов `base.DescribeDuties()`. Код должен использовать возможности C# 12: collection expressions для инициализации списка, `init`-свойства, при возможности — pattern matching в расчётах. Проект должен собираться командой `dotnet build` без ошибок, тесты — проходить командой `dotnet test`. Имена файлов и классов должны точно соответствовать шагам. В выводе `Program.cs` должно быть видно, что через ссылку `Employee` вызываются корректные полиморфные версии `CalculateBonus()` и `DescribeDuties()`, а `FormatReport()` через базовую ссылку вызывает родительскую версию, а через приведение — скрытую.

#### Тонкости и подводные камни

Главная тонкость, которую разбирает урок: `override` выбирает версию по фактическому типу объекта во время выполнения (динамическое связывание), а `new` — по типу ссылки во время компиляции (статическое связывание). Студенты регулярно путают их и «удивляются», почему через `Employee legacy = new LegacyContractor()` вызывается базовый `FormatReport()`. Объясните в комментариях, что `FormatReport()` не виртуальный, поэтому диспетчеризации нет вообще; `new` лишь подавляет предупреждение, но не создаёт полиморфный слот. Частая ошибка — забыть `virtual` в базовом классе и получить ошибку компиляции на `override`. Другая частая ошибка — использовать `new` там, где нужен `override`, и получить «молчащий» баг: полиморфизм сломан, тесты зелёные, но в продакшене вызывается не та версия. Поэтому почти всегда выбирают `override`; `new` оправдан только когда вы не контролируете базовый класс.

`sealed override` фиксирует поведение: попытка переопределить `ToString()` в классе-наследнике `Developer` даст ошибку компиляции. Это и защита инварианта (формат лога), и сигнал JIT для девиртуализации. Важно: `sealed` на методе имеет смысл только вместе с `override`; `sealed` на обычном методе не пишут. Также помните про `base.Method()`: если родительский метод выполнял важную работу (например, логирование или валидацию), забытый `base.` её потеряет. В `Manager.DescribeDuties()` вы обязаны вызвать `base.DescribeDuties()`, чтобы сохранить родительскую строку и дополнить её. Наконец, следите за контрактом Лискова: переопределённый `CalculateBonus()` не должен возвращать отрицательные значения или нарушать инварианты базового метода.

#### Критерии приёмки

- [ ] Проект `PolymorphLabs.sln` собирается командой `dotnet build` без ошибок и предупреждений.
- [ ] Целевая платформа — `net8.0`, язык — C# 12 (`LangVersion` явно или неявно latest).
- [ ] Базовый класс `Employee` содержит `virtual CalculateBonus()` и `virtual DescribeDuties()`.
- [ ] `Employee.FormatReport()` НЕ виртуальный (для демонстрации `new`).
- [ ] `Developer` переопределяет `CalculateBonus()` и `DescribeDuties()` через `override`.
- [ ] `Developer.ToString()` помечен `sealed override` с комментарием о фиксации формата.
- [ ] `Manager.CalculateBonus()` учитывает `TeamSize`.
- [ ] `Manager.DescribeDuties()` вызывает `base.DescribeDuties()`.
- [ ] `SalesPerson` переопределяет оба виртуальных метода.
- [ ] `LegacyContractor` использует `new` для `FormatReport()` с поясняющим комментарием.
- [ ] `Program.cs` использует collection expressions (`[...]`) для инициализации `List<Employee>`.
- [ ] Вывод `dotnet run` демонстрирует полиморфную диспетчеризацию (разные суммы бонусов).
- [ ] Вывод показывает две разные версии `FormatReport()` в зависимости от типа ссылки.
- [ ] Все тесты xUnit проходят (`dotnet test` зелёный).
- [ ] Тест `PolymorphicDispatch_ThroughBaseReference` явно проверяет вызов через `Employee`.
- [ ] В комментариях RU+EN объяснён выбор `override` vs `new` для каждого случая.

#### Подсказки (без прямого ответа)

- Вспомните аналогию из урока про должностную инструкцию: кто «видит» объект как `Employee`, тот видит общую инструкцию, но выполняется специфичная — только если метод `virtual`/`override`.
- Чтобы проверить, что `new` не создаёт полиморфизма, создайте две переменные одного объекта через разные типы ссылок и сравните вывод.
- Для подсчёта технологий в `Stack` разработчика используйте `Stack.Split(',').Length` и pattern matching.
- Не забудьте: `sealed override` не позволяет наследнику снова написать `override` — это можно проверить попыткой компиляции.
- Если компилятор ругается на `override`, проверьте, помечен ли базовый метод `virtual`.
- Если видите предупреждение CS0108, значит вы написали метод с тем же именем без `new` и без `override` — решите осознанно, что вы хотели.

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8 — PolymorphLabs: virtual / override / new / sealed override
// Файл: src/PolymorphLabs.App/Employee.cs
namespace PolymorphLabs;

// Базовый класс: виртуальные методы разрешают переопределение.
// Base class: virtual methods allow overriding.
public class Employee
{
    public string Name { get; init; } = string.Empty;
    public decimal BaseSalary { get; init; }

    // virtual: разрешаем полиморфную замену в наследниках.
    // virtual: allow polymorphic replacement in derived classes.
    public virtual decimal CalculateBonus() => BaseSalary * 0.10m;

    public virtual string DescribeDuties() => $"{Name}: общие обязанности / general duties";

    // НЕ виртуальный: нужен для демонстрации new (сокрытия).
    // Non-virtual: used to demonstrate new (hiding).
    public string FormatReport() => $"{Name}: стандартный отчёт / standard report";
}

// Файл: Developer.cs
public class Developer : Employee
{
    public string Stack { get; init; } = "C#";

    public override decimal CalculateBonus()
    {
        // 15% + 2000 за каждую технологию. Сохраняем контракт Лискова: бонус >= 0.
        // 15% + 2000 per technology. Preserve Liskov: bonus >= 0.
        var techCount = Stack.Split(',', StringSplitOptions.TrimEntries).Length;
        return BaseSalary * 0.15m + 2000m * techCount;
    }

    public override string DescribeDuties() => $"{Name}: пишет код на {Stack} / writes code in {Stack}";

    // sealed override: фиксируем формат и помогаем JIT-оптимизациям.
    // sealed override: lock the format and help JIT optimizations.
    public sealed override string ToString() => $"Developer[{Name}, {Stack}]";
}

// Файл: Manager.cs
public class Manager : Employee
{
    public int TeamSize { get; init; }

    public override decimal CalculateBonus() => BaseSalary * 0.12m + TeamSize * 1000m;

    public override string DescribeDuties()
    {
        // base.DescribeDuties() сохраняет родительскую логику и дополняет её.
        // base.DescribeDuties() preserves parent logic and extends it.
        return $"{base.DescribeDuties()} + управляет командой из {TeamSize} / manages team of {TeamSize}";
    }
}

// Файл: SalesPerson.cs
public class SalesPerson : Employee
{
    public int DealsClosed { get; init; }

    public override decimal CalculateBonus() => BaseSalary * 0.08m + DealsClosed * 500m;
    public override string DescribeDuties() => $"{Name}: закрывает сделки ({DealsClosed}) / closes deals";
}

// Файл: LegacyContractor.cs
public class LegacyContractor : Employee
{
    // new: СОКРЫВАЕМ, а не переопределяем. Через ссылку Employee вызовется базовая версия.
    // Это намеренный выбор: легаси-интеграция требует старого поведения через базовый тип.
    // new: HIDE, not override. Through an Employee reference the base version runs.
    // Intentional: legacy integration needs old behavior through the base type.
    public new string FormatReport() => $"{Name}: устаревший отчёт подрядчика / legacy contractor report";
}

// Файл: Program.cs (top-level statements)
using PolymorphLabs;

List<Employee> staff =
[
    new Employee   { Name = "Иван",  BaseSalary = 100_000 },
    new Developer  { Name = "Анна",  BaseSalary = 150_000, Stack = "C#, F#, TypeScript" },
    new Manager    { Name = "Олег",  BaseSalary = 140_000, TeamSize = 6 },
    new SalesPerson{ Name = "Мария", BaseSalary = 110_000, DealsClosed = 12 },
    new LegacyContractor { Name = "Пётр", BaseSalary = 90_000 }
];

// Полиморфный вызов: версия выбирается по фактическому типу объекта.
// Polymorphic call: version chosen by actual runtime type.
foreach (var e in staff)
    Console.WriteLine($"{e.DescribeDuties()} → бонус {e.CalculateBonus():N0}");

// override vs new — ключевая разница через одну и ту же ссылку.
// override vs new — the key difference through the same reference.
Employee legacy = new LegacyContractor { Name = "Пётр", BaseSalary = 90_000 };
Console.WriteLine(legacy.FormatReport());                      // стандартный отчёт
Console.WriteLine(((LegacyContractor)legacy).FormatReport());  // устаревший отчёт
```

Разбор по строкам. Базовый `Employee.CalculateBonus()` помечен `virtual` — это необходимое условие для любого `override` в наследниках; без `virtual` компилятор отклонит `override` (частая ошибка из урока). `Developer.CalculateBonus()` использует `override`, поэтому через ссылку `Employee` вызовется именно разработческая версия — это и есть динамическое связывание, ради которого существует полиморфизм. Подсчёт технологий через `Split` с `StringSplitOptions.TrimEntries` — современный идиоматичный C#. `Developer.ToString()` помечен `sealed override`: с одной стороны, `override`替换 унаследованный от `object` виртуальный метод, с другой — `sealed` запрещает дальнейшее переопределение и даёт JIT сигнал для девиртуализации. В `Manager.DescribeDuties()` обязательный вызов `base.DescribeDuties()` демонстрирует best practice из урока: сохраняем родительскую логику и расширяем её, не теряя инвариантов. `SalesPerson` показывает ещё один независимый полиморфный вариант. `LegacyContractor.FormatReport()` использует `new`: это сокрытие, а не переопределение — метод `FormatReport()` в базовом классе не виртуальный, поэтому полиморфного слота нет; `new` лишь подавляет предупреждение CS0108. Финальная демонстрация в `Program.cs` доказывает главное: через `Employee legacy` вызывается базовая версия `FormatReport()`, а через приведение типа — скрытая; в то же время `CalculateBonus()` и `DescribeDuties()` диспетчеризируются корректно, потому что они `virtual`/`override`. Collection expression `[...]` — это C# 12. Всё вместе демонстрирует четыре модификатора и разницу между полиморфизмом и сокрытием, которую требует урок.

#### Задания на углубление (бонус)

1. Добавьте класс `Intern : Employee` с переопределением `CalculateBonus()`, возвращающим 0, и переопределением `ToString()` с `sealed override`. Проверьте, что попытка создать `SeniorIntern : Intern` с `override ToString()` не компилируется.
2. Реализуйте метод расширения `decimal TotalBonus(this IEnumerable<Employee> employees)`, использующий LINQ `Sum`, и напишите тест, проверяющий сумму по смешанному списку.
3. Добавьте виртуальное свойство `virtual string Department => "General"` и переопределите его в каждом классе; исследуйте, работают ли виртуальные свойства так же, как методы.
4. Сравните производительность `virtual` и `sealed override` методов с помощью `BenchmarkDotNet`: измерьте разницу во времени вызова в цикле на 10 миллионов итераций и объясните результат через девиртуализацию.

---

## Statement in English / Постановка на английском

#### Context & motivation

You have joined a small but fast-growing IT company, Polymorph Labs, which now employs several categories of workers: regular employees (`Employee`), developers (`Developer`), managers (`Manager`), and salespeople (`SalesPerson`). The accounting team is tired of maintaining a giant `switch` statement over employee types to compute the yearly bonus — every new category forces edits in five places, tests break, and production keeps surfacing bugs where a manager accidentally receives the bonus of a regular employee. Management asks you to design a single object model in which both bonus calculation and duty description are polymorphic: the same call `employee.CalculateBonus()` through a reference typed as `Employee` must return the correct amount depending on the actual runtime type of the object. This is exactly the dynamic dispatch mechanism the lesson describes, and your job is to make it real, tested, and self-documenting.

In parallel, the company has legacy contractors (`LegacyContractor`) that do not fit the new model. Their reporting is computed with an old format and must not change, because a 2017-era external accounting system depends on it. You need to demonstrate what happens when you attempt to “override” a contractor's report using `new` instead of `override`, and explain to accounting why the old version still runs when the object is held through a base-typed reference. This is the canonical scenario from the lesson: `new` hides, but does not override, and the dispatch follows the compile-time type of the reference rather than the runtime type of the object.

Finally, for the `Developer` role, management wants to permanently lock the string representation format so that log parsers do not break during refactoring, and to help the JIT apply optimizations. For that you will use `sealed override`. All work must be accompanied by xUnit unit tests proving each behavior: which method is invoked through which reference. This teaches you not to trust intuition but to verify dispatch with tests — exactly as the lesson's self-check checklist demands.

#### What to do step by step

1. Create a new console application project and a test project. Run the commands:
   ```
   dotnet new sln -n PolymorphLabs
   dotnet new console -n PolymorphLabs.App -o src/PolymorphLabs.App
   dotnet new xunit -n PolymorphLabs.Tests -o tests/PolymorphLabs.Tests
   dotnet sln add src/PolymorphLabs.App tests/PolymorphLabs.Tests
   dotnet add tests/PolymorphLabs.Tests reference src/PolymorphLabs.App
   ```
   Verify that both `.csproj` files declare `<TargetFramework>net8.0</TargetFramework>` and `<LangVersion>latest</LangVersion>` so that C# 12 features (collection expressions, class primary constructors, `init` properties) are available.

2. In `src/PolymorphLabs.App/Employee.cs` define the base class `Employee` with properties `Name` (string, `init`) and `BaseSalary` (decimal, `init`). Add a virtual method `public virtual decimal CalculateBonus()` returning 10% of `BaseSalary`. Add a virtual method `public virtual string DescribeDuties()` returning `"{Name}: general duties"`. Add a NON-virtual method `public string FormatReport()` returning `"{Name}: standard report"` — it will be used to demonstrate `new`.

3. In `Developer.cs` create `Developer : Employee` with a `Stack` property (string, `init`, default `"C#"`). Override `CalculateBonus()` so that the developer's bonus is 15% of `BaseSalary` plus 2000 per technology listed in `Stack` (if `Stack` contains several comma-separated technologies, count them). Override `DescribeDuties()`. Override `ToString()` with the `sealed override` modifier to lock the format `"Developer[{Name}, {Stack}]"` and help JIT optimizations.

4. In `Manager.cs` create `Manager : Employee` with a `TeamSize` property (int, `init`). Override `CalculateBonus()`: 12% of `BaseSalary` plus `TeamSize * 1000`. In the override of `DescribeDuties()` you MUST call `base.DescribeDuties()` and extend its result with team information — to demonstrate preserving parent logic.

5. In `SalesPerson.cs` create `SalesPerson : Employee` with a `DealsClosed` property (int, `init`). Bonus: 8% of `BaseSalary` plus `DealsClosed * 500`. Override both virtual methods.

6. In `LegacyContractor.cs` create `LegacyContractor : Employee`. Deliberately use `new` to hide `FormatReport()`: `public new string FormatReport() => $"{Name}: legacy contractor report";`. Do NOT override `CalculateBonus()` — let the contractor keep the standard 10%. The goal is to demonstrate the `override` vs `new` difference.

7. In `Program.cs` (top-level statements) build a `List<Employee>` with one object of each type using C# 12 collection expressions: `List<Employee> staff = [new Employee{...}, new Developer{...}, ...];`. In a `foreach` loop print `e.CalculateBonus()` and `e.DescribeDuties()`. Then create `Employee legacy = new LegacyContractor{...};` and print `legacy.FormatReport()` and `((LegacyContractor)legacy).FormatReport()` to visibly show two different versions. Run `dotnet run --project src/PolymorphLabs.App` and save the output.

8. In the test project write xUnit tests: `CalculateBonus_ReturnsExpected_ForEachType`, `PolymorphicDispatch_ThroughBaseReference`, `NewHidesButDoesNotOverride_ThroughBaseReference`, `SealedOverride_PreventsFurtherOverride` (the last one can be a comment-based explanation or an `Assert.Throws` if you can provoke it). Run `dotnet test` and make it green.

#### Requirements

The solution must compile without warnings (in particular, the CS0108 method-hiding warning must either be resolved with an explicit `new` or explained). All public methods of the base class that are intended to be overridden must be marked `virtual`; all overrides must use `override` with the same signature and contract (Liskov substitution: postconditions must not weaken). Using `new` is allowed only for `LegacyContractor.FormatReport()` and must be explicitly documented with a comment explaining why hiding was chosen over overriding.

`sealed override` must be applied to `Developer.ToString()` with a comment about locking the format and JIT optimization. `Manager.DescribeDuties()` must call `base.DescribeDuties()`. The code should use C# 12 features: collection expressions for list initialization, `init` properties, and pattern matching in calculations where reasonable. The project must build with `dotnet build` without errors and tests must pass with `dotnet test`. File and class names must exactly match the steps. The output of `Program.cs` must show that correct polymorphic versions of `CalculateBonus()` and `DescribeDuties()` are invoked through an `Employee` reference, while `FormatReport()` through a base reference calls the parent version and through a cast calls the hidden one.

#### Pitfalls

The central subtlety the lesson unpacks: `override` selects the version by the actual runtime type of the object (dynamic dispatch), whereas `new` selects it by the compile-time type of the reference (static binding). Students routinely confuse the two and are “surprised” that `Employee legacy = new LegacyContractor()` calls the base `FormatReport()`. Explain in comments that `FormatReport()` is not virtual, so there is no dispatch slot at all; `new` only suppresses the warning, it does not create a polymorphic slot. A frequent mistake is forgetting `virtual` on the base method and getting a compile error on `override`. Another frequent mistake is using `new` where `override` is needed, producing a silent bug: polymorphism is broken, tests are green, but production calls the wrong version. Therefore `override` is almost always the right choice; `new` is justified only when you do not control the base class.

`sealed override` locks behavior: attempting to override `ToString()` in a class derived from `Developer` yields a compile error. This is both an invariant safeguard (the log format) and a JIT devirtualization signal. Note that `sealed` on a method only makes sense together with `override`; you never write `sealed` on a regular method. Also remember `base.Method()`: if the parent method did important work (logging, validation), a forgotten `base.` loses it. In `Manager.DescribeDuties()` you must call `base.DescribeDuties()` to preserve the parent string and extend it. Finally, watch the Liskov contract: an overridden `CalculateBonus()` must not return negative values or break invariants of the base method.

#### Acceptance criteria

- [ ] The `PolymorphLabs.sln` solution builds with `dotnet build` without errors or warnings.
- [ ] Target framework is `net8.0`, language is C# 12 (`LangVersion` latest, explicit or implicit).
- [ ] Base class `Employee` contains `virtual CalculateBonus()` and `virtual DescribeDuties()`.
- [ ] `Employee.FormatReport()` is NON-virtual (to demonstrate `new`).
- [ ] `Developer` overrides `CalculateBonus()` and `DescribeDuties()` with `override`.
- [ ] `Developer.ToString()` is marked `sealed override` with a comment about locking the format.
- [ ] `Manager.CalculateBonus()` accounts for `TeamSize`.
- [ ] `Manager.DescribeDuties()` calls `base.DescribeDuties()`.
- [ ] `SalesPerson` overrides both virtual methods.
- [ ] `LegacyContractor` uses `new` for `FormatReport()` with an explanatory comment.
- [ ] `Program.cs` uses collection expressions (`[...]`) to initialize `List<Employee>`.
- [ ] The `dotnet run` output demonstrates polymorphic dispatch (different bonus amounts).
- [ ] The output shows two different `FormatReport()` versions depending on the reference type.
- [ ] All xUnit tests pass (`dotnet test` is green).
- [ ] Test `PolymorphicDispatch_ThroughBaseReference` explicitly checks invocation through `Employee`.
- [ ] RU+EN comments explain the `override` vs `new` choice for every case.

#### Hints (no direct answer)

- Recall the lesson's job-description analogy: whoever “sees” the object as an `Employee` sees the generic description, but the specific one executes — only if the method is `virtual`/`override`.
- To verify that `new` does not create polymorphism, hold one object through two different reference types and compare the output.
- To count technologies in a developer's `Stack`, use `Stack.Split(',').Length` and pattern matching.
- Remember that `sealed override` forbids a descendant from writing `override` again — you can check this with a compile attempt.
- If the compiler rejects `override`, check whether the base method is marked `virtual`.
- If you see warning CS0108, you wrote a method with the same name without `new` and without `override` — decide consciously what you meant.

#### Reference solution walk-through

```csharp
// C# 12 / .NET 8 — PolymorphLabs: virtual / override / new / sealed override
// File: src/PolymorphLabs.App/Employee.cs
namespace PolymorphLabs;

// Base class: virtual methods allow overriding.
public class Employee
{
    public string Name { get; init; } = string.Empty;
    public decimal BaseSalary { get; init; }

    // virtual: allow polymorphic replacement in derived classes.
    public virtual decimal CalculateBonus() => BaseSalary * 0.10m;

    public virtual string DescribeDuties() => $"{Name}: general duties";

    // Non-virtual: used to demonstrate new (hiding).
    public string FormatReport() => $"{Name}: standard report";
}

// File: Developer.cs
public class Developer : Employee
{
    public string Stack { get; init; } = "C#";

    public override decimal CalculateBonus()
    {
        // 15% + 2000 per technology. Preserve Liskov: bonus >= 0.
        var techCount = Stack.Split(',', StringSplitOptions.TrimEntries).Length;
        return BaseSalary * 0.15m + 2000m * techCount;
    }

    public override string DescribeDuties() => $"{Name}: writes code in {Stack}";

    // sealed override: lock the format and help JIT optimizations.
    public sealed override string ToString() => $"Developer[{Name}, {Stack}]";
}

// File: Manager.cs
public class Manager : Employee
{
    public int TeamSize { get; init; }

    public override decimal CalculateBonus() => BaseSalary * 0.12m + TeamSize * 1000m;

    public override string DescribeDuties()
    {
        // base.DescribeDuties() preserves parent logic and extends it.
        return $"{base.DescribeDuties()} + manages team of {TeamSize}";
    }
}

// File: SalesPerson.cs
public class SalesPerson : Employee
{
    public int DealsClosed { get; init; }

    public override decimal CalculateBonus() => BaseSalary * 0.08m + DealsClosed * 500m;
    public override string DescribeDuties() => $"{Name}: closes deals ({DealsClosed})";
}

// File: LegacyContractor.cs
public class LegacyContractor : Employee
{
    // new: HIDE, not override. Through an Employee reference the base version runs.
    // Intentional: legacy integration needs old behavior through the base type.
    public new string FormatReport() => $"{Name}: legacy contractor report";
}

// File: Program.cs (top-level statements)
using PolymorphLabs;

List<Employee> staff =
[
    new Employee   { Name = "Ivan",  BaseSalary = 100_000 },
    new Developer  { Name = "Anna",  BaseSalary = 150_000, Stack = "C#, F#, TypeScript" },
    new Manager    { Name = "Oleg",  BaseSalary = 140_000, TeamSize = 6 },
    new SalesPerson{ Name = "Maria", BaseSalary = 110_000, DealsClosed = 12 },
    new LegacyContractor { Name = "Petr", BaseSalary = 90_000 }
];

// Polymorphic call: version chosen by actual runtime type.
foreach (var e in staff)
    Console.WriteLine($"{e.DescribeDuties()} → bonus {e.CalculateBonus():N0}");

// override vs new — the key difference through the same reference.
Employee legacy = new LegacyContractor { Name = "Petr", BaseSalary = 90_000 };
Console.WriteLine(legacy.FormatReport());                      // standard report
Console.WriteLine(((LegacyContractor)legacy).FormatReport());  // legacy contractor report
```

Line-by-line walk-through. The base `Employee.CalculateBonus()` is marked `virtual` — a necessary condition for any `override` in descendants; without `virtual` the compiler rejects `override` (a common mistake from the lesson). `Developer.CalculateBonus()` uses `override`, so through an `Employee` reference the developer version runs — this is the dynamic dispatch that polymorphism exists for. Counting technologies with `Split` and `StringSplitOptions.TrimEntries` is modern, idiomatic C#. `Developer.ToString()` is `sealed override`: `override` replaces the virtual method inherited from `object`, while `sealed` forbids further overriding and signals the JIT to devirtualize. In `Manager.DescribeDuties()` the mandatory `base.DescribeDuties()` call demonstrates the lesson's best practice: preserve parent logic and extend it without losing invariants. `SalesPerson` shows another independent polymorphic variant. `LegacyContractor.FormatReport()` uses `new`: this is hiding, not overriding — the base `FormatReport()` is not virtual, so there is no polymorphic slot; `new` only suppresses the CS0108 warning. The final demonstration in `Program.cs` proves the main point: through `Employee legacy` the base `FormatReport()` runs, while through a cast the hidden one runs; meanwhile `CalculateBonus()` and `DescribeDuties()` dispatch correctly because they are `virtual`/`override`. The collection expression `[...]` is C# 12. Together this demonstrates all four modifiers and the difference between polymorphism and hiding that the lesson requires.

#### Going deeper (bonus)

1. Add an `Intern : Employee` class that overrides `CalculateBonus()` to return 0 and overrides `ToString()` with `sealed override`. Verify that creating a `SeniorIntern : Intern` with `override ToString()` does not compile.
2. Implement an extension method `decimal TotalBonus(this IEnumerable<Employee> employees)` using LINQ `Sum`, and write a test that checks the sum over a mixed list.
3. Add a virtual property `virtual string Department => "General"` and override it in each class; investigate whether virtual properties behave the same as virtual methods.
4. Compare the performance of `virtual` and `sealed override` methods using `BenchmarkDotNet`: measure the call-time difference in a 10-million-iteration loop and explain the result through devirtualization.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Решение собрано в `PolymorphLabs.sln`, `dotnet build` без ошибок и предупреждений.
- [ ] (RU) Все четыре модификатора `virtual`/`override`/`new`/`sealed override` применены и задокументированы.
- [ ] (RU) Тесты xUnit зелёные, включая проверку диспетчеризации через базовую ссылку.
- [ ] (RU) Вывод `dotnet run` сохранён и приложен к сдаче.
- [ ] (EN) Solution built in `PolymorphLabs.sln`, `dotnet build` is clean of errors and warnings.
- [ ] (EN) All four modifiers `virtual`/`override`/`new`/`sealed override` are applied and documented.
- [ ] (EN) xUnit tests are green, including dispatch verification through a base reference.
- [ ] (EN) `dotnet run` output is saved and attached to the submission.

#### Ресурсы / Resources
- [Microsoft Learn — virtual (C# Reference)](https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/virtual)
- [Microsoft Learn — override (C# Reference)](https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/override)
- [Microsoft Learn — new Modifier](https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/new-modifier)
- [Microsoft Learn — sealed (C# Reference)](https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/sealed)
- [Microsoft Learn — Polymorphism](https://learn.microsoft.com/dotnet/csharp/fundamentals/object-oriented/polymorphism)
- [Microsoft Learn — C# 12 collection expressions](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/collection-expressions)
