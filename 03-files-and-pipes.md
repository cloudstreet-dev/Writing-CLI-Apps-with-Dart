# Chapter 3: Files and Pipes

**Being a good Unix citizen**

There's a philosophy in Unix-land that goes like this: write programs that do one thing well, and write programs that work together. The magic happens when you chain simple tools together with pipes:

```bash
$ cat access.log | grep "ERROR" | wc -l
42
```

Three simple tools, combined to answer "how many errors are in my log file?" This is the Unix way, and it's beautiful.

But here's the thing: for your tool to play nicely in this ecosystem, you need to respect the rules. You need to understand stdin, stdout, stderr, and how to handle files without being a jerk about it.

Let's learn how.

## The Three Streams

Every process has three standard streams:

- **stdin** (file descriptor 0): Input *to* your program
- **stdout** (file descriptor 1): Normal output *from* your program
- **stderr** (file descriptor 2): Error messages and diagnostics

In Dart, these are available from `dart:io`:

```dart
import 'dart:io';

void main() {
  // Read from stdin
  String? line = stdin.readLineSync();

  // Write to stdout
  stdout.writeln('Normal output');

  // Write to stderr
  stderr.writeln('Error output');
}
```

### Why Three Streams?

Imagine you have a tool that processes files and reports errors:

```bash
$ mytool input.txt > output.txt
Error: Line 42 is malformed
```

Because errors go to **stderr**, they appear on your terminal even when stdout is redirected to a file. This way, users can:

```bash
# Save output to a file, see errors on screen
$ mytool input.txt > output.txt 2> errors.txt

# Discard errors (sometimes useful)
$ mytool input.txt > output.txt 2>/dev/null

# Combine both streams
$ mytool input.txt > combined.txt 2>&1
```

**Rule #1**: Normal output goes to stdout. Errors and diagnostics go to stderr. Never mix them.

## Reading from stdin

Many Unix tools read from stdin when no file is specified:

```bash
$ cat file.txt          # Read from file
$ echo "hello" | cat    # Read from stdin
```

Let's make our tool do the same:

```dart
// bin/wordcount.dart
import 'dart:io';

void main(List<String> arguments) async {
  Stream<String> lines;

  if (arguments.isEmpty) {
    // No arguments: read from stdin
    lines = stdin
        .transform(systemEncoding.decoder)
        .transform(const LineSplitter());
  } else {
    // Read from file
    final file = File(arguments.first);
    if (!await file.exists()) {
      stderr.writeln('Error: File not found: ${arguments.first}');
      exit(1);
    }
    lines = file
        .openRead()
        .transform(systemEncoding.decoder)
        .transform(const LineSplitter());
  }

  // Count lines, words, and characters
  int lineCount = 0;
  int wordCount = 0;
  int charCount = 0;

  await for (final line in lines) {
    lineCount++;
    wordCount += line.split(RegExp(r'\s+')).where((w) => w.isNotEmpty).length;
    charCount += line.length;
  }

  stdout.writeln('$lineCount $wordCount $charCount');
}
```

Now it works both ways:

```bash
# From a file
$ dart run bin/wordcount.dart myfile.txt
42 312 1847

# From stdin
$ echo "hello world" | dart run bin/wordcount.dart
1 2 11
```

### Understanding Streams

Notice we're using `Stream<String>` and `await for`. This is important!

**Bad approach** (reads entire file into memory):
```dart
// ❌ Don't do this for large files
final content = await file.readAsString();
final lines = content.split('\n');
```

**Good approach** (streams line by line):
```dart
// ✅ Memory efficient for large files
await for (final line in lines) {
  processLine(line);
}
```

Streams are lazy — they only read data as you consume it. This means your tool can process gigabyte files without using gigabytes of RAM.

## Writing to stdout and stderr

We've seen this already, but let's be explicit:

```dart
// Normal output
stdout.writeln('Result: 42');
print('Result: 42');  // Same thing

// Errors and warnings
stderr.writeln('Warning: Something seems off');
stderr.writeln('Error: Something went wrong');

// Progress messages (debatable, but stderr is common)
stderr.write('Processing... ');
stderr.writeln('done');
```

### The `print()` Shortcut

`print()` is just `stdout.writeln()` with less typing:

```dart
print('hello');           // Same as:
stdout.writeln('hello');  // This
```

For most cases, `print()` is fine. Use explicit `stdout.writeln()` when you want to be clear about where output goes.

## Buffering: A Hidden Gotcha

Here's a fun bug:

```dart
void main() async {
  stderr.write('Processing');
  await Future.delayed(Duration(seconds: 2));
  stderr.write('.');
  await Future.delayed(Duration(seconds: 2));
  stderr.write('.');
  await Future.delayed(Duration(seconds: 2));
  stderr.writeln(' done');
}
```

