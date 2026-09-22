# Полный курс по C# / Complete C# Course

> Двуязычный курс (RU / EN) · C# 12+ · .NET 8+ · Модули M01–M20
> Bilingual course (RU / EN) · C# 12+ · .NET 8+ · Modules M01–M20

---

## О курсе / About

Этот репозиторий содержит полный, масштабируемый курс по C# — от основ языка до enterprise-архитектуры. Курс построен как система из 7 специализированных ролей (архитектор, контент-мейкер, инженер заданий, тест-дизайнер, ревьюер кода, продуктовый менеджер, локализатор).

This repository contains a complete, scalable C# course — from language fundamentals to enterprise architecture. The course is built as a system of 7 specialized roles (architect, content creator, exercise engineer, quiz designer, code reviewer, product manager, localizer).

**Версия / Version:** v1.0 (Foundation track — в разработке / in progress)

---

## Структура репозитория / Repository structure

```
course/
├── 00-architecture/          # Фаза 1: архитектура курса / Phase 1: course architecture
│   ├── 01-course-passport.md   # Паспорт курса / Course passport
│   ├── 02-tracks.md            # Таблицы треков / Track tables
│   ├── 03-modules-table.md     # Карточки модулей M01–M20 / Module cards
│   └── 04-roadmap.md           # Дорожная карта v1→v2→v3 / Roadmap
├── modules/                  # Фазы 2–4: контент, практика, тесты / Phases 2–4
│   ├── M01/                    # Введение в C# и .NET / Intro to C# & .NET
│   ├── M02/                    # Типы, переменные, операторы / Types, Variables, Operators
│   ├── M03/                    # Управление потоком, методы / Control Flow & Methods
│   └── ...                     # M04–M20 (расширяется в v2.0/v3.0)
└── README.md                 # Этот файл / This file
```

---

## Треки / Tracks

| Трек / Track | Уровень / Level | Модули / Modules | Назначение / Purpose |
|---|---|---|---|
| **Foundation** | Начинающий / Beginner | M01–M10 | Основа языка и .NET / Language & .NET fundamentals |
| **Professional** | Middle | M06–M16 | Веб, данные, тестирование, архитектура / Web, data, testing, architecture |
| **Enterprise** | Senior | M15–M20 | Производительность, безопасность, DevOps, микросервисы / Performance, security, DevOps, microservices |

См. / See: [`00-architecture/02-tracks.md`](00-architecture/02-tracks.md)

---

## Список модулей / Modules index

| ID | Название RU / Title EN | Уровень | Домен | Статус | Готовность / Readiness |
|----|------------------------|---------|-------|--------|------------------------|
| M01 | Введение в C# и .NET / Intro to C# & .NET | Foundation | Core | Core | ✅ контент+задания+quiz+ревью |
| M02 | Типы, переменные, операторы / Types, Variables, Operators | Foundation | Core | Core | ✅ контент+задания+quiz+ревью |
| M03 | Управление потоком, методы / Control Flow & Methods | Foundation | Core | Core | ✅ контент+задания+quiz+ревью |
| M04 | ООП: классы, объекты / OOP Basics | Foundation | Core | Core | ⏳ запланирован / planned |
| M05 | ООП: наследование, полиморфизм / OOP Advanced | Foundation | Core | Core | ⏳ запланирован / planned |
| M06 | Коллекции и дженерики / Collections & Generics | Foundation | Core | Core | ⏳ запланирован / planned |
| M07 | Обработка исключений / Exception Handling | Foundation | Core | Core | ⏳ запланирован / planned |
| M08 | LINQ | Foundation | Core | Core | ⏳ запланирован / planned |
| M09 | Асинхронное программирование / Async Programming | Foundation | Core | Core | ⏳ запланирован / planned |
| M10 | Файлы, потоки, сериализация / I/O & Serialization | Foundation | Core | Core | ⏳ запланирован / planned |
| M11 | Многопоточное программирование / Multithreading | Professional | Core | Core | ⏳ v2.0 |
| M12 | Базы данных (EF Core) / Databases & EF Core | Professional | Data | Core | ⏳ v2.0 |
| M13 | ASP.NET Core основы / Web Basics | Professional | Web | Core | ⏳ v2.0 |
| M14 | Web API, REST, Minimal API / Web API | Professional | Web | Core | ⏳ v2.0 |
| M15 | Тестирование (xUnit, Moq) / Testing | Professional | Testing | Core | ⏳ v2.0 |
| M16 | Архитектура: паттерны, SOLID, DI / Architecture & Patterns | Professional | Architecture | Core | ⏳ v2.0 |
| M17 | Производительность и профилирование / Performance | Enterprise | Performance | Pro | ⏳ v3.0 |
| M18 | Безопасность / Security | Enterprise | Architecture | Pro | ⏳ v3.0 |
| M19 | DevOps: CI/CD, Docker, деплой / DevOps & Deployment | Enterprise | Architecture | Pro | ⏳ v3.0 |
| M20 | Микросервисы и enterprise-архитектура / Microservices | Enterprise | Architecture | Pro | ⏳ v3.0 |

