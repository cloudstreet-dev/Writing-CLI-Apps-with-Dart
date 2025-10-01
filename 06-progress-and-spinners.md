# Chapter 6: Progress and Spinners

**The art of "please wait..."**

Few things are more frustrating than running a command and staring at a blank screen, wondering: Is it working? Did it crash? Should I Ctrl+C and try again?

**Don't do this to your users.**

Long-running operations need visual feedback. This chapter is about giving users confidence that your tool is doing something, how much progress has been made, and approximately how long they'll be waiting.

Let's make waiting less painful.

## The Simple Approach: Print Messages

The most basic feedback is just printing what you're doing:

```dart
void main() async {
  print('Downloading file...');
  await downloadFile();

  print('Processing data...');
  await processData();

  print('Saving results...');
  await saveResults();

  print('Done!');
}
```

This works, but it's not very informative. How long will each step take? Are we stuck? Better than nothing, but we can do better.

## Spinners: For Indeterminate Operations

When you don't know how long something will take, use a **spinner**:

```dart
import 'package:cli_util/cli_logging.dart';

Future<void> main() async {
  final logger = Logger.standard();

  final progress = logger.progress('Connecting to server');
  await Future.delayed(Duration(seconds: 2));
  progress.finish(message: 'Connected!');

  final processing = logger.progress('Processing data');
  await Future.delayed(Duration(seconds: 3));
  processing.finish(message: 'Processing complete');
}
```

Output:
```
⠋ Connecting to server...
✓ Connected!
⠙ Processing data...
✓ Processing complete
```

The spinner animates while work happens, then shows a checkmark when done. Much better!

### Different Completion States

```dart
// Success
progress.finish(message: 'Success!');

// Show completion without explicit success
progress.finish(message: 'Done', showTiming: true);

// Just stop (no message)
progress.cancel();
```

### Nested Progress

```dart
Future<void> main() async {
  final logger = Logger.standard();

  final overall = logger.progress('Building project');

  stdout.writeln('  Installing dependencies...');
  await Future.delayed(Duration(seconds: 1));

  stdout.writeln('  Compiling code...');
  await Future.delayed(Duration(seconds: 2));

  stdout.writeln('  Running tests...');
  await Future.delayed(Duration(seconds: 1));

  overall.finish(message: 'Build complete!');
}
```

## Progress Bars: For Determinate Operations

When you know the total work (downloading a file, processing items), show a progress bar.

Unfortunately, Dart doesn't have a great built-in progress bar package, so let's build a simple one:

```dart
// lib/progress_bar.dart
import 'dart:io';

class ProgressBar {
  final int total;
  final int width;
  int _current = 0;

  ProgressBar({
    required this.total,
    this.width = 40,
  });

  void update(int current) {
    _current = current;
    _render();
  }

  void increment() {
    _current++;
    _render();
  }

  void _render() {
    final percentage = (_current / total * 100).toStringAsFixed(1);
    final filled = (_current / total * width).round();
    final empty = width - filled;

    final bar = '█' * filled + '░' * empty;

    // \r returns to start of line without newline
    stdout.write('\r[$bar] $percentage% ($_current/$total)');

    if (_current >= total) {
      stdout.writeln();  // New line when complete
    }
  }

  void finish() {
    update(total);
  }
}

// Usage
Future<void> main() async {
  final items = List.generate(100, (i) => i);
  final progress = ProgressBar(total: items.length);

  for (final item in items) {
    await Future.delayed(Duration(milliseconds: 50));
    progress.increment();
  }
}
```

Output:
```
[████████████████████░░░░░░░░░░░░░░░░░░░░] 50.0% (50/100)
```

The bar fills as work progresses. Satisfying!

### With Colors

Make it prettier:

```dart
import 'package:tint/tint.dart';

class ProgressBar {
  // ... previous code ...

  void _render() {
    final percentage = (_current / total * 100).toStringAsFixed(1);
    final filled = (_current / total * width).round();
    final empty = width - filled;

    final bar = ('█' * filled).green() + ('░' * empty).dim();
    final stats = '$percentage% ($_current/$total)';

    stdout.write('\r[$bar] $stats');

    if (_current >= total) {
      stdout.writeln(' ${'✓'.green()}');
    }
  }
}
```

