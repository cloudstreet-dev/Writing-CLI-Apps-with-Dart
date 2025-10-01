# Chapter 1: Why Dart for CLI?

**In which we justify our life choices**

"Wait, you're writing CLI apps in *Dart*? Isn't that the Flutter language?"

I've heard this question more times than I can count. And I get it — when most people think of Dart, they think of beautiful mobile apps with buttery-smooth animations. But here's the thing: Dart is secretly amazing for command-line tools, and most developers have no idea.

Let me convince you.

## The Preconceptions

Before we dive in, let's address the elephant in the room. When developers think about CLI languages, they usually reach for:

- **Python**: "It's easy to write and has libraries for everything!"
- **Go**: "Single binary, fast, perfect for CLI!"
- **Rust**: "Blazing fast and safe!"
- **Node.js**: "I already know JavaScript!"
- **Shell scripts**: "Why overcomplicate things?"

These are all valid choices! But Dart deserves a seat at this table, and by the end of this chapter, you'll understand why.

## The Case for Dart

### 1. Startup Time That Won't Make You Wait

Let's get the most important thing out of the way first: **startup performance**.

Nothing is more frustrating than a CLI tool that takes seconds to start. Looking at you, JVM-based tools that launch slower than my morning routine. And Python? Love it, but that import time adds up quickly.

Dart's AOT (Ahead-of-Time) compilation gives you near-instant startup:

```bash
# Compile your Dart CLI to a native executable
$ dart compile exe bin/mytool.dart -o mytool

# Now it starts faster than you can blink
$ time ./mytool --version
mytool 1.0.0

real    0m0.003s  # That's 3 milliseconds!
```

