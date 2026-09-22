---
[← К уроку M05-L08](lesson-M05-L08-record-inheritance.md) | [⬆ К модулю M05](../README.md) | [Следующее ДЗ →](homework-M05-L09-composition-vs-inheritance.md)
---

### Домашнее задание M05-L08: record и inheritance, with / Homework M05-L08: record and inheritance, with

**Урок / Lesson:** M05-L08
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться строить иерархии records, корректно применять `with`-выражения с сохранением runtime-типа, переопределять `virtual`-свойства через `override` и понимать равенство по значениям с учётом фактического типа в иерархии. (EN) Learn to build record hierarchies, apply `with`-expressions correctly while preserving the runtime type, override `virtual` properties with `override`, and understand value equality that accounts for the actual runtime type across a hierarchy.

#### Связь с уроком / Connection to the lesson
(RU) Урок показывает, что `record` — это полноценный ссылочный тип с наследованием, где `with` использует виртуальный клонирующий механизм и сохраняет runtime-тип, а равенство требует совпадения самого производного типа. Это ДЗ закрепляет все эти механизки на реалистичной предметной модели: иерархия сотрудников учебного заведения.
(EN) The lesson shows that a `record` is a full reference type supporting inheritance, where `with` relies on a virtual cloning mechanism and preserves the runtime type, and equality requires the most derived types to match. This homework cements all of these mechanisms on a realistic domain model: a hierarchy of educational institution members.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Вы разрабатываете модуль учёта кадров учебного заведения «Лицей №7». В системе фигурируют разные категории людей: преподаватели, студенты и административный персонал. Все они имеют общие атрибуты — имя и возраст, — но у каждой категории есть своя специфика: у преподавателя — предмет и зарплата, у студента — учебная группа и средний балл, у администратора — должность и ставка. Требуется смоделировать эти данные так, чтобы они были строго иммутабельны, сравнивались по значениям, удобно «обновлялись» функционально (без мутации исходного объекта) и безопасно сериализовались. Команда архитекторов выбрала `record` именно потому, что он даёт готовое равенство, `ToString`, `Deconstruct` и `with`-обновления, а наследование позволяет выразить иерархию без boilerplate. При этом критически важно, чтобы «обновление» записи через `with` сохраняло её реальный тип (например, повышение зарплаты преподавателю не превращало бы его в базового `Person`), а сравнение двух людей разных категорий никогда не давало ложного равенства, даже если имя и возраст совпадают. Эти свойства — прямое следствие того, как компилятор C# 12 генерирует `Equals` и клонирующий конструктор для records. В этом задании вы пройдёте весь цикл: от создания проекта через `dotnet` до проверки каждого свойства системы на практике, включая работу с коллекцией разнородных записей и `with`-обновления после upcast к базовому типу.

#### Что нужно сделать (пошагово)

1. Создайте консольный проект .NET 8 с именем `SchoolRecords`:
   ```bash
   dotnet new console -n SchoolRecords -o SchoolRecords --framework net8.0
   cd SchoolRecords
   ```
   Убедитесь, что в `SchoolRecords.csproj` указано `<LangVersion>latest</LangVersion>` или используется C# 12 по умолчанию для .NET 8. Откройте `Program.cs` и удалите шаблонный код.

2. Определите базовый `record Person(string Name, int Age)` с виртуальным свойством `public virtual string Display => $"{Name} ({Age})";`. Это будет корень иерархии; именно он задаёт общие поля и точку расширения для форматированного вывода.

3. Создайте производный `record Teacher(string Name, int Age, string Subject, decimal Salary) : Person(Name, Age)` с переопределённым `Display` через `override` и вычисляемым `public decimal AnnualSalary => Salary * 12m;`. Обязательно передайте параметры базового конструктора через `base(...)` — для records с первичным конструктором это делается в сигнатуре через `: Person(Name, Age)`.

4. Создайте `record Student(string Name, int Age, string Group, double Gpa) : Person(Name, Age)` со своим `override Display` и свойством `public bool IsHonors => Gpa >= 4.5;`.

5. Создайте `record Admin(string Name, int Age, string Position, decimal HourlyRate) : Person(Name, Age)` со своим `override Display` и свойством `public decimal MonthlyPay(int hours) => HourlyRate * hours;`.

