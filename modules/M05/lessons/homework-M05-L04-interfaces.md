---
[← К уроку M05-L04](lesson-M05-L04-interfaces.md) | [⬆ К модулю M05](../README.md) | [Предыдущее ДЗ ←](homework-M05-L03-abstract.md) | [Следующее ДЗ →](homework-M05-L05-polymorphism-practice.md)
---

### Домашнее задание M05-L04: Интерфейсы, default interface methods, множественная реализация / Homework M05-L04: Interfaces, default interface methods, multiple implementation

**Урок / Lesson:** M05-L04
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться проектировать интерфейсы с единственной ответственностью, реализовывать их явно и неявно, безопасно расширять контракты через default interface methods, разрешать конфликты имён при множественной реализации и применять вариантность (`out`/`in`) для повторного использования кода. (EN) Learn to design single-responsibility interfaces, implement them implicitly and explicitly, safely extend contracts via default interface methods, resolve name conflicts under multiple implementation, and apply variance (`out`/`in`) to reuse code.

#### Связь с уроком / Connection to the lesson
(RU) Урок вводит интерфейс как «стандарт розетки»: контракт описывает **что** должен делать тип, а не **как**. В этом ДЗ вы построите именно такую систему каналов уведомлений, где разные реализации (Email, SMS, Push) подключаются к единому контракту `INotifier`, а DIM-методы позволяют дополнить контракт новыми возможностями без перекомпиляции старых реализаций. Задание целенаправленно провоцирует конфликт одноимённых методов `Log` в `INotifier` и `IAuditable`, чтобы вы применили явную реализацию и почувствовали разницу между поведением «по классу» и «по интерфейсу».
(EN) The lesson introduces the interface as a “socket standard”: a contract describes **what** a type must do, not **how**. In this homework you will build exactly such a notification-channel system, where different implementations (Email, SMS, Push) plug into a single `INotifier` contract, and DIM methods let you augment the contract with new capabilities without recompiling older implementations. The task deliberately provokes a name conflict on `Log` between `INotifier` and `IAuditable` so that you apply explicit implementation and feel the difference between class-typed and interface-typed behaviour.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Вы присоединились к команде, которая разрабатывает модуль уведомлений для корпоративной платформы заказов. Архитектор настаивает, чтобы ядро модуля зависело только от интерфейсов, а не от конкретных классов: это позволит добавлять новые каналы доставки (электронная почта, SMS, push, мессенджеры) и менять их реализации без перекомпиляции бизнес-логики. Однако команда столкнулась с тремя реальными проблемами, которые прямо перекликаются с темами урока M05-L04.

Во-первых, в интерфейс `INotifier` за годы существования добавилось множество методов, и каждое новое требование рискует сломать десятки существующих реализаций в других сборках. Команда хочет безопасно расширять контракт, не нарушая обратной совместимости, — именно для этого в C# 8 появились default interface methods. Во-вторых, в системе уже есть интерфейс `IAuditable`, у которого тоже есть метод `Log`. Если класс `EmailNotifier` реализует оба контракта одной неявной реализацией, поведение случайно сливается, и аудитные записи начинают уходить в почтовый лог вместо отдельной аудиторской очереди. В-третьих, у команды есть обобщённые фабрики и компараторы, которые хотелось бы переиспользовать для производных типов — без вариантности компилятор отказывается приводить `IProducer<string>` к `IProducer<object>`.

Цель задания — спроектировать и реализовать这个小 модуль так, чтобы он решал все три проблемы, демонстрировал корректную работу DIM только через переменную интерфейса, корректно разрешал конфликты имён через явную реализацию и показывал практическую пользу ковариантности и контравариантности. Каждое решение должно быть подкреплено работающей демонстрацией в `Program.cs`, которая выводит понятные строки и не требует внешних сервисов.

#### Что нужно сделать (пошагово)

1. Создайте новый проект консольного приложения под .NET 8 и C# 12. Выполните команду `dotnet new console -n Notifications.Core -o Notifications.Core --framework net8.0`, перейдите в папку проекта `cd Notifications.Core` и откройте файл `Notifications.Core.csproj`. Убедитесь, что свойство `<LangVersion>latest</LangVersion>` присутствует, и при необходимости добавьте его внутрь `<PropertyGroup>`, чтобы включить все возможности C# 12. Запустите `dotnet build` — оно должно завершиться без ошибок.

2. В файле `INotifier.cs` объявите интерфейс `INotifier` с обязательным методом `void Send(string recipient, string message)`. Добавьте в него три default interface method: `void Log(string message) => Send("system", $"[LOG] {message}");`, `void SendBatch(string recipient, IEnumerable<string> messages)`, который внутри цикла вызывает `Send`, и `void NotifyError(string recipient, Exception ex) => Send(recipient, $"[ERROR] {ex.Message}");`. Комментарии на русском и английском объясняют, что DIM доступны только через переменную интерфейса. Важно, чтобы интерфейс оставался узким по смыслу (одна ответственность — доставка уведомления), а DIM лишь расширяли его удобными обёртками.

