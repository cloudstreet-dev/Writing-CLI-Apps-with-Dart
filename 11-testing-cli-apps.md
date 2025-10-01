# Chapter 11: Testing CLI Apps

**How to test the "untestable"**

Here's the thing about CLI apps: they're notoriously hard to test. They interact with stdin, stdout, files, the environment, and the terminal. How do you write automated tests for something that expects user input and prints to the screen?

The answer: **very carefully**, with the right abstractions and tools.

This chapter covers:
- Unit testing business logic
- Testing output (stdout/stderr)
- Mocking file systems and stdin
- Integration testing full CLI flows
- Snapshot testing for output
- Testing interactive prompts
- CI/CD strategies

Let's make your code bulletproof.

## The Testing Pyramid for CLI Apps

Like all software, CLI apps benefit from a testing pyramid:

```
        /\
       /  \      E2E Tests (few)
      /────\     - Full program execution
     /      \    - Real files, real terminal
    /────────\   Integration Tests (some)
   /          \  - Components together
  /────────────\ - Mocked I/O
 /______________\ Unit Tests (many)
                  - Pure functions
                  - Business logic
```

Most tests should be fast unit tests. Fewer should be integration tests. Very few should be end-to-end tests.

## Unit Testing Pure Functions

The easiest place to start: extract pure functions and test them.

```dart
// lib/parser.dart
class LogEntry {
  final DateTime timestamp;
  final String level;
  final String message;

  LogEntry(this.timestamp, this.level, this.message);
}

/// Parse a log line
LogEntry? parseLogLine(String line) {
  // Format: [2024-01-15 10:30:45] ERROR: Something went wrong
  final regex = RegExp(r'\[(.+?)\] (\w+): (.+)');
  final match = regex.firstMatch(line);

  if (match == null) return null;

  final timestamp = DateTime.tryParse(match.group(1)!);
  if (timestamp == null) return null;

  return LogEntry(
    timestamp,
    match.group(2)!,
    match.group(3)!,
  );
}
```

Test it:

```dart
// test/parser_test.dart
import 'package:test/test.dart';
import 'package:myapp/parser.dart';

void main() {
  group('parseLogLine', () {
    test('parses valid log line', () {
      final line = '[2024-01-15 10:30:45] ERROR: Something went wrong';
      final entry = parseLogLine(line);

      expect(entry, isNotNull);
      expect(entry!.level, 'ERROR');
      expect(entry.message, 'Something went wrong');
      expect(entry.timestamp.year, 2024);
    });

    test('returns null for invalid format', () {
      final entry = parseLogLine('not a log line');
      expect(entry, isNull);
    });

    test('returns null for invalid timestamp', () {
      final entry = parseLogLine('[invalid-date] ERROR: message');
      expect(entry, isNull);
    });

    test('handles different log levels', () {
      final levels = ['ERROR', 'WARN', 'INFO', 'DEBUG'];

      for (final level in levels) {
        final line = '[2024-01-15 10:30:45] $level: Test message';
        final entry = parseLogLine(line);

        expect(entry, isNotNull);
        expect(entry!.level, level);
      }
    });
  });
}
```

Run tests:

```bash
$ dart test
```

**Key principle**: Extract as much logic as possible into pure functions, then test those exhaustively.

## Testing Output (stdout/stderr)

CLI apps write to stdout and stderr. How do you test that?

### Option 1: Capture Output in Tests

```dart
// lib/greeter.dart
void greet(String name, {bool excited = false}) {
  var message = 'Hello, $name';
  if (excited) message += '!';
  print(message);
}
```

Test by capturing print statements:

```dart
// test/greeter_test.dart
import 'dart:io';
import 'package:test/test.dart';
import 'package:myapp/greeter.dart';

void main() {
  test('greets with name', () {
    final output = captureOutput(() {
      greet('Alice');
    });

    expect(output, 'Hello, Alice\n');
  });

  test('adds excitement', () {
    final output = captureOutput(() {
      greet('Bob', excited: true);
    });

    expect(output, 'Hello, Bob!\n');
  });
}

/// Capture stdout from a function
String captureOutput(void Function() fn) {
  final buffer = StringBuffer();

  runZoned(
    fn,
    zoneSpecification: ZoneSpecification(
      print: (self, parent, zone, line) {
        buffer.writeln(line);
      },
    ),
  );

  return buffer.toString();
}
```

