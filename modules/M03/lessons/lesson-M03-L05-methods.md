---
[← Предыдущий: M03-L04](lesson-M03-L04-break-continue.md) | [⬆ К модулю M03](../README.md) | [Следующий: M03-L06 →](lesson-M03-L06-ref-out-in-params.md)
---

### Урок M03-L05: Объявление методов, сигнатура, возвращаемые значения / Declaring methods, signature, return values

**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Метод — это именованный блок кода, выполняющий одну логическую задачу. Можно сравнить его с рецептом на кухне: у рецепта есть название («Испечь хлеб»), список ингредиентов (параметры), последовательность шагов (тело) и результат — готовый хлеб (возвращаемое значение). Когда повар вызывает рецепт, он передаёт ингредиенты и получает на выходе блюдо.

Объявление метода в C# состоит из пяти обязательных частей, которые идут в строгом порядке:

```
модификатор_доступа  тип_возврата  ИмяМетода(параметры)
{
    // тело метода
}
```

1. **Модификатор доступа** — кто может вызывать метод. `public` виден всем, `private` — только внутри своего класса, `internal` — внутри сборки, `protected` — в классе и его наследниках. По умолчанию для методов действует `private`.
2. **Тип возврата** — что метод отдаёт обратно. Если метод должен вернуть значение, пишем конкретный тип (`int`, `string`, `List<Order>`). Если ничего не возвращает — пишем `void`.
3. **Имя метода** — именуется в стиле **PascalCase** и обязательно **глаголом**, потому что метод — это действие: `CalculateTotal`, `SendEmail`, `ValidateInput`. Существительное `Total` для метода — плохой стиль; так называют свойства.
4. **Параметры** — входные данные в скобках `(Type name, Type name)`. У каждого параметра есть тип и имя. Метод может не иметь параметров — тогда скобки пустые `()`.
5. **Тело метода** — код между `{ }`, который выполняется при вызове.

Совокупность модификатора доступа, типа возврата, имени и типов параметров (но не имён!) называют **сигнатурой метода**. Сигнатура определяет, можно ли перегрузить метод: две версии метода с одинаковым именем допустимы, только если их списки типов параметров различаются.

**static vs instance.** Метод без ключевого слова `static` — экземплярный: он работает с данными конкретного объекта и вызывается через переменную `order.Print()`. Метод `static` принадлежит самому классу, а не объекту, и вызывается через имя класса `Math.Max(a, b)`. Аналогия: экземплярный метод — это «приготовь ужин из продуктов в этом холодильнике», а статический — «посчитай НДС по ставке» — холодильник не нужен, результат зависит только от аргументов.

**void vs return.** Метод с `void` просто выполняет действие (например, печатает в консоль) и ничего не отдаёт. Метод с типом возврата обязан вернуть значение через `return значение;` и, как правило, не должен иметь побочных эффектов. Разделяйте «команды» (void) и «запросы» (return) — это принцип CQS (Command-Query Separation).

**Ранний возврат (early return).** Вместо того чтобы обрастать вложенными `if-else`, проверяйте невалидные условия в начале метода и выходите сразу:

```csharp
decimal CalculateDiscount(decimal price, Customer customer)
{
    if (price <= 0) return 0m;          // ранний возврат
    if (customer is null) return 0m;
    if (!customer.IsLoyal) return 0m;

    return price * 0.1m;                // основной путь
}
```

Так код читается сверху вниз как список правил, без «лесенки» отступов. Главное правило раннего возврата: выходите как можно раньше для невалидных и крайних случаев, а «счастливый путь» оставляйте напоследок.

#### Theory (EN)

A method is a named block of code that performs a single logical task. Think of it as a kitchen recipe: the recipe has a name (“Bake bread”), a list of ingredients (parameters), a sequence of steps (the body), and a result — the baked loaf (the return value). When a cook invokes the recipe, they pass ingredients in and get a dish out.

A method declaration in C# consists of five mandatory parts, written in a strict order:

```
access_modifier  return_type  MethodName(parameters)
{
    // method body
}
```

