# Chapter 4: Pretty Colors

**Making your terminal output fabulous**

Let me tell you a secret: nobody actually wants to use your CLI tool. They *have* to use it to get work done. So the least you can do is make it pleasant to look at.

Monochrome terminal output is functional, sure. But colors? Colors can:

- **Highlight important information** (errors in red, success in green)
- **Improve readability** (syntax highlighting for code/logs)
- **Provide visual hierarchy** (headers, sections, emphasis)
- **Make your tool feel modern** (it's not 1985 anymore)

In this chapter, you'll learn how to add colors, bold text, underlines, and more to your CLI output. Plus, you'll learn how to do it *responsibly* — respecting terminal capabilities and user preferences.

## ANSI Escape Codes: The Magic Behind Colors

Colors in terminals work using **ANSI escape codes** — special sequences that tell the terminal to change formatting. They look like this:

```
\x1b[31mThis is red text\x1b[0m
```

Breaking it down:
- `\x1b[` — The escape sequence start
- `31` — The color code (31 = red)
- `m` — End of the code
- `\x1b[0m` — Reset formatting

You could write these manually:

```dart
void main() {
  print('\x1b[31mError: Something went wrong!\x1b[0m');
  print('\x1b[32mSuccess!\x1b[0m');
}
```

This works, but it's a pain. You have to remember codes, handle resets, and deal with edge cases. Let's use a package instead.

## The Easy Way: ANSI Packages

Dart has several excellent packages for terminal colors. Let's explore two popular ones.

### Option 1: `ansi_styles`

Clean, simple, chain-able API:

```bash
$ dart pub add ansi_styles
```

```dart
import 'package:ansi_styles/ansi_styles.dart';

void main() {
  print(AnsiStyles.red('Error: File not found'));
  print(AnsiStyles.green('Success!'));
  print(AnsiStyles.yellow('Warning: Deprecated API'));

  // Chain multiple styles
  print(AnsiStyles.bold(AnsiStyles.blue('Important Message')));

  // Or use the shorter syntax
  print(AnsiStyles.red.bold('Critical Error!'));
}
```

### Option 2: `tint`

Lightweight and expressive:

```bash
$ dart pub add tint
```

```dart
import 'package:tint/tint.dart';

void main() {
  print('Error: File not found'.red());
  print('Success!'.green());
  print('Warning: Deprecated API'.yellow());

  // Chain styles
  print('Important Message'.blue().bold());

  // Background colors
  print('Highlighted'.white().onRed());

  // Strip colors
  final colored = 'Hello'.red();
  final plain = colored.strip();  // 'Hello'
}
```

I prefer `tint` for its clean extension-method syntax. We'll use it for the rest of this chapter.

## Basic Colors

The 8 standard colors work everywhere:

```dart
import 'package:tint/tint.dart';

void main() {
  print('Black text'.black());
  print('Red text'.red());
  print('Green text'.green());
  print('Yellow text'.yellow());
  print('Blue text'.blue());
  print('Magenta text'.magenta());
  print('Cyan text'.cyan());
  print('White text'.white());

  // Bright variants
  print('Bright red'.brightRed());
  print('Bright green'.brightGreen());
  print('Bright yellow'.brightYellow());
  // ... etc
}
```

## Text Formatting

Beyond colors:

```dart
print('Bold text'.bold());
print('Dimmed text'.dim());
print('Italic text'.italic());
print('Underlined text'.underline());
print('Inverse colors'.inverse());
print('Hidden text'.hidden());  // Useful for passwords
print('Strikethrough'.strikethrough());
```

## Background Colors

```dart
print('Red background'.onRed());
print('Green background'.onGreen());
print('Yellow background'.onYellow());

// Combine with foreground colors
print('Black on white'.black().onWhite());
print('White on red'.white().onRed());
```

## Chaining Styles

The magic of extension methods:

```dart
// Multiple styles
print('Error!'.red().bold().underline());

// Complex combinations
print('Critical Warning'.yellow().onRed().bold());

// Readable code
final message = 'Server started'
    .green()
    .bold();
print(message);
```

## 256 Colors and True Color

Modern terminals support more than 8 colors:

```dart
// 256-color mode (most terminals)
print('Custom color'.rgb(5, 2, 3));  // 0-5 for each channel

// True color / 24-bit (modern terminals)
print('Custom color'.rgb(100, 150, 200));  // 0-255 for each channel
```

But be careful — not all terminals support this. More on that later.

## Semantic Colors: A Better Approach

Instead of sprinkling colors everywhere, use semantic helpers:

```dart
// Bad: color names scattered through code
stderr.writeln('Error occurred'.red());
print('Processing...'.yellow());

// Good: semantic meaning
void printError(String message) {
  stderr.writeln('Error: $message'.red().bold());
}

void printWarning(String message) {
  stderr.writeln('Warning: $message'.yellow());
}

void printSuccess(String message) {
  print('✓ $message'.green());
}

void printInfo(String message) {
  print('ℹ $message'.cyan());
}

// Usage is clearer
printError('File not found');
printSuccess('File created');
printWarning('File is large');
printInfo('Processing 1000 items');
```

Benefits:
1. Consistent colors across your app
2. Easy to change the color scheme
3. Self-documenting code
4. Can disable colors in one place

## Building a Logger

Let's create a proper logging utility:

```dart
// lib/logger.dart
import 'dart:io';
import 'package:tint/tint.dart';

enum LogLevel {
  debug,
  info,
  warning,
  error,
  success,
}

class Logger {
  final bool enableColors;
  final LogLevel minLevel;

  Logger({
    bool? enableColors,
    this.minLevel = LogLevel.info,
  }) : enableColors = enableColors ?? stdout.supportsAnsiEscapes;

  void debug(String message) => _log(LogLevel.debug, message);
  void info(String message) => _log(LogLevel.info, message);
  void warning(String message) => _log(LogLevel.warning, message);
  void error(String message) => _log(LogLevel.error, message);
  void success(String message) => _log(LogLevel.success, message);

  void _log(LogLevel level, String message) {
    if (level.index < minLevel.index) return;

    final prefix = _getPrefix(level);
    final formattedMessage = enableColors
        ? _colorize(level, '$prefix $message')
        : '$prefix $message';

    final output = level == LogLevel.error ? stderr : stdout;
    output.writeln(formattedMessage);
  }

  String _getPrefix(LogLevel level) {
    return switch (level) {
      LogLevel.debug => '[DEBUG]',
      LogLevel.info => '[INFO] ',
      LogLevel.warning => '[WARN] ',
      LogLevel.error => '[ERROR]',
      LogLevel.success => '[✓]    ',
    };
  }

  String _colorize(LogLevel level, String message) {
    return switch (level) {
      LogLevel.debug => message.dim(),
      LogLevel.info => message.cyan(),
      LogLevel.warning => message.yellow(),
      LogLevel.error => message.red().bold(),
      LogLevel.success => message.green(),
    };
  }
}

// Usage
void main() {
  final logger = Logger();

  logger.debug('Starting application...');
  logger.info('Loading configuration');
  logger.warning('Using default config');
  logger.success('Configuration loaded');
  logger.error('Failed to connect to database');
}
```

Output (imagine the colors):
```
[INFO]  Loading configuration
[WARN]  Using default config
[✓]     Configuration loaded
[ERROR] Failed to connect to database
```

Notice:
- Debug messages are hidden (below `minLevel`)
- Errors go to stderr
- Colors can be disabled
- Consistent formatting throughout

## Detecting Terminal Capabilities

Not all terminals support colors. Respect that:

```dart
import 'dart:io';

void main() {
  // Check if terminal supports ANSI codes
  if (stdout.supportsAnsiEscapes) {
    print('Colors work!'.green());
  } else {
    print('Colors work!');  // Plain text fallback
  }

  // Check if output is redirected to a file
  if (stdout.hasTerminal) {
    print('Interactive terminal'.blue());
  } else {
    print('Output is redirected');
  }
}
```

### The `NO_COLOR` Convention

Many users set a `NO_COLOR` environment variable to disable colors globally:

```dart
bool shouldUseColors() {
  // Respect NO_COLOR environment variable
  if (Platform.environment.containsKey('NO_COLOR')) {
    return false;
  }

  // Check terminal capabilities
  return stdout.supportsAnsiEscapes;
}

void main() {
  final useColors = shouldUseColors();

  final message = 'Hello, World!';
  print(useColors ? message.green() : message);
}
```

Users can now disable colors:

```bash
$ NO_COLOR=1 dart run bin/myapp.dart
```

## Practical Example: Syntax Highlighting

Let's build a simple JSON pretty-printer with syntax highlighting:

```dart
// bin/json_pretty.dart
import 'dart:io';
import 'dart:convert';
import 'package:tint/tint.dart';

Future<void> main(List<String> arguments) async {
  final useColors = !Platform.environment.containsKey('NO_COLOR') &&
      stdout.supportsAnsiEscapes;

  // Read JSON from stdin or file
  final String input;
  if (arguments.isEmpty) {
    input = await stdin.transform(utf8.decoder).join();
  } else {
    input = await File(arguments.first).readAsString();
  }

  // Parse JSON
  try {
    final data = jsonDecode(input);
    final formatted = _formatJson(data, useColors);
    print(formatted);
  } on FormatException catch (e) {
    stderr.writeln('Error: Invalid JSON'.red());
    stderr.writeln(e.message);
    exit(1);
  }
}

String _formatJson(dynamic data, bool useColors, [int indent = 0]) {
  final spaces = '  ' * indent;

  if (data is Map) {
    final buffer = StringBuffer();
    buffer.write('{');

    final entries = data.entries.toList();
    for (var i = 0; i < entries.length; i++) {
      final entry = entries[i];
      buffer.write('\n$spaces  ');

      // Key (in cyan)
      final key = '"${entry.key}"';
      buffer.write(useColors ? key.cyan() : key);
      buffer.write(': ');

      // Value (recursive)
      buffer.write(_formatJson(entry.value, useColors, indent + 1));

      if (i < entries.length - 1) buffer.write(',');
    }

    buffer.write('\n$spaces}');
    return buffer.toString();
  } else if (data is List) {
    if (data.isEmpty) return '[]';

    final buffer = StringBuffer();
    buffer.write('[');

    for (var i = 0; i < data.length; i++) {
      buffer.write('\n$spaces  ');
      buffer.write(_formatJson(data[i], useColors, indent + 1));
      if (i < data.length - 1) buffer.write(',');
    }

    buffer.write('\n$spaces]');
    return buffer.toString();
  } else if (data is String) {
    final quoted = '"$data"';
    return useColors ? quoted.green() : quoted;
  } else if (data is num) {
    final text = data.toString();
    return useColors ? text.yellow() : text;
  } else if (data is bool) {
    final text = data.toString();
    return useColors ? text.magenta() : text;
  } else if (data == null) {
    final text = 'null';
    return useColors ? text.dim() : text;
  }

  return data.toString();
}
```

Usage:

```bash
$ echo '{"name":"Alice","age":30,"active":true}' | dart run bin/json_pretty.dart
{
  "name": "Alice",
  "age": 30,
  "active": true
}

# With NO_COLOR
$ NO_COLOR=1 echo '{"name":"Alice"}' | dart run bin/json_pretty.dart
```

The output has:
- Cyan keys
- Green strings
- Yellow numbers
- Magenta booleans
- Dimmed null values

## Building a Status Display

For long-running operations, show status with colors:

```dart
import 'dart:io';
import 'package:tint/tint.dart';

class StatusLine {
  final bool useColors;

  StatusLine({bool? useColors})
      : useColors = useColors ?? stdout.supportsAnsiEscapes;

  void show(String status, String message) {
    final icon = _getIcon(status);
    final coloredIcon = useColors ? _colorize(status, icon) : icon;
    stdout.writeln('$coloredIcon $message');
  }

  String _getIcon(String status) {
    return switch (status) {
      'pending' => '○',
      'running' => '◐',
      'success' => '✓',
      'error' => '✗',
      'warning' => '⚠',
      _ => '•',
    };
  }

  String _colorize(String status, String text) {
    return switch (status) {
      'pending' => text.dim(),
      'running' => text.cyan(),
      'success' => text.green(),
      'error' => text.red(),
      'warning' => text.yellow(),
      _ => text,
    };
  }
}

void main() async {
  final status = StatusLine();

  status.show('pending', 'Connecting to server');
  await Future.delayed(Duration(seconds: 1));

  status.show('running', 'Downloading data');
  await Future.delayed(Duration(seconds: 2));

  status.show('success', 'Data downloaded');
  status.show('running', 'Processing data');
  await Future.delayed(Duration(seconds: 1));

  status.show('warning', 'Some records skipped');
  status.show('success', 'Processing complete');
}
```

Output:
```
○ Connecting to server
◐ Downloading data
✓ Data downloaded
◐ Processing data
⚠ Some records skipped
✓ Processing complete
```

(With appropriate colors for each line!)

## Tables with Colors

Want to display tabular data? Add colors for readability:

```dart
import 'package:tint/tint.dart';

void printTable(List<Map<String, String>> data, {bool useColors = true}) {
  if (data.isEmpty) return;

  // Get headers
  final headers = data.first.keys.toList();

  // Calculate column widths
  final widths = <String, int>{};
  for (final header in headers) {
    widths[header] = header.length;
    for (final row in data) {
      final value = row[header] ?? '';
      if (value.length > widths[header]!) {
        widths[header] = value.length;
      }
    }
  }

  // Print header
  final headerRow = headers
      .map((h) => h.padRight(widths[h]!))
      .join(' │ ');
  print(useColors ? headerRow.bold().cyan() : headerRow);

  // Print separator
  final separator = headers
      .map((h) => '─' * widths[h]!)
      .join('─┼─');
  print(useColors ? separator.dim() : separator);

  // Print rows
  for (final row in data) {
    final rowStr = headers
        .map((h) => (row[h] ?? '').padRight(widths[h]!))
        .join(' │ ');
    print(rowStr);
  }
}

void main() {
  final data = [
    {'Name': 'Alice', 'Age': '30', 'Status': 'Active'},
    {'Name': 'Bob', 'Age': '25', 'Status': 'Inactive'},
    {'Name': 'Charlie', 'Age': '35', 'Status': 'Active'},
  ];

  printTable(data);
}
```

Output:
```
Name    │ Age │ Status
────────┼─────┼─────────
Alice   │ 30  │ Active
Bob     │ 25  │ Inactive
Charlie │ 35  │ Active
```

(Header in bold cyan, separator dimmed)

## Color Schemes: Dark vs Light Terminals

Some users use light terminal backgrounds. Be considerate:

```dart
class ColorScheme {
  final String error;
  final String warning;
  final String success;
  final String info;

  const ColorScheme({
    required this.error,
    required this.warning,
    required this.success,
    required this.info,
  });

  static const dark = ColorScheme(
    error: 'red',
    warning: 'yellow',
    success: 'green',
    info: 'cyan',
  );

  static const light = ColorScheme(
    error: 'brightRed',
    warning: 'brightYellow',
    success: 'brightGreen',
    info: 'brightCyan',
  );
}

// Detect terminal background (heuristic)
ColorScheme detectColorScheme() {
  final colorfgbg = Platform.environment['COLORFGBG'];
  if (colorfgbg != null && colorfgbg.endsWith('15')) {
    return ColorScheme.light;
  }
  return ColorScheme.dark;
}
```

Or just let users configure it themselves!

## Best Practices

1. **Always provide a way to disable colors** (respect `NO_COLOR`)
2. **Check terminal capabilities** (`stdout.supportsAnsiEscapes`)
3. **Use semantic colors** (success = green, error = red)
4. **Don't overdo it** — too many colors is visual noise
5. **Test in different terminals** (some support more colors than others)
6. **Provide plain-text fallbacks** (for piped output)
7. **Use extension methods for clean code** (`message.red()` vs manual codes)
8. **Consider accessibility** (some users are colorblind)

## Accessibility Considerations

Colors aren't enough:

```dart
// ❌ Bad: relies only on color
print('Error'.red());
print('Success'.green());

// ✅ Good: color + icon/text
print('✗ Error'.red());
print('✓ Success'.green());

// ✅ Also good: explicit label
print('[ERROR]'.red().bold() + ' Connection failed');
print('[SUCCESS]'.green() + ' File saved');
```

Colorblind users (8% of males!) can still understand the message.

## What's Next?

Your CLI tools can now be beautiful! You've learned:
- How to add colors with `tint`
- Text formatting (bold, underline, etc.)
- Detecting terminal capabilities
- Respecting user preferences (`NO_COLOR`)
- Building semantic loggers
- Creating colored tables and status displays

In the next chapter, we'll make your tools *interactive* — asking questions, getting user input, and building conversational CLIs.

Time to have a chat with your users!

---

**[← Previous: Files and Pipes](03-files-and-pipes.md)** | **[Next: Interactive Prompts →](05-interactive-prompts.md)**
