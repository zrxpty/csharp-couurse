# Дорожная карта развития курса / Course Roadmap

> Эволюция курса «Полный курс по C#» от минимально жизнеспособного MVP (v1.0) до долгосрочного vision (v3.0).
> Evolution of the «Complete C# Course» from a minimum viable MVP (v1.0) to long-term vision (v3.0).
>
> Связанные артефакты / Related artifacts: `01-course-passport.md`, `02-tracks.md`, `03-modules-table.md`.

---

## Принципы дорожной карты / Roadmap Principles

1. **Сначала ядро, потом глубина** — каждая версия добавляет устойчивую ценность, не разрушая предыдущую.
2. **Треки как вехи** — v1.0 = Foundation готов; v2.0 = + Professional; v3.0 = + Enterprise + новые форматы.
3. **Каждая версия имеет метрики успеха и риски** — без них roadmap превращается в пожелания.
4. **Обратная совместимость по ID** — номера модулей M0X не меняются после публикации; контент эволюционирует внутри карточки.

---

# Версия v1.0 — «MVP: Foundation» / MVP: Foundation

> Цель: запустить минимально жизнеспособный курс — полностью готовый Foundation-трек, покрывающий ядро языка и подготовку к Professional.

## Что готово при запуске / What is ready at launch

### Модули (все Core, уровень Foundation) / Modules

| ID | Модуль | Статус при v1.0 |
|---|---|---|
| M01 | Intro | ✅ Core, готов |
| M02 | Types & Operators | ✅ Core, готов |
| M03 | Control Flow & Methods | ✅ Core, готов |
| M04 | OOP Basics | ✅ Core, готов |
| M05 | OOP Advanced | ✅ Core, готов |
| M06 | Collections & Generics | ✅ Core, готов |
| M07 | Exception Handling | ✅ Core, готов |
| M08 | LINQ | ✅ Core, готов |
| M09 | Async Programming | ✅ Core, готов |
| M10 | I/O & Serialization | ✅ Core, готов (Foundation capstone) |

### Треки / Tracks
- ✅ **Foundation** — полностью готов (M01–M10, ~120 ч).

### Домены, покрытые в v1.0 / Domains covered
Core (язык/ООП), Data (LINQ, I/O), Concurrency (async) — базовый слой.

### Форматы / Formats
- Теория: текст + код-примеры.
- Практика: лабораторные с автопроверкой (базовые unit-тесты как чек-контракты).
- Проекты: Foundation capstone (M10) — консольное CRUD с JSON.

## Цели v1.0 / Goals
1. Дать выпускнику возможность пройти Foundation-трек от нуля до самостоятельных мини-проектов.
2. Зафиксировать ID-структуру M01–M20, метки Core/Optional/Pro/Deprecated и граф зависимостей.
3. Получить обратную связь от первых студентов для приоритизации v2.0.

## Метрики успеха / Success Metrics
- **Completion rate Foundation-трека** ≥ 30% (из начавших доходят до M10).
- **Capstone M10 сдан** у ≥ 50% прошедших M01–M09.
- **NPS первого потока** ≥ 30.
- **Время до первого работающего кода** (M01, Hello World) ≤ 30 минут.
- **Покрытие тестами автопроверки** для всех 10 модулей = 100%.

## Риски v1.0 / Risks
| Риск | Вероятность | Влияние | Митигация |
|---|---|---|---|
| Перегруз новичка в M09 (async — сложная тема в Foundation) | Высокая | Высокое | Дать M09 в «двухслойном» формате: базовое использование + Optional «deep dive» с пометкой Pro-вектор. |
| M05 (полиморфизм) вызывает провал в M06 (дженерики) | Средняя | Среднее | Чёткие prerequisites и диагностический тест перед M06. |
| Недостаток автопроверки лабораторных | Средняя | Высокое | Использовать unit-тесты как чек-контракты; в v2.0 — platform-level checking. |
| Слишком «сухая» теория без проектов | Средняя | Среднее | Каждый модуль имеет мини-проект; capstone M10 объединяет M01–M10. |
| Несоответствие версий .NET у студентов | Низкая | Низкое | Зафиксировать .NET 8 LTS в паспорте; инструкции по установке. |

---

# Версия v2.0 — «Professional» / Professional

> Цель: добавить Professional-трек — многопоточность, БД (EF Core), веб (ASP.NET Core), тесты и архитектуру. Курс становится пригодным для production-разработчиков.

