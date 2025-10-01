# Chapter 9: Building a TUI

**Moving beyond line-based output**

Up until now, we've been writing tools that print lines and read input sequentially. That's perfectly fine for many tasks. But sometimes you want more:

- **Full-screen interfaces** like `htop`, `vim`, or `git add -i`
- **Real-time updates** without scrolling
- **Interactive navigation** with arrow keys
- **Multiple panes** showing different data
- **Visual feedback** that updates dynamically

This is the world of **Terminal User Interfaces (TUIs)**. They feel like GUI apps but run entirely in your terminal.

In this chapter, we'll learn the fundamentals: taking over the terminal, handling keyboard input, drawing to specific screen positions, and building our first interactive TUI application.

Welcome to the deep end. Let's dive in!

## What Makes a TUI Different?

Normal CLI apps:
```
$ mytool
Processing file 1...
Processing file 2...
Processing file 3...
Done!
```

TUI apps:
```
┌─ File Browser ─────────────────────────────┐
│ ▸ Documents/                               │
│ ▸ Downloads/                               │
│ ▾ Projects/                                │
│   ▸ dart-cli/                              │
│   ▸ website/                               │
│   • README.md                              │
│                                            │
│ [Enter] Open  [↑↓] Navigate  [q] Quit     │
└────────────────────────────────────────────┘
```

The TUI updates in place, responds to keystrokes immediately, and has a visual layout. It's a different paradigm.

## Terminal Control Basics

TUIs work by:

1. **Switching to raw mode** — disable line buffering and echo
2. **Using escape codes** — move cursor, clear screen, etc.
3. **Reading keyboard input** — one key at a time, including arrow keys
4. **Managing state** — tracking cursor position, selected items, etc.
5. **Rendering** — drawing the UI and updating it

### The `dart_console` Package

Rather than implementing all this ourselves, we'll use `dart_console`:

```bash
$ dart pub add dart_console
```

This package provides:
- Terminal control (cursor movement, clearing, colors)
- Keyboard input (raw mode, arrow keys, special keys)
- Screen management (size detection, buffering)

## Hello, TUI!

Let's start with the simplest possible TUI:

```dart
import 'dart:io';
import 'package:dart_console/dart_console.dart';

void main() {
  final console = Console();

  // Clear screen and move cursor to top-left
  console.clearScreen();
  console.cursorPosition = Coordinate(0, 0);

  // Write at specific position
  console.writeAt('Hello, TUI!', 5, 5);

  // Show cursor position
  console.writeAt('Press any key to exit', 0, console.windowHeight - 1);

  // Wait for key press
  console.readKey();

  // Clean up: clear screen and show cursor
  console.clearScreen();
  console.cursorPosition = Coordinate(0, 0);
  console.showCursor();
}
```

Run this and you'll see "Hello, TUI!" appear at row 5, column 5, and the prompt at the bottom.

### Understanding Coordinates

Terminal coordinates start at (0, 0) in the top-left:

```
(0,0)  (1,0)  (2,0) ...
(0,1)  (1,1)  (2,1) ...
(0,2)  (1,2)  (2,2) ...
...
```

`console.writeAt(text, x, y)` writes text at column x, row y.

## Raw Mode and Keyboard Input

Normal terminal mode waits for Enter before giving you input. Raw mode gives you keys immediately:

```dart
void main() {
  final console = Console();
  console.clearScreen();

  console.writeLine('Press keys (q to quit):');
  console.writeLine('');

  while (true) {
    final key = console.readKey();

    if (key.controlChar == ControlCharacter.ctrlC ||
        key.char == 'q') {
      break;
    }

    console.writeLine('You pressed: ${_describeKey(key)}');
  }

  console.clearScreen();
}

String _describeKey(Key key) {
  if (key.isControl) {
    return 'Control char: ${key.controlChar}';
  }
  return 'Char: ${key.char}';
}
```

This reads keys immediately and describes them. Try arrow keys, Enter, Escape, etc.

### Handling Special Keys