For comparison:
- **Python**: 20-100ms just to start the interpreter (more with imports)
- **Node.js**: 50-150ms
- **JVM**: *[laughs in 2-second warmup]*
- **Go/Rust**: 1-5ms (Dart's in the same ballpark!)

When your tool is invoked hundreds of times in a script or CI pipeline, those milliseconds matter.

### 2. Single Binary Distribution (No Runtime Required)

Here's a conversation you never want to have:

> **User**: "Your tool doesn't work!"
> **You**: "Do you have Python 3.9+ installed?"
> **User**: "I have Python 3.7..."
> **You**: "That won't work. Also, did you install the dependencies?"
> **User**: "What's pip?"
> **You**: *[internal screaming]*

With AOT compilation, Dart produces a **single, standalone executable**:

```bash
$ dart compile exe myapp.dart -o myapp
$ ls -lh myapp
-rwxr-xr-x  1 user  staff   6.2M Oct  1 12:00 myapp

# Ship this anywhere - no runtime needed!
$ scp myapp user@server:/usr/local/bin/
```

No interpreter. No virtual environments. No dependency hell. Just a binary that works.

### 3. Type Safety That Actually Helps

Let's be honest: CLI apps deal with a *lot* of string manipulation. Arguments, file paths, config files, output formatting — it's strings all the way down.

In dynamically typed languages, you write code like this:

```python
# Python example
def greet(name, greeting="Hello", excited=False):
    message = f"{greeting}, {name}"
    if excited:
        message += "!"
    return message

# What could go wrong?
greet("World", excited="yes")  # Runtime error!
greet(123)  # Also a runtime error!
```

In Dart, these errors are caught *before* your users see them:

```dart
String greet(String name, {String greeting = 'Hello', bool excited = false}) {
  var message = '$greeting, $name';
  if (excited) {
    message += '!';
  }
  return message;
}

// These won't compile:
greet('World', excited: 'yes');  // ❌ Error: String isn't a bool
greet(123);  // ❌ Error: int isn't a String
```

Type safety means fewer bugs, better autocomplete, and easier refactoring. Your future self will thank you.

### 4. Modern Language Features (That Don't Feel Bolted On)

Dart isn't trying to maintain backwards compatibility with decisions made in 1995. It's a modern language with modern features:

**Null Safety** (no more null pointer exceptions):
```dart
String? maybeGetConfig(String key) {
  // Returns null if key not found
}

var config = maybeGetConfig('database_url');
print(config.length);  // ❌ Won't compile - config might be null!

// You have to handle null explicitly:
if (config != null) {
  print(config.length);  // ✅ OK - Dart knows it's not null here
}
```

**Pattern Matching** (Dart 3.0+):
```dart
String describe(dynamic value) {
  return switch (value) {
    int n when n < 0 => 'negative number',
    int n => 'positive number',
    String s when s.isEmpty => 'empty string',
    String s => 'string: $s',
    List l => 'list with ${l.length} items',
    _ => 'something else',
  };
}
```

**Extension Methods** (add methods to existing types):
```dart
extension StringHelpers on String {
  String truncate(int maxLength) {
    if (length <= maxLength) return this;
    return '${substring(0, maxLength - 3)}...';
  }
}

print('This is a very long string'.truncate(10));  // "This is..."
```

These features make CLI code cleaner and more maintainable.

### 5. Excellent Tooling and Package Ecosystem

Dart comes with an incredibly polished toolchain:

**Package Management** (that just works):
```bash
$ dart pub add args  # Add dependencies
$ dart pub get       # Fetch dependencies
$ dart pub upgrade   # Update dependencies
```

**Code Formatting** (no more style arguments):
```bash
$ dart format .      # Formats all code, opinionated and consistent
```

**Analysis and Linting** (catch issues before they're problems):
```bash
$ dart analyze       # Static analysis with helpful suggestions
```

**Hot Reload for CLI** (yes, really!):
```bash
$ dart run --enable-vm-service bin/myapp.dart

# Edit your code, then in another terminal:
$ curl http://localhost:8181/
# Your app reloads with changes!
```

And the package ecosystem has everything you need for CLI development:
- `args` — Argument parsing
- `dart_console` — Full terminal control
- `mason` — Code generation and scaffolding
- `path` — Cross-platform path handling
- `yaml`, `json`, `toml` — Configuration file parsing
- `test` — Excellent testing framework

### 6. Cross-Platform (For Real)

```dart
import 'dart:io';

void main() {
  if (Platform.isWindows) {
    print('Hello from Windows!');
  } else if (Platform.isMacOS) {
    print('Hello from macOS!');
  } else if (Platform.isLinux) {
    print('Hello from Linux!');
  }

  // Or use path.join() that works everywhere:
  var configPath = path.join(
    Platform.environment['HOME'] ?? Platform.environment['USERPROFILE']!,
    '.myapp',
    'config.yaml'
  );
}
```

Compile once for each platform, and your tool works everywhere. No Python version mismatches, no "works on my machine" issues.

## When NOT to Use Dart

Let's be real — Dart isn't perfect for everything:

**❌ Don't use Dart when:**

1. **You need maximum raw performance for CPU-bound tasks**: Rust, C, or C++ will be faster. Dart is plenty fast for most CLI work, but if you're processing gigabytes of data and every microsecond counts, consider a systems language.

2. **The ecosystem doesn't have what you need**: If you're doing specialized scientific computing, Python's ecosystem is unbeatable. Machine learning? PyTorch and TensorFlow aren't coming to Dart anytime soon.

3. **Your team doesn't know Dart and doesn't want to learn**: Sometimes JavaScript/Python/Go is the right choice because your team already knows it. That's okay!

4. **You need to run on obscure platforms**: Dart supports Windows, macOS, Linux, and ARM. If you need to run on an obscure embedded system or BSD variant, shell scripts or Go might be safer bets.

5. **Quick-and-dirty shell scripts**: For a 10-line throwaway script, just use bash/zsh/PowerShell. Don't overthink it.

**✅ DO use Dart when:**

1. You're building a real tool that users will install and use
2. Startup performance matters (invoked frequently)
3. You want type safety and modern language features
4. Single binary distribution is important
5. Cross-platform support is needed
6. You're building a TUI (Terminal User Interface) application
7. You want excellent testing and tooling support

## The Secret Weapon

Here's the thing that really sold me on Dart for CLI: **the development experience is just pleasant**.

Coming from Python, I missed the REPL. Coming from Go, I missed generics (pre-Go 1.18). Coming from Rust, I missed not fighting the borrow checker for simple scripts. Coming from Node.js, I missed not dealing with callback hell and `node_modules` bloat.

Dart hits a sweet spot:
- Modern enough to be productive
- Performant enough to be fast
- Simple enough to not get in your way
- Powerful enough to build complex tools

## Real-World Examples

Don't just take my word for it. Dart is used for CLI tools in the wild:

- **`dart`** itself — The Dart SDK's CLI tool is written in Dart (naturally)
- **`pub`** — Dart's package manager
- **`very_good_cli`** — Project scaffolding tool by Very Good Ventures
- **`dart_frog`** — Backend framework CLI
- **`mason_cli`** — Template-based code generation
- **`melos`** — Monorepo management tool

These aren't toy projects — they're production tools used by thousands of developers daily.

## Show Me the Code!

Enough philosophy. Let's see what a Dart CLI tool actually looks like:

```dart
// bin/hello.dart
import 'dart:io';
import 'package:args/args.dart';

void main(List<String> arguments) {
  final parser = ArgParser()
    ..addOption('name', abbr: 'n', defaultsTo: 'World')
    ..addFlag('excited', abbr: 'e', help: 'Add excitement!');

  try {
    final results = parser.parse(arguments);
    final name = results['name'] as String;
    final excited = results['excited'] as bool;

    var greeting = 'Hello, $name';
    if (excited) greeting += '!';

    print(greeting);
  } on FormatException catch (e) {
    print(e.message);
    print(parser.usage);
    exit(1);
  }
}
```

Run it:
```bash
$ dart run bin/hello.dart
Hello, World

$ dart run bin/hello.dart --name Alice --excited
Hello, Alice!

$ dart run bin/hello.dart --invalid
Unknown option "invalid".
-n, --name        (defaults to "World")
-e, --excited     Add excitement!
```

Clean. Readable. Type-safe. And we're just getting started.

## What's Next?

Now that you're (hopefully) convinced that Dart is worth considering for CLI apps, let's actually build something.

In the next chapter, we'll dive deep into argument parsing. You'll learn how to handle flags, options, commands, subcommands, validation, and all the other fun stuff that makes a CLI tool feel professional.

But first, make sure you have Dart installed:

```bash
# Check if you have Dart
$ dart --version

# If not, install from https://dart.dev/get-dart
```

Ready? Let's build something awesome.

---

**[← Back to Table of Contents](00-table-of-contents.md)** | **[Next: Hello, Arguments! →](02-hello-arguments.md)**
