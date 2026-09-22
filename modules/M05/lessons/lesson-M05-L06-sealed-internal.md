[⬆ К модулю M05](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M05-L06: sealed, internal / sealed, internal

**Модуль / Module:** M05
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

В C# модификаторы `sealed` и `internal` управляют двумя разными, но одинаково важными аспектами типа: можно ли от него наследовать и откуда к нему можно обратиться. Вместе они формируют «периметр безопасности» вашего кода — границу, за которую никто не должен выходить без веской причины.

Начнём с `sealed`. Представьте, что вы спроектировали идеальный класс-фундамент: пусть это будет бетонный блок заводского размера. Вы точно знаете его прочность, поведение и все тесты, которые он прошёл. Если разрешить строить из него «надстройки», подрядчики могут изменить геометрию, и весь расчёт рассыпется. Модификатор `sealed` — это пломба: класс помечается как окончательный, и никто не сможет отнаследоваться от него. Компилятор просто запретит запись `class Derived : MySealedClass`.

Когда это полезно? Во-первых, для безопасности инвариантов. Класс `Money` с полями «сумма» и «валюта» гарантирует неизменяемость; наследник мог бы нарушить инвариант, добавив изменяемое состояние. Запечатав класс, вы защищаете контракт. Во-вторых, для производительности. JIT-компилятор .NET умеет «девиртуализировать» вызовы методов, если видит, что метод точно не переопределён. Для sealed-классов компилятор знает, что виртуальной диспетчеризации не будет, и может встроить метод (devirtualization + inlining). В горячих циклах это даёт ощутимый прирост. В-третьих, для публичного API библиотек: автор фреймворка оставляет себе свободу рефакторить внутренние классы, не боясь сломать чужие наследники.

Теперь про `sealed` для метода. Если базовый класс объявил метод как `virtual` (или `override`), наследник может его переопределить. Но в какой-то момент вы решаете: «дальше переопределять нельзя». Вы пишете `public sealed override void Draw()` — метод переопределён последний раз, и более глубокие наследники вынуждены использовать вашу реализацию. Это позволяет зафиксировать критичное поведение на одном уровне иерархии, не запечатывая весь класс.

Перейдём к `internal`. Этот модификатор ограничивает доступ текущей сборкой (assembly). Тип или член, помеченный `internal`, виден всему коду внутри одного `.dll`/`.exe`, но невидим внешним сборкам. Аналогия — внутренний двор компании: сотрудники ходят свободно, а посетители с улицы не попадают. По умолчанию классы в C# без явного модификатора и так получают `internal`, но хорошим тоном считается писать его явно для читаемости.

Когда применять `internal`? Для типов, реализующих публичный фасад библиотеки: вы хотите, чтобы несколько публичных классов совместно использовали вспомогательные сервисы, но не хотите выставлять их в публичный API. Это снижает поверхность атаки, упрощает документацию и даёт свободу менять внутреннюю структуру между версиями без semver-нарушений. Существует также связка `InternalsVisibleTo`, которая позволяет «открыть двор» конкретной дружественной сборке — например, модульному тест-проекту.

Как `sealed` и `internal` сочетаются? Часто их комбинируют: внутренний класс сервисной инфраструктуры делают `internal sealed` — он не наследуется снаружи и не виден за пределами сборки. Это самый строгий и при этом самый «свободный для рефакторинга» вариант: вы можете переименовать, разделить или удалить его в следующей версии, не ломая контракт. Главное правило — запечатывайте всё, что не предназначено для наследования, и держите внутренним всё, что не обязано быть публичным. Наследование и публичная поверхность должны быть осознанным дизайном, а не побочным эффектом.

#### Theory (EN)

In C#, the `sealed` and `internal` modifiers govern two different but equally important aspects of a type: whether it can be inherited from, and where it can be reached from. Together they form the “security perimeter” of your code — a boundary no one should cross without a deliberate reason.

Let us start with `sealed`. Imagine you have engineered an ideal foundational class: think of a factory-produced concrete block with a certified size and strength. You know exactly how it behaves and which tests it has passed. If you let contractors build “extensions” on top of it, they may change the geometry and the whole structural calculation falls apart. The `sealed` modifier is the tamper-evident seal: the class is marked final, and nobody can derive from it. The compiler simply refuses `class Derived : MySealedClass`.

When is this useful? First, to protect invariants. A `Money` class with fields “amount” and “currency” guarantees immutability; a subclass could break that invariant by adding mutable state. By sealing the class, you protect the contract. Second, for performance. The .NET JIT compiler can “devirtualize” method calls when it can prove the method is never overridden. For sealed classes, the compiler knows there will be no virtual dispatch, so it may inline the method (devirtualization + inlining). In hot loops this is a measurable win. Third, for public library APIs: the framework author keeps the freedom to refactor internal classes without breaking someone’s derived types.

Now consider `sealed` on a method. If a base class declared a method as `virtual` (or as `override`), a derived class can override it. But at some point you may decide “no further overrides are allowed.” You write `public sealed override void Draw()` — the method is overridden for the last time, and deeper descendants must use your implementation. This lets you freeze critical behavior at a single level of the hierarchy without sealing the entire class.

Let us turn to `internal`. This modifier restricts access to the current assembly. A type or member marked `internal` is visible to all code inside the same `.dll`/`.exe`, but invisible to external assemblies. The analogy is a company’s inner courtyard: employees move freely, while visitors from the street cannot enter. By default, classes in C# without an explicit modifier already get `internal`, but writing it explicitly is a good practice for readability.

When should you use `internal`? For types that implement the public façade of a library: you want several public classes to share helper services, but you do not want to expose those services in the public API. This shrinks the attack surface, simplifies documentation, and gives you freedom to reshape the internals between versions without semver violations. There is also `InternalsVisibleTo`, which “opens the courtyard” to a specific friendly assembly — for example, a unit-test project.

How do `sealed` and `internal` combine? They are often used together: an internal infrastructure-service class is declared `internal sealed` — it cannot be inherited from the outside and is invisible beyond the assembly. This is the strictest and, at the same time, the most refactor-friendly option: you can rename, split, or delete it in the next release without breaking any contract. The guiding rule is: seal everything that is not designed for inheritance, and keep internal everything that does not have to be public. Inheritance and the public surface should be the result of deliberate design, not an accidental side effect.

#### Пример кода / Code Example

```csharp
// C# 12+ / .NET 8+. sealed classes, sealed methods, internal accessibility.
// sealed-классы, sealed-методы, internal-доступность.

using System;

namespace M05L06;

// public sealed — нельзя унаследовать, виден всем / cannot be inherited, visible everywhere.
public sealed class Money
{
    public decimal Amount { get; }      // неизменяемое свойство / immutable property.
    public string Currency { get; }

    public Money(decimal amount, string currency)
    {
        if (amount < 0) throw new ArgumentOutOfRangeException(nameof(amount));
        Amount = amount;
        Currency = currency;
    }

    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException("Currency mismatch / несовпадение валют.");

        return new Money(Amount + other.Amount, Currency);
    }

    public override string ToString() => $"{Amount} {Currency}";
}

// Иерархия с sealed-методом / hierarchy with a sealed method.
public class Shape
{
    public virtual void Draw() => Console.WriteLine("Shape.Draw / фигура");
}

public class Circle : Shape
{
    // override + seal: последний раз переопределён / overridden for the last time.
    public sealed override void Draw() => Console.WriteLine("Circle.Draw / круг");
}

// class RedCircle : Circle { public override void Draw() {} }  // ОШИБКА CS0239: Draw запечатан / ERROR: Draw is sealed.

// internal — виден только в этой сборке / visible only inside this assembly.
internal sealed class ExchangeRateProvider
{
    private readonly decimal _rate;
    public ExchangeRateProvider(decimal rate) => _rate = rate;
    public decimal Convert(Money money, string targetCurrency) =>
        money.Currency == targetCurrency ? money.Amount : money.Amount * _rate;
}

// public-фасад, использующий internal-сервис / public façade using an internal service.
public static class MoneyService
{
    private static readonly ExchangeRateProvider UsdToEur = new(0.92m);

    public static Money ToEur(Money usd)
    {
        if (usd.Currency != "USD")
            throw new ArgumentException("Only USD supported / поддерживается только USD.");

        var eur = UsdToEur.Convert(usd, "EUR");
        return new Money(Math.Round(eur, 2), "EUR");
    }
}

public static class Demo
{
    public static void Run()
    {
        var price = new Money(100m, "USD");           // 100 USD
        var total = price.Add(new Money(25m, "USD")); // 125 USD
        var eur = MoneyService.ToEur(total);          // 115.00 EUR

        Console.WriteLine(total); // 125 USD
        Console.WriteLine(eur);   // 115.00 EUR

        var c = new Circle();
        c.Draw();                 // Circle.Draw / круг
    }
}
```

#### Best Practices

- Запечатывайте по умолчанию все классы, не предназначенные для наследования; оставляйте `virtual` только там, где расширение явно планируется.
- Используйте `internal` для вспомогательных типов и сервисов библиотеки, чтобы минимизировать публичную поверхность API.
- Комбинируйте `internal sealed` для внутренней инфраструктуры — это одновременно безопасно и удобно для рефакторинга.
- Применяйте `sealed override`, чтобы зафиксировать поведение метода на конкретном уровне иерархии, не запечатывая весь класс.
- Документируйте намерение: комментарий или XML-док должны объяснять, почему класс запечатан или почему член внутренний.
- Для unit-тестов используйте `InternalsVisibleTo`, а не делайте члены публичными ради тестируемости.

- Seal by default every class that is not designed for inheritance; keep methods `virtual` only where extension is explicitly planned.
- Use `internal` for helper types and library services to minimize the public API surface.
- Combine `internal sealed` for internal infrastructure — it is both safe and refactor-friendly.
- Apply `sealed override` to lock a method’s behavior at a specific hierarchy level without sealing the whole class.
- Document the intent: a comment or XML doc should explain why a class is sealed or why a member is internal.
- For unit tests, prefer `InternalsVisibleTo` over making members public just for testability.

#### Частые ошибки / Common Mistakes

- Запечатывание класса, который позже действительно нужно наследовать → сначала проектируй как незапечатанный, запечатывай только когда уверен в финальности / seal only when you are sure the class is final; design it unsealed first if extension is plausible.
- `sealed override` на методе, который базовый класс не объявил `virtual`/`override` → проверяй цепочку переопределений, `sealed` допустим только поверх `override` / `sealed` is valid only on an `override`; verify the override chain.
- Ожидание, что `internal` члены видны в другом проекте → помни, что граница `internal` — это сборка, а не пространство имён / remember `internal` boundary is the assembly, not the namespace.
- Доступ к `internal` типам из тестов через «public ради тестов» → используй `InternalsVisibleTo` вместо раздувания API / use `InternalsVisibleTo` instead of bloating the API for tests.
- Использование `sealed` ради «оптимизации» без измерений → сначала измерь бенчмарком, потом запечатывай только если выигрыш реален / measure with a benchmark first; seal only if the gain is real.
- Путаница `private` vs `internal` → `private` ограничивает одним типом, `internal` — одной сборкой / `private` restricts to one type, `internal` restricts to one assembly.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Я понимаю, что `sealed` запрещает наследование от класса.
- [ ] Я могу объяснить, чем `sealed override` отличается от `sealed class`.
- [ ] Я знаю, что `internal` ограничивает доступ текущей сборкой, а не пространством имён.
- [ ] Я могу назвать три причины использовать `sealed`: инварианты, производительность, стабильность API.
- [ ] Я умею открывать `internal` члены тест-проекту через `InternalsVisibleTo`.
- [ ] Я понимаю, что `internal sealed` — самая строгая и одновременно самая гибкая для рефакторинга комбинация.
- [ ] Я не запечатываю классы «на всякий случай», а делаю это осознанно.

- [ ] I understand that `sealed` forbids inheriting from the class.
- [ ] I can explain the difference between `sealed override` and `sealed class`.
- [ ] I know that `internal` restricts access to the current assembly, not to a namespace.
- [ ] I can name three reasons to use `sealed`: invariants, performance, API stability.
- [ ] I can expose `internal` members to a test project via `InternalsVisibleTo`.
- [ ] I understand that `internal sealed` is the strictest and yet the most refactor-friendly combination.
- [ ] I do not seal classes “just in case” — I do it deliberately.

#### Ресурсы / Resources

- [Microsoft Learn — sealed — https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/sealed](https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/sealed)
- [Microsoft Learn — internal — https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/internal](https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/internal)
- [Microsoft Learn — InternalsVisibleToAttribute — https://learn.microsoft.com/dotnet/api/system.runtime.compilerservices.internalsvisibletoattribute](https://learn.microsoft.com/dotnet/api/system.runtime.compilerservices.internalsvisibletoattribute)

---

[⬆ К модулю M05](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