3. В файле `IAuditable.cs` объявите второй интерфейс `IAuditable` с одноимённым методом `void Log(string message)` и свойством `DateTime LastChanged { get; }`. Это создаст конфликт имён, который требует явной реализации.

4. В файле `EmailNotifier.cs` создайте класс `EmailNotifier`, который реализует сразу два интерфейса `INotifier, IAuditable`. Метод `INotifier.Send` реализуйте неявно и публично — он пишет в `Console` строку вида `[Email → recipient] message` и обновляет `LastChanged`. Метод `INotifier.Log` наследуется как DIM и не переопределяется. Метод `IAuditable.Log` реализуйте **явно** через `void IAuditable.Log(string message)`, чтобы он форматировал сообщение как `[AUDIT @ timestamp] message` и тоже обновлял `LastChanged`. Тем самым класс чётко разделяет два поведения с одинаковым именем.

5. В файле `SmsNotifier.cs` создайте класс `SmsNotifier`, реализующий только `INotifier`. Он должен переопределить DIM `Log`, чтобы формат SMS отличался: `Send("sms-gateway", $"[SMS-LOG] {message}")`. Это покажет, что DIM можно переопределять, когда поведение по умолчанию не подходит. Добавьте класс `PushNotifier : INotifier`, который **не** переопределяет DIM и получает поведение по умолчанию — для контраста.

6. В файле `IProducer.cs` объявите ковариантный интерфейс `IProducer<out T>` с методом `T Produce()`. В файле `IComparator.cs` объявите контравариантный интерфейс `IComparator<in T>` с методом `int Compare(T x, T y)`. Реализуйте `StringProducer : IProducer<string>` и `LengthComparator : IComparator<string>`, а в `Program.cs` покажите, что `IProducer<string>` приводится к `IProducer<object>` (ковариантность), а `IComparator<object>` приводится к `IComparator<string>` (контравариантность) — оба приведения должны компилироваться и работать.

7. В `Program.cs` соберите список `List<INotifier>` с экземплярами `EmailNotifier`, `SmsNotifier` и `PushNotifier`. Пройдитесь по нему циклом `foreach` и вызовите `Send`, затем вызовите DIM `Log` и `SendBatch` через переменную интерфейса. Отдельно приведите `EmailNotifier` к `IAuditable` и вызовите `IAuditable.Log`, чтобы показать, что это другая реализация. Выведите `LastChanged`. Ожидаемый вывод содержит строки `[Email → alice] Привет`, `[SMS → bob] Код 1234`, `[AUDIT @ ...] Документ подписан`, `[LOG] Старт` и `LastChanged = ...`.

8. Запустите приложение командой `dotnet run --project Notifications.Core` и убедитесь, что вывод совпадает с ожидаемым. Сделайте скриншот или скопируйте вывод в файл `output.txt` в папке проекта.

#### Требования к решению

Решение должно компилироваться под .NET 8 без предупреждений уровня `error` и работать детерминированно (без случайных чисел, без обращения к сети и файловой системе за пределами `Console`). Все интерфейсы должны именоваться с префикса `I` и иметь одну ответственность: `INotifier` — доставка, `IAuditable` — след аудита, `IProducer<out T>` — производство значения, `IComparator<in T>` — сравнение. Класс `EmailNotifier` обязан реализовать конфликтующий метод `Log` из обоих интерфейсов раздельно: неявно для `INotifier` (через DIM или собственную реализацию) и явно для `IAuditable`. DIM методы должны вызываться только через переменную типа интерфейса — вызов через переменную класса должен намеренно закомментирован с пояснением, почему он не компилируется. Код должен использовать возможности C# 12 там, где это уместно: top-level statements в `Program.cs`, collection expressions для инициализации `List<INotifier>`, pattern matching при необходимости, raw string literals для многострочных сообщений в `EmailNotifier`. Решение должно сопровождаться коротким `README.md` с описанием структуры файлов. Запрещено использовать абстрактный класс вместо интерфейса там, где требуется именно контракт без состояния.

#### Тонкости и подводные камни

Главная ловушка DIM — попытка вызвать его через переменную типа класса. Если вы напишете `EmailNotifier e = new(); e.Log("x");`, компилятор не увидит метод `Log`, потому что DIM не становится членом класса: это поведение интерфейса. Нужно либо объявить переменную как `INotifier n = e;`, либо привести через `((INotifier)e).Log("x")`. В уроке прямо указано: «метод по умолчанию доступен только через переменную типа интерфейса». Вторая ловушка — попытка из DIM обратиться к полям класса. У интерфейса нет экземплярного состояния класса, поэтому DIM может вызывать только другие члены интерфейса. Если вам нужно значение из класса, объявите в интерфейсе абстрактное свойство и реализуйте его в классе.

Третья ловушка — слияние поведения при одноимённых методах. Если `EmailNotifier` реализует `INotifier.Log` и `IAuditable.Log` одним публичным методом `public void Log(string)`, оба контракта будут считать его своей реализацией, и аудиторский вызов уйдёт в почтовый лог. Явная реализация `void IAuditable.Log(string message)` решает проблему: такой член недоступен через класс, только через `(IAuditable)obj`. Помните, что явная реализация не имеет модификатора доступа — попытка поставить `public` приведёт к ошибке компиляции.

