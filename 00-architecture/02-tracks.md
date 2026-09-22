# Таблицы треков / Track Tables

> Распределение модулей M01–M20 по трекам: Foundation / Professional / Enterprise.
> Distribution of modules M01–M20 across tracks: Foundation / Professional / Enterprise.

---

## Обзорная карта треков / Track Overview Map

| Трек / Track | Модули / Modules | Часы / Hours | Аудитория / Audience |
|---|---|---|---|
| Foundation | M01 → M10 | ~120 | Новичок / Beginner |
| Professional | M06 → M16 | ~120 | Middle |
| Enterprise | M15 → M20 | ~100 | Senior / Architect |

> Перекрытия намеренные: треки выстроены как «восходящие» — выпускник Foundation может продолжить в Professional, выпускник Professional — в Enterprise. Дублирующие модули (например, M06–M10) в старшем треке проходятся быстрее и с большим акцентом на Pro-детали.

---

## 1. Трек Foundation / Foundation Track

### 1.1 Модули трека / Modules in track
`M01, M02, M03, M04, M05, M06, M07, M08, M09, M10`

### 1.2 Цель трека / Track goal
Дать прочную базу языка C# и платформы .NET: от синтаксиса и типов до ООП, коллекций, LINQ, исключений, асинхронности и ввода-вывода. Выпускник способен самостоятельно писать корректные консольные и небольшие прикладные программы, читать чужой C#-код и отлаживать собственный.

> Provide a solid foundation in C# and .NET: from syntax and types to OOP, collections, LINQ, exceptions, async and I/O. A graduate can independently write correct console and small application programs, read third-party C# code and debug their own.

### 1.3 Рекомендуемая последовательность / Recommended sequence

```
M01 → M02 → M03 → M04 → M05 → M06 → M07 → M08 → M09 → M10
```

- M01–M03 — синтаксис и базовые конструкции (Core, строго последовательно).
- M04–M05 — ООП (опирается на M02–M03).
- M06 — коллекции и дженерики (опирается на M05).
- M07 — исключения (опирается на M03, M06).
- M08 — LINQ (опирается на M06, M07).
- M09 — async/await (опирается на M07).
- M10 — I/O и сериализация (опирается на M08, M09).

### 1.4 Пререквизиты трека / Prerequisites
- Базовая компьютерная грамотность / basic computer literacy.
- Установленный .NET SDK 8+ и IDE (VS / VS Code / Rider).
- Английский на уровне чтения технической документации (рекомендуется / recommended).
- Предыдущих модулей не требуется / no prior modules required.

### 1.5 Ожидаемый результат / Expected outcome
Выпускник Foundation-трека умеет / A Foundation graduate can:
- Писать структурированный C#-код с использованием типов, методов, ООП.
- Применять коллекции, дженерики, обработку исключений.
- Использовать LINQ для запросов к данным в памяти.
- Писать асинхронные методы и работать с файлами/потоками.
- Запускать и отлаживать .NET-приложения в IDE.
- Создать финальный мини-проект: консольное приложение с CRUD над файлом/JSON (объединяет M01–M10).

---

## 2. Трек Professional / Professional Track

### 2.1 Модули трека / Modules in track
`M06, M07, M08, M09, M10, M11, M12, M13, M14, M15, M16`

### 2.2 Цель трека / Track goal
Перевести разработчика с «умею писать рабочий код» на «умею проектировать и поставлять production-качественные приложения»: многопоточность, БД (EF Core), веб (ASP.NET Core Web API), тестирование и архитектура (SOLID, DI, паттерны).

> Move a developer from «can write working code» to «can design and deliver production-quality applications»: multithreading, databases (EF Core), web (ASP.NET Core Web API), testing and architecture (SOLID, DI, patterns).

### 2.3 Рекомендуемая последовательность / Recommended sequence

```
M06 → M07 → M08 → M09 → M10 → M11 → M12 → M13 → M14 → M15 → M16
```

- M06–M10 — ускоренное повторение базы + Pro-детали (для тех, кто пришёл извне курса).
- M11 — многопоточность (опирается на M09 async).
- M12 — EF Core и БД (опирается на M08 LINQ, M10 сериализация).
- M13 → M14 — ASP.NET Core: основы → Web API (опирается на M12, M11).
- M15 — тестирование (опирается на M13, M14).
- M16 — архитектура, SOLID, DI (опирается на M04, M05, M15) — синтез трека.

> Для выпускников Foundation-трека модули M06–M10 можно проходить ускоренно, концентрируясь на Pro-слое.

### 2.4 Пререквизиты трека / Prerequisites
- Завершённый Foundation-трек **ИЛИ** эквивалентные знания (M01–M10 по alignSelf-assessment).
- Уверенное владение: типы, ООП, LINQ, базовый `async`/`await`, исключения.
- Опыт работы в командной строке (`dotnet` CLI).
- Понимание основ HTTP и SQL (рекомендуется / recommended).