### With Time Estimates

Calculate ETA based on current progress:

```dart
import 'dart:io';

class ProgressBar {
  final int total;
  final int width;
  final DateTime _startTime = DateTime.now();
  int _current = 0;

  ProgressBar({required this.total, this.width = 40});

  void update(int current) {
    _current = current;
    _render();
  }

  void increment() => update(_current + 1);

  void _render() {
    final percentage = (_current / total * 100).toStringAsFixed(1);
    final filled = (_current / total * width).round();
    final empty = width - filled;

    final bar = '█' * filled + '░' * empty;

    // Calculate ETA
    final elapsed = DateTime.now().difference(_startTime);
    final rate = _current / elapsed.inSeconds;
    final remaining = ((total - _current) / rate).round();
    final eta = remaining > 0 ? _formatDuration(Duration(seconds: remaining)) : '';

    stdout.write('\r[$bar] $percentage% ($_current/$total) $eta');

    if (_current >= total) {
      final totalTime = _formatDuration(elapsed);
      stdout.writeln(' Done in $totalTime');
    }
  }

  String _formatDuration(Duration d) {
    if (d.inHours > 0) {
      return '${d.inHours}h ${d.inMinutes.remainder(60)}m';
    } else if (d.inMinutes > 0) {
      return '${d.inMinutes}m ${d.inSeconds.remainder(60)}s';
    } else {
      return '${d.inSeconds}s';
    }
  }

  void finish() => update(total);
}
```

Output:
```
[████████████████████████████████████████] 100.0% (100/100) Done in 5s
```

## Multi-Step Progress

For complex operations with multiple steps:

```dart
class MultiStepProgress {
  final List<String> steps;
  int _currentStep = 0;

  MultiStepProgress(this.steps);

  void next(String? message) {
    if (_currentStep < steps.length) {
      stdout.writeln('✓ ${steps[_currentStep]}'.green());
    }
    _currentStep++;

    if (_currentStep < steps.length) {
      stdout.write('⠋ ${steps[_currentStep]}...'.cyan());
    }
  }

  void complete() {
    if (_currentStep < steps.length) {
      stdout.writeln('\r✓ ${steps[_currentStep]}'.green());
    }
    stdout.writeln('\nAll steps completed!'.green().bold());
  }

  void error(String message) {
    stdout.writeln('\r✗ ${steps[_currentStep]}'.red());
    stderr.writeln('Error: $message'.red());
  }
}

Future<void> main() async {
  final progress = MultiStepProgress([
    'Downloading dependencies',
    'Compiling code',
    'Running tests',
    'Building executable',
  ]);

  for (var i = 0; i < 4; i++) {
    progress.next(null);
    await Future.delayed(Duration(seconds: 1));
  }

  progress.complete();
}
```

Output:
```
✓ Downloading dependencies
✓ Compiling code
✓ Running tests
✓ Building executable

All steps completed!
```

## Real-World Example: File Downloader

Let's build a complete file downloader with progress:

