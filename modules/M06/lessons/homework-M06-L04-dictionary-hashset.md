---
[← К уроку M06-L04](lesson-M06-L04-dictionary-hashset.md) | [⬆ К модулю M06](../README.md) | [Следующее ДЗ →](homework-M06-L05-queue-stack-linkedlist.md)
---

### Домашнее задание M06-L04: Dictionary<K,V>, HashSet<T>, равенство и GetHashCode / Homework M06-L04: Dictionary<K,V>, HashSet<T>, equality and GetHashCode

**Урок / Lesson:** M06-L04
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) На практике закрепить работу хеш-таблиц `Dictionary<TKey,TValue>` и `HashSet<T>`, научиться корректно реализовывать контракт `Equals`/`GetHashCode` для собственных типов-ключей, применять `IEquatable<T>` и кастомные `IEqualityComparer<T>`, понимать цену мутации ключа после вставки и уметь предсказывать, когда среднее `O(1)` вырождается в `O(n)`. (EN) Get hands-on practice with the `Dictionary<TKey,TValue>` and `HashSet<T>` hash tables, learn to implement the `Equals`/`GetHashCode` contract correctly for custom key types, apply `IEquatable<T>` and custom `IEqualityComparer<T>`, understand the cost of mutating a key after insertion, and predict when average `O(1)` degrades to `O(n)`.

#### Связь с уроком / Connection to the lesson
(RU) Урок вводит хеш-таблицы `Dictionary` и `HashSet`, объясняет контракт `GetHashCode`/`Equals`, роль `EqualityComparer<T>.Default`, `IEquatable<T>`, `HashCode.Combine` и коварные баги с мутацией ключа. Это ДЗ заставит вас применить каждую из этих концепций в одном связном проекте: вы построите мини-кэш HTTP-запросов, где ключом служит собственный тип, а строковые коллекции используют регистронезависимое сравнение. (EN) The lesson introduces the `Dictionary` and `HashSet` hash tables, explains the `GetHashCode`/`Equals` contract, the role of `EqualityComparer<T>.Default`, `IEquatable<T>`, `HashCode.Combine`, and the nasty bugs caused by mutating keys. This homework makes you apply every one of those concepts in a single cohesive project: you will build a mini HTTP-request cache where the key is a custom type and the string collections use case-insensitive comparison.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Вы пишете учебный «front-end» для крошечного HTTP-фронтенда: входящие запросы кэшируются, чтобы не дёргать бэкенд повторно для идентичных маршрутов, а аналитика параллельно считает, сколько уникальных имён параметров запроса вы встретили и сколько запросов пришло на каждый хост. Это ровно те задачи, для которых в .NET существуют `Dictionary<TKey,TValue>` и `HashSet<T>`: быстрый поиск «по содержимому ключа» в среднем за `O(1)` и набор уникальных значений без полезной нагрузки. Наивная реализация на `List<T>` с линейным `Contains` начинала бы тормозить уже на нескольких тысячах записей, тогда как хеш-таблица остаётся почти плоской. Однако хеш-таблица работает корректно только при соблюдении строгого контракта между `Equals` и `GetHashCode`: если равные объекты дают разный хеш, вы положите элемент и тут же «потеряете» его при поиске. Поэтому ключевым элементом задания становится собственный тип-ключ `Endpoint` (HTTP-метод + путь), который вы сделаете неизменяемым и корректно реализуете через `IEquatable<T>` и `HashCode.Combine`. Дополнительно вы столкнётесь со строковыми ключами, где регистр не должен иметь значения (имена параметров и имена хостов), — и здесь правильным решением будет не «ручная» нормализация `ToLower()` по всему коду, а передача готового `StringComparer.OrdinalIgnoreCase` в конструктор коллекции. Наконец, вы должны прочувствовать на себе, что произойдёт, если нарушить правило «ключ в таблице — константа»: для этого в задании есть отдельный шаг с демонстрацией «пропавшего» значения. В итоге вы получаете не просто рабочий код, а осознанное владение контрактом хеш-таблиц — навык, который отличает джуниора от уверенного middle-разработчика.

#### Что нужно сделать (пошагово)
1. Создайте новый консольный проект на .NET 8:
   ```
   dotnet new console -n RequestCache -f net8.0
   cd RequestCache
   ```
   Убедитесь, что в `RequestCache.csproj` значение `<TargetFramework>net8.0</TargetFramework>` и язык C# 12 (можно добавить `<LangVersion>latest</LangVersion>` для подстраховки).

2. Создайте файл `Endpoint.cs` с типом-ключом `Endpoint`. Это должен быть `readonly struct` с двумя полями: `Method` (строка, например `"GET"`) и `Path` (строка, например `"/users/42"`). Реализуйте `IEquatable<Endpoint>`: типобезопасный `Equals(Endpoint other)`, переопределённый `Equals(object?)`, `GetHashCode()` через `HashCode.Combine(Method, Path)` и операторы `==`/`!=`. Сделайте ключ **неизменяемым**: поля только для чтения, изменять их после создания нельзя.

3. В `Program.cs` (top-level statements) создайте кэш ответов:
   ```csharp
   var cache = new Dictionary<Endpoint, string>(capacity: 16);
   ```
   Задайте начальную ёмкость, чтобы избежать лишних перехеширований — это best practice из урока. Добавьте несколько пар «маршрут → ответ» через индексатор, попробуйте `TryGetValue` для существующего и несуществующего ключа, выведите результат.

4. Создайте `HashSet<string>` для накопления уникальных имён параметров запроса, **регистронезависимо**. Передайте в конструктор `StringComparer.OrdinalIgnoreCase`. Добавьте параметры `"id"`, `"ID"`, `"Id"`, `"page"` и убедитесь, что `Count == 2`. Выведите содержимое.

5. Создайте `Dictionary<string, int>` для подсчёта запросов по хосту, тоже **регистронезависимо**. Используйте `StringComparer.OrdinalIgnoreCase`. Добавьте хосты `"api.example.com"` и `"API.Example.COM"` — они должны слиться в один ключ со значением `2`. Выведите итоговую статистику.

6. Реализуйте метод `RegisterHit(Dictionary<string,int> counts, string host)`, который через `TryGetValue`/индексатор увеличивает счётчик. Не используйте `ContainsKey` + двойной поиск — это антипаттерн; `TryGetValue` даёт значение и факт наличия за один проход.