### Option 2: Dependency Injection

Better: inject the output sink.

```dart
// lib/greeter.dart
class Greeter {
  final IOSink output;

  Greeter({IOSink? output}) : output = output ?? stdout;

  void greet(String name, {bool excited = false}) {
    var message = 'Hello, $name';
    if (excited) message += '!';
    output.writeln(message);
  }
}
```

Test with a fake sink:

```dart
// test/greeter_test.dart
import 'dart:convert';
import 'dart:io';
import 'package:test/test.dart';
import 'package:myapp/greeter.dart';

void main() {
  test('greets with name', () {
    final buffer = StringBuffer();
    final sink = _StringSink(buffer);
    final greeter = Greeter(output: sink);

    greeter.greet('Alice');

    expect(buffer.toString(), 'Hello, Alice\n');
  });
}

class _StringSink implements IOSink {
  final StringBuffer buffer;

  _StringSink(this.buffer);

  @override
  void writeln([Object? obj = '']) {
    buffer.writeln(obj);
  }

  @override
  void write(Object? obj) {
    buffer.write(obj);
  }

  // Implement other IOSink methods as needed
  @override
  void add(List<int> data) => buffer.write(utf8.decode(data));

  @override
  void writeAll(Iterable objects, [String separator = '']) =>
      buffer.writeAll(objects, separator);

  @override
  Future addStream(Stream<List<int>> stream) async {}

  @override
  Future close() async {}

  @override
  Future flush() async {}

  @override
  Encoding get encoding => utf8;

  @override
  set encoding(Encoding _encoding) {}

  @override
  Future get done => Future.value();
}
```

Now your code is testable without capturing zone output.

## Mocking the File System

Don't test against real files. Mock the file system.

```dart
// lib/file_processor.dart
abstract class FileSystem {
  Future<bool> exists(String path);
  Future<String> readAsString(String path);
  Future<void> writeAsString(String path, String contents);
}

class RealFileSystem implements FileSystem {
  @override
  Future<bool> exists(String path) => File(path).exists();

  @override
  Future<String> readAsString(String path) => File(path).readAsString();

  @override
  Future<void> writeAsString(String path, String contents) =>
      File(path).writeAsString(contents);
}

class FileProcessor {
  final FileSystem fs;

  FileProcessor(this.fs);

  Future<int> processFile(String inputPath, String outputPath) async {
    if (!await fs.exists(inputPath)) {
      throw Exception('File not found: $inputPath');
    }

    final contents = await fs.readAsString(inputPath);
    final lines = contents.split('\n');
    final processed = lines.map((line) => line.toUpperCase()).join('\n');

    await fs.writeAsString(outputPath, processed);

    return lines.length;
  }
}
```

Test with a fake file system:

```dart
// test/file_processor_test.dart
import 'package:test/test.dart';
import 'package:myapp/file_processor.dart';

void main() {
  group('FileProcessor', () {
    test('processes file correctly', () async {
      final fs = FakeFileSystem({
        'input.txt': 'hello\nworld',
      });

      final processor = FileProcessor(fs);
      final lines = await processor.processFile('input.txt', 'output.txt');

      expect(lines, 2);
      expect(fs.files['output.txt'], 'HELLO\nWORLD');
    });

    test('throws when file not found', () async {
      final fs = FakeFileSystem({});
      final processor = FileProcessor(fs);

      expect(
        () => processor.processFile('missing.txt', 'output.txt'),
        throwsException,
      );
    });
  });
}

class FakeFileSystem implements FileSystem {
  final Map<String, String> files;

  FakeFileSystem(this.files);

  @override
  Future<bool> exists(String path) async => files.containsKey(path);

  @override
  Future<String> readAsString(String path) async {
    if (!files.containsKey(path)) {
      throw Exception('File not found: $path');
    }
    return files[path]!;
  }

  @override
  Future<void> writeAsString(String path, String contents) async {
    files[path] = contents;
  }
}
```

