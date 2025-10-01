# Chapter 5: Interactive Prompts

**Talking to humans (scary, I know)**

Not every CLI tool can work purely from arguments. Sometimes you need to ask the user questions:

```bash
$ npm init
package name: (my-project) awesome-app
version: (1.0.0)
description: A really cool app
...
```

This is more user-friendly than forcing users to remember 15 different flags. It's a conversation, not a command.

In this chapter, we'll build interactive prompts that are intuitive, forgiving, and don't make users want to Ctrl+C out of frustration.

## The Simple Way: `stdin.readLineSync()`

Dart's built-in approach:

```dart
import 'dart:io';

void main() {
  stdout.write('What is your name? ');
  final name = stdin.readLineSync();

  stdout.write('How old are you? ');
  final ageStr = stdin.readLineSync();
  final age = int.tryParse(ageStr ?? '');

  if (name != null && age != null) {
    print('Hello, $name! You are $age years old.');
  } else {
    print('Invalid input!');
  }
}
```

This works, but it's bare-bones:
- No default values shown
- No validation until after input
- No masked password input
- No list selection
- Lots of manual parsing

We can do better.

## Meet `interact`: Interactive Prompts Done Right

The `interact` package provides a clean API for common prompt patterns:

```bash
$ dart pub add interact
```

Let's rebuild our example:

```dart
import 'package:interact/interact.dart';

void main() {
  final name = Input(
    prompt: 'What is your name?',
    defaultValue: 'Anonymous',
  ).interact();

  final age = Input(
    prompt: 'How old are you?',
    defaultValue: '25',
    validator: (String x) {
      final number = int.tryParse(x);
      return number != null && number > 0 && number < 150;
    },
  ).interact();

  print('Hello, $name! You are $age years old.');
}
```

Better! Now we have:
- ✅ Default values
- ✅ Validation
- ✅ Clear prompts

## Types of Prompts

### 1. Text Input

Basic text input with optional validation:

```dart
final email = Input(
  prompt: 'Email address',
  validator: (String x) => x.contains('@'),
).interact();
```

With a default value:

```dart
final username = Input(
  prompt: 'Username',
  defaultValue: 'user123',
).interact();
```

Custom error message:

```dart
final port = Input(
  prompt: 'Port number',
  defaultValue: '8080',
  validator: (String x) {
    final num = int.tryParse(x);
    return num != null && num > 0 && num < 65536;
  },
).interact();
```

### 2. Confirmation (Yes/No)

Ask yes/no questions:

```dart
final confirmed = Confirm(
  prompt: 'Do you want to continue?',
  defaultValue: true,  // [Y/n]
).interact();

if (confirmed) {
  print('Continuing...');
} else {
  print('Cancelled.');
  exit(0);
}
```

This shows `[Y/n]` for default=true, or `[y/N]` for default=false. Users can type `y`, `yes`, `n`, `no`, or just press Enter for the default.

### 3. Select from a List

Single selection:

```dart
final flavor = Select(
  prompt: 'Choose a flavor',
  options: ['Vanilla', 'Chocolate', 'Strawberry'],
).interact();

print('You selected: $flavor');
```

The user sees:

```
Choose a flavor
❯ Vanilla
  Chocolate
  Strawberry
```

They can use arrow keys to navigate and Enter to select. Much nicer than typing!

With a default selected:

```dart
final framework = Select(
  prompt: 'Choose a framework',
  options: ['React', 'Vue', 'Angular', 'Svelte'],
  initialIndex: 0,  // React is pre-selected
).interact();
```

### 4. Multi-Select

Select multiple items:

```dart
final features = MultiSelect(
  prompt: 'Which features do you want?',
  options: ['Authentication', 'Database', 'API', 'Admin Panel'],
).interact();

print('Selected features: ${features.join(', ')}');
```

Users press Space to toggle items, Enter when done:

```
Which features do you want?
  ◯ Authentication
  ◉ Database
  ◉ API
  ◯ Admin Panel
```

### 5. Password Input

Hide input for sensitive data:

