[⬆ К модулю M05](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M05-L01: Наследование, base, конструкторы базового класса / Inheritance, base, base constructors

**Модуль / Module:** M05
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Наследование — это один из четырёх китов объектно-ориентированного программирования (вместе с инкапсуляцией, полиморфизмом и абстракцией). В C# наследование позволяет создать новый класс, который автоматически получает все члены существующего класса: поля, свойства, методы, события. Существующий класс называют **базовым** (base class), а новый — **производным** (derived class). Представьте себе чертёж автомобиля: базовый класс «Автомобиль» описывает колёса, двигатель и руль. Производный класс «Электромобиль» берёт всё это готовое и добавляет батарею и электромотор — не нужно пересоздавать то, что уже есть.

Важное ограничение C#: класс может наследовать только **один** другой класс. Это называется одиночным наследованием (single inheritance). В отличие от C++, где допустимо множественное наследование классов, в C# от него отказались ради простоты и предсказуемости — слишком много проблем возникало с конфликтами имён и порядком инициализации. Зато интерфейсы можно реализовать сколько угодно, и это покрывает большинство сценариев.

Любой класс в C# неявно наследуется от `System.Object` (синоним — ключевое слово `object`). Это значит, что каждый объект в .NET гарантированно имеет методы `ToString()`, `Equals()`, `GetHashCode()` и `GetType()`. Даже если вы напишете `class Foo {}`, компилятор «под капотом» сделает `class Foo : object`. Так `object` служит единым корнем всей иерархии типов — благодаря ему любой объект можно сохранить в переменную типа `object` или поместить в `List<object>`.

Синтаксис наследования прост: после имени производного класса ставится двоеточие и имя базового: `class Dog : Animal`. Члены базового класса, объявленные как `public` или `protected`, становятся доступны в производном. `private` члены тоже физически существуют в производном объекте, но напрямую обратиться к ним нельзя — только через открытые или защищённые методы базы. Методы базового класса можно **переопределить** (override), если они помечены `virtual` или `abstract`, а можно **скрыть** (new) — но второе часто приводит к трудноуловимым багам и его лучше избегать.

Ключевое слово `base` — это «ручка» к базовому классу из производного. Оно решает две задачи. Во-первых, через `base.MethodName()` можно вызвать версию метода, определённую в базе, даже если в производном классе этот метод переопределён. Во-вторых, и это главное — `base(...)` используется для явного вызова **конструктора базового класса** из конструктора производного.

Конструкторы **не наследуются**. Поэтому, когда вы создаёте объект производного класса, сначала полностью отрабатывает конструктор базового, и только потом — конструктор производного. Порядок строго от корня (`object`) вниз до самого производного. Если базовый класс имеет конструктор без параметров, компилятор вызовет его автоматически. Но если базовый класс требует аргументы (например, `public Animal(string name)`), производный класс **обязан** явно передать их через `base(name)`:

```csharp
class Animal
{
    public Animal(string name) { /* ... */ }
}
class Dog : Animal
{
    public Dog(string name, string breed) : base(name) { /* ... */ }
}
```

Аналогия: чтобы построить второй этаж (производный класс), сначала нужно построить первый (базовый). Нельзя начать со второго этажа в воздухе. Строка `: base(name)` — это указание прорабу: «возьми такой-то материал и передай его первому этажу». Если не указать `base(...)`, компилятор попытается вызвать конструктор без параметров — и если такого нет, выдаст ошибку компиляции.

Подведём: наследование переиспользует код и моделирует отношение «is-a» (собака — это животное); одиночное наследование и корень `object` упрощают модель; `base` позволяет обращаться к родителю; конструкторы вызываются от базы к производному, и аргументы базового конструктора передаются явно через `base(...)`.

#### Theory (EN)

Inheritance is one of the four pillars of object-oriented programming, alongside encapsulation, polymorphism, and abstraction. In C#, inheritance lets you create a new class that automatically acquires every member of an existing class: fields, properties, methods, events. The existing class is called the **base class**, the new one the **derived class**. Imagine a blueprint for a car: the base class `Car` already describes wheels, an engine, and a steering wheel. The derived class `ElectricCar` takes all of that for free and adds a battery and an electric motor — no need to rebuild what already exists.

An important C# constraint: a class may inherit from **only one** other class. This is called single inheritance. Unlike C++, which allows multiple class inheritance, C# deliberately avoids it for simplicity and predictability — multiple inheritance historically caused name conflicts and ambiguous initialization order. In exchange, C# lets you implement any number of interfaces, which covers most real-world design needs.

Every class in C# implicitly inherits from `System.Object` (the keyword alias is `object`). This means every .NET object is guaranteed to have `ToString()`, `Equals()`, `GetHashCode()`, and `GetType()`. Even `class Foo {}` is compiled as `class Foo : object`. So `object` is the single root of the whole type hierarchy — thanks to it, any value can be stored in an `object` variable or added to a `List<object>`.

The inheritance syntax is simple: place a colon and the base name after the derived class name: `class Dog : Animal`. Members of the base class declared as `public` or `protected` become reachable from the derived class. `private` members still physically exist inside the derived object, but you cannot touch them directly — only through public or protected methods of the base. Base methods can be **overridden** (with `override`) if they are marked `virtual` or `abstract`, or **hidden** (with `new`) — but the latter often causes subtle bugs and is best avoided.

The `base` keyword is your "handle" to the base class from inside the derived one. It does two jobs. First, `base.MethodName()` calls the version of a method defined in the base, even when the derived class has overridden it. Second — and most important — `base(...)` is used to explicitly invoke a **base class constructor** from a derived constructor.

Constructors are **not inherited**. So when you instantiate a derived object, the base constructor runs to completion first, and only then the derived constructor. The order strictly goes from the root (`object`) down to the most derived type. If the base class has a parameterless constructor, the compiler calls it automatically. But if the base class requires arguments (for example, `public Animal(string name)`), the derived class **must** pass them explicitly via `base(name)`:

```csharp
class Animal
{
    public Animal(string name) { /* ... */ }
}
class Dog : Animal
{
    public Dog(string name, string breed) : base(name) { /* ... */ }
}
```

Analogy: to build the second floor (the derived class), you must first build the ground floor (the base). You cannot start the second floor in mid-air. The line `: base(name)` tells the builder: "take this material and hand it to the ground floor." If you omit `base(...)`, the compiler tries to call a parameterless constructor — and if none exists, you get a compile-time error.

In summary: inheritance reuses code and models the "is-a" relationship (a dog is an animal); single inheritance and the `object` root keep the model simple; `base` reaches up to the parent; constructors run from base to derived, and base-constructor arguments are passed explicitly via `base(...)`.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8
// Демонстрация наследования, ключевого слова base и вызова конструктора базы.
// Demo of inheritance, the base keyword, and calling a base constructor.

using System;

// Базовый класс / Base class
public class Animal
{
    public string Name { get; }

    // Конструктор базового класса требует аргумент / Base ctor requires an argument
    public Animal(string name)
    {
        if (string.IsNullOrWhiteSpace(name))
            throw new ArgumentException("Имя не может быть пустым / Name cannot be empty", nameof(name));

        Name = name;
        Console.WriteLine($"  Animal.ctor: создан/created '{Name}'");
    }

    // Виртуальный метод можно переопределить / Virtual method can be overridden
    public virtual string Describe() => $"Animal: {Name}";

    // Метод базового класса, доступный производному / Base method reachable from derived
    public override string ToString() => Name;
}

// Производный класс — одиночное наследование / Derived class — single inheritance
public class Dog : Animal
{
    public string Breed { get; }

    // Явный вызов конструктора базы через base(name) / Explicit base ctor call via base(name)
    public Dog(string name, string breed) : base(name)
    {
        Breed = breed;
        Console.WriteLine($"  Dog.ctor: порода/breed '{Breed}'");
    }

    // Переопределение виртуального метода / Override of the virtual method
    public override string Describe() => $"Dog: {Name} ({Breed})";

    public void Bark() => Console.WriteLine($"{Name}: Гав! / Woof!");
}

internal static class Demo
{
    public static void Run()
    {
        // Сначала отрабатывает Animal.ctor, затем Dog.ctor / Animal.ctor runs first, then Dog.ctor
        var rex = new Dog("Rex", "Немецкая овчарка / German Shepherd");
        Console.WriteLine(rex.Describe());
        rex.Bark();

        // Upcast: Dog можно трактовать как Animal (и как object) / Upcast works because of inheritance
        Animal asAnimal = rex;
        object asObject = rex;          // object — корень иерархии / object is the hierarchy root
        Console.WriteLine($"asAnimal.Describe(): {asAnimal.Describe()}");
        Console.WriteLine($"asObject.GetType(): {asObject.GetType().Name}");
    }
}
```

#### Best Practices

- Предпочитайте наследование для отношения «is-a» (собака — это животное), а композицию — для «has-a» (автомобиль имеет двигатель). Композиция гибче и её проще тестировать.
- Помечайте методы `virtual` только если действительно ожидаете переопределение; иначе запечатывайте классы и методы `sealed`, чтобы защитить инварианты.
- Всегда передавайте обязательные данные базовому классу через `base(...)`, а не пытайтесь обойти инициализацию.
- Prefer inheritance for an "is-a" relationship (a dog is an animal) and composition for "has-a" (a car has an engine). Composition is more flexible and easier to test.
- Mark methods `virtual` only when you genuinely expect overrides; otherwise seal classes and methods with `sealed` to protect invariants.
- Always pass required data to the base class through `base(...)`, never try to skip initialization.

#### Частые ошибки / Common Mistakes

- [Забыли `: base(...)` при базовом конструкторе с параметрами] → [Компилятор ищет конструктор без параметров и выдаёт ошибку CS1729; явно укажите `base(...)` с нужными аргументами.]
- [Использование `new` для скрытия метода вместо `override`] → [Поведение зависит от статического типа переменной, возникают скрытые баги; пометьте метод базы `virtual` и используйте `override`.]
- [Вызов виртуального метода из конструктора базы] → [В этот момент производный класс ещё не инициализирован, переопределение сработает на «полуготовом» объекте; избегайте вызовов виртуальных членов в конструкторах.]
- [Попытка множественного наследования классов `class A : B, C`] → [В C# запрещено; используйте интерфейсы для нескольких контрактов.]
- [Forgetting `: base(...)` when the base ctor has parameters] → [The compiler looks for a parameterless ctor and raises CS1729; explicitly add `base(...)` with the required arguments.]
- [Using `new` to hide a method instead of `override`] → [Behavior depends on the static variable type, causing subtle bugs; mark the base method `virtual` and use `override`.]
- [Calling a virtual method from the base constructor] → [The derived class is not initialized yet, so the override runs on a half-built object; avoid virtual member calls in constructors.]
- [Trying multiple class inheritance `class A : B, C`] → [Not allowed in C#; use interfaces for multiple contracts.]

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Я могу объяснить, почему в C# одиночное наследование и чем оно отличается от интерфейсов.
- [ ] Я знаю, что `object` — корень иерархии, и могу назвать его методы.
- [ ] Я умею писать конструктор производного класса с `: base(...)` и понимаю порядок вызова конструкторов.
- [ ] Я различаю `override` и `new` и знаю, почему `new` опасен.
- [ ] Я не вызываю виртуальные методы из конструкторов.
- [ ] I can explain why C# uses single inheritance and how it differs from interfaces.
- [ ] I know that `object` is the hierarchy root and can list its methods.
- [ ] I can write a derived constructor with `: base(...)` and explain the constructor call order.
- [ ] I distinguish `override` from `new` and understand why `new` is risky.
- [ ] I avoid calling virtual methods from constructors.

#### Ресурсы / Resources

- [Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/fundamentals/object-oriented/inheritance](https://learn.microsoft.com/dotnet/csharp/fundamentals/object-oriented/inheritance)

---

[⬆ К модулю M05](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
