# Chapter 8: Error Handling

**Failing gracefully like a professional**

Here's the harsh truth: your CLI tool will fail. Files won't exist. Networks will be down. Users will provide garbage input. Disks will fill up. Permissions will be denied.

The difference between a professional tool and a toy is how it handles these failures.

Bad error handling:
```
Unhandled exception:
FileSystemException: Cannot open file, path = 'config.yaml' (OS Error: No such file or directory, errno = 2)
#0      _File.open.<anonymous closure> (dart:io/file_impl.dart:366:9)
#1      _rootRunUnary (dart:async/zone.dart:1434:47)
... 15 more lines of stack trace that users don't care about
```

Good error handling:
```
Error: Configuration file not found: config.yaml

To create a default config file, run:
  mytool init

For help, run:
  mytool --help
```

This chapter is about handling errors with grace, providing helpful messages, and exiting cleanly.

## Exit Codes: The Language of Success and Failure

When your program exits, it returns a code to the shell. This is how scripts know if commands succeeded:

```dart
import 'dart:io';

void main() {
  exit(0);  // Success
  exit(1);  // Failure
}
```

Scripts check these codes:

```bash
#!/bin/bash
if mytool process data.txt; then
  echo "Success!"
else
  echo "Failed with code $?"
fi
```

### Standard Exit Codes

While conventions vary, these are widely understood:

```dart
// Exit codes
const exitSuccess = 0;      // Everything worked
const exitFailure = 1;      // Generic error
const exitUsage = 2;        // Command line usage error
const exitNoInput = 66;     // Cannot open input file
const exitUnavailable = 69; // Service unavailable
const exitSoftware = 70;    // Internal software error
const exitIOError = 74;     // I/O error
const exitTempFail = 75;    // Temporary failure
const exitPermission = 77;  // Permission denied
```

These come from the POSIX `sysexits.h` standard. Using them makes your tool play nicely with others.

### Practical Exit Code Usage

```dart
import 'dart:io';

Future<void> main(List<String> args) async {
  // Usage error
  if (args.isEmpty) {
    stderr.writeln('Error: Missing required argument');
    stderr.writeln('Usage: mytool <file>');
    exit(2);  // exitUsage
  }

  final file = File(args.first);

  // File not found
  if (!await file.exists()) {
    stderr.writeln('Error: File not found: ${args.first}');
    exit(66);  // exitNoInput
  }

  // Permission denied
  try {
    await file.readAsString();
  } on FileSystemException catch (e) {
    if (e.osError?.errorCode == 13) {  // EACCES
      stderr.writeln('Error: Permission denied: ${args.first}');
      exit(77);  // exitPermission
    }
    rethrow;
  }

  // Success!
  exit(0);
}
```

**Best practice**: Exit 0 for success, 1 for generic errors, specific codes for specific errors.

## Exceptions vs Result Types

Dart has exceptions, but they're not always the best choice for error handling.

### Using Exceptions

Good for unexpected errors:

```dart
class ConfigException implements Exception {
  final String message;
  ConfigException(this.message);

  @override
  String toString() => 'ConfigException: $message';
}

Map<String, dynamic> parseConfig(String yaml) {
  if (yaml.isEmpty) {
    throw ConfigException('Config file is empty');
  }

  // ... parse ...

  return config;
}

void main() {
  try {
    final config = parseConfig(yamlString);
    print('Config loaded');
  } on ConfigException catch (e) {
    stderr.writeln('Error: ${e.message}');
    exit(1);
  }
}
```

### Using Result Types

Good for expected errors (like network failures):

```dart
class Result<T, E> {
  final T? value;
  final E? error;

  Result.success(this.value) : error = null;
  Result.failure(this.error) : value = null;

  bool get isSuccess => value != null;
  bool get isFailure => error != null;

  T unwrap() {
    if (value != null) return value!;
    throw StateError('Called unwrap() on a failure: $error');
  }

  T unwrapOr(T defaultValue) => value ?? defaultValue;
}

Future<Result<String, String>> fetchData(String url) async {
  try {
    final response = await http.get(Uri.parse(url));

    if (response.statusCode == 200) {
      return Result.success(response.body);
    } else {
      return Result.failure('HTTP ${response.statusCode}');
    }
  } catch (e) {
    return Result.failure('Network error: $e');
  }
}

void main() async {
  final result = await fetchData('https://api.example.com/data');

  if (result.isSuccess) {
    print('Data: ${result.value}');
  } else {
    stderr.writeln('Error: ${result.error}');
    exit(1);
  }
}
```

