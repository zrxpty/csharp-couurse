---
[← К уроку M05-L01](lesson-M05-L01-inheritance-base.md) | [⬆ К модулю M05](../README.md) | [Следующее ДЗ →](homework-M05-L02-virtual-override.md)
---

### Домашнее задание M05-L01: Наследование, base, конструкторы базового класса / Homework M05-L01: Inheritance, base, base constructors

**Урок / Lesson:** M05-L01
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) Научиться строить иерархию классов с одиночным наследованием, явно вызывать конструктор базового класса через `base(...)`, использовать `base.MethodName()`, работать с модификатором `protected`, демонстрировать порядок вызова конструкторов «от базы к производному» и закрепить различие между `override` и `new`, а также понимание того, что `object` — единый корень иерархии типов .NET. (EN) Learn to build a single-inheritance class hierarchy, explicitly invoke the base constructor through `base(...)`, use `base.MethodName()`, work with the `protected` modifier, demonstrate the "base-to-derived" constructor call order, and reinforce the distinction between `override` and `new` together with the understanding that `object` is the single root of the .NET type hierarchy.

#### Связь с уроком / Connection to the lesson
(RU) Урок вводит наследование как механизм переиспользования кода и моделирования отношения «is-a», объясняет одиночное наследование и корень `object`, показывает ключевое слово `base` в двух ролях (вызов метода и вызов конструктора) и подчёркивает строгий порядок инициализации от базового класса к производному. Данное задание заставляет применить каждую из этих идей на практике: вы построите иерархию из трёх классов, где базовый конструктор требует аргументы, и убедитесь, что без явного `base(...)` код просто не скомпилируется.
(EN) The lesson introduces inheritance as a code-reuse mechanism and a way to model the "is-a" relationship, explains single inheritance and the `object` root, shows the `base` keyword in two roles (method call and constructor call), and stresses the strict base-to-derived initialization order. This homework forces you to apply every one of those ideas in practice: you will build a three-class hierarchy where the base constructor requires arguments, and you will see firsthand that without an explicit `base(...)` the code simply will not compile.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Представьте, что вы присоединились к команде, которая разрабатывает внутреннюю систему учёта сотрудников небольшой IT-компании. Сейчас в коде царит хаос: каждый тип сотрудника описан отдельным классом, которые почти не связаны между собой, и одни и те же поля (`Name`, `HireDate`, `BaseSalary`) копируются из класса в класс с бесконечными правками. Ведущий архитектор просит вас навести порядок: выделить общий базовый класс `Employee`, который инкапсулирует общие данные и поведение, а специфичные роли (`Manager`, `Developer`) сделать производными от него. Это классический сценарий, в котором наследование буквально напрашивается: менеджер **есть** сотрудник, разработчик **есть** сотрудник — это отношение «is-a», а не «has-a».

Заодно нужно исправить две давние проблемы. Во-первых, валидация входных данных: сегодня любой код может создать сотрудника с пустым именем, отрицательной зарплатой или датой найма из будущего, и такие объекты потом «взрываются» в продакшене. В уроке вы видели, как базовый конструктор `Animal` выбрасывал `ArgumentException` для пустого имени — вы перенесёте эту идею на `Employee` и убедитесь, что производные классы не смогут обойти проверку, потому что они обязаны вызвать `base(...)` и пройти через неё. Во-вторых, нужно наглядно показать порядок вызова конструкторов: при создании `Manager` сначала должен отработать `Employee.ctor`, и только потом `Manager.ctor`. Это та самая аналогия из урока про «второй этаж нельзя строить в воздухе».

В результате вы получите маленькую, но честную иерархию, которая демонстрирует одиночное наследование, явный вызов базового конструктора, доступ к `protected`-членам из производных классов, переопределение виртуального метода через `override`, использование `base.MethodName()` для дополнения, а не замены, поведения базы, а также апкаст к `Employee` и далее к `object` — корню всей иерархии типов .NET.

