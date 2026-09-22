---
[← К уроку M04-L05](lesson-M04-L05-static-readonly-const.md) | [⬆ К модулю M04](../README.md) | [Следующее ДЗ →](homework-M04-L06-this-indexers.md)
---

### Домашнее задание M04-L05: static vs instance, const vs readonly / Homework M04-L05: static vs instance, const vs readonly

**Урок / Lesson:** M04-L05
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) Научиться различать принадлежность члена типу (`static`) или экземпляру (`instance`), выбирать между константой времени компиляции (`const`) и полем, доступным только для чтения на этапе выполнения (`readonly`), и понимать, какие значения вообще не должны быть кодом, а должны становиться конфигурацией. (EN) Learn to distinguish members that belong to the type (`static`) from members that belong to an instance (`instance`), choose between a compile-time constant (`const`) and a runtime read-only field (`readonly`), and understand which values should not be code at all but configuration instead.

#### Связь с уроком / Connection to the lesson
(RU) Это задание закрепляет ключевые разграничения урока M04-L05: «принадлежность типу vs принадлежность объекту» и «вычисление на компиляции vs вычисление в рантайме». Вы будете моделировать компонент реального приложения и на каждом шаге явно обосновывать выбор модификатора, опираясь на риски частичной перекомпиляции, потокобезопасность и природу значения (истинная константа, вычисляемое один раз значение, изменяемая конфигурация).
(EN) This homework reinforces the core distinctions of lesson M04-L05: "belongs to the type vs belongs to the object" and "computed at compile time vs computed at runtime". You will model a component of a real application and at every step explicitly justify the chosen modifier, relying on the risks of partial recompilation, thread safety, and the nature of the value (true constant, value computed once, mutable configuration).

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Представьте, что вы присоединяетесь к команде, пишущей внутренний SDK для работы с геоданными и внешним API маршрутизации. В репозитории уже есть набросок, но предыдущий разработчик не различал `const`, `readonly` и конфигурацию: он «зашивал» URL сервиса как `const string`, использовал изменяемые `static` поля без синхронизации и пытался объявить `const` поле со значением `DateTime.Now`. В результате после каждого изменения URL приходилось перекомпилировать и переиздавать все зависимые сборки, а под нагрузкой счётчик запросов иногда терял инкременты. Команда просит вас переписать ключевой модуль так, чтобы каждый модификатор использовался по назначению.

В этом задании вы построите небольшой, но реалистичный компонент `GeoCalculator`: статический утилитный класс с истинными константами и вычисляемыми один раз значениями, экземплярный класс `RouteSession` с `readonly` полями и instance-состоянием, а также класс `RoutingApiClient`, который получает все внешние параметры (URL, таймаут, лимит запросов) из конфигурации, а не из кода. Попутно вы на практике столкнётесь с тем, почему `const` нельзя сделать instance, почему `readonly` можно присвоить только в конструкторе и почему изменяемое `static` поле нужно защищать `Interlocked` или `lock`.

Задание намеренно приближено к жизни: оно учит не просто «правилу языка», а инженерному выбору, где цена ошибки — устаревшее значение в продакшене или гонка данных в многопоточной среде. Выполняя шаги, постоянно возвращайтесь к вопросу: «Может ли это значение измениться без перекомпиляции всего решения?» — это главный критерий выбора.

#### Что нужно сделать (пошагово)
1. Создайте новый консольный проект .NET 8 командой `dotnet new console -n GeoCalculator -o GeoCalculator --framework net8.0`. Перейдите в папку проекта `cd GeoCalculator` и проверьте версию SDK командой `dotnet --version` — ожидается 8.x.

