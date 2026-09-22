---
[← К уроку M08-L07](lesson-M08-L07-first-single-elementat.md) | [⬆ К модулю M08](../README.md) | [Следующее ДЗ →](homework-M08-L08-deferred-execution.md)
---

### Домашнее задание M08-L07: First/SingleOrDefault/ElementAt и исключения / Homework M08-L07: First/SingleOrDefault/ElementAt and exceptions

**Урок / Lesson:** M08-L07
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) Научиться осознанно выбирать между `First`/`FirstOrDefault`, `Single`/`SingleOrDefault` и `ElementAt`/`ElementAtOrDefault` в зависимости от бизнес-семантики отсутствия или дублирования данных, корректно обрабатывать исключения `InvalidOperationException` и `ArgumentOutOfRangeException`, а также учитывать разницу между `IEnumerable` и `IList` с точки зрения производительности и nullable-контекстов. (EN) Learn to consciously choose between `First`/`FirstOrDefault`, `Single`/`SingleOrDefault` and `ElementAt`/`ElementAtOrDefault` based on the business semantics of missing or duplicated data, handle `InvalidOperationException` and `ArgumentOutOfRangeException` correctly, and account for the difference between `IEnumerable` and `IList` in terms of performance and nullable contexts.

#### Связь с уроком / Connection to the lesson
(RU) Урок вводит три семейства операторов доступа к одиночным элементам и объясняет, что суффикс `OrDefault` спасает только от «пусто», но не от «слишком много». ДЗ закрепляет это на реалистичной задаче репозитория пользователей, где каждый выбор оператора должен быть продиктован требованиями к данным, а не привычкой. (EN) The lesson introduces three families of single-element access operators and explains that the `OrDefault` suffix only saves you from "empty", not from "too many". This homework reinforces that on a realistic user-repository task, where each operator choice must be driven by data requirements, not habit.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Вы присоединяетесь к команде, разрабатывающей небольшой сервис управления пользователями на C# 12 / .NET 8. В кодовой базе уже есть сущность `User` с полями `Id`, `Email`, `Role` и `RegisteredAt`, а также in-memory репозиторий `UserRepository`, хранящий `List<User>`. Исторически разработчики обращались к коллекции напрямую и использовали оператор `First()` «на автомате» практически везде: для поиска по идентификатору, для получения первого администратора, для выборки по email и даже для проверки, есть ли хоть один пользователь с заданной ролью. Из-за этого в продакшене периодически всплывают две категории проблем. Первая — необработанные `InvalidOperationException` с сообщением «Sequence contains no elements», которые превращают пустые результаты в 500-е ошибки там, где клиент ждал 404. Вторая — тихие баги, когда `FirstOrDefault` возвращал `null`, но разработчик забывал проверить результат и получал `NullReferenceException` на несколько строк ниже.

Ваш лид технический попросил вас провести ревизию доступа к одиночным элементам, переписать критичные места так, чтобы выбор оператора соответствовал бизнес-семантике, и подготовить набор юнит-тестов, которые фиксируют поведение каждого оператора на пустых, одноэлементных и многоэлементных последовательностях. Заодно вы должны показать, как `ElementAt` и индексный синтаксис `^1` помогают получать элементы по позиции, и объяснить, почему `Single` уместен при поиске по уникальному ключу, но вреден там, где дубликаты допустимы. ДЗ моделирует именно эту работу: вы создаёте классы, реализуете методы с правильными операторами, пишете тесты и сопровождаете всё комментариями с обоснованием выбора.

#### Что нужно сделать (пошагово)

1. Создайте новый консольный проект: `dotnet new console -n UserAccess -o UserAccess` с параметрами C# 12 и .NET 8. Перейдите в папку: `cd UserAccess`. Убедитесь, что в `UserAccess.csproj` указано `<TargetFramework>net8.0</TargetFramework>` и `<Nullable>enable</Nullable>`.
2. Добавьте проект xUnit для тестов: `dotnet new xunit -n UserAccess.Tests -o UserAccess.Tests`, затем `dotnet add UserAccess.Tests/UserAccess.Tests.csproj reference UserAccess/UserAccess.csproj`. Установите FluentAssertions, если привыкли, но достаточно стандартного `Assert.*`.
3. В основном проекте создайте файл `User.cs` с записью `public sealed record User(int Id, string Email, string Role, DateTime RegisteredAt);`. Поле `Email` считайте уникальным в рамках набора данных.
4. Создайте `UserRepository.cs` с классом `UserRepository`, который в конструкторе принимает `IReadOnlyList<User>` и хранит его в приватном поле. Реализуйте публичные методы:
   - `User GetById(int id)` — должен бросать `InvalidOperationException` со своим сообщением, если пользователь не найден, потому что отсутствие Id в системе — ошибка; используйте `Single`.
   - `User? FindById(int id)` — должен вернуть `null`, если пользователя нет; используйте `SingleOrDefault` и поясните в комментарии, что дубликат Id всё равно бросит исключение, потому что это инвариант данных.
   - `User GetByEmail(string email)` — поиск по уникальному email; используйте `Single` и задокументируйте, почему именно он.
   - `User? FindAdmin()` — первый администратор по роли, если его нет — `null`; используйте `FirstOrDefault`.
   - `User GetFirstRegistered()` — самый ранний по `RegisteredAt`; если список пуст — ошибка конфигурации, используйте `First` после сортировки.
   - `User? GetByIndex(int index)` — обёртка над `ElementAtOrDefault`, возвращающая `null` при выходе за границы; в комментарии укажите, что `ElementAt` бросил бы `ArgumentOutOfRangeException`.
   - `User GetLastByIndex()` — через `ElementAt(^1)` для демонстрации `Index`; бросает `InvalidOperationException`, если репозиторий пуст.