7. Демонстрация мутации ключа (главная ловушка урока): в отдельном методе `DemonstrateMutationBug()` создайте `class MutableEndpoint` с публичными сеттерами, переопределёнными `Equals`/`GetHashCode` (зависящими от полей). Положите экземпляр в `Dictionary<MutableEndpoint, string>`, затем **измените поле ключа после вставки** и попробуйте найти значение по тому же экземпляру объекта. Выведите результат и прокомментируйте в коде, почему значение «потерялось»: хеш изменился, элемент остался в старой корзине, а поиск идёт в новой.

8. Демонстрация контракта: в методе `DemonstrateBrokenContract()` покажите тип, у которого `Equals` переопределён, а `GetHashCode` возвращает константу (или не переопределён). Поместите несколько элементов в `Dictionary` и объясните в комментарии, почему поиск по-прежнему работает, но производительность вырождается в `O(n)` из-за коллизий в одной корзине.

9. Соберите и запустите проект:
   ```
   dotnet build
   dotnet run
   ```
   Ожидаемый вывод должен содержать: найденное значение кэша для существующего ключа, `false` для отсутствующего, `Count == 2` для уникальных параметров, итог `api.example.com => 2` для хостов, и строки-комментарии о «потерянном» значении и деградации до `O(n)`.

10. (Опционально, но желательно) Добавьте asserts-проверки через `Debug.Assert` или просто `if`-проверки с `Console.WriteLine`, которые падают с понятным сообщением, если инварианты нарушены (например, если `cache.TryGetValue(...)` вернул `false` там, где должен был `true`).

#### Требования к решению
- Целевая платформа — .NET 8, язык C# 12. Используйте top-level statements в `Program.cs`, pattern matching (`is`-выражения), collection expressions для инициализации там, где это уместно, и raw string literals для длинных строковых литералов в выводе, если они нужны.
- Тип `Endpoint` обязан быть `readonly struct`, реализующим `IEquatable<Endpoint>`. Обязательно переопределены **и** `Equals`, **и** `GetHashCode` согласованно: в `GetHashCode` входят ровно те же поля, что участвуют в `Equals`. Хеш считается через `HashCode.Combine(...)`, а не вручную.
- `Endpoint` неизменяем: поля `init`-only или `readonly`. Никаких публичных сеттеров у ключа. Имена хостов и параметров сравниваются через `StringComparer.OrdinalIgnoreCase`, переданный в конструктор `HashSet`/`Dictionary`, а не через `ToLower()` в прикладном коде.
- Для значимых типов-ключей реализован `IEquatable<T>`, чтобы избежать боксинга при сравнении внутри хеш-таблицы.
- `Dictionary` создаётся с предзаданной ёмкостью там, где размер известен заранее.
- Метод `RegisterHit` использует `TryGetValue` (один проход), а не `ContainsKey` + индексатор (два прохода).
- Демонстрации мутации и сломанного контракта выделены в отдельные методы и снабжены комментариями, объясняющими механизм бага.
- Код компилируется без предупреждений (или с обоснованными) и запускается. Вывод соответствует ожидаемому.
- Запрещено использовать LINQ для подмены хеш-таблиц: цель — понять `Dictionary`/`HashSet` напрямую.

#### Тонкости и подводные камни
- **Контракт — это не рекомендация, а закон.** Если `a.Equals(b)` истинно, то `a.GetHashCode() == b.GetHashCode()` обязано выполняться. Обратное неверно: разные объекты могут давать одинаковый хеш (коллизия), это нормально и не ломает корректность — лишь слегка замедляет. Нарушение прямого направления ломает поиск: элемент вставляется в корзину по одному хешу, а ищется по другому.
- **Переопределяете `Equals` — переопределяйте и `GetHashCode`.** Всегда парой. Для `class` «забытый» `GetHashCode` унаследует ссылочную семантику, и два равных по значению объекта окажутся в разных корзинах. Для `struct` дефолтный `GetHashCode` комбинирует поля, но это undocumented-поведение, на которое нельзя полагаться.
- **Мутирующий ключ — главный враг.** Если после вставки в `Dictionary`/`HashSet` изменить поле, участвующее в хеше, элемент «телепортируется» логически в другую корзину, оставшись физически в старой. Результат: утечка памяти в `HashSet`, «пропавшее» значение в `Dictionary`, потенциально бесконечный цикл при переборе. Правило урока: ключ в таблице — константа; меняющиеся данные уносите в `TValue`.
- **`HashCode.Combine` лучше ручных формул.** Старый приём `x * 31 + y` работает, но чувствителен к порядку и к коллизиям для близких значений. `HashCode.Combine` использует качественное смешивание и устойчив к патологическим входам.
- **`IEquatable<T>` убирает боксинг** для значимых типов: без него сравнение идёт через `object.Equals(object)`, что боксит `struct` и аллоцирует на куче. Для горячих путей это критично.
- **`EqualityComparer<T>.Default` уже умеет регистронезависимость для строк?** Нет — по умолчанию строки сравниваются с учётом регистра. Нужен явный `StringComparer.OrdinalIgnoreCase` в конструкторе.
- **`O(1)` — среднее, не гарантированное.** Худший случай — `O(n)` при сплошных коллизиях (плохой comparer, константный хеш, злонамеренный ввод). Поэтому никогда не возвращайте константный хеш «потому что проще».
- **Ёмкость.** `Dictionary` автоматически растёт, но каждое расширение — перехеширование всех элементов. Если размер известен, задавайте `capacity` заранее.
- **`TryGetValue` vs `ContainsKey`.** `ContainsKey` + индексатор — два поиска; `TryGetValue` — один. В горячих циклах разница заметна.
- **`record` — удобная альтернатива** `readonly struct` для ключей: он автоматически реализует `Equals`/`GetHashCode` на основе значений полей и неизменяем через `init`-свойства. Но для значимой семантики без аллокаций `readonly struct` + `IEquatable<T>` эффективнее.

