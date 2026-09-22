[⬆ К модулю M05](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M05-L02: virtual/override/new / virtual/override/new

**Модуль / Module:** M05
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Полиморфизм — одна из трёх опор объектно-ориентированного программирования (вместе с инкапсуляцией и наследованием). В C# он чаще всего проявляется через механизм виртуальных методов. Суть проста: базовый класс объявляет метод с модификатором `virtual` и тем самым говорит «у меня есть реализация по умолчанию, но производный класс вправе предложить свою, более специфичную». Когда вы вызываете такой метод через ссылку на базовый класс, среда выполнения смотрит не на тип ссылки, а на фактический тип объекта в памяти, и выбирает нужную версию. Это и есть динамическое связывание (dynamic dispatch).

Аналогия: представьте должностную инструкцию менеджера. Базовый класс `Employee` содержит метод `Work()`, который описывает общие обязанности. Класс `Developer` наследуется от `Employee` и переопределяет `Work()` через `override`, потому что программист работает иначе. Если директор обращается к сотруднику как к `Employee` (видит только общую инструкцию), но на самом деле перед ним `Developer`, выполнится именно «программистская» версия метода. Модификатор `override` — это явное заявление: «я заменяю родительскую реализацию, сохраняя контракт».

Чтобы переопределение было возможным, родительский метод должен быть помечен `virtual`, `abstract` или уже быть `override` (в последних двух случаях переопределение разрешено неявно). Просто добавить `override` к обычному методу нельзя — компилятор выдаст ошибку. Это защита от случайных замен: вы должны сознательно разрешить переопределение в базовом классе.

Теперь о ключевом слове `new`. Если в производном классе вы создаёте метод с тем же именем и сигнатурой, но без `override`, компилятор выдаёт предупреждение: «метод скрывает метод базового класса». Если вы действительно хотите скрыть родительский метод (а не переопределить), нужно явно написать `new`. В чём разница? `override` — это полиморфизм: выбор версии идёт по фактическому типу объекта во время выполнения. `new` — это сокрытие: выбор идёт по типу ссылки во время компиляции. Если объект `Developer` хранится в переменной типа `Employee`, вызовется родительский `Work()`, а не «новый». Это часто становится источником тонких багов.

Ключевое слово `sealed` в сочетании с `override` запрещает дальнейшее переопределение в следующем поколении наследников. Это полезно по двум причинам: во-первых, вы фиксируете поведение, которое не должно меняться (например, безопасность или инвариант), во-вторых, компилятор и JIT могут применять оптимизации вроде девиртуализации (devirtualization) и встраивания (inlining), что ускоряет вызов. Помечать `sealed` стоит тогда, когда вы уверены: никто из наследников не должен менять логику метода, а производительность важна.

Разница `override` vs `new` — главный вопрос на собеседованиях. Запомните правило: `override` меняет поведение для всех вызывающих, которые смотрят через базовый тип; `new` создаёт «параллельный» метод, видимый только когда переменная имеет тип производного класса. Если сомневаетесь — почти всегда нужен `override`. `new` оправдан редко: например, когда вы не контролируете базовый класс (он не `virtual`), но хотите расширить API, не ломая существующее поведение.

Полиморфизм через `virtual`/`override` позволяет писать гибкий код: коллекция `List<Employee>` может содержать разных сотрудников, и один цикл `foreach (var e in employees) e.Work();` вызовет корректную реализацию для каждого. Это и есть сила виртуальных методов — один интерфейс, множество форм.

#### Theory (EN)

Polymorphism is one of the three pillars of object-oriented programming, alongside encapsulation and inheritance. In C# it most commonly shows up through virtual methods. The idea is simple: a base class marks a method with the `virtual` modifier, which says “I have a default implementation, but a derived class is allowed to provide its own, more specific one.” When you call such a method through a reference typed as the base class, the runtime looks not at the reference type, but at the actual type of the object in memory, and picks the right version. This mechanism is called dynamic dispatch.

An analogy: think of a job description. The base class `Employee` has a method `Work()` describing general duties. The class `Developer` inherits from `Employee` and overrides `Work()` with the `override` keyword, because a programmer works differently. If a manager addresses the person as an `Employee` (seeing only the generic description) but the actual object is a `Developer`, the programmer’s version of the method runs. The `override` modifier is an explicit declaration: “I am replacing the parent implementation while preserving the contract.”

For an override to be possible, the parent method must be marked `virtual`, `abstract`, or already be an `override` (in the last two cases overriding is allowed implicitly). You cannot just slap `override` on a regular method — the compiler will complain. This is a safeguard against accidental replacement: you must consciously permit overriding in the base class.

Now consider the `new` keyword. If, in a derived class, you declare a method with the same name and signature but without `override`, the compiler emits a warning: “method hides a member from the base class.” When you genuinely intend to hide the parent method rather than override it, you write `new` explicitly. The difference matters. `override` is polymorphism: the version chosen depends on the actual runtime type of the object. `new` is hiding: the version chosen depends on the compile-time type of the reference. If a `Developer` object is held in a variable of type `Employee`, the parent `Work()` is called, not the “new” one. This is a frequent source of subtle bugs.

The `sealed` keyword combined with `override` forbids further overriding in the next generation of descendants. This is useful for two reasons. First, you lock behavior that must not change (a security invariant, for instance). Second, the compiler and JIT can apply optimizations such as devirtualization and inlining, which speed up calls. Marking `sealed` is appropriate when you are certain no descendant should change the method’s logic and performance matters.

The `override` vs `new` distinction is the classic interview question. Remember the rule: `override` changes behavior for every caller that sees the object through the base type; `new` creates a “parallel” method visible only when the variable is typed as the derived class. When in doubt, almost always choose `override`. `new` is rarely justified — for example, when you do not control the base class (it is not `virtual`) but want to extend the API without breaking existing behavior.

Polymorphism through `virtual`/`override` lets you write flexible code. A `List<Employee>` can hold many kinds of employees, and a single loop `foreach (var e in employees) e.Work();` invokes the correct implementation for each. That is the power of virtual methods — one interface, many forms.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — virtual / override / new / sealed
using System;
using System.Collections.Generic;

// Базовый класс с виртуальным методом / Base class with a virtual method
public class Employee
{
    public string Name { get; init; } = string.Empty;

    // virtual: разрешаем переопределение / virtual: allow overriding
    public virtual string Work() => $"{Name} выполняет общие задачи. / performs general tasks.";

    // Невиртуальный метод / Non-virtual method
    public string Report() => $"{Name}: отчёт по умолчанию. / default report.";
}

// Производный класс с override / Derived class with override
public class Developer : Employee
{
    public string Stack { get; init; } = "C#";

    // override: полиморфная замена / override: polymorphic replacement
    public override string Work() => $"{Name} пишет код на {Stack}. / writes code in {Stack}.";

    // sealed override: запрещаем дальнейшее переопределение
    // sealed override: forbid further overriding
    public sealed override string ToString() => $"Developer[{Name}, {Stack}]";
}

// Ещё один наследник / Another descendant
public class Manager : Employee
{
    public int TeamSize { get; init; }

    public override string Work() => $"{Name} управляет командой из {TeamSize} человек. / manages a team of {TeamSize}.";
}

// Демонстрация new (скрытие метода) / Demonstrating new (method hiding)
public class LegacyEmployee : Employee
{
    // new: скрываем, но НЕ переопределяем / new: hide, do NOT override
    public new string Report() => $"{Name}: устаревший отчёт. / legacy report.";
}

public static class Demo
{
    public static void Run()
    {
        var employees = new List<Employee>
        {
            new Employee { Name = "Иван" },
            new Developer { Name = "Анна", Stack = "F#" },
            new Manager  { Name = "Олег", TeamSize = 5 }
        };

        // Полиморфный вызов: версия выбирается по фактическому типу
        // Polymorphic call: version chosen by actual runtime type
        foreach (var e in employees)
            Console.WriteLine(e.Work());

        // override vs new — ключевая разница / override vs new — the key difference
        Employee legacy = new LegacyEmployee { Name = "Пётр" };

        // Report() НЕ виртуальный, а в LegacyEmployee скрыт через new.
        // Поэтому вызывается версия базового класса — по типу ссылки.
        // Report() is non-virtual and hidden via new in LegacyEmployee.
        // So the base class version runs — chosen by the reference type.
        Console.WriteLine(legacy.Report());            // default report
        Console.WriteLine(((LegacyEmployee)legacy).Report()); // legacy report

        // sealed: нельзя унаследовать и переопределить ToString дальше
        // sealed: further overriding of ToString is forbidden
        var dev = new Developer { Name = "Анна", Stack = "F#" };
        Console.WriteLine(dev.ToString());
    }
}
```

#### Best Practices

- Помечайте метод `virtual` только если действительно планируете переопределение; иначе класс становится хрупким. / Mark a method `virtual` only when overriding is genuinely intended; otherwise the class becomes fragile.
- Предпочитайте `override` вместо `new`; используйте `new` лишь когда не контролируете базовый класс. / Prefer `override` over `new`; use `new` only when you do not control the base class.
- Используйте `sealed override`, чтобы зафиксировать поведение и помочь JIT-оптимизациям. / Use `sealed override` to lock behavior and help JIT optimizations.
- Не переопределяйте методы так, чтобы нарушать контракт базового класса (правило Лисков). / Do not override methods in a way that breaks the base class contract (Liskov principle).
- В переопределённом методе вызывайте `base.Method()`, если нужно сохранить родительскую логику. / Call `base.Method()` in an override when the parent logic should be preserved.

#### Частые ошибки / Common Mistakes

- [RU] Забыли `virtual` в базовом классе → компилятор не даст `override`. **Как избежать:** сначала пометьте базовый метод `virtual`/`abstract`, затем `override` в наследнике.
- [RU] Использовали `new` вместо `override` → полиморфизм не работает, вызывается родительская версия. **Как избежать:** проверяйте, что нужно именно переопределение, а не сокрытие; почти всегда нужен `override`.
- [RU] Переопределение меняет постусловия базового метода → нарушается подстановка Лисков. **Как избежать:** сохраняйте контракт и инварианты базового класса.
- [RU] Забыли `base.` и потеряли важную логику родителя. **Как избежать:** явно решайте, нужен ли вызов `base.Method()`, и документируйте это.
- [EN] Forgot `virtual` in the base class → the compiler rejects `override`. **How to avoid:** mark the base method `virtual`/`abstract` first, then `override` in the derived class.
- [EN] Used `new` instead of `override` → polymorphism breaks, the base version runs. **How to avoid:** decide whether you need overriding, not hiding; `override` is almost always correct.
- [EN] Override changes the postconditions of the base method → violates Liskov substitution. **How to avoid:** preserve the contract and invariants of the base class.
- [EN] Forgot `base.` and lost important parent logic. **How to avoid:** decide explicitly whether `base.Method()` is needed and document it.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Базовый метод помечен `virtual` (или `abstract`) перед использованием `override`. (RU)
- [ ] Я выбрал `override` там, где нужен полиморфизм, и `new` только для сокрытия. (RU)
- [ ] Переопределённый метод сохраняет контракт базового класса (правило Лисков). (RU)
- [ ] `sealed override` использован там, где поведение фиксируется намеренно. (RU)
- [ ] Вызов через базовую ссылку выбирает правильную версию (проверено тестом). (RU)
- [ ] The base method is marked `virtual` (or `abstract`) before using `override`. (EN)
- [ ] I chose `override` where polymorphism is needed and `new` only for hiding. (EN)
- [ ] The overridden method preserves the base class contract (Liskov rule). (EN)
- [ ] `sealed override` is used where behavior is intentionally locked. (EN)
- [ ] A call through a base reference selects the correct version (verified by a test). (EN)

#### Ресурсы / Resources

- [Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/virtual](https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/virtual)
- [Microsoft Learn — override (C# Reference) — https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/override](https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/override)
- [Microsoft Learn — new Modifier — https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/new-modifier](https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/new-modifier)
- [Microsoft Learn — sealed (C# Reference) — https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/sealed](https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/sealed)
- [Microsoft Learn — Polymorphism — https://learn.microsoft.com/dotnet/csharp/fundamentals/object-oriented/polymorphism](https://learn.microsoft.com/dotnet/csharp/fundamentals/object-oriented/polymorphism)

---

[⬆ К модулю M05](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