5. В `Program.cs` продемонстрируйте работу: создайте список из 4 пользователей, вызовите каждый метод, для бросающих вариантов оберните вызовы в `try/catch (InvalidOperationException)` и `try/catch (ArgumentOutOfRangeException)`, выводя результат в консоль. Ожидаемый вывод должен содержать строки вида `By Id=2: bob@example.com`, `FindById 999: <null>`, `First registered: alice@example.com`, `Last by Index: charlie@example.com`, а также сообщения о пойманных исключениях.
6. В тестовом проекте создайте `UserRepositoryTests.cs` и напишите минимум 10 тестов: пустой список и `GetById` → `Assert.Throws<InvalidOperationException>`; `FindById` на отсутствующем → `Assert.Null`; `GetByEmail` на существующем; `GetByEmail` на дублирующемся email → `Assert.Throws<InvalidOperationException>`; `FindAdmin` без администраторов → `Assert.Null`; `GetByIndex(-1)` → `Assert.Null`; `GetByIndex(100)` → `Assert.Null`; `GetLastByIndex` на пустом → `Assert.Throws<InvalidOperationException>`; проверка производительности через `IList` (опционально, через `BenchmarkDotNet` или просто комментарий).
7. В комментариях к каждому методу обязательно укажите, какой оператор использован, почему именно он, какое исключение возможен и какова сложность на `IList` vs `IEnumerable`. Запустите `dotnet test` — все тесты должны быть зелёными. Запустите `dotnet run` — вывод должен соответствовать описанию выше.

#### Требования к решению

- Код должен компилироваться под C# 12 и .NET 8 без предупреждений, связанных с nullable. Включите `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` в оба csproj, чтобы форсировать чистоту.
- Все возвращаемые типы должны корректно отражать nullable-семантику: методы, которые могут не найти элемент и возвращают `null`, обязаны иметь `?` в сигнатуре (`User?`), а методы, бросающие исключение, — возвращать `User` без `?`.
- Запрещено использовать `Where(...).First(...)` — применяйте перегрузку с предикатом напрямую: `First(u => ...)`, `Single(u => ...)`, `FirstOrDefault(u => ...)`. Это избегает лишней аллокации итератора.
- Каждый публичный метод репозитория должен содержать XML-комментарий `<summary>` с указанием оператора и семантики исключения, например: «Использует `Single`; бросает `InvalidOperationException`, если Id не найден или встречен дубликат».
- В тестах используйте `Assert.Throws<T>` для проверки исключений; не ловите их через `try/catch` внутри тестов вручную. Тесты должны быть независимыми и не полагаться на порядок выполнения.
- Решение должно быть структурировано: модель в `User.cs`, репозиторий в `UserRepository.cs`, демонстрация в `Program.cs`, тесты в `UserRepositoryTests.cs`. Не сваливайте всё в один файл.
- Не вводите собственные классы исключений в базовом варианте — работайте со стандартными `InvalidOperationException` и `ArgumentOutOfRangeException`, чтобы закрепить именно LINQ-семантику.

#### Тонкости и подводные камни