#### Критерии приёмки
- [ ] Проект `RequestCache` собирается под .NET 8 / C# 12 без ошибок и предупреждений компилятора.
- [ ] Тип `Endpoint` — `readonly struct`, реализующий `IEquatable<Endpoint>`.
- [ ] В `Endpoint` переопределены **и** `Equals(object?)`, **и** `GetHashCode()`, согласованно (те же поля).
- [ ] `GetHashCode()` использует `HashCode.Combine(Method, Path)`.
- [ ] Операторы `==` и `!=` для `Endpoint` определены и согласованы с `Equals`.
- [ ] `Endpoint` неизменяем: нет публичных сеттеров у полей, влияющих на хеш.
- [ ] `Dictionary<Endpoint, string>` создаётся с предзаданной `capacity`.
- [ ] Поиск существующего ключа через `TryGetValue` возвращает `true` и корректное значение.
- [ ] Поиск несуществующего ключа возвращает `false`, `out`-параметр имеет значение по умолчанию.
- [ ] `HashSet<string>` имён параметров создан с `StringComparer.OrdinalIgnoreCase` и содержит ровно 2 элемента после добавления `"id"`, `"ID"`, `"Id"`, `"page"`.
- [ ] `Dictionary<string, int>` хостов регистронезависим: `"api.example.com"` и `"API.Example.COM"` дают счётчик `2`.
- [ ] `RegisterHit` использует `TryGetValue` (один проход), а не `ContainsKey` + индексатор.
- [ ] Метод `DemonstrateMutationBug()` показывает «потерю» значения после мутации ключа и содержит комментарий-объяснение механизма.
- [ ] Метод `DemonstrateBrokenContract()` показывает деградацию до `O(n)` при константном/отсутствующем `GetHashCode` и содержит комментарий.
- [ ] Вывод `dotnet run` соответствует ожидаемому: найденное значение, `false` для отсутствующего, `Count == 2`, `api.example.com => 2`, пояснения по двум демонстрациям.

#### Подсказки (без прямого ответа)
- В `HashCode.Combine` можно передать несколько аргументов — он сам их смешает; не нужно писать собственную формулу.
- Для `readonly struct` операторы `==`/`!=` не появляются автоматически от `IEquatable<T>` — их нужно задать явно, делегировав к `Equals`.
- `StringComparer.OrdinalIgnoreCase` — это готовый `IEqualityComparer<string>` (и `IComparer<string>`), его можно напрямую передать в конструктор.
- Чтобы «потерять» значение при мутации, достаточно изменить поле, входящее в `GetHashCode`, после `dict[key] = value`. Попробуйте потом `dict.TryGetValue(key, out _)` по тому же экземпляру — удивитесь.
- Для демонстрации `O(n)` поместите много элементов в словарь с типом-ключом, у которого `GetHashCode() => 0`: поиск останется корректным, но все элементы окажутся в одной корзине.
- `TryGetValue` возвращает `bool` и заполняет `out`-параметр — используйте оба результата: первый для ветвления, второй для инкремента счётчика.
- Не путайте `Dictionary<TKey,TValue>.Keys`/`Values` с самими парами — для перебора пар используйте `KeyValuePair` в `foreach`.

#### Эталонное решение (разбор)
```csharp
// Endpoint.cs — RU: неизменяемый ключ, реализует IEquatable<T>; EN: immutable key implementing IEquatable<T>
// RU: readonly struct гарантирует, что поля нельзя изменить после создания — хеш стабилен.
// EN: readonly struct guarantees fields cannot change after creation — the hash is stable.
public readonly struct Endpoint : IEquatable<Endpoint>
{
    public string Method { get; }   // RU: HTTP-метод, например "GET"  EN: HTTP method, e.g. "GET"
    public string Path   { get; }   // RU: путь запроса, например "/users/42"  EN: request path, e.g. "/users/42"

    public Endpoint(string method, string path)
    {
        Method = method;
        Path   = path;
    }

    // RU: типобезопасное сравнение без боксинга (IEquatable<T>).
    // EN: type-safe comparison without boxing (IEquatable<T>).
    public bool Equals(Endpoint other)
        => string.Equals(Method, other.Method, StringComparison.Ordinal)
           && string.Equals(Path, other.Path, StringComparison.Ordinal);

    // RU: переопределение object.Equals делегирует к типизированному; null/чужой тип → false.
    // EN: override of object.Equals delegates to the typed version; null/wrong type → false.
    public override bool Equals(object? obj) => obj is Endpoint e && Equals(e);

    // RU: контракт — те же поля, что в Equals; смешивание через HashCode.Combine.
    // EN: contract — same fields as in Equals; mixing via HashCode.Combine.
    public override int GetHashCode() => HashCode.Combine(Method, Path);

    // RU: операторы для удобства; делегируют к Equals, чтобы не рассинхронизировать логику.
    // EN: operators for convenience; delegate to Equals to keep logic in sync.
    public static bool operator ==(Endpoint a, Endpoint b) => a.Equals(b);
    public static bool operator !=(Endpoint a, Endpoint b) => !a.Equals(b);
}

// RU: намеренно сломанный тип для демонстрации контракта.
// EN: intentionally broken type to demonstrate the contract.
public sealed class ConstantHashEndpoint
{
    public string Method { get; init; }
    public string Path   { get; init; }

    public override bool Equals(object? obj)
        => obj is ConstantHashEndpoint e
           && string.Equals(Method, e.Method, StringComparison.Ordinal)
           && string.Equals(Path, e.Path, StringComparison.Ordinal);

    // RU: константный хеш — корректно, но все элементы в одной корзине → O(n).
    // EN: constant hash — correct, but all entries in one bucket → O(n).
    public override int GetHashCode() => 0;
}

// RU: мутабельный ключ для демонстрации бага мутации.
// EN: mutable key to demonstrate the mutation bug.
public sealed class MutableEndpoint
{
    public string Method { get; set; }
    public string Path   { get; set; }

    public override bool Equals(object? obj)
        => obj is MutableEndpoint e
           && string.Equals(Method, e.Method, StringComparison.Ordinal)
           && string.Equals(Path, e.Path, StringComparison.Ordinal);

    public override int GetHashCode() => HashCode.Combine(Method, Path);
}
```