1. **Access modifier** — who is allowed to call the method. `public` is visible to everyone, `private` only inside its own class, `internal` inside the assembly, `protected` in the class and its inheritors. The implicit default for methods is `private`.
2. **Return type** — what the method gives back. If it must return a value, write a concrete type (`int`, `string`, `List<Order>`). If it returns nothing, write `void`.
3. **Method name** — named in **PascalCase** and must be a **verb**, because a method is an action: `CalculateTotal`, `SendEmail`, `ValidateInput`. A noun like `Total` for a method is a poor style — that name belongs to a property.
4. **Parameters** — input data in parentheses `(Type name, Type name)`. Each parameter has a type and a name. A method may have no parameters, in which case the parentheses are empty `()`.
5. **Method body** — the code between `{ }` that runs when the method is called.

The combination of the access modifier, return type, name, and parameter types (but not their names!) is called the **method signature**. The signature governs overloading: two overloads with the same name are allowed only if their parameter type lists differ.

**static vs instance.** A method without the `static` keyword is an instance method: it works with the data of a specific object and is called through a variable, `order.Print()`. A `static` method belongs to the class itself rather than to an object and is called through the class name, `Math.Max(a, b)`. Analogy: an instance method is “cook dinner from the food in *this* fridge”, while a static method is “compute VAT at the given rate” — no fridge needed, the result depends only on the arguments.

**void vs return.** A `void` method simply performs an action (e.g. prints to the console) and yields nothing. A method with a return type must return a value via `return value;` and ideally should have no side effects. Separate “commands” (void) from “queries” (return) — this is the Command-Query Separation (CQS) principle.

**Early return.** Instead of nesting `if-else` blocks, check invalid conditions at the top of the method and exit immediately:

```csharp
decimal CalculateDiscount(decimal price, Customer customer)
{
    if (price <= 0) return 0m;          // early return
    if (customer is null) return 0m;
    if (!customer.IsLoyal) return 0m;

    return price * 0.1m;                // happy path
}
```

This reads top-to-bottom as a list of rules, without a staircase of indentation. The key rule of early return: bail out as early as possible for invalid and edge cases, and leave the happy path for last.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — объявление методов, сигнатура, ранний возврат
// Declaring methods, signature, early return

using System.Collections.Generic;

public class Customer
{
    public string Name { get; init; }
    public bool IsLoyal { get; init; }
}

public static class OrderService
{
    // Статический метод, возвращает int — вычисляет количество бонусных баллов.
    // Static method, returns int — computes loyalty bonus points.
    public static int CalculatePoints(decimal amount)
    {
        if (amount <= 0m) return 0;             // ранний возврат / early return
        if (amount < 100m) return 1;
        if (amount < 1000m) return 10;

        return 50;                              // счастливый путь / happy path
    }

    // Перегрузка по сигнатуре: та же сигнатура имени, но другой список типов.
    // Overload by signature: same name, different parameter type list.
    public static int CalculatePoints(decimal amount, Customer customer)
    {
        int basePoints = CalculatePoints(amount);   // вызов первой перегрузки
        return customer?.IsLoyal == true ? basePoints * 2 : basePoints;
    }
}

public class Order
{
    public decimal Total { get; init; }
    public Customer? Customer { get; init; }

    // Экземплярный метод void — действие без возвращаемого значения.
    // Instance void method — an action with no return value.
    public void PrintSummary()
    {
        var points = OrderService.CalculatePoints(Total, Customer);
        var name = Customer?.Name ?? "Гость / Guest";
        System.Console.WriteLine(
            $"Заказ / Order: {Total:C}, {name}, баллы / points: {points}");
    }
}

// Демонстрация вызова / Usage demo
var order = new Order
{
    Total = 750m,
    Customer = new Customer { Name = "Иван / Ivan", IsLoyal = true }
};

