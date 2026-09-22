[⬆ К модулю M04](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M04-L01: Класс vs объект, объявление класса / Class vs object, declaring a class

**Модуль / Module:** M04
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

В объектно-ориентированном программировании (ООП) два понятия — **класс** и **объект** — являются фундаментом, на котором строится всё остальное. Понимание разницы между ними критически важно: без этого невозможно писать код на C#, Java, Python или любом другом ООП-языке.

**Класс — это чертёж.** Представьте, что вы архитектор и нарисовали подробный план дома: сколько комнат, где дверь, какие трубы и провода. Этот чертёж сам по себе не является домом — в нём нельзя жить. Но по этому чертежу можно построить сколько угодно реальных домов. Точно так же **класс** описывает структуру некоторого понятия: какие данные оно хранит и какие действия с ними выполняет, но сам по себе данных не содержит.

**Объект — это построенный дом.** Когда вы «строите» дом по чертежу, вы получаете конкретный экземпляр: с реальным адресом, реальными стенами, в котором уже можно жить. В программировании это называется **экземпляр (instance)** класса. Один класс может породить множество объектов — как один чертёж порождает целый квартал одинаковых по структуре, но разных по наполнению домов.

Класс объединяет **данные** и **поведение**. Данные хранятся в **полях** (fields) и **свойствах** (properties), а поведение описывается **методами** (methods). Например, класс `House` может иметь поле `Address` (адрес) и метод `OpenDoor()` (открыть дверь). Каждый объект-дом будет иметь свой собственный адрес, но метод «открыть дверь» общий для всех — это инструкция, а не уникальное состояние.

Объявление класса в C# начинается с ключевого слова `class`, за которым следует имя класса (в PascalCase — с заглавной буквы). Тело класса заключается в фигурные скобки. Внутри описываются поля, свойства, методы и конструкторы. Например:

```csharp
class House
{
    public string Address;
    public int Floors;

    public void OpenDoor()
    {
        Console.WriteLine($"Дверь открыта по адресу {Address}");
    }
}
```

Чтобы создать объект (экземпляр), используется оператор `new`. Он выделяет память под объект и инициализирует его. После `new` можно сразу задать значения свойствам через объектный инициализатор:

```csharp
var myHouse = new House { Address = "ул. Ленина, 1", Floors = 2 };
myHouse.OpenDoor();
```

Здесь `myHouse` — переменная, ссылающаяся на конкретный объект. Можно создать ещё один объект `neighborHouse` с другим адресом — оба независимы, но построены по одному чертежу `House`.

Важно различать **статические** и **экземплярные** члены класса. Экземплярные поля и методы принадлежат конкретному объекту. Статические (помеченные `static`) принадлежат самому классу и доступны без создания объекта. Например, счётчик построенных домов логично сделать статическим — он относится ко всему классу, а не к одному дому.

Ключевая идея урока: **класс — это шаблон, объект — его конкретное воплощение**. Класс существует «на бумаге» (в коде), а объекты живут в памяти программы во время выполнения. Каждый объект имеет собственное состояние (значения полей), но разделяет с другими объектами того же класса общее поведение (методы).

Помните о соглашениях об именовании: классы — `PascalCase`, публичные поля и методы — `PascalCase`, локальные переменные и параметры — `camelCase`. Это принятый в C# стиль, который делает код читаемым и предсказуемым для всей команды.

#### Theory (EN)

In object-oriented programming (OOP), two concepts — **class** and **object** — form the foundation on which everything else is built. Understanding the difference between them is critical: without it, you cannot write code in C#, Java, Python, or any other OOP language.

**A class is a blueprint.** Imagine you are an architect and you draw a detailed house plan: how many rooms, where the door is, which pipes and wires run where. The blueprint itself is not a house — you cannot live in it. But from that blueprint you can build as many real houses as you like. In the same way, a **class** describes the structure of some concept: what data it holds and what actions it performs on that data, but it holds no data of its own.

**An object is the built house.** When you “build” a house from a blueprint, you get a concrete instance: with a real address, real walls, where you can actually live. In programming this is called an **instance** of a class. A single class can spawn many objects — just as one blueprint can spawn a whole block of houses, identical in structure but different in contents.

A class combines **data** and **behavior**. Data is stored in **fields** and **properties**, while behavior is described by **methods**. For example, a `House` class may have a field `Address` and a method `OpenDoor()`. Each house object will have its own address, but the “open door” method is shared by all — it is an instruction, not unique state.

Declaring a class in C# starts with the `class` keyword, followed by the class name (in PascalCase — starting with a capital letter). The class body is enclosed in curly braces. Inside, you describe fields, properties, methods, and constructors. For example:

```csharp
class House
{
    public string Address;
    public int Floors;

    public void OpenDoor()
    {
        Console.WriteLine($"Door opened at {Address}");
    }
}
```

To create an object (instance), you use the `new` operator. It allocates memory for the object and initializes it. After `new` you can immediately assign values to properties through an object initializer:

```csharp
var myHouse = new House { Address = "1 Main Street", Floors = 2 };
myHouse.OpenDoor();
```

Here `myHouse` is a variable that refers to a specific object. You can create another object, `neighborHouse`, with a different address — the two are independent but built from the same `House` blueprint.

It is important to distinguish **static** from **instance** members of a class. Instance fields and methods belong to a specific object. Static members (marked `static`) belong to the class itself and are accessible without creating an object. For example, a counter of built houses makes sense as static — it relates to the whole class, not to a single house.

The key idea of the lesson: **a class is a template, an object is its concrete incarnation**. A class exists “on paper” (in code), while objects live in program memory at runtime. Each object has its own state (field values), but shares common behavior (methods) with other objects of the same class.

Keep naming conventions in mind: classes use `PascalCase`, public fields and methods use `PascalCase`, local variables and parameters use `camelCase`. This is the accepted C# style that makes code readable and predictable for the whole team.

#### Пример кода / Code Example

```csharp
// C# 12+ / .NET 8+ — Класс как чертёж, объект как экземпляр
// Class as blueprint, object as instance

using System;

namespace M04.L01.Demo;

// Класс — это чертёж дома / A class is the house blueprint
public class House
{
    // Поля — данные каждого объекта / Fields — per-object data
    public string Address;   // Адрес / Address
    public int Floors;       // Этажность / Number of floors

    // Статическое поле принадлежит классу, а не объекту
    // A static field belongs to the class, not to an instance
    public static int TotalBuilt;

    // Конструктор — вызывается при создании объекта через new
    // Constructor — invoked when an object is created via new
    public House(string address, int floors)
    {
        Address = address;
        Floors = floors;
        TotalBuilt++; // Увеличиваем общий счётчик / Increment shared counter
    }

    // Метод — поведение, общее для всех объектов класса
    // Method — behavior shared by all instances of the class
    public void OpenDoor()
    {
        // У каждого объекта свой Address / Each object has its own Address
        Console.WriteLine($"Дверь открыта по адресу {Address} / Door opened at {Address}");
    }

    public void Describe()
    {
        Console.WriteLine($"Дом: {Address}, этажей: {Floors} / House: {Address}, floors: {Floors}");
    }
}

public static class Program
{
    public static void Main()
    {
        // Создаём два независимых объекта по одному чертежу
        // Create two independent objects from the same blueprint
        var myHouse = new House("ул. Ленина, 1", 2);
        var neighborHouse = new House("ул. Ленина, 2", 3);

        // Каждый объект имеет собственное состояние
        // Each object has its own state
        myHouse.Describe();           // Дом: ул. Ленина, 1, этажей: 2
        neighborHouse.Describe();     // Дом: ул. Ленина, 2, этажей: 3

        // Поведение общее, но работает с данными конкретного объекта
        // Behavior is shared, but operates on a specific object's data
        myHouse.OpenDoor();
        neighborHouse.OpenDoor();

        // Статическое поле Accessed через имя класса, а не объекта
        // A static field is accessed via the class name, not an instance
        Console.WriteLine($"Всего построено домов / Total houses built: {House.TotalBuilt}");
    }
}
```

#### Best Practices

- Используйте `PascalCase` для имён классов, публичных полей, свойств и методов; `camelCase` — для локальных переменных и параметров. Это общепринятый стиль C#.
- Предпочитайте свойства (properties) публичным полям: свойства позволяют добавить проверку или логику без изменения вызывающего кода.
- Имена классов должны быть существительными (`House`, `Customer`, `Order`), а методы — глаголами или глагольными фразами (`OpenDoor`, `CalculateTotal`).
- Один класс — одна ответственность (принцип SRP). Класс `House` должен описывать дом, а не, например, управлять базой данных.
- Группируйте связанные классы в пространства имён (`namespace`), чтобы избежать конфликтов имён и упростить навигацию по проекту.

- Use `PascalCase` for class names, public fields, properties, and methods; use `camelCase` for local variables and parameters — this is the accepted C# convention.
- Prefer properties over public fields: properties let you add validation or logic later without changing calling code.
- Class names should be nouns (`House`, `Customer`, `Order`); method names should be verbs or verb phrases (`OpenDoor`, `CalculateTotal`).
- One class — one responsibility (the SRP principle). A `House` class should describe a house, not, for example, manage a database.
- Group related classes into namespaces to avoid name collisions and make project navigation easier.

#### Частые ошибки / Common Mistakes

- **Обращение к экземплярному полю через имя класса** (`House.Address`) → обращайтесь к полям через переменную объекта (`myHouse.Address`); через имя класса доступны только `static`-члены.
- **Путаница между классом и объектом** («класс хранит адрес») → помните: класс — это чертёж, данные хранят объекты; у класса нет своего `Address`.
- **Забывают оператор `new` при создании объекта** (`House h;`) → без `new` переменная равна `null`, и обращение к её членам вызовет `NullReferenceException`.
- **Использование публичных полей вместо свойств** без необходимости → это ломает инкапсуляцию и усложняет будущие изменения; используйте свойства или приватные поля с методами доступа.
- **Имена классов в lowerCase** (`class house`) → нарушает соглашения C# и снижает читаемость; используйте `PascalCase` (`class House`).

- **Accessing an instance field via the class name** (`House.Address`) → access fields via an object variable (`myHouse.Address`); only `static` members are reachable through the class name.
- **Confusing class and object** (“the class stores the address”) → remember: a class is a blueprint, objects store data; the class has no `Address` of its own.
- **Forgetting the `new` operator when creating an object** (`House h;`) → without `new` the variable is `null`, and accessing its members throws `NullReferenceException`.
- **Using public fields instead of properties** without reason → this breaks encapsulation and makes future changes harder; use properties or private fields with accessor methods.
- **Lowercase class names** (`class house`) → violates C# conventions and hurts readability; use `PascalCase` (`class House`).

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Я могу своими словами объяснить разницу между классом и объектом.
- [ ] Я могу объявить класс с полями, конструктором и методом.
- [ ] Я могу создать объект через `new` и задать его поля.
- [ ] Я понимаю, что каждый объект имеет собственное состояние, а методы общие для класса.
- [ ] Я знаю разницу между статическими и экземплярными членами класса.
- [ ] Я соблюдаю соглашения об именовании C# (PascalCase / camelCase).

- [ ] I can explain the difference between a class and an object in my own words.
- [ ] I can declare a class with fields, a constructor, and a method.
- [ ] I can create an object using `new` and set its fields.
- [ ] I understand that each object has its own state, while methods are shared by the class.
- [ ] I know the difference between static and instance members of a class.
- [ ] I follow C# naming conventions (PascalCase / camelCase).

#### Ресурсы / Resources

- [Microsoft Learn — Object-oriented programming (C#)](https://learn.microsoft.com/dotnet/csharp/fundamentals/object-oriented/)
- [Microsoft Learn — Classes (C# Programming Guide)](https://learn.microsoft.com/dotnet/csharp/fundamentals/types/classes)
- [Microsoft Learn — Objects and classes](https://learn.microsoft.com/dotnet/csharp/fundamentals/object-oriented/objects)

---

[⬆ К модулю M04](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