Четвёртая ловушка — путаница между `out` и `in`. Ковариантность `out` разрешает возврат более производного типа: `IProducer<string>` можно привести к `IProducer<object>`, потому что строку можно безопасно вернуть там, где ждут object. Контравариантность `in` работает в обратную сторону: `IComparator<object>` можно привести к `IComparator<string>`, потому что компаратор, умеющий сравнивать любые object, справится и со строками. Нарушение направления (пометить `out` параметр, передаваемый на вход) даёт ошибку компиляции. Наконец, не пытайтесь `new INotifier()` — интерфейс инстанцировать нельзя; создайте экземпляр реализующего класса и присвойте его переменной интерфейса.

#### Критерии приёмки

- [ ] Проект `Notifications.Core` собирается под .NET 8 без ошибок и предупреждений уровня error.
- [ ] В `Notifications.Core.csproj` явно указан `<LangVersion>latest</LangVersion>`.
- [ ] Интерфейс `INotifier` содержит обязательный `Send` и минимум три DIM.
- [ ] DIM-методы `Log`, `SendBatch`, `NotifyError` имеют тела и помечены как `virtual`-по-умолчанию (тело метода интерфейса).
- [ ] Класс `EmailNotifier` реализует `INotifier` и `IAuditable` одновременно.
- [ ] `IAuditable.Log` реализован **явно**, без модификатора доступа.
- [ ] Вызов `IAuditable.Log` через приведение даёт отличающийся от `INotifier.Log` вывод.
- [ ] Класс `SmsNotifier` переопределяет DIM `Log`, а `PushNotifier` — нет.
- [ ] В `Program.cs` показан вызов DIM через переменную интерфейса, а закомментированный вызов через класс поясняет ошибку.
- [ ] `IProducer<out T>` корректно приводится от `IProducer<string>` к `IProducer<object>`.
- [ ] `IComparator<in T>` корректно приводится от `IComparator<object>` к `IComparator<string>`.
- [ ] Использованы возможности C# 12: top-level statements, collection expressions, при необходимости raw strings.
- [ ] Вывод `dotnet run` содержит строки `[Email → alice]`, `[SMS → bob]`, `[AUDIT @`, `LastChanged =`.
- [ ] Имена всех интерфейсов начинаются с `I`.
- [ ] Файл `README.md` описывает структуру проекта и rationale выбора интерфейсов.

#### Подсказки (без прямого ответа)

- Вспомните метафору урока про «розетку»: `INotifier` — стандарт, реализация — прибор. Спросите себя, какие операции «розетка» обязана предоставить, а какие — удобные обёртки над ними.
- Для DIM-метода достаточно тела в объявлении интерфейса; компилятор сам сделает его виртуальным по умолчанию.
- Когда нужно вызвать конфликтующий метод, приведите объект к нужному интерфейсу: `((IAuditable)email).Log(...)`.
- Для вариантности помните мнемонику: `out` — на выход, возвращаемое значение, можно вернуть более конкретный тип; `in` — на вход, аргумент, можно принять более общий тип.
- Чтобы проверить, что DIM действительно недоступен через класс, попробуйте написать `email.Log("x")` и прочитайте сообщение компилятора.
- Для `SendBatch` удобно использовать цикл `foreach` внутри тела DIM — это безопасно, потому что `IEnumerable<string>` — это контракт самого интерфейса.

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8 — Эталонное решение ДЗ M05-L04
// Reference solution for homework M05-L04
// Темы: интерфейсы, DIM, явная реализация, множественная реализация, вариантность.
// Topics: interfaces, DIM, explicit implementation, multiple implementation, variance.

using System;
using System.Collections.Generic;

// --- INotifier: контракт доставки уведомления + DIM-обёртки ---
// INotifier: notification delivery contract + DIM wrappers.
public interface INotifier
{
    // Обязательный член — реализуется классом / Required member, implemented by the class.
    void Send(string recipient, string message);

    // DIM: логирует сообщение, отправляя его системному получателю.
    // DIM: logs a message by sending it to a system recipient.
    public void Log(string message) => Send("system", $"[LOG] {message}");

    // DIM: отправляет пачку сообщений / DIM: sends a batch of messages.
    public void SendBatch(string recipient, IEnumerable<string> messages)
    {
        foreach (var m in messages) Send(recipient, m);
    }

    // DIM: уведомляет об ошибке / DIM: notifies about an error.
    public void NotifyError(string recipient, Exception ex)
        => Send(recipient, $"[ERROR] {ex.Message}");
}

// --- IAuditable: контракт аудита с тем же именем Log, что создаёт конфликт ---
// IAuditable: audit contract with the same Log name, creating a conflict.
public interface IAuditable
{
    void Log(string message);          // другая семантика / different semantics
    DateTime LastChanged { get; }
}

// --- EmailNotifier: реализует оба интерфейса, разрешает конфликт явно ---
// EmailNotifier: implements both interfaces, resolves the conflict explicitly.
public class EmailNotifier : INotifier, IAuditable
{
    public DateTime LastChanged { get; private set; }

