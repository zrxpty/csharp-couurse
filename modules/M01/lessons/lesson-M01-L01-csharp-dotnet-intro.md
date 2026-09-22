---
[← Предыдущий: README](../../README.md) | [⬆ К модулю M01](../README.md) | [Следующий: M01-L02 →](lesson-M01-L02-clr-il-jit.md)
---

### Урок M01-L01: Что такое C# и .NET / What is C# and .NET

**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Представь, что у тебя есть автомобиль. Двигатель — это мощная, сложная машина, которая умеет превращать топливо в движение. Но сам по себе двигатель не знает, куда ехать: ему нужны команды. C# — это язык, на котором ты пишешь эти команды («поверни налево», «остановись», «разгонись до 60»), а .NET — это сам двигатель, который эти команды исполняет. Без .NET код на C# — лишь текст; без C# у .NET нет инструкций для работы.

Исторически платформа .NET развивалась несколькими ветками. В 2002 году Microsoft выпустила **.NET Framework** — мощную, но привязанную к Windows систему. Она включала огромную библиотеку классов и среду выполнения (CLR), но работала только на Windows. Затем появился **Mono** — независимая реализация для Linux и macOS, а в 2014 году Microsoft представила **.NET Core** — современную, модульную, кроссплатформенную переработку платформы. .NET Core рос версиями 1.x, 2.x, 3.x и к версии 3.1 сталProduction-ready для серверов и консольных приложений.

Начиная с **.NET 5** (2020) Microsoft отказалась от слова «Core» и объединила ветки развития в единую платформу. Версии 5, 6 (LTS), 7 и 8 (LTS) — это одна и та же современная .NET, которая работает на Windows, Linux и macOS, поддерживает мобильные (MAUI), облачные, десктопные и игровые сценарии. .NET 8 — текущая LTS-версия, рекомендованная для новых проектов; .NET 9 и дальше продолжают эту нумерацию. Старый .NET Framework 4.x по-прежнему поддерживается, но получает лишь исправления безопасности.

Что такое **managed code** (управляемый код)? Это код, который выполняется под контролем **CLR** (Common Language Runtime) — среды выполнения .NET. CLR берёт скомпилированный промежуточный код (IL — Intermediate Language) и превращает его в машинные инструкции через JIT-компиляцию. CLR же управляет памятью: автоматически выделяет объекты и освобождает их через сборщик мусора (GC), проверяет типы, обрабатывает исключения. Ты не освобождаешь память вручную, как в C/C++; CLR делает это за тебя. За эту безопасность и удобство платишь небольшими накладными расходами и зависимостью от среды выполнения.

**Кроссплатформенность** означает, что один и тот же скомпилированный код работает на разных операционных системах. Ты пишешь библиотеку классов на Windows, публикуешь её — и она запускается на Linux-сервере или в контейнере Docker без перекомпиляции. Это возможно благодаря тому, что CLR и базовая библиотека реализованы для каждой ОС. Конечно, системно-зависимые вызовы (например, прямая работа с Win32 API) остаются привязанными к платформе, но основная логика переносима.

Наконец, **BCL** — Base Class Library — это огромная стандартная библиотека классов и функций, входящая в состав .NET. В ней есть коллекции (`List<T>`, `Dictionary<K,V>`), работа со строками, файлами (`System.IO`), сетью, датами (`DateTime`), криптографией, LINQ, асинхронность (`Task`, `async/await`). Ты не пишешь всё с нуля: BCL даёт фундамент. Аналогия: если .NET — двигатель, то BCL — это готовые узлы и агрегаты автомобиля (коробка передач, тормоза), которые ты собираешь в своё приложение. С# описывает, как эти узлы соединить и привести в движение.

#### Theory (EN)

Imagine you have a car. The engine is a powerful, complex machine that turns fuel into motion, but by itself it does not know where to go — it needs commands. **C#** is the language in which you write those commands ("turn left", "stop", "accelerate to 60"), and **.NET** is the engine that executes them. Without .NET, C# code is just text; without C#, .NET has no instructions to run.

Historically, the .NET platform evolved along several branches. In 2002 Microsoft released **.NET Framework**, a powerful but Windows-bound system. It included a huge class library and a runtime (CLR), but ran only on Windows. Then **Mono** appeared as an independent implementation for Linux and macOS, and in 2014 Microsoft introduced **.NET Core** — a modern, modular, cross-platform redesign of the platform. .NET Core grew through versions 1.x, 2.x, and 3.x, and by 3.1 it was production-ready for servers and console applications.