- **First vs FirstOrDefault:** `First` бросает `InvalidOperationException` на пустой последовательности, `FirstOrDefault` возвращает `default(T)`. Для ссылочных типов `default` — это `null`, что легко спутать с «не найдено», но для значимых типов `default(int)` — это `0`, который может быть валидным значением. Поэтому проверка `if (result == default)` для `int`-результата опасна: ноль может быть легитимным. В таких случаях лучше использовать nullable-обёртку `int?` или отдельный флаг `bool TryX(...)`.
- **Single vs SingleOrDefault:** это самая коварная пара. `SingleOrDefault` НЕ спасает от дубликатов — он бросает `InvalidOperationException`, если элементов больше одного. Суффикс `OrDefault` меняет поведение только для пустого случая. Поэтому `users.SingleOrDefault(u => u.Id == 999)` на пустом списке вернёт `null`, но `users.SingleOrDefault(u => u.Id > 0)` на списке из десяти пользователей бросит исключение. Это нужно чётко понимать и фиксировать тестами.
- **ElementAt vs ElementAtOrDefault:** `ElementAt` бросает `ArgumentOutOfRangeException` (не `InvalidOperationException`!) при отрицательном индексе или выходе за границы. Это другое семейство исключений, и его нужно ловить отдельно, если вы пишете общий обработчик. С .NET 6+ доступна перегрузка с `Index` и `Range`: `items.ElementAt(^1)` — последний элемент, `items.ElementAt(0)` — первый. `ElementAtOrDefault` возвращает `default` вместо исключения, что удобно для «мягкой» индексации.
- **Производительность:** `First` и `ElementAt` на `IList<T>` (включая массивы и `List<T>`) работают за O(1), потому что LINQ через `as IList<T>` обращается к индексатору. На произвольном `IEnumerable<T>` (например, результат `Where` или `Select`) — O(n). Если в репозитории хранится `IReadOnlyList<User>`, методы `First` и `ElementAt` будут быстрыми; если же вы передадите `IEnumerable<User>` из отложенного запроса, каждый вызов будет заново перебирать. `Single` всегда перебирает до второго совпадения, поэтому он дороже `First` даже на `IList`.
- **Разница с IQueryable:** в EF Core `First`, `Single` и `ElementAt` транслируются в SQL: `First` → `TOP 1`, `Single` → `TOP 2` с проверкой на клиенте, `ElementAt` → `OFFSET ... FETCH NEXT`. На `IQueryable` `SingleOrDefault` всё равно бросит на клиенте при >1 записи, потому что проверка уникальности выполняется в памяти после материализации. Это типичный источник неожиданных исключений в API над базой.
- **Nullable-контексты:** при `<Nullable>enable</Nullable>` `FirstOrDefault` для ссылочного типа возвращает `T?`, и компилятор заставит вас обработать `null`. Для значимого типа `FirstOrDefault` возвращает `T` без `?`, и `default(int) == 0` пройдёт без предупреждения — вот тут и кроется ловушка. Используйте `int?` или явные проверки.

#### Критерии приёмки

- [ ] Проект `UserAccess` собирается под .NET 8 с `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` без ошибок.
- [ ] Проект `UserAccess.Tests` собирается и проходит `dotnet test` с минимум 10 зелёными тестами.
- [ ] `User` реализован как `sealed record` с полями `Id`, `Email`, `Role`, `RegisteredAt`.
- [ ] `UserRepository` хранит `IReadOnlyList<User>`, не暴露 mutable список наружу.
- [ ] `GetById` использует `Single` и бросает `InvalidOperationException` при отсутствии или дубликате.
- [ ] `FindById` использует `SingleOrDefault` и возвращает `null` при отсутствии; в комментарии указано, что дубликат всё равно бросит.
- [ ] `GetByEmail` использует `Single` с обоснованием уникальности email.
- [ ] `FindAdmin` использует `FirstOrDefault(u => u.Role == "admin")` и возвращает `User?`.
- [ ] `GetFirstRegistered` сортирует по `RegisteredAt` и использует `First`; бросает на пустом списке.
- [ ] `GetByIndex` использует `ElementAtOrDefault` и возвращает `User?`.
- [ ] `GetLastByIndex` использует `ElementAt(^1)` и бросает `InvalidOperationException` на пустом.
- [ ] Все сигнатуры корректно отражают nullable-семантику (`User?` для «может не найтись», `User` для «бросает»).
- [ ] В `Program.cs` демонстрируются и try/catch для `InvalidOperationException`, и для `ArgumentOutOfRangeException`.
- [ ] Нет ни одного `Where(...).First(...)` — везде используется перегрузка с предикатом.
- [ ] Каждый публичный метод снабжён XML-комментариями с указанием оператора и семантики исключения.

#### Подсказки (без прямого ответа)

- Подумайте, какой оператор даёт строгую гарантию уникальности — это подсказка для `GetById` и `GetByEmail`.
- Вспомните, что `ElementAtOrDefault` отличается от `ElementAt` типом исключения (точнее, его отсутствием). Где это полезно? Там, где выход за границы — нормальный сценарий.
- Для «первый администратор» дубликаты администраторов допустимы — значит, строгий оператор не нужен.
- Для последнего элемента `ElementAt(^1)` читается лучше, чем `Last()`, но ведёт себя эквивалентно. В чём тогда разница? В явной позиционной семантике.
- Если вы хотите вернуть `int`-результат из `FirstOrDefault` и при этом отличить «нет элемента» от «элемент со значением 0», рассмотрите приведение к `int?` или `Select` в `int?` перед `FirstOrDefault`.

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8 — UserAccess / UserRepository.cs
// Решение домашнего задания M08-L07
// Homework solution M08-L07

