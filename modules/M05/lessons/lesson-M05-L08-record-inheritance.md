[⬆ К модулю M05](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M05-L08: record и inheritance, with / record and inheritance, with

**Модуль / Module:** M05
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Тип `record` в C# задумывался как компактный способ описывать неизменяемые модели данных с автоматически сгенерированными членами: конструктором, свойствами, равенством по значениям, `ToString` и `Deconstruct`. Однако `record` — это полноценный ссылочный тип (или `record struct` — значимый), и он полностью поддерживает наследование. Это отличает его от обычных `class` тем, что наследование у records сохраняет семантику равенства по значениям и позволяет элегантно «клонировать» объекты через `with`-выражения, в том числе по иерархии.

Представьте records как аккуратно заполненные анкеты. Базовая анкета содержит имя и возраст. Производная анкета добавляет отдел. Когда вы берёте копию анкеты через `with`, вам не нужно переписывать все поля вручную — компилятор строит новый объект того же самого runtime-типа, копируя неизменяемые значения и подменяя лишь те свойства, что вы указали. В иерархии наследования `with` вызывает виртуальный клонирующий механизм: он создаёт новую запись через защищённый синтетический «Copy constructor», который пробрасывается вниз к самому производному типу. Поэтому `with` сохраняет реальный тип объекта, а не «обрезает» его до базового.

Наследование records строится по простому правилу: базовый тип обязан тоже быть `record` (нельзя унаследовать record от обычного `class`). Синтаксис — привычный: `record Derived : Base`. Параметры первичного конструктора базового record нужно передать через `base(...)`. Если в базовом record есть `virtual` свойства, их можно переопределить через `override` в производном. Именно так добавляют новые поля в производные records, сохраняя при этом совместимость с `with`-механикой.

Равенство в иерархии record работает по особенным правилам. Компилятор генерирует `Equals` так, что он сначала проверяет runtime-тип: две записи считаются равными, только если совпадает их фактический (самый производный) тип и равны значения всех свойств и полей. Это значит, что `Teacher` и `Student` никогда не будут равны, даже если они унаследованы от одного `Person` и совпадают по общим полям. Более того, сравнение `base.Equals` обязательно вызывается, чтобы проверить поля родителя. Поэтому равенство «корректно по иерархии»: два `Teacher` равны, только если равны и их `Person`-поля, и `Teacher`-поля.

`with`-выражения в иерархии особенно полезны для так называемых «функциональных обновлений». Например, у вас есть неизменяемый `Teacher` и нужно изменить только зарплату — вы пишете `teacher with { Salary = 90000 }`. Компилятор построит новый `Teacher`, скопировав всё остальное. Если же у базового `Person` есть `virtual` свойство, а в производном оно переопределено, `with` корректно использует переопределённое значение. Главное правило: `with` всегда возвращает тот же runtime-тип, что и исходный объект, поэтому его можно безопасно применять в коллекциях разнородных записей.

Virtual-члены в records ведут себя как в классах, но с одним бонусом: при использовании `with` переопределённые свойства участвуют в клонировании корректно. Это позволяет строить гибкие иерархии, где производные records добавляют поведение, не ломая равенство по значениям. Однако будьте осторожны: если переопределённое свойство возвращает разные значения для одного и того же состояния (например, через вычисление на основе изменяемого поля), семантика равенства может стать неочевидной.

В итоге наследование records даёт вам: компактный синтаксис, автоматическое равенство с учётом runtime-типа, безопасные `with`-обновления по всей иерархии и поддержку `virtual` для расширения поведения. Это идеальный инструмент для моделей предметной области, DTO и value-объектов, где важны иммутабельность и предсказуемое сравнение.

#### Theory (EN)

The `record` type in C# was designed as a concise way to describe immutable data models with auto-generated members: a constructor, properties, value-based equality, a formatted `ToString`, and a `Deconstruct` method. Yet a `record` is a full reference type (or a `record struct` for a value type), and it fully supports inheritance. What makes records special compared to plain `class` is that inheritance preserves value-equality semantics and enables elegant object cloning through `with`-expressions, including across hierarchies.

Think of records as neatly filled forms. The base form holds a name and an age. A derived form adds a department. When you copy a form through `with`, you do not have to rewrite every field by hand — the compiler builds a new object of the same runtime type, copying the immutable values and overriding only the properties you specify. In an inheritance hierarchy, `with` relies on a virtual cloning mechanism: it constructs a new record through a synthetic protected copy constructor that is dispatched down to the most derived type. Because of this, `with` preserves the actual runtime type of the object instead of truncating it to the base type.

Inheritance for records follows a simple rule: the base type must itself be a `record` — you cannot inherit a record from a plain `class`. The syntax is familiar: `record Derived : Base`. Primary constructor parameters of the base record must be forwarded through `base(...)`. If the base record declares `virtual` properties, you can override them with `override` in the derived record. This is exactly how derived records add new fields while staying compatible with the `with` mechanism.

Equality in a record hierarchy follows special rules. The compiler generates `Equals` so that it first checks the runtime type: two records are equal only when their actual (most derived) types match and all properties and fields are equal by value. This means a `Teacher` and a `Student` are never equal, even when both inherit from the same `Person` and share identical common fields. Moreover, the generated equality calls `base.Equals` to compare the parent fields. That makes equality correct across the hierarchy: two `Teacher` instances are equal only when both their `Person` fields and their `Teacher` fields are equal.

`with`-expressions in a hierarchy are especially useful for so-called functional updates. For example, when you have an immutable `Teacher` and need to change only the salary, you write `teacher with { Salary = 90000 }`. The compiler builds a new `Teacher`, copying everything else. If a base `Person` declares a `virtual` property that is overridden in a derived record, `with` uses the overridden value correctly. The key rule is that `with` always returns the same runtime type as the original object, so it is safe to use in collections of heterogeneous records.

Virtual members in records behave like virtual members in classes, with one bonus: during a `with` clone, overridden properties participate correctly. This lets you build flexible hierarchies where derived records add behavior without breaking value equality. Be careful, however: if an overridden property returns different values for the same state (for example, by computing from a mutable field), equality semantics can become confusing.

In summary, record inheritance gives you a compact syntax, automatic equality that accounts for the runtime type, safe `with` updates across the whole hierarchy, and support for `virtual` to extend behavior. It is an ideal tool for domain models, DTOs, and value objects where immutability and predictable comparison matter.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — record inheritance, with-expressions, virtual, equality
// Запуск: dotnet run — рабочий и самодостаточный пример / runnable and self-contained

using System;

// Базовый record с виртуальным свойством / Base record with a virtual property
// Базовая анкета: имя и возраст / Base form: name and age
public record Person(string Name, int Age)
{
    // Виртуальное свойство можно переопределить в наследнике / virtual can be overridden
    public virtual string Display => $"{Name} ({Age})";
}

// Производный record добавляет поле и переопределяет Display
// Derived record adds a field and overrides Display
public record Teacher(string Name, int Age, string Subject, decimal Salary)
    : Person(Name, Age)
{
    // override сохраняет корректное поведение с with / override stays correct with with
    public override string Display => $"Teacher {Name} teaches {Subject}";

    // Дополнительный неизменяемый член / additional immutable member
    public decimal AnnualSalary => Salary * 12m;
}

// Ещё один производный record — другой ветвь иерархии
// Another derived record — a different branch of the hierarchy
public record Student(string Name, int Age, string Group)
    : Person(Name, Age)
{
    public override string Display => $"Student {Name} in group {Group}";
}

internal static class Program
{
    private static void Main()
    {
        // 1) with сохраняет runtime-тип в иерархии / with keeps the runtime type
        Teacher t1 = new("Анна", 35, "Math", 80000m);
        Teacher t2 = t1 with { Salary = 90000m };       // новый Teacher / new Teacher
        Console.WriteLine(t2);                           // Teacher { Name = Анна, Age = 35, Subject = Math, Salary = 90000 }
        Console.WriteLine(t2.Display);                   // Teacher Анна teaches Math
        Console.WriteLine(t2.AnnualSalary);              // 1080000

        // 2) Равенство учитывает runtime-тип / equality accounts for runtime type
        Teacher t3 = t1 with { Salary = 80000m };
        Console.WriteLine(t1 == t3);                     // True — тот же тип, равные значения / same type, equal values
        Console.WriteLine(t1.Equals(t3));                // True

        // 3) Разные производные типы никогда не равны / different derived types never equal
        Student s1 = new("Анна", 35, "A1");
        Console.WriteLine(t1 == s1);                     // False — Teacher != Student, даже если имя/возраст совпадают

        // 4) with и виртуальные свойства в иерархии / with and virtual properties
        Person p = t1;                                   // upcast — реальный тип остаётся Teacher / upcast keeps Teacher
        Person p2 = p with { Age = 36 };                 // создаётся новый Teacher / a new Teacher is created
        Console.WriteLine(p2.GetType().Name);            // Teacher
        Console.WriteLine(p2.Display);                   // Teacher Анна teaches Math

        // 5) Равенство по базовым полям + своим / equality combines base and derived fields
        Teacher t4 = new("Анна", 35, "Math", 80000m);
        Teacher t5 = new("Анна", 35, "Physics", 80000m);
        Console.WriteLine(t4 == t5);                     // False — Subject отличается / Subject differs
    }
}
```

#### Best Practices

- Делайте базовый record максимально простым и иммутабельным, а расширения — через производные records, а не через изменяемые поля. Это сохраняет предсказуемое равенство.
- Keep the base record simple and immutable; extend behavior through derived records rather than mutable fields. This keeps equality predictable.
- Используйте `with` для «функциональных обновлений» в иерархии вместо ручного копирования — компилятор гарантирует сохранение runtime-типа.
- Use `with` for functional updates in a hierarchy instead of manual copying — the compiler guarantees the runtime type is preserved.
- Объявляйте `virtual` свойства в базовом record только если их действительно нужно переопределять; иначе компилятор всё сделает за вас.
- Declare `virtual` properties in the base record only when you genuinely need to override them; otherwise let the compiler do the work.
- Не смешивайте `class` и `record` в одной иерархии наследования — базовый тип для record обязан быть record.
- Do not mix `class` and `record` in one inheritance chain — a record's base type must also be a record.
- Переопределяйте `Display`/`ToString` через `virtual` + `override`, если формат вывода зависит от производного типа.
- Override `Display`/`ToString` through `virtual` + `override` when the output format depends on the derived type.

#### Частые ошибки / Common Mistakes

- Попытка унаследовать record от обычного `class` → компилятор выдаст ошибку; используйте только record как базовый тип / Trying to inherit a record from a plain `class` → the compiler errors; use only a record as the base type.
- Ожидание, что `with` «обрежет» объект до базового типа → `with` всегда сохраняет runtime-тип; upcast не влияет на результат / Expecting `with` to truncate the object to the base type → `with` always keeps the runtime type; upcasting does not change the result.
- Сравнение records разных производных типов через `==` и ожидание `true` → равенство требует совпадения runtime-типа; разные ветви иерархии никогда не равны / Comparing records of different derived types with `==` and expecting `true` → equality requires the same runtime type; different branches are never equal.
- Забыли передать параметры в `base(...)` у производного record → компилятор требует явного вызова базового конструктора / Forgetting to pass parameters to `base(...)` in a derived record → the compiler requires an explicit base constructor call.
- Переопределение свойства через `new` вместо `override` → `with` и виртуальный вызов будут использовать базовую версию; используйте `override` / Shadowing a property with `new` instead of `override` → `with` and virtual dispatch use the base version; use `override`.
- Изменение состояния внутри переопределённого свойства, влияющее на равенство → нарушение семантики record; оставляйте свойства детерминированными / Mutating state inside an overridden property that affects equality → breaks record semantics; keep properties deterministic.
- Использование `with` на `object`, имеющем ссылку на изменяемый внешний объект → глубокая копия не создаётся; `with` копирует только ссылки / Using `with` on a record that references a mutable external object → no deep copy is made; `with` copies references only.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Базовый тип для производного record сам является record.
- [ ] The base type of a derived record is itself a record.
- [ ] В производном record параметры базового конструктора переданы через `base(...)`.
- [ ] In the derived record, base constructor parameters are forwarded through `base(...)`.
- [ ] При `with` runtime-тип результата совпадает с исходным объектом.
- [ ] After a `with` expression, the result runtime type matches the source object.
- [ ] Два records разных производных типов не равны, даже если общие поля совпадают.
- [ ] Two records of different derived types are not equal even when shared fields match.
- [ ] `virtual` свойства переопределены через `override`, а не через `new`.
- [ ] `virtual` properties are overridden with `override`, not `new`.
- [ ] `with` корректно копирует переопределённые свойства производного типа.
- [ ] `with` correctly copies overridden properties of the derived type.
- [ ] Равенство вызывается через `==` или `Equals` и учитывает все поля иерархии.
- [ ] Equality via `==` or `Equals` accounts for all fields across the hierarchy.

#### Ресурсы / Resources

- [Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/fundamentals/types/records](https://learn.microsoft.com/dotnet/csharp/fundamentals/types/records)

---

[⬆ К модулю M05](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