No real files created. Tests run fast and in isolation.

## Testing Argument Parsing

Argument parsing is critical. Test it thoroughly:

```dart
// test/cli_test.dart
import 'package:test/test.dart';
import 'package:args/args.dart';
import 'package:myapp/cli.dart';

void main() {
  group('Argument parsing', () {
    test('parses name option', () {
      final parser = buildArgParser();
      final results = parser.parse(['--name', 'Alice']);

      expect(results['name'], 'Alice');
    });

    test('uses default name', () {
      final parser = buildArgParser();
      final results = parser.parse([]);

      expect(results['name'], 'World');
    });

    test('parses excited flag', () {
      final parser = buildArgParser();
      final results = parser.parse(['--excited']);

      expect(results['excited'], true);
    });

    test('rejects invalid arguments', () {
      final parser = buildArgParser();

      expect(
        () => parser.parse(['--invalid']),
        throwsA(isA<FormatException>()),
      );
    });
  });
}

ArgParser buildArgParser() {
  return ArgParser()
    ..addOption('name', defaultsTo: 'World')
    ..addFlag('excited', defaultsTo: false);
}
```

## Integration Testing

Test entire workflows with mocked I/O:

```dart
// test/integration_test.dart
import 'package:test/test.dart';
import 'package:myapp/app.dart';

void main() {
  group('End-to-end flow', () {
    test('processes file successfully', () async {
      final fs = FakeFileSystem({
        'input.txt': 'line 1\nline 2\nline 3',
      });

      final output = StringBuffer();
      final sink = _StringSink(output);

      final app = App(
        fileSystem: fs,
        stdout: sink,
        stderr: sink,
      );

      final exitCode = await app.run(['--input', 'input.txt', '--output', 'output.txt']);

      expect(exitCode, 0);
      expect(fs.files['output.txt'], 'LINE 1\nLINE 2\nLINE 3');
      expect(output.toString(), contains('Processed 3 lines'));
    });

    test('handles missing file', () async {
      final fs = FakeFileSystem({});
      final stderr = StringBuffer();

      final app = App(
        fileSystem: fs,
        stderr: _StringSink(stderr),
      );

      final exitCode = await app.run(['--input', 'missing.txt']);

      expect(exitCode, 1);
      expect(stderr.toString(), contains('File not found'));
    });
  });
}
```

## Snapshot Testing

For output that's too complex to manually verify, use snapshot testing:

```dart
// test/output_test.dart
import 'package:test/test.dart';
import 'package:myapp/formatter.dart';

void main() {
  test('formats table correctly', () {
    final data = [
      {'name': 'Alice', 'age': 30},
      {'name': 'Bob', 'age': 25},
    ];

    final output = formatTable(data);

    expect(output, matchesGoldenFile('golden/table_output.txt'));
  });
}

/// Custom matcher for golden file comparison
Matcher matchesGoldenFile(String path) {
  return _GoldenFileMatcher(path);
}

class _GoldenFileMatcher extends Matcher {
  final String path;

  _GoldenFileMatcher(this.path);

  @override
  bool matches(dynamic item, Map matchState) {
    final actual = item.toString();
    final expected = File(path).readAsStringSync();

    if (actual == expected) return true;

    // If not matching, update the golden file (when running with --update-goldens)
    if (Platform.environment.containsKey('UPDATE_GOLDENS')) {
      File(path).writeAsStringSync(actual);
      return true;
    }

    matchState['actual'] = actual;
    matchState['expected'] = expected;
    return false;
  }

  @override
  Description describe(Description description) =>
      description.add('matches golden file $path');

  @override
  Description describeMismatch(
    dynamic item,
    Description mismatch,
    Map matchState,
    bool verbose,
  ) {
    return mismatch
        .add('expected:\n${matchState['expected']}\n')
        .add('but got:\n${matchState['actual']}');
  }
}
```

Update golden files when output changes intentionally:

```bash
$ UPDATE_GOLDENS=1 dart test
```

## Testing Interactive Prompts

