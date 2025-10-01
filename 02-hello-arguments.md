# Chapter 2: Hello, Arguments!

**Parsing arguments without losing your mind**

Every CLI application starts the same way: someone runs it with a bunch of arguments, and you need to figure out what they want. Simple, right?

```bash
$ mytool --input file.txt --output result.txt --verbose --format json
```

Wrong! Argument parsing is deceptively complex:

- Is it `--verbose` or `-v`? Or both?
- What if they write `--format=json` instead of `--format json`?
- Should `--help` show up before processing other arguments?
- What about subcommands like `git commit` vs `git push`?
- How do you handle invalid input gracefully?

You *could* parse `List<String> arguments` manually with a bunch of `if` statements. People did this in the '80s. They also had mullets. We've evolved.

Let's use Dart's `args` package and do this properly.

## Meet the `args` Package

The `args` package is the standard way to parse command-line arguments in Dart. It's:

- **Declarative**: Describe what you want, not how to parse it
- **Flexible**: Handles flags, options, commands, subcommands
- **Helpful**: Generates usage text automatically
- **Type-safe**: Returns parsed values with proper types

First, let's add it to a project:

```bash
# Create a new Dart CLI project
$ dart create -t console-full greeter
$ cd greeter

# Add the args package
$ dart pub add args
```

Your `pubspec.yaml` now includes:

```yaml
dependencies:
  args: ^2.4.2  # Version may vary
```

## Flags vs Options: A Love Story

Before we dive in, let's clarify terminology:

**Flags** are boolean switches that are either on or off:
```bash
$ mytool --verbose    # flag is ON
$ mytool              # flag is OFF (default)
```

**Options** take values:
```bash
$ mytool --output results.txt  # option with value
$ mytool -o results.txt         # same, with abbreviation
```

**Positional arguments** don't have names:
```bash
$ mytool input.txt output.txt  # two positional args
```

Got it? Good. Let's build something.

## Example: The Greeting Tool

We'll build a `greeter` tool that demonstrates all the key concepts. It will support:

- Different greeting styles (hello, hi, hey)
- Custom names
- Excitement levels
- Multiple output formats
- A `--help` message that doesn't make users cry

### Version 1: Basic Flags and Options

Let's start simple:

```dart
// bin/greeter.dart
import 'package:args/args.dart';

void main(List<String> arguments) {
  final parser = ArgParser()
    ..addOption('name',
        abbr: 'n',
        defaultsTo: 'World',
        help: 'The name to greet')
    ..addOption('greeting',
        abbr: 'g',
        defaultsTo: 'Hello',
        help: 'The greeting to use',
        allowed: ['Hello', 'Hi', 'Hey', 'Greetings'])
    ..addFlag('excited',
        abbr: 'e',
        negatable: false,
        help: 'Add excitement with !!!')
    ..addFlag('help',
        abbr: 'h',
        negatable: false,
        help: 'Show this help message');

  final ArgResults results;
  try {
    results = parser.parse(arguments);
  } on FormatException catch (e) {
    // Parsing failed
    print('Error: ${e.message}');
    print('');
    print('Usage: greeter [options]');
    print(parser.usage);
    return;
  }

  // Show help if requested
  if (results['help'] as bool) {
    print('Usage: greeter [options]');
    print('');
    print('Options:');
    print(parser.usage);
    return;
  }

  // Build the greeting
  final name = results['name'] as String;
  final greeting = results['greeting'] as String;
  final excited = results['excited'] as bool;

  var message = '$greeting, $name';
  if (excited) {
    message += '!!!';
  } else {
    message += '.';
  }

  print(message);
}
```

Let's test it:

```bash
$ dart run bin/greeter.dart
Hello, World.

$ dart run bin/greeter.dart --name Alice
Hello, Alice.

$ dart run bin/greeter.dart -n Bob -g Hey -e
Hey, Bob!!!

$ dart run bin/greeter.dart --help
Usage: greeter [options]

Options:
-n, --name          The name to greet
                    (defaults to "World")
-g, --greeting      The greeting to use
                    [Hello, Hi, Hey, Greetings]
                    (defaults to "Hello")
-e, --excited       Add excitement with !!!
-h, --help          Show this help message
```

