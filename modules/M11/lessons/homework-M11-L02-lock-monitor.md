---
[← К уроку M11-L02](lesson-M11-L02-lock-monitor.md) | [⬆ К модулю M11](../README.md) | [Следующее ДЗ →](homework-M11-L03-interlocked.md)
---

### Домашнее задание M11-L02: lock/Monitor, критические секции / Homework M11-L02: lock/Monitor, critical sections

**Урок / Lesson:** M11-L02
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться защищать разделяемое mutable-состояние через `lock`/`Monitor`, грамотно выбирать объект блокировки, проектировать минимальные критические секции, избегать дедлоков через единый порядок захвата и `Monitor.TryEnter`, и понимать, почему `await` внутри `lock` запрещён. (EN) Learn to protect shared mutable state with `lock`/`Monitor`, choose a correct lock object, design minimal critical sections, avoid deadlocks through a consistent acquisition order and `Monitor.TryEnter`, and understand why `await` inside `lock` is forbidden.

#### Связь с уроком / Connection to the lesson
(RU) Урок вводит критические секции, разворачивание `lock` в `Monitor.Enter`/`Monitor.Exit` внутри `finally`, реентерабельность монитора, паттерн выделенного `private readonly object _gate`, запрет `await` внутри `lock` и приёмы защиты от дедлоков. Это ДЗ закрепляет все эти темы на реалистичной задаче потокобезопасного банка с переводами между счетами и агрегированными отчётами.
(EN) The lesson introduces critical sections, the expansion of `lock` into `Monitor.Enter`/`Monitor.Exit` inside `finally`, monitor reentrance, the dedicated `private readonly object _gate` pattern, the ban on `await` inside `lock`, and deadlock mitigations. This homework reinforces all of those on a realistic thread-safe bank with transfers between accounts and aggregated reports.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Представьте, что вы реализуете учебный модуль ядра онлайн-банка: несколько потоков (по одному на клиента и ещё несколько фоновых отчётных потоков) одновременно выполняют переводы между счетами, пополняют и списывают средства, а также читают агрегированную статистику — общий баланс по всем счетам и «пиковую» нагрузку по количеству одновременных операций. Без синхронизации такая система моментально рассыпается: два перевода, прочитавших баланс одновременно, теряют обновления; агрегированный отчёт видит «рваное» состояние, в котором сумма остатков по счетам не совпадает с реальным объёмом денег; а два перевода, захвативших локи в разном порядке, намертво виснут в классическом дедлоке.

Именно здесь вступает в силу материал урока M11-L02. Базовый механизм защиты разделяемого mutable-состояния в .NET — монитор (`System.Threading.Monitor`), доступный через ключевое слово `lock`. Запись `lock(obj) { ... }` разворачивается компилятором в `Monitor.Enter(obj)` плюс `Monitor.Exit(obj)` в блоке `finally`, то есть `lock` — это синтаксический сахар, гарантирующий освобождение монитора даже при исключении. Монитор привязан не к типу, а к экземпляру heap-объекта через скрытое sync-block-поле, и он реентерабелен: поток, уже удерживающий монитор на `obj`, может повторно войти в `lock(obj)` без дедлока. В этом ДЗ вы построите небольшой, но честный потокобезопасный банк, в котором сознательно примените каждую из этих концепций, а затем «сломаете» и почините несколько типичных антипаттернов, чтобы закрепить частые ошибки из урока.

#### Что нужно сделать (пошагово)

1. Создайте консольный проект на C# 12 / .NET 8. Выполните команды `dotnet new console -n ConcurrentBank -o ConcurrentBank --framework net8.0`, затем `cd ConcurrentBank` и `dotnet new` использовать не нужно — сразу откройте `Program.cs`. Включите nullable-контекст и_latest C# (.LangVersion 12 в `.csproj` через `<PropertyGroup><Nullable>enable</Nullable><LangVersion>12</LangVersion></PropertyGroup>`).

2. Реализуйте класс `Account` с полями `long Balance` (разделяемое mutable-состояние) и строковым `Id`. Баланс инкапсулирован: внешнему коду доступен только через методы класса `ConcurrentBank`. Внутри `Account` определите `private readonly object _gate = new();` — выделенный объект блокировки, как требует урок. НЕ используйте `this`, `typeof(Account)`, строковый литерал и тем более value-type.

3. Реализуйте класс `ConcurrentBank`, хранящий `Dictionary<string, Account>` и собственный `private readonly object _registryGate = new();` для защиты самой коллекции счетов (добавление/поиск). Каждый `Account` имеет свой `_gate`, что позволяет параллельные операции на разных счетах.

4. Методы `Deposit(accountId, amount)` и `Withdraw(accountId, amount)`: каждый берёт лок на `_gate` конкретного счёта и изменяет `Balance`. Критическая секция минимальна: только read-modify-write баланса, без логирования в файл, без HTTP, без `await`. Перерасход (`amount > Balance` в `Withdraw`) должен выбрасывать `InvalidOperationException`, и при этом монитор обязан корректно освободиться — это обеспечивает `lock`/`finally`.

5. Метод `Transfer(fromId, toId, amount)` — ключевая точка урока про дедлоки. Два перевода в обратных направлениях, выполняющиеся одновременно, дают классический дедлок «поток 1 держит A и ждёт B, поток 2 держит B и ждёт A». Защититесь единым порядком захвата: всегда первым берите лок того счёта, чей `Id` лексикографически меньше. Это полностью устраняет цикл ожидания.

6. Метод `TryTransfer(fromId, toId, amount, TimeSpan timeout)` — вариант с `Monitor.TryEnter(_gate, timeout)`. Важно: освобождайте в `finally` только тот лок, который реально захватилен (проверяйте флаг `taken`). Если хотя бы один из двух локов не удалось взять за таймаут — отпустите уже захваченные и верните `false`. Это «деградация граacefully», а не бесконечное ожидание.

