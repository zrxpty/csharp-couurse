[⬆ К модулю M10](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M10-L01: File/FileInfo/Directory/Path / File/FileInfo/Directory/Path

**Модуль / Module:** M10
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Работа с файловой системой в .NET — это фундамент, на котором строится почти любое реальное приложение: логирование, кэш, конфигурации, импорт/экспорт данных. В пространстве имён `System.IO` есть несколько ключевых типов, и важно понимать разницу между ними, чтобы выбирать правильный инструмент под задачу.

Начнём с `File` и `Directory`. Это **статические классы** — все их методы вызываются напрямую, без создания экземпляра. Они идеально подходят для **разовых операций**: прочитать файл целиком, записать строку, проверить существование, скопировать, удалить. Каждый вызов статического метода выполняет внутреннюю проверку прав и открывает/закрывает поток. Аналогия: `File` — это курьер-одиночка, которому вы дали одно поручение. Он приехал, забрал посылку, уехал. Если у вас десять посылок, вы вызываете курьера десять раз — каждый раз с накладными расходами.

С другой стороны — `FileInfo` и `DirectoryInfo`. Это **инстансные классы** (объекты), создаваемые через `new FileInfo(path)`. Они кэшируют информацию о файле (размер, атрибуты, время создания) и переиспользуют её между операциями. Аналогия: вы нанимаете курьера на полный рабочий день. Он помнит адрес, знает, где лежат документы, и выполняет серию задач без повторного оформления. Если нужно сделать несколько операций над одним файлом — `FileInfo` эффективнее.

**Когда что выбирать?** Одноразовое действие — `File`. Множество операций над одним путём — `FileInfo`. Это правилоthumb, но не догма: иногда статический вызов чище читается, даже если он «дороже».

Класс `Path` — это «чистая геометрия путей». Он **не обращается к диску**. `Path.Combine("folder", "file.txt")` склеивает части, корректно подставляя разделитель (`\` на Windows, `/` на Linux/macOS). `Path.GetFileName` извлекает имя, `Path.GetExtension` — расширение, `Path.GetTempPath` — путь к временной папке системы. Использовать ручную конкатенацию строк с `+` и хардкод `\\` — плохая практика: код сломается на другой ОС.

**Кроссплатформенность.** .NET работает на Windows, Linux и macOS. Разделители путей разные, но `Path` абстрагирует это. Главное правило: **никогда не хардкодьте `\\` в путях**. Используйте `Path.Combine` или `Path.DirectorySeparatorChar`. При работе с относительными путями помните, что `Path.GetFullPath` раскрывает их относительно `Directory.GetCurrentDirectory()`, а не относительно исполняемого файла — это частый источник багов.

**Временные файлы.** `Path.GetTempPath()` возвращает системную временную директорию (например, `/tmp` на Linux или `%TEMP%` на Windows). `Path.GetTempFileName()` создаёт уникальный пустой файл нулевой длины и возвращает его путь. Внимание: на Windows `GetTempFileName` генерирует имя из 8 символов и может исчерпать лимит при интенсивном использовании — в серверных сценариях лучше генерировать имя через `Guid` + `Path.Combine`.

**Concurrency-аспект.** Файловые операции не атомарны. Если несколько потоков пишут в один файл, нужен синхронизатор (например, `SemaphoreSlim` или `ReaderWriterLockSlim`), либо модель с одним писателем через `Channel<T>`. Чтение через `FileShare.Read` позволяет нескольким читателям, но блокирует писателей. Метод `File.ReadAllText` открывает файл с `FileShare.Read`, что безопасно для конкурирующего чтения, но не для записи. Для логов в многопоточной среде рассмотрите `FileStream` с `FileShare` и блокировкой, либо готовые решения вроде `Serilog`.

Помните: файловая система — медленный ресурс по сравнению с памятью. Дисковый I/O — это всегда потенциальное узкое место, поэтому в высоконагруженных сценариях используйте асинхронные методы (`ReadAllTextAsync`, `WriteAllTextAsync`) и буферизацию.

#### Theory (EN)

File-system operations in .NET are the foundation beneath almost every real application: logging, caching, configuration, data import/export. The `System.IO` namespace offers several key types, and understanding the distinction between them is essential for picking the right tool for the job.

Start with `File` and `Directory`. These are **static classes** — every method is called directly, no instance needed. They are ideal for **one-shot operations**: read a whole file, write a string, check existence, copy, delete. Each static call performs an internal permission check and opens/closes a stream. Analogy: `File` is a single-trip courier you dispatch with one errand. They arrive, pick up the parcel, leave. Ten parcels means ten couriers, each with overhead.

On the other side sit `FileInfo` and `DirectoryInfo`. These are **instance classes** (objects) created via `new FileInfo(path)`. They cache file information (size, attributes, creation time) and reuse it across operations. Analogy: you hire a courier for the whole workday. They remember the address, know where documents live, and execute a series of tasks without re-paperwork. When you need multiple operations on the same path, `FileInfo` is more efficient.

**When to choose which?** Single action — `File`. Many operations on one path — `FileInfo`. This is a rule of thumb, not dogma: sometimes a static call reads more cleanly even if it is marginally costlier.

The `Path` class is pure "path geometry." It **never touches the disk**. `Path.Combine("folder", "file.txt")` joins parts with the correct separator (`\` on Windows, `/` on Linux/macOS). `Path.GetFileName` extracts the name, `Path.GetExtension` the extension, `Path.GetTempPath` the system temp directory. Manual string concatenation with `+` and hardcoded `\\` is poor practice — the code breaks on another OS.

**Cross-platform.** .NET runs on Windows, Linux, and macOS. Path separators differ, but `Path` abstracts that away. The golden rule: **never hardcode `\\` in paths**. Use `Path.Combine` or `Path.DirectorySeparatorChar`. With relative paths, remember that `Path.GetFullPath` resolves them against `Directory.GetCurrentDirectory()`, not against the executable location — a frequent source of bugs.

**Temporary files.** `Path.GetTempPath()` returns the system temp directory (e.g., `/tmp` on Linux or `%TEMP%` on Windows). `Path.GetTempFileName()` creates a unique zero-length file and returns its path. Caveat: on Windows, `GetTempFileName` generates an 8-character name and can exhaust its limit under heavy use — in server scenarios prefer generating a name via `Guid` + `Path.Combine`.

**Concurrency angle.** File operations are not atomic. If multiple threads write to the same file, you need a synchronizer (e.g., `SemaphoreSlim` or `ReaderWriterLockSlim`) or a single-writer model through `Channel<T>`. Reading with `FileShare.Read` allows multiple readers but blocks writers. `File.ReadAllText` opens with `FileShare.Read`, safe for concurrent reads but not writes. For logs in multi-threaded contexts, consider a `FileStream` with proper `FileShare` and locking, or mature solutions like `Serilog`.

Remember: the file system is slow compared to memory. Disk I/O is always a potential bottleneck, so in high-load scenarios use asynchronous methods (`ReadAllTextAsync`, `WriteAllTextAsync`) and buffering.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8+ — File, FileInfo, Directory, Path examples
// Полный рабочий пример: создание, чтение, временные файлы, кроссплатформенные пути
// Full working example: creation, reading, temp files, cross-platform paths

using System.IO;

// --- 1. Path: чистая работа с путями без обращения к диску / pure path handling, no disk access ---
string folder = "data";
string fileName = "report.txt";
// Правильно: склеиваем через Path.Combine (кроссплатформенно) / Correct: join via Path.Combine (cross-platform)
string fullPath = Path.Combine(folder, fileName);
Console.WriteLine($"Полный путь / Full path: {Path.GetFullPath(fullPath)}");
Console.WriteLine($"Только имя / Name only: {Path.GetFileName(fullPath)}");
Console.WriteLine($"Расширение / Extension: {Path.GetExtension(fullPath)}");
Console.WriteLine($"Директория / Directory: {Path.GetDirectoryName(fullPath)}");

// --- 2. Directory: создание директорий / creating directories ---
if (!Directory.Exists(folder))
{
    Directory.CreateDirectory(folder); // Создаёт все промежуточные папки / Creates all intermediate folders
}

// --- 3. File: разовые операции / one-shot operations ---
string content = "Hello, System.IO!\nПривет, файловая система!";
// Асинхронная запись — рекомендуется для I/O / Async write — recommended for I/O
await File.WriteAllTextAsync(fullPath, content);

// Проверка существования / existence check
if (File.Exists(fullPath))
{
    // Чтение целиком / read all at once
    string readBack = await File.ReadAllTextAsync(fullPath);
    Console.WriteLine($"Прочитано / Read: {readBack.Length} символов / chars");
}

// --- 4. FileInfo: серия операций над одним файлом / multiple ops on one file ---
var fileInfo = new FileInfo(fullPath);
Console.WriteLine($"Размер / Size: {fileInfo.Length} байт / bytes");
Console.WriteLine($"Создан / Created: {fileInfo.CreationTime:O}");
Console.WriteLine($"Атрибуты / Attributes: {fileInfo.Attributes}");

// Копирование через FileInfo / copy via FileInfo
string copyPath = Path.Combine(folder, "report_copy.txt");
fileInfo.CopyTo(copyPath, overwrite: true); // Копируем с перезаписью / copy with overwrite

// --- 5. DirectoryInfo: перечисление содержимого / enumerate directory contents ---
var dirInfo = new DirectoryInfo(folder);
Console.WriteLine($"Файлов в папке / Files in folder: {dirInfo.GetFiles().Length}");
foreach (var f in dirInfo.GetFiles())
{
    Console.WriteLine($"  - {f.Name} ({f.Length} bytes)");
}

// --- 6. Временные файлы / temporary files ---
string tempDir = Path.GetTempPath(); // Системная temp-папка / system temp dir
string tempFile = Path.GetTempFileName(); // Создаёт уникальный пустой файл / creates unique empty file
Console.WriteLine($"Temp dir: {tempDir}");
Console.WriteLine($"Temp file: {tempFile}");

// Альтернатива для серверных сценариев: GUID-имя / alternative for servers: GUID name
string safeTempName = Path.Combine(tempDir, $"dsh-{Guid.NewGuid():N}.tmp");
await File.WriteAllTextAsync(safeTempName, "payload");

// --- 7. Очистка / cleanup ---
File.Delete(tempFile);
File.Delete(safeTempName);
fileInfo.Delete();
File.Delete(copyPath);
// Directory.Delete(folder, recursive: true); // Раскомментируйте для удаления папки / uncomment to delete folder

// --- 8. Concurrency: безопасное конкурентное чтение / safe concurrent reading ---
// File.ReadAllText использует FileShare.Read — несколько читателей, но нет писателей
// File.ReadAllText uses FileShare.Read — multiple readers, but no writers
// Для логгинга в многопоточной среде используйте FileStream с блокировкой или Serilog
// For logging in multi-threaded env, use FileStream with locking or Serilog

Console.WriteLine("Готово / Done.");
```

#### Best Practices

- **RU:** Используйте `Path.Combine` вместо ручной конкатенации — это гарантирует кроссплатформенность и читаемость.
- **RU:** Для одноразовых операций берите статические `File`/`Directory`, для серии операций над одним путём — `FileInfo`/`DirectoryInfo`.
- **RU:** Предпочитайте асинхронные методы (`*Async`) для I/O, чтобы не блокировать поток.
- **RU:** Всегда закрывайте/утилизируйте `FileStream` через `using` или `await using`.
- **RU:** Не хардкодьте пути и разделители — используйте `Path.Combine` и `Path.DirectorySeparatorChar`.
- **EN:** Use `Path.Combine` over manual concatenation — it guarantees cross-platform correctness and readability.
- **EN:** For single operations use static `File`/`Directory`; for many operations on one path use `FileInfo`/`DirectoryInfo`.
- **EN:** Prefer async methods (`*Async`) for I/O so you don't block the thread.
- **EN:** Always dispose `FileStream` via `using` or `await using`.
- **EN:** Never hardcode paths or separators — use `Path.Combine` and `Path.DirectorySeparatorChar`.

#### Частые ошибки / Common Mistakes

- **RU:** Хардкод `\\` в путях → Используйте `Path.Combine` или `Path.DirectorySeparatorChar` — код будет работать на Linux/macOS.
- **RU:** Открытие `FileStream` без `using` → Утечка дескрипторов; оборачивайте в `using` или `await using`.
- **RU:** Синхронный `File.ReadAllText` в горячем пути → Используйте `ReadAllTextAsync`, чтобы не блокировать поток.
- **RU:** Ожидание, что `Path.GetFullPath` раскрывает путь относительно .exe → Он раскрывает относительно `Directory.GetCurrentDirectory()`.
- **RU:** Параллельная запись нескольких потоков в один файл без блокировки → Используйте `SemaphoreSlim`, `Channel<T>` или один писатель.
- **RU:** `Path.GetTempFileName()` в плотном цикле → Лимит имён на Windows; генерируйте имя через `Guid`.
- **EN:** Hardcoding `\\` in paths → Use `Path.Combine` or `Path.DirectorySeparatorChar` so the code runs on Linux/macOS.
- **EN:** Opening a `FileStream` without `using` → Handle leaks; wrap in `using` or `await using`.
- **EN:** Synchronous `File.ReadAllText` on a hot path → Use `ReadAllTextAsync` to avoid blocking the thread.
- **EN:** Expecting `Path.GetFullPath` to resolve relative to the .exe → It resolves against `Directory.GetCurrentDirectory()`.
- **EN:** Multiple threads writing to one file without locking → Use `SemaphoreSlim`, `Channel<T>`, or a single writer.
- **EN:** `Path.GetTempFileName()` in a tight loop → Name limit on Windows; generate names via `Guid`.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] RU: Я использую `Path.Combine` для склейки путей, а не `+` и хардкод разделителей.
- [ ] RU: Я выбираю `File` для одной операции и `FileInfo` для серии операций.
- [ ] RU: Я использую асинхронные методы (`*Async`) для файлового I/O.
- [ ] RU: Я оборачиваю `FileStream`/`StreamReader` в `using` или `await using`.
- [ ] RU: Я понимаю, что `Path` не обращается к диску.
- [ ] RU: Я знаю про лимит `GetTempFileName()` и альтернативу с `Guid`.
- [ ] RU: Я учитываю кроссплатформенность (Windows `\` vs Linux `/`).
- [ ] RU: Я синхронизирую конкурентную запись в файлы.
- [ ] EN: I use `Path.Combine` to join paths, not `+` and hardcoded separators.
- [ ] EN: I pick `File` for one operation and `FileInfo` for a series.
- [ ] EN: I use async methods (`*Async`) for file I/O.
- [ ] EN: I wrap `FileStream`/`StreamReader` in `using` or `await using`.
- [ ] EN: I understand that `Path` never touches the disk.
- [ ] EN: I know the `GetTempFileName()` limit and the `Guid` alternative.
- [ ] EN: I account for cross-platform paths (Windows `\` vs Linux `/`).
- [ ] EN: I synchronize concurrent writes to files.

#### Ресурсы / Resources

- [Microsoft Learn — File](https://learn.microsoft.com/dotnet/api/system.io.file)
- [Microsoft Learn — FileInfo](https://learn.microsoft.com/dotnet/api/system.io.fileinfo)
- [Microsoft Learn — Directory](https://learn.microsoft.com/dotnet/api/system.io.directory)
- [Microsoft Learn — DirectoryInfo](https://learn.microsoft.com/dotnet/api/system.io.directoryinfo)
- [Microsoft Learn — Path](https://learn.microsoft.com/dotnet/api/system.io.path)

---

[⬆ К модулю M10](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