6. В `Program.cs` (top-level statements) продемонстрируйте:
   - создание `Teacher t1`, обновление зарплаты через `t1 with { Salary = 90000m }` и вывод `t2.Display`, `t2.AnnualSalary`;
   - проверку равенства `t1 == t3` где `t3` получен `with` с теми же значениями — ожидается `True`;
   - создание `Student s1` с теми же `Name` и `Age`, что у `t1`, и вывод `t1 == s1` — ожидается `False` (разные runtime-типы);
   - upcast `Person p = t1;` и обновление `Person p2 = p with { Age = 36 };`, затем вывод `p2.GetType().Name` — ожидается `Teacher`, а не `Person`;
   - коллекцию `List<Person> people = [t1, s1, new Admin(...)];` с использованием collection expression и обход через `switch`-pattern matching, печатающий `Display` и категорию.

7. Запустите проект: `dotnet run`. Зафиксируйте вывод в виде комментария в конце файла. Ожидаемые строки — например: `Teacher`, `Teacher Анна teaches Math`, `True`, `False` и т.д.

8. Добавьте unit-тесты (опционально, но рекомендуется): создайте проект `dotnet new xunit -n SchoolRecords.Tests` и проверьте равенство, сохранение типа через `with` и поведение `Display` после upcast.

#### Требования к решению

Решение должно компилироваться под .NET 8 / C# 12 без предупреждений уровня error. Все типы должны быть `record`, базовым типом для `Teacher`, `Student` и `Admin` обязан быть `Person` — попытка унаследовать record от обычного `class` в коде не допускается и должна отсутствовать. Параметры базового конструктора обязаны передаваться через `: Person(Name, Age)` в сигнатуре производного record; вручную реализовывать конструктор не нужно. Виртуальное свойство `Display` должно переопределяться только через `override`; использование `new` для сокрытия считается ошибкой и будет проверяться через поведение `with` после upcast. Все свойства данных должны оставаться иммутабельными (init-only через первичный конструктор); изменяемые поля не допускаются. `with`-выражения применяются для всех «обновлений», ручное копирование полей запрещено. Код должен использовать современные возможности C# 12: top-level statements, collection expressions (`[ ... ]`), pattern matching в `switch`, вывод типов `new(...)` где уместно. Вывод программы должен строго соответствовать ожидаемому — это доказывает, что механизмы `with` и равенства работают так, как описано в уроке.

#### Тонкости и подводные камни

- `with` всегда сохраняет runtime-тип исходного объекта. После `Person p = t1; Person p2 = p with { Age = 36 };` тип `p2` остаётся `Teacher`, а не `Person`. Компилятор вызывает виртуальный защищённый клонирующий конструктор, который пробрасывается к самому производному типу. Если вы по ошибке замените `override` на `new` у `Display`, то после upcast и `with` будет вызвана базовая версия — это частый источник «странных» выводов.
- Равенство records требует совпадения самого производного runtime-типа. `Teacher` и `Student` с одинаковыми `Name`/`Age` не равны, потому что `Equals` первым делом сравнивает `EqualityContract` (внутреннее свойство, хранящее тип). Не пытайтесь «починить» это ручной реализацией `Equals` — вы сломаете иерархическую корректность.
- Базовый тип для record обязан быть record. Наследование `record X : SomeClass` вызовет ошибку компиляции. Если у вас есть общий класс, превратите его в record или используйте композицию (тема следующего урока M05-L09).
- `with` копирует ссылки, а не делает глубокую копию. Если в record есть свойство-коллекция или ссылка на изменяемый объект, оба экземпляра будут разделять этот объект. Для коллекций используйте неизменяемые типы (`ImmutableArray`, `FrozenSet`) или аккуратно пересоздавайте их в `with`.
- Вычисляемые свойства вроде `AnnualSalary` не участвуют в равенстве (равенство строится по свойствам первичного конструктора и явным полям), но если вы переопределите `virtual` свойство так, что оно вернёт разные значения для равных состояний, это запутает логику. Держите `override`-свойства детерминированными и чистыми.
- Не забудьте `: Person(Name, Age)` — без него компилятор не сможет вызвать базовый конструктор и выдаст ошибку. Порядок параметров в первичном конструкторе производного record не обязан совпадать с базовым, но передаваемые аргументы должны соответствовать.
- `==` для records вызывает сгенерированный `op_Equality`, который внутри сводится к `Equals`; `null == null` тоже корректно обрабатывается. Но при сравнении через `object.Equals(a, b)` или `EqualityComparer<T>` используется тот же сгенерированный `Equals`.