```csharp
// Program.cs — top-level statements, C# 12 / .NET 8
using System;
using System.Collections.Generic;

// 1. RU: кэш ответов с предзаданной ёмкостью; ключ — собственный тип Endpoint.
//    EN: response cache with preset capacity; key is the custom Endpoint type.
var cache = new Dictionary<Endpoint, string>(capacity: 16)
{
    [new Endpoint("GET", "/users/42")] = "200 OK: user 42",
    [new Endpoint("POST", "/orders")]  = "201 Created: order #7",
};

// 2. RU: TryGetValue — один проход, в отличие от ContainsKey + индексатор.
//    EN: TryGetValue — single pass, unlike ContainsKey + indexer.
if (cache.TryGetValue(new Endpoint("GET", "/users/42"), out var body))
{
    Console.WriteLine($"RU: найдено / EN: found → {body}");
}
else
{
    Console.WriteLine("RU: не найдено / EN: not found");
}

// 3. RU: проверка отсутствующего ключа.
//    EN: check for a missing key.
bool hit = cache.TryGetValue(new Endpoint("DELETE", "/users/42"), out _);
Console.WriteLine($"RU: DELETE есть в кэше? / EN: DELETE in cache? {hit}");

// 4. RU: HashSet уникальных имён параметров, регистронезависимо через BCL-компарер.
//    EN: HashSet of unique parameter names, case-insensitive via a BCL comparer.
var paramNames = new HashSet<string>(StringComparer.OrdinalIgnoreCase)
{
    "id", "ID", "Id", "page",
};
Console.WriteLine($"RU: уникальных параметров / EN: unique params: {paramNames.Count}"); // → 2

// 5. RU: счётчик запросов по хосту, тоже регистронезависимо.
//    EN: per-host request counter, also case-insensitive.
var hostCounts = new Dictionary<string, int>(StringComparer.OrdinalIgnoreCase);
RegisterHit(hostCounts, "api.example.com");
RegisterHit(hostCounts, "API.Example.COM");   // RU: тот же ключ  EN: same key
RegisterHit(hostCounts, "cdn.example.com");
foreach (var (host, count) in hostCounts)
{
    Console.WriteLine($"RU: хост / EN: host {host} => {count}");
}

// 6. RU: метод увеличивает счётчик за один TryGetValue.
//    EN: increments the counter in a single TryGetValue.
static void RegisterHit(Dictionary<string, int> counts, string host)
{
    if (counts.TryGetValue(host, out var current))
    {
        counts[host] = current + 1;
    }
    else
    {
        counts[host] = 1;
    }
}

DemonstrateMutationBug();
DemonstrateBrokenContract();

// 7. RU: ключ мутирован после вставки — значение «теряется» в корзине.
//    EN: key mutated after insertion — the value is "lost" in its bucket.
static void DemonstrateMutationBug()
{
    var dict = new Dictionary<MutableEndpoint, string>();
    var key  = new MutableEndpoint { Method = "GET", Path = "/old" };
    dict[key] = "RU: секретное значение / EN: secret value";

    // RU: мутируем поле, входящее в GetHashCode — хеш меняется, корзина осталась старой.
    // EN: mutate a field used by GetHashCode — the hash changes, the bucket is stale.
    key.Path = "/new";

    bool found = dict.TryGetValue(key, out var v);
    Console.WriteLine(
        $"RU: найдено после мутации? / EN: found after mutation? {found} " +
        $"({(found ? v : "RU: потеряно / EN: lost")})");
    // RU: ожидается found == false — хеш больше не указывает на нужную корзину.
    // EN: expected found == false — the hash no longer points to the right bucket.
}

// 8. RU: константный хеш — корректность сохранена, но всё в одной корзине → O(n).
//    EN: constant hash — correctness preserved, but everything in one bucket → O(n).
static void DemonstrateBrokenContract()
{
    var dict = new Dictionary<ConstantHashEndpoint, int>();
    for (int i = 0; i < 1000; i++)
    {
        dict[new ConstantHashEndpoint
        {
            Method = "GET",
            Path    = $"/item/{i}"
        }] = i;
    }

    // RU: поиск по-прежнему найдёт элемент (Equals работает), но пройдёт всю корзину.
    // EN: lookup still finds the entry (Equals works) but scans the entire bucket.
    bool found = dict.TryGetValue(
        new ConstantHashEndpoint { Method = "GET", Path = "/item/500" },
        out var value);
    Console.WriteLine(
        $"RU: константный хеш — найдено? / EN: constant hash — found? {found}, " +
        $"value={value}. RU: но сложность O(n), не O(1). " +
        $"EN: but complexity is O(n), not O(1).");
}
```

Разбор по строкам. Тип `Endpoint` объявлен `readonly struct`: это даёт сразу две гарантии урока — значимая семантика (без аллокаций) и неизменяемость полей (хеш стабилен после создания). Реализация `IEquatable<Endpoint>` с типизированным `Equals(Endpoint other)` убирает боксинг, который иначе возник бы при вызове `object.Equals(object)` внутри хеш-таблицы — это best practice прямо из урока. Переопределённый `Equals(object?)` использует pattern matching (`obj is Endpoint e`) и делегирует к типизированной версии, чтобы логика сравнения жила в одном месте. `GetHashCode()` построен через `HashCode.Combine(Method, Path)` — урок явно предостерегает от ручных формул вида `x * 31 + y`, потому что они дают патологические коллизии на близких входах. Важно, что в `GetHashCode` входят **те же поля**, что и в `Equals`: это и есть буквальное выполнение контракта. Операторы `==`/`!=` определены явно, потому что `IEquatable<T>` их не генерирует автоматически; они делегируют к `Equals`, чтобы не рассинхронизировать поведение с `Equals(object?)`.

В `Program.cs` кэш создаётся с `capacity: 16` — это best practice «предзадавайте ёмкость, когда размер известен», он убирает лишние перехеширования при росте. Поиск выполняется через `TryGetValue`, а не через `ContainsKey` + индексатор: первый делает один проход по хеш-таблице, второй — два, и в горячих путях разница существенна. `HashSet<string>` для имён параметров и `Dictionary<string,int>` для хостов оба получают `StringComparer.OrdinalIgnoreCase` в конструкторе — это рекомендуемый уроком способ работать с регистронезависимыми строковыми ключами вместо ручной нормализации `ToLower()`, которая засоряет код и порождает баги. Метод `RegisterHit` иллюстрирует идиому «`TryGetValue` → инкремент `out`-значения или инициализация единицей».