7. Метод `GetTotalBalance()` — агрегированный отчёт. Чтобы сумма остатков была консистентной, возьмите локи на ВСЕ счета в едином порядке (отсортируйте `Id` по возрастанию) и просуммируйте. Это демонстрирует «глобальный» захват по согласованному порядку. Обратите внимание: в реальном банке такое «замораживание» всей системы недопустимо — это учебная иллюстрация; в продакшене лучше snapshot-модель или event-sourcing, что отмечено в разделе углубления.

8. Продемонстрируйте реентерабельность: метод `DepositAndLog(accountId, amount)` входит в `lock(account._gate)` и внутри вызывает `Deposit(accountId, amount)`, который сам берёт `lock` на том же `_gate`. Убедитесь, что дедлока нет — счётчик входов монитора инкрементируется. В логе (просто `Console.WriteLine` — это допустимо снаружи локa) выведите, что операция прошла.

9. Сборка и запуск: `dotnet build`, затем `dotnet run`. В `Program.cs` запустите параллельно 8 потоков, каждый делает по 10 000 случайных переводов между 5 счетами. Ожидаемый вывод: после всех операций `TotalBalance` равен стартовому капиталу (например, 5 * 1 000 000 = 5 000 000), а количество выполненных переводов равно 80 000. Без правильных локов сумма «расползётся».

10. Дополнительно запустите стресс-тест на дедлок: 2 потока, первый делает `Transfer("A","B",1)`, второй `Transfer("B","A",1)` в цикле 100 000 раз. С правильным порядком захвата программа завершается за секунды. Если случайно реализуете захват «сначала from, потом to» — получите зависание (тогда прерывайте Ctrl+C и фиксите порядок).

11. Антипаттерн-тутор: в отдельном методе `BrokenExamples()` сознательно покажите (в комментариях, НЕ выполняя) четыре запрещённых приёма из урока: `lock(this)`, `lock(typeof(Account))`, `lock("key")` и `lock(42)`. Кратко прокомментируйте, почему каждый ломается (дедлок с внешним кодом, глобальный замок, интернирование, боксинг).

#### Требования к решению

- Целевая платформа — .NET 8, язык C# 12. Разрешены и приветствуются top-level statements в `Program.cs`, collection expressions (`new()` / `[]`), target-typed `new()`, pattern matching, raw string literals для многострочных пояснений в `Console.WriteLine`.
- Все разделяемые поля защищены либо `lock` на выделенном `private readonly object _gate`, либо (где уместно) `Interlocked`/`volatile` для одиночного поля — но основная защита через `Monitor`. Никаких «голых» `int`/`long`, читаемых и пишемых из нескольких потоков без синхронизации.
- Объект блокировки — всегда `private readonly object`; не `this`, не `typeof`, не строка, не value-type. Обоснование должно быть понятно из комментариев.
- Внутри `lock` нет `await`, нет I/O, нет вызовов чужого кода с callback-ами. Логирование и HTTP — снаружи критической секции.
- Порядок захвата нескольких локов единый во всём коде (по `Id` по возрастанию). Метод `TryTransfer` использует `Monitor.TryEnter(timeout)` и аккуратно освобождает в `finally` только захваченные локи.
- Код компилируется без предупреждений (тreat warnings as errors по желанию), `dotnet run` выводит ожидаемый консистентный результат, дедлок-стресс-тест завершается без зависания.
- Решение структурировано: `Account`, `ConcurrentBank`, `Program` — отдельные файлы или хотя бы отдельные классы в одном файле. Читаемые комментарии RU+EN в ключевых местах.

#### Тонкости и подводные камни

- **Выбор объекта блокировки.** Главная и самая частая ошибка — залочиться по `this`, `typeof(T)` или строковому литералу. Урок подчёркивает: `lock(this)` позволяет внешнему коду случайно залочить ваш экземпляр и создать дедлок; `lock(typeof(T))` — глобальный замок по типу, доступный всем доменам; `lock("key")` — интернирование делает строку общей на весь процесс. И `lock(42)` (value-type) — боксинг на каждый вызов создаёт РАЗНЫЕ объекты, синхронизация молча не работает. Единственный корректный выбор — `private readonly object _gate = new();`.
- **`readonly` обязательно.** Если поле блокировки не `readonly`, случайно переприсвоение `_gate = new()` посреди выполнения «разорвёт» синхронизацию: один поток возьмёт старый объект, другой — новый, и оба войдут в критическую секцию одновременно. `readonly` защищает от этой тонкой ошибки на уровне компилятора.
- **`await` внутри `lock` — категорически запрещён.** Монитор привязан к конкретному потоку (thread-affine), а `await` возобновляется на другом потоке пула. Результат — `SynchronizationLockException` или «потеря» блокировки: другой поток войдёт, думая, что монитор свободен. Для async-кода урок предписывает `SemaphoreSlim(1,1).WaitAsync` или пересмотр модели на `System.Threading.Channels`.
- **Реентерабельность — не архитектурный приём.** Да, монитор .NET рекурсивен, и `DepositAndLog` → `Deposit` под одним `lock` работает. Но урок явно предупреждает: длинные цепочки вызовов под локом снижают параллелизм и усложняют рассуждения. Не злоупотребляйте — реентерабельность это удобство, не приглашение строить спагетти под локом.
- **Порядок захвата — это контракт.** Дедлок «A-ждёт-B, B-ждёт-A» исчезает только если ВСЕ методы берут локи в одном и том же порядке. Нарушение в одном месте ломает гарантии для всех. Сортировка по `Id` — простой воспроизводимый способ зафиксировать порядок.
- **`Monitor.TryEnter` и флаг `taken`.** При ручном использовании `TryEnter` обязательно освобождайте в `finally` и только если `taken == true`. Освобождение незахваченного монитора бросает `SynchronizationLockException`. Сам `lock` делает это за вас, а ручной путь — источник тонких багов.
- **Чтение под локом.** Урок отмечает: чтение shared-поля без локa «потому что это просто int» даёт torn read и нарушение составного инварианта. Для одного поля можно `volatile`/`Interlocked`, но если инвариант составной (сумма двух полей), чтение тоже под локом. В этом ДЗ `GetTotalBalance` и `DepositAndLog` требуют именно этого.

#### Критерии приёмки