#### Критерии приёмки

- [ ] Создан проект `SchoolRecords` на .NET 8 через `dotnet new console`.
- [ ] Базовый `record Person(string Name, int Age)` объявлен с `virtual string Display`.
- [ ] `Teacher`, `Student`, `Admin` унаследованы от `Person` через `: Person(Name, Age)`.
- [ ] В каждом производном record `Display` переопределён через `override` (не `new`).
- [ ] `Teacher.AnnualSalary`, `Student.IsHonors`, `Admin.MonthlyPay` реализованы как вычисляемые свойства.
- [ ] `with` применяется для всех обновлений; ручное копирование полей отсутствует.
- [ ] После `Person p = t1; Person p2 = p with { Age = 36 };` выводится `Teacher` как тип `p2`.
- [ ] `t1 == t3` (одинаковые значения) даёт `True`.
- [ ] `t1 == s1` (разные производные типы) даёт `False`.
- [ ] Коллекция `List<Person>` создана через collection expression `[ ... ]`.
- [ ] Обход коллекции использует `switch` pattern matching с типами.
- [ ] Программа запускается `dotnet run` без ошибок и предупреждений.
- [ ] Вывод зафиксирован в комментарии в конце файла и соответствует ожидаемому.
- [ ] Код использует top-level statements и C# 12.
- [ ] (Бонус) Добавлены xUnit-тесты на равенство и сохранение типа через `with`.

#### Подсказки (без прямого ответа)

- Вспомните, что `with` для record — это синтаксический сахар над клонирующим конструктором. Подумайте, почему именно поэтому тип сохраняется.
- Если `Display` ведёт себя «неожиданно» после upcast, проверьте, `override` ли это, а не `new`.
- Для pattern matching по `Person` используйте `switch` с `case Teacher t:` / `case Student s:` / `case Admin a:` / `case Person p:` (последний — fallback).
- Collection expression `List<Person> people = [t1, s1, admin];` работает начиная с C# 12; убедитесь, что `LangVersion` актуален.
- Равенство проверяйте и через `==`, и через `.Equals()` — они должны дать одинаковый результат.

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8 — SchoolRecords
// Запуск: dotnet run
// Базовый record + три производных, with, override, equality, pattern matching / Base + 3 derived, with, override, equality, pattern matching

// Базовая анкета: имя и возраст, точка расширения Display / Base form: name, age, Display extension point
public record Person(string Name, int Age)
{
    // virtual — чтобы производные records могли переопределить / virtual so derived records can override
    public virtual string Display => $"{Name} ({Age})";
}

// Производный: преподаватель, добавляет предмет и зарплату / Derived: teacher adds subject and salary
public record Teacher(string Name, int Age, string Subject, decimal Salary)
    : Person(Name, Age) // base(...) обязательно / base(...) is mandatory
{
    // override сохраняет корректное поведение с with / override keeps with correct
    public override string Display => $"Teacher {Name} teaches {Subject}";
    // Вычисляемое свойство не входит в равенство / computed property is not part of equality
    public decimal AnnualSalary => Salary * 12m;
}

// Производный: студент, добавляет группу и средний балл / Derived: student adds group and GPA
public record Student(string Name, int Age, string Group, double Gpa)
    : Person(Name, Age)
{
    public override string Display => $"Student {Name}, group {Group}";
    public bool IsHonors => Gpa >= 4.5;
}

// Производный: администратор, добавляет должность и ставку / Derived: admin adds position and rate
public record Admin(string Name, int Age, string Position, decimal HourlyRate)
    : Person(Name, Age)
{
    public override string Display => $"Admin {Name}, {Position}";
    public decimal MonthlyPay(int hours) => HourlyRate * hours;
}