using System;
using System.Collections.Generic;
using System.Linq;

namespace UserAccess;

/// <summary>
/// Пользователь системы. Email уникален в рамках набора данных.
/// System user. Email is unique within the dataset.
/// </summary>
public sealed record User(int Id, string Email, string Role, DateTime RegisteredAt);

/// <summary>
/// In-memory репозиторий пользователей. Демонстрирует выбор операторов
/// First / Single / ElementAt и их OrDefault-вариантов по бизнес-семантике.
/// In-memory user repository. Demonstrates operator choice by business semantics.
/// </summary>
public sealed class UserRepository
{
    private readonly IReadOnlyList<User> _users;

    public UserRepository(IReadOnlyList<User> users) => _users = users;

    /// <summary>
    /// Использует Single: отсутствие или дубликат Id — ошибка данных.
    /// Throws InvalidOperationException when Id is missing or duplicated.
    /// </summary>
    public User GetById(int id) =>
        _users.Single(u => u.Id == id);

    /// <summary>
    /// Использует SingleOrDefault: возвращает null при отсутствии,
    /// но дубликат Id всё равно бросит InvalidOperationException.
    /// Returns null when missing; duplicate still throws.
    /// </summary>
    public User? FindById(int id) =>
        _users.SingleOrDefault(u => u.Id == id);

    /// <summary>
    /// Single: email уникален, дубликат — инвариантное нарушение.
    /// Single: email is unique, duplicate breaks an invariant.
    /// </summary>
    public User GetByEmail(string email) =>
        _users.Single(u => u.Email == email);

    /// <summary>
    /// FirstOrDefault: администраторов может быть несколько, нам нужен первый.
    /// FirstOrDefault: multiple admins allowed, we need the first one.
    /// </summary>
    public User? FindAdmin() =>
        _users.FirstOrDefault(u => u.Role == "admin");

    /// <summary>
    /// Сортируем по RegisteredAt и берём First; пустой список — ошибка конфигурации.
    /// Order by RegisteredAt then First; empty list is a config error.
    /// </summary>
    public User GetFirstRegistered() =>
        _users.OrderBy(u => u.RegisteredAt).First();

    /// <summary>
    /// ElementAtOrDefault: мягкая индексация, null при выходе за границы.
    /// ElementAtOrDefault: soft indexing, null when out of range.
    /// </summary>
    public User? GetByIndex(int index) =>
        _users.ElementAtOrDefault(index);