#### Что нужно сделать (пошагово)
1. Создайте новый консольный проект .NET 8 командой `dotnet new console -n HrInheritance` в подходящей папке. Откройте получившийся каталог, удалите шаблонный `Program.cs` и создайте вместо него файл `Program.cs` с одной строкой top-level statement: `HrInheritance.Demo.Run();`. Весь код иерархии поместите в тот же файл ниже — для учебного задания этого достаточно, хотя в реальном проекте вы бы разнесли классы по отдельным файлам.
2. Опишите базовый класс `public class Employee`. У него должны быть три свойства, доступные только для чтения: `public string Name { get; }`, `public DateTime HireDate { get; }` и `protected decimal BaseSalary { get; }`. Обратите внимание на модификатор `protected` у зарплаты — он делает член доступным производным классам, но недоступным внешнему коду; это ключевой инструмент инкапсуляции, упомянутый в уроке наряду с `public` и `private`.
3. Реализуйте конструктор `public Employee(string name, DateTime hireDate, decimal baseSalary)`, который выполняет три проверки и выбрасывает исключения: пустое или whitespace-имя → `ArgumentException` с `nameof(name)`; отрицательная зарплата → `ArgumentOutOfRangeException` с `nameof(baseSalary)`; дата найма в будущем (`hireDate > DateTime.Today`) → `ArgumentOutOfRangeException` с `nameof(hireDate)`. После проверок присвойте значения свойствам. Добавьте в конец конструктора `Console.WriteLine($"  Employee.ctor: {Name}");` — это поможет увидеть порядок вызова.
4. Добавьте вычисляемое свойство `public int YearsOfService`, которое честно считает полные годы стажа с учётом того, прошёл ли день годовщины в текущем году. Добавьте `public virtual decimal CalculatePay() => BaseSalary + YearsOfService * 1000m;` — виртуальный метод, который производные классы будут переопределять. Переопределите также `ToString()`, возвращая строку вида `"Имя (TypeName), найм: yyyy-MM-dd"`.
5. Опишите производный класс `public sealed class Manager : Employee`. Модификатор `sealed` здесь намеренный: вы фиксируете, что от менеджера нельзя унаследовать дальше, и защищаете инварианты — это лучшая практика из урока. Добавьте свойство `public int TeamSize { get; }`. Конструктор `public Manager(string name, DateTime hireDate, decimal baseSalary, int teamSize) : base(name, hireDate, baseSalary)` обязан явно вызвать `base(...)`, потому что у `Employee` нет конструктора без параметров. Проверьте `teamSize >= 0` и выбросьте `ArgumentOutOfRangeException`, если отрицательно. В конце конструктора выведите `Console.WriteLine($"  Manager.ctor: team {TeamSize}");`.
6. Переопределите `CalculatePay()` у `Manager` так, чтобы он возвращал `base.CalculatePay() + TeamSize * 500m`. Здесь `base.CalculatePay()` — это второй способ применения `base`, отличный от вызова конструктора: вы не заменяете логику базы полностью, а **дополняете** её. Это правильный паттерн, который сохраняет единообразие поведения и не дублирует формулу надбавки за стаж.
7. Опишите третий класс `public class Developer : Employee` (без `sealed`, чтобы оставить точку расширения для будущего урока про виртуальные методы). Добавьте `public IReadOnlyList<string> Languages { get; }`. Конструктор `public Developer(string name, DateTime hireDate, decimal baseSalary, params string[] languages) : base(...)` должен проверять, что передан хотя бы один язык, иначе выбрасывать `ArgumentException`, и оборачивать массив в `Array.AsReadOnly(languages)` для неизменяемости. Переопределите `CalculatePay()`, добавляя надбавку `Languages.Count * 1500m` поверх `base.CalculatePay()`.
8. В классе `Demo` создайте метод `public static void Run()`. Создайте `Manager` с реальными данными (например, Анна, дата найма 2015-03-01, зарплата 80000, команда 5 человек), сохраните его в переменную типа `Employee` (это апкаст — он работает именно благодаря наследованию). Выведите объект через `Console.WriteLine(emp)` (сработает ваш `ToString()`), затем вызовите `emp.CalculatePay()` и выведите результат в валютном формате `:C`. Проделайте то же для `Developer` с двумя языками.
9. Дополнительно продемонстрируйте корень иерархии: присвойте объект менеджера переменной типа `object asObject = emp;` и выведите `asObject.GetType().Name` — это покажет, что реальный тип всё ещё `Manager`, хотя статический тип переменной `object`. Это прямой пример из урока про то, что `object` — единый корень.
10. Соберите и запустите проект: `dotnet build`, затем `dotnet run`. Внимательно прочитайте вывод в консоли: вы должны увидеть, что для каждого объекта сначала печатается строка `Employee.ctor: ...` и только потом строка конкретного производного конструктора. Это эмпирическое подтверждение порядка «база → производный».
11. Проведите «негативный» эксперимент в отдельной ветке или закомментированном блоке: временно уберите `: base(name, hireDate, baseSalary)` из конструктора `Manager` и попробуйте собрать проект. Вы должны получить ошибку компиляции **CS1729** — ровно ту, про которую предупреждает урок. Зафиксируйте текст ошибки в комментарии, затем верните `base(...)` обратно. Точно так же временно попробуйте описать класс `class Multi : Employee, IDisposable, IComparable` как `class Multi : Employee, SomeOtherClass` (попытка множественного наследования классов) и убедитесь, что компилятор это запрещает.
12. Убедитесь, что нигде в конструкторах вы не вызываете виртуальные методы (урок явно предупреждает об этой ошибке: в момент работы базового конструктора производный класс ещё не инициализирован, и переопределение сработает на «полуготовом» объекте). Если у вас возникнет соблазн вызвать `CalculatePay()` из `Employee.ctor` для логирования — не делайте этого; вместо этого логируйте только уже готовые поля.

#### Требования к решению
- Целевая платформа: C# 12 и .NET 8. Используйте top-level statements для точки входа, разрешены collection expressions и `params`-массивы. Код должен собираться без предупреждений уровня `error` при стандартных настройках `dotnet build`.
- Должна присутствовать иерархия ровно из трёх классов: `Employee` (база), `Manager` (производный, `sealed`), `Developer` (производный). Отношение между ними — «is-a»: менеджер является сотрудником, разработчик является сотрудником. Не путайте с композицией: у сотрудника нет поля-сотрудника.
- Базовый конструктор `Employee` обязан принимать три параметра и не иметь перегрузки без параметров. Это сделано намеренно, чтобы `base(...)` стало обязательным и вы прочувствовали механизм, а не полагались на автоВызов конструктора по умолчанию.
- Все проверки аргументов выполняются в конструкторах и выбрасывают стандартные типы исключений (`ArgumentException`, `ArgumentOutOfRangeException`) с осмысленным сообщением и `nameof(...)`. Производные конструкторы проверяют только свои специфичные параметры (например, `teamSize`), общие проверки делегируются базе.
- Модификаторы доступа расставлены осознанно: `Name` и `HireDate` — `public` get-only; `BaseSalary` — `protected` get-only (доступен производным, но не внешнему миру); свойства производных классов — `public` get-only. Никаких публичных сеттеров, нарушающих неизменяемость.
- Виртуальный метод `CalculatePay()` в базе переопределён в обоих производных классах через `override` (не через `new`). В переопределённых версиях обязательно вызывается `base.CalculatePay()` для reuse базовой формулы. `ToString()` переопределён в базе.
- В классе `Demo` продемонстрированы: апкаст `Manager` → `Employee`, апкаст `Employee` → `object`, вызов `GetType().Name` для доказательства реального типа, вывод порядка конструкторов в консоль.