    // Неявная реализация INotifier.Send — доступна через класс и интерфейс.
    // Implicit INotifier.Send — visible through class and interface.
    public void Send(string recipient, string message)
    {
        Console.WriteLine($"[Email → {recipient}] {message}");
        LastChanged = DateTime.UtcNow;
    }

    // Явная реализация IAuditable.Log — доступна только через (IAuditable).
    // Explicit IAuditable.Log — visible only via (IAuditable).
    void IAuditable.Log(string message)
    {
        Console.WriteLine($"[AUDIT @ {DateTime.UtcNow:O}] {message}");
        LastChanged = DateTime.UtcNow;
    }
}

// --- SmsNotifier переопределяет DIM, PushNotifier — нет ---
// SmsNotifier overrides DIM, PushNotifier does not.
public class SmsNotifier : INotifier
{
    public void Send(string recipient, string message)
        => Console.WriteLine($"[SMS → {recipient}] {message}");

    // Переопределяем DIM под формат SMS / Override the DIM with SMS formatting.
    public void Log(string message) => Send("sms-gateway", $"[SMS-LOG] {message}");
}

public class PushNotifier : INotifier
{
    public void Send(string recipient, string message)
        => Console.WriteLine($"[PUSH → {recipient}] {message}");
    // Log и SendBatch берутся из интерфейса / Log and SendBatch come from the interface.
}

// --- Ковариантный интерфейс производителя / Covariant producer interface ---
public interface IProducer<out T>
{
    T Produce();
}

public class StringProducer : IProducer<string>
{
    public string Produce() => "hello";
}

// --- Контравариантный компаратор / Contravariant comparator ---
public interface IComparator<in T>
{
    int Compare(T x, T y);
}

public class LengthComparator : IComparator<string>
{
    public int Compare(string x, string y) => x.Length.CompareTo(y.Length);
}

// --- Program.cs (top-level statements, C# 12) ---
// Список уведомителей через collection expression / Notifier list via collection expression.
List<INotifier> notifiers = [new EmailNotifier(), new SmsNotifier(), new PushNotifier()];

foreach (var n in notifiers)
{
    n.Send("alice", "Привет / Hello");
    n.Log("Старт канала / Channel started");      // DIM через интерфейс / DIM via interface
    n.SendBatch("bob", ["Код 1234", "Не отвечать / Do not reply"]);
}

var email = new EmailNotifier();
((IAuditable)email).Log("Документ подписан / Document signed"); // другая реализация
Console.WriteLine($"LastChanged = {email.LastChanged:O}");

// Ковариантность / Covariance: string → object
IProducer<string> sp = new StringProducer();
IProducer<object> op = sp;          // допустимо благодаря out / legal due to out
Console.WriteLine(op.Produce());

// Контравариантность / Contravariance: object → string
IComparator<string> lenCmp = new LengthComparator();
// IComparator<object> можно привести к IComparator<string> благодаря in
IComparator<object> objCmp = new UniversalLengthComparator();
IComparator<string> strCmp = objCmp; // допустимо благодаря in / legal due to in
Console.WriteLine(strCmp.Compare("abc", "ab"));