    /// <summary>
    /// ElementAt(^1): последний элемент через Index (.NET 6+).
    /// На пустом бросает InvalidOperationException.
    /// ElementAt(^1): last element via Index; throws on empty.
    /// </summary>
    public User GetLastByIndex() =>
        _users.ElementAt(^1);
}
```

Разбор по строкам. `GetById` использует `Single`, потому что отсутствие Id в системе — это не «может быть, а может не быть», а именно ошибка: либо данных нет, либо их слишком много (дубликат ключа). `Single` ловит обе ситуации одним исключением. `FindById` — это «мягкая» версия: клиент хочет узнать, есть ли пользователь, и `null` — валидный ответ, но дубликат Id по-прежнему бросает, потому что `SingleOrDefault` отличает «пусто» от «слишком много». В комментарии это явно зафиксировано, чтобы следующий разработчик не ожидал, что метод «всегда безопасен». `GetByEmail` повторяет логику `GetById`, потому что email тоже инвариант: дублирующиеся email в системе — это баг импорта данных, и лучше взорваться рано, чем тихо вернуть один из дублей.

`FindAdmin` использует `FirstOrDefault`, потому что администраторов может быть несколько, и нам нужен любой первый. Здесь дубликаты допустимы, поэтому `Single` был бы вреден — он бросал бы ложные исключения в системах с двумя и более админами. `GetFirstRegistered` сначала сортирует, потом берёт `First`: пустой список здесь — ошибка конфигурации (репозиторий не должен быть пуст в проде), поэтому `FirstOrDefault` не нужен. Сортировка превращает `IReadOnlyList` в `IOrderedEnumerable`, так что последующий `First` работает за O(n log n) на сортировке плюс O(1) на первом элементе.

`GetByIndex` использует `ElementAtOrDefault`, потому что выход за границы — нормальный сценарий для пагинации и клиентских запросов: вернуть `null` лучше, чем бросать `ArgumentOutOfRangeException`. `GetLastByIndex` намеренно использует `ElementAt(^1)`, чтобы продемонстрировать `Index`-синтаксис .NET 6+: `^1` означает «один с конца». На пустом списке `ElementAt(^1)` бросит `InvalidOperationException` (а не `ArgumentOutOfRangeException`, как при числовом индексе за границами — это тонкая разница перегрузок). Все методы возвращают `User?` ровно тогда, когда `null` — валидный результат, и `User` без `?`, когда отсутствие — ошибка; так nullable-контекст форсирует корректную обработку на стороне вызывающего.

#### Задания на углубление (бонус)

1. Реализуйте вариант `GetById` с `IQueryable<User>` (имитируя EF Core): напишите класс `FakeUserQuery` и покажите, что `SingleOrDefault` всё равно бросает на клиенте при дубликате после материализации. Объясните, почему это так.
2. Добавьте метод `bool TryGetById(int id, out User user)`, который внутренне использует `Where(...).Take(2)` и ручную проверку, чтобы избежать двойного перебора `Single`. Сравните производительность с `SingleOrDefault` на больших списках через `BenchmarkDotNet`.
3. Покажите, что `FirstOrDefault`, возвращающий `int` для значимого типа, может вернуть `0` как валидное значение. Реализуйте `int? FindAgeByEmail(string email)` через `Select(u => (int?)u.Age).FirstOrDefault(...)`, чтобы отличить «нет элемента» от «возраст 0».
4. Напишите метод, принимающий `IEnumerable<User>` (не `IReadOnlyList`), и с помощью `BenchmarkDotNet` покажите разницу O(1) vs O(n) для `First` и `ElementAt` в зависимости от реального типа источника.

---

## Statement in English / Постановка на английском

#### Context & motivation

You are joining a team building a small user-management service on C# 12 / .NET 8. The codebase already contains a `User` entity with fields `Id`, `Email`, `Role` and `RegisteredAt`, plus an in-memory repository `UserRepository` that holds a `List<User>`. Historically the developers reached into the collection directly and used the `First()` operator almost everywhere by reflex: for lookups by identifier, for getting the first administrator, for selecting by email, and even for checking whether at least one user with a given role exists. As a result, two categories of problems keep surfacing in production. The first is unhandled `InvalidOperationException` with the message "Sequence contains no elements", which turns empty results into HTTP 500 errors where the client expected a 404. The second is silent bugs where `FirstOrDefault` returned `null`, but the developer forgot to check and hit a `NullReferenceException` a few lines later.

Your tech lead has asked you to audit single-element access, rewrite the critical spots so that the operator choice matches the business semantics, and prepare a set of unit tests that fix the behavior of each operator on empty, single-element and multi-element sequences. Along the way you must show how `ElementAt` and the `^1` index syntax help fetch elements by position, and explain why `Single` is appropriate for unique-key lookups but harmful where duplicates are acceptable. This homework models exactly that work: you create the classes, implement methods with the correct operators, write tests, and annotate everything with comments justifying the choices.

#### What to do step by step

1. Create a new console project: `dotnet new console -n UserAccess -o UserAccess` targeting C# 12 and .NET 8. Navigate into it: `cd UserAccess`. Ensure that `UserAccess.csproj` contains `<TargetFramework>net8.0</TargetFramework>` and `<Nullable>enable</Nullable>`.
2. Add an xUnit test project: `dotnet new xunit -n UserAccess.Tests -o UserAccess.Tests`, then `dotnet add UserAccess.Tests/UserAccess.Tests.csproj reference UserAccess/UserAccess.csproj`. You may install FluentAssertions if you prefer it, but the standard `Assert.*` is enough.
3. In the main project create `User.cs` with the record `public sealed record User(int Id, string Email, string Role, DateTime RegisteredAt);`. Treat `Email` as unique within the dataset.
4. Create `UserRepository.cs` with a `UserRepository` class whose constructor accepts `IReadOnlyList<User>` and stores it in a private field. Implement the following public methods:
   - `User GetById(int id)` — must throw `InvalidOperationException` with a custom message when the user is not found, because a missing Id is an error; use `Single`.
   - `User? FindById(int id)` — must return `null` when the user is absent; use `SingleOrDefault` and explain in a comment that a duplicate Id will still throw, because that is a data invariant.
   - `User GetByEmail(string email)` — lookup by the unique email; use `Single` and document why.
   - `User? FindAdmin()` — the first administrator by role, or `null` if none; use `FirstOrDefault`.
   - `User GetFirstRegistered()` — the earliest by `RegisteredAt`; an empty list is a configuration error, so use `First` after ordering.
   - `User? GetByIndex(int index)` — a wrapper over `ElementAtOrDefault` that returns `null` when out of range; in the comment note that `ElementAt` would throw `ArgumentOutOfRangeException`.
   - `User GetLastByIndex()` — via `ElementAt(^1)` to demonstrate `Index`; throws `InvalidOperationException` when the repository is empty.
5. In `Program.cs` demonstrate the behavior: build a list of four users, call every method, wrap the throwing variants in `try/catch (InvalidOperationException)` and `try/catch (ArgumentOutOfRangeException)`, printing results to the console. The expected output should contain lines like `By Id=2: bob@example.com`, `FindById 999: <null>`, `First registered: alice@example.com`, `Last by Index: charlie@example.com`, plus messages for the caught exceptions.
6. In the test project create `UserRepositoryTests.cs` and write at least ten tests: empty list and `GetById` → `Assert.Throws<InvalidOperationException>`; `FindById` on a missing id → `Assert.Null`; `GetByEmail` on an existing email; `GetByEmail` on a duplicated email → `Assert.Throws<InvalidOperationException>`; `FindAdmin` with no admins → `Assert.Null`; `GetByIndex(-1)` → `Assert.Null`; `GetByIndex(100)` → `Assert.Null`; `GetLastByIndex` on an empty repo → `Assert.Throws<InvalidOperationException>`; an optional performance check through `IList` (via `BenchmarkDotNet` or just a comment).
7. In the comments on every method, state which operator is used, why, what exception is possible, and the complexity on `IList` vs `IEnumerable`. Run `dotnet test` — all tests must be green. Run `dotnet run` — the output must match the description above.

#### Requirements

- The code must compile under C# 12 and .NET 8 with no nullable-related warnings. Enable `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` in both csproj files to enforce cleanliness.
- All return types must correctly reflect nullable semantics: methods that may not find an element and return `null` must carry `?` in the signature (`User?`), while methods that throw must return `User` without `?`.
- It is forbidden to use `Where(...).First(...)` — apply the predicate overload directly: `First(u => ...)`, `Single(u => ...)`, `FirstOrDefault(u => ...)`. This avoids an extra iterator allocation.
- Every public repository method must carry an XML `<summary>` comment naming the operator and the exception semantics, for example: "Uses `Single`; throws `InvalidOperationException` when the Id is missing or duplicated."
- In tests, use `Assert.Throws<T>` to verify exceptions; do not catch them manually with `try/catch` inside tests. Tests must be independent and must not rely on execution order.
- The solution must be structured: the model in `User.cs`, the repository in `UserRepository.cs`, the demo in `Program.cs`, the tests in `UserRepositoryTests.cs`. Do not dump everything into a single file.
- Do not introduce custom exception classes in the baseline variant — work with the standard `InvalidOperationException` and `ArgumentOutOfRangeException`, so that the LINQ semantics themselves are reinforced.

#### Pitfalls

- **First vs FirstOrDefault:** `First` throws `InvalidOperationException` on an empty sequence, `FirstOrDefault` returns `default(T)`. For reference types `default` is `null`, which is easy to mistake for "not found", but for value types `default(int)` is `0`, which may be a valid value. So checking `if (result == default)` on an `int` result is dangerous: zero may be legitimate. In such cases prefer a nullable wrapper `int?` or a separate `bool TryX(...)` pattern.
- **Single vs SingleOrDefault:** this is the trickiest pair. `SingleOrDefault` does NOT save you from duplicates — it throws `InvalidOperationException` when there is more than one element. The `OrDefault` suffix only changes the empty case. So `users.SingleOrDefault(u => u.Id == 999)` on an empty list returns `null`, but `users.SingleOrDefault(u => u.Id > 0)` on a list of ten users throws. You must understand this clearly and pin it down with tests.
- **ElementAt vs ElementAtOrDefault:** `ElementAt` throws `ArgumentOutOfRangeException` (not `InvalidOperationException`!) on a negative index or out-of-range access. This is a different exception family and must be caught separately if you write a general handler. With .NET 6+ there is an overload accepting `Index` and `Range`: `items.ElementAt(^1)` is the last element, `items.ElementAt(0)` the first. `ElementAtOrDefault` returns `default` instead of throwing, which is convenient for "soft" indexing.
- **Performance:** `First` and `ElementAt` run in O(1) on `IList<T>` (including arrays and `List<T>`) because LINQ casts to `IList<T>` and uses the indexer. On a general `IEnumerable<T>` (for example the result of `Where` or `Select`) they are O(n). If the repository stores `IReadOnlyList<User>`, `First` and `ElementAt` are fast; if you pass an `IEnumerable<User>` from a deferred query, every call enumerates again. `Single` always enumerates up to the second match, so it is more expensive than `First` even on `IList`.
- **IQueryable difference:** in EF Core `First`, `Single` and `ElementAt` are translated to SQL: `First` becomes `TOP 1`, `Single` becomes `TOP 2` with a client-side check, `ElementAt` becomes `OFFSET ... FETCH NEXT`. On `IQueryable`, `SingleOrDefault` still throws on the client when more than one row is materialized, because the uniqueness check happens in memory. This is a typical source of surprising exceptions in APIs over a database.
- **Nullable contexts:** under `<Nullable>enable</Nullable>`, `FirstOrDefault` for a reference type returns `T?`, and the compiler forces you to handle `null`. For a value type `FirstOrDefault` returns `T` without `?`, and `default(int) == 0` passes without a warning — that is exactly the trap. Use `int?` or explicit checks.

#### Acceptance criteria

- [ ] The `UserAccess` project builds under .NET 8 with `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` and no errors.
- [ ] The `UserAccess.Tests` project builds and `dotnet test` passes with at least ten green tests.
- [ ] `User` is a `sealed record` with fields `Id`, `Email`, `Role`, `RegisteredAt`.
- [ ] `UserRepository` stores `IReadOnlyList<User>` and does not expose a mutable list.
- [ ] `GetById` uses `Single` and throws `InvalidOperationException` on absence or duplicate.
- [ ] `FindById` uses `SingleOrDefault` and returns `null` on absence; the comment notes that a duplicate still throws.
- [ ] `GetByEmail` uses `Single` with a justification of email uniqueness.
- [ ] `FindAdmin` uses `FirstOrDefault(u => u.Role == "admin")` and returns `User?`.
- [ ] `GetFirstRegistered` orders by `RegisteredAt` and uses `First`; throws on an empty list.
- [ ] `GetByIndex` uses `ElementAtOrDefault` and returns `User?`.
- [ ] `GetLastByIndex` uses `ElementAt(^1)` and throws `InvalidOperationException` on an empty list.
- [ ] All signatures correctly reflect nullable semantics (`User?` for "may be missing", `User` for "throws").
- [ ] `Program.cs` demonstrates both `try/catch (InvalidOperationException)` and `try/catch (ArgumentOutOfRangeException)`.
- [ ] There is no `Where(...).First(...)` anywhere — predicate overloads are used throughout.
- [ ] Every public method has XML comments naming the operator and exception semantics.

#### Hints (no direct answer)

- Think about which operator gives a strict uniqueness guarantee — that is the hint for `GetById` and `GetByEmail`.
- Recall that `ElementAtOrDefault` differs from `ElementAt` in the exception type (more precisely, in its absence). Where is that useful? Where going out of bounds is a normal scenario.
- For "first administrator", duplicate admins are acceptable — so a strict operator is not needed.
- For the last element, `ElementAt(^1)` reads better than `Last()`, but behaves equivalently. What is the difference then? In explicit positional semantics.
- If you want to return an `int` result from `FirstOrDefault` and still distinguish "no element" from "element with value 0", consider casting to `int?` or projecting to `int?` before `FirstOrDefault`.

#### Reference solution walk-through

```csharp
// C# 12 / .NET 8 — UserAccess / UserRepository.cs
// Homework M08-L07 solution