You'd expect to see:
```
Processing...(2 sec)...(2 sec)...(2 sec) done
```

But you might see nothing for 6 seconds, then:
```
Processing... done
```

Why? **Buffering**. By default, output is buffered and only flushed when:
1. The buffer is full
2. A newline is written
3. The program exits
4. You explicitly flush

Fix it:
```dart
void main() async {
  stderr.write('Processing');
  await stderr.flush();  // Force it to appear now

  await Future.delayed(Duration(seconds: 2));
  stderr.write('.');
  await stderr.flush();

  await Future.delayed(Duration(seconds: 2));
  stderr.write('.');
  await stderr.flush();

  await Future.delayed(Duration(seconds: 2));
  stderr.writeln(' done');
}
```

Now it works as expected!

## Reading Files the Right Way

Dart gives you several ways to read files. Choose wisely:

### Small Files: Read All at Once

```dart
final file = File('config.json');
final content = await file.readAsString();
```

Simple and fine for small files (< 1 MB). Don't use this for log files or large datasets.

### Large Files: Stream Line by Line

```dart
final file = File('large.log');
final lines = file
    .openRead()
    .transform(systemEncoding.decoder)
    .transform(const LineSplitter());

await for (final line in lines) {
  if (line.contains('ERROR')) {
    print(line);
  }
}
```

Memory efficient. Use this for files of unknown size.

### Binary Files: Read as Bytes

```dart
final file = File('image.png');
final bytes = await file.readAsBytes();
```

Returns `List<int>` (bytes). Use for non-text files.

### Streaming Binary Data

```dart
final file = File('large_file.bin');
final stream = file.openRead();

await for (final chunk in stream) {
  // chunk is List<int>
  processChunk(chunk);
}
```

Reads in chunks (typically 64KB). Very memory efficient.

## Writing Files the Right Way

### Small Content: Write All at Once

```dart
await File('output.txt').writeAsString('Hello, World!\n');
```

Simple and safe. Atomically replaces the file.

### Large Content: Stream It

```dart
final output = File('output.txt').openWrite();

try {
  output.writeln('Line 1');
  output.writeln('Line 2');
  output.writeln('Line 3');
} finally {
  await output.close();  // Always close!
}
```

Or use the cleaner pattern:

```dart
final sink = File('output.txt').openWrite();
await sink.addStream(inputStream);
await sink.close();
```

### Appending to Files

```dart
final output = File('log.txt').openWrite(mode: FileMode.append);
output.writeln('New log entry');
await output.close();
```

### Binary Files

```dart
final bytes = <int>[0xFF, 0xD8, 0xFF, 0xE0];
await File('output.bin').writeAsBytes(bytes);
```

## Building a Proper Pipe-Friendly Tool

Let's build `grep-lite` — a simplified version of grep that demonstrates all these concepts:

```dart
// bin/grep_lite.dart
import 'dart:io';
import 'dart:convert';
import 'package:args/args.dart';

Future<void> main(List<String> arguments) async {
  final parser = ArgParser()
    ..addFlag('ignore-case', abbr: 'i', help: 'Case-insensitive search')
    ..addFlag('invert-match', abbr: 'v', help: 'Invert match')
    ..addFlag('count', abbr: 'c', help: 'Only show count of matches')
    ..addFlag('line-number', abbr: 'n', help: 'Show line numbers')
    ..addFlag('help', abbr: 'h', negatable: false);

  ArgResults results;
  try {
    results = parser.parse(arguments);
  } on FormatException catch (e) {
    stderr.writeln('Error: ${e.message}');
    _printUsage(parser);
    exit(1);
  }

  if (results['help'] as bool) {
    _printUsage(parser);
    exit(0);
  }

  // Get the pattern and optional file
  final rest = results.rest;
  if (rest.isEmpty) {
    stderr.writeln('Error: Missing pattern');
    _printUsage(parser);
    exit(1);
  }

  final pattern = rest.first;
  final filePath = rest.length > 1 ? rest[1] : null;

  // Extract flags
  final ignoreCase = results['ignore-case'] as bool;
  final invertMatch = results['invert-match'] as bool;
  final showCount = results['count'] as bool;
  final showLineNumbers = results['line-number'] as bool;

  // Create regex pattern
  final regex = RegExp(
    pattern,
    caseSensitive: !ignoreCase,
  );

  // Get input stream
  final Stream<String> lines;
  if (filePath == null) {
    // Read from stdin
    lines = stdin.transform(utf8.decoder).transform(const LineSplitter());
  } else {
    // Read from file
    final file = File(filePath);
    if (!await file.exists()) {
      stderr.writeln('Error: File not found: $filePath');
      exit(1);
    }
    lines = file
        .openRead()
        .transform(utf8.decoder)
        .transform(const LineSplitter());
  }

  // Process lines
  int matchCount = 0;
  int lineNumber = 0;

  await for (final line in lines) {
    lineNumber++;
    final matches = regex.hasMatch(line);
    final shouldPrint = invertMatch ? !matches : matches;

    if (shouldPrint) {
      matchCount++;
      if (!showCount) {
        if (showLineNumbers) {
          stdout.writeln('$lineNumber:$line');
        } else {
          stdout.writeln(line);
        }
      }
    }
  }

  if (showCount) {
    stdout.writeln(matchCount);
  }

  // Exit with appropriate code
  exit(matchCount > 0 ? 0 : 1);
}

void _printUsage(ArgParser parser) {
  print('Usage: grep_lite [options] <pattern> [file]');
  print('');
  print('Search for a pattern in a file or stdin');
  print('');
  print('Options:');
  print(parser.usage);
  print('');
  print('Examples:');
  print('  grep_lite "error" logfile.txt');
  print('  cat logfile.txt | grep_lite -i "warning"');
  print('  grep_lite -c "TODO" main.dart');
}
```

