# Quiz модуля M01 / Quiz: M01
# Введение в C# и .NET / Introduction to C# and .NET

> 12 вопросов / 12 questions
> Вопросы на понимание, не запоминание / Understanding, not memorization
> Код примеров: C# 12 / .NET 8 / Code samples: C# 12 / .NET 8

---

## Вопрос 1 (Multiple Choice / Множественный выбор)

**Вопрос / Question:**
Что верно о соотношении C# и .NET?
What is true about the relationship between C# and .NET?

**Варианты / Options:**
- A) C# и .NET — это одно и то же, просто два названия одной технологии. / C# and .NET are the same thing, just two names for one technology.
- B) C# — это язык программирования, а .NET — платформа (CLR + BCL + runtime), на которой этот язык исполняется; на .NET можно писать и на F#, VB.NET. / C# is a programming language, while .NET is the platform (CLR + BCL + runtime) that executes it; .NET also supports F# and VB.NET.
- C) .NET — это библиотека классов для C#, а C# включает компилятор. / .NET is a class library for C#, while C# includes the compiler.
- D) C# компилируется напрямую в машинный код процессора, а .NET лишь редактор кода. / C# compiles directly to CPU machine code, and .NET is just a code editor.

**Правильный ответ / Correct:** B

**Объяснение / Explanation:**
(RU) B верно: C# — язык, .NET — платформа (движок + BCL + CLR), и IL позволяет языкам C#, F#, VB.NET взаимодействовать. A неверно — это разные сущности. C неверно — .NET это не только библиотека, а среда выполнения. D неверно — C# компилируется в IL, а не в машинный код; машинный код создаёт JIT.
(EN) B is correct: C# is a language, .NET is the platform (runtime + BCL + CLR), and shared IL lets C#, F#, and VB.NET interoperate. A is wrong — they are distinct entities. C is wrong — .NET is a runtime, not just a library. D is wrong — C# compiles to IL, not machine code; the JIT produces native code.

**Сложность / Difficulty:** 1/5

**Тип / Type:** Understanding

---

## Вопрос 2 (Multiple Choice / Множественный выбор)

**Вопрос / Question:**
Что выведет следующий код на C# 12 / .NET 8?
What does the following C# 12 / .NET 8 code print?

```csharp
object maybe = Random.Shared.Next(2) == 0 ? "hello" : 42;
string description = maybe switch
{
    string s => $"String: {s}",
    int i    => $"Number: {i}",
    _        => "Unknown"
};
Console.WriteLine(description);
```

**Варианты / Options:**
- A) Всегда `Unknown` / Always `Unknown`
- B) Либо `String: hello`, либо `Number: 42` в зависимости от случайного значения / Either `String: hello` or `Number: 42` depending on the random value
- C) Ошибка компиляции: нельзя сопоставлять `object` с `string` и `int` / Compile error: cannot match `object` against `string` and `int`
- D) Всегда `Number: 42` / Always `Number: 42`

**Правильный ответ / Correct:** B

**Объяснение / Explanation:**
(RU) B верно: pattern matching в `switch` безопасно проверяет тип времени выполнения благодаря type safety CLR; `maybe` — это либо строка, либо число. A неверно — `_` срабатывает только при отсутствии совпадения. C неверно — pattern matching по типу допустим. D неверно — значение случайно.
(EN) B is correct: pattern matching in `switch` safely checks the runtime type thanks to CLR type safety; `maybe` is either a string or an int. A is wrong — `_` only fires when nothing matches. C is wrong — type patterns are legal. D is wrong — the value is random.

**Сложность / Difficulty:** 2/5

**Тип / Type:** Understanding

---

## Вопрос 3 (Multiple Choice / Множественный выбор)

**Вопрос / Question:**
Какой путь компиляции C#-кода в .NET является правильным?
Which compilation path of C# code in .NET is correct?

**Варианты / Options:**
- A) `C# → машинный код → IL` (один шаг, как в C) / `C# → machine code → IL` (one step, like C)
- B) `C# → IL (в сборке .dll/.exe)` на этапе сборки, затем `IL → машинный код` через JIT при первом вызове метода / `C# → IL (inside .dll/.exe assembly)` at build time, then `IL → machine code` via JIT on first method call
- C) `C# → интерпретируется построчно CLR` без промежуточного кода / `C# → interpreted line-by-line by the CLR`, no intermediate code
- D) `C# → IL → AOT-компиляция всегда при сборке` / `C# → IL → always AOT-compiled at build`