### 2.5 Ожидаемый результат / Expected outcome
Выпускник Professional-трека умеет / A Professional graduate can:
- Проектировать многопоточные и асинхронные приложения, понимать trade-offs.
- Моделировать предметную область и работать с БД через EF Core (migrations, LINQ-providers).
- Создавать REST Web API на ASP.NET Core (Minimal API, Middleware, DI, конфигурация).
- Покрывать код unit- и integration-тестами (xUnit, Moq), обеспечивать тестируемость.
- Применять SOLID, DI-контейнеры, базовые паттерны (Repository, Factory, Strategy).
- Создать финальный проект: многослойное Web API с БД, тестами и DI (объединяет M11–M16).

---

## 3. Трек Enterprise / Enterprise Track

### 3.1 Модули трека / Modules in track
`M15, M16, M17, M18, M19, M20`

### 3.2 Цель трека / Track goal
Подготовить архитекторов и senior-разработчиков к созданию масштабируемых, надёжных, безопасных и наблюдаемых систем: performance, security, DevOps/CI-CD, микросервисы и enterprise-архитектура. Фокус — на эксплуатации, масштабировании и культуре delivery.

> Prepare architects and senior developers to build scalable, reliable, secure and observable systems: performance, security, DevOps/CI-CD, microservices and enterprise architecture. Focus is on operations, scaling and delivery culture.

### 3.3 Рекомендуемая последовательность / Recommended sequence

```
M15 → M16 → M17 → M18 → M19 → M20
```

- M15–M16 — тесты и архитектура как фундамент enterprise-качества (если уже пройдены в Professional — повторить с enterprise-акцентом).
- M17 — performance и профилирование (опирается на M09, M11, M16).
- M18 — безопасность (опирается на M13, M14, M16).
- M19 — DevOps, CI/CD, Docker, деплой (опирается на M14, M15).
- M20 — микросервисы и enterprise-архитектура — синтез всего трека (опирается на M16, M17, M18, M19).

### 3.4 Пререквизиты трека / Prerequisites
- Завершённый Professional-трек **ИЛИ** эквивалентный production-опыт (3+ года).
- Уверенное владение: ASP.NET Core, EF Core, тесты, SOLID/DI.
- Опыт работы с Git, CI/CD (на базовом уровне).
- Понимание сетей, HTTP, Docker на бытовом уровне (рекомендуется / recommended).

### 3.5 Ожидаемый результат / Expected outcome
Выпускник Enterprise-трека умеет / An Enterprise graduate can:
- Профилировать и оптимизировать .NET-приложения (CPU, memory, GC, allocations).
- Проектировать безопасные системы: аутентификация, авторизация (JWT, OAuth), шифрование, защита от типовых угроз (OWASP).
- Выстраивать CI/CD-пайплайны, контейнеризировать приложения (Docker), деплоить в cloud/on-prem.
- Проектировать микросервисные и распределённые системы: межсервисное взаимодействие, resilience (Polly), observability (OpenTelemetry, logging, tracing, metrics).
- Принимать архитектурные решения с учётом trade-offs (команда, эксплуатация, масштаб, стоимость).
- Создать финальный проект: микросервисное решение с CI/CD, observability и security (объединяет M17–M20).

---

## 4. Матрица «Модуль × Трек» / Module × Track Matrix

| Модуль / Module | Foundation | Professional | Enterprise |
|---|:---:|:---:|:---:|
| M01 Intro | ✅ | — | — |
| M02 Types & Operators | ✅ | — | — |
| M03 Control Flow & Methods | ✅ | — | — |
| M04 OOP Basics | ✅ | — | — |
| M05 OOP Advanced | ✅ | — | — |
| M06 Collections & Generics | ✅ | ✅ | — |
| M07 Exception Handling | ✅ | ✅ | — |
| M08 LINQ | ✅ | ✅ | — |
| M09 Async Programming | ✅ | ✅ | — |
| M10 I/O & Serialization | ✅ | ✅ | — |
| M11 Multithreading | — | ✅ | — |
| M12 Databases & EF Core | — | ✅ | — |
| M13 Web Basics | — | ✅ | — |
| M14 Web API | — | ✅ | — |
| M15 Testing | — | ✅ | ✅ |
| M16 Architecture & Patterns | — | ✅ | ✅ |
| M17 Performance | — | — | ✅ |
| M18 Security | — | — | ✅ |
| M19 DevOps & Deployment | — | — | ✅ |
| M20 Microservices & Enterprise | — | — | ✅ |

---

## 5. Рекомендации по прохождению / Sequencing Guidance

- **Линейный путь новичка:** M01 → … → M20 (полный курс, ~340 ч).
- **Сокращённый путь middle:** начать с M06 (быстрое повторение базы) → … → M20.
- **Сокращённый путь senior:** начать с M15/M16 (или сразу M17, если базы достаточно) → … → M20.
- **Модули-мостики:** M06–M10 служат мостом Foundation → Professional; M15–M16 — мостом Professional → Enterprise.

---

*Треки согласованы с таблицей модулей (`03-modules-table.md`) и дорожной картой (`04-roadmap.md`).*
*Tracks are consistent with the modules table (`03-modules-table.md`) and the roadmap (`04-roadmap.md`).*