This tool:
- ✅ Reads from stdin if no file specified
- ✅ Writes matches to stdout
- ✅ Writes errors to stderr
- ✅ Uses streams for memory efficiency
- ✅ Exits with proper codes (0 if found, 1 if not)
- ✅ Plays nicely with pipes

Usage:

```bash
# Search in a file
$ dart run bin/grep_lite.dart "ERROR" app.log

# Search from stdin
$ cat app.log | dart run bin/grep_lite.dart "ERROR"

# Pipe through multiple tools
$ cat app.log | dart run bin/grep_lite.dart "ERROR" | wc -l

# Count matches
$ dart run bin/grep_lite.dart -c "TODO" main.dart
7

# Case-insensitive with line numbers
$ dart run bin/grep_lite.dart -in "warning" app.log
42:WARNING: Low disk space
103:Warning: Connection timeout
```

## Exit Codes: The Unsung Heroes

Exit codes tell the shell (and scripts) whether your program succeeded:

```dart
import 'dart:io';

void main() {
  // Success
  exit(0);

  // Generic error
  exit(1);

  // Specific error codes (convention varies)
  exit(2);   // Misuse of command
  exit(127); // Command not found
  exit(130); // Terminated by Ctrl+C
}
```

Scripts can check these:

```bash
#!/bin/bash
if mytool input.txt; then
  echo "Success!"
else
  echo "Failed with code $?"
fi
```

**Best practices**:
- Exit 0 on success
- Exit 1 for generic errors
- Exit with specific codes for different error types (optional but helpful)
- Document your exit codes if you use custom ones

## Text Encoding: Don't Assume UTF-8

Most files are UTF-8, but not all:

```dart
import 'dart:io';
import 'dart:convert';

// UTF-8 (most common)
final utf8Content = await file.readAsString();  // Assumes UTF-8

// Explicit encoding
final latin1Content = await file.readAsString(encoding: latin1);

// System encoding (respects locale)
final systemContent = await file.readAsString(encoding: systemEncoding);
```

For streams:

```dart
// UTF-8
file.openRead().transform(utf8.decoder)

// System encoding
file.openRead().transform(systemEncoding.decoder)

// Latin-1
file.openRead().transform(latin1.decoder)
```

**Rule of thumb**: Use `systemEncoding` for stdin/stdout, and UTF-8 for files you create.

## Practical Example: Log File Analyzer

Let's build something useful — a tool that analyzes log files:

```dart
// bin/logstats.dart
import 'dart:io';
import 'dart:convert';

Future<void> main(List<String> arguments) async {
  final Stream<String> lines;

  if (arguments.isEmpty) {
    lines = stdin.transform(utf8.decoder).transform(const LineSplitter());
  } else {
    final file = File(arguments.first);
    if (!await file.exists()) {
      stderr.writeln('Error: File not found: ${arguments.first}');
      exit(1);
    }
    lines = file
        .openRead()
        .transform(utf8.decoder)
        .transform(const LineSplitter());
  }

  // Track statistics
  int totalLines = 0;
  int errorCount = 0;
  int warningCount = 0;
  int infoCount = 0;
  final Map<String, int> errorTypes = {};

  stderr.write('Analyzing log file... ');
  await stderr.flush();

  await for (final line in lines) {
    totalLines++;

    if (line.contains('ERROR')) {
      errorCount++;
      // Extract error type (simplified)
      final match = RegExp(r'ERROR: (\w+)').firstMatch(line);
      if (match != null) {
        final errorType = match.group(1)!;
        errorTypes[errorType] = (errorTypes[errorType] ?? 0) + 1;
      }
    } else if (line.contains('WARN')) {
      warningCount++;
    } else if (line.contains('INFO')) {
      infoCount++;
    }

    // Progress indicator every 10000 lines
    if (totalLines % 10000 == 0) {
      stderr.write('.');
      await stderr.flush();
    }
  }

  stderr.writeln(' done');

  // Output results to stdout (so they can be piped)
  print('=== Log Statistics ===');
  print('Total lines: $totalLines');
  print('Errors:      $errorCount');
  print('Warnings:    $warningCount');
  print('Info:        $infoCount');
  print('');

  if (errorTypes.isNotEmpty) {
    print('Error breakdown:');
    final sorted = errorTypes.entries.toList()
      ..sort((a, b) => b.value.compareTo(a.value));

    for (final entry in sorted) {
      print('  ${entry.key.padRight(20)} ${entry.value}');
    }
  }

  // Exit with error if there were errors in the log
  exit(errorCount > 0 ? 1 : 0);
}
```