#### Тонкости и подводные камни
- **Забыли `: base(...)`** — самая частая ошибка. Если у базового класса нет конструктора без параметров, компилятор выдаёт **CS1729** «'Employee' does not contain a constructor that takes 0 arguments». Лекарство одно: явно написать `: base(name, hireDate, baseSalary)` с нужными аргументами. Урок прямо перечисляет это в «Частых ошибках».
- **`new` вместо `override`** — вторая классическая ловушка. Если вы объявите `public new decimal CalculatePay()` вместо `public override decimal CalculatePay()`, то вызов `emp.CalculatePay()` для переменной типа `Employee` пойдёт в базовую версию, а не в `Manager.CalculatePay()`, потому что при `new` диспетчеризация зависит от статического типа переменной. Это порождает трудноуловимые баги. Пометьте метод базы `virtual` и используйте `override`.
- **Виртуальный вызов из конструктора** — опасная практика, упомянутая в уроке. В момент работы `Employee.ctor` поля `Manager` (например, `TeamSize`) ещё не инициализированы. Если бы `CalculatePay()` вызывался из базового конструктора, переопределённая версия `Manager.CalculatePay()` прочитала бы `TeamSize == 0`. Избегайте вызовов виртуальных членов в конструкторах.
- **Модификатор `protected`** — не делайте всё `public`. Зарплата — внутренние данные; производные классы legitimately читают её для расчёта выплаты, но внешний код не должен иметь к ней доступа. `protected` — это именно «доступ по наследству».
- **Множественное наследование классов запрещено**. Конструкция `class A : B, C` где `B` и `C` — классы, не компилируется. Для нескольких контрактов используйте интерфейсы (в этом задании они не нужны, но держите в уме).
- **`object` как корень**. Любой `Employee` автоматически можно трактовать как `object`, и обратно через `GetType()` узнать реальный тип. Не путайте статический тип переменной (`Employee` или `object`) с реальным типом объекта (`Manager`).
- **`sealed` — защита инвариантов**. Помечая `Manager` как `sealed`, вы запрещаете дальнейшее наследование и гарантируете, что никто не сломает логику `CalculatePay()`. Урок рекомендует запечатывать то, что не предназначено для расширения.

#### Критерии приёмки
- [ ] Проект `HrInheritance` собирается командой `dotnet build` без ошибок и без предупреждений уровня error.
- [ ] Присутствуют ровно три класса: `Employee`, `Manager`, `Developer`, связанных наследованием.
- [ ] `Employee` имеет конструктор с тремя параметрами и не имеет конструктора без параметров.
- [ ] Конструкторы `Manager` и `Developer` явно вызывают `base(name, hireDate, baseSalary)`.
- [ ] Модификатор `protected` применён к `BaseSalary`; `Name` и `HireDate` — `public` get-only.
- [ ] В конструкторе `Employee` реализованы три проверки аргументов с выбросом `ArgumentException`/`ArgumentOutOfRangeException` и `nameof(...)`.
- [ ] Метод `CalculatePay()` в `Employee` помечен `virtual`; в `Manager` и `Developer` переопределён через `override` (не `new`).
- [ ] В переопределённых `CalculatePay()` используется `base.CalculatePay()` для дополнения, а не замены, базовой логики.
- [ ] `Manager` помечен `sealed`; `Developer` — нет.
- [ ] В классе `Demo` демонстрируется апкаст `Manager` → `Employee` и далее → `object`, а также `GetType().Name`.
- [ ] Вывод консоли показывает, что `Employee.ctor` печатается раньше конструктора производного класса для каждого объекта.
- [ ] Выполнен негативный эксперимент: закомментирован `base(...)`, зафиксирован текст ошибки CS1729 в комментарии, затем `base(...)` возвращён.
- [ ] В конструкторах нет вызовов виртуальных методов.
- [ ] Код использует возможности C# 12 / .NET 8 (top-level statements, get-only auto-properties, можно `params`).
- [ ] `dotnet run` выводит осмысленный результат: данные сотрудников и их выплаты в формате валюты.

#### Подсказки (без прямого ответа)
- Подсчёт «полных лет стажа» удобно сделать через разность годов с коррекцией на то, наступил ли день годовщины в текущем году. Подумайте, что даёт `DateTime.Today.DayOfYear` по сравнению с `HireDate.DayOfYear`.
- Чтобы `Languages` был неизменяемым, не отдавайте наружу исходный массив — оберните его. В .NET есть готовый метод-обёртка для массивов.
- Проверка «хотя бы один язык» элегантно записывается тернарным оператором прямо в теле конструктора после `base(...)`, но можно и обычным `if`.
- Для негативного эксперимента с CS1729 не обязательно реально ломать сборку надолго — закомментируйте `base(...)`, соберите, скопируйте текст ошибки в комментарий, верните `base(...)`. Главное — увидеть ошибку своими глазами.
- Вспомните аналогию урока про «этажи здания»: второй этаж (производный класс) нельзя строить, пока не построен первый (базовый). Это объясняет, почему `Console.WriteLine` в базовом конструкторе срабатывает раньше.

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8
// Эталонное решение ДЗ M05-L01: наследование, base, конструкторы базы.
// Reference solution for HW M05-L01: inheritance, base, base constructors.
using System;
using System.Collections.ObjectModel;  // для ReadOnlyCollection / for ReadOnlyCollection

// Базовый класс — корень нашей маленькой иерархии / Base class — root of our small hierarchy
public class Employee
{
    public string Name { get; }              // public get-only / публичный только для чтения
    public DateTime HireDate { get; }
    protected decimal BaseSalary { get; }    // protected — доступно производным / available to derived