## Что добавляется / расширяется / What is added / extended

### Новые модули (Core, Professional) / New modules

| ID | Модуль | Статус при v2.0 |
|---|---|---|
| M11 | Multithreading | ✅ Core, новый |
| M12 | Databases & EF Core | ✅ Core, новый |
| M13 | Web Basics | ✅ Core, новый |
| M14 | Web API | ✅ Core, новый |
| M15 | Testing | ✅ Core, новый |
| M16 | Architecture & Patterns | ✅ Core, новый (Professional capstone) |

### Расширение существующих / Extension of existing
- M06–M10 получают **Pro-слой** (Optional/Pro-теги): глубже по `Span<T>`, async streams, `System.Text.Json` source-gen, EF-провайдеры как мост к M12.
- Вводится **автопроверка проектов** на уровне платформы (integration-тесты через `WebApplicationFactory`).

### Треки / Tracks
- ✅ Foundation (готов с v1.0).
- ✅ **Professional** — полностью готов (M06–M16, ~120 ч).

### Домены, дополнительно покрытые / Domains additionally covered
Concurrency (потоки), Data (EF Core), Web (ASP.NET Core), Testing, Architecture.

### Форматы / Formats
- + Integration-тесты как автопроверка для веб/БД-лабораторных.
- + Code review-рубрикатор для capstone M16.
- + Возможность «ускоренного пути» для middle (skip-модули M01–M05 по диагностическому тесту).

## Цели v2.0 / Goals
1. Довести middle-разработчика до production-качества: тесты, архитектура, веб, БД.
2. Ввести automated grading для веб-проектов (запуск API + проверка контрактов).
3. Связать Professional capstone (M16) с реалистичным многослойным Web API.

## Метрики успеха / Success Metrics
- **Completion rate Professional-трека** ≥ 25% (из начавших M06 доходят до M16).
- **Capstone M16 сдан** у ≥ 40% прошедших M11–M15.
- **NPS** ≥ 40.
- **Доля студентов, использующих «ускоренный путь»** 20–40% (здорова воронка middle).
- **Среднее время прохождения Professional-трека** ≤ 8 недель (при 15 ч/нед).

## Риски v2.0 / Risks
| Риск | Вероятность | Влияние | Митигация |
|---|---|---|---|
| M16 (архитектура) — «размытая» тема, трудно автопроверять | Высокая | Высокое | Rubric-based code review + обязательные архитектурные ограничения в capstone. |
| EF Core (M12) требует БД-окружения — сложно в автопроверке | Высокая | Среднее | SQLite in-memory + `WebApplicationFactory`; миграции тестируются отдельно. |
| Разрыв между «Frontend нет» и реальными веб-проектами | Средняя | Среднее | Чётко зафиксировать scope: только API; frontend — Optional/ссылки. |
| M11 (многопоточность) слишком сложен после базового async | Средняя | Высокое | Мост M09 → M11 через `Task.Run` и пул потоков; Pro-детали (memory model) в Optional. |
| Устаревание версий EF Core/ASP.NET Core | Низкая | Среднее | Pin версий (.NET 8, EF Core 8); roadmap-точка v2.1 для минор-апдейтов. |

---

# Версия v3.0 — «Enterprise & Vision» / Enterprise & Vision

> Цель: завершить Enterprise-трек (performance, security, DevOps, микросервисы) и заложить долгосрочный vision курса: новые форматы, расширение стека, адаптация к платформенной эволюции .NET.

## Что добавляется / расширяется / What is added / extended

### Новые модули (Pro, Enterprise) / New modules

| ID | Модуль | Статус при v3.0 |
|---|---|---|
| M17 | Performance | ✅ Pro, новый |
| M18 | Security | ✅ Pro, новый |
| M19 | DevOps & Deployment | ✅ Pro, новый |
| M20 | Microservices & Enterprise | ✅ Pro, новый (Enterprise capstone) |

### Расширение существующих / Extension of existing
- M15–M16 получают **enterprise-акцент** (observability, security-by-design, ADR) — теперь полноценный мост в Enterprise.
- Вводятся **Deprecated-теги**: legacy-материалы (например, `Thread.Abort`, `Web Forms`, старые конфиг-подходы) выносятся в справочные приложения с указанием современной альтернативы.

### Треки / Tracks
- ✅ Foundation, ✅ Professional (готовы).
- ✅ **Enterprise** — полностью готов (M15–M20, ~100 ч).

