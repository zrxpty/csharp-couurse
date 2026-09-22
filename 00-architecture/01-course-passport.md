# Паспорт курса / Course Passport

> Полный курс по C# — от основ до enterprise-уровня
> Complete C# Course — from foundations to enterprise level

---

## 1. Идентификация курса / Course Identification

| Поле / Field | Значение / Value |
|---|---|
| **Название (RU)** | Полный курс по C#: от основ до enterprise-архитектуры |
| **Title (EN)** | Complete C# Course: From Foundations to Enterprise Architecture |
| **Код курса / Course code** | CSHARP-FULL |
| **Версия / Version** | v1.0 (текущая разрабатываемая / current draft) |
| **Статус / Status** | In Development / В разработке |
| **Язык контента / Content language** | RU + EN (двуязычно / bilingual) |
| **Создан / Created** | 2025 |
| **Владелец / Owner** | Команда методологии / Methodology Team (Agent 1 «Course Architect») |

---

## 2. Цели курса / Learning Outcomes

К концу курса выпускник сможет / By the end of the course, a graduate will be able to:

1. **Основы языка / Language foundations** — писать корректный, идиоматичный C# 12 код, понимать систему типов, управление памятью (GC), и применять современные возможности языка (records, pattern matching, top-level statements).
2. **ООП и дизайн / OOP & design** — проектировать типы и иерархии с применением принципов SOLID, паттернов проектирования и инверсии зависимостей (DI).
3. **Работа с данными / Data access** — использовать LINQ, дженерики, коллекции и Entity Framework Core для эффективной работы с данными в памяти и в БД.
4. **Асинхронность и параллелизм / Async & concurrency** — уверенно применять `async`/`await`, `Task`, многопоточность и понимать trade-offs производительности и безопасности.
5. **Веб-разработка / Web development** — создавать REST API и веб-приложения на ASP.NET Core 8 (Minimal API, Middleware, DI, конфигурация).
6. **Качество и тестирование / Quality & testing** — покрывать код тестами (xUnit, Moq), обеспечивать тестируемость архитектуры, применять profiling и диагностику.
7. **Enterprise-практика / Enterprise practice** — применять микросервисную архитектуру, CI/CD, Docker, безопасность и observability для production-ready решений.

---

## 3. Целевая аудитория / Target Audience

### 3.1 Новичок / Beginner (Foundation track)
- **Профиль:** Нет или минимальный опыт программирования; возможно знакомство с другим языком.
- **Цель:** Получить прочную базу C# и .NET, дойти до самостоятельных мини-проектов.
- **Боли:** Перегруз терминологией, страх ошибок, непонимание «как мыслит компьютер».
- **Трек:** Foundation (M01–M10, Core-модули).

### 3.2 Middle-разработчик / Mid-level (Professional track)
- **Профиль:** 1–3 года опыта; знает основы, пишет рабочий код, но хочет системности и глубины.
- **Цель:** Освоить LINQ, async, тестирование, архитектуру, веб — перейти к production-качеству.
- **Боли:** «Знаю как, но не знаю почему», слабые места в дизайне и тестах.
- **Трек:** Professional (M06–M16, Core + Pro).

### 3.3 Senior / Enterprise-разработчик (Enterprise track)
- **Профиль:** 3+ года опыта; проектирует системы, отвечает за архитектуру и delivery.
- **Цель:** Освоить performance, безопасность, DevOps, микросервисы, observability.
- **Боли:** Масштаб, надёжность, эксплуатация, культура команды.
- **Трек:** Enterprise (M15–M20, Pro-модули).

---

## 4. Длительность курса / Course Duration

| Составляющая / Component | Часы / Hours |
|---|---|
| Теория / Theory | 120 |
| Практика (лабораторные, упражнения) / Practice (labs, exercises) | 140 |
| Проектная работа / Project work | 80 |
| **Итого / Total** | **340 ч / hours** |

- **Foundation:** ~120 ч (теория 50 + практика 50 + проекты 20)
- **Professional:** ~120 ч (теория 40 + практика 50 + проекты 30)
- **Enterprise:** ~100 ч (теория 30 + практика 40 + проекты 30)

> Длительность ориентировочная; зависит от темпа обучающегося и глубины опциональных модулей.

---

## 5. Технологический стек и версии / Tech Stack & Versions

| Компонент / Component | Версия / Version | Примечание / Note |
|---|---|---|
| C# | 12+ | Top-level statements, records, primary constructors, pattern matching |
| .NET | 8+ (LTS) | Базовый рантайм и SDK / base runtime & SDK |
| .NET SDK | 8.0.x | `dotnet` CLI |
| IDE | Visual Studio 2022 / VS Code / Rider | На выбор / at choice |
| ASP.NET Core | 8 | Minimal API, MVC, Middleware, DI |
| EF Core | 8 | Code-first, migrations, LINQ providers |
| xUnit | 2.x + | Тест-фреймворк / test framework |
| Moq | 4.20+ | Моки / mocking |
| Docker | 24+ | Контейнеризация / containerization |
| CI/CD | GitHub Actions / Azure DevOps | На выбор / at choice |
| СУБД / DB | SQL Server / PostgreSQL / SQLite | Учебные примеры / teaching examples |