    // Конструктор базы требует три аргумента и валидирует их / Base ctor takes 3 args and validates them
    public Employee(string name, DateTime hireDate, decimal baseSalary)
    {
        if (string.IsNullOrWhiteSpace(name))
            throw new ArgumentException("Имя не может быть пустым / Name cannot be empty", nameof(name));
        if (baseSalary < 0)
            throw new ArgumentOutOfRangeException(nameof(baseSalary),
                "Зарплата не может быть отрицательной / Salary cannot be negative");
        if (hireDate > DateTime.Today)
            throw new ArgumentOutOfRangeException(nameof(hireDate),
                "Дата найма в будущем / Hire date is in the future");

        Name = name;
        HireDate = hireDate;
        BaseSalary = baseSalary;
        // Логируем — это НЕ виртуальный вызов, безопасно / Logging — NOT a virtual call, safe
        Console.WriteLine($"  Employee.ctor: {Name}");
    }

    // Полные годы стажа с учётом годовщины / Full years of service accounting for anniversary
    public int YearsOfService =>
        DateTime.Today.Year - HireDate.Year -
        (DateTime.Today.DayOfYear < HireDate.DayOfYear ? 1 : 0);

    // Виртуальный метод — производные переопределяют / Virtual — derived classes override
    public virtual decimal CalculatePay() => BaseSalary + YearsOfService * 1000m;

    public override string ToString() => $"{Name} ({GetType().Name}), найм/Hire {HireDate:yyyy-MM-dd}";
}

// sealed — нельзя унаследовать дальше, защищаем инварианты / sealed — no further inheritance
public sealed class Manager : Employee
{
    public int TeamSize { get; }

    // Явный base(...) — ОБЯЗАТЕЛЕН, у Employee нет ctor без параметров / Explicit base(...) — MANDATORY
    public Manager(string name, DateTime hireDate, decimal baseSalary, int teamSize)
        : base(name, hireDate, baseSalary)
    {
        if (teamSize < 0)
            throw new ArgumentOutOfRangeException(nameof(teamSize),
                "Размер команды отрицательный / Team size is negative");

        TeamSize = teamSize;
        Console.WriteLine($"  Manager.ctor: team {TeamSize}");
    }

    // override + base.CalculatePay() — дополнение, а не замена / override + base call — extend, don't replace
    public override decimal CalculatePay() => base.CalculatePay() + TeamSize * 500m;
}

// Не sealed — оставляем точку расширения / Not sealed — leave an extension point
public class Developer : Employee
{
    public IReadOnlyList<string> Languages { get; }

    public Developer(string name, DateTime hireDate, decimal baseSalary, params string[] languages)
        : base(name, hireDate, baseSalary)
    {
        // params может прийти пустым — проверяем / params may arrive empty — validate
        if (languages is null || languages.Length == 0)
            throw new ArgumentException(
                "Нужен хотя бы один язык / At least one language required", nameof(languages));

        Languages = Array.AsReadOnly(languages);  // неизменяемая обёртка / immutable wrapper
        Console.WriteLine($"  Developer.ctor: langs {string.Join(", ", Languages)}");
    }

    public override decimal CalculatePay() => base.CalculatePay() + Languages.Count * 1500m;
}

internal static class Demo
{
    public static void Run()
    {
        // Апкаст Manager -> Employee (работает благодаря наследованию) / Upcast Manager -> Employee
        Employee emp = new Manager("Анна/Anna", new DateTime(2015, 3, 1), 80000m, 5);
        Console.WriteLine(emp);                         // сработает наш ToString / our ToString runs
        Console.WriteLine($"  Выплата/Pay: {emp.CalculatePay():C}");

        var dev = new Developer("Иван/Ivan", new DateTime(2019, 9, 15), 70000m, "C#", "F#");
        Console.WriteLine(dev);
        Console.WriteLine($"  Выплата/Pay: {dev.CalculatePay():C}");

        // object — корень иерархии, апкаст работает дальше / object is the root, upcast continues
        object asObject = emp;
        Console.WriteLine($"  asObject.GetType().Name = {asObject.GetType().Name}"); // Manager
    }
}