Starting with **.NET 5** (2020), Microsoft dropped the word "Core" and merged the development branches into a single platform. Versions 5, 6 (LTS), 7, and 8 (LTS) are the same modern .NET that runs on Windows, Linux, and macOS and supports mobile (MAUI), cloud, desktop, and game scenarios. .NET 8 is the current LTS release recommended for new projects; .NET 9 and beyond continue this numbering. The older .NET Framework 4.x is still supported but only receives security fixes.

What is **managed code**? It is code that runs under the control of the **CLR** (Common Language Runtime), the .NET execution environment. The CLR takes compiled intermediate code (IL — Intermediate Language) and turns it into machine instructions through JIT compilation. The CLR also manages memory: it allocates objects and frees them automatically through the garbage collector (GC), checks types, and handles exceptions. You do not free memory by hand as in C/C++; the CLR does it for you. The price for this safety and convenience is a small overhead and dependence on a runtime.

**Cross-platform** means that the same compiled code runs on different operating systems. You write a class library on Windows, publish it, and it runs on a Linux server or in a Docker container without recompilation. This works because the CLR and the base library are implemented for each OS. Of course, system-dependent calls (for example direct Win32 API usage) stay tied to a platform, but the core logic is portable.

Finally, the **BCL** — Base Class Library — is the large standard library of classes and functions that ships with .NET. It contains collections (`List<T>`, `Dictionary<K,V>`), string handling, files (`System.IO`), networking, dates (`DateTime`), cryptography, LINQ, and asynchrony (`Task`, `async/await`). You do not build everything from scratch: the BCL is the foundation. To extend the analogy: if .NET is the engine, the BCL is the ready-made components of the car (gearbox, brakes) that you assemble into your application, and C# describes how to connect them and set them in motion.

#### Пример кода / Code Example

```csharp
// Урок M01-L01: знакомство с C# 12 / .NET 8
// Lesson M01-L01: introduction to C# 12 / .NET 8

// Top-level statements — точка входа без boilerplate-класса Program.
// Top-level statements — entry point without a boilerplate Program class.

using System;
using System.Collections.Generic;
using System.IO;
using System.Linq;

// Демонстрируем кроссплатформенность: путь берётся из BCL, работает на любой ОС.
// Demonstrate cross-platform behavior: the path comes from the BCL and works on any OS.
string osLabel = Environment.OSVersion.Platform switch
{
    PlatformID.Win32NT => "Windows",
    PlatformID.Unix    => "Linux/macOS",
    _                  => "Unknown OS"
};

Console.WriteLine($"Привет от .NET {Environment.Version} на {osLabel}!");
// Hello from .NET {version} on {os}!

// BCL: коллекции и LINQ — без внешних библиотек.
// BCL: collections and LINQ — no external libraries needed.
List<string> frameworks =
[
    ".NET Framework 4.8",
    ".NET Core 3.1",
    ".NET 6 (LTS)",
    ".NET 8 (LTS)",
];

string recommended = frameworks
    .Where(f => f.Contains("8"))
    .Select(f => f.Replace(" (LTS)", ""))
    .First();

// Raw string literal (C# 11+) — удобно для многострочного текста.
// Raw string literal (C# 11+) — handy for multi-line text.
string summary = $"""
    Рекомендованная версия для новых проектов: {recommended}
    Recommended version for new projects: {recommended}
    """;

Console.WriteLine(summary);

// Managed code: сборщик мусора сам освободит Dictionary, вручную ничего не делаем.
// Managed code: the garbage collector frees the Dictionary on its own; nothing to do by hand.
var meta = new Dictionary<string, object>
{
    ["Language"] = "C#",
    ["Version"] = 12,
    ["Platform"] = ".NET 8",
    ["CrossPlatform"] = true,
    ["Managed"] = true,
};

foreach (var (key, value) in meta)
{
    Console.WriteLine($"{key,-15}: {value}");
}

// Запишем результат в файл через System.IO — это часть BCL.
// Write the result to a file through System.IO — part of the BCL.
string outputDir = Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData), "M01-L01");
Directory.CreateDirectory(outputDir);
string filePath = Path.Combine(outputDir, "summary.txt");
File.WriteAllText(filePath, summary);

Console.WriteLine($"Файл сохранён / File saved: {filePath}");
```