order.PrintSummary();   // Заказ / Order: 750,00 ₽, Иван / Ivan, баллы / points: 20
```

#### Best Practices

- Именуйте методы глаголом в PascalCase: `CalculateTotal`, не `Total` и не `calculate_total`.
- Держите метод коротким (до ~20 строк) и сфокусированным на одной задаче — одна ответственность.
- Разделяйте команды (void, меняют состояние) и запросы (возвращают значение, без побочных эффектов) — принцип CQS.
- Используйте ранний возврат для невалидных входов и крайних случаев, оставляя «счастливый путь» без вложенности.
- Предпочитайте `static` для методов, не работающих с состоянием экземпляра: их проще тестировать и переиспользовать.
- Явно указывайте модификатор доступа даже если действует значение по умолчанию — так намерение очевидно.

- Name methods with a verb in PascalCase: `CalculateTotal`, not `Total` and not `calculate_total`.
- Keep methods short (~20 lines) and focused on a single task — one responsibility.
- Separate commands (void, mutate state) from queries (return a value, no side effects) — the CQS principle.
- Use early return for invalid inputs and edge cases, keeping the happy path free of nesting.
- Prefer `static` for methods that do not touch instance state: they are easier to test and reuse.
- State the access modifier explicitly even when the default applies — it makes intent obvious.

#### Частые ошибки / Common Mistakes

- Имя метода — существительное (`Total()`) → используйте глагол (`GetTotal()` или `CalculateTotal()`).
- Метод с типом возврата забыл `return` на каком-то пути → компилятор выдаст ошибку; покройте все ветки или используйте ранний возврат.
- Вложенные `if-else` глубиной в 4 уровня → замените проверками-стражами и ранним возвратом.
- Возврат значения и одновременно изменение состояния → нарушаете CQS; разделите на `void`-команду и запрос-возвращатель.
- Путаница `static` и экземплярного метода: вызов `this.Calculate()` для логики без состояния → сделайте метод `static` и вызывайте через класс.
- Слишком много параметров (6+) → сгруппируйте их в объект-параметр (record) или пересмотрите ответственность метода.
- Игнорирование `null` для ссылочных параметров → проверяйте через `is null` в начале метода и возвращайте безопасное значение.

- Method named with a noun (`Total()`) → use a verb (`GetTotal()` or `CalculateTotal()`).
- A method with a return type omits `return` on some path → the compiler errors out; cover all branches or use early return.
- Nested `if-else` four levels deep → replace with guard clauses and early return.
- Returning a value while also mutating state → you break CQS; split into a `void` command and a query.
- Confusing `static` and instance methods: calling `this.Calculate()` for stateless logic → make the method `static` and call through the class.
- Too many parameters (6+) → group them into a parameter object (record) or rethink the method's responsibility.
- Ignoring `null` for reference parameters → check with `is null` at the top and return a safe value.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Метод назван глаголом в PascalCase.
- [ ] Модификатор доступа указан явно и соответствует намерению.
- [ ] Тип возврата выбран корректно: `void` для действий, конкретный тип для вычислений.
- [ ] Все пути выполнения, требующие возврата, содержат `return`.
- [ ] Использован ранний возврат для невалидных входов и крайних случаев.
- [ ] `static` выбран осознанно, где не нужно состояние экземпляра.
- [ ] Метод решает одну задачу и умещается примерно в 20 строк.
- [ ] Команды (void) отделены от запросов (return).
- [ ] Параметры проверяются на `null` там, где это критично.

- [ ] Method is named with a verb in PascalCase.
- [ ] Access modifier is stated explicitly and matches intent.
- [ ] Return type is correct: `void` for actions, a concrete type for computations.
- [ ] Every execution path that must return a value has a `return`.
- [ ] Early return is used for invalid inputs and edge cases.
- [ ] `static` is a deliberate choice where instance state is not needed.
- [ ] The method does one job and fits in roughly 20 lines.
- [ ] Commands (void) are separated from queries (return).
- [ ] Parameters are checked for `null` where it matters.

#### Ресурсы / Resources

- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/methods — Методы (C#) / Methods (C#)
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/methods — Методы (руководство по программированию) / Methods (Programming Guide)

---
[← Предыдущий: M03-L04](lesson-M03-L04-break-continue.md) | [⬆ К модулю M03](../README.md) | [Следующий: M03-L06 →](lesson-M03-L06-ref-out-in-params.md)
---