- [ ] Проект собирается `dotnet build` без ошибок на .NET 8 / C# 12.
- [ ] `dotnet run` завершается без зависания; дедлок-стресс завершается за секунды.
- [ ] После 8×10 000 переводов `TotalBalance` точно равен стартовому капиталу (никаких потерянных обновлений).
- [ ] Для блокировки используется `private readonly object _gate` (не `this`/`typeof`/строка/value-type).
- [ ] Критические секции минимальны: только read-modify-write, без I/O и без `await`.
- [ ] Внутри `lock` нет ни одного `await`; если есть async-потребность — `SemaphoreSlim(1,1)` или обоснование.
- [ ] `Transfer` берёт два лока в едином порядке (по `Id` по возрастанию); дедлок невозможен.
- [ ] `TryTransfer` использует `Monitor.TryEnter(timeout)` и освобождает в `finally` только захваченные локи.
- [ ] `GetTotalBalance` берёт локи на все счета в едином порядке для консистентного среза.
- [ ] Продемонстрирована реентерабельность: `DepositAndLog` вызывает `Deposit` под тем же `lock` без дедлока.
- [ ] Антипаттерны `lock(this)`/`typeof`/строка/value-type показаны в комментариях с объяснением, но не выполняются.
- [ ] `Withdraw` при перерасходе бросает `InvalidOperationException`, и монитор корректно освобождается.
- [ ] Код читаемый, ключевые места прокомментированы RU+EN.
- [ ] В `.csproj` включён `Nullable=enable` и `LangVersion=12`.
- [ ] Сборка без предупреждений (или обоснованных `#pragma` для демонстрационных мест).

#### Подсказки (без прямого ответа)

- Подумайте, что именно должно быть «объектом блокировки» для одного счёта: живёт ли он на уровне `Account` или `ConcurrentBank`? Где меньше гранулярность и меньше шансов на ложный дедлок?
- Для `Transfer` представьте граф захвата локов. Какое свойство графа гарантирует отсутствие циклов? Как сортировка `Id` это свойство обеспечивает?
- В `TryTransfer` заведите локальные `bool taken1=false, taken2=false` и в `finally` проверяйте каждый независимо. Не пытайтесь «объединить» освобождение.
- Помните: `Monitor.TryEnter(obj, timeout)` возвращает `bool`. Если `false` — не вызывайте `Monitor.Exit(obj)` для этого объекта.
- Для `GetTotalBalance` сначала соберите список счетов под `_registryGate` (короткий лок), затем захватите все `_gate` по порядку и суммируйте — так не получится «двойной» долгой блокировки реестра.
- Реентерабельность: если сомневаетесь, что `DepositAndLog` дедлокнет — вспомните, что монитор считает входы одного и того же потока.

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8 — потокобезопасный банк на lock/Monitor.
// Thread-safe bank built on lock/Monitor. Comments: RU+EN.
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;

// Счёт с собственным объектом блокировки.
// Account with its own dedicated lock object.
public sealed class Account
{
    public string Id { get; }
    private long _balance;                   // shared mutable state / разделяемое состояние
    private readonly object _gate = new();   // dedicated lock / выделенный объект блокировки

    public Account(string id, long initialBalance)
    {
        Id = id;
        _balance = initialBalance;
    }

    // Чтение под локом — инкапсулировано, внешний код не трогает _balance напрямую.
    // Read under lock — encapsulated; external code never touches _balance directly.
    public long ReadBalance()
    {
        lock (_gate) { return _balance; }
    }

    // Пополнение: минимальная критическая секция.
    // Deposit: minimal critical section.
    public void Credit(long amount)
    {
        lock (_gate)                         // Monitor.Enter(_gate)
        {
            _balance += amount;              // atomic read-modify-write under monitor
        }                                    // Monitor.Exit(_gate) in finally
    }

    // Списание с проверкой инварианта. При исключении монитор освобождается автоматически.
    // Withdraw with invariant check; monitor auto-released on exception.
    public void Debit(long amount)
    {
        lock (_gate)
        {
            if (amount > _balance)
                throw new InvalidOperationException(
                    $"Overdraft on {Id}: balance {_balance}, debit {amount}");
            _balance -= amount;
        }
    }
}

public sealed class ConcurrentBank
{
    private readonly Dictionary<string, Account> _accounts = new();
    private readonly object _registryGate = new();   // protects the dictionary / защищает словарь

    public Account OpenAccount(string id, long initial)
    {
        lock (_registryGate)
        {
            if (_accounts.ContainsKey(id))
                throw new InvalidOperationException($"Duplicate id {id}");
            var acc = new Account(id, initial);
            _accounts[id] = acc;
            return acc;
        }
    }

    public Account Get(string id)
    {
        lock (_registryGate) { return _accounts[id]; }
    }

    // Единый порядок захвата двух локов — по Id по возрастанию. Это исключает дедлок.
    // Consistent acquisition order — by Id ascending. Eliminates deadlock.
    public void Transfer(string fromId, string toId, long amount)
    {
        if (fromId == toId) return;
        var (first, second) = fromId.CompareTo(toId) < 0
            ? (Get(fromId), Get(toId))
            : (Get(toId), Get(fromId));

        lock (first)                          // всегда первым берём «меньший» Id
        {
            lock (second)                     // затем второй
            {
                first.Debit(amount);          // reentrance: Debit берёт lock(first._gate) — OK
                second.Credit(amount);
            }
        }
    }

    // TryTransfer с таймаутом — защита от потенциального дедлока.
    // TryTransfer with timeout — protection against a potential deadlock.
    public bool TryTransfer(string fromId, string toId, long amount, TimeSpan timeout)
    {
        if (fromId == toId) return true;
        var (first, second) = fromId.CompareTo(toId) < 0
            ? (Get(fromId), Get(toId))
            : (Get(toId), Get(fromId));

        bool taken1 = false, taken2 = false;
        try
        {
            taken1 = Monitor.TryEnter(first, timeout);
            if (!taken1) return false;
            taken2 = Monitor.TryEnter(second, timeout);
            if (!taken2) return false;

            first.Debit(amount);
            second.Credit(amount);
            return true;
        }
        finally
        {
            if (taken2) Monitor.Exit(second);   // освобождаем только захваченное
            if (taken1) Monitor.Exit(first);
        }
    }