using System;
using System.Collections.Generic;
using System.Linq;

namespace UserAccess;

/// <summary>
/// System user. Email is unique within the dataset.
/// </summary>
public sealed record User(int Id, string Email, string Role, DateTime RegisteredAt);

/// <summary>
/// In-memory user repository. Demonstrates the choice of
/// First / Single / ElementAt and their OrDefault variants by business semantics.
/// </summary>
public sealed class UserRepository
{
    private readonly IReadOnlyList<User> _users;

    public UserRepository(IReadOnlyList<User> users) => _users = users;

    /// <summary>
    /// Uses Single: a missing or duplicated Id is a data error.
    /// Throws InvalidOperationException when Id is missing or duplicated.
    /// </summary>
    public User GetById(int id) =>
        _users.Single(u => u.Id == id);

    /// <summary>
    /// Uses SingleOrDefault: returns null when missing,
    /// but a duplicate Id still throws InvalidOperationException.
    /// </summary>
    public User? FindById(int id) =>
        _users.SingleOrDefault(u => u.Id == id);

    /// <summary>
    /// Single: email is unique, a duplicate breaks an invariant.
    /// </summary>
    public User GetByEmail(string email) =>
        _users.Single(u => u.Email == email);

    /// <summary>
    /// FirstOrDefault: multiple admins are allowed, we want the first one.
    /// </summary>
    public User? FindAdmin() =>
        _users.FirstOrDefault(u => u.Role == "admin");