```dart
// Note: interact doesn't have built-in password masking,
// but we can use dart:io directly with stdin.echoMode
import 'dart:io';

String readPassword(String prompt) {
  stdout.write('$prompt: ');
  stdin.echoMode = false;  // Hide input
  final password = stdin.readLineSync() ?? '';
  stdin.echoMode = true;   // Restore echo
  stdout.writeln();  // New line after hidden input
  return password;
}

void main() {
  final password = readPassword('Enter password');
  final confirm = readPassword('Confirm password');

  if (password == confirm) {
    print('Password set!');
  } else {
    print('Passwords do not match!');
    exit(1);
  }
}
```

## Building a Project Scaffolder

Let's combine these concepts into something useful — a tool that scaffolds new projects:

```dart
// bin/scaffold.dart
import 'dart:io';
import 'package:interact/interact.dart';
import 'package:path/path.dart' as path;

void main() async {
  print('=== Project Scaffolder ===\n');

  // Project name
  final projectName = Input(
    prompt: 'Project name',
    defaultValue: 'my-project',
    validator: (String x) => RegExp(r'^[a-z0-9-]+$').hasMatch(x),
  ).interact();

  // Description
  final description = Input(
    prompt: 'Description',
    defaultValue: 'A new project',
  ).interact();

  // Language/Framework
  final framework = Select(
    prompt: 'Framework',
    options: ['Dart Console', 'Dart Web', 'Dart Server', 'Flutter'],
  ).interact();

  // Features
  final features = MultiSelect(
    prompt: 'Additional features',
    options: ['Testing', 'Linting', 'CI/CD', 'Docker'],
  ).interact();

  // Git initialization
  final initGit = Confirm(
    prompt: 'Initialize git repository?',
    defaultValue: true,
  ).interact();

  // Confirm
  print('\n=== Summary ===');
  print('Name:        $projectName');
  print('Description: $description');
  print('Framework:   $framework');
  print('Features:    ${features.join(', ')}');
  print('Git:         ${initGit ? 'Yes' : 'No'}');
  print('');

  final confirm = Confirm(
    prompt: 'Create project?',
    defaultValue: true,
  ).interact();

  if (!confirm) {
    print('Cancelled.');
    return;
  }

  // Create the project
  await createProject(
    name: projectName,
    description: description,
    framework: framework,
    features: features,
    initGit: initGit,
  );

  print('\n✓ Project created successfully!'.green());
  print('\nNext steps:');
  print('  cd $projectName');
  print('  dart pub get');
  print('  dart run');
}

Future<void> createProject({
  required String name,
  required String description,
  required String framework,
  required List<String> features,
  required bool initGit,
}) async {
  final dir = Directory(name);

  // Create directory
  await dir.create();

  // Create pubspec.yaml
  final pubspec = '''
name: $name
description: $description
version: 1.0.0

environment:
  sdk: ^3.0.0

dependencies:
  # Add dependencies here

dev_dependencies:
  ${features.contains('Linting') ? 'lints: ^2.0.0' : '# No linting'}
  ${features.contains('Testing') ? 'test: ^1.24.0' : '# No testing'}
''';

  await File(path.join(name, 'pubspec.yaml')).writeAsString(pubspec);

  // Create main file
  final mainContent = '''
void main() {
  print('Hello from $name!');
}
''';

  await Directory(path.join(name, 'bin')).create();
  await File(path.join(name, 'bin', 'main.dart')).writeAsString(mainContent);

  // Create README
  final readme = '''
# $name

$description

## Getting Started

\`\`\`bash
dart pub get
dart run
\`\`\`
''';

  await File(path.join(name, 'README.md')).writeAsString(readme);

  // Initialize git
  if (initGit) {
    await Process.run('git', ['init'], workingDirectory: name);
    await File(path.join(name, '.gitignore')).writeAsString('''
.dart_tool/
.packages
pubspec.lock
''');
  }

  // CI/CD
  if (features.contains('CI/CD')) {
    await Directory(path.join(name, '.github', 'workflows')).create(recursive: true);
    await File(path.join(name, '.github', 'workflows', 'dart.yml'))
        .writeAsString('''
name: Dart CI

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: dart-lang/setup-dart@v1
      - run: dart pub get
      - run: dart analyze
      ${features.contains('Testing') ? '- run: dart test' : ''}
''');
  }

  // Docker
  if (features.contains('Docker')) {
    await File(path.join(name, 'Dockerfile')).writeAsString('''
FROM dart:stable AS build
WORKDIR /app
COPY . .
RUN dart pub get
RUN dart compile exe bin/main.dart -o bin/server

FROM scratch
COPY --from=build /app/bin/server /app/bin/server
CMD ["/app/bin/server"]
''');
  }
}
```