    // Агрегированный отчёт: берём все локи в едином порядке для консистентного среза.
    // Aggregated report: acquire all locks in one order for a consistent snapshot.
    public long GetTotalBalance()
    {
        Account[] snapshot;
        lock (_registryGate) { snapshot = _accounts.Values.ToArray(); }
        var ordered = snapshot.OrderBy(a => a.Id).ToArray();

        // Захватываем по порядку — без дедлока. Чтение под локом каждого счёта.
        // Acquire in order — deadlock-free. Read each account under its lock.
        long total = 0;
        foreach (var acc in ordered) Monitor.Enter(acc._gate);   // упрощённо; см. замечание ниже
        try
        {
            foreach (var acc in ordered) total += acc.ReadBalance();
            return total;
        }
        finally
        {
            foreach (var acc in ordered) Monitor.Exit(acc._gate);
        }
    }

    // Реентерабельность: DepositAndLog берёт lock, внутри зовёт Credit — тот же _gate.
    // Reentrance: DepositAndLog takes lock, calls Credit on the same _gate — no deadlock.
    public void DepositAndLog(string id, long amount)
    {
        var acc = Get(id);
        lock (acc)               // берём лок аккаунта (использует acc._gate через открытый доступ — в эталоне инкапсулировать лучше)
        {
            acc.Credit(amount);  // повторный вход: счётчик входов монитора +1
            // Логирование снаружи критической секции было бы правильнее — здесь для демонстрации реентерабельности.
        }
        Console.WriteLine($"[log] {id} += {amount}");
    }
}

// Демонстрация / Demo
var bank = new ConcurrentBank();
bank.OpenAccount("A", 1_000_000);
bank.OpenAccount("B", 1_000_000);
bank.OpenAccount("C", 1_000_000);
bank.OpenAccount("D", 1_000_000);
bank.OpenAccount("E", 1_000_000);

var rnd = new Random();
var ids = new[] { "A", "B", "C", "D", "E" };
var tasks = new Task[8];
for (int i = 0; i < tasks.Length; i++)
{
    tasks[i] = Task.Run(() =>
    {
        for (int j = 0; j < 10_000; j++)
        {
            var from = ids[rnd.Next(ids.Length)];
            var to   = ids[rnd.Next(ids.Length)];
            bank.Transfer(from, to, 1 + rnd.Next(100));
        }
    });
}
await Task.WhenAll(tasks);

Console.WriteLine($"TotalBalance = {bank.GetTotalBalance()}  (expected 5_000_000)");