Or use the `dartz` package for a more complete Result/Either implementation.

**Rule of thumb**:
- Exceptions for programmer errors (null pointers, invalid state)
- Result types for expected failures (network errors, validation failures)

## Writing Helpful Error Messages

Bad error messages frustrate users. Good ones help them fix the problem.

### Bad Error Messages

```dart
// ❌ Too vague
stderr.writeln('Error');

// ❌ Too technical
stderr.writeln('Error: ENOENT');

// ❌ Not actionable
stderr.writeln('Error: Something went wrong');

// ❌ Lying (it's not a warning if you're exiting)
stderr.writeln('Warning: File not found');
exit(1);
```

### Good Error Messages

```dart
// ✅ Specific and actionable
stderr.writeln('Error: Configuration file not found: config.yaml');
stderr.writeln('');
stderr.writeln('To create a default config:');
stderr.writeln('  mytool init');

// ✅ Suggests fixes
stderr.writeln('Error: Port 8080 is already in use');
stderr.writeln('');
stderr.writeln('Try a different port:');
stderr.writeln('  mytool --port 8081');

// ✅ Explains what went wrong
stderr.writeln('Error: Invalid JSON in config file (line 5)');
stderr.writeln('  Expected comma after "name" field');
stderr.writeln('');
stderr.writeln('Check your config file:');
stderr.writeln('  ${configPath}');
```

### Error Message Template

A good error message has:

1. **What happened** (the error)
2. **Why it happened** (context)
3. **How to fix it** (actionable advice)

```dart
void reportError({
  required String what,
  String? why,
  String? howToFix,
}) {
  stderr.writeln('Error: $what'.red().bold());

  if (why != null) {
    stderr.writeln('');
    stderr.writeln(why.dim());
  }

  if (howToFix != null) {
    stderr.writeln('');
    stderr.writeln('To fix this:');
    stderr.writeln('  $howToFix');
  }
}

void main() {
  reportError(
    what: 'Database connection failed',
    why: 'Unable to connect to postgresql://localhost:5432',
    howToFix: 'Check that PostgreSQL is running:\n  systemctl status postgresql',
  );
}
```

Output:
```
Error: Database connection failed

Unable to connect to postgresql://localhost:5432

To fix this:
  Check that PostgreSQL is running:
    systemctl status postgresql
```

## Stack Traces: To Show or Not To Show

Stack traces are for developers, not users.

### Default Behavior (Bad)

```dart
void main() {
  throw Exception('Something broke');
  // Shows full stack trace to users
}
```

### Better: Catch and Report

```dart
void main() {
  try {
    dangerousOperation();
  } catch (e, stackTrace) {
    stderr.writeln('Error: $e');

    // Only show stack trace if --verbose
    if (verbose) {
      stderr.writeln('');
      stderr.writeln('Stack trace:');
      stderr.writeln(stackTrace);
    } else {
      stderr.writeln('');
      stderr.writeln('Run with --verbose for more details');
    }

    exit(1);
  }
}
```

### Even Better: Custom Error Types

```dart
class UserFacingError implements Exception {
  final String message;
  final String? details;
  final String? howToFix;

  UserFacingError(this.message, {this.details, this.howToFix});

  void report() {
    stderr.writeln('Error: $message'.red());

    if (details != null) {
      stderr.writeln('');
      stderr.writeln(details);
    }

    if (howToFix != null) {
      stderr.writeln('');
      stderr.writeln('To fix:');
      stderr.writeln('  $howToFix');
    }
  }
}

void main() {
  try {
    processFile('missing.txt');
  } on UserFacingError catch (e) {
    e.report();
    exit(1);
  } catch (e, stackTrace) {
    // Unexpected error - show details
    stderr.writeln('Unexpected error: $e');
    stderr.writeln(stackTrace);
    exit(70);  // exitSoftware
  }
}

void processFile(String path) {
  if (!File(path).existsSync()) {
    throw UserFacingError(
      'File not found: $path',
      howToFix: 'Check the file path and try again',
    );
  }
}
```