Демонстрация `DemonstrateMutationBug` материализует главную ловушку урока: после `dict[key] = value` мы меняем `key.Path`, который входит в `GetHashCode`. Хеш объекта меняется, но элемент физически остался в корзине, соответствующей **старому** хешу. Последующий `TryGetValue(key, ...)` вычисляет **новый** хеш и ищет в другой корзине — и не находит элемент, хотя экземпляр объекта тот же самый. Это и есть «пропавшее» значение. Демонстрация `DemonstrateBrokenContract` показывает обратный крайний случай: `GetHashCode` возвращает константу, контракт формально соблюдён (равные объекты действительно дают равный хеш — он у всех одинаковый), и поиск **корректен**, но все элементы оказываются в одной корзине, превращая среднее `O(1)` в худшее `O(n)`. Урок подчёркивает, что `O(1)` — это среднее время, а плохой comparer способен свести на нет все преимущества хеш-таблицы. Вместе обе демонстрации дают ученику интуицию о двух режимах отказа хеш-таблицы: «вообще не находится» (нарушен прямой контракт) и «находится, но медленно» (хеш корректен, но бесполезен).

#### Задания на углубление (бонус)
1. Реализуйте собственный `IEqualityComparer<Endpoint>`, который сравнивает пути **без учёта trailing slash**: `"/users"` и `"/users/"` считаются одним ключом. Подсчитайте хеш так, чтобы он совпадал для обоих вариантов (например, нормализуйте путь перед `HashCode.Combine`). Передайте comparer в конструктор `Dictionary` и проверьте слияние ключей.
2. Перепишите `Endpoint` как `record` (а не `readonly struct`) и сравните поведение: `record` автоматически реализует `Equals`/`GetHashCode` на основе значений. Измерьте через `BenchmarkDotNet` разницу в аллокациях между `readonly struct` и `record` (class) при миллионе операций `TryGetValue`. Объясните, почему `readonly struct` + `IEquatable<T>` эффективнее для горячих путей.
3. Добавьте «LRU-эвикцию»: храните время последнего доступа и при превышении `capacity` удаляйте самые старые записи. Подумайте, почему простое удаление по ключу из `Dictionary` сохраняет корректность хеш-таблицы (в отличие от мутации ключа).
4. Протестируйте крайний случай: 10 000 ключей с `Path = "/item/" + i`, но с тем же comparer, что в п.1. Сравните время поиска при `HashCode.Combine` и при константном хеше — численно подтвердите деградацию до `O(n)`.

---

## Statement in English / Постановка на английском

#### Context & motivation
You are writing a small in-memory front-end for a tiny HTTP service: incoming requests are cached so that identical routes do not hit the backend twice, while analytics simultaneously tracks how many unique query-parameter names you have seen and how many requests arrived at each host. These are exactly the tasks for which .NET ships `Dictionary<TKey,TValue>` and `HashSet<T>`: fast lookup “by key content” in average `O(1)`, and a set of unique values with no payload attached. A naive `List<T>`-based implementation with linear `Contains` would already slow down at a few thousand entries, whereas a hash table stays nearly flat. However, a hash table only behaves correctly when a strict contract between `Equals` and `GetHashCode` is honoured: if equal objects produce different hashes, you will insert an element and immediately “lose” it on lookup. That is why the central piece of this assignment is a custom key type `Endpoint` (HTTP method + path) that you make immutable and implement correctly through `IEquatable<T>` and `HashCode.Combine`. On top of that, you will deal with string keys where case must not matter (parameter names and host names) — and the right answer there is not a scattered `ToLower()` normalisation across the code base, but a single `StringComparer.OrdinalIgnoreCase` passed into the collection constructor. Finally, you must feel on your own skin what happens when the rule “a key in a table is a constant” is broken: the assignment has a dedicated step that demonstrates a “vanished” value. In the end you get not just working code, but a deliberate mastery of the hash-table contract — the skill that separates a junior from a confident mid-level developer.

#### What to do step by step
1. Create a new console project on .NET 8:
   ```
   dotnet new console -n RequestCache -f net8.0
   cd RequestCache
   ```
   Make sure `RequestCache.csproj` contains `<TargetFramework>net8.0</TargetFramework>` and C# 12 (you may add `<LangVersion>latest</LangVersion>` to be safe).

2. Create a file `Endpoint.cs` with the key type `Endpoint`. It must be a `readonly struct` with two fields: `Method` (a string, e.g. `"GET"`) and `Path` (a string, e.g. `"/users/42"`). Implement `IEquatable<Endpoint>`: a type-safe `Equals(Endpoint other)`, an overridden `Equals(object?)`, a `GetHashCode()` built with `HashCode.Combine(Method, Path)`, and the `==`/`!=` operators. Make the key **immutable**: fields are read-only and cannot change after construction.

3. In `Program.cs` (top-level statements) create the response cache:
   ```csharp
   var cache = new Dictionary<Endpoint, string>(capacity: 16);
   ```
   Pre-set the capacity to avoid extra rehashing — this is a best practice from the lesson. Add a few “route → response” pairs through the indexer, try `TryGetValue` for an existing and a missing key, and print the result.

4. Create a `HashSet<string>` to accumulate unique query-parameter names, **case-insensitively**. Pass `StringComparer.OrdinalIgnoreCase` to the constructor. Add the parameters `"id"`, `"ID"`, `"Id"`, `"page"` and confirm that `Count == 2`. Print the contents.

5. Create a `Dictionary<string, int>` to count requests per host, also **case-insensitively**. Use `StringComparer.OrdinalIgnoreCase`. Add the hosts `"api.example.com"` and `"API.Example.COM"` — they must merge into a single key with value `2`. Print the final statistics.

6. Implement a method `RegisterHit(Dictionary<string,int> counts, string host)` that increments the counter using `TryGetValue`/indexer. Do not use `ContainsKey` plus a double lookup — that is an anti-pattern; `TryGetValue` gives you both the value and the presence flag in a single pass.

7. Mutation demonstration (the main trap of the lesson): in a separate method `DemonstrateMutationBug()` create a `class MutableEndpoint` with public setters and overridden `Equals`/`GetHashCode` depending on the fields. Put an instance into a `Dictionary<MutableEndpoint, string>`, then **mutate a key field after insertion** and try to find the value using the very same object instance. Print the result and comment in the code on why the value was “lost”: the hash changed, the entry stayed in the old bucket, and the lookup goes to the new one.

8. Contract demonstration: in a method `DemonstrateBrokenContract()` show a type where `Equals` is overridden but `GetHashCode` returns a constant (or is not overridden at all). Put several entries into a `Dictionary` and explain in a comment why lookup still works but performance degrades to `O(n)` due to collisions all landing in one bucket.