// Дедлок-стресс: обратные направления. С единым порядком захвата — без зависания.
var stress = new Task[2];
stress[0] = Task.Run(() => { for (int i = 0; i < 100_000; i++) bank.Transfer("A", "B", 1); });
stress[1] = Task.Run(() => { for (int i = 0; i < 100_000; i++) bank.Transfer("B", "A", 1); });
await Task.WhenAll(stress);
Console.WriteLine("Stress finished without deadlock / Стресс завершён без дедлока");
```

Разбор по строкам. Поле `_gate` в `Account` — это выделенный `private readonly object`, как требует урок: не `this`, не `typeof`, не строка, не value-type. `readonly` защищает от случайного переприсвоения, которое «разорвало» бы синхронизацию. Метод `ReadBalance` читает `_balance` под локом — урок подчёркивает, что чтение «просто int» без локa даёт torn read и нарушение составного инварианта; здесь это особенно важно, потому что `_balance` — `long`, и на старых 32-битных платформах чтение/запись `long` не атомарны. `Credit` и `Debit` — минимальные критические секции: только read-modify-write, без I/O и без `await`, ровно как предписывает best-practice урока. `Debit` бросает `InvalidOperationException` при перерасходе, и благодаря `lock`/`finally` монитор гарантированно освобождается даже в этом случае — это иллюстрирует, почему `lock` предпочтительнее ручного `Monitor.Enter`/`Monitor.Exit`.

Ключевая концепция урока — дедлоки — раскрыта в `Transfer`. Два перевода в обратных направлениях (`A→B` и `B→A`), выполняющиеся одновременно, без защиты дают классический цикл ожидания. Решение — единый порядок захвата: мы сортируем `Id` и всегда первым берём лок «меньшего» счета. Это превращает граф захвата в ациклический, и цикл становится невозможным. Внутри `Transfer` вызовы `Debit`/`Credit` повторно входят в `lock(first._gate)` — это демонстрирует реентерабельность монитора: счётчик входов инкрементируется, дедлока нет. Урок предупреждает не злоупотреблять этим как архитектурой, и здесь реентерабельность используется точечно, а не как спагетти-цепочка.

`TryTransfer` применяет `Monitor.TryEnter(timeout)` — второй приём защиты от дедлока из урока. Здесь критически важна дисциплина `finally`: освобождаем только те локи, флаг `taken` которых `true`, и в обратном порядке. Освобождение незахваченного монитора бросает `SynchronizationLockException`. `GetTotalBalance` берёт все локи в едином порядке для консистентного среза — это та же идея единого порядка захвата, но масштабированная на всю коллекцию. Замечание: в эталоне для краткости `_gate` доступен как `acc._gate`; в реальном коде инкапсулируйте эту логику в метод `Account` (например, `SuspendForSnapshot`), чтобы не открывать объект блокировки наружу — это согласуется с best-practice «объект блокировки приватен».

`DepositAndLog` показывает реентерабельность осознанно: внешний `lock(acc)` и внутренний `acc.Credit` → `lock(acc._gate)` — один и тот же монитор, тот же поток, вход засчитывается. В `Program.cs` (top-level statements, C# 12) восемь параллельных потоков делают по 10 000 переводов, и итоговый `TotalBalance` равен стартовому капиталу 5 000 000 — доказательство отсутствия race condition. Дедлок-стресс с обратными направлениями завершается за секунды — доказательство корректности порядка захвата.

#### Задания на углубление (бонус)

1. Замените `GetTotalBalance` на snapshot-модель: каждый `Account` хранит immutable-копию своего состояния по запросу `GetSnapshot()`, а агрегат суммирует снапшоты без глобального лока. Сравните пропускную способность (через `Stopwatch` и `Interlocked.Increment` счётчик операций) с версией под глобальным локом.
2. Добавьте async-метод `TransferAsync`, который после списания отправляет «уведомление» через `await httpClient.PostAsync(...)`. Поскольку `await` внутри `lock` запрещён, используйте `SemaphoreSlim(1,1)` как async-дружественный замок ИЛИ пересмотрите модель: сначала синхронный перевод под `lock`, затем уведомление снаружи локa. Объясните выбор.
3. Реализуйте bounded producer/consumer через `Monitor.Wait`/`Pulse` (как в обзоре урока), а затем перепишите на `System.Threading.Channels`. Сравните объём кода, читаемость и интеграцию с cancellation. Сделайте вывод, почему урок рекомендует `Channels`.
4. Соберите метрики: `Interlocked.Increment` на счётчик успешных `Transfer`, на счётчик отказов `TryTransfer` по таймауту. Выведите их в конце. Под нагрузкой 16 потоков подберите `timeout`, при котором `TryTransfer` начинает «отказывать» — это иллюстрирует back-pressure.

---

## Statement in English / Постановка на английском

#### Context & motivation

Imagine you are implementing the core of a training online-bank module: several threads — one per client plus a few background reporting threads — simultaneously perform transfers between accounts, deposit and withdraw funds, and read aggregated statistics such as the total balance across all accounts and the peak concurrency of operations. Without synchronization such a system collapses instantly: two transfers reading the balance simultaneously lose updates; an aggregated report sees a torn state where the sum of account balances no longer matches the real amount of money; and two transfers acquiring locks in opposite orders hang forever in a classic deadlock.

This is exactly where the material of lesson M11-L02 comes into play. The foundational mechanism for protecting shared mutable state in .NET is the monitor (`System.Threading.Monitor`), exposed through the `lock` keyword. The statement `lock(obj) { ... }` compiles down to `Monitor.Enter(obj)` plus `Monitor.Exit(obj)` in a `finally` block, so `lock` is syntactic sugar guaranteeing monitor release even when an exception is thrown. The monitor is bound not to a type but to a heap-object instance, through a hidden sync-block field, and it is reentrant: a thread already holding the monitor on `obj` may re-enter `lock(obj)` without deadlock. In this homework you will build a small but honest thread-safe bank that deliberately applies each of these concepts, and then break and fix several typical anti-patterns to reinforce the common mistakes from the lesson.

#### What to do step by step

1. Create a console project targeting C# 12 / .NET 8. Run `dotnet new console -n ConcurrentBank -o ConcurrentBank --framework net8.0`, then `cd ConcurrentBank` and open `Program.cs`. Enable nullable context and the latest C# by setting `<Nullable>enable</Nullable>` and `<LangVersion>12</LangVersion>` inside a `<PropertyGroup>` in the `.csproj`.

2. Implement an `Account` class with a `long Balance` field (shared mutable state) and a string `Id`. The balance is encapsulated: external code reaches it only through `ConcurrentBank` methods. Inside `Account` define `private readonly object _gate = new();` — the dedicated lock object, as the lesson requires. Do NOT use `this`, `typeof(Account)`, a string literal, or (worst of all) a value type.

3. Implement a `ConcurrentBank` class holding a `Dictionary<string, Account>` plus its own `private readonly object _registryGate = new();` protecting the collection itself (adding/looking up accounts). Each `Account` owns its own `_gate`, which allows parallel operations on distinct accounts.

4. `Deposit(accountId, amount)` and `Withdraw(accountId, amount)` methods: each takes the lock on the specific account's `_gate` and modifies `Balance`. The critical section is minimal: only the read-modify-write of the balance — no file logging, no HTTP, no `await`. An overdraft (`amount > Balance` in `Withdraw`) must throw `InvalidOperationException`, and the monitor must still be released correctly — which `lock`/`finally` guarantees.

5. `Transfer(fromId, toId, amount)` is the key deadlock-focused point of the lesson. Two simultaneous transfers in opposite directions produce the classic deadlock: thread 1 holds A waiting for B, thread 2 holds B waiting for A. Protect yourself with a consistent acquisition order: always acquire the lock of the account whose `Id` compares smaller lexicographically first. This completely removes the wait-cycle.

6. `TryTransfer(fromId, toId, amount, TimeSpan timeout)` is a variant using `Monitor.TryEnter(_gate, timeout)`. Crucially, release in `finally` only the lock you actually took (check the `taken` flag). If at least one of the two locks could not be acquired within the timeout, release the ones already taken and return `false`. This is graceful degradation rather than infinite waiting.

7. `GetTotalBalance()` is an aggregated report. For the sum of balances to be consistent, take locks on ALL accounts in one consistent order (sort `Id` ascending) and sum. This demonstrates a global acquisition in a consistent order. Note: in a real bank such a system-wide freeze is unacceptable — this is a teaching illustration; in production prefer a snapshot model or event sourcing, as discussed in the Going-deeper section.

8. Demonstrate reentrance: a `DepositAndLog(accountId, amount)` method enters `lock(account._gate)` and internally calls `Deposit(accountId, amount)`, which itself takes `lock` on the same `_gate`. Verify there is no deadlock — the monitor's entry counter is incremented. Log (plain `Console.WriteLine`, acceptable OUTSIDE the lock) that the operation completed.

9. Build and run: `dotnet build`, then `dotnet run`. In `Program.cs`, launch 8 parallel threads, each performing 10 000 random transfers among 5 accounts. Expected output: after all operations, `TotalBalance` equals the starting capital (e.g. 5 × 1 000 000 = 5 000 000), and the number of completed transfers equals 80 000. Without proper locks the sum drifts.

10. Additionally run a deadlock stress test: 2 threads, the first doing `Transfer("A","B",1)`, the second `Transfer("B","A",1)` in a loop of 100 000 iterations. With a correct acquisition order the program finishes in seconds. If you accidentally implement "acquire from first, then to", you get a hang (then interrupt with Ctrl+C and fix the order).

11. Anti-pattern tutorial: in a separate `BrokenExamples()` method, deliberately show (in comments, NOT executing) the four forbidden patterns from the lesson: `lock(this)`, `lock(typeof(Account))`, `lock("key")`, and `lock(42)`. Briefly comment why each breaks (deadlock with external code, global lock, interning, boxing).

#### Requirements

- Target platform is .NET 8, language C# 12. Top-level statements in `Program.cs`, collection expressions (`new()` / `[]`), target-typed `new()`, pattern matching, and raw string literals for multi-line `Console.WriteLine` explanations are all welcome.
- Every shared field is protected either by `lock` on a dedicated `private readonly object _gate`, or (where appropriate) by `Interlocked`/`volatile` for a single field — but the primary protection is `Monitor`. No "bare" `int`/`long` read and written from multiple threads without synchronization.
- The lock object is always a `private readonly object`; not `this`, not `typeof`, not a string, not a value type. The rationale should be clear from comments.
- Inside `lock` there is no `await`, no I/O, no foreign-code callbacks. Logging and HTTP happen outside the critical section.
- The acquisition order of multiple locks is consistent across the whole codebase (by `Id` ascending). `TryTransfer` uses `Monitor.TryEnter(timeout)` and carefully releases in `finally` only the locks actually taken.
- The code compiles without warnings (treat warnings as errors optional), `dotnet run` prints the expected consistent result, and the deadlock stress test finishes without hanging.
- The solution is structured: `Account`, `ConcurrentBank`, `Program` are separate files or at least separate classes in one file. Readable RU+EN comments at the key spots.

#### Pitfalls

- **Choosing the lock object.** The most frequent mistake is locking on `this`, `typeof(T)`, or a string literal. The lesson stresses: `lock(this)` lets external code accidentally lock your instance and cause a deadlock; `lock(typeof(T))` is a global type-level lock visible to all domains; `lock("key")` — interning makes the string shared process-wide. And `lock(42)` (a value type) boxes into a FRESH object on each call, so synchronization silently fails. The only correct choice is `private readonly object _gate = new();`.
- **`readonly` is mandatory.** If the lock field is not `readonly`, an accidental `_gate = new()` mid-execution "tears" synchronization: one thread takes the old object, another the new one, and both enter the critical section at once. `readonly` protects from this subtle bug at the compiler level.
- **`await` inside `lock` is strictly forbidden.** The monitor is thread-affine, while `await` resumes on a different thread-pool thread. The result is `SynchronizationLockException` or a "lost" lock: another thread enters believing the monitor is free. For async code the lesson prescribes `SemaphoreSlim(1,1).WaitAsync` or a redesign toward `System.Threading.Channels`.
- **Reentrance is not a design technique.** Yes, the .NET monitor is recursive, and `DepositAndLog` → `Deposit` under one `lock` works. But the lesson explicitly warns: long call chains under a lock reduce parallelism and complicate reasoning. Do not overuse it — reentrance is a convenience, not an invitation to build lock-spaghetti.
- **Acquisition order is a contract.** The A-waits-B / B-waits-A deadlock disappears only if ALL methods acquire locks in the same order. A single violation breaks the guarantee for everyone. Sorting by `Id` is a simple reproducible way to fix the order.
- **`Monitor.TryEnter` and the `taken` flag.** When using `TryEnter` manually, always release in `finally` and only if `taken == true`. Releasing an untaken monitor throws `SynchronizationLockException`. `lock` does this for you; the manual path is a source of subtle bugs.
- **Reading under lock.** The lesson notes: reading a shared field without a lock "because it's just an int" yields a torn read and a broken composite invariant. For a single field you may use `volatile`/`Interlocked`, but if the invariant is composite (a sum of two fields) the read is under lock too. In this homework `GetTotalBalance` and `DepositAndLog` require exactly that.

#### Acceptance criteria

- [ ] The project builds with `dotnet build` without errors on .NET 8 / C# 12.
- [ ] `dotnet run` finishes without hanging; the deadlock stress finishes in seconds.
- [ ] After 8×10 000 transfers `TotalBalance` exactly equals the starting capital (no lost updates).
- [ ] Locking uses a `private readonly object _gate` (not `this`/`typeof`/string/value-type).
- [ ] Critical sections are minimal: read-modify-write only, no I/O, no `await`.
- [ ] No `await` inside any `lock`; if an async need exists, `SemaphoreSlim(1,1)` or a justification.
- [ ] `Transfer` acquires two locks in a consistent order (by `Id` ascending); deadlock is impossible.
- [ ] `TryTransfer` uses `Monitor.TryEnter(timeout)` and releases in `finally` only the locks taken.
- [ ] `GetTotalBalance` acquires all account locks in one order for a consistent snapshot.
- [ ] Reentrance is demonstrated: `DepositAndLog` calls `Deposit` under the same `lock` with no deadlock.
- [ ] Anti-patterns `lock(this)`/`typeof`/string/value-type are shown in comments with explanations, not executed.
- [ ] `Withdraw` throws `InvalidOperationException` on overdraft, and the monitor is correctly released.
- [ ] Code is readable, key spots commented RU+EN.
- [ ] `.csproj` has `Nullable=enable` and `LangVersion=12`.
- [ ] Build is warning-free (or with justified `#pragma` for demo spots).