Beautiful! Notice a few things:

1. **Abbreviations** (`-n`) work automatically
2. **Default values** are handled for us
3. **Help text** is auto-generated
4. **Validation** happens automatically (try `--greeting Yo`)
5. **Type casting** is explicit but simple

### Understanding `negatable`

You might have noticed `negatable: false` on our flags. Let's talk about that.

By default, flags in Dart are *negatable*, meaning you can turn them off explicitly:

```dart
..addFlag('verbose', defaultsTo: true)  // negatable by default
```

This creates TWO flags:
```bash
$ mytool --verbose      # Set to true
$ mytool --no-verbose   # Set to false explicitly
$ mytool                # Uses default (true)
```

Sometimes this is useful! But often, you want a simple on/off flag without the `--no-` version. That's when you use `negatable: false`.

### The `allowed` List

Options can be restricted to specific values:

```dart
..addOption('format',
    allowed: ['json', 'yaml', 'toml'],
    help: 'Output format')
```

If the user provides an invalid value, they get a helpful error:

```bash
$ greeter --format xml
Error: "xml" is not an allowed value for option "format"
```

The `allowed` list also shows up in the help text automatically. Nice!

## Positional Arguments

Sometimes you want arguments without names:

```bash
$ greeter Alice     # Simpler than --name Alice
$ cp source.txt dest.txt   # Two positional args
```

After parsing named arguments, leftover arguments are available in `results.rest`:

```dart
void main(List<String> arguments) {
  final parser = ArgParser()
    ..addFlag('excited', abbr: 'e');

  final results = parser.parse(arguments);
  final excited = results['excited'] as bool;

  // Positional args are in results.rest
  if (results.rest.isEmpty) {
    print('Error: Please provide a name');
    return;
  }

  final name = results.rest.first;
  var message = 'Hello, $name';
  if (excited) message += '!';

  print(message);
}
```

Usage:
```bash
$ dart run bin/greeter.dart Alice
Hello, Alice

$ dart run bin/greeter.dart Alice -e
Hello, Alice!

$ dart run bin/greeter.dart
Error: Please provide a name
```

## Commands and Subcommands

Real tools often have multiple commands, like Git:

```bash
$ git commit -m "message"
$ git push origin main
$ git log --oneline
```

The `args` package handles this beautifully with `ArgParser.addCommand()`:

```dart
// bin/tool.dart
import 'package:args/args.dart';

void main(List<String> arguments) {
  final parser = ArgParser();

  // Add a 'greet' command
  final greetCommand = parser.addCommand('greet')
    ..addOption('name', abbr: 'n', defaultsTo: 'World')
    ..addFlag('excited', abbr: 'e');

  // Add a 'farewell' command
  final farewellCommand = parser.addCommand('farewell')
    ..addOption('name', abbr: 'n', defaultsTo: 'friend')
    ..addFlag('sad', abbr: 's', help: 'Add sadness :(');

  // Add global help flag
  parser.addFlag('help', abbr: 'h', negatable: false);

  final results = parser.parse(arguments);

  if (results['help'] as bool || results.command == null) {
    print('Usage: tool <command> [options]');
    print('');
    print('Commands:');
    print('  greet       Say hello');
    print('  farewell    Say goodbye');
    print('');
    print('Run "tool <command> --help" for more info on a command');
    return;
  }

  // Handle the command
  switch (results.command!.name) {
    case 'greet':
      handleGreet(results.command!);
      break;
    case 'farewell':
      handleFarewell(results.command!);
      break;
  }
}

void handleGreet(ArgResults results) {
  final name = results['name'] as String;
  final excited = results['excited'] as bool;

  var message = 'Hello, $name';
  if (excited) message += '!!!';
  print(message);
}

void handleFarewell(ArgResults results) {
  final name = results['name'] as String;
  final sad = results['sad'] as bool;

  var message = 'Goodbye, $name';
  if (sad) message += ' :(';
  print(message);
}
```

Now we have a multi-command tool:

```bash
$ dart run bin/tool.dart greet --name Alice -e
Hello, Alice!!!

$ dart run bin/tool.dart farewell -n Bob --sad
Goodbye, Bob :(

$ dart run bin/tool.dart --help
Usage: tool <command> [options]

Commands:
  greet       Say hello
  farewell    Say goodbye

Run "tool <command> --help" for more info on a command
```