```dart
void handleKey(Key key) {
  if (key.isControl) {
    switch (key.controlChar) {
      case ControlCharacter.arrowUp:
        print('Up arrow');
        break;
      case ControlCharacter.arrowDown:
        print('Down arrow');
        break;
      case ControlCharacter.arrowLeft:
        print('Left arrow');
        break;
      case ControlCharacter.arrowRight:
        print('Right arrow');
        break;
      case ControlCharacter.enter:
        print('Enter');
        break;
      case ControlCharacter.escape:
        print('Escape');
        break;
      case ControlCharacter.ctrlC:
        print('Ctrl+C');
        break;
      default:
        print('Other control: ${key.controlChar}');
    }
  } else {
    print('Regular key: ${key.char}');
  }
}
```

## Drawing Boxes

Every good TUI has boxes. Let's draw one:

```dart
void drawBox(Console console, int x, int y, int width, int height, String title) {
  // Top border
  console.writeAt('┌', x, y);
  console.writeAt('─' * (width - 2), x + 1, y);
  console.writeAt('┐', x + width - 1, y);

  // Title
  if (title.isNotEmpty) {
    console.writeAt(' $title ', x + 2, y);
  }

  // Sides
  for (var i = 1; i < height - 1; i++) {
    console.writeAt('│', x, y + i);
    console.writeAt('│', x + width - 1, y + i);
  }

  // Bottom border
  console.writeAt('└', x, y + height - 1);
  console.writeAt('─' * (width - 2), x + 1, y + height - 1);
  console.writeAt('┘', x + width - 1, y + height - 1);
}

void main() {
  final console = Console();
  console.clearScreen();

  drawBox(console, 5, 2, 40, 10, 'My Box');

  console.cursorPosition = Coordinate(0, console.windowHeight - 1);
  console.write('Press any key to exit');
  console.readKey();

  console.clearScreen();
}
```

Box drawing characters:
- `┌ ┐ └ ┘` — corners
- `─ │` — horizontal and vertical lines
- `├ ┤ ┬ ┴ ┼` — T-junctions and crosses

## The Event Loop

TUIs need an event loop: render, read input, update state, repeat.

```dart
void main() {
  final console = Console();
  var running = true;
  var counter = 0;

  while (running) {
    // Render
    console.clearScreen();
    console.writeAt('Counter: $counter', 5, 5);
    console.writeAt('Press [↑] to increment, [↓] to decrement, [q] to quit', 0, console.windowHeight - 1);

    // Handle input
    final key = console.readKey();

    if (key.isControl) {
      switch (key.controlChar) {
        case ControlCharacter.arrowUp:
          counter++;
          break;
        case ControlCharacter.arrowDown:
          counter--;
          break;
        case ControlCharacter.ctrlC:
          running = false;
          break;
        default:
          break;
      }
    } else if (key.char == 'q') {
      running = false;
    }
  }

  console.clearScreen();
}
```

This is the basic pattern:
1. Clear screen (or update specific parts)
2. Render current state
3. Read input
4. Update state based on input
5. Repeat

## Example: Interactive Menu

Let's build a menu you can navigate with arrow keys:

```dart
import 'package:dart_console/dart_console.dart';

class Menu {
  final Console console;
  final List<String> items;
  int selectedIndex = 0;

  Menu({
    required this.console,
    required this.items,
  });

  String run() {
    var running = true;

    while (running) {
      render();

      final key = console.readKey();

      if (key.isControl) {
        switch (key.controlChar) {
          case ControlCharacter.arrowUp:
            selectedIndex = (selectedIndex - 1) % items.length;
            if (selectedIndex < 0) selectedIndex = items.length - 1;
            break;

          case ControlCharacter.arrowDown:
            selectedIndex = (selectedIndex + 1) % items.length;
            break;

          case ControlCharacter.enter:
            running = false;
            break;

          case ControlCharacter.ctrlC:
            console.clearScreen();
            exit(0);

          default:
            break;
        }
      } else if (key.char == 'q') {
        console.clearScreen();
        exit(0);
      }
    }

    console.clearScreen();
    return items[selectedIndex];
  }

  void render() {
    console.clearScreen();

    console.writeAt('Select an option:', 2, 1);

    for (var i = 0; i < items.length; i++) {
      final prefix = i == selectedIndex ? '▶ ' : '  ';
      final item = items[i];

      if (i == selectedIndex) {
        console.writeAt(prefix + item, 2, 3 + i, TextStyle.inverse);
      } else {
        console.writeAt(prefix + item, 2, 3 + i);
      }
    }

    console.writeAt('[↑↓] Navigate  [Enter] Select  [q] Quit', 0, console.windowHeight - 1);
  }
}

void main() {
  final console = Console();

  final menu = Menu(
    console: console,
    items: [
      'Start new game',
      'Load saved game',
      'Settings',
      'Quit',
    ],
  );

  final choice = menu.run();

  print('You selected: $choice');
}
```