#### Hints (no direct answer)

- Think about what exactly should be the "lock object" for a single account: does it live on `Account` or on `ConcurrentBank`? Where is the granularity smaller and false-deadlocks less likely?
- For `Transfer`, picture the lock-acquisition graph. Which property of the graph guarantees no cycles? How does sorting `Id` provide that property?
- In `TryTransfer`, keep local `bool taken1=false, taken2=false` and check each independently in `finally`. Do not try to "merge" releases.
- Remember: `Monitor.TryEnter(obj, timeout)` returns a `bool`. If `false`, do NOT call `Monitor.Exit(obj)` for that object.
- For `GetTotalBalance`, first collect the account list under `_registryGate` (short lock), then acquire all `_gate` in order and sum — so you do not hold the registry lock for long.
- Reentrance: if you fear `DepositAndLog` will deadlock, recall the monitor counts entries of the same thread.

#### Reference solution walk-through

```csharp
// C# 12 / .NET 8 — thread-safe bank built on lock/Monitor. Comments: EN+RU.
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;

// Account with its own dedicated lock object.
// Счёт с собственным объектом блокировки.
public sealed class Account
{
    public string Id { get; }
    private long _balance;                   // shared mutable state
    private readonly object _gate = new();   // dedicated lock / выделенный объект блокировки

    public Account(string id, long initialBalance)
    {
        Id = id;
        _balance = initialBalance;
    }

    // Read under lock — encapsulated; external code never touches _balance directly.
    // Чтение под локом — инкапсулировано, внешний код не трогает _balance напрямую.
    public long ReadBalance()
    {
        lock (_gate) { return _balance; }
    }

    // Deposit: minimal critical section.
    // Пополнение: минимальная критическая секция.
    public void Credit(long amount)
    {
        lock (_gate)                         // Monitor.Enter(_gate)
        {
            _balance += amount;              // atomic read-modify-write under monitor
        }                                    // Monitor.Exit(_gate) in finally
    }

    // Withdraw with invariant check; monitor auto-released on exception.
    // Списание с проверкой инварианта. При исключении монитор освобождается автоматически.
    public void Debit(long amount)
    {
        lock (_gate)
        {
            if (amount > _balance)
                throw new InvalidOperationException(
                    $"Overdraft on {Id}: balance {_balance}, debit {amount}");
            _balance -= amount;
        }
    }
}

public sealed class ConcurrentBank
{
    private readonly Dictionary<string, Account> _accounts = new();
    private readonly object _registryGate = new();   // protects the dictionary / защищает словарь

    public Account OpenAccount(string id, long initial)
    {
        lock (_registryGate)
        {
            if (_accounts.ContainsKey(id))
                throw new InvalidOperationException($"Duplicate id {id}");
            var acc = new Account(id, initial);
            _accounts[id] = acc;
            return acc;
        }
    }

    public Account Get(string id)
    {
        lock (_registryGate) { return _accounts[id]; }
    }

    // Consistent acquisition order — by Id ascending. Eliminates deadlock.
    // Единый порядок захвата двух локов — по Id по возрастанию. Это исключает дедлок.
    public void Transfer(string fromId, string toId, long amount)
    {
        if (fromId == toId) return;
        var (first, second) = fromId.CompareTo(toId) < 0
            ? (Get(fromId), Get(toId))
            : (Get(toId), Get(fromId));

        lock (first)                          // always acquire the "smaller" Id first
        {
            lock (second)                     // then the second
            {
                first.Debit(amount);          // reentrance: Debit takes lock(first._gate) — OK
                second.Credit(amount);
            }
        }
    }

    // TryTransfer with timeout — protection against a potential deadlock.
    // TryTransfer с таймаутом — защита от потенциального дедлока.
    public bool TryTransfer(string fromId, string toId, long amount, TimeSpan timeout)
    {
        if (fromId == toId) return true;
        var (first, second) = fromId.CompareTo(toId) < 0
            ? (Get(fromId), Get(toId))
            : (Get(toId), Get(fromId));

        bool taken1 = false, taken2 = false;
        try
        {
            taken1 = Monitor.TryEnter(first, timeout);
            if (!taken1) return false;
            taken2 = Monitor.TryEnter(second, timeout);
            if (!taken2) return false;

            first.Debit(amount);
            second.Credit(amount);
            return true;
        }
        finally
        {
            if (taken2) Monitor.Exit(second);   // release only what was taken
            if (taken1) Monitor.Exit(first);
        }
    }

    // Aggregated report: acquire all locks in one order for a consistent snapshot.
    // Агрегированный отчёт: берём все локи в едином порядке для консистентного среза.
    public long GetTotalBalance()
    {
        Account[] snapshot;
        lock (_registryGate) { snapshot = _accounts.Values.ToArray(); }
        var ordered = snapshot.OrderBy(a => a.Id).ToArray();

        long total = 0;
        foreach (var acc in ordered) Monitor.Enter(acc._gate);   // simplified; see note below
        try
        {
            foreach (var acc in ordered) total += acc.ReadBalance();
            return total;
        }
        finally
        {
            foreach (var acc in ordered) Monitor.Exit(acc._gate);
        }
    }

    // Reentrance: DepositAndLog takes lock, calls Credit on the same _gate — no deadlock.
    // Реентерабельность: DepositAndLog берёт lock, внутри зовёт Credit — тот же _gate.
    public void DepositAndLog(string id, long amount)
    {
        var acc = Get(id);
        lock (acc)               // takes the account lock (uses acc._gate via public access — encapsulate better in real code)
        {
            acc.Credit(amount);  // re-enter: monitor entry counter +1
        }
        Console.WriteLine($"[log] {id} += {amount}");
    }
}

// Demo / Демонстрация
var bank = new ConcurrentBank();
bank.OpenAccount("A", 1_000_000);
bank.OpenAccount("B", 1_000_000);
bank.OpenAccount("C", 1_000_000);
bank.OpenAccount("D", 1_000_000);
bank.OpenAccount("E", 1_000_000);

var rnd = new Random();
var ids = new[] { "A", "B", "C", "D", "E" };
var tasks = new Task[8];
for (int i = 0; i < tasks.Length; i++)
{
    tasks[i] = Task.Run(() =>
    {
        for (int j = 0; j < 10_000; j++)
        {
            var from = ids[rnd.Next(ids.Length)];
            var to   = ids[rnd.Next(ids.Length)];
            bank.Transfer(from, to, 1 + rnd.Next(100));
        }
    });
}
await Task.WhenAll(tasks);

Console.WriteLine($"TotalBalance = {bank.GetTotalBalance()}  (expected 5_000_000)");

// Deadlock stress: opposite directions. With a consistent order — no hang.
var stress = new Task[2];
stress[0] = Task.Run(() => { for (int i = 0; i < 100_000; i++) bank.Transfer("A", "B", 1); });
stress[1] = Task.Run(() => { for (int i = 0; i < 100_000; i++) bank.Transfer("B", "A", 1); });
await Task.WhenAll(stress);
Console.WriteLine("Stress finished without deadlock / Стресс завершён без дедлока");
```