// top-level statements — точка входа / top-level statements — entry point
Teacher t1 = new("Анна", 35, "Math", 80000m);
Teacher t2 = t1 with { Salary = 90000m };               // новый Teacher, копия остальных полей / new Teacher, rest copied
Console.WriteLine(t2);                                   // Teacher { Name = Анна, Age = 35, Subject = Math, Salary = 90000 }
Console.WriteLine(t2.Display);                           // Teacher Анна teaches Math
Console.WriteLine(t2.AnnualSalary);                      // 1080000

Teacher t3 = t1 with { Salary = 80000m };
Console.WriteLine(t1 == t3);                             // True — тот же тип, равные значения / same type, equal values
Console.WriteLine(t1.Equals(t3));                        // True

Student s1 = new("Анна", 35, "A1", 4.7);
Console.WriteLine(t1 == s1);                             // False — разные runtime-типы / different runtime types

// upcast + with: runtime-тип сохраняется / upcast + with: runtime type preserved
Person p = t1;
Person p2 = p with { Age = 36 };
Console.WriteLine(p2.GetType().Name);                    // Teacher
Console.WriteLine(p2.Display);                           // Teacher Анна teaches Math

// коллекция разнородных records через collection expression / heterogeneous collection via collection expression
List<Person> people = [t1, s1, new Admin("Игорь", 42, "Registrar", 500m)];
foreach (Person person in people)
{
    string category = person switch
    {
        Teacher => "преподаватель / teacher",
        Student st when st.IsHonors => "студент-отличник / honors student",
        Student => "студент / student",
        Admin => "администратор / admin",
        _ => "человек / person"
    };
    Console.WriteLine($"{person.Display}  [{category}]");
}

