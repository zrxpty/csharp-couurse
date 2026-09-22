[⬆ К модулю M15](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M15-L07: TDD, red-green-refactor / TDD, red-green-refactor

**Модуль / Module:** M15
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

TDD (Test-Driven Development, разработка через тестирование) — это не про «тесты как таковые», а про способ думать о дизайне API до того, как он написан. Главный механизм — короткий цикл **red-green-refactor**, повторяемый десятки раз в день.

**Red.** Сначала пишется один failing-тест на то поведение, которое нужно добавить. Тест компилируется, но падает — потому что кода ещё нет или он не делает нужное. Красный бар здесь — это сигнал «я понял, чего хочу», а не авария. Важно: красным должен быть именно Assert, а не ошибка компиляции на пустом классе (иначе шаг растягивается и теряется ритм).

**Green.** Пишется минимально возможный код, чтобы тест стал зелёным. Минимально — буквально: можно вернуть константу, продублировать ветку, написать «грязное» решение. Цикл длится минуты, а не часы. Главная цель зелёного — корректность «здесь и сейчас», а не красота.

**Refactor.** Тесты зелёные и защищают поведение — теперь можно безопасно наводить порядок: вынести дублирование, переименовать, разбить метод, удалить мёртвый код, упростить условие. На этом шаге нового поведения не появляется; если захотелось нового — открываешь новый цикл. Зелёный цвет тестов должен сохраняться всё время рефакторинга.

**Test-first как дисциплина.** Порядок «сначала тест» заставляет думать о вызове кода с точки зрения клиента. Если тест писать трудно — значит, у API проблемы с зависимостями, непонятно как создавать объект, либо метод делает слишком многое. Это ранний сигнал дизайн-запаха, который на этапе «код сначала» ловится с опозданием.

**Эволюция дизайна через тесты.** TDD поощряет маленькие шаги и постоянное «выдавливание» интерфейса из реального использования, а не из прогнозов. Появление второго-третьего теста часто вскрывает дублирование, которое убирается рефакторингом — так рождается обобщение (параметризация, полиморфизм, выделение стратегии). Дизайн растёт органически и остаётся проверяемым.

**Когда TDD уместен.** Хорошо работает там, где поведение чётко специфицируемо: бизнес-логика, алгоритмы, парсеры, валидаторы, денежные расчёты, доменные правила. Любое место с детерминированными входами/выходами и ясными правилами — идеальный кандидат. TDD особенно силён при багфиксе: сначала тест, воспроизводящий баг (red), затем фикс (green).

**Вне TDD.** Не стоит насильно применять TDD там, где внешних ограничений слишком много или результат неизвестен: исследования и прототипы (spikes), разовый UI-эксперимент, работа с нестабильными внешними API, где тесты всё равно хрупки, перф-эксперименты, инфраструктурный клей (Dockerfiles, CI-скрипты). Там, где ты не знаешь, что хочешь получить, тест-first лишь фиксирует неуверенность. Спайки пишутся без тестов и выбрасываются — либо после них переписывается «по-настоящему» уже через TDD.

**Аналогия.** TDD — как строительство с лесов: ты ставишь леса (тест) перед тем, как класть кирпич (код), и леса остаются, поддерживая стену, пока меняется её отделка (refactor). Без лесов каждый косметический ремонт рискует обрушить стену.

**Практический ритм.** Цикл короче 5–10 минут. Если дольше — шаг слишком большой, разбей. Пара «red → green» за пару минут — индикатор здоровья.

#### Theory (EN)

TDD (Test-Driven Development) is not really about tests — it is a way to design an API by thinking about how it will be called before writing it. The engine is a short **red-green-refactor** loop repeated dozens of times a day.

**Red.** Write one failing test for the behavior you want to add. The test compiles but fails because the production code does not exist yet or does not do the right thing. A red bar here is a statement of intent — “I know what I want” — not an accident. Note: the failure should be an assertion failure, not a compile error on a missing class; otherwise the step drags on and the rhythm breaks.

**Green.** Write the minimum code that turns the test green. Minimum means literal: a constant return, a duplicated branch, an ugly conditional are all acceptable. The cycle is measured in minutes, not hours. The goal of green is correctness right now, not beauty.

**Refactor.** With tests green and behavior locked, clean up safely: remove duplication, rename, split a method, delete dead code, simplify a condition. No new behavior appears on this step — if you want more, start a new cycle. Tests must stay green throughout.

**Test-first as discipline.** Writing the test first forces you to design the call from the consumer’s perspective. When a test is hard to write, the API usually has problems: awkward dependencies, unclear construction, or a method doing too much. These are design smells that “code-first” tends to discover much later.

**Design evolving through tests.** TDD rewards small steps and lets interfaces emerge from real usage rather than forecasts. The second or third test often exposes duplication, which the refactor step removes — and generalization (parameterization, polymorphism, a strategy) is born. Design grows organically and stays testable by construction.

**When TDD fits.** Anywhere behavior is clearly specifiable: business logic, algorithms, parsers, validators, money math, domain rules. Anything with deterministic inputs/outputs and crisp rules is a prime candidate. TDD also shines for bug fixes: write a test reproducing the bug (red), then fix (green).

**Outside TDD.** Do not force TDD where constraints are too external or the outcome is unknown: research and spikes, one-off UI experiments, thin glue around volatile third-party APIs where tests become brittle anyway, performance experiments, infrastructure like Dockerfiles or CI scripts. Where you do not yet know what you want, test-first just freezes uncertainty. Spikes are written without tests and thrown away — or, after a spike, the feature is rewritten “properly” with TDD.

**Analogy.** TDD is like building with scaffolding: you erect the scaffolding (test) before laying the brick (code), and the scaffolding stays, supporting the wall while its finish changes (refactor). Without scaffolding, every cosmetic repair risks collapsing the wall.

**Practical rhythm.** A cycle should be shorter than 5–10 minutes. If it is longer, the step is too big — split it. A red-to-green pair in a couple of minutes is a healthy sign.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — эволюция класса корзины через red-green-refactor
// C# 12 / .NET 8 — shopping cart evolved via red-green-refactor

using System.Collections.Generic;
using System.Linq;

namespace Shop.Domain;

// === RED 1: тест Basket.Total возвращает 0 для пустой корзины ===
// === RED 1: test expects Basket.Total == 0 for an empty basket ===
public sealed class Basket
{
    private readonly List<BasketLine> _lines = new();

    public decimal Total => _lines.Sum(l => l.Subtotal); // 0 для пустой — green 1
                                                         // 0 for empty — green 1

    public void Add(string sku, int qty, decimal unitPrice)
    {
        // RED 2: тест ждёт, что добавление учитывает цену и количество
        // RED 2: test expects Add to account for price and quantity
        var line = _lines.FirstOrDefault(l => l.Sku == sku);
        if (line is null)
        {
            line = new BasketLine(sku, unitPrice);
            _lines.Add(line);
        }
        line.AddQuantity(qty); // green 2: появляется поведение, тест зелёный
                               // green 2: behavior appears, test passes
    }
}

public sealed class BasketLine
{
    public string Sku { get; }
    public int Quantity { get; private set; }
    public decimal UnitPrice { get; }
    public decimal Subtotal => Quantity * UnitPrice;

    public BasketLine(string sku, decimal unitPrice)
    {
        Sku = sku;
        UnitPrice = unitPrice;
    }

    public void AddQuantity(int qty) => Quantity += qty;
}

// === REFACTOR: после 3-го теста (скидка) выделаем стратегию расчета ===
// === REFACTOR: after the 3rd test (discount) extract a pricing strategy ===
public interface IPricing
{
    decimal LineTotal(int qty, decimal unitPrice);
}

public sealed class StandardPricing : IPricing
{
    public decimal LineTotal(int qty, decimal unitPrice) => qty * unitPrice;
}

public sealed class BulkDiscountPricing : IPricing
{
    private readonly decimal _threshold;
    private readonly decimal _discountRate;

    public BulkDiscountPricing(decimal threshold, decimal discountRate)
    {
        _threshold = threshold;
        _discountRate = discountRate;
    }

    public decimal LineTotal(int qty, decimal unitPrice)
    {
        // Тест: при qty >= threshold применяется скидка
        // Test: when qty >= threshold a discount applies
        var baseTotal = qty * unitPrice;
        return qty >= _threshold
            ? baseTotal * (1m - _discountRate)
            : baseTotal;
    }
}
```

```csharp
// xUnit — тесты, которые двигали код выше
// xUnit — tests that drove the code above
using Shop.Domain;
using Xunit;

namespace Shop.Tests;

public class BasketTests
{
    [Fact]
    public void Total_is_zero_for_empty_basket() // RED 1
    {
        var basket = new Basket();
        Assert.Equal(0m, basket.Total);
    }

    [Fact]
    public void Add_increments_total_by_price_times_quantity() // RED 2
    {
        var basket = new Basket();
        basket.Add("A1", 2, 10m);
        Assert.Equal(20m, basket.Total);
    }

    [Fact]
    public void Same_sku_merges_into_one_line() // ещё один красный → green
    {
        var basket = new Basket();
        basket.Add("A1", 1, 10m);
        basket.Add("A1", 1, 10m);
        Assert.Equal(20m, basket.Total);
    }
}
```

#### Best Practices

- Держи цикл red-green-refactor короче 5–10 минут; большие шаги — признак того, что надо декомпозировать.
- Пиши по одному тесту за раз: один red, минимальный green, затем refactor.
- Сначала намеренно «грязный» green — красоту наводи только на этапе refactor при зелёных тестах.
- Не добавляй новое поведение в refactor; для нового поведения открывай новый цикл.
- Изолируйся от внешних зависимостей (БД, время, сеть) через интерфейсы и fakes — иначе цикл становится хрупким и медленным.
- Багфиксы начинай с воспроизводящего теста (red), затем фикс (green), без исключений.

- Keep the red-green-refactor loop under 5–10 minutes; longer steps mean you should decompose.
- Write one test at a time: one red, minimal green, then refactor.
- Accept an intentionally ugly green first; improve beauty only during refactor with tests green.
- Never add behavior during refactor; for new behavior open a new cycle.
- Isolate external dependencies (DB, time, network) behind interfaces and fakes — otherwise the loop becomes brittle and slow.
- Always start bug fixes with a reproducing test (red), then fix (green), no exceptions.

#### Частые ошибки / Common Mistakes

- Несколько падающих тестов одновременно → пиши по одному тесту за шаг, иначе зелёный ambiguous.
- Шаг «дольше часа» без refactor → дроби поведение на меньшие инкременты до красного/зелёного.
- Рефакторинг с непрошедшими тестами → никогда не рефактори при красном; сначала верни зелёный.
- Тесты зависят от реальной БД/сети → выдели интерфейс и используй fake/in-memory, держи цикл быстрым.
- Зеленый через возвращение константы без обобщения → после первого зелёного сразу обобщай на следующем red.
- Игнорирование дизайн-запаха «трудно писать тест» → трудный тест — сигнал пересмотреть зависимости и границы.

- Multiple failing tests at once → write one test per step, otherwise green is ambiguous.
- Hour-long step without a refactor → break behavior into smaller increments leading to red/green.
- Refactoring while tests are red → never refactor on red; restore green first.
- Tests depend on real DB/network → introduce an interface and use a fake/in-memory to keep the loop fast.
- Green via a hard-coded constant that never generalizes → generalize right after the first green on the next red.
- Ignoring the “hard to test” smell → a hard test is a signal to rethink dependencies and boundaries.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] У меня есть ровно один новый падающий тест на текущий шаг.
- [ ] Я пишу минимальный код, чтобы сделать тест зелёным, без лишнего поведения.
- [ ] После зелёного я провожу refactor только при проходящих тестах.
- [ ] Мой цикл укладывается в 5–10 минут; при превышении я дроблю шаг.
- [ ] Я выделил внешние зависимости за интерфейсами и не обращаюсь к БД/сети в тестах.
- [ ] Для багфикса я сначала написал воспроизводящий тест, затем фикс.
- [ ] Я сознаю, когда нахожусь «вне TDD» (спайк, прототип) и не выдаю это за постоянную практику.

- [ ] I have exactly one new failing test for the current step.
- [ ] I write minimal code to make the test green, with no extra behavior.
- [ ] After green, I refactor only while tests pass.
- [ ] My loop fits within 5–10 minutes; when it exceeds, I split the step.
- [ ] I have isolated external dependencies behind interfaces and do not hit DB/network in tests.
- [ ] For a bug fix, I wrote a reproducing test first, then the fix.
- [ ] I am aware when I am “outside TDD” (spike, prototype) and do not pretend it is the steady practice.

#### Ресурсы / Resources

- [Microsoft Learn — Тестирование приложений ASP.NET Core MVC — https://learn.microsoft.com/dotnet/architecture/modern-web-apps-azure/test-asp-net-core-mvc-apps](https://learn.microsoft.com/dotnet/architecture/modern-web-apps-azure/test-asp-net-core-mvc-apps)

---

[⬆ К модулю M15](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
