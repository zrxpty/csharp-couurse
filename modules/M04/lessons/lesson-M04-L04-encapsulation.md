[⬆ К модулю M04](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M04-L04: Инкапсуляция: модификаторы доступа / Encapsulation: access modifiers

**Модуль / Module:** M04
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Инкапсуляция — это один из четырёх китов ООП (вместе с наследованием, полиморфизмом и абстракцией), который отвечает за сокрытие внутреннего состояния объекта и контроль доступа к нему. Главная идея проста: объект сам управляет своими данными, а внешний код не может «залезть» внутрь и сломать инварианты — правила, при которых объект считается корректным.

Представьте банковский счёт. Баланс — это приватное поле. Если бы любой код мог напрямую писать в `balance`, можно было бы установить отрицательное значение или «накрутить» миллион. Поэтому поле закрывают, а доступ открывают через методы или свойства, которые проверяют условия: например, нельзя списать больше, чем есть на счёте. Так инкапсуляция защищает данные от «грязных рук».

В C# доступ к членам типа регулируется модификаторами доступа. Их семь, и важно понимать каждый:

- **public** — доступен всем: из любой сборки и любого кода. Это «открытая дверь».
- **private** — доступен только внутри того же класса или структуры. Это значение по умолчанию для членов класса; «дверь заперта на ключ».
- **protected** — доступен внутри класса и в его наследниках (даже если наследник в другой сборке).
- **internal** — доступен внутри одной сборки (DLL/EXE). Значение по умолчанию для типов верхнего уровня без явного модификатора.
- **protected internal** — объединение: доступен внутри сборки ИЛИ из наследников (где бы они ни находились). Логика «ИЛИ».
- **private protected** — пересечение: доступен внутри сборки И только если код при этом ещё и наследник. Логика «И».

Уровней вложенности два: модификатор члена класса всегда ограничен модификатором самого типа. Например, `public` поле внутри `internal` класса всё равно недоступно за пределами сборки, потому что сам класс невидим.

Сокрытие состояния в C# традиционно делают через свойства (properties). Поле делают `private`, а свойство `public` оборачивает его, добавляя проверки в `get` и `set`. Автосвойства (`public int X { get; set; }`) удобны, но не дают контроля — это просто «удобная обёртка». Когда нужна защита, используйте полное свойство с явным полем-хранилищем (`backing field`) и логикой в аксессорах.

Начиная с C# 6 доступно свойство только для чтения через автосвойство с приватным сеттером: `public string Name { get; }`. Значение задаётся в конструкторе и больше не меняется — отличный способ создать неизменяемые (immutable) объекты, которые потокобезопасны «по построению».

Ключевые принципы хорошего дизайна:
1. **Закрывай по умолчанию.** Начинай с `private` и открывай доступ только когда это действительно нужно. Принцип минимальной открытости.
2. **Поля — всегда private.** Публичные поля — это антипаттерн: они не позволяют добавить проверку позже без ломающей совместимости.
3. **Свойства для публичного контракта.** Даже если сейчас логики нет, свойство оставляет дверь открытой для будущих проверок.
4. **internal для «внутреннего API».** Если класс нужен только внутри библиотеки, не делай его `public` — это загрязняет публичный контракт.
5. **Не открывай коллекции напрямую.** Возвращай копии или `IReadOnlyCollection<T>`, иначе инкапсуляция нарушена: внешний код очистит ваш список.

Инкапсуляция — это не про «спрятать ради секрета», а про **контракт**: объект обещает всегда быть в согласованном состоянии, а модификаторы доступа — инструменты, которые заставляют внешний код соблюдать правила этого контракта.

#### Theory (EN)

Encapsulation is one of the four pillars of object-oriented programming (alongside inheritance, polymorphism, and abstraction), and its job is to hide an object's internal state and control how outside code reaches it. The core idea is simple: an object manages its own data, and no external code can poke inside and break the invariants — the rules that keep the object valid.

Imagine a bank account. The balance is a private field. If any code could write directly into `balance`, someone could set it negative or conjure a million out of thin air. So the field is locked away, and access is funnelled through methods or properties that enforce rules: you cannot withdraw more than you have. That is encapsulation protecting data from careless or hostile hands.

In C#, access to members is governed by access modifiers. There are seven, and each is worth understanding:

- **public** — accessible everywhere: from any assembly and any code. The wide-open door.
- **private** — accessible only inside the same class or struct. This is the default for class members; the door is locked with the only key held inside.
- **protected** — accessible inside the class and in its derived types, even when the derived type lives in a different assembly.
- **internal** — accessible anywhere within the same assembly (DLL/EXE). The default for top-level types that declare no explicit modifier.
- **protected internal** — a union: accessible inside the assembly OR from derived types anywhere. Think "OR" logic.
- **private protected** — an intersection: accessible only inside the assembly AND only when the code is also a derived type. Think "AND" logic.

There are two layers of nesting: a member's modifier is always bounded by the modifier of the type that contains it. A `public` field inside an `internal` class is still unreachable outside the assembly, because the class itself is invisible.

State hiding in C# is conventionally done through properties. The field is made `private`, and a `public` property wraps it, adding checks in `get` and `set`. Auto-properties (`public int X { get; set; }`) are convenient but offer no control — they are just a tidy wrapper. When you need protection, use a full property with an explicit backing field and logic in the accessors.

Since C# 6 you can express a read-only property through an auto-property with a private setter: `public string Name { get; }`. The value is assigned in the constructor and never changes afterward — a clean way to build immutable objects that are thread-safe by construction.

The core principles of good design:

1. **Default to closed.** Start with `private` and open up only when truly necessary — the principle of least exposure.
2. **Fields are always private.** Public fields are an anti-pattern: they leave no room to add validation later without breaking compatibility.
3. **Properties for the public contract.** Even with no logic today, a property keeps the door open for future checks.
4. **internal for "internal API".** If a class is only needed inside your library, do not mark it `public` — that pollutes the public contract.
5. **Do not expose collections directly.** Return copies or `IReadOnlyCollection<T>`, or encapsulation is broken: outside code will clear your list.

Encapsulation is not about "hiding for the sake of secrecy" — it is about **a contract**. The object promises to stay in a consistent state, and access modifiers are the tools that force outside code to honour that contract.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — Инкапсуляция и модификаторы доступа
// Encapsulation and access modifiers

using System;
using System.Collections.Generic;

namespace Course.M04;

// Внутренний тип сборки: наружу не виден.
// Internal type: not visible outside the assembly.
internal class BankAccount
{
    // Поле всегда private — баланс спрятан.
    // Field is always private — balance is hidden.
    private decimal _balance;

    // Поле только для чтения задаётся в конструкторе.
    // Read-only field assigned in the constructor.
    private readonly string _currency;

    // Автосвойство только для чтения (C# 6+) — неизменяемое.
    // Read-only auto-property (C# 6+) — immutable.
    public string Owner { get; }

    // Полное свойство с контролем: нельзя задать отрицательный баланс.
    // Full property with validation: no negative balance allowed.
    public decimal Balance
    {
        get => _balance;
        private set
        {
            if (value < 0)
                throw new ArgumentOutOfRangeException(nameof(value), "Баланс не может быть отрицательным. / Balance cannot be negative.");
            _balance = value;
        }
    }

    // Свойство без сеттера — вычисляется, не хранится.
    // Get-only computed property — not stored.
    public bool IsEmpty => _balance == 0;

    // Конструктор: единственная точка, где задаётся состояние.
    // Constructor: the single place where state is set.
    public BankAccount(string owner, decimal initialDeposit, string currency = "RUB")
    {
        Owner = owner ?? throw new ArgumentNullException(nameof(owner));
        _currency = currency ?? throw new ArgumentNullException(nameof(currency));
        Balance = initialDeposit; // идёт через проверку свойства / goes through property validation
    }

    // public метод — часть контракта с внешним миром.
    // public method — part of the contract with the outside world.
    public void Deposit(decimal amount)
    {
        if (amount <= 0)
            throw new ArgumentException("Сумма должна быть положительной. / Amount must be positive.", nameof(amount));
        Balance += amount;
    }

    public void Withdraw(decimal amount)
    {
        if (amount <= 0)
            throw new ArgumentException("Сумма должна быть положительной. / Amount must be positive.", nameof(amount));
        if (amount > _balance)
            throw new InvalidOperationException("Недостаточно средств. / Insufficient funds.");
        Balance -= amount;
    }

    // private метод — деталь реализации, недоступна снаружи.
    // private method — an implementation detail, not reachable from outside.
    private string FormatBalance() => $"{_balance:N2} {_currency}";

    public override string ToString() => $"{Owner}: {FormatBalance()}";
}

// Демонстрация protected / protected internal / private protected.
// Demonstrating protected / protected internal / private protected.
public class Entity
{
    protected Guid Id { get; }          // виден наследникам в любой сборке / visible to derived types in any assembly
    internal string Tag { get; set; }   // виден внутри сборки / visible within the assembly
    protected internal string Note { get; set; } // сборка ИЛИ наследник / assembly OR derived
    private protected DateTime CreatedAt { get; } // сборка И наследник / assembly AND derived

    public Entity(Guid id) { Id = id; CreatedAt = DateTime.UtcNow; }
}

// Наследник внутри той же сборки — видит protected, internal, private protected.
// Derived type in the same assembly — sees protected, internal, private protected.
public class UserEntity : Entity
{
    public UserEntity(Guid id) : base(id) { }

    public string Describe() =>
        $"Id={Id}, Tag={Tag}, Note={Note}, CreatedAt={CreatedAt:O}";
}

// Безопасный возврат коллекции: наружу — только чтение.
// Safe collection exposure: read-only to the outside.
internal class AccountRegistry
{
    private readonly List<BankAccount> _accounts = new();

    public void Add(BankAccount account) => _accounts.Add(account);

    // IReadOnlyCollection защищает внутренний список от изменений снаружи.
    // IReadOnlyCollection shields the internal list from outside mutation.
    public IReadOnlyCollection<BankAccount> Accounts => _accounts;
}

public static class Demo
{
    public static void Run()
    {
        var account = new BankAccount("Анна / Anna", 1000m);
        account.Deposit(500m);
        account.Withdraw(200m);
        Console.WriteLine(account);          // Анна / Anna: 1 300,00 RUB
        Console.WriteLine(account.IsEmpty);  // False

        var registry = new AccountRegistry();
        registry.Add(account);
        Console.WriteLine($"Счетов / Accounts: {registry.Accounts.Count}");
    }
}
```

#### Best Practices

- Начинай с `private` и расширяй доступ только при необходимости — принцип наименьшей открытости.
- Поля всегда `private`; публичный контракт реализуй через свойства.
- Используй `init` или get-only автосвойства для неизменяемых данных, задаваемых при создании.
- Для типов верхнего уровня без причины не открывай `public` — `internal` снижает площадь публичного API.
- Возвращай коллекции как `IReadOnlyCollection<T>` / `IReadOnlyList<T>`, а не как изменяемый `List<T>`.
- Не ставь проверки в свойствах так, чтобы свойство могло выбросить исключение в `get` — `get` должен быть безопасным и без побочных эффектов.

- Default to `private` and widen access only when needed — the principle of least privilege.
- Fields are always `private`; expose the public contract through properties.
- Use `init` or get-only auto-properties for immutable data set at construction time.
- Do not mark a top-level type `public` without cause — `internal` shrinks the public API surface.
- Return collections as `IReadOnlyCollection<T>` / `IReadOnlyList<T>`, never as a mutable `List<T>`.
- Keep `get` accessors free of side effects and exceptions; put validation in `set` or in methods.

#### Частые ошибки / Common Mistakes

- Публичные поля вместо свойств → всегда оборачивай поля в свойства, чтобы сохранить возможность добавить проверку.
- Возврат внутреннего `List<T>` напрямую → возвращай `IReadOnlyCollection<T>` или копию.
- Сеттер без проверки устанавливает некорректное значение → добавь валидацию в `set` и кидай осмысленные исключения.
- `public` класс там, где достаточно `internal` → засоряет публичный API библиотеки.
- Путаница `protected internal` (ИЛИ) и `private protected` (И) → проверяй таблицу доступности при написании библиотечного кода.
- Изменение состояния в `get` → `get` должен быть чистым; меняй состояние через методы или `set`.

- Public fields instead of properties → always wrap fields in properties so you can add validation later.
- Returning the internal `List<T>` directly → return `IReadOnlyCollection<T>` or a copy.
- A setter without validation accepts invalid values → validate in `set` and throw meaningful exceptions.
- A `public` class where `internal` would do → pollutes the library's public API.
- Confusing `protected internal` (OR) with `private protected` (AND) → consult the accessibility table when writing library code.
- Mutating state in a `get` accessor → keep `get` pure; change state through methods or `set`.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Все поля класса объявлены `private`.
- [ ] Публичный контракт реализован через свойства, а не поля.
- [ ] В сеттерах, меняющих критичное состояние, есть валидация.
- [ ] Неизменяемые данные используют `init` или get-only свойства.
- [ ] Типы верхнего уровня, не предназначенные для внешнего использования, помечены `internal`.
- [ ] Коллекции наружу возвращаются только для чтения.
- [ ] Я могу объяснить разницу между `protected internal` и `private protected`.

- [ ] All class fields are declared `private`.
- [ ] The public contract is exposed via properties, not fields.
- [ ] Setters that change critical state include validation.
- [ ] Immutable data uses `init` or get-only properties.
- [ ] Top-level types not meant for external use are marked `internal`.
- [ ] Collections are returned read-only to the outside.
- [ ] I can explain the difference between `protected internal` and `private protected`.

#### Ресурсы / Resources

- [Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/access-modifiers](https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/access-modifiers)
- [Properties (C# Programming Guide) — https://learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/properties](https://learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/properties)
- [Access Modifiers (C# Programming Guide) — https://learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/access-modifiers](https://learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/access-modifiers)

---

[⬆ К модулю M04](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