```dart
// bin/download.dart
import 'dart:io';
import 'package:http/http.dart' as http;
import 'package:path/path.dart' as path;

Future<void> main(List<String> args) async {
  if (args.isEmpty) {
    stderr.writeln('Usage: download <url> [output-file]');
    exit(1);
  }

  final url = args[0];
  final filename = args.length > 1
      ? args[1]
      : path.basename(Uri.parse(url).path);

  await downloadFile(url, filename);
}

Future<void> downloadFile(String url, String filename) async {
  print('Downloading: $url');
  print('         to: $filename');
  print('');

  final request = await HttpClient().getUrl(Uri.parse(url));
  final response = await request.close();

  if (response.statusCode != 200) {
    stderr.writeln('Error: HTTP ${response.statusCode}');
    exit(1);
  }

  final contentLength = response.contentLength;
  final file = File(filename);
  final sink = file.openWrite();

  int downloaded = 0;
  final startTime = DateTime.now();

  // Progress bar
  final progressWidth = 40;

  await for (final chunk in response) {
    downloaded += chunk.length;
    sink.add(chunk);

    // Update progress
    final percentage = (downloaded / contentLength * 100).toStringAsFixed(1);
    final filled = (downloaded / contentLength * progressWidth).round();
    final empty = progressWidth - filled;
    final bar = '█' * filled + '░' * empty;

    // Calculate speed
    final elapsed = DateTime.now().difference(startTime).inSeconds;
    final speed = elapsed > 0 ? downloaded / elapsed : 0;
    final speedStr = _formatBytes(speed.round()) + '/s';

    // Calculate ETA
    final remaining = contentLength - downloaded;
    final eta = speed > 0 ? remaining / speed : 0;
    final etaStr = _formatDuration(Duration(seconds: eta.round()));

    final sizeStr = '${_formatBytes(downloaded)}/${_formatBytes(contentLength)}';

    stdout.write('\r[$bar] $percentage% $sizeStr $speedStr ETA: $etaStr');
  }

  await sink.close();

  final elapsed = DateTime.now().difference(startTime);
  stdout.writeln('\n\n✓ Downloaded in ${_formatDuration(elapsed)}');
}

String _formatBytes(int bytes) {
  if (bytes < 1024) return '${bytes}B';
  if (bytes < 1024 * 1024) return '${(bytes / 1024).toStringAsFixed(1)}KB';
  if (bytes < 1024 * 1024 * 1024) {
    return '${(bytes / (1024 * 1024)).toStringAsFixed(1)}MB';
  }
  return '${(bytes / (1024 * 1024 * 1024)).toStringAsFixed(1)}GB';
}

String _formatDuration(Duration d) {
  if (d.inHours > 0) return '${d.inHours}h ${d.inMinutes % 60}m';
  if (d.inMinutes > 0) return '${d.inMinutes}m ${d.inSeconds % 60}s';
  return '${d.inSeconds}s';
}
```

Add the http package:
```bash
$ dart pub add http
```

Usage:
```bash
$ dart run bin/download.dart https://example.com/large-file.zip
Downloading: https://example.com/large-file.zip
         to: large-file.zip

[████████████████████░░░░░░░░░░░░░░░░░░░░] 51.2% 512.3MB/1.0GB 2.5MB/s ETA: 3m 12s
```

Beautiful!

## Parallel Progress

Showing progress for multiple concurrent operations:

```dart
import 'dart:io';
import 'dart:async';

class ParallelProgress {
  final List<String> tasks;
  final Map<int, String> _status = {};
  final Map<int, double> _progress = {};

  ParallelProgress(this.tasks) {
    for (var i = 0; i < tasks.length; i++) {
      _progress[i] = 0.0;
      _status[i] = 'pending';
    }
  }

  void update(int taskIndex, double progress, [String status = 'running']) {
    _progress[taskIndex] = progress;
    _status[taskIndex] = status;
    _render();
  }

  void complete(int taskIndex) {
    _progress[taskIndex] = 1.0;
    _status[taskIndex] = 'done';
    _render();
  }

  void _render() {
    // Move cursor up to overwrite previous render
    if (_status.values.any((s) => s != 'pending')) {
      stdout.write('\x1b[${tasks.length}A');
    }

    for (var i = 0; i < tasks.length; i++) {
      final task = tasks[i];
      final progress = (_progress[i]! * 100).toStringAsFixed(0);
      final status = _status[i]!;

      final icon = switch (status) {
        'done' => '✓',
        'running' => '◐',
        'error' => '✗',
        _ => '○',
      };

      final bar = _makeBar(_progress[i]!, 20);
      stdout.write('\r$icon $task [$bar] $progress%');
      stdout.writeln(' ' * 20);  // Clear rest of line
    }
  }

  String _makeBar(double progress, int width) {
    final filled = (progress * width).round();
    return '█' * filled + '░' * (width - filled);
  }
}

Future<void> main() async {
  final progress = ParallelProgress([
    'Download file 1',
    'Download file 2',
    'Download file 3',
  ]);

  // Simulate parallel downloads
  final tasks = [
    _simulateTask(0, progress),
    _simulateTask(1, progress),
    _simulateTask(2, progress),
  ];

  await Future.wait(tasks);

  print('\nAll downloads complete!');
}

Future<void> _simulateTask(int index, ParallelProgress progress) async {
  for (var i = 0; i <= 100; i += 10) {
    await Future.delayed(Duration(milliseconds: 100 + index * 50));
    progress.update(index, i / 100);
  }
  progress.complete(index);
}
```