2. Добавьте в проект файл `MathHelper.cs` и опишите в нём статический класс `MathHelper`. Внутри объявите: `public const double EarthRadiusKm = 6371.0;` (истинная константа — радиус Земли в километрах не меняется между релизами), `public const double Pi = Math.PI;` нельзя использовать напрямую, поэтому используйте литерал `3.141592653589793` или ссылку на `System.Math.PI` через `static readonly` (объясните в комментарии, почему `const` не может ссылаться на `Math.PI` в старых версиях языка и как это ведёт себя в C# 12). Добавьте `public const int MaxWaypointsPerRoute = 25;` и `public const string DefaultLocale = "en-US";`.

3. В том же классе добавьте `public static readonly DateTime BuildDate = DateTime.UtcNow;` и `public static readonly int ProcessorCount = Environment.ProcessorCount;`. В комментарии RU+EN поясните, почему эти значения не могут быть `const`: они вычисляются в рантайме при инициализации типа.

4. Реализуйте статический метод `public static double HaversineDistance(double lat1, double lon1, double lat2, double lon2)`, возвращающий расстояние между двумя точками в километрах по формуле Гаверсинуса. Используйте внутри `EarthRadiusKm` как `const` (он будет встроен в IL как литерал). Добавьте приватное статическое изменяемое поле `private static long _callCount;` и инкрементируйте его потокобезопасно через `Interlocked.Increment(ref _callCount)`. Откройте его через `public static long CallCount => _callCount;`.

5. Создайте файл `RouteSession.cs` с экземплярным `public sealed class RouteSession`. Поля: `public readonly string SessionId;` (задаётся один раз в конструкторе), `private readonly DateTime _startedAt;`, приватное изменяемое `private int _waypoints;`. Конструктор `RouteSession(string sessionId)` инициализирует `readonly` поля. Метод `public void AddWaypoint(double lat, double lon)` увеличивает `_waypoints` и проверяет, что не превышен `MathHelper.MaxWaypointsPerRoute` (бросает `InvalidOperationException`). Попытайтесь сознательно написать `SessionId = "x";` внутри метода — убедитесь по ошибке компилятора CS0191, что `readonly` можно присвоить только в конструкторе или при объявлении, и верните код обратно.

6. Создайте `RoutingApiClient.cs`: класс получает `HttpClient`, URL, таймаут и лимит запросов через конструктор из конфигурации. Все эти поля — `private readonly`. Никаких `const` для URL. Добавьте `appsettings.json` с секцией `Routing` и продемонстрируйте чтение через `Microsoft.Extensions.Configuration.Json` (пакет `Microsoft.Extensions.Configuration` и `Microsoft.Extensions.Configuration.Json`).

7. В `Program.cs` покажите использование: вызовите `MathHelper.HaversineDistance(...)`, создайте две `RouteSession`, убедитесь, что `MathHelper.CallCount` растёт, а `_waypoints` у каждой сессии свой. Выведите `MathHelper.BuildDate` и `MathHelper.ProcessorCount`. Запустите `dotnet run` и зафиксируйте ожидаемый вывод в комментариях.

8. Соберите проект командой `dotnet build` — должно быть 0 ошибок и 0 предупреждений. Запустите `dotnet run` и убедитесь, что вывод совпадает с ожидаемым.

#### Требования к решению
- Целевая платформа: .NET 8, язык C# 12. Проект должен собираться с `dotnet build` без ошибок и предупреждений (включая уровень `Nullable` enabled — добавьте `<Nullable>enable</Nullable>` в `.csproj`).
- Каждый модификатор должен быть использован по назначению и снабжён комментарием-обоснованием на двух языках (RU+EN): почему именно `const`, `static readonly`, `readonly` или конфигурация. Комментарии должны ссылаться на природу значения, а не на личное предпочтение.
- Запрещено: `const` для значений, которые могут меняться между релизами (URL, таймауты, лимиты, ключи); `const` для instance-полей; присвоение `readonly` вне конструктора; изменяемые `static` поля без синхронизации в многопоточных сценариях; создание экземпляра статического класса.
- URL сервиса, таймаут и лимит запросов обязаны поступать из `appsettings.json` через `IConfiguration`/`IOptions`-подобный паттерн (достаточно `IConfiguration` для консольного приложения). Значение по умолчанию можно задать в коде как fallback.
- Код должен быть потокобезопасным в части изменяемого `static` поля `_callCount`: инкремент через `Interlocked.Increment`. Если добавите кэш — защищайте его `lock` или используйте `ConcurrentDictionary`.
- Имена типов и методов — из задания; сигнатуры не меняйте. Покрытие тестами не обязательно, но приветствуется отдельный проект `*.Tests` с парой проверок на `HaversineDistance` и `MaxWaypointsPerRoute`.
- Файлы должны быть разнесены по одному классу на файл (style convention курса).

#### Тонкости и подводные камни
- **`const` встраивается в вызывающий код.** Если `EarthRadiusKm` объявлен `const` и вы поменяете значение в библиотеке `GeoCalculator.Lib`, то консольное приложение, не перекомпилированное, продолжит использовать старое значение. Поэтому `const` — только для величин, изменение которых требует перекомпиляции всего решения и это приемлемо. Для значений, которые могут корректироваться (например, более точный радиус Земли в новой модели), безопаснее `static readonly`.
- **`const` неявно статичен.** Нельзя написать `public const int X = 5;` рассчитывая на per-instance значение — это поле типа. Если значение должно быть привязано к экземпляру и неизменно после создания, используйте `readonly` (`public readonly int X;` с присвоением в конструкторе).
- **`readonly` присваивается только при объявлении или в конструкторе.** Попытка записать `SessionId = "x";` внутри метода даёт CS0191. Конструкторов может быть несколько — в каждом можно присвоить, но ровно один путь исполнения на объект.
- **`static readonly` вычисляется при инициализации типа.** Поле `BuildDate` получит значение `DateTime.UtcNow` в момент первого обращения к типу, а не «при компиляции». Это и источник гибкости (значение свежее), и источник неявной зависимости (инициализация типа может бросить исключение и «сломать» весь тип — обрабатывайте аккуратно).
- **Потокобезопасность статических полей.** `_callCount++` — не атомарная операция (чтение-изменение-запись); под нагрузкой инкременты теряются. Используйте `Interlocked.Increment(ref _callCount)` или `lock`. Инициализация `static readonly` полей сама по себе потокобезопасна (runtime гарантирует через `beforefieldinit`/тип-инициализатор), но изменяемые поля защищать нужно вручную.
- **`const` в атрибутах.** Значения параметров атрибутов обязаны быть константами времени компиляции — здесь `const` уместен, а `static readonly` не подойдёт (компилятор выдаст ошибку). Если параметр атрибута — перечисление или литерал, `const` — правильный выбор.
- **`readonly struct`.** В C# 7+ можно пометить всю структуру `readonly struct`, что гарантирует неизменяемость и позволяет компилятору оптимизировать `in`-параметры. Для маленьких геометрических структур (например, `readonly struct GeoPoint`) это хороший приём — упомяните его в бонусе.
- **Конфигурация против `const`.** URL, таймаут, лимиты, ключи — это конфигурация. Зашивать их `const` — значит требовать перекомпиляцию при каждом изменении окружения. Читайте их в рантайме и инжектируйте через DI/IOptions.

#### Критерии приёмки
- [ ] Проект `GeoCalculator` собирается `dotnet build` без ошибок и предупреждений под .NET 8 / C# 12.
- [ ] `MathHelper` — статический класс; попытка `new MathHelper()` вызывает ошибку компиляции CS0723 (демонстрируется в закомментированном коде).
- [ ] В `MathHelper` есть `const` поля `EarthRadiusKm`, `MaxWaypointsPerRoute`, `DefaultLocale` с RU+EN комментарием-обоснованием.
- [ ] В `MathHelper` есть `static readonly` поля `BuildDate` и `ProcessorCount` с объяснением, почему они не могут быть `const`.
- [ ] Метод `HaversineDistance` использует `EarthRadiusKm` и возвращает корректное расстояние (проверка на паре точек).
- [ ] Изменяемое `static` поле `_callCount` инкрементируется через `Interlocked.Increment`, доступно через `CallCount`.
- [ ] `RouteSession` имеет `readonly` поля `SessionId` и `_startedAt`, присваиваемые только в конструкторе.
- [ ] `RouteSession.AddWaypoint` проверяет `MathHelper.MaxWaypointsPerRoute` и бросает `InvalidOperationException` при превышении.
- [ ] В коде есть закомментированная попытка присвоить `readonly` вне конструктора с пометкой об ошибке CS0191.
- [ ] `RoutingApiClient` получает URL, таймаут и лимит из `appsettings.json` через `IConfiguration`; все поля `private readonly`.
- [ ] В `appsettings.json` есть секция `Routing` с `BaseUrl`, `TimeoutSeconds`, `MaxRequestsPerMinute`.
- [ ] `Program.cs` демонстрирует вызов `HaversineDistance`, создание двух сессий, рост `CallCount`, вывод `BuildDate`/`ProcessorCount`.
- [ ] `dotnet run` выводит ожидаемые значения, совпадающие с комментариями в коде.
- [ ] Каждый модификатор снабжён комментарием-обоснованием на двух языках.
- [ ] Нет ни одного `const` для значений, зависящих от окружения (URL, таймаут, лимиты).

#### Подсказки (без прямого ответа)
- Задайте себе вопрос: «Изменится ли это значение без перекомпиляции всего решения?» Если да — это не `const`.
- Для `const` проверьте, что правая часть — литерал или константное выражение; `Math.PI` — это `public const double`, поэтому `const double Pi = Math.PI;` допустимо в C# 12, но `DateTime.UtcNow` — нет.
- Потокобезопасный инкремент целого — `Interlocked.Increment`; для `long` подпись та же.
- `readonly` можно присвоить в любом конструкторе, но только один раз по пути исполнения; несколько конструкторов, вызывающих друг друга через `: this(...)`, — нормальная практика.
- Чтение `appsettings.json` в консоли: добавьте пакет `Microsoft.Extensions.Configuration.Json`, используйте `new ConfigurationBuilder().AddJsonFile("appsettings.json").Build()`.
- Скопируйте `appsettings.json` в выходной каталог: `<None Update="appsettings.json"><CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory></None>` в `.csproj`.

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 — эталонное решение / reference solution
using System;
using System.Threading;

// Статический класс: нельзя инстанцировать, нельзя унаследовать.
// Static class: cannot be instantiated, cannot be inherited.
public static class MathHelper
{
    // const — истинная константа, вшивается в вызывающий код как литерал.
    // const — true constant, baked into the caller as a literal.
    // Радиус Земли не меняется между релизами — уместен const.
    // Earth radius does not change between releases — const is appropriate.
    public const double EarthRadiusKm = 6371.0;

    // const для перечислимого лимита и локали по умолчанию.
    // const for a countable limit and default locale.
    public const int MaxWaypointsPerRoute = 25;
    public const string DefaultLocale = "en-US";

    // static readonly — вычисляется один раз при инициализации типа (runtime).
    // static readonly — computed once at type initialization (runtime).
    // Не может быть const: DateTime.UtcNow и Environment.ProcessorCount — runtime-значения.
    // Cannot be const: DateTime.UtcNow and Environment.ProcessorCount are runtime values.
    public static readonly DateTime BuildDate = DateTime.UtcNow;
    public static readonly int ProcessorCount = Environment.ProcessorCount;

    // Изменяемое статическое поле — защищаем через Interlocked.
    // Mutable static field — protected with Interlocked.
    private static long _callCount;
    public static long CallCount => _callCount;

    public static double HaversineDistance(double lat1, double lon1, double lat2, double lon2)
    {
        Interlocked.Increment(ref _callCount); // потокобезопасный инкремент / thread-safe increment

        double ToRadians(double deg) => deg * Math.PI / 180.0;
        double dLat = ToRadians(lat2 - lat1);
        double dLon = ToRadians(lon2 - lon1);
        double a = Math.Sin(dLat / 2) * Math.Sin(dLat / 2) +
                   Math.Cos(ToRadians(lat1)) * Math.Cos(ToRadians(lat2)) *
                   Math.Sin(dLon / 2) * Math.Sin(dLon / 2);
        double c = 2 * Math.Atan2(Math.Sqrt(a), Math.Sqrt(1 - a));
        return EarthRadiusKm * c; // const EarthRadiusKm встроен как литерал / const baked as literal
    }
}

// Экземплярный класс с readonly полями / instance class with readonly fields.
public sealed class RouteSession
{
    public readonly string SessionId;       // задаётся один раз в конструкторе / set once in ctor
    private readonly DateTime _startedAt;   // неизменно после создания / immutable after creation
    private int _waypoints;                 // изменяемое instance-состояние / mutable instance state

    public RouteSession(string sessionId)
    {
        SessionId = sessionId;   // OK: присвоение readonly в конструкторе / OK: readonly assignment in ctor
        _startedAt = DateTime.UtcNow;
        _waypoints = 0;
    }

    // public void BreakIt() { SessionId = "x"; } // CS0191: readonly нельзя присвоить вне конструктора
    // CS0191: a readonly field cannot be assigned outside a constructor

    public DateTime StartedAt => _startedAt;
    public int Waypoints => _waypoints;

    public void AddWaypoint(double lat, double lon)
    {
        if (_waypoints >= MathHelper.MaxWaypointsPerRoute) // const используется как литерал
            throw new InvalidOperationException(
                $"Превышен лимит точек маршрута ({MathHelper.MaxWaypointsPerRoute}). / " +
                $"Waypoint limit exceeded ({MathHelper.MaxWaypointsPerRoute}).");

        _waypoints++;
    }
}

// Все внешние параметры приходят из конфигурации, а не из const.
// All external parameters come from configuration, not from const.
public sealed class RoutingApiClient
{
    private readonly HttpClient _http;
    private readonly Uri _endpoint;          // readonly — задан в конструкторе из конфигурации
    private readonly TimeSpan _timeout;
    private readonly int _maxRequestsPerMinute;

    public RoutingApiClient(HttpClient http, string baseUrl, int timeoutSeconds, int maxRequestsPerMinute)
    {
        _http = http;
        _endpoint = new Uri(baseUrl);                          // runtime-значение / runtime value
        _timeout = TimeSpan.FromSeconds(timeoutSeconds);
        _maxRequestsPerMinute = maxRequestsPerMinute;
        _http.Timeout = _timeout;
    }

    public Uri Endpoint => _endpoint;
    public int MaxRequestsPerMinute => _maxRequestsPerMinute;
}

class Program
{
    static void Main()
    {
        // const и static readonly доступны через тип без new.
        // const and static readonly accessed via the type, no new.
        Console.WriteLine($"EarthRadiusKm = {MathHelper.EarthRadiusKm}");  // 6371
        Console.WriteLine($"MaxWaypoints  = {MathHelper.MaxWaypointsPerRoute}"); // 25
        Console.WriteLine($"BuildDate     = {MathHelper.BuildDate}");      // момент запуска / startup moment
        Console.WriteLine($"ProcessorCount= {MathHelper.ProcessorCount}"); // число ядер / core count

        double moscowToSpb = MathHelper.HaversineDistance(55.7558, 37.6173, 59.9343, 30.3351);
        Console.WriteLine($"MSQ→SPB       = {moscowToSpb:F1} km");          // ~635 km
        Console.WriteLine($"CallCount     = {MathHelper.CallCount}");      // 1

        var s1 = new RouteSession("order-42");
        var s2 = new RouteSession("order-43");
        s1.AddWaypoint(55.75, 37.61);
        s1.AddWaypoint(55.80, 37.70);
        s2.AddWaypoint(59.93, 30.34);
        Console.WriteLine($"{s1.SessionId}: {s1.Waypoints} wp");           // order-42: 2 wp
        Console.WriteLine($"{s2.SessionId}: {s2.Waypoints} wp");           // order-43: 1 wp

        // var mh = new MathHelper(); // CS0723: нельзя создать экземпляр статического класса
    }
}
```

Разбор по строкам. `MathHelper` объявлен `static` — это гарантирует, что никто не создаст его экземпляр и не унаследуется (CS0723 при попытке `new`). `EarthRadiusKm`, `MaxWaypointsPerRoute` и `DefaultLocale` — `const`: их значения известны на этапе компиляции и не меняются между релизами, поэтому встраивание литерала в вызывающий код безопасно. В методе `HaversineDistance` видно ключевое следствие: `EarthRadiusKm * c` компилируется так, будто написано `6371.0 * c` — ссылка на поле отсутствует в IL. `BuildDate` и `ProcessorCount` сделаны `static readonly`, потому что `DateTime.UtcNow` и `Environment.ProcessorCount` вычисляются только в рантайме — `const` здесь невозможен, компилятор отказал бы с CS0133. Поле `_callCount` изменяемое и статическое; инкремент `++` не атомарный, поэтому применён `Interlocked.Increment` — это закрывает гонку данных в многопоточной среде. В `RouteSession` поля `SessionId` и `_startedAt` — `readonly`: они задаются один раз в конструкторе и не могут быть перезаписаны в методе (демонстрация CS0191 в закомментированном `BreakIt`). Изменяемое `_waypoints` остаётся обычным instance-полем, потому что должно расти с каждым вызовом `AddWaypoint`. Проверка лимита использует `MathHelper.MaxWaypointsPerRoute` — `const` встроен в условие как литерал, что допустимо, поскольку лимит — истинная константа домена. В `RoutingApiClient` все внешние параметры (`baseUrl`, `timeoutSeconds`, `maxRequestsPerMinute`) приходят параметрами конструктора из конфигурации и хранятся в `private readonly` полях: здесь `const` был бы ошибкой, ведь URL и таймаут меняются между окружениями без перекомпиляции. Итог: каждый модификатор выбран по природе значения, а не по привычке, что и есть цель урока.

#### Задания на углубление (бонус)
1. Вынесите `MathHelper` и `RouteSession` в отдельную библиотеку классов `GeoCalculator.Lib` (`dotnet new classlib`). В консольном приложении сошлитесь на неё. Затем поменяйте значение `EarthRadiusKm` в библиотеке, пересоберите только библиотеку и покажите, что консольное приложение без перекомпиляции продолжает использовать старое значение — демонстрация встраивания `const`. После этого переведите поле в `static readonly` и покажите, что новое значение подхватывается.
2. Реализуйте `readonly struct GeoPoint { public readonly double Lat; public readonly double Lon; ... }` и используйте его как параметр `in GeoPoint` в `HaversineDistance`. Сравните с `class`-версией: что даёт `readonly struct` для защиты от мутаций и для производительности `in`-параметров?
3. Добавьте ленивый потокобезопасный кэш вычисленных расстояний через `ConcurrentDictionary<(GeoPoint, GeoPoint), double>` или `Lazy<T>`. Покажите, что `static readonly` инициализация кэша потокобезопасна, а изменяемое содержимое защищено структурой данных.
4. Переведите чтение конфигурации на `IOptions<RoutingOptions>` через `Microsoft.Extensions.DependencyInjection` и `Microsoft.Extensions.Options.ConfigurationExtensions`. Покажите, что `RoutingOptions` — это `sealed class` с `readonly` свойствами или `init`-сеттерами, и объясните, почему `const` здесь был бы неуместен даже для «значения по умолчанию».

---

## Statement in English / Постановка на английском

#### Context & motivation
Imagine you are joining a team that maintains an internal SDK for geospatial data and an external routing API. The repository already contains a draft, but the previous developer did not distinguish between `const`, `readonly`, and configuration: he baked the service URL as a `const string`, used mutable `static` fields without synchronization, and even tried to declare a `const` field initialized to `DateTime.Now`. As a consequence, every URL change forced a full recompile and republication of all dependent assemblies, and under load the request counter occasionally lost increments. The team asks you to rewrite the core module so that every modifier is used for its intended purpose.

In this assignment you will build a small but realistic `GeoCalculator` component: a static utility class with true constants and values computed once, an instance `RouteSession` class with `readonly` fields and instance state, and a `RoutingApiClient` class that receives every external parameter (URL, timeout, rate limit) from configuration rather than from code. Along the way you will confront in practice why `const` cannot be an instance field, why `readonly` can be assigned only in a constructor, and why a mutable `static` field must be protected with `Interlocked` or `lock`.

The exercise is deliberately close to real life: it teaches not just a "language rule" but an engineering choice, where the cost of a mistake is a stale value in production or a data race under concurrency. As you work through the steps, keep returning to the question: "Can this value change without recompiling the entire solution?" — that is the main criterion for choosing a modifier.

#### What to do step by step
1. Create a new .NET 8 console project with `dotnet new console -n GeoCalculator -o GeoCalculator --framework net8.0`. Move into the project folder with `cd GeoCalculator` and verify the SDK version with `dotnet --version` — expect 8.x.

2. Add a file `MathHelper.cs` and define a static class `MathHelper` inside it. Declare `public const double EarthRadiusKm = 6371.0;` (a true constant — the Earth radius in kilometers does not change between releases), `public const int MaxWaypointsPerRoute = 25;`, and `public const string DefaultLocale = "en-US";`. For `Pi`, explain in a comment why historically `const double Pi = Math.PI;` was not allowed and how it behaves in C# 12 (today `Math.PI` is itself `public const double`, so the assignment is legal; if you want to be conservative, use the literal `3.141592653589793`).

3. In the same class add `public static readonly DateTime BuildDate = DateTime.UtcNow;` and `public static readonly int ProcessorCount = Environment.ProcessorCount;`. In a RU+EN comment explain why these values cannot be `const`: they are evaluated at runtime when the type is initialized.

4. Implement a static method `public static double HaversineDistance(double lat1, double lon1, double lat2, double lon2)` that returns the great-circle distance in kilometers between two points using the Haversine formula. Use `EarthRadiusKm` inside; because it is `const`, the value is inlined into the IL as a literal. Add a private mutable static field `private static long _callCount;` and increment it in a thread-safe way with `Interlocked.Increment(ref _callCount)`. Expose it via `public static long CallCount => _callCount;`.

5. Create `RouteSession.cs` with an instance `public sealed class RouteSession`. Fields: `public readonly string SessionId;` (assigned exactly once in the constructor), `private readonly DateTime _startedAt;`, and a private mutable `private int _waypoints;`. The constructor `RouteSession(string sessionId)` initializes the `readonly` fields. A method `public void AddWaypoint(double lat, double lon)` increments `_waypoints` and checks that `MathHelper.MaxWaypointsPerRoute` is not exceeded (throwing `InvalidOperationException`). Deliberately try to write `SessionId = "x";` inside a method, observe compiler error CS0191 confirming that a `readonly` field can be assigned only in a constructor or at declaration, then revert the code.

6. Create `RoutingApiClient.cs`: the class takes `HttpClient`, URL, timeout, and rate limit through its constructor from configuration. All those fields are `private readonly`. There must be no `const` for the URL. Add an `appsettings.json` with a `Routing` section and demonstrate reading it through `Microsoft.Extensions.Configuration.Json` (packages `Microsoft.Extensions.Configuration` and `Microsoft.Extensions.Configuration.Json`).

7. In `Program.cs` demonstrate usage: call `MathHelper.HaversineDistance(...)`, create two `RouteSession` instances, confirm that `MathHelper.CallCount` grows while `_waypoints` stays per-session, and print `MathHelper.BuildDate` and `MathHelper.ProcessorCount`. Run `dotnet run` and record the expected output in comments.

8. Build the project with `dotnet build` — there must be 0 errors and 0 warnings. Run `dotnet run` and make sure the output matches what is expected.

#### Requirements
- Target platform: .NET 8, language C# 12. The project must build with `dotnet build` without errors or warnings, including `Nullable` enabled — add `<Nullable>enable</Nullable>` to the `.csproj`.
- Every modifier must be used intentionally and accompanied by a RU+EN comment justifying the choice: why exactly `const`, `static readonly`, `readonly`, or configuration. Comments must refer to the nature of the value, not to personal preference.
- Forbidden: `const` for values that may change between releases (URLs, timeouts, limits, keys); `const` for instance fields; assigning a `readonly` field outside a constructor; mutable `static` fields without synchronization in concurrent scenarios; instantiating a static class.
- The service URL, timeout, and rate limit must come from `appsettings.json` via `IConfiguration` (an `IOptions`-like pattern is enough for a console app). A default value may live in code as a fallback.
- The code must be thread-safe with respect to the mutable `static` field `_callCount`: increment via `Interlocked.Increment`. If you add a cache, protect it with `lock` or use `ConcurrentDictionary`.
- Type and method names come from the assignment; signatures must not change. Test coverage is optional but a separate `*.Tests` project with a couple of checks for `HaversineDistance` and `MaxWaypointsPerRoute` is welcome.
- Place one class per file (course style convention).

#### Pitfalls
- **`const` is inlined into the caller.** If `EarthRadiusKm` is `const` and you change the value in a `GeoCalculator.Lib` library, a console app that is not recompiled keeps using the old value. Therefore `const` is only for values whose change would require recompiling the entire solution and that is acceptable. For values that may be refined (say, a more accurate Earth radius in a new model), `static readonly` is safer.
- **`const` is implicitly static.** You cannot write `public const int X = 5;` expecting a per-instance value — it is a field of the type. If a value must be tied to an instance and stay immutable after creation, use `readonly` (`public readonly int X;` assigned in the constructor).
- **`readonly` is assigned only at declaration or in a constructor.** Trying to write `SessionId = "x";` inside a method yields CS0191. There may be multiple constructors — assignment is allowed in each, but only once along any execution path.
- **`static readonly` is evaluated at type initialization.** The field `BuildDate` captures `DateTime.UtcNow` at the moment the type is first touched, not "at compile time". This is both the source of its flexibility (the value is fresh) and a hidden dependency (type initialization may throw and poison the whole type — handle it carefully).
- **Thread safety of static fields.** `_callCount++` is not atomic (read-modify-write); under load increments are lost. Use `Interlocked.Increment(ref _callCount)` or a `lock`. Initialization of `static readonly` fields is itself thread-safe (the runtime guarantees it through the type initializer), but mutable fields must be protected manually.
- **`const` in attributes.** Attribute parameter values must be compile-time constants — here `const` is appropriate and `static readonly` will not work (the compiler rejects it). For attribute parameters that are enums or literals, `const` is the right choice.
- **`readonly struct`.** Since C# 7 you can mark a whole struct as `readonly struct`, which guarantees immutability and lets the compiler optimize `in` parameters. For small geometric structures (for example `readonly struct GeoPoint`) this is a good technique — mention it in the bonus.
- **Configuration vs `const`.** URLs, timeouts, limits, keys are configuration. Baking them as `const` means forcing a recompile on every environment change. Read them at runtime and inject through DI/IOptions.

#### Acceptance criteria
- [ ] The `GeoCalculator` project builds with `dotnet build` without errors or warnings on .NET 8 / C# 12.
- [ ] `MathHelper` is a static class; an attempt to `new MathHelper()` triggers compiler error CS0723 (demonstrated in commented code).
- [ ] `MathHelper` contains `const` fields `EarthRadiusKm`, `MaxWaypointsPerRoute`, `DefaultLocale` with RU+EN justification comments.
- [ ] `MathHelper` contains `static readonly` fields `BuildDate` and `ProcessorCount` with an explanation of why they cannot be `const`.
- [ ] `HaversineDistance` uses `EarthRadiusKm` and returns the correct distance (checked on a pair of points).
- [ ] The mutable `static` field `_callCount` is incremented via `Interlocked.Increment` and exposed through `CallCount`.
- [ ] `RouteSession` has `readonly` fields `SessionId` and `_startedAt`, assigned only in the constructor.
- [ ] `RouteSession.AddWaypoint` checks `MathHelper.MaxWaypointsPerRoute` and throws `InvalidOperationException` when exceeded.
- [ ] The code contains a commented attempt to assign a `readonly` field outside a constructor, marked with CS0191.
- [ ] `RoutingApiClient` receives URL, timeout, and rate limit from `appsettings.json` via `IConfiguration`; all fields are `private readonly`.
- [ ] `appsettings.json` has a `Routing` section with `BaseUrl`, `TimeoutSeconds`, `MaxRequestsPerMinute`.
- [ ] `Program.cs` demonstrates calling `HaversineDistance`, creating two sessions, growing `CallCount`, and printing `BuildDate`/`ProcessorCount`.
- [ ] `dotnet run` prints the expected values matching the comments in the code.
- [ ] Every modifier has a RU+EN justification comment.
- [ ] There is no `const` for any environment-dependent value (URLs, timeouts, limits).

#### Hints (no direct answer)
- Ask yourself: "Will this value change without recompiling the entire solution?" If yes, it is not `const`.
- For `const`, make sure the right-hand side is a literal or a constant expression; `Math.PI` is `public const double`, so `const double Pi = Math.PI;` is allowed in C# 12, but `DateTime.UtcNow` is not.
- A thread-safe integer increment is `Interlocked.Increment`; the `long` overload has the same signature.
- A `readonly` field can be assigned in any constructor, but only once per execution path; multiple constructors calling each other with `: this(...)` is normal.
- Reading `appsettings.json` in a console app: add the `Microsoft.Extensions.Configuration.Json` package and use `new ConfigurationBuilder().AddJsonFile("appsettings.json").Build()`.
- Copy `appsettings.json` to the output directory via `<None Update="appsettings.json"><CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory></None>` in the `.csproj`.

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8 — reference solution
using System;
using System.Threading;

// Static class: cannot be instantiated, cannot be inherited.
public static class MathHelper
{
    // const — true constant, baked into the caller as a literal.
    // Earth radius does not change between releases — const is appropriate.
    public const double EarthRadiusKm = 6371.0;

    // const for a countable limit and a default locale.
    public const int MaxWaypointsPerRoute = 25;
    public const string DefaultLocale = "en-US";

    // static readonly — computed once at type initialization (runtime).
    // Cannot be const: DateTime.UtcNow and Environment.ProcessorCount are runtime values.
    public static readonly DateTime BuildDate = DateTime.UtcNow;
    public static readonly int ProcessorCount = Environment.ProcessorCount;

    // Mutable static field — protected with Interlocked.
    private static long _callCount;
    public static long CallCount => _callCount;

    public static double HaversineDistance(double lat1, double lon1, double lat2, double lon2)
    {
        Interlocked.Increment(ref _callCount); // thread-safe increment

        double ToRadians(double deg) => deg * Math.PI / 180.0;
        double dLat = ToRadians(lat2 - lat1);
        double dLon = ToRadians(lon2 - lon1);
        double a = Math.Sin(dLat / 2) * Math.Sin(dLat / 2) +
                   Math.Cos(ToRadians(lat1)) * Math.Cos(ToRadians(lat2)) *
                   Math.Sin(dLon / 2) * Math.Sin(dLon / 2);
        double c = 2 * Math.Atan2(Math.Sqrt(a), Math.Sqrt(1 - a));
        return EarthRadiusKm * c; // const EarthRadiusKm inlined as a literal
    }
}

// Instance class with readonly fields.
public sealed class RouteSession
{
    public readonly string SessionId;       // assigned once in the constructor
    private readonly DateTime _startedAt;   // immutable after construction
    private int _waypoints;                 // mutable instance state

    public RouteSession(string sessionId)
    {
        SessionId = sessionId;   // OK: readonly assignment in the constructor
        _startedAt = DateTime.UtcNow;
        _waypoints = 0;
    }

    // public void BreakIt() { SessionId = "x"; } // CS0191: a readonly field cannot be assigned outside a constructor

    public DateTime StartedAt => _startedAt;
    public int Waypoints => _waypoints;

    public void AddWaypoint(double lat, double lon)
    {
        if (_waypoints >= MathHelper.MaxWaypointsPerRoute) // const used as a literal
            throw new InvalidOperationException(
                $"Waypoint limit exceeded ({MathHelper.MaxWaypointsPerMinute}).");
        _waypoints++;
    }
}

// All external parameters come from configuration, not from const.
public sealed class RoutingApiClient
{
    private readonly HttpClient _http;
    private readonly Uri _endpoint;          // readonly — set in the constructor from configuration
    private readonly TimeSpan _timeout;
    private readonly int _maxRequestsPerMinute;

    public RoutingApiClient(HttpClient http, string baseUrl, int timeoutSeconds, int maxRequestsPerMinute)
    {
        _http = http;
        _endpoint = new Uri(baseUrl);                          // runtime value
        _timeout = TimeSpan.FromSeconds(timeoutSeconds);
        _maxRequestsPerMinute = maxRequestsPerMinute;
        _http.Timeout = _timeout;
    }

    public Uri Endpoint => _endpoint;
    public int MaxRequestsPerMinute => _maxRequestsPerMinute;
}

class Program
{
    static void Main()
    {
        // const and static readonly accessed via the type, no new.
        Console.WriteLine($"EarthRadiusKm = {MathHelper.EarthRadiusKm}");   // 6371
        Console.WriteLine($"MaxWaypoints  = {MathHelper.MaxWaypointsPerRoute}"); // 25
        Console.WriteLine($"BuildDate     = {MathHelper.BuildDate}");       // startup moment
        Console.WriteLine($"ProcessorCount= {MathHelper.ProcessorCount}");  // core count

        double moscowToSpb = MathHelper.HaversineDistance(55.7558, 37.6173, 59.9343, 30.3351);
        Console.WriteLine($"MSQ→SPB       = {moscowToSpb:F1} km");           // ~635 km
        Console.WriteLine($"CallCount     = {MathHelper.CallCount}");       // 1

        var s1 = new RouteSession("order-42");
        var s2 = new RouteSession("order-43");
        s1.AddWaypoint(55.75, 37.61);
        s1.AddWaypoint(55.80, 37.70);
        s2.AddWaypoint(59.93, 30.34);
        Console.WriteLine($"{s1.SessionId}: {s1.Waypoints} wp");            // order-42: 2 wp
        Console.WriteLine($"{s2.SessionId}: {s2.Waypoints} wp");            // order-43: 1 wp

        // var mh = new MathHelper(); // CS0723: cannot create an instance of a static class
    }
}
```

Walk-through, line by line. `MathHelper` is declared `static` — this guarantees that no one can instantiate it or inherit from it (CS0723 on `new`). `EarthRadiusKm`, `MaxWaypointsPerRoute`, and `DefaultLocale` are `const`: their values are known at compile time and do not change between releases, so inlining a literal into the caller is safe. Inside `HaversineDistance` you can see the key consequence: `EarthRadiusKm * c` compiles as if you had written `6371.0 * c` — there is no field reference in the IL. `BuildDate` and `ProcessorCount` are `static readonly` because `DateTime.UtcNow` and `Environment.ProcessorCount` are evaluated only at runtime — `const` is impossible here, the compiler would reject it with CS0133. The field `_callCount` is mutable and static; the `++` operator is not atomic, so `Interlocked.Increment` is used to close the data race under concurrency. In `RouteSession` the fields `SessionId` and `_startedAt` are `readonly`: they are assigned exactly once in the constructor and cannot be overwritten in a method (CS0191 is demonstrated in the commented `BreakIt`). The mutable `_waypoints` stays a plain instance field because it must grow on every `AddWaypoint` call. The limit check uses `MathHelper.MaxWaypointsPerRoute` — a `const` inlined into the condition as a literal, which is acceptable because the limit is a true domain constant. In `RoutingApiClient` all external parameters (`baseUrl`, `timeoutSeconds`, `maxRequestsPerMinute`) arrive as constructor parameters from configuration and are stored in `private readonly` fields: a `const` would be a mistake here, since URLs and timeouts change between environments without a recompile. The bottom line: every modifier is chosen by the nature of the value, not by habit, which is exactly the goal of the lesson.

#### Going deeper (bonus)
1. Move `MathHelper` and `RouteSession` into a separate class library `GeoCalculator.Lib` (`dotnet new classlib`). Reference it from the console app. Then change the value of `EarthRadiusKm` in the library, rebuild only the library, and demonstrate that the console app — without recompilation — keeps using the old value: a live demonstration of `const` inlining. After that, switch the field to `static readonly` and show that the new value is picked up.
2. Implement a `readonly struct GeoPoint { public readonly double Lat; public readonly double Lon; ... }` and use it as an `in GeoPoint` parameter to `HaversineDistance`. Compare with a `class` version: what does `readonly struct` give you in terms of mutation safety and `in`-parameter performance?
3. Add a lazy thread-safe cache of computed distances using `ConcurrentDictionary<(GeoPoint, GeoPoint), double>` or `Lazy<T>`. Show that `static readonly` initialization of the cache is thread-safe, while the mutable contents are protected by the data structure itself.
4. Migrate configuration reading to `IOptions<RoutingOptions>` through `Microsoft.Extensions.DependencyInjection` and `Microsoft.Extensions.Options.ConfigurationExtensions`. Show that `RoutingOptions` is a `sealed class` with `readonly` properties or `init` setters, and explain why `const` would be inappropriate even for a "default value".

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Проект собирается без ошибок и предупреждений под .NET 8 / C# 12.
- [ ] (RU) Каждый модификатор (`const`, `static readonly`, `readonly`, конфигурация) снабжён комментарием-обоснованием на двух языках.
- [ ] (RU) `MathHelper` — статический класс; попытка `new` демонстрирует CS0723.
- [ ] (RU) `HaversineDistance` использует `const` `EarthRadiusKm`, инкремент `_callCount` через `Interlocked`.
- [ ] (RU) `RouteSession` имеет `readonly` поля, присваиваемые только в конструкторе; есть закомментированный пример CS0191.
- [ ] (RU) `RoutingApiClient` получает URL/таймаут/лимит из `appsettings.json`; ни одного `const` для окружения.
- [ ] (RU) `dotnet run` выводит ожидаемые значения, зафиксированные в комментариях.
- [ ] (EN) The project builds without errors or warnings on .NET 8 / C# 12.
- [ ] (EN) Every modifier (`const`, `static readonly`, `readonly`, configuration) has a RU+EN justification comment.
- [ ] (EN) `MathHelper` is a static class; a `new` attempt demonstrates CS0723.
- [ ] (EN) `HaversineDistance` uses the `const` `EarthRadiusKm`; `_callCount` is incremented via `Interlocked`.
- [ ] (EN) `RouteSession` has `readonly` fields assigned only in the constructor; a commented CS0191 example is present.
- [ ] (EN) `RoutingApiClient` reads URL/timeout/limit from `appsettings.json`; no `const` for environment values.
- [ ] (EN) `dotnet run` prints the expected values recorded in comments.

#### Ресурсы / Resources
- [Microsoft Learn — static (C# Reference)](https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/static)
- [Microsoft Learn — const (C# Reference)](https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/const)
- [Microsoft Learn — readonly (C# Reference)](https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/readonly)
- [Microsoft Learn — Static Classes and Static Class Members](https://learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/static-classes-and-static-class-members)
- [Microsoft Learn — Configuration in .NET](https://learn.microsoft.com/dotnet/core/extensions/configuration)
- [Microsoft Learn — readonly struct (C# Reference)](https://learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/struct#readonly-struct)
- [Microsoft Learn — Interlocked Class](https://learn.microsoft.com/dotnet/api/system.threading.interlocked)
- [Microsoft Learn — Options pattern in .NET](https://learn.microsoft.com/dotnet/core/extensions/options)