### Nested Subcommands

You can nest commands multiple levels deep (like `git remote add`):

```dart
final remoteCommand = parser.addCommand('remote');
remoteCommand.addCommand('add')
  ..addOption('url', mandatory: true);
remoteCommand.addCommand('remove');
```

But be careful — deeply nested commands can confuse users. Keep it to 2-3 levels max.

## Advanced Patterns

### Mandatory Options

Sometimes an option is required:

```dart
..addOption('output',
    abbr: 'o',
    mandatory: true,
    help: 'Output file (required)')
```

If the user doesn't provide it, they get an error:

```bash
$ mytool
Error: Option output is mandatory.
```

### Multiple Values

Some options can be specified multiple times:

```dart
..addMultiOption('include',
    abbr: 'i',
    help: 'Files to include (can be used multiple times)')
```

Usage:
```bash
$ mytool -i file1.txt -i file2.txt -i file3.txt
```

Access as a list:
```dart
final includes = results['include'] as List<String>;
// ['file1.txt', 'file2.txt', 'file3.txt']
```

Or comma-separated:
```bash
$ mytool -i file1.txt,file2.txt,file3.txt
```

Both work! The `args` package handles splitting automatically.

### Custom Value Parsing

Want to parse something more complex than strings?

```dart
..addOption('count', defaultsTo: '10')
```

Then parse it yourself:
```dart
final count = int.tryParse(results['count'] as String) ?? 10;
if (count < 1) {
  print('Error: count must be positive');
  return;
}
```

Or write a helper:
```dart
int getIntOption(ArgResults results, String name, int defaultValue) {
  final value = results[name] as String;
  final parsed = int.tryParse(value);
  if (parsed == null) {
    throw FormatException('Option $name must be an integer');
  }
  return parsed;
}

// Usage:
final count = getIntOption(results, 'count', 10);
```

## Real-World Example: File Converter

Let's build something more realistic — a file format converter:

```dart
// bin/convert.dart
import 'dart:io';
import 'package:args/args.dart';

void main(List<String> arguments) async {
  final parser = ArgParser()
    ..addOption('input',
        abbr: 'i',
        mandatory: true,
        help: 'Input file')
    ..addOption('output',
        abbr: 'o',
        help: 'Output file (default: stdout)')
    ..addOption('format',
        abbr: 'f',
        allowed: ['json', 'yaml', 'csv'],
        defaultsTo: 'json',
        help: 'Output format')
    ..addFlag('pretty',
        abbr: 'p',
        help: 'Pretty-print output')
    ..addFlag('help',
        abbr: 'h',
        negatable: false);

  ArgResults results;
  try {
    results = parser.parse(arguments);
  } on FormatException catch (e) {
    stderr.writeln('Error: ${e.message}');
    stderr.writeln('');
    printUsage(parser);
    exit(1);
  }

  if (results['help'] as bool) {
    printUsage(parser);
    exit(0);
  }

  // Extract options
  final inputPath = results['input'] as String;
  final outputPath = results['output'] as String?;
  final format = results['format'] as String;
  final pretty = results['pretty'] as bool;

  // Validate input file exists
  final inputFile = File(inputPath);
  if (!await inputFile.exists()) {
    stderr.writeln('Error: Input file not found: $inputPath');
    exit(1);
  }

  // Read and convert (simplified for example)
  final content = await inputFile.readAsString();
  final converted = convertFormat(content, format, pretty);

  // Write to output or stdout
  if (outputPath != null) {
    await File(outputPath).writeAsString(converted);
    print('Converted to $outputPath');
  } else {
    print(converted);
  }
}

void printUsage(ArgParser parser) {
  print('Usage: convert [options]');
  print('');
  print('Convert between file formats');
  print('');
  print('Options:');
  print(parser.usage);
  print('');
  print('Examples:');
  print('  convert -i data.json -o data.yaml -f yaml');
  print('  convert -i data.json -f csv --pretty');
}

String convertFormat(String content, String format, bool pretty) {
  // Actual conversion logic would go here
  return 'Converted to $format (pretty: $pretty)\n$content';
}
```