Usage:

```bash
$ dart run bin/logstats.dart app.log
Analyzing log file... .......... done
=== Log Statistics ===
Total lines: 125847
Errors:      42
Warnings:    312
Info:        125493

Error breakdown:
  DatabaseTimeout      18
  NetworkError         15
  ValidationError      9

# Or from a pipe
$ cat app.log | dart run bin/logstats.dart
```

Notice:
- Progress goes to stderr (so it doesn't interfere with output)
- Results go to stdout (so they can be piped)
- Exit code reflects whether errors were found

## Path Handling: Cross-Platform Sanity

Don't do this:

```dart
// ❌ Breaks on Windows
final path = '$directory/$filename';
```

Do this:

```dart
import 'package:path/path.dart' as path;

// ✅ Works everywhere
final filePath = path.join(directory, filename);

// Other useful functions
path.basename('/foo/bar/file.txt');  // 'file.txt'
path.dirname('/foo/bar/file.txt');   // '/foo/bar'
path.extension('/foo/bar/file.txt'); // '.txt'
path.basenameWithoutExtension('/foo/bar/file.txt'); // 'file'

// Check if path is absolute
path.isAbsolute('/foo/bar');  // true
path.isAbsolute('foo/bar');   // false

// Convert to absolute
path.absolute('foo/bar');  // '/current/dir/foo/bar'
```

Add it to your project:

```bash
$ dart pub add path
```

## File System Operations

Common file operations you'll need:

```dart
import 'dart:io';

// Check if file exists
if (await File('data.txt').exists()) {
  print('File exists');
}

// Check if directory exists
if (await Directory('/tmp').exists()) {
  print('Directory exists');
}

// Create directory (and parents)
await Directory('path/to/dir').create(recursive: true);

// List directory contents
final dir = Directory('/tmp');
await for (final entity in dir.list()) {
  if (entity is File) {
    print('File: ${entity.path}');
  } else if (entity is Directory) {
    print('Dir: ${entity.path}');
  }
}

// Delete file
await File('temp.txt').delete();

// Delete directory (recursive)
await Directory('temp_dir').delete(recursive: true);

// Copy file
await File('source.txt').copy('dest.txt');

// Rename/move file
await File('old.txt').rename('new.txt');

// Get file stats
final stat = await File('data.txt').stat();
print('Size: ${stat.size} bytes');
print('Modified: ${stat.modified}');
```

## A Word on Atomic Writes

When writing critical files (configs, data), write atomically:

```dart
Future<void> writeFileSafely(String path, String content) async {
  final tempPath = '$path.tmp';

  // Write to temp file
  await File(tempPath).writeAsString(content);

  // Atomic rename (replaces old file)
  await File(tempPath).rename(path);
}
```

Why? If your program crashes mid-write, you won't corrupt the original file.

## Best Practices Recap

1. **Read from stdin if no file specified** — makes your tool pipe-friendly
2. **Write output to stdout, diagnostics to stderr** — never mix them
3. **Use streams for large files** — don't blow up memory
4. **Flush when needed** — especially for progress indicators
5. **Exit with proper codes** — scripts need to know if you succeeded
6. **Use the `path` package** — cross-platform path handling
7. **Handle missing files gracefully** — check before reading
8. **Respect encodings** — don't assume UTF-8 everywhere
9. **Write atomically for critical files** — avoid corruption

## What's Next?

You now know how to handle files and streams like a Unix wizard. Your tools can:
- Read from files or stdin
- Write to files or stdout
- Stream large files efficiently
- Play nicely with pipes
- Exit with proper codes

In the next chapter, we'll make your output *beautiful* with colors, bold text, and all the ANSI goodness your terminal can handle.

Time to get fabulous.

---

**[← Previous: Hello, Arguments!](02-hello-arguments.md)** | **[Next: Pretty Colors →](04-pretty-colors.md)**