**Правильный ответ / Correct:** B

**Объяснение / Explanation:**
(RU) B верно: компилятор `csc` переводит C# в IL внутри сборки; JIT превращает IL в нативный код при первом вызове метода и кэширует результат. A неверно — нет прямого шага C#→машинный код в классической модели. C неверно — CLR не интерпретатор; JIT компилирует целые методы. D неверно — Native AOT опционален, JIT остаётся основным путём.
(EN) B is correct: `csc` translates C# into IL inside an assembly; the JIT turns IL into native code on first method call and caches it. A is wrong — there is no direct C#→machine-code step in the classic model. C is wrong — the CLR is not an interpreter; the JIT compiles whole methods. D is wrong — Native AOT is optional; JIT is the default path.

**Сложность / Difficulty:** 2/5

**Тип / Type:** Understanding

---

## Вопрос 4 (Matching / Сопоставление понятий)

**Вопрос / Question:**
Сопоставьте каждый компонент .NET с его основной обязанностью.
Match each .NET component to its primary responsibility.

**Варианты / Options:**
- A) CLR → 1) Промежуточный байткод, общий для всех .NET-языков / Universal bytecode shared by all .NET languages
- B) IL  → 2) Среда выполнения: загрузка сборок, JIT, GC, type safety / Runtime: assembly loading, JIT, GC, type safety
- C) JIT → 3) Переводит IL метода в нативный код при первом вызове / Translates a method's IL into native code on first call
- D) Сборка (Assembly) → 4) Файл `.dll`/`.exe` с IL + метаданными + манифестом / A `.dll`/`.exe` file with IL + metadata + manifest

**Правильный ответ / Correct:** A-2, B-1, C-3, D-4

**Объяснение / Explanation:**
(RU) CLR — среда выполнения (2). IL — промежуточный язык-«эсперанто» (1). JIT компилирует методы в нативный код (3). Сборка — контейнер с IL, метаданными и манифестом (4). Иные комбинации нарушают определения из урока M01-L02.
(EN) The CLR is the runtime (2). IL is the universal intermediate "Esperanto" (1). The JIT compiles methods to native code (3). An assembly is the container with IL, metadata, and a manifest (4). Any other pairing contradicts the definitions from lesson M01-L02.

**Сложность / Difficulty:** 3/5

**Тип / Type:** Understanding

---

## Вопрос 5 (Multiple Choice / Множественный выбор)

**Вопрос / Question:**
Вы установили только .NET Runtime на машину разработки и пытаетесь выполнить `dotnet build`. Что произойдёт?
You installed only the .NET Runtime on a development machine and run `dotnet build`. What happens?

**Варианты / Options:**
- A) Сборка пройдёт успешно — Runtime содержит компилятор Roslyn. / The build succeeds — the Runtime includes the Roslyn compiler.
- B) `dotnet build` не сработает корректно: для разработки нужен SDK (Runtime + компилятор + CLI + шаблоны). Runtime только запускает готовые приложения. / `dotnet build` will not work correctly: development requires the SDK (Runtime + compiler + CLI + templates). The Runtime only runs finished applications.
- C) Программа соберётся, но не запустится. / The program builds but does not run.
- D) Runtime автоматически докачает SDK из интернета. / The Runtime automatically downloads the SDK from the internet.

**Правильный ответ / Correct:** B

**Объяснение / Explanation:**
(RU) B верно: SDK = Runtime + Roslyn + инструменты сборки + шаблоны + dotnet CLI; на машине разработчика ставят именно SDK, Runtime ставят на сервер/у клиента. A неверно — Roslyn входит в SDK, а не в Runtime. C неверно — без компилятора сборка невозможна. D неверно — авто-докачки SDK не существует.
(EN) B is correct: the SDK = Runtime + Roslyn + build tools + templates + the dotnet CLI; install the SDK for development and the Runtime only on servers/end-user machines. A is wrong — Roslyn ships with the SDK, not the Runtime. C is wrong — without a compiler there is no build. D is wrong — there is no automatic SDK download.