---

## 6. Треки / Tracks

### 6.1 Foundation — «Основы»
- **Для кого:** Новички, переходящие из других языков.
- **Что входит:** M01–M10 (язык, ООП, коллекции, исключения, LINQ, async, I/O).
- **Цель:** Самостоятельно писать консольные и небольшие прикладные программы на C#.
- **Метка контента по умолчанию:** Core.

### 6.2 Professional — «Профессионал»
- **Для кого:** Middle-разработчики, стремящиеся к production-качеству.
- **Что входит:** M06–M16 (LINQ, async, I/O, потоки, БД, веб, тесты, архитектура).
- **Цель:** Проектировать и реализовывать тестируемые веб-приложения и сервисы с БД.
- **Метка контента по умолчанию:** Core + Pro.

### 6.3 Enterprise — «Enterprise»
- **Для кого:** Senior-разработчики и архитекторы.
- **Что входит:** M15–M20 (тесты, архитектура, performance, безопасность, DevOps, микросервисы).
- **Цель:** Строить масштабируемые, наблюдаемые, безопасные системы и управлять их delivery.
- **Метка контента по умолчанию:** Pro.

> Подробное описание треков — в файле `02-tracks.md`.

---

## 7. Версионирование контента / Content Versioning Tags

Каждый урок/модуль носит одну из меток, определяющую его место в курсе:

| Метка / Tag | Значение / Meaning | Когда использовать / When to use |
|---|---|---|
| **Core** | Обязательное ядро курса. Без этого материал не считается усвоенным. | Базовые концепции, необходимые всем трекам. Должно быть в v1.0. |
| **Optional** | Полезно, но не обязательно для основного пути. Можно пропустить без потери целостности. | Углублённые детали, нишевые API, альтернативные подходы. |
| **Pro** | Продвинутый материал для Professional/Enterprise. Требует устойчивой базы. | Performance, security, микросервисы, deep async, advanced DI. |
| **Deprecated** | Устаревший материал. Оставлен для справки/миграции legacy. | Старые API (например, `Web Forms`, `Thread.Abort`), подходы .NET Framework. |

**Правила / Rules:**
1. Core-модули входят во все треки, где они пререквизиты.
2. Optional можно проходить по желанию; не влияет на «зачёт» трека.
3. Pro требует завершения всех Core-пререквизитов.
4. Deprecated помечается явно и сопровождается современной альтернативой.

---

## 8. Домены / Domains

Домены — это тематические оси, пересекающие модули. Каждый модуль принадлежит одному основному домену.

| Домен / Domain | Определение / Definition | Примеры модулей / Example modules |
|---|---|---|
| **Core** | Основы языка и платформы: типы, синтаксис, ООП, поток выполнения, коллекции. | M01, M02, M03, M04, M05, M06, M07 |
| **Data** | Работа с данными: in-memory коллекции, LINQ, сериализация, БД, EF Core. | M08, M10, M12 |
| **Concurrency** | Асинхронность и параллелизм: `async`/`await`, `Task`, потоки, синхронизация. | M09, M11 |
| **Web** | Веб-разработка на ASP.NET Core: API, REST, Middleware, маршрутизация. | M13, M14 |
| **Testing** | Качество ПО: unit/integration тесты, моки, тестируемость. | M15 |
| **Architecture** | Дизайн систем: паттерны, SOLID, DI, слои, микросервисы. | M16, M20 |
| **Performance** | Производительность и диагностика: профилирование, аллокации, GC. | M17 |
| **Security** | Безопасность: аутентификация, авторизация, шифрование, угрозы. | M18 |
| **DevOps** | Доставка: CI/CD, Docker, конфигурация окружений, деплой. | M19 |

> В таблице модулей (`03-modules-table.md`) домен указан явно в каждой карточке. Для простоты паспорта используется укрупнённый список: **Core / Web / Data / Architecture / Testing / Performance** (Concurrency и Security входят как подобласти; DevOps — отдельная область доставки).

---

## 9. Связанные артефакты / Related Artifacts

- `02-tracks.md` — таблицы треков / track tables
- `03-modules-table.md` — карточки модулей M01–M20 / module cards
- `04-roadmap.md` — дорожная карта v1.0 → v2.0 → v3.0 / roadmap

---

*Документ поддерживается Агентом 1 «Course Architect». Изменения версионируются вместе с roadmap.*
*Maintained by Agent 1 «Course Architect». Changes are versioned alongside the roadmap.*