### Домены, дополнительно покрытые / Domains additionally covered
Performance, Security, DevOps, расширенная Architecture (микросервисы, distributed systems).

### Новые форматы (vision) / New formats (vision)
- **Capstone-проекты с реальным CI/CD и observability** — студенты деплоят в sandbox-cloud.
- **ADR (Architecture Decision Records)** как формат заданий в M16/M20.
- **Симуляция инцидентов** (game days) в Enterprise-треке — отказы, degraded state, трейсинг.
- **Адаптивные пути**: диагностика → персональная последовательность (особенно для middle/senior).

## Цели v3.0 / Goals
1. Дать senior/архитекторам путь к enterprise-практике: performance, security, delivery, распределённые системы.
2. Ввести «живые» capstone-проекты с эксплуатацией (deploy, observability, incidents).
3. Зафиксировать Deprecated-политику и план миграции контента на будущие версии .NET.
4. Подготовить курс к масштабированию: адаптивные пути, локализации, форматы.

## Метрики успеха / Success Metrics
- **Completion rate Enterprise-трека** ≥ 20% (из начавших M15 доходят до M20).
- **Capstone M20 сдан** у ≥ 35% прошедших M17–M19.
- **NPS** ≥ 45.
- **Доля студентов, деплоивших capstone в sandbox** ≥ 60% от сдавших M20.
- **Покрытие всех 9 доменов** = 100% (Core/Data/Concurrency/Web/Testing/Architecture/Performance/Security/DevOps).

## Риски v3.0 / Risks
| Риск | Вероятность | Влияние | Митигация |
|---|---|---|---|
| Enterprise-теки требуют инфраструктуры (cloud, Docker registry, tracing) | Высокая | Высокое | Sandbox-cloud на стороне платформы; локальный Docker-first как fallback. |
| M20 (микросервисы) — слишком широкий scope, риск поверхностности | Высокая | Высокое | Чёткий scope: 2–3 сервиса, 1 API Gateway, 1 брокер (на выбор); глубина — в Optional-уроках. |
| Быстрая смена практик observability/security (OpenTelemetry, OWASP) | Средняя | Среднее | Pin версий + ежегодный аудит контента (v3.1, v3.2). |
| Нехватка экспертов-ревьюеров для enterprise capstone | Средняя | Высокое | Rubric-based review + peer review + автопроверка контрактов; v3.1 — менторская программа. |
| Перегруз senior-аудитории «базовыми» повторами M15–M16 | Средняя | Низкое | Адаптивный путь: diagnostic skip для M15–M16 при подтверждённом опыте. |

---

# Сводный таймлайн версий / Versions Timeline

| Версия | Фокус | Модули | Треки готовы | Домены покрыты | Часы курса |
|---|---|---|---|---|---|
| **v1.0** | MVP: язык и основы | M01–M10 | Foundation | Core, Data, Concurrency (base) | ~120 |
| **v2.0** | Production-разработчик | + M11–M16 | Foundation + Professional | + Web, Testing, Architecture, Concurrency (full) | ~240 |
| **v3.0** | Enterprise & vision | + M17–M20 | + Enterprise | + Performance, Security, DevOps | ~340 |

---

# Принятие решений по версиям / Version Decision Log

- **Почему v1.0 = только Foundation?** Минимально жизнеспособный курс должен давать законченную ценность (самостоятельные программы), быть полностью оттестированным и собрать обратную связь до инвестиций в старшие треки.
- **Почему Professional capstone — M16, а не M14?** Архитектура (SOLID/DI/слои) — синтез всего Professional-трека; Web API (M14) — лишь одна из его составляющих.
- **Почему M15 (Testing) входит и в Professional, и в Enterprise?** Тесты — фундамент enterprise-качества; в Enterprise трек тесты переосмысляются через призму архитектуры и эксплуатации.
- **Почему Enterprise-модули помечены Pro, а не Core?** Pro-метка явно сигнализирует о требованиях к базе и целевой аудитории (senior/архитектор); это не означает «необязательно», но «требует зрелости».
- **Почему ID модулей фиксированы?** Обратная совместимость: студенты, закладки и внешние ссылки на M0X остаются валидны; контент эволюционирует внутри карточки.

---

*Roadmap поддерживается совместно с паспортом курса (`01-course-passport.md`) и обновляется при переходе между версиями.*
*Roadmap is maintained alongside the course passport (`01-course-passport.md`) and updated on version transitions.*