#### Best Practices
- Выбирай .NET 8 (LTS) для новых проектов — стабильность и долгосрочная поддержка от Microsoft. / Choose .NET 8 (LTS) for new projects — stability and long-term Microsoft support.
- Используй top-level statements в небольших программах, чтобы убрать лишний шаблонный код; переключайся на класс `Program` только когда нужно. / Use top-level statements in small programs to remove boilerplate; switch to a `Program` class only when needed.
- Опирайся на BCL и NuGet-пакеты вместо написания собственных велосипедов; так код легче поддерживать и тестировать. / Rely on the BCL and NuGet packages instead of reinventing the wheel; the code is easier to maintain and test.
- Таргетируй `net8.0` как базовый TFM, добавляй `net8.0-windows`/`net8.0-ios` только если используешь платформо-зависимые API. / Target `net8.0` as the base TFM, add `net8.0-windows`/`net8.0-ios` only when you use platform-specific APIs.
- Доверяй сборщику мусора; не вызывай `GC.Collect()` без profiling-обоснования. / Trust the garbage collector; do not call `GC.Collect()` without profiling evidence.

#### Частые ошибки / Common Mistakes
- Путать .NET Framework 4.x и современный .NET (5/6/7/8) — это разные среды выполнения. → Запомни: .NET Framework = только Windows и в режиме поддержки; новый .NET = кроссплатформенный и активно развивается. (RU)
- Думать, что C# и .NET — это одно и то же. → C# — язык, .NET — платформа (движок + BCL + CLR); код можно писать и на F#, VB.NET. (RU)
- Пытаться освобождать память вручную через `Dispose` или финализаторы для всех объектов. → `Dispose` нужен только для неуправляемых ресурсов (файлы, сокеты); остальное забирает GC. (RU)
- Использовать `Thread.Sleep` для асинхронного ожидания. → Применяй `await Task.Delay(...)`, чтобы не блокировать поток. (RU)
- Confusing .NET Framework 4.x with modern .NET (5/6/7/8) — they are different runtimes. → Remember: .NET Framework = Windows-only and in maintenance; modern .NET = cross-platform and actively developed. (EN)
- Thinking C# and .NET are the same thing. → C# is a language, .NET is the platform (runtime + BCL + CLR); you can also write F# or VB.NET. (EN)
- Trying to free memory manually via `Dispose` or finalizers for every object. → `Dispose` is only for unmanaged resources (files, sockets); the GC handles the rest. (EN)
- Using `Thread.Sleep` for asynchronous waiting. → Use `await Task.Delay(...)` so you do not block the thread. (EN)

#### Чек-лист самопроверки / Self-check Checklist
- [ ] Я могу объяснить разницу между .NET Framework, .NET Core и .NET 5/6/7/8. (RU)
- [ ] Я понимаю, что такое managed code и какую роль играет CLR и сборщик мусора. (RU)
- [ ] Я могу назвать хотя бы три пространства имён из BCL (`System.IO`, `System.Linq`, `System.Collections.Generic`). (RU)
- [ ] Я знаю, почему код на C# работает на разных ОС без перекомпиляции. (RU)
- [ ] Я могу запустить пример урока локально на `dotnet 8` и вижу путь к сохранённому файлу. (RU)
- [ ] I can explain the difference between .NET Framework, .NET Core, and .NET 5/6/7/8. (EN)
- [ ] I understand what managed code is and the role of the CLR and garbage collector. (EN)
- [ ] I can name at least three namespaces from the BCL (`System.IO`, `System.Linq`, `System.Collections.Generic`). (EN)
- [ ] I know why C# code runs on different OSes without recompilation. (EN)
- [ ] I can run the lesson example locally on `dotnet 8` and see the path to the saved file. (EN)

#### Ресурсы / Resources
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/tour-of-csharp/ — Tour of C# / Обзор языка C# (RU/EN)
- Microsoft Learn — https://learn.microsoft.com/dotnet/core/introduction — Introduction to .NET / Введение в .NET (RU/EN)
- Microsoft Learn — https://learn.microsoft.com/dotnet/standard/components — .NET architectural components / Архитектурные компоненты .NET (RU/EN)
- .NET GitHub — https://github.com/dotnet — исходный код платформы / platform source code (EN)

---
[← Предыдущий: README](../../README.md) | [⬆ К модулю M01](../README.md) | [Следующий: M01-L02 →](lesson-M01-L02-clr-il-jit.md)
---