9. Build and run the project:
   ```
   dotnet build
   dotnet run
   ```
   The expected output must contain: the cached value found for an existing key, `false` for a missing key, `Count == 2` for unique parameters, the final `api.example.com => 2` for hosts, and explanatory comment lines about the “lost” value and the `O(n)` degradation.

10. (Optional but recommended) Add assertions via `Debug.Assert` or simple `if` checks with `Console.WriteLine` that fail with a clear message when invariants are violated (for example, when `cache.TryGetValue(...)` returns `false` where it should have returned `true`).

#### Requirements
- Target .NET 8, language C# 12. Use top-level statements in `Program.cs`, pattern matching (`is`-expressions), collection expressions for initialisation where appropriate, and raw string literals for long string literals in the output if needed.
- The `Endpoint` type must be a `readonly struct` implementing `IEquatable<Endpoint>`. Both `Equals` **and** `GetHashCode` must be overridden consistently: `GetHashCode` must include exactly the same fields that take part in `Equals`. The hash must be computed via `HashCode.Combine(...)`, not hand-rolled.
- `Endpoint` is immutable: `init`-only or `readonly` fields. No public setters on the key. Host and parameter names are compared through `StringComparer.OrdinalIgnoreCase` passed to the `HashSet`/`Dictionary` constructor, not through `ToLower()` in application code.
- For value-type keys `IEquatable<T>` is implemented to avoid boxing during comparisons inside the hash table.
- The `Dictionary` is created with a pre-set capacity wherever the size is known in advance.
- The `RegisterHit` method uses `TryGetValue` (one pass) rather than `ContainsKey` plus indexer (two passes).
- The mutation and broken-contract demonstrations live in separate methods and carry comments explaining the failure mechanism.
- The code compiles without warnings (or with justified ones) and runs. The output matches the expected one.
- Do not use LINQ to replace the hash tables: the goal is to understand `Dictionary`/`HashSet` directly.

#### Pitfalls
- **The contract is a law, not a recommendation.** If `a.Equals(b)` is true, then `a.GetHashCode() == b.GetHashCode()` must hold. The reverse is not required: different objects may share a hash (a collision), which is fine and does not break correctness — it only slows things down slightly. Breaking the forward direction breaks lookup: an element is inserted into the bucket for one hash and searched for in the bucket for another.
- **Override `Equals` — override `GetHashCode`.** Always as a pair. For a `class`, a forgotten `GetHashCode` inherits reference semantics, and two value-equal objects end up in different buckets. For a `struct`, the default `GetHashCode` combines fields, but that is undocumented behaviour you must not rely on.
- **A mutating key is the chief enemy.** If, after inserting into a `Dictionary`/`HashSet`, you mutate a field that participates in the hash, the element logically “teleports” to another bucket while physically staying in the old one. The result: memory leaks in `HashSet`, “vanished” values in `Dictionary`, potentially infinite loops during enumeration. The lesson’s rule: a key in a table is a constant; move mutable data into `TValue`.
- **`HashCode.Combine` is better than hand-rolled formulas.** The old trick `x * 31 + y` works but is sensitive to ordering and to collisions for close values. `HashCode.Combine` uses quality mixing and is robust against pathological inputs.
- **`IEquatable<T>` removes boxing** for value types: without it, comparison goes through `object.Equals(object)`, which boxes the `struct` and allocates on the heap. For hot paths this is critical.
- **Does `EqualityComparer<T>.Default` already handle case-insensitive strings?** No — by default strings compare case-sensitively. You need an explicit `StringComparer.OrdinalIgnoreCase` in the constructor.
- **`O(1)` is average, not guaranteed.** The worst case is `O(n)` under continuous collisions (a poor comparer, a constant hash, malicious input). Never return a constant hash “because it is simpler”.
- **Capacity.** A `Dictionary` grows automatically, but every resize rehashes every entry. When the size is known, set `capacity` up front.
- **`TryGetValue` vs `ContainsKey`.** `ContainsKey` plus indexer is two lookups; `TryGetValue` is one. The difference matters in hot loops.
- **`record` is a convenient alternative** to `readonly struct` for keys: it auto-implements `Equals`/`GetHashCode` based on field values and is immutable through `init`-only properties. But for value semantics without allocations, `readonly struct` plus `IEquatable<T>` is more efficient.

#### Acceptance criteria
- [ ] The `RequestCache` project builds under .NET 8 / C# 12 with no compiler errors or warnings.
- [ ] The `Endpoint` type is a `readonly struct` implementing `IEquatable<Endpoint>`.
- [ ] Both `Equals(object?)` **and** `GetHashCode()` are overridden in `Endpoint` consistently (same fields).
- [ ] `GetHashCode()` uses `HashCode.Combine(Method, Path)`.
- [ ] The `==` and `!=` operators for `Endpoint` are defined and consistent with `Equals`.
- [ ] `Endpoint` is immutable: no public setters on fields that affect the hash.
- [ ] The `Dictionary<Endpoint, string>` is created with a pre-set `capacity`.
- [ ] Lookup of an existing key through `TryGetValue` returns `true` and the correct value.
- [ ] Lookup of a missing key returns `false`, and the `out` parameter has the default value.
- [ ] The `HashSet<string>` of parameter names is created with `StringComparer.OrdinalIgnoreCase` and contains exactly 2 elements after adding `"id"`, `"ID"`, `"Id"`, `"page"`.
- [ ] The `Dictionary<string, int>` of hosts is case-insensitive: `"api.example.com"` and `"API.Example.COM"` yield a counter of `2`.
- [ ] `RegisterHit` uses `TryGetValue` (one pass), not `ContainsKey` plus indexer.
- [ ] The `DemonstrateMutationBug()` method shows the value “vanishing” after key mutation and carries a comment explaining the mechanism.
- [ ] The `DemonstrateBrokenContract()` method shows `O(n)` degradation under a constant/missing `GetHashCode` and carries a comment.
- [ ] The `dotnet run` output matches the expectation: found value, `false` for missing, `Count == 2`, `api.example.com => 2`, and explanations for both demonstrations.