This menu:
- Shows items with the selected one highlighted
- Responds to arrow keys instantly
- Wraps around (down from the last item goes to first)
- Returns the selected value

## Double Buffering (Preventing Flicker)

Clearing the screen every frame causes flicker. Better approach: only update what changed.

```dart
class Screen {
  final Console console;
  late List<List<String>> buffer;
  late List<List<String>> previousBuffer;

  Screen(this.console) {
    final width = console.windowWidth;
    final height = console.windowHeight;

    buffer = List.generate(height, (_) => List.filled(width, ' '));
    previousBuffer = List.generate(height, (_) => List.filled(width, ' '));
  }

  void writeAt(String text, int x, int y) {
    if (y < 0 || y >= buffer.length) return;

    for (var i = 0; i < text.length; i++) {
      final col = x + i;
      if (col >= 0 && col < buffer[y].length) {
        buffer[y][col] = text[i];
      }
    }
  }

  void render() {
    for (var y = 0; y < buffer.length; y++) {
      for (var x = 0; x < buffer[y].length; x++) {
        // Only update if changed
        if (buffer[y][x] != previousBuffer[y][x]) {
          console.cursorPosition = Coordinate(x, y);
          console.write(buffer[y][x]);
          previousBuffer[y][x] = buffer[y][x];
        }
      }
    }
  }

  void clear() {
    for (var y = 0; y < buffer.length; y++) {
      for (var x = 0; x < buffer[y].length; x++) {
        buffer[y][x] = ' ';
      }
    }
  }
}
```

Now you can update the buffer and only changed cells are redrawn.

## Example: File Browser

Let's build a simple file browser:

```dart
import 'dart:io';
import 'package:dart_console/dart_console.dart';
import 'package:path/path.dart' as path;

class FileBrowser {
  final Console console;
  late List<FileSystemEntity> entries;
  late Directory currentDir;
  int selectedIndex = 0;
  int scrollOffset = 0;

  FileBrowser(this.console, String startPath) {
    currentDir = Directory(startPath);
    _loadEntries();
  }

  void _loadEntries() {
    entries = currentDir.listSync()..sort((a, b) {
      // Directories first, then by name
      final aIsDir = a is Directory;
      final bIsDir = b is Directory;

      if (aIsDir && !bIsDir) return -1;
      if (!aIsDir && bIsDir) return 1;

      return path.basename(a.path).compareTo(path.basename(b.path));
    });

    selectedIndex = 0;
    scrollOffset = 0;
  }

  void run() {
    console.hideCursor();
    var running = true;

    try {
      while (running) {
        render();

        final key = console.readKey();

        if (key.isControl) {
          switch (key.controlChar) {
            case ControlCharacter.arrowUp:
              if (selectedIndex > 0) {
                selectedIndex--;
                _adjustScroll();
              }
              break;

            case ControlCharacter.arrowDown:
              if (selectedIndex < entries.length - 1) {
                selectedIndex++;
                _adjustScroll();
              }
              break;

            case ControlCharacter.enter:
              _openSelected();
              break;

            case ControlCharacter.ctrlC:
              running = false;
              break;

            default:
              break;
          }
        } else if (key.char == 'q') {
          running = false;
        } else if (key.char == 'h' || key.char == 'b') {
          // Go to parent directory
          _goToParent();
        }
      }
    } finally {
      console.clearScreen();
      console.showCursor();
    }
  }

  void _adjustScroll() {
    final visibleLines = console.windowHeight - 4;  // Leave room for header/footer

    if (selectedIndex < scrollOffset) {
      scrollOffset = selectedIndex;
    } else if (selectedIndex >= scrollOffset + visibleLines) {
      scrollOffset = selectedIndex - visibleLines + 1;
    }
  }

  void _openSelected() {
    if (entries.isEmpty) return;

    final selected = entries[selectedIndex];

    if (selected is Directory) {
      currentDir = selected;
      _loadEntries();
    }
  }

  void _goToParent() {
    final parent = currentDir.parent;
    if (parent.path != currentDir.path) {
      currentDir = parent;
      _loadEntries();
    }
  }

  void render() {
    console.clearScreen();

    // Header
    console.writeAt('─' * console.windowWidth, 0, 0);
    console.writeAt(' File Browser: ${currentDir.path}', 1, 0);

    // File list
    final visibleLines = console.windowHeight - 4;
    final endIndex = (scrollOffset + visibleLines).clamp(0, entries.length);

    for (var i = scrollOffset; i < endIndex; i++) {
      final y = 2 + (i - scrollOffset);
      final entry = entries[i];
      final isSelected = i == selectedIndex;

      final icon = entry is Directory ? '📁' : '📄';
      final name = path.basename(entry.path);
      final text = '$icon $name';

      if (isSelected) {
        console.writeAt('▶ $text', 1, y, TextStyle.inverse);
      } else {
        console.writeAt('  $text', 1, y);
      }
    }

    // Footer
    final footerY = console.windowHeight - 1;
    console.writeAt('─' * console.windowWidth, 0, footerY - 1);
    console.writeAt('[↑↓] Navigate  [Enter] Open  [h] Parent  [q] Quit', 1, footerY);

    // Scroll indicator
    if (entries.length > visibleLines) {
      final scrollPercent = (scrollOffset / (entries.length - visibleLines) * 100).round();
      console.writeAt('${selectedIndex + 1}/${entries.length} ($scrollPercent%)',
          console.windowWidth - 20, footerY);
    } else {
      console.writeAt('${selectedIndex + 1}/${entries.length}',
          console.windowWidth - 20, footerY);
    }
  }
}

void main() {
  final console = Console();
  final browser = FileBrowser(console, Directory.current.path);
  browser.run();
}
```