> Подробные карточки с prerequisites, временем, сложностью — см. [`00-architecture/03-modules-table.md`](00-architecture/03-modules-table.md)

---

## Структура каждого модуля / Per-module structure

Каждый модуль `modules/M0X/` содержит артефакты трёх фаз:
Each module `modules/M0X/` contains artifacts from three phases:

| Файл / File | Фаза / Phase | Автор / Author | Содержание / Content |
|---|---|---|---|
| `lessons/M0X-L0Y.md` | 2 | Агент 2 / Content Creator | Теория (RU+EN) + код + best practices + чек-лист |
| `exercises.md` | 3 | Агент 3 / Exercise Engineer | Задания уроков + мини-проект (base + pro) |
| `quiz.md` | 4 | Агент 4 / Quiz Designer | 12–14 вопросов с объяснениями |
| `code-review.md` | 2 | Агент 5 / Code Reviewer | Ревью кода уроков |

---

## Роли системы / System roles

| # | Роль / Role | Фаза / Phase | Объект работы / Scope |
|---|---|---|---|
| 1 | Архитектор / Course Architect | 1 | Структура, треки, карта курса |
| 2 | Контент-мейкер / Content Creator | 2 | Теория и код уроков |
| 3 | Инженер заданий / Exercise Engineer | 3 | Практика и проекты |
| 4 | Тест-дизайнер / Quiz Designer | 4 | Quizzes и оценка |
| 5 | Ревьюер / Code Reviewer | 2–3 | Проверка кода и решений |
| 6 | Продуктовый менеджер / Product Manager | 5 | Тарифы, маркетинг, метрики |
| 7 | Локализатор / Localizer | 6 | Глоссарий, адаптация |

---

## Статус сборки / Build status

- **Фаза 1 — Архитектура / Architecture:** ✅ Завершено / Complete (M01–M20, 4 файла / 4 files)
- **Фаза 2 — Контент / Content (M01–M03):** ✅ Завершено / Complete (22 урока / 22 lessons, RU+EN)
- **Фаза 2 — Ревью кода / Code review (M01–M03):** ✅ Завершено / Complete (3 отчёта + исправления M02)
- **Фаза 3 — Практика / Exercises (M01–M03):** ✅ Завершено / Complete (задания + мини-проекты base+pro)
- **Фаза 4 — Тесты / Quizzes (M01–M03):** ✅ Завершено / Complete (12+14+14 = 40 вопросов / questions)
- **Фазы 5–6 (Продукт, локализация):** за рамками текущего объёма / out of current scope

**Итого / Totals:** 36 файлов / 36 files · ~697 КБ / ~697 KB · C# 12+ / .NET 8+