Interactive prompts are tricky. Abstract them:

```dart
// lib/prompter.dart
abstract class Prompter {
  String ask(String question);
  bool confirm(String question);
}

class InteractivePrompter implements Prompter {
  @override
  String ask(String question) {
    stdout.write('$question: ');
    return stdin.readLineSync() ?? '';
  }

  @override
  bool confirm(String question) {
    stdout.write('$question (y/n): ');
    final answer = stdin.readLineSync()?.toLowerCase();
    return answer == 'y' || answer == 'yes';
  }
}

class FakePrompter implements Prompter {
  final Map<String, String> answers;
  final Map<String, bool> confirmations;

  FakePrompter({
    this.answers = const {},
    this.confirmations = const {},
  });

  @override
  String ask(String question) {
    return answers[question] ?? '';
  }

  @override
  bool confirm(String question) {
    return confirmations[question] ?? false;
  }
}
```

Test with fake prompter:

```dart
// test/interactive_test.dart
import 'package:test/test.dart';
import 'package:myapp/app.dart';

void main() {
  test('creates project with user input', () async {
    final prompter = FakePrompter(
      answers: {
        'Project name': 'my-app',
        'Description': 'My cool app',
      },
      confirmations: {
        'Initialize git?': true,
      },
    );

    final app = ProjectScaffolder(prompter: prompter);
    await app.run();

    // Verify project was created correctly
    expect(app.projectName, 'my-app');
    expect(app.description, 'My cool app');
    expect(app.initGit, true);
  });
}
```

## Testing Error Handling

Test both happy paths and error cases:

```dart
// test/error_handling_test.dart
import 'package:test/test.dart';
import 'package:myapp/processor.dart';

void main() {
  group('Error handling', () {
    test('handles file not found', () async {
      final fs = FakeFileSystem({});
      final processor = Processor(fs);

      expect(
        () => processor.process('missing.txt'),
        throwsA(predicate((e) =>
          e is ProcessingException &&
          e.message.contains('File not found')
        )),
      );
    });

    test('handles invalid format', () async {
      final fs = FakeFileSystem({'file.json': 'not valid json'});
      final processor = Processor(fs);

      expect(
        () => processor.process('file.json'),
        throwsA(predicate((e) =>
          e is ProcessingException &&
          e.message.contains('Invalid format')
        )),
      );
    });

    test('cleans up on error', () async {
      final fs = FakeFileSystem({'input.txt': 'data'});
      final processor = Processor(fs);

      try {
        await processor.processWithError('input.txt', 'output.txt');
      } catch (e) {
        // Expected to fail
      }

      // Verify cleanup happened
      expect(fs.files.containsKey('output.txt'), false);
    });
  });
}
```

## Testing Exit Codes

Verify your app exits with correct codes:

```dart
// test/exit_codes_test.dart
import 'package:test/test.dart';
import 'package:myapp/app.dart';

void main() {
  test('exits 0 on success', () async {
    final app = createTestApp();
    final exitCode = await app.run(['--input', 'file.txt']);

    expect(exitCode, 0);
  });

  test('exits 1 on generic error', () async {
    final app = createTestApp();
    final exitCode = await app.run(['--input', 'missing.txt']);

    expect(exitCode, 1);
  });

  test('exits 2 on usage error', () async {
    final app = createTestApp();
    final exitCode = await app.run(['--invalid-flag']);

    expect(exitCode, 2);
  });
}
```

## Running Tests in CI/CD

### GitHub Actions

```yaml
# .github/workflows/test.yml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - uses: dart-lang/setup-dart@v1
        with:
          sdk: stable

      - name: Install dependencies
        run: dart pub get

      - name: Run tests
        run: dart test

      - name: Check code coverage
        run: |
          dart pub global activate coverage
          dart pub global run coverage:test_with_coverage

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage/lcov.info
```

### Test Coverage

Generate coverage reports:

```bash
# Install coverage tool
$ dart pub global activate coverage

# Run tests with coverage
$ dart pub global run coverage:test_with_coverage

# View coverage report
$ genhtml coverage/lcov.info -o coverage/html
$ open coverage/html/index.html
```