// Точка входа — top-level statements / Entry point — top-level statements
HrInheritance.Demo.Run();
```

Разбор по строкам и применённым концепциям урока. Класс `Employee` построен точно по образцу `Animal` из урока: get-only свойства, конструктор с валидацией и выбросом `ArgumentException`, явное логирование в конце конструктора — это позволяет увидеть порядок вызова «база → производный». Модификатор `protected` у `BaseSalary` — это именно тот инструмент, про который урок говорит: «`private` члены существуют физически, но напрямую не доступны; `public` и `protected` — доступны». Мы выбрали `protected`, потому что производным классам нужно читать зарплату для расчёта выплаты, но внешнему коду — нет. Класс `Manager` помечен `sealed` в соответствии с лучшей практикой урока: «запечатывайте классы и методы `sealed`, чтобы защитить инварианты». Конструктор `Manager` обязан вызвать `: base(name, hireDate, baseSalary)`, потому что у `Employee` нет конструктора без параметров — если убрать `base(...)`, компилятор выдаст именно **CS1729**, о котором предупреждает урок. Внутри `Manager.CalculatePay()` используется `base.CalculatePay()` — это второй режим ключевого слова `base`, отличающийся от вызова конструктора: мы не переписываем формулу надбавки за стаж, а переиспользуем её и добавляем свою часть. Это согласуется с рекомендацией урока «всегда передавайте обязательные данные базовому классу через `base(...)`, а не пытайтесь обойти инициализацию» и в более широком смысле с идеей, что `base` — «ручка к родителю». Класс `Developer` оставлен без `sealed`, чтобы в следующем уроке (`M05-L02` про `virtual`/`override`) можно было от него унаследовать `SeniorDeveloper` и продолжить эксперименты. В методе `Run` демонстрируется апкаст: переменная `Employee emp` хранит объект `Manager`, и `emp.CalculatePay()` благодаря `override` (а не `new`!) вызывает именно `Manager.CalculatePay()`. Дальнейший апкаст к `object` и `GetType().Name` иллюстрирует, что `object` — единый корень иерархии, а реальный тип сохраняется. Порядок строк в выводе (`Employee.ctor` раньше `Manager.ctor`) — эмпирическое подтверждение правила из урока: «сначала полностью отрабатывает конструктор базового, и только потом — конструктор производного». Наконец, обратите внимание, что из конструкторов мы не вызываем `CalculatePay()` — это соблюдение запрета урока на виртуальные вызовы в конструкторах: в момент работы `Employee.ctor` поля `Manager` ещё не инициализированы.

#### Задания на углубление (бонус)
1. Добавьте четвёртый класс `Intern : Employee` с полем `Mentor` (строка) и переопределённым `CalculatePay()`, который платит фиксированную стипендию `BaseSalary * 0.5m` без надбавок за стаж. Подумайте, как при этом вызвать `base.CalculatePay()` или, наоборот, намеренно обойти его, и обоснуйте выбор.
2. Реализуйте перегрузку конструктора `Employee`, принимающую только `name` и `hireDate`, и ставящую `baseSalary` по умолчанию через `: this(name, hireDate, 50000m)`. Исследуйте, как работает делегирование конструкторов (`this(...)`) в сочетании с `base(...)`.
3. Временно перепишите `Manager.CalculatePay()` через `new` вместо `override` и вызовите его через переменную типа `Employee`. Зафиксируйте в комментарии, какой метод отработал и почему, — это наглядно покажет ловушку, описанную в уроке.
4. Реализуйте интерфейс `IComparable<Employee>`, сравнивающий сотрудников по `YearsOfService`, и попробуйте отсортировать `Employee[]`. Заодно убедитесь, что интерфейсы можно добавлять к классам без нарушения одиночного наследования.

---

## Statement in English / Постановка на английском

#### Context & motivation
Imagine you have just joined a team building an internal HR system for a small IT company. Right now the codebase is a mess: every employee role lives in its own standalone class, and the same fields (`Name`, `HireDate`, `BaseSalary`) are copy-pasted from class to class with endless manual edits. The lead architect asks you to clean this up: extract a shared base class `Employee` that encapsulates the common data and behavior, and turn the specific roles (`Manager`, `Developer`) into derived classes of it. This is the textbook scenario where inheritance literally suggests itself: a manager **is an** employee, a developer **is an** employee — that is an "is-a" relationship, not a "has-a" relationship.

Along the way you must fix two long-standing problems. First, input validation: today any code can construct an employee with an empty name, a negative salary, or a hire date from the future, and such objects later blow up in production. In the lesson you saw how the base constructor of `Animal` threw `ArgumentException` for an empty name — you will carry that idea into `Employee` and make sure the derived classes cannot bypass the checks, because they are forced to call `base(...)` and therefore to flow through them. Second, you must visibly demonstrate the constructor call order: when a `Manager` is created, `Employee.ctor` must run first, and only then `Manager.ctor`. That is the very analogy from the lesson about the second floor of a building not being buildable in mid-air.

The result is a small but honest hierarchy that demonstrates single inheritance, the explicit call of the base constructor, access to `protected` members from derived classes, the overriding of a virtual method with `override`, the use of `base.MethodName()` to extend rather than to replace the base behavior, and the upcast to `Employee` and further to `object` — the single root of the entire .NET type hierarchy.

#### What to do step by step
1. Create a fresh .NET 8 console project with `dotnet new console -n HrInheritance` in a suitable folder. Open the resulting directory, delete the templated `Program.cs`, and replace it with a `Program.cs` containing a single top-level statement: `HrInheritance.Demo.Run();`. Put the whole hierarchy code in the same file just below — for a homework exercise this is fine, although in a real project you would split the classes into separate files.
2. Declare the base class `public class Employee`. It must expose three read-only properties: `public string Name { get; }`, `public DateTime HireDate { get; }`, and `protected decimal BaseSalary { get; }`. Pay attention to the `protected` modifier on the salary — it makes the member reachable from derived classes but unreachable from outside code; this is the key encapsulation tool the lesson mentions alongside `public` and `private`.
3. Implement the constructor `public Employee(string name, DateTime hireDate, decimal baseSalary)` that performs three checks and throws exceptions: an empty or whitespace name → `ArgumentException` with `nameof(name)`; a negative salary → `ArgumentOutOfRangeException` with `nameof(baseSalary)`; a hire date in the future (`hireDate > DateTime.Today`) → `ArgumentOutOfRangeException` with `nameof(hireDate)`. After the checks, assign the values to the properties. Append `Console.WriteLine($"  Employee.ctor: {Name}");` at the end — this makes the call order visible.
4. Add a computed property `public int YearsOfService` that honestly counts the full years of tenure, taking into account whether the anniversary day has passed in the current year. Add `public virtual decimal CalculatePay() => BaseSalary + YearsOfService * 1000m;` — a virtual method that derived classes will override. Also override `ToString()`, returning a string like `"Name (TypeName), hire: yyyy-MM-dd"`.
5. Declare the derived class `public sealed class Manager : Employee`. The `sealed` modifier is intentional: you fix that nothing can inherit from a manager and you protect its invariants — this is a best practice straight from the lesson. Add the property `public int TeamSize { get; }`. The constructor `public Manager(string name, DateTime hireDate, decimal baseSalary, int teamSize) : base(name, hireDate, baseSalary)` must explicitly call `base(...)`, because `Employee` has no parameterless constructor. Check `teamSize >= 0` and throw `ArgumentOutOfRangeException` if it is negative. At the end of the constructor print `Console.WriteLine($"  Manager.ctor: team {TeamSize}");`.
6. Override `CalculatePay()` on `Manager` so that it returns `base.CalculatePay() + TeamSize * 500m`. Here `base.CalculatePay()` is the second way of using `base`, distinct from a constructor call: you do not replace the base logic wholesale, you **extend** it. This is the correct pattern that preserves behavioral consistency and does not duplicate the tenure-bonus formula.
7. Declare a third class `public class Developer : Employee` (without `sealed`, to leave an extension point for the next lesson on virtual methods). Add `public IReadOnlyList<string> Languages { get; }`. The constructor `public Developer(string name, DateTime hireDate, decimal baseSalary, params string[] languages) : base(...)` must verify that at least one language was supplied, otherwise throw `ArgumentException`, and wrap the array in `Array.AsReadOnly(languages)` for immutability. Override `CalculatePay()`, adding a bonus of `Languages.Count * 1500m` on top of `base.CalculatePay()`.
8. In the `Demo` class create a method `public static void Run()`. Construct a `Manager` with realistic data (for example, Anna, hire date 2015-03-01, salary 80000, team of 5) and store it in a variable of type `Employee` (this is an upcast — it works precisely because of inheritance). Print the object through `Console.WriteLine(emp)` (your `ToString()` will run), then call `emp.CalculatePay()` and print the result in currency format `:C`. Do the same for a `Developer` with two languages.
9. Additionally demonstrate the root of the hierarchy: assign the manager object to a variable of type `object asObject = emp;` and print `asObject.GetType().Name` — this shows that the real type is still `Manager` even though the static type of the variable is `object`. This is a direct example from the lesson about `object` being the single root.
10. Build and run the project: `dotnet build`, then `dotnet run`. Read the console output carefully: for every object you must see the line `Employee.ctor: ...` printed first and only then the line of the specific derived constructor. This is empirical proof of the "base → derived" order.
11. Run a "negative" experiment in a separate branch or a commented block: temporarily remove `: base(name, hireDate, baseSalary)` from the `Manager` constructor and try to build the project. You should get a compile-time error **CS1729** — exactly the one the lesson warns about. Record the error text in a comment, then restore `base(...)`. In the same vein, temporarily try to declare a class like `class Multi : Employee, SomeOtherClass` (an attempt at multiple class inheritance) and confirm the compiler forbids it.
12. Make sure that nowhere in your constructors do you call virtual methods (the lesson explicitly warns about this mistake: at the moment the base constructor runs, the derived class is not initialized yet, and the override would fire on a half-built object). If you feel tempted to call `CalculatePay()` from `Employee.ctor` for logging — do not; instead, log only the already-assigned fields.

#### Requirements
- Target platform: C# 12 and .NET 8. Use top-level statements for the entry point; collection expressions and `params` arrays are allowed. The code must build without error-level warnings under the standard `dotnet build` settings.
- The hierarchy must contain exactly three classes: `Employee` (base), `Manager` (derived, `sealed`), `Developer` (derived). The relationship between them is "is-a": a manager is an employee, a developer is an employee. Do not confuse this with composition: an employee does not have a field of type employee.
- The base constructor `Employee` must take three parameters and have no parameterless overload. This is intentional, so that `base(...)` becomes mandatory and you actually feel the mechanism instead of relying on an automatic default-constructor call.
- All argument checks are performed in the constructors and throw standard exception types (`ArgumentException`, `ArgumentOutOfRangeException`) with a meaningful message and `nameof(...)`. Derived constructors check only their own specific parameters (for example, `teamSize`); the common checks are delegated to the base.
- Access modifiers are set deliberately: `Name` and `HireDate` are `public` get-only; `BaseSalary` is `protected` get-only (reachable from derived classes, but not from the outside world); properties of derived classes are `public` get-only. No public setters that break immutability.
- The virtual method `CalculatePay()` in the base is overridden in both derived classes via `override` (not via `new`). The overridden versions must call `base.CalculatePay()` to reuse the base formula. `ToString()` is overridden in the base.
- The `Demo` class demonstrates: an upcast `Manager` → `Employee`, an upcast `Employee` → `object`, a call to `GetType().Name` to prove the real type, and a console output showing the constructor order.

#### Pitfalls
- **Forgetting `: base(...)`** — the most frequent mistake. If the base class has no parameterless constructor, the compiler raises **CS1729** "'Employee' does not contain a constructor that takes 0 arguments". The only remedy is to write `: base(name, hireDate, baseSalary)` explicitly with the right arguments. The lesson lists this directly under "Common Mistakes".
- **`new` instead of `override`** — the second classic trap. If you write `public new decimal CalculatePay()` instead of `public override decimal CalculatePay()`, then a call `emp.CalculatePay()` on a variable of type `Employee` will go to the base version, not to `Manager.CalculatePay()`, because with `new` dispatch depends on the static type of the variable. This breeds subtle bugs. Mark the base method `virtual` and use `override`.
- **A virtual call from a constructor** — a dangerous practice the lesson mentions. At the moment `Employee.ctor` runs, the fields of `Manager` (for instance, `TeamSize`) are not yet initialized. If `CalculatePay()` were called from the base constructor, the overridden `Manager.CalculatePay()` would read `TeamSize == 0`. Avoid virtual member calls in constructors.
- **The `protected` modifier** — do not make everything `public`. A salary is internal data; derived classes legitimately read it to compute a payout, but external code must not have access. `protected` is exactly "access by inheritance".
- **Multiple class inheritance is forbidden.** The construct `class A : B, C` where both `B` and `C` are classes does not compile. For multiple contracts, use interfaces (not needed in this homework, but keep it in mind).
- **`object` as the root.** Any `Employee` can automatically be treated as `object`, and conversely `GetType()` reveals the real type. Do not confuse the static type of a variable (`Employee` or `object`) with the real type of the object (`Manager`).
- **`sealed` protects invariants.** By marking `Manager` as `sealed` you forbid further inheritance and guarantee that no one breaks the logic of `CalculatePay()`. The lesson recommends sealing whatever is not meant to be extended.

#### Acceptance criteria
- [ ] The `HrInheritance` project builds with `dotnet build` without errors and without error-level warnings.
- [ ] Exactly three classes are present: `Employee`, `Manager`, `Developer`, connected by inheritance.
- [ ] `Employee` has a constructor with three parameters and no parameterless constructor.
- [ ] The `Manager` and `Developer` constructors explicitly call `base(name, hireDate, baseSalary)`.
- [ ] The `protected` modifier is applied to `BaseSalary`; `Name` and `HireDate` are `public` get-only.
- [ ] The `Employee` constructor implements three argument checks that throw `ArgumentException`/`ArgumentOutOfRangeException` with `nameof(...)`.
- [ ] The `CalculatePay()` method in `Employee` is marked `virtual`; in `Manager` and `Developer` it is overridden via `override` (not `new`).
- [ ] The overridden `CalculatePay()` methods use `base.CalculatePay()` to extend, not to replace, the base logic.
- [ ] `Manager` is marked `sealed`; `Developer` is not.
- [ ] The `Demo` class demonstrates an upcast `Manager` → `Employee` and further → `object`, plus `GetType().Name`.
- [ ] The console output shows `Employee.ctor` printed before the derived constructor for every object.
- [ ] The negative experiment is done: `base(...)` is commented out, the CS1729 error text is recorded in a comment, then `base(...)` is restored.
- [ ] No virtual methods are called from constructors.
- [ ] The code uses C# 12 / .NET 8 features (top-level statements, get-only auto-properties, `params` is fine).
- [ ] `dotnet run` produces a meaningful result: employee data and their payouts in currency format.

#### Hints (no direct answer)
- Counting "full years of tenure" is conveniently done via the difference of years with a correction for whether the anniversary day has arrived in the current year. Think about what `DateTime.Today.DayOfYear` versus `HireDate.DayOfYear` gives you.
- To make `Languages` immutable, do not hand out the original array — wrap it. .NET has a ready-made wrapper method for arrays.
- The "at least one language" check can be written elegantly with a ternary operator right inside the constructor body after `base(...)`, but a plain `if` is fine too.
- For the CS1729 negative experiment you do not have to keep the build broken for long — comment out `base(...)`, build, copy the error text into a comment, restore `base(...)`. The point is to see the error with your own eyes.
- Recall the lesson's "floors of a building" analogy: the second floor (the derived class) cannot be built before the first (the base). That explains why the `Console.WriteLine` in the base constructor fires earlier.

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8
// Reference solution for HW M05-L01: inheritance, base, base constructors.
using System;
using System.Collections.ObjectModel;  // for ReadOnlyCollection

// Base class — root of our small hierarchy
public class Employee
{
    public string Name { get; }              // public get-only
    public DateTime HireDate { get; }
    protected decimal BaseSalary { get; }    // protected — reachable from derived

    // Base ctor takes 3 args and validates them
    public Employee(string name, DateTime hireDate, decimal baseSalary)
    {
        if (string.IsNullOrWhiteSpace(name))
            throw new ArgumentException("Name cannot be empty", nameof(name));
        if (baseSalary < 0)
            throw new ArgumentOutOfRangeException(nameof(baseSalary),
                "Salary cannot be negative");
        if (hireDate > DateTime.Today)
            throw new ArgumentOutOfRangeException(nameof(hireDate),
                "Hire date is in the future");

        Name = name;
        HireDate = hireDate;
        BaseSalary = baseSalary;
        // Logging — NOT a virtual call, safe
        Console.WriteLine($"  Employee.ctor: {Name}");
    }

    // Full years of service accounting for the anniversary
    public int YearsOfService =>
        DateTime.Today.Year - HireDate.Year -
        (DateTime.Today.DayOfYear < HireDate.DayOfYear ? 1 : 0);

    // Virtual — derived classes override
    public virtual decimal CalculatePay() => BaseSalary + YearsOfService * 1000m;

    public override string ToString() => $"{Name} ({GetType().Name}), hire {HireDate:yyyy-MM-dd}";
}

// sealed — no further inheritance, protect invariants
public sealed class Manager : Employee
{
    public int TeamSize { get; }

    // Explicit base(...) — MANDATORY, Employee has no parameterless ctor
    public Manager(string name, DateTime hireDate, decimal baseSalary, int teamSize)
        : base(name, hireDate, baseSalary)
    {
        if (teamSize < 0)
            throw new ArgumentOutOfRangeException(nameof(teamSize),
                "Team size is negative");

        TeamSize = teamSize;
        Console.WriteLine($"  Manager.ctor: team {TeamSize}");
    }

    // override + base.CalculatePay() — extend, don't replace
    public override decimal CalculatePay() => base.CalculatePay() + TeamSize * 500m;
}

// Not sealed — leave an extension point
public class Developer : Employee
{
    public IReadOnlyList<string> Languages { get; }

    public Developer(string name, DateTime hireDate, decimal baseSalary, params string[] languages)
        : base(name, hireDate, baseSalary)
    {
        // params may arrive empty — validate
        if (languages is null || languages.Length == 0)
            throw new ArgumentException(
                "At least one language required", nameof(languages));

        Languages = Array.AsReadOnly(languages);  // immutable wrapper
        Console.WriteLine($"  Developer.ctor: langs {string.Join(", ", Languages)}");
    }

    public override decimal CalculatePay() => base.CalculatePay() + Languages.Count * 1500m;
}

internal static class Demo
{
    public static void Run()
    {
        // Upcast Manager -> Employee (works thanks to inheritance)
        Employee emp = new Manager("Anna", new DateTime(2015, 3, 1), 80000m, 5);
        Console.WriteLine(emp);                         // our ToString runs
        Console.WriteLine($"  Pay: {emp.CalculatePay():C}");

        var dev = new Developer("Ivan", new DateTime(2019, 9, 15), 70000m, "C#", "F#");
        Console.WriteLine(dev);
        Console.WriteLine($"  Pay: {dev.CalculatePay():C}");

        // object is the root of the hierarchy, upcast continues
        object asObject = emp;
        Console.WriteLine($"  asObject.GetType().Name = {asObject.GetType().Name}"); // Manager
    }
}

// Entry point — top-level statements
HrInheritance.Demo.Run();
```