**Сложность / Difficulty:** 2/5

**Тип / Type:** Best practices

---

## Вопрос 6 (Multiple Choice / Множественный выбор)

**Вопрос / Question:**
Какая команда создаёт новый консольный проект `HelloApp` в одноимённой папке на .NET 8?
Which command creates a new console project `HelloApp` in a folder of the same name on .NET 8?

**Варианты / Options:**
- A) `dotnet new console HelloApp`
- B) `dotnet create console -n HelloApp`
- C) `dotnet new console -n HelloApp`
- D) `dotnet init HelloApp --type console`

**Правильный ответ / Correct:** C

**Объяснение / Explanation:**
(RU) C верно: `dotnet new <шаблон> -n <Имя>` — стандартный синтаксис; флаг `-n` задаёт имя проекта и папки. A неверно — имя не позиционный аргумент. B неверно — нет команды `dotnet create`. D неверно — нет команды `dotnet init`.
(EN) C is correct: `dotnet new <template> -n <Name>` is the standard syntax; `-n` sets both project and folder name. A is wrong — the name is not a positional argument. B is wrong — there is no `dotnet create`. D is wrong — there is no `dotnet init`.

**Сложность / Difficulty:** 1/5

**Тип / Type:** Understanding

---

## Вопрос 7 (Code Completion / Допиши код)

**Вопрос / Question:**
Допишите минимальную точку входа на C# 12 (.NET 8) с top-level statements, которая печатает `Hello, .NET 8!` и возвращает код выхода `0`.
Complete the minimal C# 12 (.NET 8) entry point using top-level statements that prints `Hello, .NET 8!` and returns exit code `0`.

```csharp
// Program.cs
________
```

**Варианты / Options:**
- A)
```csharp
using System;

class Program
{
    static int Main()
    {
        Console.WriteLine("Hello, .NET 8!");
        return 0;
    }
}
```
- B)
```csharp
Console.WriteLine("Hello, .NET 8!");
return 0;
```
- C)
```csharp
static void Main() => Console.WriteLine("Hello, .NET 8!");
```
- D)
```csharp
print("Hello, .NET 8!");
exit(0);
```

**Правильный ответ / Correct:** B

**Объяснение / Explanation:**
(RU) B верно: top-level statements позволяют писать инструкции напрямую без класса и `Main`; `return` задаёт код выхода процесса. A — классическая форма, валидна, но не является top-level statements (тема вопроса). C неверно — не top-level и `void` не возвращает код. D неверно — таких функций в C# нет.
(EN) B is correct: top-level statements let you write code directly without a class or `Main`; `return` sets the process exit code. A is the classic form — valid but not top-level statements (the question's topic). C is wrong — not top-level, and `void` returns no code. D is wrong — these functions do not exist in C#.

**Сложность / Difficulty:** 2/5

**Тип / Type:** Code completion

---

## Вопрос 8 (Debug / Найди баг в коде)

**Вопрос / Question:**
В проекте `HelloApp` на .NET 8 компилятор выдаёт ошибку `CS7022`. Найдите проблему в структуре проекта.
In a .NET 8 project `HelloApp` the compiler emits error `CS7022`. Find the problem in the project structure.

```csharp
// File: Program.cs
Console.WriteLine("Hello from Program.cs");

// File: Startup.cs
Console.WriteLine("Hello from Startup.cs");
```

**Варианты / Options:**
- A) Оба файла используют `Console.WriteLine` без `using System;` — добавьте using в каждый. / Both files use `Console.WriteLine` without `using System;` — add the using to each.
- B) В одном проекте может быть только один файл с top-level statements; здесь их два — точка входа неоднозначна. Удалите top-level код из `Startup.cs` или оберните его в класс/метод. / A project may contain at most one file with top-level statements; here there are two, making the entry point ambiguous. Remove top-level code from `Startup.cs` or wrap it in a class/method.
- C) Имена файлов должны совпадать: переименуйте `Startup.cs` в `Program2.cs`. / File names must match: rename `Startup.cs` to `Program2.cs`.
- D) Не указан `<OutputType>Exe</OutputType>` в `.csproj`. / `<OutputType>Exe</OutputType>` is missing from `.csproj`.