## Testing Best Practices

1. **Test behavior, not implementation** — focus on what, not how
2. **Use dependency injection** — makes mocking easy
3. **Keep tests fast** — mock I/O, avoid real files/network
4. **Test edge cases** — empty input, huge input, invalid input
5. **Test error paths** — not just happy paths
6. **Use descriptive test names** — `test('handles file not found')` not `test('test1')`
7. **One assertion per test** (when possible) — easier to debug failures
8. **Use setup/teardown** — for common test data
9. **Test exit codes** — CLI apps communicate via exit codes
10. **Run tests in CI** — catch regressions automatically

## Test Organization

```
my_cli_app/
├── lib/
│   ├── src/
│   │   ├── config.dart
│   │   ├── parser.dart
│   │   └── processor.dart
│   └── my_cli_app.dart
├── test/
│   ├── fixtures/           # Test data files
│   │   ├── input.txt
│   │   └── expected.txt
│   ├── golden/             # Golden file snapshots
│   │   └── output.txt
│   ├── helpers/            # Test utilities
│   │   ├── fake_fs.dart
│   │   └── matchers.dart
│   ├── config_test.dart    # One test file per source file
│   ├── parser_test.dart
│   ├── processor_test.dart
│   └── integration_test.dart
└── pubspec.yaml
```

## Example: Complete Test Suite

```dart
// test/my_app_test.dart
import 'package:test/test.dart';
import 'package:myapp/myapp.dart';

void main() {
  group('MyApp', () {
    late FakeFileSystem fs;
    late StringBuffer stdout;
    late StringBuffer stderr;
    late MyApp app;

    setUp(() {
      fs = FakeFileSystem({});
      stdout = StringBuffer();
      stderr = StringBuffer();

      app = MyApp(
        fileSystem: fs,
        stdout: _StringSink(stdout),
        stderr: _StringSink(stderr),
      );
    });

    group('argument parsing', () {
      test('requires input file', () async {
        final exitCode = await app.run([]);

        expect(exitCode, 2);  // Usage error
        expect(stderr.toString(), contains('input file required'));
      });

      test('accepts valid arguments', () async {
        fs.files['input.txt'] = 'data';

        final exitCode = await app.run(['--input', 'input.txt']);

        expect(exitCode, 0);
      });
    });

    group('file processing', () {
      test('processes file successfully', () async {
        fs.files['input.txt'] = 'hello\nworld';

        await app.run(['--input', 'input.txt', '--output', 'output.txt']);

        expect(fs.files['output.txt'], 'HELLO\nWORLD');
        expect(stdout.toString(), contains('Processed 2 lines'));
      });

      test('handles empty file', () async {
        fs.files['input.txt'] = '';

        final exitCode = await app.run(['--input', 'input.txt']);

        expect(exitCode, 0);
        expect(stdout.toString(), contains('Processed 0 lines'));
      });
    });

    group('error handling', () {
      test('reports file not found', () async {
        final exitCode = await app.run(['--input', 'missing.txt']);

        expect(exitCode, 1);
        expect(stderr.toString(), contains('File not found'));
      });

      test('handles write errors gracefully', () async {
        fs.files['input.txt'] = 'data';
        fs.makeReadOnly('output.txt');

        final exitCode = await app.run([
          '--input', 'input.txt',
          '--output', 'output.txt',
        ]);

        expect(exitCode, 1);
        expect(stderr.toString(), contains('Cannot write'));
      });
    });
  });
}
```

## What's Next?

You now know how to test CLI apps thoroughly! You've learned:

- Unit testing pure functions
- Mocking stdout/stderr
- Faking file systems
- Testing argument parsing
- Integration testing
- Snapshot testing for output
- Testing interactive prompts
- CI/CD setup

In the final chapter, we'll cover **Packaging and Distribution** — getting your beautiful, well-tested CLI tool into users' hands.

Time to ship it!

---

**[← Previous: Advanced TUI Patterns](10-advanced-tui.md)** | **[Next: Packaging and Distribution →](12-packaging-and-distribution.md)**