#### Hints (no direct answer)
- `HashCode.Combine` accepts several arguments and mixes them itself; you do not need a custom formula.
- For a `readonly struct`, the `==`/`!=` operators do not appear automatically from `IEquatable<T>` — define them explicitly, delegating to `Equals`.
- `StringComparer.OrdinalIgnoreCase` is a ready-made `IEqualityComparer<string>` (and `IComparer<string>`); pass it straight to the constructor.
- To “lose” a value via mutation, it is enough to change a field used by `GetHashCode` after `dict[key] = value`. Then try `dict.TryGetValue(key, out _)` on the same instance — you will be surprised.
- To demonstrate `O(n)`, put many entries into a dictionary whose key type has `GetHashCode() => 0`: lookup stays correct but every entry lands in one bucket.
- `TryGetValue` returns a `bool` and fills the `out` parameter — use both: the first to branch, the second to increment the counter.
- Do not confuse `Dictionary<TKey,TValue>.Keys`/`Values` with the pairs themselves — iterate pairs with `KeyValuePair` in `foreach`.

#### Reference solution walk-through
```csharp
// Endpoint.cs — immutable key implementing IEquatable<T>.
// readonly struct guarantees fields cannot change after creation — the hash is stable.
public readonly struct Endpoint : IEquatable<Endpoint>
{
    public string Method { get; }   // HTTP method, e.g. "GET"
    public string Path   { get; }   // request path, e.g. "/users/42"

    public Endpoint(string method, string path)
    {
        Method = method;
        Path   = path;
    }

    // Type-safe comparison without boxing (IEquatable<T>).
    public bool Equals(Endpoint other)
        => string.Equals(Method, other.Method, StringComparison.Ordinal)
           && string.Equals(Path, other.Path, StringComparison.Ordinal);

    // object.Equals delegates to the typed version; null/wrong type → false.
    public override bool Equals(object? obj) => obj is Endpoint e && Equals(e);

    // Contract — same fields as in Equals; mixing via HashCode.Combine.
    public override int GetHashCode() => HashCode.Combine(Method, Path);

    // Operators for convenience; delegate to Equals to keep logic in sync.
    public static bool operator ==(Endpoint a, Endpoint b) => a.Equals(b);
    public static bool operator !=(Endpoint a, Endpoint b) => !a.Equals(b);
}

// Intentionally broken type to demonstrate the contract.
public sealed class ConstantHashEndpoint
{
    public string Method { get; init; }
    public string Path   { get; init; }

    public override bool Equals(object? obj)
        => obj is ConstantHashEndpoint e
           && string.Equals(Method, e.Method, StringComparison.Ordinal)
           && string.Equals(Path, e.Path, StringComparison.Ordinal);

    // Constant hash — correct, but all entries in one bucket → O(n).
    public override int GetHashCode() => 0;
}

// Mutable key to demonstrate the mutation bug.
public sealed class MutableEndpoint
{
    public string Method { get; set; }
    public string Path   { get; set; }

    public override bool Equals(object? obj)
        => obj is MutableEndpoint e
           && string.Equals(Method, e.Method, StringComparison.Ordinal)
           && string.Equals(Path, e.Path, StringComparison.Ordinal);

    public override int GetHashCode() => HashCode.Combine(Method, Path);
}
```

```csharp
// Program.cs — top-level statements, C# 12 / .NET 8
using System;
using System.Collections.Generic;

// 1. Response cache with preset capacity; key is the custom Endpoint type.
var cache = new Dictionary<Endpoint, string>(capacity: 16)
{
    [new Endpoint("GET", "/users/42")] = "200 OK: user 42",
    [new Endpoint("POST", "/orders")]  = "201 Created: order #7",
};

// 2. TryGetValue — single pass, unlike ContainsKey + indexer.
if (cache.TryGetValue(new Endpoint("GET", "/users/42"), out var body))
{
    Console.WriteLine($"found → {body}");
}
else
{
    Console.WriteLine("not found");
}

// 3. Check for a missing key.
bool hit = cache.TryGetValue(new Endpoint("DELETE", "/users/42"), out _);
Console.WriteLine($"DELETE in cache? {hit}");

// 4. HashSet of unique parameter names, case-insensitive via a BCL comparer.
var paramNames = new HashSet<string>(StringComparer.OrdinalIgnoreCase)
{
    "id", "ID", "Id", "page",
};
Console.WriteLine($"unique params: {paramNames.Count}"); // → 2

// 5. Per-host request counter, also case-insensitive.
var hostCounts = new Dictionary<string, int>(StringComparer.OrdinalIgnoreCase);
RegisterHit(hostCounts, "api.example.com");
RegisterHit(hostCounts, "API.Example.COM");   // same key
RegisterHit(hostCounts, "cdn.example.com");
foreach (var (host, count) in hostCounts)
{
    Console.WriteLine($"host {host} => {count}");
}

// 6. Increments the counter in a single TryGetValue.
static void RegisterHit(Dictionary<string, int> counts, string host)
{
    if (counts.TryGetValue(host, out var current))
    {
        counts[host] = current + 1;
    }
    else
    {
        counts[host] = 1;
    }
}

DemonstrateMutationBug();
DemonstrateBrokenContract();

// 7. Key mutated after insertion — the value is "lost" in its bucket.
static void DemonstrateMutationBug()
{
    var dict = new Dictionary<MutableEndpoint, string>();
    var key  = new MutableEndpoint { Method = "GET", Path = "/old" };
    dict[key] = "secret value";

    // Mutate a field used by GetHashCode — the hash changes, the bucket is stale.
    key.Path = "/new";

    bool found = dict.TryGetValue(key, out var v);
    Console.WriteLine(
        $"found after mutation? {found} " +
        $"({(found ? v : "lost")})");
    // Expected found == false — the hash no longer points to the right bucket.
}

// 8. Constant hash — correctness preserved, but everything in one bucket → O(n).
static void DemonstrateBrokenContract()
{
    var dict = new Dictionary<ConstantHashEndpoint, int>();
    for (int i = 0; i < 1000; i++)
    {
        dict[new ConstantHashEndpoint
        {
            Method = "GET",
            Path    = $"/item/{i}"
        }] = i;
    }

    // Lookup still finds the entry (Equals works) but scans the entire bucket.
    bool found = dict.TryGetValue(
        new ConstantHashEndpoint { Method = "GET", Path = "/item/500" },
        out var value);
    Console.WriteLine(
        $"constant hash — found? {found}, value={value}. " +
        $"But complexity is O(n), not O(1).");
}
```

