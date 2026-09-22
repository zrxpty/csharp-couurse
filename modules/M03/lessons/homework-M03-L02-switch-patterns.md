---
[← К уроку M03-L02](lesson-M03-L02-switch-patterns.md) | [⬆ К модулю M03](../README.md) | [Следующее ДЗ →](homework-M03-L03-loops.md)
---

### Домашнее задание M03-L02: switch, switch expressions, pattern matching / Homework M03-L02: switch, switch expressions, pattern matching

**Урок / Lesson:** M03-L02
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться осознанно выбирать между классическим оператором `switch` и `switch expression`, применять константные паттерны, паттерны типа, свойства и кортежей, фильтры `when`, `or`-паттерны и discard `_`, гарантировать полноту (exhaustiveness) и избегать типичных ошибок порядка веток и непокрытых значений. (EN) Learn to consciously choose between the classic `switch` statement and the `switch` expression, apply constant, type, property and tuple patterns, `when` filters, `or` patterns and the discard `_`, guarantee exhaustiveness, and avoid typical mistakes of arm ordering and uncovered values.

#### Связь с уроком / Connection to the lesson
(RU — 2–3 предложения) ДЗ закрепляет все ключевые темы урока M03-L02: классический `switch` с побочными эффектами и ранними `return`, `switch expression` как декларативное отображение входа в значение, четыре вида паттернов (константный, типа, свойств, discard), `when`-фильтры и `or`-паттерны, а также кортежные `switch`. Особое внимание уделяется полноте (exhaustiveness) и порядку веток от специфичного к общему — главным источникам багов из раздела «Частые ошибки».
(EN) The homework reinforces every key topic of lesson M03-L02: the classic `switch` with side effects and early `return`, the `switch` expression as a declarative mapping of input to value, the four pattern kinds (constant, type, property, discard), `when` filters and `or` patterns, and tuple `switch`. Special attention goes to exhaustiveness and arm ordering from specific to general — the main sources of bugs from the “Common Mistakes” section.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Вы присоединяетесь к команде сервиса логистики «FastShip», который рассчитывает стоимость доставки и маршрутизирует посылки. В кодовой базе исторически сложился «суп из `if`»: каскады вложенных условий определяют тариф по типу посылки, стране назначения, статусу клиента VIP, срочности и весу. Код читается тяжело, новые правила вносятся болезненно, а баги с непокрытыми комбинациями всплывают в продакшене (например, тяжёлая срочная посылка в новую страну уезжала в ветку «по умолчанию» и тариф не считался).

Команда принимает решение: переписать ключевую логику тарификации и маршрутизации с использованием современных средств C# 12 / .NET 8 — `switch expression` для отображения «вход → значение», классического `switch` там, где нужны побочные эффекты (логирование, мутация состояния), и паттернов для декомпозиции доменных объектов. Это позволит сделать правила явными, заставить компилятор следить за полнотой через exhaustiveness-проверку, и избавиться от «забытых веток».

Вам выдан набор типов предметной области: записи (`record`) для посылки, клиента и заказа, перечисление (`enum`) для типа посылки и срочности. Ваша задача — реализовать четыре функции: расчёт базового тарифа (switch expression), расчёт итоговой стоимости с учётом VIP и страны (property + tuple patterns), маршрутизацию посылки по отделениям (классический switch с побочными эффектами — логирование), и форматирование человекочитаемого описания (tuple switch expression с `when`-фильтрами). Каждая функция должна быть рабочей, компилироваться без предупреждений о непокрытых ветках и проходить набор тестовых вызовов из `Program.cs`.

Эта задача моделирует реальную инженерную работу: вы не просто пишете «Hello World» с switch, а превращаете запутанную условную логику в декларативные правила, за полноту которых отвечает компилятор. Именно так `switch expression` и паттерны применяются в продакшен-коде .NET 8.