Line-by-line walk-through and lesson concepts applied. The `Employee` class is built exactly after the `Animal` example from the lesson: get-only properties, a constructor with validation that throws `ArgumentException`, and explicit logging at the end of the constructor — all of which makes the "base → derived" call order visible. The `protected` modifier on `BaseSalary` is the exact tool the lesson describes: "`private` members physically exist but are not directly reachable; `public` and `protected` are reachable." We chose `protected` because the derived classes need to read the salary to compute a payout, but the outside world does not. The `Manager` class is marked `sealed` in keeping with the lesson's best practice: "seal classes and methods with `sealed` to protect invariants." The `Manager` constructor must call `: base(name, hireDate, baseSalary)`, because `Employee` has no parameterless constructor — remove `base(...)` and the compiler raises exactly **CS1729**, the error the lesson warns about. Inside `Manager.CalculatePay()` we use `base.CalculatePay()` — the second mode of the `base` keyword, distinct from a constructor call: we do not rewrite the tenure-bonus formula, we reuse it and add our own part. This aligns with the lesson's recommendation to "always pass required data to the base class through `base(...)`, never try to skip initialization" and, more broadly, with the idea that `base` is the "handle to the parent." The `Developer` class is intentionally left unsealed so that in the next lesson (`M05-L02` on `virtual`/`override`) you can derive `SeniorDeveloper` from it and keep experimenting. The `Run` method demonstrates the upcast: the variable `Employee emp` holds a `Manager` object, and `emp.CalculatePay()` — thanks to `override` rather than `new`! — invokes `Manager.CalculatePay()`. The further upcast to `object` and the `GetType().Name` call illustrate that `object` is the single root of the hierarchy while the real type is preserved. The order of the printed lines (`Employee.ctor` before `Manager.ctor`) is empirical confirmation of the lesson's rule: "the base constructor runs to completion first, and only then the derived constructor." Finally, note that we never call `CalculatePay()` from a constructor — this honors the lesson's prohibition on virtual calls in constructors: at the moment `Employee.ctor` runs, the fields of `Manager` are not yet initialized.