This browser:
- Lists files and directories
- Lets you navigate with arrow keys
- Opens directories with Enter
- Goes to parent directory with 'h'
- Handles scrolling for long lists
- Shows your current location

## Handling Terminal Resize

Users can resize their terminal. Handle it:

```dart
void main() {
  final console = Console();
  var lastWidth = console.windowWidth;
  var lastHeight = console.windowHeight;

  while (true) {
    // Check for resize
    final currentWidth = console.windowWidth;
    final currentHeight = console.windowHeight;

    if (currentWidth != lastWidth || currentHeight != lastHeight) {
      // Terminal was resized!
      lastWidth = currentWidth;
      lastHeight = currentHeight;

      console.clearScreen();
      console.writeAt('Terminal resized to ${currentWidth}x$currentHeight', 1, 1);
    }

    // ... rest of event loop
  }
}
```

For production TUIs, you'd rebuild your entire layout when this happens.

## Best Practices for TUIs

1. **Always clean up** — clear screen, show cursor, restore terminal mode
2. **Handle Ctrl+C gracefully** — users expect it to work
3. **Provide visual feedback** — highlight selections, show what's focused
4. **Support keyboard shortcuts** — hjkl for vim users, arrows for everyone
5. **Show help** — status bar with available keys
6. **Handle small terminals** — don't assume 80x24
7. **Use double buffering** — prevent flicker
8. **Test in different terminals** — behavior varies slightly

## Cleanup Pattern

Always restore terminal state:

```dart
void main() {
  final console = Console();

  try {
    console.hideCursor();
    console.clearScreen();

    // Your TUI logic here
    runTUI(console);
  } finally {
    // Always cleanup, even on errors
    console.showCursor();
    console.clearScreen();
    console.resetColorAttributes();
  }
}
```

Or use a wrapper:

```dart
T withTUI<T>(Console console, T Function() action) {
  console.hideCursor();
  console.clearScreen();

  try {
    return action();
  } finally {
    console.showCursor();
    console.clearScreen();
    console.resetColorAttributes();
  }
}

void main() {
  final console = Console();

  withTUI(console, () {
    // Your TUI code here
    runApp(console);
  });
}
```

## What's Next?

You now know the fundamentals of TUI programming! You've learned:

- Terminal control and raw mode
- Keyboard input handling
- Cursor positioning and screen updates
- Drawing boxes and UI elements
- The event loop pattern
- Building interactive menus and file browsers

In the next chapter, we'll take it further: **Advanced TUI patterns**. We'll build complex interfaces with tables, tree views, split panes, modal dialogs, and sophisticated keyboard handling.

The fun is just beginning!

---

**[← Previous: Error Handling](08-error-handling.md)** | **[Next: Advanced TUI Patterns →](10-advanced-tui.md)**