This demonstrates:
- ✅ Mandatory options
- ✅ Optional options
- ✅ Validation (file exists)
- ✅ Error messages to stderr
- ✅ Proper exit codes
- ✅ Helpful usage examples

## Best Practices

After building dozens of CLI tools, here's what I've learned:

### 1. Always Provide `--help`

Every tool should have a help flag. Users will thank you.

```dart
..addFlag('help', abbr: 'h', negatable: false, help: 'Show this help')
```

Check it before doing anything else:
```dart
if (results['help'] as bool) {
  printUsage(parser);
  exit(0);
}
```

### 2. Make Help Text Useful

Include:
- Brief description of what the tool does
- List of options with descriptions
- Examples of common usage

```dart
void printUsage(ArgParser parser) {
  print('mytool - Does something useful');
  print('');
  print('Usage: mytool [options] <file>');
  print('');
  print(parser.usage);
  print('');
  print('Examples:');
  print('  mytool input.txt');
  print('  mytool -v --output result.txt input.txt');
}
```

### 3. Validate Early, Fail Fast

Check for errors before doing any work:

```dart
// Validate required files exist
if (!await File(inputPath).exists()) {
  stderr.writeln('Error: File not found: $inputPath');
  exit(1);
}

// Validate ranges
if (count < 1 || count > 100) {
  stderr.writeln('Error: count must be between 1 and 100');
  exit(1);
}
```

### 4. Use `stderr` for Errors

```dart
// ❌ Don't do this:
print('Error: something went wrong');

// ✅ Do this:
stderr.writeln('Error: something went wrong');
```

Why? Users can redirect stdout without seeing error messages mixed in.

### 5. Support Both Long and Short Forms

```dart
..addOption('output', abbr: 'o')  // Supports both --output and -o
```

Power users love short forms. New users appreciate long forms being self-documenting.

### 6. Provide Sensible Defaults

```dart
..addOption('timeout',
    defaultsTo: '30',
    help: 'Timeout in seconds (default: 30)')
```

Most users shouldn't need to configure everything.

### 7. Exit with Proper Codes

```dart
import 'dart:io';

// Success
exit(0);

// Generic error
exit(1);

// Specific errors (optional, but helpful in scripts)
exit(2);  // Misuse of command (invalid arguments)
exit(127); // Command not found
```

Scripts can check `$?` to see what went wrong.

## Common Pitfalls

### Pitfall 1: Forgetting to Handle `rest`

```dart
final results = parser.parse(arguments);

// ❌ Oops, what if there are extra arguments?
final name = results['name'] as String;

// ✅ Check for unexpected arguments:
if (results.rest.isNotEmpty) {
  stderr.writeln('Error: Unexpected arguments: ${results.rest.join(' ')}');
  exit(1);
}
```

### Pitfall 2: Not Catching `FormatException`

```dart
// ❌ This will crash with an ugly stack trace
final results = parser.parse(arguments);

// ✅ Catch parsing errors gracefully
try {
  final results = parser.parse(arguments);
} on FormatException catch (e) {
  stderr.writeln('Error: ${e.message}');
  printUsage(parser);
  exit(1);
}
```

### Pitfall 3: Unclear Error Messages

```dart
// ❌ Cryptic
stderr.writeln('Invalid input');

// ✅ Helpful
stderr.writeln('Error: Input file must be a JSON file, got: $inputPath');
stderr.writeln('Try: mytool --input data.json');
```

## What's Next?

You now know how to parse arguments like a pro! Your tools can handle:
- Flags and options
- Abbreviations
- Validation
- Commands and subcommands
- Helpful error messages

In the next chapter, we'll tackle files and pipes — reading from stdin, writing to stdout, and playing nicely with other Unix tools in a pipeline.

But first, **try building something**:

**Exercise**: Create a `calc` tool that:
- Takes two numbers as positional arguments
- Has an `--operation` option (add, subtract, multiply, divide)
- Has a `--precision` option for decimal places
- Handles division by zero gracefully
- Shows helpful error messages

Solution in the next chapter... or build it yourself first!

---

**[← Previous: Why Dart for CLI?](01-why-dart-for-cli.md)** | **[Next: Files and Pipes →](03-files-and-pipes.md)**