**Правильный ответ / Correct:** B

**Объяснение / Explanation:**
(RU) B верно: `CS7022` означает несколько точек входа; top-level statements допустимы только в одном файле проекта. A неверно — `System` входит в implicit usings. C неверно — имя файла не причина. D неверно — `OutputType` не связан с `CS7022`.
(EN) B is correct: `CS7022` means multiple entry points; top-level statements are allowed in only one file per project. A is wrong — `System` is covered by implicit usings. C is wrong — the file name is not the cause. D is wrong — `OutputType` is unrelated to `CS7022`.

**Сложность / Difficulty:** 3/5

**Тип / Type:** Debug

---

## Вопрос 9 (Multiple Choice / Множественный выбор)

**Вопрос / Question:**
Какой артефакт в `bin/Debug/net8.0/` содержит отладочные символы (номера строк в стеке вызовов)?
Which artifact in `bin/Debug/net8.0/` holds debug symbols (line numbers in stack traces)?

**Варианты / Options:**
- A) `MyApp.dll`
- B) `MyApp.deps.json`
- C) `MyApp.runtimeconfig.json`
- D) `MyApp.pdb`

**Правильный ответ / Correct:** D

**Объяснение / Explanation:**
(RU) D верно: `.pdb` (Program Database) — файл отладочных символов; без него стек вызовов не содержит номеров строк. A — основная сборка с IL. B — граф зависимостей для загрузчика. C — конфигурация runtime и версии .NET.
(EN) D is correct: `.pdb` (Program Database) is the debug-symbol file; without it stack traces lack line numbers. A is the main IL assembly. B is the dependency graph for the loader. C is the runtime/.NET-version configuration.

**Сложность / Difficulty:** 2/5

**Тип / Type:** Understanding

---

## Вопрос 10 (Best practices / Как лучше сделать?)

**Вопрос / Question:**
Команда из 5 человек жалуется на «плавающие» сборки: у одного разработчика проект собирается, у другого падает из-за версии SDK. Какое лучшее решение?
A 5-person team reports "flaky" builds: one developer's machine builds fine, another fails due to an SDK version. What is the best solution?

**Варианты / Options:**
- A) Заставить всех установить самую новую версию SDK вручную. / Make everyone install the latest SDK manually.
- B) Зафиксировать версию SDK в `global.json` в корне решения, чтобы все и CI использовали одинаковый SDK. / Pin the SDK version in a `global.json` at the solution root so every teammate and CI uses the same SDK.
- C) Удалить `obj/` и `bin/` у всех. / Delete `obj/` and `bin/` for everyone.
- D) Перейти с .NET 8 на .NET Framework 4.8. / Switch from .NET 8 to .NET Framework 4.8.

**Правильный ответ / Correct:** B

**Объяснение / Explanation:**
(RU) B верно: `global.json` фиксирует версию SDK и поле `rollForward`, обеспечивая воспроизводимые сборки у всех и в CI. A — ручная установка нестабильна. C не решает корень проблемы. D — шаг назад к Windows-only платформе в режиме поддержки.
(EN) B is correct: `global.json` pins the SDK version and `rollForward` policy, giving reproducible builds across machines and CI. A is unreliable. C does not address the root cause. D is a regression to a Windows-only, maintenance-only platform.

**Сложность / Difficulty:** 3/5

**Тип / Type:** Best practices

---

## Вопрос 11 (Multiple Choice / Множественный выбор)

**Вопрос / Question:**
Что выведет программа при запуске `dotnet run -- Alice --count 3` (по логике урока M01-L06)?
What does the program output when run as `dotnet run -- Alice --count 3` (per lesson M01-L06 logic)?

**Варианты / Options:**
- A) Одно сообщение `Привет, Alice!` / A single `Привет, Alice!` message
- B) Три сообщения с нумерацией `[1]`, `[2]`, `[3]`, затем итог / Three numbered messages `[1]`, `[2]`, `[3]`, then a summary
- C) `IndexOutOfRangeException` / `IndexOutOfRangeException`
- D) Ничего, потому что `args` равен `null` без аргументов / Nothing, because `args` is `null` without arguments