This creates a fully interactive project setup experience, similar to `npm init` or `cargo new`.

## Input Validation Patterns

Good validation helps users provide correct input:

### Email Validation

```dart
final email = Input(
  prompt: 'Email',
  validator: (String x) =>
      RegExp(r'^[\w-\.]+@([\w-]+\.)+[\w-]{2,4}$').hasMatch(x),
).interact();
```

### URL Validation

```dart
final url = Input(
  prompt: 'Website URL',
  validator: (String x) => Uri.tryParse(x)?.hasScheme ?? false,
).interact();
```

### Number Range

```dart
final age = Input(
  prompt: 'Age',
  validator: (String x) {
    final num = int.tryParse(x);
    return num != null && num >= 0 && num <= 120;
  },
).interact();
```

### Non-Empty

```dart
final name = Input(
  prompt: 'Name',
  validator: (String x) => x.trim().isNotEmpty,
).interact();
```

### File Exists

```dart
final filePath = Input(
  prompt: 'Input file',
  validator: (String x) => File(x).existsSync(),
).interact();
```

## Handling Errors Gracefully

When validation fails, provide helpful messages:

```dart
String? validatePort(String input) {
  final port = int.tryParse(input);

  if (port == null) {
    return 'Port must be a number';
  }

  if (port < 1 || port > 65535) {
    return 'Port must be between 1 and 65535';
  }

  return null;  // Valid
}

// Custom validation with error messages
String getValidatedInput(String prompt, String? Function(String) validator) {
  while (true) {
    stdout.write('$prompt: ');
    final input = stdin.readLineSync() ?? '';

    final error = validator(input);
    if (error == null) {
      return input;  // Valid!
    }

    stderr.writeln('Error: $error'.red());
  }
}

void main() {
  final port = getValidatedInput('Port', validatePort);
  print('Using port: $port');
}
```

## Default Values: Do's and Don'ts

Good defaults make tools faster to use:

```dart
// ✅ Good: sensible default
final port = Input(
  prompt: 'Port',
  defaultValue: '8080',  // Most web apps
).interact();

// ✅ Good: detect from environment
final defaultName = Platform.environment['USER'] ?? 'user';
final author = Input(
  prompt: 'Author name',
  defaultValue: defaultName,
).interact();

// ❌ Bad: no clear default
final timeout = Input(
  prompt: 'Timeout in milliseconds',
  // No default — user has to know what's reasonable
).interact();

// ✅ Better
final timeout = Input(
  prompt: 'Timeout (ms)',
  defaultValue: '5000',  // 5 seconds
).interact();
```

## Progressive Disclosure

Don't overwhelm users with too many questions. Ask the basics first:

```dart
void main() {
  // Essential questions
  final projectName = Input(prompt: 'Project name').interact();
  final framework = Select(
    prompt: 'Framework',
    options: ['Web', 'CLI', 'Server'],
  ).interact();

  // Advanced options (optional)
  final advanced = Confirm(
    prompt: 'Configure advanced options?',
    defaultValue: false,
  ).interact();

  if (advanced) {
    final port = Input(
      prompt: 'Port',
      defaultValue: '8080',
    ).interact();

    final features = MultiSelect(
      prompt: 'Features',
      options: ['Auth', 'Database', 'Caching', 'Monitoring'],
    ).interact();

    // ... more options
  }

  // Create project with provided settings
  createProject(name: projectName, framework: framework);
}
```

## Keyboard Shortcuts and UX

Users expect certain shortcuts:

- **Ctrl+C**: Cancel/abort (handled automatically)
- **Enter**: Confirm default value
- **Arrow keys**: Navigate lists (handled by `interact`)
- **Space**: Toggle multi-select items (handled by `interact`)
- **Escape**: Go back (implement yourself if needed)

## Conditional Prompts

Skip irrelevant questions based on previous answers:

