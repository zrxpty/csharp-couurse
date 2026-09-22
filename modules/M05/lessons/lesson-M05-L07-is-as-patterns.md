[⬆ К модулю M05](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M05-L07: is/as, pattern matching типов / is/as, type pattern matching

**Модуль / Module:** M05
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

В C# типы бывают разные: базовые классы, интерфейсы, записи (`record`), sealed-типы. Часто у вас есть ссылка на базовый тип, а нужно понять, какой именно производный тип скрывается за ней, и безопасно получить к нему доступ. Для этого служат операторы `is` и `as`, а также мощный механизм **pattern matching** (сопоставление шаблонов).

**Оператор `is`** проверяет совместимость типа и возвращает `bool`. Выражение `obj is Dog` отвечает `true`, только если `obj` действительно (или совместимо) является `Dog` — включая наследников и реализации интерфейсов. В отличие от прямого приведения `(Dog)obj`, `is` никогда не бросает `InvalidCastException`: он просто вернёт `false`, если тип не подходит. Это делает его идеальным «щитом» перед рискованным кастом.

**Оператор `as`** идёт дальше: он и проверяет, и приводит за один шаг. `Dog? dog = obj as Dog;` вернёт объект как `Dog`, если проверка прошла, и `null`, если нет. Главное правило — `as` работает только со **ссылочными и nullable-типами**, потому что для `int` вернуть `null` нельзя. Если нужен value type, используйте `is` с последующим приведением или `is Type x` — современный способ.

Здесь начинается **pattern matching**. Конструкция `obj is Dog dog` одновременно проверяет тип и объявляет переменную `dog`, в которую попадает приведённое значение. Это «type pattern» — он чище, чем связка `as` + `null check`: компилятор гарантирует, что внутри `if` переменная точно присвоена. Далее идут **property patterns**: `obj is Dog { Age: > 3 }` проверяет не только тип, но и свойство. Можно углубляться: `is Dog { Owner.Name: "Alice" }` — вкладывать свойства друг в друга.

Ещё мощнее — **switch по типам**. Начиная с C# 7 выражение `switch` превратилось в инструмент сопоставления:

```csharp
string Describe(Animal a) => a switch
{
    Dog d when d.Age < 1 => "Щенок",
    Dog d => $"Собака {d.Age} лет",
    Cat c => $"Кошка {c.Lives} жизней",
    null => "Пусто",
    _ => "Неизвестное животное"
};
```

Здесь каждый `case` — это шаблон: тип + необязательное условие (`when`) + привязка переменной. В C# 8 добавился **switch expressions** (как выше), в C# 9 — **relational patterns** (`> 3`, `< 10`) и **logical patterns** (`and`, `or`, `not`), в C# 10 — расширенные property patterns с вложенностью. К C# 12 / .NET 8 набор стал зрелым: можно писать `is not null`, `is Dog and { Age: >= 2 }`, комбинировать `or` для перечислений типов.

**Аналогия.** Представьте сортировочную станцию посылок: конвейер (`obj`) несёт коробки разных размеров. `is` — окошко, где оператор смотрит ярлык «хрупкое?» и кивает. `as` — робот-рукá, который аккуратно снимает посылку, если оно нужного класса, иначе пропускает. Pattern matching — это умный сканер с правилами: «если коробка большая И красная И подписана Alice — положить на полку B». Вы описываете форму данных, а компилятор строит эффективный код проверки.

**Когда что применять?** `is Type x` — современный дефолт для одной проверки. `as` — уместен, когда вы планируете дальше работать с `null`-результатом как с осмысленным состоянием. `switch` — когда ветвей несколько, особенно для иерархий и discriminated unions. Избегайте каскадов `if-else if` с ручным кастом: они многословны и легко рождают дублирование проверок. С pattern matching код читается как декларация бизнес-правил, а не как императивная инструкция.

#### Theory (EN)

In C#, types come in many flavors: base classes, interfaces, records, sealed types. Very often you hold a reference typed as a base class and need to discover which concrete derived type is hiding behind it — and get safe access to it. Two operators, `is` and `as`, plus the broader **pattern matching** engine, exist precisely for this job.

**The `is` operator** tests type compatibility and returns a `bool`. The expression `obj is Dog` answers `true` only when `obj` really is (or is compatible with) a `Dog` — including derived classes and interface implementations. Unlike a hard cast `(Dog)obj`, `is` never throws `InvalidCastException`: it just returns `false` when the type does not fit. That makes it the safe shield to hold up before any risky cast.

**The `as` operator** goes one step further: it both tests and converts in a single move. `Dog? dog = obj as Dog;` returns the object already typed as `Dog` when the check passes, and `null` otherwise. The key rule: `as` works only with **reference types and nullable value types**, because returning `null` for a plain `int` is impossible. For value types, reach for `is` plus an explicit cast, or better, `is Type x` — the modern idiom.

That is where **pattern matching** begins. The form `obj is Dog dog` simultaneously tests the type and declares a variable `dog` that receives the converted value. This is the **type pattern**, and it is cleaner than the `as`-plus-null-check combo: the compiler guarantees `dog` is assigned inside the `if` block. Then come **property patterns**: `obj is Dog { Age: > 3 }` checks both the type and a property. You can nest them: `is Dog { Owner.Name: "Alice" }` drills into sub-properties.

Even more powerful is the **type-based switch**. Since C# 7, `switch` became a pattern-matching tool:

```csharp
string Describe(Animal a) => a switch
{
    Dog d when d.Age < 1 => "Puppy",
    Dog d => $"Dog, {d.Age} years old",
    Cat c => $"Cat with {c.Lives} lives",
    null => "Empty",
    _ => "Unknown animal"
};
```

Each case is a pattern: a type, an optional `when` guard, and a variable binding. C# 8 added **switch expressions** (as above); C# 9 brought **relational patterns** (`> 3`, `< 10`) and **logical patterns** (`and`, `or`, `not`); C# 10 enriched property patterns with nested paths. By C# 12 / .NET 8 the toolkit is mature: you can write `is not null`, `is Dog and { Age: >= 2 }`, combine `or` to enumerate types.

**Analogy.** Picture a parcel-sorting station. A conveyor (`obj`) carries boxes of many shapes. `is` is the window where an operator glances at a "fragile?" label and nods yes or no. `as` is a robotic arm that gently lifts the box off the belt if it matches the target class, otherwise lets it pass. Pattern matching is the smart scanner with a rule book: "if the box is large AND red AND addressed to Alice, place it on shelf B". You describe the shape of the data; the compiler emits efficient checking code.

**When to reach for what?** `is Type x` is the modern default for a single check. `as` is fine when you intend to treat a `null` result as a meaningful state downstream. `switch` shines when there are several branches, especially for hierarchies and discriminated unions. Avoid cascading `if-else if` chains with manual casts: they are verbose and invite duplicated checks. With pattern matching, code reads like a declaration of business rules rather than an imperative recipe — and the compiler enforces exhaustiveness and order along the way.

#### Пример кода / Code Example

```csharp
// C# 12+ / .NET 8 — is/as и pattern matching типов
// is/as and type pattern matching

using System;

// Иерархия типов для демонстрации / Type hierarchy for the demo
public abstract class Animal                // Базовый класс / Base class
{
    public string Name { get; init; } = "";
}

public sealed record Dog(int Age, string Name) : Animal;          // Запись-наследник / Record derived type
public sealed record Cat(int Lives, string Name) : Animal;
public sealed record Fish(bool Freshwater, string Name) : Animal;

public static class AnimalOps
{
    // 1) Классический is + ручной каст (избегайте там, где можно применить паттерн)
    //    Classic is + manual cast (avoid where a pattern fits)
    public static string? DescribeOldWay(Animal? a)
    {
        if (a is null) return null;
        if (a is Dog)
        {
            var dog = (Dog)a;               // Безопасно: тип уже проверен / Safe: type already verified
            return $"Собака {dog.Age} лет / Dog, {dog.Age} years";
        }
        return "Неизвестно / Unknown";
    }

    // 2) as — проверить и привести одним шагом (только для ссылочных/nullable)
    //    as — test and cast in one step (reference/nullable only)
    public static string? TryAsDog(Animal? a)
    {
        Dog? dog = a as Dog;                // null, если не Dog / null when not Dog
        return dog is null
            ? "Это не собака / Not a dog"
            : $"Собака {dog.Age} лет / Dog, {dog.Age} years";
    }

    // 3) Type pattern: is Type x — современный дефолт
    //    Type pattern: is Type x — the modern default
    public static string Describe(Animal? a)
    {
        if (a is Dog { Age: < 1 } puppy)    // type + property pattern / тип + паттерн свойства
            return $"Щенок {puppy.Name} / Puppy {puppy.Name}";
        if (a is Dog d)
            return $"Собака {d.Name}, {d.Age} лет / Dog {d.Name}, {d.Age} years";
        if (a is Cat c)
            return $"Кошка {c.Name}, {c.Lives} жизней / Cat {c.Name}, {c.Lives} lives";
        if (a is Fish { Freshwater: true } fw)
            return $"Пресноводная рыба {fw.Name} / Freshwater fish {fw.Name}";
        if (a is null)                      // проверка на null / null check
            return "Пусто / Empty";
        return "Неизвестное животное / Unknown animal";
    }

    // 4) Switch expression по типам с логическими и реляционными паттернами
    //    Switch expression over types with logical and relational patterns
    public static string Category(Animal a) => a switch
    {
        Dog { Age: < 1 }            => "Молодой питомец / Young pet",
        Dog { Age: >= 10 }          => "Пожилая собака / Senior dog",
        Dog                          => "Взрослая собака / Adult dog",
        Cat { Lives: 1 }            => "Последняя жизнь / Last life",
        Cat                          => "Кошка / Cat",
        Fish { Freshwater: true }   => "Аквариумная рыба / Aquarium fish",
        Fish                         => "Морская рыба / Sea fish",
        null                         => "Пусто / Empty",
        _                            => "Неизвестно / Unknown"   // discard: всё остальное / discard: everything else
    };

    // 5) Комбинирование паттернов: and / or / not
    //    Combining patterns: and / or / not
    public static bool IsWaterCreature(Animal a) =>
        a is Fish or (Animal and not Dog and not Cat);

    public static bool IsNotNull(object? o) => o is not null;   // чище, чем o != null / cleaner than o != null
}
```

#### Best Practices

- Используйте `is Type x` вместо связки `as` + проверка на `null` — короче, и компилятор сам отслеживает присвоение переменной.
- В switch-выражениях располагайте частные паттерны раньше общих: совпадение выбирается сверху вниз, поэтому `Dog { Age: < 1 }` должен стоять выше простого `Dog`.
- Помечайте листовые типы иерархии как `sealed` — это позволяет компилятору применять более быстрые проверки и помогает exhaustive-анализу.
- Prefer `is Type x` over the `as` + null-check pair — it is shorter and the compiler tracks the variable assignment for you.
- In switch expressions, put specific patterns before general ones: matching is top-down, so `Dog { Age: < 1 }` must precede a bare `Dog`.
- Mark leaf types in a hierarchy as `sealed` so the compiler can emit faster checks and assist exhaustive analysis.

#### Частые ошибки / Common Mistakes

- [Использование `as` с value type: `int x = obj as int;`] → [Не компилируется; используйте `obj is int x` или `Convert.ToInt32`. Для nullable: `int? x = obj as int?`.]
- [Забыли ветку `_` или `null` в switch-выражении над не-sealed иерархией] → [Компилятор потребует exhaustiveness; добавьте `_ => ...` или сделайте типы `sealed`.]
- [Проверка `obj != null` вместо `obj is not null`] → [Со структурной точки зрения `is not null` строже работает с nullable-значениями и читается декларативно.]
- [Using `as` with a value type: `int x = obj as int;`] → [It does not compile; use `obj is int x` or `Convert.ToInt32`. For nullable, write `int? x = obj as int?`.]
- [Forgetting the `_` or `null` arm in a switch expression over a non-sealed hierarchy] → [The compiler demands exhaustiveness; add `_ => ...` or seal the types.]
- [Writing `obj != null` instead of `obj is not null`] → [`is not null` is stricter on nullable values and reads declaratively.]

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Я использую `is Type x` вместо связки `as` + `if (x != null)`.
- [ ] В switch-выражениях частные паттерны расположены выше общих.
- [ ] Я не применяю `as` к value type без `?`.
- [ ] Листовые типы иерархии помечены `sealed`, где это уместно.
- [ ] Я использую `is not null` вместо `!= null` для nullable-проверок.
- [ ] I use `is Type x` instead of the `as` + `if (x != null)` combo.
- [ ] In switch expressions, specific patterns appear before general ones.
- [ ] I never apply `as` to a value type without `?`.
- [ ] Leaf types in the hierarchy are marked `sealed` where appropriate.
- [ ] I prefer `is not null` over `!= null` for nullable checks.

#### Ресурсы / Resources

- [Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/fundamentals/functional/pattern-matching]

---

[⬆ К модулю M05](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