#### Going deeper (bonus)
1. Add a fourth class `Intern : Employee` with a `Mentor` field (a string) and an overridden `CalculatePay()` that pays a fixed stipend of `BaseSalary * 0.5m` with no tenure bonus. Think about whether to call `base.CalculatePay()` in this case or, on the contrary, to deliberately bypass it, and justify the choice.
2. Implement an overload of the `Employee` constructor that takes only `name` and `hireDate` and defaults `baseSalary` via `: this(name, hireDate, 50000m)`. Investigate how constructor delegation (`this(...)`) interacts with `base(...)`.
3. Temporarily rewrite `Manager.CalculatePay()` using `new` instead of `override` and call it through a variable of type `Employee`. Record in a comment which method actually ran and why — this visibly exposes the trap described in the lesson.
4. Implement the `IComparable<Employee>` interface, comparing employees by `YearsOfService`, and try to sort an `Employee[]`. Along the way, confirm that interfaces can be added to classes without breaking single inheritance.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Проект собирается без ошибок `dotnet build`.
- [ ] (RU) Три класса `Employee`/`Manager`/`Developer` с наследованием.
- [ ] (RU) Явный `base(...)` в обоих производных конструкторах.
- [ ] (RU) `protected BaseSalary`, проверки аргументов, `override` (не `new`), `base.CalculatePay()`.
- [ ] (RU) Демонстрация апкаста к `Employee` и `object`, `GetType().Name`.
- [ ] (RU) Зафиксирована ошибка CS1729 в комментарии после негативного эксперимента.
- [ ] (EN) Project builds cleanly with `dotnet build`.
- [ ] (EN) Three classes `Employee`/`Manager`/`Developer` wired by inheritance.
- [ ] (EN) Explicit `base(...)` in both derived constructors.
- [ ] (EN) `protected BaseSalary`, argument checks, `override` (not `new`), `base.CalculatePay()`.
- [ ] (EN) Demonstration of upcast to `Employee` and `object`, `GetType().Name`.
- [ ] (EN) CS1729 error text recorded in a comment after the negative experiment.

#### Ресурсы / Resources
- [Microsoft Learn — Inheritance (C#)](https://learn.microsoft.com/dotnet/csharp/fundamentals/object-oriented/inheritance)
- [Microsoft Learn — Constructors (C# programming guide)](https://learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/constructors)
- [Microsoft Learn — Access modifiers (C#)](https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/access-modifiers)
- [Microsoft Learn — base keyword (C# reference)](https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/base)
- [Microsoft Learn — sealed (C# reference)](https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/sealed)

---
[← К уроку M05-L01](lesson-M05-L01-inheritance-base.md) | [⬆ К модулю M05](../README.md) | [Следующее ДЗ →](homework-M05-L02-virtual-override.md)