```dart
void main() {
  final useDatabase = Confirm(
    prompt: 'Use a database?',
    defaultValue: true,
  ).interact();

  if (useDatabase) {
    final dbType = Select(
      prompt: 'Database type',
      options: ['PostgreSQL', 'MySQL', 'SQLite', 'MongoDB'],
    ).interact();

    if (dbType != 'SQLite') {
      // SQLite doesn't need connection info
      final host = Input(
        prompt: 'Database host',
        defaultValue: 'localhost',
      ).interact();

      final port = Input(
        prompt: 'Database port',
        defaultValue: dbType == 'PostgreSQL' ? '5432' : '3306',
      ).interact();
    }
  }

  // Continue with setup...
}
```

## Spinner While Processing

Show a spinner after input while processing:

```dart
import 'package:cli_util/cli_logging.dart';

void main() async {
  final logger = Logger.standard();

  final projectName = Input(prompt: 'Project name').interact();

  final progress = logger.progress('Creating project');

  // Simulate work
  await Future.delayed(Duration(seconds: 2));

  progress.finish(message: 'Project created!');
}
```

## Testing Interactive Prompts

Interactive prompts are tricky to test. Here's a pattern:

```dart
// lib/prompts.dart
class ProjectConfig {
  final String name;
  final String description;
  final bool useGit;

  ProjectConfig({
    required this.name,
    required this.description,
    required this.useGit,
  });
}

// For testing: accept input as parameters
ProjectConfig getProjectConfig({
  String? name,
  String? description,
  bool? useGit,
}) {
  return ProjectConfig(
    name: name ?? Input(prompt: 'Project name').interact(),
    description: description ?? Input(prompt: 'Description').interact(),
    useGit: useGit ?? Confirm(prompt: 'Use git?').interact(),
  );
}

// In tests
void main() {
  test('creates project config', () {
    final config = getProjectConfig(
      name: 'test-project',
      description: 'Test description',
      useGit: true,
    );

    expect(config.name, 'test-project');
    expect(config.description, 'Test description');
    expect(config.useGit, true);
  });
}
```

## Best Practices

1. **Provide sensible defaults** — most users should be able to press Enter repeatedly
2. **Validate early** — catch errors before processing
3. **Give helpful error messages** — explain what's wrong and how to fix it
4. **Use the right prompt type** — Select is better than freeform text when options are limited
5. **Show what you're doing** — spinner/progress during operations
6. **Allow Ctrl+C** — always let users bail out
7. **Keep it conversational** — write prompts like you're talking to a friend
8. **Confirm destructive actions** — "Are you sure you want to delete everything?"
9. **Support both interactive and non-interactive modes** — provide CLI flags for automation

## Non-Interactive Mode

Always support non-interactive usage for scripts/CI:

```dart
import 'dart:io';
import 'package:args/args.dart';
import 'package:interact/interact.dart';

void main(List<String> arguments) {
  final parser = ArgParser()
    ..addOption('name')
    ..addOption('description')
    ..addFlag('git');

  final results = parser.parse(arguments);

  // Use flags if provided, otherwise prompt
  final name = results['name'] as String? ??
      Input(prompt: 'Project name').interact();

  final description = results['description'] as String? ??
      Input(prompt: 'Description', defaultValue: '').interact();

  final useGit = results.wasParsed('git')
      ? results['git'] as bool
      : Confirm(prompt: 'Use git?', defaultValue: true).interact();

  print('Creating project: $name');
  // ...
}
```

Now it works both ways:

```bash
# Interactive
$ dart run bin/scaffold.dart

# Non-interactive (for automation)
$ dart run bin/scaffold.dart --name my-app --description "Cool app" --git
```

## What's Next?

You can now build interactive CLI tools that feel like conversations! You've learned:

- Text input with validation
- Yes/no confirmations
- Single and multi-select lists
- Password input
- Building a complete project scaffolder
- Testing strategies

Next up: progress bars and spinners. Because nobody likes staring at a blank terminal while your tool does work.

Let's make waiting less painful!

---

**[← Previous: Pretty Colors](04-pretty-colors.md)** | **[Next: Progress and Spinners →](06-progress-and-spinners.md)**