// Ожидаемый вывод / Expected output:
// Teacher { Name = Анна, Age = 35, Subject = Math, Salary = 90000 }
// Teacher Анна teaches Math
// 1080000
// True
// True
// False
// Teacher
// Teacher Анна teaches Math
// Teacher Анна teaches Math  [преподаватель / teacher]
// Student Анна, group A1  [студент-отличник / honors student]
// Admin Игорь, Registrar  [администратор / admin]
```

Разбор по строкам. Базовый `record Person` задаёт общие неизменяемые поля и `virtual Display` — это та точка, через которую производные records смогут менять формат вывода, не ломая равенство (равенство строится по свойствам первичного конструктора, а `Display` в него не входит). `Teacher`, `Student`, `Admin` наследуются через `: Person(Name, Age)`: это обязательная форма передачи аргументов базовому конструктору для record с первичным конструктором; без неё компилятор не соберёт код. В каждом производном record `Display` переопределён через `override`, а не `new` — это критично: именно `override` гарантирует, что после upcast к `Person` и `with` будет вызвана версия производного типа, потому что клонирующий механизм сохраняет runtime-тип, а виртуальный диспетчер выбирает переопределённый член. Вычисляемые свойства (`AnnualSalary`, `IsHonors`, `MonthlyPay`) показывают, что в record можно добавлять поведение, не влияя на равенство: компилятор включает в `Equals` только члены первичного конструктора и явно заданные поля.

Строка `Teacher t2 = t1 with { Salary = 90000m };` — ядро задания. Компилятор превращает её в вызов клонирующего конструктора `Teacher`, который копирует все поля и подменяет `Salary`. Результат — новый `Teacher`, а не `Person`. Проверка `t1 == t3` показывает, что два `Teacher` с одинаковыми значениями всех полей равны: сгенерированный `Equals` сравнивает runtime-тип (через `EqualityContract`) и затем значения всех свойств. Проверка `t1 == s1` подтверждает второе правило: разные производные типы никогда не равны, даже если `Name` и `Age` совпадают, — первым сравнением идёт именно тип. Блок `Person p = t1; Person p2 = p with { Age = 36 };` — это и есть демонстрация виртуального клонирования: несмотря на то, что статический тип `p` — `Person`, `with` создаёт `Teacher`, что и подтверждает `p2.GetType().Name`. Collection expression `[t1, s1, new Admin(...)]` и `switch` с типами и `when`-фильтром показывают, как разнородные records обрабатываются единообразно через базовый тип, сохраняя при этом доступ к специфике производных.

#### Задания на углубление (бонус)

1. Добавьте `record DepartmentHead` — производный от `Teacher`, добавляющий `Bonus`. Проверьте, что `with` на `DepartmentHead` сохраняет и `Teacher`-поля, и `DepartmentHead`-поля, а равенство учитывает всю глубину иерархии.
2. Реализуйте immutable-коллекцию предметов у преподавателя: `ImmutableArray<string> Subjects` вместо одного `Subject`. Покажите, что `with` копирует ссылку на массив, и предложите безопасный способ обновления через `teacher with { Subjects = teacher.Subjects.Add("Physics") }`.
3. Сериализуйте коллекцию `people` в JSON через `System.Text.Json` и десериализуйте обратно с сохранением производных типов (потребуется кастомный конвертер или `[JsonDerivedType]`). Обсудите, почему полиморфная сериализация records нетривиальна.
4. Сравните производительность `with` против ручного конструирования через `BenchmarkDotNet` на 100 000 итераций и объясните результат через сгенерированный IL клонирующего конструктора.

---

## Statement in English / Постановка на английском

#### Context & motivation

You are developing a staffing module for an educational institution, "Lyceum No. 7". The system deals with several categories of people: teachers, students, and administrative staff. They all share common attributes — a name and an age — but each category has its own specifics: a teacher has a subject and a salary, a student has a study group and a grade point average, and an administrator has a position and an hourly rate. You need to model this data so that it is strictly immutable, compared by value, conveniently updated functionally (without mutating the original object), and safe to serialize. The architecture team chose `record` precisely because it provides ready-made equality, `ToString`, `Deconstruct`, and `with`-based updates, while inheritance lets you express the hierarchy without boilerplate. It is critically important that updating a record through `with` preserves its real type — for example, giving a teacher a raise must not turn them into a base `Person` — and that comparing two people of different categories never yields a false equality, even when their name and age coincide. These properties follow directly from how the C# 12 compiler generates `Equals` and the cloning constructor for records. In this assignment you will go through the whole cycle: from creating a project with `dotnet` to verifying every property of the system in practice, including work with a collection of heterogeneous records and `with`-updates after an upcast to the base type.

#### What to do step by step

1. Create a .NET 8 console project named `SchoolRecords`:
   ```bash
   dotnet new console -n SchoolRecords -o SchoolRecords --framework net8.0
   cd SchoolRecords
   ```
   Make sure `SchoolRecords.csproj` sets `<LangVersion>latest</LangVersion>` or simply relies on the C# 12 default that ships with .NET 8. Open `Program.cs` and remove the template code.

2. Define a base `record Person(string Name, int Age)` with a virtual property `public virtual string Display => $"{Name} ({Age})";`. This will be the root of the hierarchy; it declares the common fields and an extension point for formatted output.

3. Create a derived `record Teacher(string Name, int Age, string Subject, decimal Salary) : Person(Name, Age)` with an overridden `Display` via `override` and a computed `public decimal AnnualSalary => Salary * 12m;`. You must forward base constructor parameters through `base(...)`, which for records with a primary constructor is written directly in the signature as `: Person(Name, Age)`.

4. Create `record Student(string Name, int Age, string Group, double Gpa) : Person(Name, Age)` with its own `override Display` and a property `public bool IsHonors => Gpa >= 4.5;`.

5. Create `record Admin(string Name, int Age, string Position, decimal HourlyRate) : Person(Name, Age)` with its own `override Display` and a method `public decimal MonthlyPay(int hours) => HourlyRate * hours;`.

6. In `Program.cs` (top-level statements), demonstrate:
   - creating a `Teacher t1`, updating the salary through `t1 with { Salary = 90000m }`, and printing `t2.Display` and `t2.AnnualSalary`;
   - an equality check `t1 == t3` where `t3` is obtained via `with` with the same values — expect `True`;
   - creating a `Student s1` with the same `Name` and `Age` as `t1` and printing `t1 == s1` — expect `False` (different runtime types);
   - an upcast `Person p = t1;` and an update `Person p2 = p with { Age = 36 };`, then printing `p2.GetType().Name` — expect `Teacher`, not `Person`;
   - a collection `List<Person> people = [t1, s1, new Admin(...)];` built with a collection expression, iterated with a `switch` pattern matching expression that prints `Display` and the category.

7. Run the project: `dotnet run`. Capture the output as a comment at the end of the file. The expected lines include `Teacher`, `Teacher Анна teaches Math`, `True`, `False`, and so on.

8. Add unit tests (optional but recommended): create a project with `dotnet new xunit -n SchoolRecords.Tests` and verify equality, type preservation through `with`, and `Display` behavior after an upcast.

#### Requirements

The solution must compile under .NET 8 / C# 12 with no error-level warnings. All types must be `record`; the base type of `Teacher`, `Student`, and `Admin` must be `Person` — trying to inherit a record from a plain `class` is not allowed and must not appear in the code. Base constructor parameters must be forwarded through `: Person(Name, Age)` in the derived record signature; do not implement the constructor manually. The virtual `Display` property must be overridden only with `override`; using `new` to shadow it is considered an error and will be checked through the behavior of `with` after an upcast. All data properties must remain immutable (init-only through the primary constructor); mutable fields are not allowed. `with`-expressions are used for all updates; manual field copying is forbidden. The code should use modern C# 12 features: top-level statements, collection expressions (`[ ... ]`), pattern matching in `switch`, and target-typed `new(...)` where appropriate. The program output must exactly match the expected output — this is the proof that the `with` and equality mechanisms behave as described in the lesson.

#### Pitfalls

- `with` always preserves the runtime type of the source object. After `Person p = t1; Person p2 = p with { Age = 36 };`, the type of `p2` stays `Teacher`, not `Person`. The compiler invokes a virtual protected cloning constructor that is dispatched to the most derived type. If you mistakenly replace `override` with `new` on `Display`, the base version will be called after an upcast and `with` — a common source of "weird" output.
- Record equality requires the most derived runtime types to match. A `Teacher` and a `Student` with identical `Name`/`Age` are not equal, because `Equals` first compares `EqualityContract` (an internal property holding the type). Do not try to "fix" this with a hand-written `Equals` — you will break hierarchical correctness.
- The base type of a record must itself be a record. Inheriting as `record X : SomeClass` causes a compile error. If you have an existing common class, turn it into a record or use composition (the topic of the next lesson M05-L09).
- `with` copies references; it does not deep-copy. If a record has a collection property or a reference to a mutable object, both instances will share that object. For collections, use immutable types (`ImmutableArray`, `FrozenSet`) or carefully recreate them inside `with`.
- Computed properties such as `AnnualSalary` do not participate in equality (equality is built from primary constructor properties and explicit fields), but if you override a `virtual` property so that it returns different values for equal states, you will confuse the logic. Keep `override` properties deterministic and pure.
- Do not forget `: Person(Name, Age)` — without it the compiler cannot call the base constructor and will error. The order of parameters in the derived record's primary constructor need not match the base, but the forwarded arguments must align.
- `==` for records calls the generated `op_Equality`, which delegates to `Equals`; `null == null` is also handled correctly. But comparing through `object.Equals(a, b)` or `EqualityComparer<T>` uses the same generated `Equals`.

#### Acceptance criteria

- [ ] A `SchoolRecords` project on .NET 8 is created via `dotnet new console`.
- [ ] The base `record Person(string Name, int Age)` declares `virtual string Display`.
- [ ] `Teacher`, `Student`, and `Admin` inherit from `Person` through `: Person(Name, Age)`.
- [ ] In every derived record, `Display` is overridden with `override` (not `new`).
- [ ] `Teacher.AnnualSalary`, `Student.IsHonors`, and `Admin.MonthlyPay` are implemented as computed members.
- [ ] `with` is used for all updates; manual field copying is absent.
- [ ] After `Person p = t1; Person p2 = p with { Age = 36 };`, the program prints `Teacher` as the type of `p2`.
- [ ] `t1 == t3` (equal values) yields `True`.
- [ ] `t1 == s1` (different derived types) yields `False`.
- [ ] The `List<Person>` collection is created with a collection expression `[ ... ]`.
- [ ] Iterating the collection uses a `switch` pattern matching expression with type patterns.
- [ ] The program runs with `dotnet run` without errors or warnings.
- [ ] The output is captured in a comment at the end of the file and matches the expected output.
- [ ] The code uses top-level statements and C# 12.
- [ ] (Bonus) xUnit tests for equality and type preservation through `with` are added.

#### Hints (no direct answer)

- Recall that `with` on a record is syntactic sugar over the cloning constructor. Think about why exactly that is what preserves the type.
- If `Display` behaves "unexpectedly" after an upcast, check whether it is an `override` rather than a `new`.
- For pattern matching over `Person`, use a `switch` with `case Teacher t:` / `case Student s:` / `case Admin a:` / `case Person p:` (the last one as a fallback).
- The collection expression `List<Person> people = [t1, s1, admin];` works starting with C# 12; make sure `LangVersion` is current.
- Verify equality both via `==` and via `.Equals()` — they must produce the same result.

#### Reference solution walk-through

```csharp
// C# 12 / .NET 8 — SchoolRecords
// Run: dotnet run
// Base record + three derived, with, override, equality, pattern matching