    /// <summary>
    /// Order by RegisteredAt then First; an empty list is a config error.
    /// </summary>
    public User GetFirstRegistered() =>
        _users.OrderBy(u => u.RegisteredAt).First();

    /// <summary>
    /// ElementAtOrDefault: soft indexing, null when out of range.
    /// </summary>
    public User? GetByIndex(int index) =>
        _users.ElementAtOrDefault(index);

    /// <summary>
    /// ElementAt(^1): last element via Index (.NET 6+).
    /// Throws InvalidOperationException on an empty list.
    /// </summary>
    public User GetLastByIndex() =>
        _users.ElementAt(^1);
}
```

Line-by-line walk-through. `GetById` uses `Single` because a missing Id in the system is not a "maybe yes, maybe no" situation but a genuine error: either the data is absent or there is too much of it (a duplicate key). `Single` catches both cases with a single exception. `FindById` is the "soft" version: the caller wants to know whether a user exists, and `null` is a valid answer, but a duplicate Id still throws because `SingleOrDefault` distinguishes "empty" from "too many". The comment makes this explicit, so the next developer does not assume the method is "always safe". `GetByEmail` mirrors `GetById` because email is also an invariant: duplicate emails in the system are an import bug, and it is better to fail loudly than to silently return one of the duplicates.

`FindAdmin` uses `FirstOrDefault` because there may be several administrators and any first one will do. Duplicates are acceptable here, so `Single` would be harmful — it would throw false exceptions in systems with two or more admins. `GetFirstRegistered` sorts first, then takes `First`: an empty list is a configuration error (the repository must not be empty in production), so `FirstOrDefault` is unnecessary. Sorting turns the `IReadOnlyList` into an `IOrderedEnumerable`, so the subsequent `First` costs O(n log n) for the sort plus O(1) for the first element.

`GetByIndex` uses `ElementAtOrDefault` because going out of bounds is a normal scenario for pagination and client requests: returning `null` is better than throwing `ArgumentOutOfRangeException`. `GetLastByIndex` deliberately uses `ElementAt(^1)` to demonstrate the .NET 6+ `Index` syntax: `^1` means "one from the end". On an empty list `ElementAt(^1)` throws `InvalidOperationException` (rather than the `ArgumentOutOfRangeException` you get from a numeric index out of range — a subtle difference between the overloads). Every method returns `User?` exactly when `null` is a valid result, and `User` without `?` when absence is an error; this way the nullable context forces correct handling on the caller.

#### Going deeper (bonus)

1. Implement a variant of `GetById` against `IQueryable<User>` (simulating EF Core): write a `FakeUserQuery` class and show that `SingleOrDefault` still throws on the client after materialization when there is a duplicate. Explain why.
2. Add a method `bool TryGetById(int id, out User user)` that internally uses `Where(...).Take(2)` and a manual check, to avoid the double enumeration of `Single`. Compare the performance against `SingleOrDefault` on large lists with `BenchmarkDotNet`.
3. Show that a `FirstOrDefault` returning `int` for a value type may yield `0` as a valid value. Implement `int? FindAgeByEmail(string email)` via `Select(u => (int?)u.Age).FirstOrDefault(...)` to distinguish "no element" from "age 0".
4. Write a method that accepts `IEnumerable<User>` (not `IReadOnlyList`), and use `BenchmarkDotNet` to show the O(1) vs O(n) difference for `First` and `ElementAt` depending on the actual source type.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Проект UserAccess собирается под .NET 8 с TreatWarningsAsErrors.
- [ ] (RU) Минимум 10 тестов в UserAccess.Tests проходят зелёно.
- [ ] (RU) Все методы используют правильные операторы с XML-комментариями.
- [ ] (RU) Нет `Where(...).First(...)`, используются перегрузки с предикатом.
- [ ] (RU) Nullable-семантика соблюдена в сигнатурах.
- [ ] (EN) The UserAccess project builds under .NET 8 with TreatWarningsAsErrors.
- [ ] (EN) At least 10 tests in UserAccess.Tests pass green.
- [ ] (EN) All methods use the correct operators with XML comments.
- [ ] (EN) No `Where(...).First(...)`; predicate overloads are used.
- [ ] (EN) Nullable semantics are respected in signatures.

#### Ресурсы / Resources
- [Microsoft Learn — Enumerable.First](https://learn.microsoft.com/dotnet/api/system.linq.enumerable.first)
- [Microsoft Learn — Enumerable.FirstOrDefault](https://learn.microsoft.com/dotnet/api/system.linq.enumerable.firstordefault)
- [Microsoft Learn — Enumerable.Single](https://learn.microsoft.com/dotnet/api/system.linq.enumerable.single)
- [Microsoft Learn — Enumerable.SingleOrDefault](https://learn.microsoft.com/dotnet/api/system.linq.enumerable.singleordefault)
- [Microsoft Learn — Enumerable.ElementAt](https://learn.microsoft.com/dotnet/api/system.linq.enumerable.elementat)
- [Microsoft Learn — Enumerable.ElementAtOrDefault](https://learn.microsoft.com/dotnet/api/system.linq.enumerable.elementatordefault)
- [Microsoft Learn — Index struct](https://learn.microsoft.com/dotnet/api/system.index)