**Правильный ответ / Correct:** B

**Объяснение / Explanation:**
(RU) B верно: `--count 3` повторяет вывод 3 раза с нумерацией, затем печатается итог через raw string literal. A неверно — `count` по умолчанию 1, но флаг задаёт 3. C неверно — код проверяет `args.Length` и диапазон 1–100. D неверно — при отсутствии аргументов `args` — пустой массив, а не `null`.
(EN) B is correct: `--count 3` repeats the output 3 times with numbering, then a summary is printed via a raw string literal. A is wrong — the default `count` is 1 but the flag sets 3. C is wrong — the code checks `args.Length` and the 1–100 range. D is wrong — with no arguments `args` is an empty array, not `null`.

**Сложность / Difficulty:** 3/5

**Тип / Type:** Understanding

---

## Вопрос 12 (Debug / Найди баг в коде)

**Вопрос / Question:**
Разработчик пытается опубликовать self-contained приложение одной командой и получает ошибку. Найдите проблему.
A developer tries to publish a self-contained app with a single command and gets an error. Find the problem.

```bash
dotnet publish -c Release --self-contained -p:PublishSingleFile=true
```

**Варианты / Options:**
- A) Флаг `-p:PublishSingleFile=true` нельзя использовать вместе с `--self-contained`. / The `-p:PublishSingleFile=true` flag cannot be combined with `--self-contained`.
- B) Для `--self-contained` обязательно указать Runtime Identifier (RID), например `-r win-x64` или `-r linux-x64`, потому что runtime платформо-зависим. / `--self-contained` requires a Runtime Identifier (RID), e.g. `-r win-x64` or `-r linux-x64`, because the runtime is platform-specific.
- C) Нужно сначала выполнить `dotnet build`, иначе publish не сработает. / You must run `dotnet build` first, otherwise publish fails.
- D) Конфигурация `-c Release` недопустима для `publish`. / The `-c Release` configuration is invalid for `publish`.

**Правильный ответ / Correct:** B

**Объяснение / Explanation:**
(RU) B верно: self-contained публикация включает платформо-зависимый runtime, поэтому требует RID. A неверно — комбинация разрешена и часто используется вместе. C неверно — `publish` сам вызывает сборку. D неверно — `Release` допустим и рекомендуется для публикации.
(EN) B is correct: self-contained publishing bundles a platform-specific runtime, so it requires a RID. A is wrong — the combination is allowed and commonly used. C is wrong — `publish` invokes the build itself. D is wrong — `Release` is allowed and recommended for publishing.

**Сложность / Difficulty:** 3/5

**Тип / Type:** Debug

---

## Ключ ответов / Answer key

| № | Ответ / Answer | Сложность / Difficulty | Тип / Type |
|---|----------------|------------------------|------------|
| 1  | B              | 1/5 | Understanding      |
| 2  | B              | 2/5 | Understanding      |
| 3  | B              | 2/5 | Understanding      |
| 4  | A-2, B-1, C-3, D-4 | 3/5 | Matching       |
| 5  | B              | 2/5 | Best practices     |
| 6  | C              | 1/5 | Understanding      |
| 7  | B              | 2/5 | Code completion    |
| 8  | B              | 3/5 | Debug              |
| 9  | D              | 2/5 | Understanding      |
| 10 | B              | 3/5 | Best practices     |
| 11 | B              | 3/5 | Understanding      |
| 12 | B              | 3/5 | Debug              |

---

## Покрытие тем / Topic coverage

| Тема / Topic | Урок / Lesson | Вопросы / Questions |
|--------------|---------------|---------------------|
| Что такое C#/.NET, managed code, BCL | M01-L01 | 1, 2 |
| CLR / IL / JIT / сборки | M01-L02 | 3, 4, 9 |
| Установка SDK / IDE / CLI, global.json | M01-L03 | 5, 10 |
| dotnet new / top-level statements / структура проекта | M01-L04 | 6, 7, 8 |
| dotnet build / run / Debug-Release / publish | M01-L05 | 9, 12 |
| Hello World / args / отладка / NuGet | M01-L06 | 11 |