// Base form: name and age, Display extension point
public record Person(string Name, int Age)
{
    // virtual so derived records can override
    public virtual string Display => $"{Name} ({Age})";
}

// Derived: teacher adds subject and salary
public record Teacher(string Name, int Age, string Subject, decimal Salary)
    : Person(Name, Age) // base(...) is mandatory
{
    // override keeps with correct
    public override string Display => $"Teacher {Name} teaches {Subject}";
    // computed property is not part of equality
    public decimal AnnualSalary => Salary * 12m;
}

// Derived: student adds group and GPA
public record Student(string Name, int Age, string Group, double Gpa)
    : Person(Name, Age)
{
    public override string Display => $"Student {Name}, group {Group}";
    public bool IsHonors => Gpa >= 4.5;
}

// Derived: admin adds position and rate
public record Admin(string Name, int Age, string Position, decimal HourlyRate)
    : Person(Name, Age)
{
    public override string Display => $"Admin {Name}, {Position}";
    public decimal MonthlyPay(int hours) => HourlyRate * hours;
}

// top-level statements — entry point
Teacher t1 = new("Anna", 35, "Math", 80000m);
Teacher t2 = t1 with { Salary = 90000m };               // new Teacher, rest copied
Console.WriteLine(t2);                                   // Teacher { Name = Anna, Age = 35, Subject = Math, Salary = 90000 }
Console.WriteLine(t2.Display);                           // Teacher Anna teaches Math
Console.WriteLine(t2.AnnualSalary);                      // 1080000