## Logging

For long-running CLI tools, logging is essential.

### Simple Logger

```dart
enum LogLevel {
  debug,
  info,
  warning,
  error,
}

class Logger {
  final LogLevel minLevel;
  final bool colorize;

  Logger({
    this.minLevel = LogLevel.info,
    bool? colorize,
  }) : colorize = colorize ?? stdout.supportsAnsiEscapes;

  void debug(String message) => _log(LogLevel.debug, message);
  void info(String message) => _log(LogLevel.info, message);
  void warning(String message) => _log(LogLevel.warning, message);
  void error(String message) => _log(LogLevel.error, message);

  void _log(LogLevel level, String message) {
    if (level.index < minLevel.index) return;

    final timestamp = DateTime.now().toIso8601String();
    final levelStr = level.name.toUpperCase().padRight(7);

    final output = level == LogLevel.error ? stderr : stdout;

    final formatted = '[$timestamp] $levelStr $message';

    if (colorize) {
      final colored = switch (level) {
        LogLevel.debug => formatted.dim(),
        LogLevel.info => formatted,
        LogLevel.warning => formatted.yellow(),
        LogLevel.error => formatted.red(),
      };
      output.writeln(colored);
    } else {
      output.writeln(formatted);
    }
  }
}

void main() {
  final logger = Logger(minLevel: LogLevel.debug);

  logger.debug('Starting application');
  logger.info('Connecting to database');
  logger.warning('Using default configuration');
  logger.error('Failed to connect');
}
```

Output:
```
[2024-01-15T10:30:45.123] DEBUG   Starting application
[2024-01-15T10:30:45.234] INFO    Connecting to database
[2024-01-15T10:30:45.345] WARNING Using default configuration
[2024-01-15T10:30:45.456] ERROR   Failed to connect
```

### Logging to Files

```dart
class FileLogger extends Logger {
  final String logPath;
  IOSink? _sink;

  FileLogger({
    required this.logPath,
    super.minLevel,
  }) : super(colorize: false) {
    _sink = File(logPath).openWrite(mode: FileMode.append);
  }

  @override
  void _log(LogLevel level, String message) {
    super._log(level, message);  // Also log to console

    final timestamp = DateTime.now().toIso8601String();
    final levelStr = level.name.toUpperCase().padRight(7);
    _sink?.writeln('[$timestamp] $levelStr $message');
  }

  void close() {
    _sink?.close();
  }
}
```

### Using the `logging` Package

For more features, use Dart's `logging` package:

```bash
$ dart pub add logging
```

```dart
import 'package:logging/logging.dart';

void setupLogging({bool verbose = false}) {
  Logger.root.level = verbose ? Level.ALL : Level.INFO;

  Logger.root.onRecord.listen((record) {
    final output = record.level >= Level.WARNING ? stderr : stdout;
    output.writeln('[${record.level.name}] ${record.message}');

    if (record.error != null) {
      output.writeln('  Error: ${record.error}');
    }

    if (record.stackTrace != null && verbose) {
      output.writeln('  Stack trace:');
      output.writeln(record.stackTrace);
    }
  });
}

void main() {
  setupLogging(verbose: true);

  final log = Logger('MyApp');

  log.fine('Debug message');
  log.info('Info message');
  log.warning('Warning message');
  log.severe('Error message', Exception('Something broke'));
}
```

## Retries and Backoff

For transient failures (network errors), retry with exponential backoff:

```dart
import 'dart:math';

Future<T> retryWithBackoff<T>(
  Future<T> Function() operation, {
  int maxAttempts = 3,
  Duration initialDelay = const Duration(seconds: 1),
  double multiplier = 2.0,
  Duration? maxDelay,
}) async {
  var attempt = 0;
  var delay = initialDelay;

  while (true) {
    attempt++;

    try {
      return await operation();
    } catch (e) {
      if (attempt >= maxAttempts) {
        rethrow;  // Give up
      }

      stderr.writeln('Attempt $attempt failed: $e');
      stderr.writeln('Retrying in ${delay.inSeconds}s...');

      await Future.delayed(delay);

      delay *= multiplier;
      if (maxDelay != null && delay > maxDelay) {
        delay = maxDelay;
      }
    }
  }
}

// Usage
Future<void> main() async {
  try {
    final data = await retryWithBackoff(
      () => fetchDataFromAPI('https://api.example.com/data'),
      maxAttempts: 5,
      initialDelay: Duration(seconds: 2),
    );

    print('Success: $data');
  } catch (e) {
    stderr.writeln('Failed after all retries: $e');
    exit(1);
  }
}

Future<String> fetchDataFromAPI(String url) async {
  // Simulated API call that might fail
  if (Random().nextBool()) {
    throw Exception('Network error');
  }
  return 'data';
}
```