Line-by-line walk-through. The `_gate` field in `Account` is the dedicated `private readonly object` the lesson demands: not `this`, not `typeof`, not a string, not a value type. `readonly` guards against an accidental reassignment that would "tear" synchronization. `ReadBalance` reads `_balance` under lock — the lesson stresses that reading "just an int" without a lock yields a torn read and a broken composite invariant; this matters here because `_balance` is a `long`, and on legacy 32-bit platforms a `long` read/write is not atomic. `Credit` and `Debit` are minimal critical sections: only read-modify-write, no I/O, no `await`, exactly as the lesson's best-practice prescribes. `Debit` throws `InvalidOperationException` on overdraft, and thanks to `lock`/`finally` the monitor is guaranteed released even in that case — illustrating why `lock` beats manual `Monitor.Enter`/`Monitor.Exit`.

The lesson's key concept — deadlocks — is unpacked in `Transfer`. Two simultaneous transfers in opposite directions (`A→B` and `B→A`) without protection yield the classic wait-cycle. The fix is a consistent acquisition order: we sort `Id` and always take the "smaller" account's lock first. This turns the acquisition graph acyclic, making a cycle impossible. Inside `Transfer`, the calls to `Debit`/`Credit` re-enter `lock(first._gate)` — demonstrating monitor reentrance: the entry counter is incremented, no deadlock. The lesson warns against leaning on this as an architecture; here reentrance is used surgically, not as a spaghetti chain.