#### Что нужно сделать (пошагово)
1. Создайте новый консольный проект .NET 8. Откройте терминал в папке модуля и выполните команду `dotnet new console -n FastShip -o FastShip --framework net8.0`. Перейдите в папку проекта `cd FastShip` и убедитесь, что в `FastShip.csproj` указано `<TargetFramework>net8.0</TargetFramework>` и `<LangVersion>latest</LangVersion>` (или по умолчанию C# 12 для net8.0). Откройте `Program.cs` — это будет точка входа с top-level statements.

2. Определите доменные типы в отдельном файле `Domain.cs` (или в верхней части `Program.cs`, если предпочитаете один файл). Создайте перечисление `enum PackageKind { Letter, SmallParcel, LargeParcel, Frequent }` и `enum Urgency { Standard, Express, SameDay }`. Создайте записи: `record Client(string Name, bool IsVip, string Country)`, `record Package(PackageKind Kind, decimal WeightKg, Urgency Urgency, Client Recipient)`, `record Order(decimal Amount, string Country, bool IsVip)`.

3. Реализуйте функцию `decimal BaseTariff(Package p)` с помощью **switch expression**. Правила тарифа: `Letter` стоит 50, `SmallParcel` — 120, `LargeParcel` — 300, `Frequent` (постоянные отправления) — 80. Используйте паттерн свойств, чтобы сразу извлечь `Kind`, и обязательно добавьте discard `_` для полноты (или перечислите все значения enum явно — тогда компилятор сообщит о новом члене при добавлении). Подумайте: какой подход безопаснее для закрытого множества enum, и обоснуйте выбор в комментарии.

4. Реализуйте функцию `decimal FinalPrice(Package p)` через **switch expression с кортежным и property-паттернами**. Логика: извлеките кортеж `(p.Recipient.Country, p.Recipient.IsVip, p.Urgency, p.Kind)` и примените правила: для страны `"DE"` и `IsVip == true` — скидка 0% (бесплатная доставка для VIP в Германии); для `"DE"` — базовый тариф + 19% (НДС); для `"US"` или `"CA"` — базовый тариф без налога (`or`-паттерн); для `Urgency.Express` с любым получателем — базовый тариф × 1.5; для `Urgency.SameDay` — базовый тариф × 2.2; для `IsVip == true` в любой другой стране — базовый тариф × 0.9; всё остальное — базовый тариф. Расположите ветки от наиболее специфичной к наименее специфичной, discard `_` последним.

5. Реализуйте функцию `void Route(Package p)` через **классический switch** с побочными эффектами. В зависимости от `(p.Kind, p.Urgency)` выведите в `Console` строку маршрутизации и выполните «побочный эффект» — например, увеличьте счётчик `int routedCount` (объявите его в классе или используйте `ref`/поле). Правила: `Letter` + `Standard` → «Письмо → обычное отделение / Letter → regular desk»; `LargeParcel` + `SameDay` → «Крупный груз → срочная линия / Large → express line»; и так далее для всех комбинаций. Здесь уместен именно классический `switch`, потому что в каждой ветке несколько операторов (логирование + мутация). Используйте `case ... :` с `break` и не допускайте неявного fall-through.

6. Реализуйте функцию `string Describe(Package p)` через **tuple switch expression с `when`-фильтрами**. Возвращайте человекочитаемое описание, например: вес 0 и `Letter` → «Пустое письмо / Empty letter»; `WeightKg > 20` и `LargeParcel` → «Тяжёлый крупный груз / Heavy large parcel»; диагональные комбинации `(Express, SameDay)` — невозможны (это разные значения urgency), но покажите, как `when` отличает «тяжёлый экспресс» от «лёгкого экспресса» по весу.

7. В `Program.cs` (top-level statements) вызовите все четыре функции на наборе тестовых посылок: письмо в Германию VIP, мелкая посылка Express в США, крупный груз SameDay в Канаду, частое отправление в Россию не-VIP, и посылка с новой страной (например, `"FR"`), которая проверит discard-ветку. Выведите результаты через `Console.WriteLine`. Ожидаемый вывод должен содержать вычисленные тарифы и строки маршрутизации.

8. Соберите и запустите: `dotnet build` (убедитесь, что нет предупреждений о непокрытых ветках switch expression — это важно, компилятор C# 12 выдаёт warning CS8509 если switch expression не покрывает все возможные входы и нет `_`), затем `dotnet run`. Зафиксируйте фактический вывод.

9. Осознанно проверьте порядок веток в `FinalPrice`: если вы поставите общую VIP-ветку `IsVip == true` раньше страны `"DE"`, то немецкий VIP уедет в общую скидку 10% вместо бесплатной доставки — это и есть ошибка «общий паттерн выше специфичного» из урока. Попробуйте намеренно поменять порядок, соберите, прогоните тесты и убедитесь, что результат меняется. Верните правильный порядок.

10. Добавьте в перечисление `PackageKind` новое значение `Fragile` и пересоберите проект. Если вы перечисляли все значения явно в `BaseTariff`, компилятор должен сообщить о непокрытой ветке (CS8509) — это и есть ценность exhaustiveness. Если использовали `_`, баг тихо уйдёт в discard. Обратите внимание на разницу и зафиксируйте вывод в комментариях.

#### Требования к решению
- Проект на .NET 8 с C# 12 (top-level statements в `Program.cs`, `LangVersion` не ниже 12). Используйте `record` для доменных типов, как в примере урока.
- Четыре функции реализованы каждой своим инструментом: `BaseTariff` — switch expression с property/constant pattern; `FinalPrice` — switch expression с tuple и property patterns + `or`; `Route` — классический `switch` с `case`/`break` и побочными эффектами; `Describe` — tuple switch expression с `when`-фильтрами. Смешивать стили внутри одной функции нельзя — это требование урока «не смешивайте стили без необходимости».
- Exhaustiveness обеспечен в каждой switch expression: либо полный перечень значений enum/типов, либо discard `_` с осознанным комментарием, почему он допустим. Для `FinalPrice` discard обязателен, потому что кортеж из строк/bool/enum невозможно перечислить полностью.
- Порядок веток: специфичные паттерны выше общих, `_`/`default` последним. Особенно в `FinalPrice`: `"DE"` + VIP выше общего VIP; `Urgency.Express` выше базовой ставки.
- `when`-фильтры применены только там, где паттерн формы не выражает условие (например, вес больше порога) — не滥用 `when` вместо паттерна свойства `WeightKg: > 20`.
- Код компилируется без предупреждений CS8509 (непокрытая ветка) и CS8524 (switch expression необработан). Предупреждения других видов тоже желательно устранить.
- В комментариях (RU+EN) кратко пояснен выбор инструмента в каждой функции и обоснован порядок веток. Это учит осознанному применению, а не копированию синтаксиса.

#### Тонкости и подводные камни
- **Порядок case имеет значение с C# 7.** Проверка идёт сверху вниз, первый совпавший паттерн выигрывает. Если в `FinalPrice` общая ветка `{ IsVip: true }` стоит выше `{ Country: "DE", IsVip: true }`, немецкий VIP получит скидку 10% вместо бесплатной доставки. Сортируйте от частного к общему — это прямое следствие правила из урока «более специфичные паттерны выше».
- **Discard `_` скрывает забытые ветки.** В `BaseTariff` если вы используете `_` вместо перечисления всех `PackageKind`, добавление `Fragile` тихо уйдёт в discard и тариф посчитается неправильно (по умолчанию). Для закрытых множеств (enum) предпочитайте явный перечень — тогда компилятор через CS8509 сообщит о новом члене. Это ключевая ценность exhaustiveness-проверки, упомянутая в Best Practices урока.
- **`or`-паттерн объединяет константы, но не произвольные выражения.** `case "US" or "CA"` работает, а `case x or y` где `x`/`y` — переменные — нет (нужны константы). Используйте `or` для группировки одинаковой логики, как `D or E` в примере урока, чтобы не дублировать руки.
- **`when`-фильтр vs property pattern.** `case { WeightKg: > 20 }:` предпочтительнее `case Package p when p.WeightKg > 20`, потому что первый — декларативный паттерн, второй — императивное условие. Урок прямо говорит: «`when` применяйте только там, где паттерн не выражает условие». Например, сравнение с внешним состоянием или вычисляемое условие — там `when` оправдан.
- **Неявный fall-through запрещён в C# 8+.** В классическом `switch` каждая ветка должна заканчиваться `break`, `return` или `goto`. Нельзя «провалиться» в следующую `case` без явного `goto case`. Это убирает целый класс багов C-стиля. Несколько меток на одну секцию (`case 'D': case 'E':`) допустимы — это не fall-through, а общая секция.
- **Switch expression возвращает значение, оператор — нет.** Если вам нужно несколько операторов или побочный эффект — классический switch. Если результат — отображение входа в значение — switch expression. Смешивать в одной функции не нужно: это запутывает читателя.
- **Property pattern с вложенными проверками** `case { Country: "DE", IsVip: true }` компактнее каскада `if`, но не злоупотребляйте глубиной: три уровня вложенности уже тяжело читать. Выносите сложные проверки в метод или `when`.
- **`null`-паттерн** — отдельная константа. В `FinalPrice` вход `Package` теоретически не null (это record), но если бы был `object`, ветка `null => ...` должна идти первой, иначе `NullReferenceException` при доступе к свойствам в property pattern.

#### Критерии приёмки
- [ ] Проект `FastShip` собирается командой `dotnet build` без ошибок и без предупреждений CS8509/CS8524.
- [ ] `Program.cs` использует top-level statements C# 12; `TargetFramework` = `net8.0`.
- [ ] Доменные типы — `record` (`Client`, `Package`, `Order`) и `enum` (`PackageKind`, `Urgency`).
- [ ] `BaseTariff` реализован через switch expression с property pattern и полным перечислением `PackageKind` (не `_`) — обосновано в комментарии.
- [ ] `FinalPrice` реализован через switch expression с tuple + property patterns и `or`; порядок веток от специфичного к общему; discard `_` последним.
- [ ] В `FinalPrice` ветка `"DE"` + VIP находится выше общей VIP-ветки (иначе баг со скидкой).
- [ ] `Route` реализован через классический `switch` с `case`/`break`, имеет побочный эффект (логирование + мутация счётчика), не использует switch expression.
- [ ] `Describe` реализован через tuple switch expression с минимум двумя `when`-фильтрами (например, по весу).
- [ ] Тестовые вызовы в `Program.cs` покрывают: VIP-письмо в DE, Express-мелкая посылка в US, SameDay-крупный груз в CA, Frequent в не-VIP стране, посылка в новую страну (FR) для проверки discard.
- [ ] Добавление `Fragile` в `PackageKind` вызывает CS8509 в `BaseTariff` (если перечислены явно) — это зафиксировано в комментарии.
- [ ] Намеренная перестановка веток в `FinalPrice` (общая VIP выше DE+VIP) меняет результат теста — это продемонстрировано и описано.
- [ ] В комментариях RU+EN кратко обоснован выбор инструмента в каждой из четырёх функций.
- [ ] `dotnet run` выводит вычисленные тарифы и строки маршрутизации без исключений.
- [ ] Нет смешения стилей внутри одной функции (switch expression + классический switch в одном теле).

#### Подсказки (без прямого ответа)
- Для `BaseTariff` начните с `p switch { { Kind: PackageKind.Letter } => 50m, ... }` — это property pattern. Подумайте, можно ли упростить до `p.Kind switch { PackageKind.Letter => 50m, ... }` — и какой вариант даёт лучшую exhaustiveness-проверку.
- В `FinalPrice` соберите кортеж `(p.Recipient.Country, p.Recipient.IsVip, p.Urgency)` и применяйте паттерны к кортежу. Вспомните пример `Tax(Order o)` из урока — там property pattern на самом объекте; здесь удобнее кортеж, потому что правила затрагивают несколько полей из разных вложенных объектов.
- Для `Route` вспомните классический `GradeToLabelClassic` из урока: `switch (grade) { case 'A': ...; return ...; }`. Только здесь вместо `return` — `Console.WriteLine` + инкремент счётчика + `break`.
- `when`-фильтр в `Describe`: `(var k, var w) when w > 20 => "тяжёлый"`. Но сначала попробуйте property pattern `WeightKg: > 20` — возможно, `when` здесь избыточен. Урок учит предпочитать паттерн.
- При добавлении `Fragile` соберите проект и прочитайте диагностическое сообщение компилятора — номер warning и текст подскажут, где именно не покрыта ветка.

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 — top-level statements, switch expression, classic switch, patterns
// Эталонное решение ДЗ M03-L02: тарификация и маршрутизация посылок.
// Reference solution for HW M03-L02: package pricing and routing.

using System;
using System.Collections.Generic;

// --- Доменные типы / Domain types ---
enum PackageKind { Letter, SmallParcel, LargeParcel, Frequent }
enum Urgency { Standard, Express, SameDay }
record Client(string Name, bool IsVip, string Country);
record Package(PackageKind Kind, decimal WeightKg, Urgency Urgency, Client Recipient);
record Order(decimal Amount, string Country, bool IsVip);

// --- 1. BaseTariff: switch expression + property pattern, явный перечень enum ---
// Явный перечень безопаснее _: компилятор сообщит о новом члене enum (CS8509).
// Explicit list is safer than _: the compiler reports a new enum member (CS8509).
decimal BaseTariff(Package p) => p.Kind switch
{
    PackageKind.Letter      => 50m,
    PackageKind.SmallParcel => 120m,
    PackageKind.LargeParcel => 300m,
    PackageKind.Frequent    => 80m,
    // _ => 0m, // НЕ используем discard — хотим exhaustiveness-проверку / no discard — want exhaustiveness check
};

// --- 2. FinalPrice: tuple + property patterns + or, порядок от частного к общему ---
// Порядок критичен: "DE"+VIP выше общего VIP; Express/SameDay выше базовой ставки.
// Order matters: "DE"+VIP above general VIP; Express/SameDay above base rate.
decimal FinalPrice(Package p)
{
    var base_ = BaseTariff(p); // базовый тариф / base tariff
    return (p.Recipient.Country, p.Recipient.IsVip, p.Urgency) switch
    {
        ("DE", true,  _)        => 0m,                       // бесплатная доставка VIP в DE / free for VIP in DE
        ("DE", _,     _)        => base_ * 1.19m,            // НДС 19% / VAT 19%
        ("US", _, _) or ("CA", _, _) => base_,              // or-паттерн, без налога / or-pattern, no tax
        (_, _, Urgency.Express) => base_ * 1.5m,            // экспресс-надбавка / express surcharge
        (_, _, Urgency.SameDay) => base_ * 2.2m,            // доставка в тот же день / same-day
        (_, true, _)            => base_ * 0.9m,            // общая VIP-скидка 10% / general VIP 10% — ПОСЛЕ DE
        _                       => base_,                   // discard по умолчанию / discard default
    };
}

// --- 3. Route: классический switch с побочными эффектами (логирование + мутация) ---
// Здесь switch expression не подходит: несколько операторов и side effects.
// Switch expression does not fit: multiple statements and side effects.
int routedCount = 0; // счётчик маршрутизованных посылок / counter

void Route(Package p)
{
    switch ((p.Kind, p.Urgency)) // классический switch по кортежу / classic switch on tuple
    {
        case (PackageKind.Letter, Urgency.Standard):
            Console.WriteLine("Письмо → обычное отделение / Letter → regular desk");
            routedCount++;
            break; // обязательно — иначе fall-through запрещён / mandatory — no fall-through allowed
        case (PackageKind.LargeParcel, Urgency.SameDay):
            Console.WriteLine("Крупный груз → срочная линия / Large → express line");
            routedCount++;
            break;
        case (PackageKind.LargeParcel, Urgency.Express):
            Console.WriteLine("Крупный груз → экспресс-линия / Large → express line");
            routedCount++;
            break;
        case (PackageKind.SmallParcel, Urgency.SameDay):
            Console.WriteLine("Мелкая посылка → курьер / Small → courier");
            routedCount++;
            break;
        default: // всё остальное / everything else
            Console.WriteLine($"По умолчанию → склад / Default → warehouse ({p.Kind})");
            routedCount++;
            break;
    }
}

// --- 4. Describe: tuple switch expression с when-фильтрами ---
// when применён там, где паттерн формы не выражает условие (вес — это значение, не форма).
// when used where a shape pattern cannot express the condition (weight is a value, not a shape).
string Describe(Package p) => (p.Kind, p.WeightKg, p.Urgency) switch
{
    (PackageKind.Letter, 0m, _)            => "Пустое письмо / Empty letter",
    (PackageKind.LargeParcel, > 20m, _)    => "Тяжёлый крупный груз / Heavy large parcel", // property/relational pattern
    (PackageKind.SmallParcel, _, Urgency.SameDay) when p.WeightKg < 2 => "Лёгкая срочная мелкая / Light urgent small",
    (PackageKind.SmallParcel, _, Urgency.SameDay) => "Срочная мелкая / Urgent small",
    (var k, var w, Urgency.Express) when w > 10 => "Тяжёлый экспресс / Heavy express",
    _                                       => "Обычная посылка / Ordinary package",
};

// --- Демонстрационный запуск / Demo run ---
var clients = new[]
{
    new Client("Иван / Ivan", true,  "DE"),
    new Client("Mary",        false, "US"),
    new Client("Pierre",      false, "CA"),
    new Client("Анна / Anna", false, "RU"),
    new Client("Luc",         false, "FR"),
};
var packages = new[]
{
    new Package(PackageKind.Letter,      0.1m, Urgency.Standard, clients[0]),
    new Package(PackageKind.SmallParcel, 1.5m, Urgency.Express,  clients[1]),
    new Package(PackageKind.LargeParcel, 25m,  Urgency.SameDay,  clients[2]),
    new Package(PackageKind.Frequent,    2m,   Urgency.Standard, clients[3]),
    new Package(PackageKind.SmallParcel, 3m,   Urgency.Standard, clients[4]),
};
foreach (var pkg in packages)
{
    Console.WriteLine($"{Describe(pkg)} | tariff={BaseTariff(pkg)} | final={FinalPrice(pkg):F2}");
    Route(pkg);
}
Console.WriteLine($"Всего маршрутизовано / Total routed: {routedCount}");
```

**Разбор по строкам (почему так).** `BaseTariff` использует `p.Kind switch { ... }` без discard — это осознанный выбор: для закрытого множества `enum PackageKind` явный перечень даёт exhaustiveness-проверку. Если в шаге 10 вы добавите `Fragile`, компилятор выдаст CS8509 «Switch expression не покрывает все возможные входы» именно потому, что discard отсутствует. Это ценность, которую урок подчёркивает в Best Practices: «предпочитай явный перечень для закрытого множества (enum)».

`FinalPrice` применяет **кортежный switch expression** — это прямой аналог `Tax(Order o)` и `Describe(int x, int y)` из урока. Кортеж `(Country, IsVip, Urgency)` удобнее, чем property pattern на `Package`, потому что правила затрагивают поля из вложенного `Recipient`. Порядок веток критичен: `("DE", true, _)` стоит выше `(_, true, _)`, иначе немецкий VIP уехал бы в общую скидку 10% вместо бесплатной доставки. `or`-паттерн `("US", _, _) or ("CA", _, _)` объединяет две страны с одинаковой логикой — это рекомендация урока «не повторяй логику, используй `or`». Discard `_` последним закрывает полноту, потому что кортеж из строк/bool/enum перечислить полностью невозможно.

`Route` — **классический switch** с `case`/`break` и побочными эффектами (`Console.WriteLine` + `routedCount++`). Это та ситуация, где switch expression не подходит: ветки содержат несколько операторов и мутируют состояние. Урок прямо говорит: «классический `switch` остаётся уместным, когда ветки содержат сложные побочные эффекты, несколько операторов или ранние `return`». Обязательный `break` в каждой ветке предотвращает неявный fall-through — частую ошибку из урока.

`Describe` сочетает **property/relational pattern** `(PackageKind.LargeParcel, > 20m, _)` и **`when`-фильтр** `when p.WeightKg < 2`. Relational pattern `> 20m` — это паттерн (декларативный), и урок советует предпочитать его `when`. `when` оставлен только там, где условие действительно не выражается формой — например, лёгкая срочная мелкая посылка, где нужно отличить «лёгкую» от «тяжёлой» в сочетании с конкретным Kind. Это иллюстрирует правило «`when` применяйте только там, где паттерн не выражает условие».

Тестовые вызовы покрывают все ключевые ветки: VIP-письмо в DE (бесплатно), Express-мелкая в US (без налога, ×1.5), SameDay-крупный в CA (×2.2), Frequent в России (базовый тариф), посылка в FR (discard-ветка в FinalPrice). Это гарантирует, что ни одна ветка не осталась непроверенной.

#### Задания на углубление (бонус)
1. **Список паттернов (list patterns).** C# 11+ поддерживает `[`, `..`, `]` для массивов. Реализуйте функцию `string SummarizeRoute(Package[] batch)`, которая через list pattern определяет: пустой массив → «Пустая партия / Empty batch», `[PackageKind.LargeParcel, ..]` → «Начинается с крупного / Starts with large», `[.., PackageKind.Letter]` → «Заканчивается письмом / Ends with letter». Это расширяет тему паттернов за рамки урока.
2. **Рекурсивные паттерны свойств.** Расширьте `FinalPrice` так, чтобы учитывать `Recipient.Country` через вложенный property pattern на самом `Package`: `case { Recipient: { Country: "DE", IsVip: true } } => 0m`. Сравните читаемость с кортежным подходом и обоснуйте выбор.
3. **Обобщённый switch по `object`.** Реализуйте `decimal PriceAny(object item)` как в примере урока (`Price(object item)`), поддержав `Book`, `ElectronProduct` и ваш `Package` через type pattern + `when`. Продемонстрируйте, что discard с `throw` ловит неизвестные типы.
4. **Тесты на порядок веток.** Напишите unit-тест (через `dotnet test` + xUnit), который проверяет, что немецкий VIP получает 0, а не 10%-скидку, и что перестановка веток ломает тест. Это закрепляет понимание, что порядок — это контракт.

---

## Statement in English / Постановка на английском

#### Context & motivation
You are joining the team of the logistics service “FastShip”, which computes delivery prices and routes parcels. The codebase historically grew into an “`if` soup”: cascades of nested conditions determine the tariff by parcel kind, destination country, customer VIP status, urgency and weight. The code is hard to read, new rules are painful to add, and bugs with uncovered combinations surface in production (for example, a heavy urgent parcel to a new country silently fell into the “default” branch and the tariff was never computed).

The team decides to rewrite the core pricing and routing logic with modern C# 12 / .NET 8 features — `switch expression` for the “input → value” mapping, the classic `switch` where side effects are needed (logging, state mutation), and patterns to decompose domain objects. This makes the rules explicit, lets the compiler guard completeness through exhaustiveness checking, and removes the whole class of “forgot a branch” bugs.

You are given a set of domain types: records (`record`) for a parcel, a client and an order, and enumerations (`enum`) for parcel kind and urgency. Your task is to implement four functions: base tariff computation (switch expression), final price computation accounting for VIP and country (property + tuple patterns), parcel routing across desks (classic switch with side effects — logging), and human-readable description formatting (tuple switch expression with `when` filters). Each function must be working, compile without warnings about uncovered arms, and pass the set of test calls from `Program.cs`.

This task models real engineering work: you are not writing a “Hello World” switch, but turning tangled conditional logic into declarative rules whose completeness is enforced by the compiler. That is exactly how `switch expression` and patterns are used in production .NET 8 code.

#### What to do step by step
1. Create a new .NET 8 console project. Open a terminal in the module folder and run `dotnet new console -n FastShip -o FastShip --framework net8.0`. Move into the project folder `cd FastShip` and verify that `FastShip.csproj` contains `<TargetFramework>net8.0</TargetFramework>` and `<LangVersion>latest</LangVersion>` (or the C# 12 default for net8.0). Open `Program.cs` — it will be the top-level-statements entry point.

2. Define the domain types in a separate file `Domain.cs` (or at the top of `Program.cs` if you prefer a single file). Create the enumerations `enum PackageKind { Letter, SmallParcel, LargeParcel, Frequent }` and `enum Urgency { Standard, Express, SameDay }`. Create the records: `record Client(string Name, bool IsVip, string Country)`, `record Package(PackageKind Kind, decimal WeightKg, Urgency Urgency, Client Recipient)`, `record Order(decimal Amount, string Country, bool IsVip)`.

3. Implement the function `decimal BaseTariff(Package p)` using a **switch expression**. Tariff rules: `Letter` costs 50, `SmallParcel` — 120, `LargeParcel` — 300, `Frequent` (recurring shipments) — 80. Use a property pattern to extract `Kind` right away, and either add a discard `_` for completeness or list every enum value explicitly — then decide which approach is safer for a closed enum set and justify it in a comment.

4. Implement the function `decimal FinalPrice(Package p)` with a **switch expression using tuple and property patterns**. Logic: extract the tuple `(p.Recipient.Country, p.Recipient.IsVip, p.Urgency, p.Kind)` and apply rules: for country `"DE"` and `IsVip == true` — 0% (free delivery for VIP in Germany); for `"DE"` — base tariff + 19% (VAT); for `"US"` or `"CA"` — base tariff with no tax (`or` pattern); for `Urgency.Express` with any recipient — base tariff × 1.5; for `Urgency.SameDay` — base tariff × 2.2; for `IsVip == true` in any other country — base tariff × 0.9; everything else — base tariff. Order arms from most specific to least specific, discard `_` last.

5. Implement the function `void Route(Package p)` with a **classic switch** that has side effects. Depending on `(p.Kind, p.Urgency)`, write a routing line to `Console` and perform a “side effect” — for example, increment a counter `int routedCount` (declare it in a class or use a field). Rules: `Letter` + `Standard` → “Letter → regular desk”; `LargeParcel` + `SameDay` → “Large → express line”; and so on for all combinations. The classic `switch` is appropriate here because each arm has multiple statements (logging + mutation). Use `case ... :` with `break` and do not allow implicit fall-through.

6. Implement the function `string Describe(Package p)` with a **tuple switch expression using `when` filters**. Return a human-readable description, for example: weight 0 and `Letter` → “Empty letter”; `WeightKg > 20` and `LargeParcel` → “Heavy large parcel”; show how `when` distinguishes a “heavy express” parcel from a “light express” parcel by weight.

7. In `Program.cs` (top-level statements), call all four functions on a set of test parcels: a VIP letter to Germany, a small Express parcel to the US, a large SameDay parcel to Canada, a Frequent shipment to a non-VIP country, and a parcel to a new country (for example `"FR"`) that exercises the discard arm. Print the results with `Console.WriteLine`. The expected output must contain computed tariffs and routing strings.

8. Build and run: `dotnet build` (make sure there are no warnings about uncovered switch-expression arms — this matters; the C# 12 compiler emits warning CS8509 when a switch expression does not cover all possible inputs and has no `_`), then `dotnet run`. Capture the actual output.

9. Consciously check the arm order in `FinalPrice`: if you place the general VIP arm `IsVip == true` above the `"DE"` arm, the German VIP will fall into the general 10% discount instead of free delivery — this is exactly the “general pattern above specific” mistake from the lesson. Intentionally swap the order, rebuild, rerun the tests and confirm the result changes. Restore the correct order.

10. Add a new value `Fragile` to the `PackageKind` enum and rebuild. If you listed every value explicitly in `BaseTariff`, the compiler must report an uncovered arm (CS8509) — that is the value of exhaustiveness. If you used `_`, the bug silently goes into the discard. Note the difference and record the observation in comments.

#### Requirements
- A .NET 8 project with C# 12 (top-level statements in `Program.cs`, `LangVersion` not lower than 12). Use `record` for the domain types, as in the lesson example.
- Four functions implemented each with its own tool: `BaseTariff` — switch expression with a property/constant pattern; `FinalPrice` — switch expression with tuple and property patterns + `or`; `Route` — classic `switch` with `case`/`break` and side effects; `Describe` — tuple switch expression with `when` filters. Mixing styles inside one function is forbidden — this is the lesson requirement “do not mix styles without reason”.
- Exhaustiveness is guaranteed in every switch expression: either a full list of enum values/types, or a discard `_` with a conscious comment explaining why it is acceptable. For `FinalPrice` the discard is mandatory, because a tuple of strings/bool/enum cannot be fully enumerated.
- Arm order: specific patterns above general ones, `_`/`default` last. Especially in `FinalPrice`: `"DE"` + VIP above the general VIP; `Urgency.Express` above the base rate.
- `when` filters are used only where a shape pattern cannot express the condition (for example, weight above a threshold) — do not abuse `when` instead of a property pattern `WeightKg: > 20`.
- The code compiles without warnings CS8509 (uncovered arm) and CS8524 (unhandled switch expression). Other warnings should preferably be resolved too.
- Comments (RU+EN) briefly explain the tool choice in each function and justify the arm order. This teaches conscious application, not syntax copying.

#### Pitfalls
- **Arm order matters since C# 7.** Matching runs top-to-bottom and the first matching pattern wins. If in `FinalPrice` the general arm `{ IsVip: true }` is placed above `{ Country: "DE", IsVip: true }`, the German VIP gets a 10% discount instead of free delivery. Sort from specific to general — a direct consequence of the lesson rule “more specific patterns higher”.
- **The discard `_` hides forgotten arms.** In `BaseTariff`, if you use `_` instead of listing every `PackageKind`, adding `Fragile` silently falls into the discard and the tariff is computed wrong (by default). For closed sets (enums) prefer an explicit list — then the compiler reports the new member via CS8509. This is the core value of exhaustiveness checking mentioned in the lesson Best Practices.
- **The `or` pattern combines constants, not arbitrary expressions.** `case "US" or "CA"` works, but `case x or y` where `x`/`y` are variables does not (constants are required). Use `or` to group identical logic, like `D or E` in the lesson example, instead of duplicating arms.
- **`when` filter vs property pattern.** `case { WeightKg: > 20 }:` is preferable to `case Package p when p.WeightKg > 20`, because the first is a declarative pattern and the second is an imperative condition. The lesson says directly: “use `when` only where a pattern cannot express the condition”. For example, comparison with external state or a computed condition is where `when` is justified.
- **Implicit fall-through is forbidden in C# 8+.** In a classic `switch` every arm must end with `break`, `return` or `goto`. You cannot “fall through” to the next `case` without an explicit `goto case`. This removes a whole class of C-style bugs. Multiple labels on one section (`case 'D': case 'E':`) are allowed — that is not fall-through, that is a shared section.
- **A switch expression returns a value, a statement does not.** If you need multiple statements or a side effect — classic switch. If the result is a mapping of input to value — switch expression. Do not mix them in one function: it confuses the reader.
- **A property pattern with nested checks** `case { Country: "DE", IsVip: true }` is more compact than an `if` cascade, but do not overdo depth: three levels of nesting are already hard to read. Extract complex checks into a method or a `when`.
- **The `null` pattern** is a separate constant. In `FinalPrice` the input `Package` is theoretically non-null (it is a record), but if it were `object`, the `null => ...` arm must come first, otherwise a `NullReferenceException` occurs when properties are accessed in a property pattern.

#### Acceptance criteria
- [ ] The `FastShip` project builds with `dotnet build` without errors and without CS8509/CS8524 warnings.
- [ ] `Program.cs` uses C# 12 top-level statements; `TargetFramework` = `net8.0`.
- [ ] Domain types are `record` (`Client`, `Package`, `Order`) and `enum` (`PackageKind`, `Urgency`).
- [ ] `BaseTariff` is implemented with a switch expression and a property pattern, with a full list of `PackageKind` (not `_`) — justified in a comment.
- [ ] `FinalPrice` is implemented with a switch expression using tuple + property patterns and `or`; arm order from specific to general; discard `_` last.
- [ ] In `FinalPrice` the `"DE"` + VIP arm is above the general VIP arm (otherwise the discount bug).
- [ ] `Route` is implemented with a classic `switch` with `case`/`break`, has a side effect (logging + counter mutation), and does not use a switch expression.
- [ ] `Describe` is implemented with a tuple switch expression and at least two `when` filters (for example, by weight).
- [ ] Test calls in `Program.cs` cover: a VIP letter to DE, a small Express parcel to US, a large SameDay parcel to CA, a Frequent shipment to a non-VIP country, a parcel to a new country (FR) to exercise the discard.
- [ ] Adding `Fragile` to `PackageKind` raises CS8509 in `BaseTariff` (if listed explicitly) — recorded in a comment.
- [ ] An intentional swap of arms in `FinalPrice` (general VIP above DE + VIP) changes the test result — demonstrated and described.
- [ ] Comments RU+EN briefly justify the tool choice in each of the four functions.
- [ ] `dotnet run` prints computed tariffs and routing strings without exceptions.
- [ ] No style mixing inside one function (switch expression + classic switch in one body).

#### Hints (no direct answer)
- For `BaseTariff`, start with `p switch { { Kind: PackageKind.Letter } => 50m, ... }` — a property pattern. Consider whether you can simplify to `p.Kind switch { PackageKind.Letter => 50m, ... }` — and which variant gives a better exhaustiveness check.
- In `FinalPrice`, build the tuple `(p.Recipient.Country, p.Recipient.IsVip, p.Urgency)` and apply patterns to the tuple. Recall the `Tax(Order o)` example from the lesson — there a property pattern is applied to the object itself; here a tuple is more convenient, because the rules touch several fields from different nested objects.
- For `Route`, recall the classic `GradeToLabelClassic` from the lesson: `switch (grade) { case 'A': ...; return ...; }`. Only here, instead of `return`, you have `Console.WriteLine` + counter increment + `break`.
- The `when` filter in `Describe`: `(var k, var w) when w > 20 => "heavy"`. But first try the property pattern `WeightKg: > 20` — `when` may be redundant here. The lesson teaches to prefer the pattern.
- When you add `Fragile`, build the project and read the compiler diagnostic — the warning number and text will tell you exactly where the arm is uncovered.

#### Reference solution walk-through (EN)
```csharp
// C# 12 / .NET 8 — top-level statements, switch expression, classic switch, patterns
// Reference solution for HW M03-L02: package pricing and routing.

using System;
using System.Collections.Generic;

// --- Domain types ---
enum PackageKind { Letter, SmallParcel, LargeParcel, Frequent }
enum Urgency { Standard, Express, SameDay }
record Client(string Name, bool IsVip, string Country);
record Package(PackageKind Kind, decimal WeightKg, Urgency Urgency, Client Recipient);
record Order(decimal Amount, string Country, bool IsVip);

// --- 1. BaseTariff: switch expression + property pattern, explicit enum list ---
// An explicit list is safer than _: the compiler reports a new enum member (CS8509).
decimal BaseTariff(Package p) => p.Kind switch
{
    PackageKind.Letter      => 50m,
    PackageKind.SmallParcel => 120m,
    PackageKind.LargeParcel => 300m,
    PackageKind.Frequent    => 80m,
    // _ => 0m, // no discard — we want the exhaustiveness check
};

// --- 2. FinalPrice: tuple + property patterns + or, order from specific to general ---
// Order matters: "DE"+VIP above general VIP; Express/SameDay above the base rate.
decimal FinalPrice(Package p)
{
    var base_ = BaseTariff(p);
    return (p.Recipient.Country, p.Recipient.IsVip, p.Urgency) switch
    {
        ("DE", true,  _)        => 0m,                       // free for VIP in DE
        ("DE", _,     _)        => base_ * 1.19m,            // VAT 19%
        ("US", _, _) or ("CA", _, _) => base_,              // or-pattern, no tax
        (_, _, Urgency.Express) => base_ * 1.5m,            // express surcharge
        (_, _, Urgency.SameDay) => base_ * 2.2m,            // same-day
        (_, true, _)            => base_ * 0.9m,            // general VIP 10% — AFTER DE
        _                       => base_,                   // discard default
    };
}

// --- 3. Route: classic switch with side effects (logging + mutation) ---
// A switch expression does not fit: multiple statements and side effects.
int routedCount = 0;

void Route(Package p)
{
    switch ((p.Kind, p.Urgency)) // classic switch on a tuple
    {
        case (PackageKind.Letter, Urgency.Standard):
            Console.WriteLine("Letter → regular desk");
            routedCount++;
            break; // mandatory — no fall-through allowed
        case (PackageKind.LargeParcel, Urgency.SameDay):
            Console.WriteLine("Large → express line");
            routedCount++;
            break;
        case (PackageKind.LargeParcel, Urgency.Express):
            Console.WriteLine("Large → express line");
            routedCount++;
            break;
        case (PackageKind.SmallParcel, Urgency.SameDay):
            Console.WriteLine("Small → courier");
            routedCount++;
            break;
        default: // everything else
            Console.WriteLine($"Default → warehouse ({p.Kind})");
            routedCount++;
            break;
    }
}

// --- 4. Describe: tuple switch expression with when filters ---
// when is used where a shape pattern cannot express the condition (weight is a value, not a shape).
string Describe(Package p) => (p.Kind, p.WeightKg, p.Urgency) switch
{
    (PackageKind.Letter, 0m, _)            => "Empty letter",
    (PackageKind.LargeParcel, > 20m, _)    => "Heavy large parcel", // property/relational pattern
    (PackageKind.SmallParcel, _, Urgency.SameDay) when p.WeightKg < 2 => "Light urgent small",
    (PackageKind.SmallParcel, _, Urgency.SameDay) => "Urgent small",
    (var k, var w, Urgency.Express) when w > 10 => "Heavy express",
    _                                       => "Ordinary package",
};

// --- Demo run ---
var clients = new[]
{
    new Client("Ivan",  true,  "DE"),
    new Client("Mary",  false, "US"),
    new Client("Pierre",false, "CA"),
    new Client("Anna",  false, "RU"),
    new Client("Luc",   false, "FR"),
};
var packages = new[]
{
    new Package(PackageKind.Letter,      0.1m, Urgency.Standard, clients[0]),
    new Package(PackageKind.SmallParcel, 1.5m, Urgency.Express,  clients[1]),
    new Package(PackageKind.LargeParcel, 25m,  Urgency.SameDay,  clients[2]),
    new Package(PackageKind.Frequent,    2m,   Urgency.Standard, clients[3]),
    new Package(PackageKind.SmallParcel, 3m,   Urgency.Standard, clients[4]),
};
foreach (var pkg in packages)
{
    Console.WriteLine($"{Describe(pkg)} | tariff={BaseTariff(pkg)} | final={FinalPrice(pkg):F2}");
    Route(pkg);
}
Console.WriteLine($"Total routed: {routedCount}");
```

**Line-by-line walk-through (why).** `BaseTariff` uses `p.Kind switch { ... }` with no discard — a conscious choice: for the closed `enum PackageKind` set, an explicit list gives exhaustiveness checking. If in step 10 you add `Fragile`, the compiler emits CS8509 “Switch expression does not cover all possible inputs” precisely because the discard is absent. That is the value the lesson stresses in Best Practices: “prefer an explicit list for a closed set (enum)”.

`FinalPrice` applies a **tuple switch expression** — a direct analogue of `Tax(Order o)` and `Describe(int x, int y)` from the lesson. The tuple `(Country, IsVip, Urgency)` is more convenient than a property pattern on `Package`, because the rules touch fields of the nested `Recipient`. Arm order is critical: `("DE", true, _)` is above `(_, true, _)`, otherwise the German VIP would fall into the general 10% discount instead of free delivery. The `or` pattern `("US", _, _) or ("CA", _, _)` combines two countries with identical logic — that is the lesson recommendation “do not repeat logic, use `or`”. The discard `_` last closes completeness, because a tuple of strings/bool/enum cannot be fully enumerated.

`Route` is a **classic switch** with `case`/`break` and side effects (`Console.WriteLine` + `routedCount++`). This is the situation where a switch expression does not fit: arms contain multiple statements and mutate state. The lesson says directly: “the classic `switch` remains appropriate when branches contain complex side effects, multiple statements, or early `return`s”. The mandatory `break` in every arm prevents implicit fall-through — a common mistake from the lesson.

`Describe` combines a **property/relational pattern** `(PackageKind.LargeParcel, > 20m, _)` and a **`when` filter** `when p.WeightKg < 2`. The relational pattern `> 20m` is a pattern (declarative), and the lesson advises preferring it to `when`. `when` is kept only where the condition is genuinely not expressible by shape — for example, a light urgent small parcel, where you must distinguish “light” from “heavy” in combination with a specific Kind. This illustrates the rule “use `when` only where a pattern cannot express the condition”.

The test calls cover every key arm: a VIP letter to DE (free), a small Express parcel to the US (no tax, ×1.5), a large SameDay parcel to CA (×2.2), a Frequent shipment to Russia (base tariff), a parcel to FR (the discard arm in FinalPrice). This guarantees that no arm is left untested.

#### Going deeper (bonus)
1. **List patterns.** C# 11+ supports `[`, `..`, `]` for arrays. Implement `string SummarizeRoute(Package[] batch)` that uses a list pattern: an empty array → “Empty batch”, `[PackageKind.LargeParcel, ..]` → “Starts with large”, `[.., PackageKind.Letter]` → “Ends with letter”. This extends the topic of patterns beyond the lesson.
2. **Recursive property patterns.** Extend `FinalPrice` to account for `Recipient.Country` through a nested property pattern on `Package` itself: `case { Recipient: { Country: "DE", IsVip: true } } => 0m`. Compare readability with the tuple approach and justify the choice.
3. **A generic switch over `object`.** Implement `decimal PriceAny(object item)` like the lesson example `Price(object item)`, supporting `Book`, `ElectronProduct` and your `Package` through a type pattern + `when`. Demonstrate that a discard with `throw` catches unknown types.
4. **Order-of-arms tests.** Write a unit test (via `dotnet test` + xUnit) that verifies the German VIP gets 0, not a 10% discount, and that swapping the arms breaks the test. This cements the understanding that order is a contract.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Проект FastShip собирается без ошибок и предупреждений CS8509/CS8524.
- [ ] (RU) Реализованы четыре функции каждым своим инструментом (switch expression / classic switch / patterns / when).
- [ ] (RU) Порядок веток от специфичного к общему, `_`/`default` последним.
- [ ] (RU) Добавление `Fragile` в enum демонстрирует exhaustiveness-проверку.
- [ ] (RU) Намеренная перестановка веток в `FinalPrice` меняет результат теста.
- [ ] (RU) `dotnet run` выводит тарифы и маршрутизацию без исключений.
- [ ] (EN) The FastShip project builds without errors and CS8509/CS8524 warnings.
- [ ] (EN) Four functions are implemented each with its own tool (switch expression / classic switch / patterns / when).
- [ ] (EN) Arm order from specific to general, `_`/`default` last.
- [ ] (EN) Adding `Fragile` to the enum demonstrates exhaustiveness checking.
- [ ] (EN) An intentional arm swap in `FinalPrice` changes the test result.
- [ ] (EN) `dotnet run` prints tariffs and routing without exceptions.

#### Ресурсы / Resources
- [Microsoft Learn — Selection statements (if, switch) / Операторы выбора (if, switch)](https://learn.microsoft.com/dotnet/csharp/language-reference/statements/selection-statements)
- [Microsoft Learn — Pattern matching / Сопоставление шаблонов](https://learn.microsoft.com/dotnet/csharp/fundamentals/functional/pattern-matching)
- [Microsoft Learn — Switch expressions (C# reference)](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/switch-expression)
- [Microsoft Learn — Patterns (C# reference)](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/patterns)
