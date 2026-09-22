[⬆ К модулю M05](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M05-L04: Интерфейсы, default interface methods, множественная реализация / Interfaces, default interface methods, multiple implementation

**Модуль / Module:** M05
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Интерфейс в C# — это контракт, который описывает **что** класс должен делать, но не **как**. Представьте себе электрическую розетку: неважно, кто произвёл чайник или утюг — если их вилки соответствуют стандарту розетки, они заработают. Интерфейс — это «стандарт розетки», а класс — это «прибор», который обязуется иметь совместимую «вилку» в виде реализованных методов и свойств.

Объявляется интерфейс ключевым словом `interface`. По соглашению его имя начинается с `I` (`IComparable`, `IEnumerable`, `IDisposable`). До C# 8 интерфейс мог содержать только сигнатуры методов, свойств, индексаторов и событий — без тел. Начиная с C# 8 появилась возможность задавать методы по умолчанию (default interface methods, DIM), что изменило семантику интерфейсов, приблизив их к чертам абстрактных классов с множественным наследованием.

**Неявная реализация (implicit).** Класс просто объявляет метод с той же сигнатурой и модификатором `public`. Тогда метод доступен и через ссылку на класс, и через ссылку на интерфейс. Это обычный, наиболее частый путь.

**Явная реализация (explicit).** Метод объявляется без модификатора доступа, с указанием имени интерфейса: `void ILogger.Log(string msg)`. Такой метод недоступен через переменную типа класса — только через приведение к интерфейсу. Это удобно, когда у класса есть два интерфейса с одноимёнными методами (например, `IDraw.Draw` и `ICard.Draw`), или когда нужно «спрятать» вспомогательный член API.

**Множественная реализация.** В C# класс может наследоваться только от одного базового класса, но реализовывать сколько угодно интерфейсов. Это основной механизм полиморфизма «по контракту»: один объект может быть одновременно `IComparable`, `IEnumerable` и `IDisposable`. Если интерфейсы содержат одноимённые методы, неоднозначность решается явной реализацией каждого из них.

**Default interface methods (DIM, C# 8+).** Метод интерфейса может иметь тело и модификатор `virtual`. Тогда реализация необязательна: классы, которые не переопределяют метод, получают поведение по умолчанию. Это позволяет расширять интерфейс новыми членами, **не ломая** существующие реализации — главная мотивация функции. Важные нюансы:
- метод по умолчанию доступен только через переменную типа интерфейса, не через тип класса;
- класс не «наследует» DIM в классическом смысле — это поведение интерфейса, а не класса;
- интерфейсы могут переопределять DIM друг друга при множественном наследовании; неоднозначности разрешаются правилом «наиболее конкретный производный интерфейс выигрывает»;
- DIM не может обращаться к полям класса — только к членам интерфейса.

**Когда использовать интерфейсы, а когда абстрактные классы?** Интерфейс — для описания способности («умею сравнивать», «умею удаляться»). Абстрактный класс — когда есть общее состояние и базовая логика с конструктором. Класс может наследовать один абстрактный класс + несколько интерфейсов.

**Контравариантность и ковариантность.** Параметры-типы интерфейсов можно помечать `in` (контравариантность, для входных аргументов) и `out` (ковариантность, для возвращаемых значений). Это позволяет безопасно приводить `IEnumerable<string>` к `IEnumerable<object>` или `IComparer<object>` к `IComparer<string>`.

#### Theory (EN)

An interface in C# is a contract that describes **what** a type must do, not **how** it does it. Think of an electrical socket standard: it does not matter whether the appliance is a kettle, an iron, or a laptop charger — if its plug matches the socket, it works. The interface is the socket standard; the class is the appliance that promises to ship a compatible plug, in the form of implemented methods and properties.

An interface is declared with the `interface` keyword. By convention its name starts with `I` (`IComparable`, `IEnumerable`, `IDisposable`). Before C# 8, an interface could only contain signatures of methods, properties, indexers, and events — no bodies. Starting with C# 8 you can give methods default bodies (default interface methods, DIM), which changed the semantics of interfaces and brought them closer to abstract classes with a flavour of multiple inheritance.

**Implicit implementation.** The class simply declares a method with the same signature and a `public` modifier. The method is then reachable both through a class-typed reference and through an interface-typed reference. This is the common, default path.

**Explicit implementation.** The method is declared without an access modifier, prefixed by the interface name: `void ILogger.Log(string msg)`. Such a member is not visible through a class-typed variable — only through a cast to the interface. This is useful when a class implements two interfaces that share a method name (for example `IDraw.Draw` and `ICard.Draw`), or when you want to hide a helper member from the public surface.

**Multiple implementation.** A class in C# can inherit from only one base class, but it can implement any number of interfaces. This is the primary mechanism of contract-based polymorphism: one object can simultaneously be `IComparable`, `IEnumerable`, and `IDisposable`. When interfaces declare methods with the same name, the ambiguity is resolved through explicit implementation of each one.

**Default interface methods (DIM, C# 8+).** An interface method may have a body and be marked `virtual`. The implementation then becomes optional: classes that do not override it receive the default behaviour. This lets you add new members to an interface **without breaking** existing implementations — the main motivation for the feature. Key caveats:
- a default method is only callable through a variable of the interface type, not of the class type;
- the class does not “inherit” DIM in the classical sense — it is behaviour of the interface, not of the class;
- interfaces can override one another’s DIMs during multiple inheritance; ambiguities are resolved by the “most specific derived interface wins” rule;
- a DIM cannot touch instance fields of the class — only members of the interface.

**When to choose interfaces over abstract classes?** Use an interface to describe a capability (“I can be compared”, “I can be disposed”). Use an abstract class when there is shared state and base logic that needs a constructor. A class can derive from one abstract class plus several interfaces.

**Variance.** Interface type parameters can be marked `in` (contravariance, for input arguments) and `out` (covariance, for return values). This safely allows converting `IEnumerable<string>` to `IEnumerable<object>`, or `IComparer<object>` to `IComparer<string>`.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — Интерфейсы, DIM, явная/неявная реализация, множественная реализация
// Interfaces, DIM, explicit/implicit implementation, multiple implementation

using System;
using System.Collections.Generic;

// --- Интерфейсы с default-методами (C# 8+) / Interfaces with default methods ---
// ILogger расширяется новым методом, не ломая старые реализации.
// ILogger is extended with a new method without breaking old implementations.
public interface ILogger
{
    void Log(string message); // обязательный член / required member

    // Default interface method — доступен только через переменную интерфейса.
    // Default interface method — reachable only through an interface-typed variable.
    public void LogError(string message) => Log($"[ERROR] {message}");

    public void LogWarning(string message) => Log($"[WARN]  {message}");
}

// Второй интерфейс с одноимённым методом Log / Second interface with same-named Log
public interface IAuditable
{
    void Log(string message); // другой «контракт» Log / a different Log contract
    DateTime LastChanged { get; }
}

// --- Класс реализует ДВА интерфейса / Class implements TWO interfaces ---
public class FileLogger : ILogger, IAuditable
{
    private readonly string _path;
    public DateTime LastChanged { get; private set; }

    public FileLogger(string path) => _path = path;

    // Неявная реализация ILogger.Log — доступна через класс и через интерфейс.
    // Implicit implementation of ILogger.Log — visible via class and interface.
    public void Log(string message)
    {
        System.IO.File.AppendAllText(_path, message + Environment.NewLine);
        LastChanged = DateTime.UtcNow;
    }

    // Явная реализация IAuditable.Log — доступна только через (IAuditable).
    // Explicit implementation of IAuditable.Log — visible only via (IAuditable).
    void IAuditable.Log(string message)
        => Log($"[AUDIT] {message} @ {DateTime.UtcNow:O}");
}

// --- Класс, который НЕ переопределяет DIM, и использует поведение по умолчанию ---
// A class that does NOT override DIM and relies on default behaviour.
public class ConsoleLogger : ILogger
{
    public void Log(string message) => Console.WriteLine(message);
    // LogError / LogWarning берутся из интерфейса / come from the interface.
}

// --- Ковариантность интерфейса / Interface covariance ---
public interface IProcessor<out T> // out — ковариантность / covariance
{
    T Produce();
}

public class StringProcessor : IProcessor<string>
{
    public string Produce() => "hello";
}

internal static class Demo
{
    public static void Run()
    {
        var fileLogger = new FileLogger("log.txt");

        // Неявный член доступен через класс / Implicit member visible via class
        fileLogger.Log("Старт приложения / App started");

        // DIM доступен только через интерфейс / DIM reachable only via interface
        ((ILogger)fileLogger).LogError("Диск переполнен / Disk full");
        ((ILogger)fileLogger).LogWarning("Мало места / Low space");

        // Явная реализация доступна только через IAuditable
        // Explicit member reachable only via IAuditable
        ((IAuditable)fileLogger).Log("Документ подписан / Document signed");

        ConsoleLogger console = new();
        console.Log("Сообщение / Message");
        // DIM через переменную интерфейса / DIM through interface variable
        ILogger il = console;
        il.LogError("Ошибка / Error");

        // Ковариантность: IProcessor<string> -> IProcessor<object>
        // Covariance: IProcessor<string> -> IProcessor<object>
        IProcessor<string> sp = new StringProcessor();
        IProcessor<object> op = sp; // допустимо благодаря out / legal due to out
        Console.WriteLine(op.Produce());

        // Полиморфизм по контракту: один список — несколько реализаций
        // Contract-based polymorphism: one list, multiple implementations
        List<ILogger> loggers = new() { fileLogger, console };
        loggers.ForEach(l => l.Log("Широковещание / Broadcast"));
    }
}
```

#### Best Practices

- Один интерфейс = одна ответственность (ISP из SOLID): предпочитайте несколько маленьких интерфейсов одному большому «god-interface».
- Имена интерфейсов — с префикса `I`, чтобы визуально отличать их от классов в IntelliSense и в коде.
- Не добавляйте в интерфейс члены «на всякий случай»; каждый член — это обязательство для всех реализаций.
- Используйте `default interface methods` для **дополнения** существующего интерфейса, а не как замену абстрактному классу с состоянием.
- При множественной реализации с конфликтами имён применяйте явную реализацию — это честнее, чем «перекрывать» один метод другим.
- Если член интерфейса нужен только внутри API и не должен светиться в публичной поверхности класса, делайте его явным.

- One interface = one responsibility (ISP from SOLID): prefer several small interfaces over one large “god-interface”.
- Name interfaces with the `I` prefix to visually distinguish them from classes in IntelliSense and in code.
- Do not add members to an interface “just in case”; every member is a commitment for all implementations.
- Use `default interface methods` to **augment** an existing interface, not as a replacement for a stateful abstract class.
- When multiple implementation causes name conflicts, use explicit implementation — it is more honest than silently “shadowing” one method with another.
- If an interface member is only needed internally and should not appear on the class’s public surface, make it explicit.

#### Частые ошибки / Common Mistakes

- [Вызов default-метода через переменную типа класса] → [Компилятор его не видит; используйте переменную типа интерфейса или приведение `(IInterface)obj`.] (RU)
- [Попытка обратиться к полям класса из DIM] → [DIM не имеет доступа к полям класса; вынесите логику в обычный метод класса или добавьте абстрактный член в интерфейс.] (RU)
- [Два интерфейса с методом `void Draw()` реализованы неявно одним методом] → [Поведение случайно сливается; реализуйте каждый явно через `IDraw.Draw` и `ICard.Draw`.] (RU)
- [Явно реализованный член помечен `public`] → [Компилятор запретит; явная реализация не имеет модификатора доступа.] (RU)
- [Попытка `new()` экземпляра интерфейса] → [Интерфейс нельзя инстанцировать; создайте объект реализующего класса и присвойте его переменной интерфейса.] (RU)
- [Изменение интерфейса новым абстрактным методом ломает все реализации] → [Добавьте метод как DIM, чтобы сохранить обратную совместимость.] (RU)

- [Calling a default method through a class-typed variable] → [The compiler does not see it; use an interface-typed variable or a cast `(IInterface)obj`.] (EN)
- [Trying to access class fields from a DIM] → [A DIM has no access to class fields; move the logic into a normal class method or add an abstract member to the interface.] (EN)
- [Two interfaces with `void Draw()` implemented implicitly by a single method] → [Behaviour is silently merged; implement each one explicitly via `IDraw.Draw` and `ICard.Draw`.] (EN)
- [An explicitly implemented member is marked `public`] → [The compiler forbids it; explicit implementation has no access modifier.] (EN)
- [Trying to `new()` an interface instance] → [An interface cannot be instantiated; create an instance of an implementing class and assign it to an interface variable.] (EN)
- [Adding a new abstract method to an interface breaks all implementations] → [Add the method as a DIM to preserve backward compatibility.] (EN)

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Я объясняю разницу между интерфейсом и абстрактным классом одним предложением.
- [ ] Я знаю, когда применять явную реализацию, и могу привести два сценария.
- [ ] Я понимаю, что default interface method доступен только через тип интерфейса.
- [ ] Я могу безопасно расширить существующий интерфейс новым методом, не ломая чужой код.
- [ ] Я различаю ковариантность (`out`) и контравариантность (`in`) и знаю, где они применяются.
- [ ] Я могу реализовать классом два интерфейса с одноимёнными методами без конфликтов.

- [ ] I can explain the difference between an interface and an abstract class in one sentence.
- [ ] I know when to use explicit implementation and can name two scenarios.
- [ ] I understand that a default interface method is only reachable via the interface type.
- [ ] I can safely extend an existing interface with a new method without breaking other code.
- [ ] I distinguish covariance (`out`) from contravariance (`in`) and know where each applies.
- [ ] I can implement two interfaces with same-named methods in one class without conflicts.

#### Ресурсы / Resources
- [Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/fundamentals/types/interfaces]

---

[⬆ К модулю M05](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