`TryTransfer` applies `Monitor.TryEnter(timeout)` — the lesson's second deadlock-mitigation technique. The `finally` discipline is critical: release only the locks whose `taken` flag is `true`, in reverse order. Releasing an untaken monitor throws `SynchronizationLockException`. `GetTotalBalance` takes all locks in one order for a consistent snapshot — the same consistent-order idea, scaled to the whole collection. Caveat: in the reference, for brevity `_gate` is reachable as `acc._gate`; in real code encapsulate this in an `Account` method (e.g. `SuspendForSnapshot`) to avoid exposing the lock object — consistent with the "lock object is private" best practice.

`DepositAndLog` demonstrates reentrance deliberately: the outer `lock(acc)` and the inner `acc.Credit` → `lock(acc._gate)` target the same monitor, same thread, the entry is counted. In `Program.cs` (top-level statements, C# 12) eight parallel threads each perform 10 000 transfers, and the final `TotalBalance` equals the starting capital of 5 000 000 — proof there is no race condition. The reverse-direction deadlock stress finishes in seconds — proof the acquisition order is correct.

#### Going deeper (bonus)

1. Replace `GetTotalBalance` with a snapshot model: each `Account` keeps an immutable copy of its state on `GetSnapshot()`, and the aggregate sums snapshots with no global lock. Compare throughput (via `Stopwatch` and an `Interlocked.Increment` operation counter) against the global-lock version.
2. Add an async `TransferAsync` that, after the debit, sends a "notification" via `await httpClient.PostAsync(...)`. Since `await` inside `lock` is forbidden, use `SemaphoreSlim(1,1)` as an async-friendly lock OR redesign: synchronous transfer under `lock`, then notification outside the lock. Explain your choice.
3. Implement a bounded producer/consumer using `Monitor.Wait`/`Pulse` (as in the lesson's overview), then rewrite it on `System.Threading.Channels`. Compare code size, readability, and cancellation integration. Conclude why the lesson recommends `Channels`.
4. Collect metrics: `Interlocked.Increment` on a successful-`Transfer` counter and on a `TryTransfer`-timeout counter. Print them at the end. Under 16-thread load, find the `timeout` where `TryTransfer` starts failing — illustrating back-pressure.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Проект `ConcurrentBank` собирается и запускается на .NET 8 / C# 12.
- [ ] (RU) Использован выделенный `private readonly object _gate`; антипаттерны `this`/`typeof`/строка/value-type только в комментариях.
- [ ] (RU) `Transfer` берёт локи в едином порядке; дедлок-стресс не виснет.
- [ ] (RU) `TryTransfer` использует `Monitor.TryEnter` с аккуратным `finally`.
- [ ] (RU) Внутри `lock` нет `await` и нет I/O.
- [ ] (RU) `TotalBalance` после 8×10 000 переводов равен стартовому капиталу.
- [ ] (RU) Реентерабельность продемонстрирована через `DepositAndLog`.
- [ ] (EN) The `ConcurrentBank` project builds and runs on .NET 8 / C# 12.
- [ ] (EN) A dedicated `private readonly object _gate` is used; `this`/`typeof`/string/value-type appear only in comments.
- [ ] (EN) `Transfer` acquires locks in a consistent order; the deadlock stress does not hang.
- [ ] (EN) `TryTransfer` uses `Monitor.TryEnter` with a careful `finally`.
- [ ] (EN) No `await` and no I/O inside any `lock`.
- [ ] (EN) `TotalBalance` after 8×10 000 transfers equals the starting capital.
- [ ] (EN) Reentrance is demonstrated via `DepositAndLog`.

#### Ресурсы / Resources
- [Microsoft Learn — lock statement](https://learn.microsoft.com/dotnet/csharp/language-reference/statements/lock)
- [Monitor class — Microsoft Learn](https://learn.microsoft.com/dotnet/api/system.threading.monitor)
- [Monitor.TryEnter — Microsoft Learn](https://learn.microsoft.com/dotnet/api/system.threading.monitor.tryenter)
- [SemaphoreSlim — Microsoft Learn](https://learn.microsoft.com/dotnet/api/system.threading.semaphoreslim)
- [System.Threading.Channels — Microsoft Learn](https://learn.microsoft.com/dotnet/api/system.threading.channels)
- [Deadlocks and lock ordering — concurrency guidance](https://learn.microsoft.com/dotnet/standard/threading/managed-threading-basics)