Teacher t3 = t1 with { Salary = 80000m };
Console.WriteLine(t1 == t3);                             // True — same type, equal values
Console.WriteLine(t1.Equals(t3));                        // True

Student s1 = new("Anna", 35, "A1", 4.7);
Console.WriteLine(t1 == s1);                             // False — different runtime types

// upcast + with: runtime type preserved
Person p = t1;
Person p2 = p with { Age = 36 };
Console.WriteLine(p2.GetType().Name);                    // Teacher
Console.WriteLine(p2.Display);                           // Teacher Anna teaches Math

// heterogeneous collection via collection expression
List<Person> people = [t1, s1, new Admin("Igor", 42, "Registrar", 500m)];
foreach (Person person in people)
{
    string category = person switch
    {
        Teacher => "teacher",
        Student st when st.IsHonors => "honors student",
        Student => "student",
        Admin => "admin",
        _ => "person"
    };
    Console.WriteLine($"{person.Display}  [{category}]");
}

// Expected output:
// Teacher { Name = Anna, Age = 35, Subject = Math, Salary = 90000 }
// Teacher Anna teaches Math
// 1080000
// True
// True
// False
// Teacher
// Teacher Anna teaches Math
// Teacher Anna teaches Math  [teacher]
// Student Anna, group A1  [honors student]
// Admin Igor, Registrar  [admin]
```

Line-by-line walk-through. The base `record Person` defines common immutable fields and a `virtual Display` — the extension point through which derived records change the output format without breaking equality (equality is built from primary constructor properties, and `Display` is not among them). `Teacher`, `Student`, and `Admin` inherit through `: Person(Name, Age)`: this is the mandatory form of forwarding arguments to the base constructor for a record with a primary constructor; without it the code will not compile. In every derived record `Display` is overridden with `override`, not `new` — this is critical: only `override` guarantees that after an upcast to `Person` and a `with`, the derived version is invoked, because the cloning mechanism preserves the runtime type and the virtual dispatcher selects the overridden member. Computed properties (`AnnualSalary`, `IsHonors`, `MonthlyPay`) show that behavior can be added to a record without affecting equality: the compiler includes only primary constructor members and explicit fields in `Equals`.

The line `Teacher t2 = t1 with { Salary = 90000m };` is the core of the assignment. The compiler turns it into a call to the `Teacher` cloning constructor, which copies every field and substitutes `Salary`. The result is a new `Teacher`, not a `Person`. The check `t1 == t3` shows that two `Teacher` instances with identical field values are equal: the generated `Equals` compares the runtime type (via `EqualityContract`) and then the values of all properties. The check `t1 == s1` confirms the second rule: different derived types are never equal, even when `Name` and `Age` match — the type is compared first. The block `Person p = t1; Person p2 = p with { Age = 36 };` is the demonstration of virtual cloning: although the static type of `p` is `Person`, `with` creates a `Teacher`, which `p2.GetType().Name` confirms. The collection expression `[t1, s1, new Admin(...)]` and the `switch` with type patterns and a `when` filter show how heterogeneous records are processed uniformly through the base type while still giving access to derived specifics.

#### Going deeper (bonus)

1. Add a `record DepartmentHead` derived from `Teacher` that adds a `Bonus`. Verify that `with` on a `DepartmentHead` preserves both the `Teacher` fields and the `DepartmentHead` fields, and that equality accounts for the full depth of the hierarchy.
2. Implement an immutable collection of subjects for the teacher: `ImmutableArray<string> Subjects` instead of a single `Subject`. Show that `with` copies the array reference, and propose a safe update through `teacher with { Subjects = teacher.Subjects.Add("Physics") }`.
3. Serialize the `people` collection to JSON with `System.Text.Json` and deserialize it back preserving the derived types (you will need a custom converter or `[JsonDerivedType]`). Discuss why polymorphic serialization of records is non-trivial.
4. Compare the performance of `with` versus manual construction using `BenchmarkDotNet` over 100,000 iterations and explain the result in terms of the generated IL of the cloning constructor.

---

#### Чек-лист сдачи / Submission checklist
- [ ] Проект `SchoolRecords` создан и собирается под .NET 8 / C# 12.
- [ ] Иерархия `Person → Teacher/Student/Admin` реализована через record.
- [ ] `with` сохраняет runtime-тип (демонстрация с upcast).
- [ ] Равенство проверено для совпадающих и разных производных типов.
- [ ] Вывод `dotnet run` зафиксирован и совпадает с ожидаемым.
- [ ] (Бонус) xUnit-тесты добавлены и зелёные.
- [ ] The `SchoolRecords` project is created and builds under .NET 8 / C# 12.
- [ ] The `Person → Teacher/Student/Admin` hierarchy is implemented with records.
- [ ] `with` preserves the runtime type (demonstrated with an upcast).
- [ ] Equality is verified for matching and for different derived types.
- [ ] The `dotnet run` output is captured and matches the expected one.
- [ ] (Bonus) xUnit tests are added and green.

#### Ресурсы / Resources
- [Microsoft Learn — Records (C#)](https://learn.microsoft.com/dotnet/csharp/fundamentals/types/records)
- [Microsoft Learn — `with` expressions](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/with-expression)
- [Microsoft Learn — Inheritance (records)](https://learn.microsoft.com/dotnet/csharp/fundamentals/object-oriented/inheritance)
- [Microsoft Learn — Pattern matching](https://learn.microsoft.com/dotnet/csharp/fundamentals/functional/pattern-matching)
- [Microsoft Learn — Collection expressions](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/collection-expressions)

---

[← К уроку M05-L08](lesson-M05-L08-record-inheritance.md) | [⬆ К модулю M05](../README.md) | [Следующее ДЗ →](homework-M05-L09-composition-vs-inheritance.md)