## Validation Errors

Collect multiple validation errors before failing:

```dart
class ValidationError {
  final String field;
  final String message;

  ValidationError(this.field, this.message);

  @override
  String toString() => '$field: $message';
}

class ValidationException implements Exception {
  final List<ValidationError> errors;

  ValidationException(this.errors);

  void report() {
    stderr.writeln('Validation failed:');
    for (final error in errors) {
      stderr.writeln('  • $error');
    }
  }
}

class UserConfig {
  final String email;
  final int age;
  final String password;

  UserConfig({
    required this.email,
    required this.age,
    required this.password,
  });

  static UserConfig validate(Map<String, dynamic> data) {
    final errors = <ValidationError>[];

    // Validate email
    final email = data['email'] as String?;
    if (email == null || email.isEmpty) {
      errors.add(ValidationError('email', 'Email is required'));
    } else if (!email.contains('@')) {
      errors.add(ValidationError('email', 'Email must be valid'));
    }

    // Validate age
    final age = data['age'] as int?;
    if (age == null) {
      errors.add(ValidationError('age', 'Age is required'));
    } else if (age < 0 || age > 150) {
      errors.add(ValidationError('age', 'Age must be between 0 and 150'));
    }

    // Validate password
    final password = data['password'] as String?;
    if (password == null || password.isEmpty) {
      errors.add(ValidationError('password', 'Password is required'));
    } else if (password.length < 8) {
      errors.add(ValidationError('password', 'Password must be at least 8 characters'));
    }

    if (errors.isNotEmpty) {
      throw ValidationException(errors);
    }

    return UserConfig(
      email: email!,
      age: age!,
      password: password!,
    );
  }
}

void main() {
  try {
    final config = UserConfig.validate({
      'email': 'invalid-email',
      'age': 200,
      'password': '123',
    });
  } on ValidationException catch (e) {
    e.report();
    exit(1);
  }
}
```

Output:
```
Validation failed:
  • email: Email must be valid
  • age: Age must be between 0 and 150
  • password: Password must be at least 8 characters
```

## Graceful Degradation

When optional features fail, continue anyway:

```dart
Future<void> main() async {
  final logger = Logger();

  // Critical: must succeed
  try {
    await connectToDatabase();
    logger.info('Connected to database');
  } catch (e) {
    logger.error('Failed to connect to database: $e');
    exit(1);  // Can't continue without database
  }

  // Optional: can fail gracefully
  try {
    await setupCaching();
    logger.info('Caching enabled');
  } catch (e) {
    logger.warning('Failed to setup caching: $e');
    logger.warning('Continuing without cache');
    // Don't exit - caching is optional
  }

  // Continue with main work
  await processData();
}
```

## Cleanup on Error

Always clean up resources, even when errors occur:

```dart
Future<void> processFile(String inputPath, String outputPath) async {
  IOSink? output;

  try {
    // Open resources
    final input = File(inputPath);
    output = File(outputPath).openWrite();

    // Process
    await for (final line in input.openRead().transform(utf8.decoder).transform(LineSplitter())) {
      output.writeln(line.toUpperCase());
    }

    logger.info('Processing complete');
  } catch (e) {
    logger.error('Processing failed: $e');

    // Clean up partial output file
    if (output != null) {
      await output.close();
      await File(outputPath).delete();
    }

    rethrow;
  } finally {
    // Always close resources
    await output?.close();
  }
}
```

Or use `try`-with-resources pattern:

```dart
Future<void> withFile<T>(
  String path,
  Future<T> Function(IOSink sink) action,
) async {
  final file = File(path);
  final sink = file.openWrite();

  try {
    return await action(sink);
  } finally {
    await sink.close();
  }
}

// Usage
await withFile('output.txt', (sink) async {
  sink.writeln('Line 1');
  sink.writeln('Line 2');
});
```