public class UniversalLengthComparator : IComparator<object>
{
    public int Compare(object x, object y) =>
        (x?.ToString()?.Length ?? 0).CompareTo(y?.ToString()?.Length ?? 0);
}
```

Разбор по строкам. Объявление `public interface INotifier` следует соглашению урока: имя с `I`, одна ответственность — доставка уведомления. Обязательный метод `Send` — это «вилка прибора», без которой контракт не выполнен. Три DIM (`Log`, `SendBatch`, `NotifyError`) иллюстрируют главную мотивацию функции из урока: они добавляют удобные обёртки, не ломая существующие реализации, и доступны только через переменную интерфейса. Обратите внимание, что DIM не помечены модификатором `virtual` явно — тело в интерфейсе само по себе делает их виртуальными по умолчанию, как описано в теории.

Интерфейс `IAuditable` намеренно повторяет имя `Log` — это создаёт конфликт, упомянутый в разделе «Частые ошибки». Класс `EmailNotifier` реализует `INotifier.Send` неявно (доступно и через класс, и через интерфейс), а `IAuditable.Log` — явно, через `void IAuditable.Log(string)`, без модификатора доступа. Так поведение двух одноимённых методов разделено: вызов `email.Log("x")` (через DIM `INotifier.Log`) отправит лог как уведомление системному получателю, а `((IAuditable)email).Log("x")` напечатает аудиторскую строку с временной меткой. Это ровно сценарий из урока про `IDraw.Draw` и `ICard.Draw`.

`SmsNotifier` переопределяет DIM `Log`, показывая, что реализация может выбрать собственное поведение, если умолчание не подходит. `PushNotifier` не переопределяет DIM и получает поведение по умолчанию — для контраста и демонстрации того, что DIM необязателен для переопределения. В `Program.cs` список `notifiers` инициализирован через collection expression — новую возможность C# 12. Цикл `foreach` вызывает `Send`, `Log` и `SendBatch` через переменную `INotifier n`, что и даёт доступ к DIM. Если бы мы написали `email.Log("x")` без приведения, компилятор не увидел бы DIM — это проверяется закомментированным примером.

Ковариантный `IProducer<out T>` разрешает приведение `IProducer<string>` к `IProducer<object>`, потому что строку можно безопасно вернуть там, где ждут `object`. Контравариантный `IComparator<in T>` разрешает обратное приведение `IComparator<object>` к `IComparator<string>`, потому что компаратор любых object справится и со строками. `UniversalLengthComparator` реализует `IComparator<object>` и затем приводится к `IComparator<string>` — это демонстрирует практическую пользу вариантности. Все эти моменты прямо перекликаются с best practices урока: узкие интерфейсы, префикс `I`, DIM для дополнения, явная реализация для конфликтов и вариантность для переиспользования.

#### Задания на углубление (бонус)

1. Добавьте третий интерфейс `ITemplatedNotifier` с DIM `void SendTemplate(string recipient, string templateKey, IReadOnlyDictionary<string, string> args)`, который формирует сообщение из шаблона и вызывает `Send`. Сделайте так, чтобы `EmailNotifier` реализовывал и его, и убедитесь, что никаких конфликтов имён не возникает.
2. Расширьте `INotifier` новым DIM `void Retry(string recipient, string message, int attempts)`, который повторяет `Send` до успеха или исчерпания попыток, используя `try/catch`. Покажите, что старые реализации автоматически получают новое поведение без перекомпиляции их исходников.
3. Реализуйте контравариантный `IValidator<in T>` с методом `bool IsValid(T value)` и ковариантный `IFactory<out T>` с методом `T Create()`. Постройте пример, где один `IValidator<object>` валидирует строки и числа, а один `IFactory<string>` подставляется туда, где ждут `IFactory<object>`.
4. Напишите модульный тест с помощью xUnit, который проверяет, что `IAuditable.Log` и `INotifier.Log` у `EmailNotifier` дают разный вывод. Используйте `StringWriter` для перехвата `Console.Out`.

---

## Statement in English / Постановка на английском

#### Context & motivation

You have joined a team that develops the notification module of an enterprise ordering platform. The architect insists that the module core depend only on interfaces, never on concrete classes: this lets the team add new delivery channels (email, SMS, push, messengers) and swap implementations without recompiling business logic. The team has run into three real problems that map directly onto the topics of lesson M05-L04.

First, the `INotifier` interface has accumulated many members over the years, and every new requirement risks breaking dozens of existing implementations living in other assemblies. The team wants to extend the contract safely, without breaking backward compatibility — this is exactly the scenario that default interface methods were introduced for in C# 8. Second, the codebase already contains an `IAuditable` interface that also exposes a `Log` method. If the `EmailNotifier` class implements both contracts with a single implicit method, behaviour silently merges, and audit records start flowing into the email log instead of a dedicated audit queue. Third, the team owns generic factories and comparators that they would like to reuse for derived types — without variance the compiler refuses to convert `IProducer<string>` to `IProducer<object>`.

The goal of this assignment is to design and implement this small module so that it solves all three problems, demonstrates that DIM methods are only callable through an interface-typed variable, correctly resolves name conflicts through explicit implementation, and shows the practical benefit of covariance and contravariance. Every decision must be backed by a working demonstration in `Program.cs` that prints understandable lines and requires no external services.

#### What to do step by step

1. Create a new console application project targeting .NET 8 and C# 12. Run `dotnet new console -n Notifications.Core -o Notifications.Core --framework net8.0`, descend into the project folder with `cd Notifications.Core`, and open `Notifications.Core.csproj`. Ensure that the property `<LangVersion>latest</LangVersion>` is present, and add it inside `<PropertyGroup>` if needed so that all C# 12 features are enabled. Run `dotnet build` and confirm that it completes without errors.

2. In `INotifier.cs` declare the interface `INotifier` with a required method `void Send(string recipient, string message)`. Add three default interface methods to it: `void Log(string message) => Send("system", $"[LOG] {message}");`, `void SendBatch(string recipient, IEnumerable<string> messages)` that calls `Send` in a loop, and `void NotifyError(string recipient, Exception ex) => Send(recipient, $"[ERROR] {ex.Message}");`. Bilingual comments explain that DIMs are only reachable through an interface-typed variable. Keep the interface narrow in meaning (a single responsibility — notification delivery); DIMs should merely add convenient wrappers.

3. In `IAuditable.cs` declare a second interface `IAuditable` with a method named the same way, `void Log(string message)`, and a property `DateTime LastChanged { get; }`. This creates a name conflict that demands explicit implementation.

4. In `EmailNotifier.cs` create a class `EmailNotifier` that implements both interfaces, `INotifier, IAuditable`. Implement `INotifier.Send` implicitly and publicly — it writes a line like `[Email → recipient] message` to `Console` and updates `LastChanged`. The `INotifier.Log` method is inherited as a DIM and is not overridden. Implement `IAuditable.Log` **explicitly** as `void IAuditable.Log(string message)`, formatting the message as `[AUDIT @ timestamp] message` and also updating `LastChanged`. The class now cleanly separates two behaviours that share the same name.

5. In `SmsNotifier.cs` create a class `SmsNotifier` that implements only `INotifier`. It should override the DIM `Log` so that the SMS format differs: `Send("sms-gateway", $"[SMS-LOG] {message}")`. This shows that a DIM can be overridden when the default behaviour does not fit. Add a class `PushNotifier : INotifier` that does **not** override the DIM and therefore inherits the default behaviour — for contrast.

6. In `IProducer.cs` declare a covariant interface `IProducer<out T>` with a method `T Produce()`. In `IComparator.cs` declare a contravariant interface `IComparator<in T>` with a method `int Compare(T x, T y)`. Implement `StringProducer : IProducer<string>` and `LengthComparator : IComparator<string>`, and in `Program.cs` show that `IProducer<string>` converts to `IProducer<object>` (covariance) and that `IComparator<object>` converts to `IComparator<string>` (contravariance) — both conversions must compile and run.

7. In `Program.cs` build a `List<INotifier>` containing instances of `EmailNotifier`, `SmsNotifier`, and `PushNotifier`. Iterate over it with `foreach` and call `Send`, then invoke the DIM methods `Log` and `SendBatch` through the interface-typed variable. Separately cast the `EmailNotifier` to `IAuditable` and call `IAuditable.Log` to show that this is a different implementation. Print `LastChanged`. The expected output contains lines like `[Email → alice] Hello`, `[SMS → bob] Code 1234`, `[AUDIT @ ...] Document signed`, `[LOG] Start`, and `LastChanged = ...`.

8. Run the application with `dotnet run --project Notifications.Core` and confirm that the output matches expectations. Copy the output into a file named `output.txt` in the project folder.

#### Requirements

The solution must compile under .NET 8 without error-level warnings and run deterministically (no random numbers, no network or filesystem access beyond `Console`). Every interface name must start with `I` and carry a single responsibility: `INotifier` for delivery, `IAuditable` for the audit trail, `IProducer<out T>` for producing a value, `IComparator<in T>` for comparison. The `EmailNotifier` class must implement the conflicting `Log` method from both interfaces separately: implicitly for `INotifier` (via DIM or its own implementation) and explicitly for `IAuditable`. DIM methods must be called only through an interface-typed variable — the attempt to call one through a class-typed variable must be left commented out with an explanation of why it does not compile. The code should use C# 12 features where appropriate: top-level statements in `Program.cs`, collection expressions to initialise `List<INotifier>`, pattern matching where helpful, raw string literals for multi-line messages in `EmailNotifier`. A short `README.md` describing the file layout must accompany the project. An abstract class must not be used in place of an interface where a stateless contract is required.

#### Pitfalls

The main DIM trap is trying to call it through a class-typed variable. If you write `EmailNotifier e = new(); e.Log("x");`, the compiler will not see the `Log` method because a DIM does not become a member of the class: it is behaviour of the interface. You must either declare the variable as `INotifier n = e;` or cast it as `((INotifier)e).Log("x")`. The lesson states this directly: “a default method is only callable through a variable of the interface type, not of the class type.” The second trap is attempting to reach class fields from a DIM. An interface has no access to the class’s instance state, so a DIM can only call other members of the interface. If you need a value from the class, declare an abstract property on the interface and implement it in the class.

The third trap is behaviour merging when method names collide. If `EmailNotifier` implements `INotifier.Log` and `IAuditable.Log` with a single public method `public void Log(string)`, both contracts will treat it as their implementation, and an audit call will end up in the email log. Explicit implementation `void IAuditable.Log(string message)` solves the problem: such a member is invisible through the class and reachable only via `(IAuditable)obj`. Remember that explicit implementation carries no access modifier — putting `public` on it is a compile-time error.

The fourth trap is confusing `out` with `in`. Covariance `out` allows returning a more derived type: `IProducer<string>` can be converted to `IProducer<object>` because a string can be safely returned where an object is expected. Contravariance `in` works the other way: `IComparator<object>` can be converted to `IComparator<string>` because a comparator that can handle any object can also handle strings. Violating the direction (marking an input parameter as `out`) produces a compile-time error. Finally, do not try `new INotifier()` — an interface cannot be instantiated; create an instance of an implementing class and assign it to an interface-typed variable.

#### Acceptance criteria

- [ ] The `Notifications.Core` project builds under .NET 8 without errors or error-level warnings.
- [ ] `Notifications.Core.csproj` explicitly sets `<LangVersion>latest</LangVersion>`.
- [ ] The `INotifier` interface contains a required `Send` and at least three DIMs.
- [ ] The DIM methods `Log`, `SendBatch`, `NotifyError` have bodies inside the interface.
- [ ] The `EmailNotifier` class implements both `INotifier` and `IAuditable`.
- [ ] `IAuditable.Log` is implemented **explicitly**, with no access modifier.
- [ ] Calling `IAuditable.Log` through a cast produces output different from `INotifier.Log`.
- [ ] `SmsNotifier` overrides the DIM `Log`, while `PushNotifier` does not.
- [ ] `Program.cs` shows a DIM call through an interface-typed variable, with a commented-out class-typed attempt explained.
- [ ] `IProducer<out T>` is legally converted from `IProducer<string>` to `IProducer<object>`.
- [ ] `IComparator<in T>` is legally converted from `IComparator<object>` to `IComparator<string>`.
- [ ] C# 12 features are used: top-level statements, collection expressions, raw strings where useful.
- [ ] The `dotnet run` output contains the lines `[Email → alice]`, `[SMS → bob]`, `[AUDIT @`, `LastChanged =`.
- [ ] All interface names start with `I`.
- [ ] The `README.md` describes the project layout and the rationale for choosing interfaces.

#### Hints (no direct answer)

- Recall the lesson’s “socket” metaphor: `INotifier` is the standard, an implementation is the appliance. Ask yourself which operations the “socket” must provide and which are merely convenient wrappers.
- A DIM only needs a body in the interface declaration; the compiler makes it virtual automatically.
- To call a conflicting method, cast the object to the right interface: `((IAuditable)email).Log(...)`.
- For variance remember the mnemonic: `out` means output, return values, you may return a more specific type; `in` means input, arguments, you may accept a more general type.
- To confirm that a DIM is really unreachable through a class, try writing `email.Log("x")` and read the compiler message.
- For `SendBatch` a `foreach` loop inside the DIM body is safe, because `IEnumerable<string>` is part of the interface contract.

#### Reference solution walk-through

```csharp
// C# 12 / .NET 8 — Reference solution for homework M05-L04
using System;
using System.Collections.Generic;

// --- INotifier: delivery contract + DIM wrappers ---
public interface INotifier
{
    void Send(string recipient, string message); // required member

    // DIM: logs a message by sending it to a system recipient.
    public void Log(string message) => Send("system", $"[LOG] {message}");

    // DIM: sends a batch of messages.
    public void SendBatch(string recipient, IEnumerable<string> messages)
    {
        foreach (var m in messages) Send(recipient, m);
    }

    // DIM: notifies about an error.
    public void NotifyError(string recipient, Exception ex)
        => Send(recipient, $"[ERROR] {ex.Message}");
}

// --- IAuditable: audit contract with a same-named Log, creating a conflict ---
public interface IAuditable
{
    void Log(string message);
    DateTime LastChanged { get; }
}

// --- EmailNotifier: implements both interfaces, resolves the conflict explicitly ---
public class EmailNotifier : INotifier, IAuditable
{
    public DateTime LastChanged { get; private set; }

    // Implicit INotifier.Send — visible through class and interface.
    public void Send(string recipient, string message)
    {
        Console.WriteLine($"[Email → {recipient}] {message}");
        LastChanged = DateTime.UtcNow;
    }

    // Explicit IAuditable.Log — visible only via (IAuditable).
    void IAuditable.Log(string message)
    {
        Console.WriteLine($"[AUDIT @ {DateTime.UtcNow:O}] {message}");
        LastChanged = DateTime.UtcNow;
    }
}

// --- SmsNotifier overrides the DIM, PushNotifier does not ---
public class SmsNotifier : INotifier
{
    public void Send(string recipient, string message)
        => Console.WriteLine($"[SMS → {recipient}] {message}");

    // Override the DIM with SMS formatting.
    public void Log(string message) => Send("sms-gateway", $"[SMS-LOG] {message}");
}

public class PushNotifier : INotifier
{
    public void Send(string recipient, string message)
        => Console.WriteLine($"[PUSH → {recipient}] {message}");
    // Log and SendBatch come from the interface.
}

// --- Covariant producer interface ---
public interface IProducer<out T>
{
    T Produce();
}

public class StringProducer : IProducer<string>
{
    public string Produce() => "hello";
}

// --- Contravariant comparator ---
public interface IComparator<in T>
{
    int Compare(T x, T y);
}

public class LengthComparator : IComparator<string>
{
    public int Compare(string x, string y) => x.Length.CompareTo(y.Length);
}

// --- Program.cs (top-level statements, C# 12) ---
List<INotifier> notifiers = [new EmailNotifier(), new SmsNotifier(), new PushNotifier()];

foreach (var n in notifiers)
{
    n.Send("alice", "Hello");
    n.Log("Channel started");      // DIM via interface variable
    n.SendBatch("bob", ["Code 1234", "Do not reply"]);
}

var email = new EmailNotifier();
((IAuditable)email).Log("Document signed"); // different implementation
Console.WriteLine($"LastChanged = {email.LastChanged:O}");

// Covariance: string → object
IProducer<string> sp = new StringProducer();
IProducer<object> op = sp;          // legal due to out
Console.WriteLine(op.Produce());

// Contravariance: object → string
IComparator<object> objCmp = new UniversalLengthComparator();
IComparator<string> strCmp = objCmp; // legal due to in
Console.WriteLine(strCmp.Compare("abc", "ab"));

public class UniversalLengthComparator : IComparator<object>
{
    public int Compare(object x, object y) =>
        (x?.ToString()?.Length ?? 0).CompareTo(y?.ToString()?.Length ?? 0);
}
```

Line-by-line walk-through. The declaration `public interface INotifier` follows the lesson’s convention: an `I`-prefixed name and a single responsibility — notification delivery. The required method `Send` is the appliance’s “plug”; without it the contract is not satisfied. The three DIMs (`Log`, `SendBatch`, `NotifyError`) illustrate the feature’s main motivation from the lesson: they add convenient wrappers without breaking existing implementations and are only reachable through an interface-typed variable. Note that DIMs are not marked `virtual` explicitly — a body inside an interface makes them virtual by default, as the theory explains.

The `IAuditable` interface deliberately reuses the name `Log` — this is the conflict mentioned in the “Common Mistakes” section. The `EmailNotifier` class implements `INotifier.Send` implicitly (visible through both class and interface) and `IAuditable.Log` explicitly, as `void IAuditable.Log(string)`, with no access modifier. The two same-named behaviours are now separated: calling `email.Log("x")` (via the `INotifier` DIM) sends the log as a notification to a system recipient, while `((IAuditable)email).Log("x")` prints an audit line with a timestamp. This is precisely the lesson’s scenario with `IDraw.Draw` and `ICard.Draw`.

`SmsNotifier` overrides the DIM `Log`, showing that an implementation can choose its own behaviour when the default does not fit. `PushNotifier` does not override the DIM and inherits the default — for contrast, and to show that a DIM is optional to override. In `Program.cs` the `notifiers` list is initialised with a collection expression, a C# 12 feature. The `foreach` loop calls `Send`, `Log`, and `SendBatch` through the interface-typed variable `n`, which is what grants access to the DIMs. If we wrote `email.Log("x")` without a cast, the compiler would not see the DIM — the commented-out example demonstrates this.

The covariant `IProducer<out T>` allows the conversion from `IProducer<string>` to `IProducer<object>`, because a string can be safely returned where an object is expected. The contravariant `IComparator<in T>` allows the reverse conversion from `IComparator<object>` to `IComparator<string>`, because a comparator that handles any object also handles strings. `UniversalLengthComparator` implements `IComparator<object>` and is then cast to `IComparator<string>` — demonstrating the practical benefit of variance. All of these points map directly onto the lesson’s best practices: narrow interfaces, the `I` prefix, DIMs for augmentation, explicit implementation for conflicts, and variance for reuse.

#### Going deeper (bonus)

1. Add a third interface `ITemplatedNotifier` with a DIM `void SendTemplate(string recipient, string templateKey, IReadOnlyDictionary<string, string> args)` that builds a message from a template and calls `Send`. Make `EmailNotifier` implement it as well, and confirm that no name conflicts arise.
2. Extend `INotifier` with a new DIM `void Retry(string recipient, string message, int attempts)` that repeats `Send` until success or attempts are exhausted, using `try/catch`. Demonstrate that existing implementations automatically gain the new behaviour without recompilation of their source.
3. Implement a contravariant `IValidator<in T>` with a method `bool IsValid(T value)` and a covariant `IFactory<out T>` with a method `T Create()`. Build an example where a single `IValidator<object>` validates both strings and numbers, and a single `IFactory<string>` is substituted where an `IFactory<object>` is expected.
4. Write a unit test with xUnit that verifies `IAuditable.Log` and `INotifier.Log` on `EmailNotifier` produce different output. Use `StringWriter` to capture `Console.Out`.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Проект собирается под .NET 8 без ошибок и предупреждений error-уровня.
- [ ] (RU) Все интерфейсы названы с префикса `I` и имеют одну ответственность.
- [ ] (RU) `EmailNotifier` реализует `INotifier` и `IAuditable`, конфликт `Log` разрешён явно.
- [ ] (RU) DIM-методы вызываются только через переменную интерфейса; закомментированный вызов через класс поясняет ошибку.
- [ ] (RU) `IProducer<out T>` и `IComparator<in T>` демонстрируют ковариантность и контравариантность.
- [ ] (RU) В `Program.cs` используются top-level statements и collection expressions C# 12.
- [ ] (RU) Вывод `dotnet run` совпадает с ожидаемым и сохранён в `output.txt`.
- [ ] (RU) Файл `README.md` описывает структуру проекта.
- [ ] (EN) The project builds under .NET 8 with no errors or error-level warnings.
- [ ] (EN) All interfaces are `I`-prefixed and carry a single responsibility.
- [ ] (EN) `EmailNotifier` implements `INotifier` and `IAuditable`; the `Log` conflict is resolved explicitly.
- [ ] (EN) DIM methods are called only through interface-typed variables; the class-typed attempt is commented out with an explanation.
- [ ] (EN) `IProducer<out T>` and `IComparator<in T>` demonstrate covariance and contravariance.
- [ ] (EN) `Program.cs` uses top-level statements and C# 12 collection expressions.
- [ ] (EN) The `dotnet run` output matches expectations and is saved to `output.txt`.
- [ ] (EN) A `README.md` describes the project layout.

#### Ресурсы / Resources
- [Microsoft Learn — Interfaces](https://learn.microsoft.com/dotnet/csharp/fundamentals/types/interfaces)
- [Microsoft Learn — Default interface methods](https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-8#default-interface-methods)
- [Microsoft Learn — Variance in generic interfaces](https://learn.microsoft.com/dotnet/csharp/programming-guide/concepts/covariance-contravariance/variance-in-generic-interfaces)
- [Урок M05-L04](lesson-M05-L04-interfaces.md)

---

[⬆ К модулю M05](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