## Graceful Cancellation

Allow users to cancel long operations:

```dart
import 'dart:io';
import 'dart:async';

Future<void> main() async {
  print('Starting long operation... (Press Ctrl+C to cancel)');

  // Listen for SIGINT (Ctrl+C)
  late StreamSubscription subscription;
  subscription = ProcessSignal.sigint.watch().listen((_) {
    print('\n\nCancelling...');
    subscription.cancel();
    exit(130);  // Standard exit code for SIGINT
  });

  try {
    await longRunningOperation();
    print('Operation completed!');
  } finally {
    subscription.cancel();
  }
}

Future<void> longRunningOperation() async {
  for (var i = 1; i <= 100; i++) {
    await Future.delayed(Duration(milliseconds: 100));
    stdout.write('\rProgress: $i%');
  }
  stdout.writeln();
}
```

## Best Practices

1. **Show something immediately** — even if it's just "Starting..."
2. **Update frequently but not too frequently** — 10-30 times per second is good
3. **Always show completion** — success or failure
4. **Provide time estimates when possible** — ETA helps users plan
5. **Use appropriate indicators**:
   - Spinner for unknown duration
   - Progress bar for known duration
   - Percentage for clarity
6. **Handle terminal resizing** — not covered here, but important for robust tools
7. **Test on slow connections** — your users might not have gigabit fiber
8. **Allow cancellation** — respect Ctrl+C
9. **Show rates** (MB/s, items/s) — helps users understand speed
10. **Clear the line properly** — use `\r` and overwrite, not endless newlines

## Common Pitfalls

### Pitfall 1: Updating Too Frequently

```dart
// ❌ Bad: Updates every single byte (thousands per second)
await for (final byte in stream) {
  progress.update(downloaded++);  // Too much!
}

// ✅ Good: Update every 0.1 seconds or 1% progress
var lastUpdate = DateTime.now();
await for (final byte in stream) {
  downloaded++;
  final now = DateTime.now();
  if (now.difference(lastUpdate).inMilliseconds > 100) {
    progress.update(downloaded);
    lastUpdate = now;
  }
}
```

### Pitfall 2: Forgetting to Flush

```dart
// ❌ May not appear immediately due to buffering
stdout.write('\rProgress: $percent%');

// ✅ Force output immediately
stdout.write('\rProgress: $percent%');
await stdout.flush();
```

### Pitfall 3: Assuming Terminal Width

```dart
// ❌ Assumes 80-character terminal
final bar = '█' * 60;

// ✅ Detect terminal width
final terminalWidth = stdout.hasTerminal ? stdout.terminalColumns : 80;
final barWidth = (terminalWidth * 0.5).round();
```

## What's Next?

Your tools can now show beautiful progress indicators! You've learned:

- Spinners for indeterminate operations
- Progress bars with percentages and ETAs
- Multi-step progress tracking
- Parallel progress indicators
- Download progress with speed calculations
- Handling cancellation gracefully

In the next chapter, we'll tackle configuration files. Because hard-coding settings is so 2005.

Time to make your tools configurable!

---

**[← Previous: Interactive Prompts](05-interactive-prompts.md)** | **[Next: Configuration Files →](07-configuration-files.md)**