## Real-World Example: Robust File Processor

```dart
import 'dart:io';
import 'dart:convert';

class ProcessingError implements Exception {
  final String message;
  final String? details;

  ProcessingError(this.message, {this.details});

  void report() {
    stderr.writeln('Error: $message'.red());
    if (details != null) {
      stderr.writeln('  $details'.dim());
    }
  }
}

Future<void> processFiles(List<String> inputFiles, String outputDir) async {
  // Validate inputs
  if (inputFiles.isEmpty) {
    throw ProcessingError('No input files specified',
        details: 'Provide at least one file to process');
  }

  final missingFiles = <String>[];
  for (final file in inputFiles) {
    if (!await File(file).exists()) {
      missingFiles.add(file);
    }
  }

  if (missingFiles.isNotEmpty) {
    throw ProcessingError(
      'Input files not found: ${missingFiles.join(', ')}',
      details: 'Check the file paths and try again',
    );
  }

  // Create output directory
  try {
    await Directory(outputDir).create(recursive: true);
  } on FileSystemException catch (e) {
    throw ProcessingError(
      'Cannot create output directory: $outputDir',
      details: e.message,
    );
  }

  // Process files
  final errors = <String>[];

  for (final inputFile in inputFiles) {
    try {
      await _processFile(inputFile, outputDir);
      print('✓ Processed: $inputFile');
    } catch (e) {
      errors.add('$inputFile: $e');
      print('✗ Failed: $inputFile'.red());
    }
  }

  // Report summary
  final successCount = inputFiles.length - errors.length;
  print('\nProcessed $successCount/${inputFiles.length} files');

  if (errors.isNotEmpty) {
    stderr.writeln('\nErrors:');
    for (final error in errors) {
      stderr.writeln('  • $error');
    }
    exit(1);
  }
}

Future<void> _processFile(String inputPath, String outputDir) async {
  final input = File(inputPath);
  final outputPath = path.join(
    outputDir,
    path.basenameWithoutExtension(inputPath) + '.processed',
  );

  IOSink? output;

  try {
    output = File(outputPath).openWrite();

    await for (final line in input
        .openRead()
        .transform(utf8.decoder)
        .transform(const LineSplitter())) {
      // Process line
      final processed = line.trim().toUpperCase();
      output.writeln(processed);
    }
  } catch (e) {
    // Clean up partial output
    await output?.close();
    try {
      await File(outputPath).delete();
    } catch (_) {
      // Ignore cleanup errors
    }
    rethrow;
  } finally {
    await output?.close();
  }
}

void main(List<String> args) async {
  if (args.isEmpty) {
    stderr.writeln('Usage: process <input-files...> --output <dir>');
    exit(2);
  }

  try {
    final files = args.where((a) => !a.startsWith('--')).toList();
    final outputDir = 'output';  // Parse from --output flag

    await processFiles(files, outputDir);
  } on ProcessingError catch (e) {
    e.report();
    exit(1);
  } catch (e, stackTrace) {
    stderr.writeln('Unexpected error: $e');
    stderr.writeln('Stack trace:');
    stderr.writeln(stackTrace);
    exit(70);
  }
}
```

## Best Practices

1. **Exit with proper codes** — 0 for success, non-zero for errors
2. **Write errors to stderr** — not stdout
3. **Be specific** — "File not found: config.yaml" not "Error"
4. **Suggest solutions** — tell users how to fix the problem
5. **Collect validation errors** — show all problems at once
6. **Hide stack traces from users** — show them only with --verbose
7. **Clean up on failure** — delete partial files, close connections
8. **Log appropriately** — debug for developers, info for users
9. **Retry transient failures** — with exponential backoff
10. **Fail fast** — validate early, fail before doing work

## What's Next?

You now know how to handle errors like a pro! You've learned:

- Exit codes and their meanings
- Writing helpful error messages
- Logging strategies
- Retry logic with backoff
- Validation patterns
- Cleanup and resource management

In the next chapter, we level up: **Terminal User Interfaces (TUIs)**. Time to build full-screen applications that make your terminal feel like a real app.

Let's get graphical (in a text-only way)!

---

**[← Previous: Configuration Files](07-configuration-files.md)** | **[Next: Building a TUI →](09-building-a-tui.md)**