Line-by-line walk-through. The `Endpoint` type is declared `readonly struct`: this gives two guarantees from the lesson at once — value semantics (no allocations) and field immutability (the hash is stable after construction). Implementing `IEquatable<Endpoint>` with a typed `Equals(Endpoint other)` removes the boxing that would otherwise occur when the hash table calls `object.Equals(object)` — this is a best practice stated explicitly in the lesson. The overridden `Equals(object?)` uses pattern matching (`obj is Endpoint e`) and delegates to the typed version so the comparison logic lives in a single place. `GetHashCode()` is built with `HashCode.Combine(Method, Path)` — the lesson explicitly warns against hand-rolled formulas like `x * 31 + y` because they produce pathological collisions on close inputs. Crucially, `GetHashCode` includes **the same fields** as `Equals`: this is the literal fulfilment of the contract. The `==`/`!=` operators are defined explicitly because `IEquatable<T>` does not generate them automatically; they delegate to `Equals` to stay in sync with `Equals(object?)`.

In `Program.cs` the cache is created with `capacity: 16` — the best practice “pre-set the capacity when the size is known”, which removes extra rehashing during growth. Lookup uses `TryGetValue` rather than `ContainsKey` plus indexer: the former makes a single pass through the hash table, the latter makes two, and the difference matters in hot paths. The `HashSet<string>` for parameter names and the `Dictionary<string,int>` for hosts both receive `StringComparer.OrdinalIgnoreCase` in their constructors — the lesson’s recommended way to deal with case-insensitive string keys, instead of manual `ToLower()` normalisation that litters the code and breeds bugs. The `RegisterHit` method illustrates the idiom “`TryGetValue` → increment the `out` value, or initialise with one”.

The `DemonstrateMutationBug` method materialises the lesson’s main trap: after `dict[key] = value` we change `key.Path`, which participates in `GetHashCode`. The object’s hash changes, but the entry physically stayed in the bucket corresponding to the **old** hash. The subsequent `TryGetValue(key, ...)` computes the **new** hash and looks in a different bucket — and does not find the entry, even though the object instance is the very same one. This is the “vanished” value. The `DemonstrateBrokenContract` method shows the opposite extreme: `GetHashCode` returns a constant, the contract is formally satisfied (equal objects indeed produce equal hashes — everyone has the same one), and lookup is **correct**, but all entries land in one bucket, turning average `O(1)` into worst-case `O(n)`. The lesson stresses that `O(1)` is average time, and a poor comparer can wipe out every advantage of a hash table. Together, the two demonstrations give the student intuition about the two failure modes of a hash table: “never found at all” (forward contract broken) and “found, but slowly” (hash is correct but useless).

#### Going deeper (bonus)
1. Implement a custom `IEqualityComparer<Endpoint>` that compares paths **ignoring the trailing slash**: `"/users"` and `"/users/"` are treated as the same key. Compute the hash so that it matches for both forms (for example, normalise the path before `HashCode.Combine`). Pass the comparer to the `Dictionary` constructor and verify key merging.
2. Rewrite `Endpoint` as a `record` (not a `readonly struct`) and compare behaviour: a `record` auto-implements `Equals`/`GetHashCode` based on values. Measure with `BenchmarkDotNet` the allocation difference between `readonly struct` and `record` (class) across one million `TryGetValue` calls. Explain why `readonly struct` plus `IEquatable<T>` is more efficient for hot paths.
3. Add LRU eviction: store the last-access time and, when `capacity` is exceeded, drop the oldest entries. Think about why removing a key from a `Dictionary` keeps the hash table correct (unlike mutating a key).
4. Stress-test the edge case: 10 000 keys with `Path = "/item/" + i`, but under the comparer from step 1. Compare lookup time under `HashCode.Combine` versus a constant hash — numerically confirm the `O(n)` degradation.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Проект `RequestCache` собирается под .NET 8 / C# 12 без ошибок.
- [ ] (RU) `Endpoint` — `readonly struct` с `IEquatable<Endpoint>`, согласованными `Equals`/`GetHashCode`, операторами `==`/`!=`.
- [ ] (RU) `HashCode.Combine` используется для хеша; поля ключа неизменяемы.
- [ ] (RU) `Dictionary<Endpoint, string>` создан с `capacity`.
- [ ] (RU) `TryGetValue` применяется вместо `ContainsKey` + индексатор.
- [ ] (RU) `HashSet<string>` и `Dictionary<string,int>` используют `StringComparer.OrdinalIgnoreCase`.
- [ ] (RU) `DemonstrateMutationBug` и `DemonstrateBrokenContract` реализованы и прокомментированы.
- [ ] (RU) Вывод `dotnet run` соответствует ожидаемому.
- [ ] (EN) The `RequestCache` project builds under .NET 8 / C# 12 with no errors.
- [ ] (EN) `Endpoint` is a `readonly struct` with `IEquatable<Endpoint>`, consistent `Equals`/`GetHashCode`, and `==`/`!=` operators.
- [ ] (EN) `HashCode.Combine` is used for the hash; key fields are immutable.
- [ ] (EN) The `Dictionary<Endpoint, string>` is created with a `capacity`.
- [ ] (EN) `TryGetValue` is used instead of `ContainsKey` plus indexer.
- [ ] (EN) `HashSet<string>` and `Dictionary<string,int>` use `StringComparer.OrdinalIgnoreCase`.
- [ ] (EN) `DemonstrateMutationBug` and `DemonstrateBrokenContract` are implemented and commented.
- [ ] (EN) The `dotnet run` output matches the expectation.

#### Ресурсы / Resources
- [Microsoft Learn — Dictionary<TKey,TValue>](https://learn.microsoft.com/dotnet/api/system.collections.generic.dictionary-2)
- [Microsoft Learn — HashSet<T>](https://learn.microsoft.com/dotnet/api/system.collections.generic.hashset-1)
- [Microsoft Learn — Object.GetHashCode](https://learn.microsoft.com/dotnet/api/system.object.gethashcode)
- [Microsoft Learn — IEquatable<T>](https://learn.microsoft.com/dotnet/api/system.iequatable-1)
- [Microsoft Learn — IEqualityComparer<T>](https://learn.microsoft.com/dotnet/api/system.collections.generic.iequalitycomparer-1)
- [Microsoft Learn — StringComparer](https://learn.microsoft.com/dotnet/api/system.stringcomparer)
- [Microsoft Learn — HashCode.Combine](https://learn.microsoft.com/dotnet/api/system.hashcode.combine)
