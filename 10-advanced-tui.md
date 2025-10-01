# Chapter 10: Advanced TUI Patterns

**Complex interfaces and sophisticated interactions**

In the last chapter, we learned TUI basics: drawing, input handling, and simple interfaces. Now it's time to level up.

This chapter covers advanced patterns you'll find in production TUIs:

- **Tables** with sorting and filtering
- **Tree views** for hierarchical data
- **Split panes** for multiple views
- **Modal dialogs** for confirmations and input
- **Keyboard binding systems** for complex shortcuts
- **Themes and styling** for consistent UIs
- **Component architecture** for maintainable code

Let's build something impressive.

## Component Architecture

First, let's establish a pattern for building reusable UI components:

```dart
abstract class Component {
  int x;
  int y;
  int width;
  int height;

  Component({
    required this.x,
    required this.y,
    required this.width,
    required this.height,
  });

  /// Render this component to the console
  void render(Console console);

  /// Handle a key press. Returns true if handled.
  bool handleKey(Key key) => false;

  /// Check if a coordinate is within this component
  bool contains(int px, int py) {
    return px >= x && px < x + width && py >= y && py < y + height;
  }
}
```

Now we can build components that know how to render themselves and handle input.

## Tables

Tables are everywhere in TUIs. Let's build a good one:

```dart
class TableColumn {
  final String header;
  final int width;
  final String Function(Map<String, dynamic>) accessor;
  final String alignment;  // 'left', 'right', 'center'

  TableColumn({
    required this.header,
    required this.width,
    required this.accessor,
    this.alignment = 'left',
  });
}

class Table extends Component {
  final List<TableColumn> columns;
  final List<Map<String, dynamic>> data;
  int selectedRow = 0;
  int scrollOffset = 0;
  String? sortColumn;
  bool sortAscending = true;

  Table({
    required this.columns,
    required this.data,
    required super.x,
    required super.y,
    required super.width,
    required super.height,
  });

  @override
  void render(Console console) {
    final visibleRows = height - 2;  // Header + border

    // Draw header
    console.writeAt('┌', x, y);
    var currentX = x + 1;

    for (var i = 0; i < columns.length; i++) {
      final col = columns[i];
      final header = _padString(col.header, col.width, 'center');
      console.writeAt(header, currentX, y);
      currentX += col.width;

      if (i < columns.length - 1) {
        console.writeAt('┬', currentX, y);
        currentX++;
      }
    }

    console.writeAt('┐', currentX, y);

    // Draw separator
    console.writeAt('├', x, y + 1);
    currentX = x + 1;

    for (var i = 0; i < columns.length; i++) {
      final col = columns[i];
      console.writeAt('─' * col.width, currentX, y + 1);
      currentX += col.width;

      if (i < columns.length - 1) {
        console.writeAt('┼', currentX, y + 1);
        currentX++;
      }
    }

    console.writeAt('┤', currentX, y + 1);

    // Draw rows
    final endIndex = (scrollOffset + visibleRows).clamp(0, data.length);

    for (var i = scrollOffset; i < endIndex; i++) {
      final row = data[i];
      final rowY = y + 2 + (i - scrollOffset);
      final isSelected = i == selectedRow;

      console.writeAt('│', x, rowY);
      currentX = x + 1;

      for (var j = 0; j < columns.length; j++) {
        final col = columns[j];
        final value = col.accessor(row);
        final formatted = _padString(value, col.width, col.alignment);

        if (isSelected) {
          console.writeAt(formatted, currentX, rowY, TextStyle.inverse);
        } else {
          console.writeAt(formatted, currentX, rowY);
        }

        currentX += col.width;

        if (j < columns.length - 1) {
          console.writeAt('│', currentX, rowY);
          currentX++;
        }
      }

      console.writeAt('│', currentX, rowY);
    }

    // Draw bottom border
    final bottomY = y + height - 1;
    console.writeAt('└', x, bottomY);
    currentX = x + 1;

    for (var i = 0; i < columns.length; i++) {
      final col = columns[i];
      console.writeAt('─' * col.width, currentX, bottomY);
      currentX += col.width;

      if (i < columns.length - 1) {
        console.writeAt('┴', currentX, bottomY);
        currentX++;
      }
    }

    console.writeAt('┘', currentX, bottomY);
  }

  @override
  bool handleKey(Key key) {
    if (key.isControl) {
      switch (key.controlChar) {
        case ControlCharacter.arrowUp:
          if (selectedRow > 0) {
            selectedRow--;
            _adjustScroll();
          }
          return true;

        case ControlCharacter.arrowDown:
          if (selectedRow < data.length - 1) {
            selectedRow++;
            _adjustScroll();
          }
          return true;

        default:
          return false;
      }
    } else if (key.char == 's') {
      // Toggle sort
      sortAscending = !sortAscending;
      _sortData();
      return true;
    }

    return false;
  }

  void _adjustScroll() {
    final visibleRows = height - 2;

    if (selectedRow < scrollOffset) {
      scrollOffset = selectedRow;
    } else if (selectedRow >= scrollOffset + visibleRows) {
      scrollOffset = selectedRow - visibleRows + 1;
    }
  }

  void _sortData() {
    if (sortColumn != null) {
      data.sort((a, b) {
        final col = columns.firstWhere((c) => c.header == sortColumn);
        final aVal = col.accessor(a);
        final bVal = col.accessor(b);

        final result = aVal.compareTo(bVal);
        return sortAscending ? result : -result;
      });
    }
  }

  String _padString(String text, int width, String alignment) {
    if (text.length > width) {
      return text.substring(0, width - 3) + '...';
    }

    final padding = width - text.length;

    return switch (alignment) {
      'right' => '${' ' * padding}$text',
      'center' => '${' ' * (padding ~/ 2)}$text${' ' * (padding - padding ~/ 2)}',
      _ => '$text${' ' * padding}',  // left (default)
    };
  }

  Map<String, dynamic>? getSelectedRow() {
    if (data.isEmpty) return null;
    return data[selectedRow];
  }
}

// Usage
void main() {
  final console = Console();

  final table = Table(
    columns: [
      TableColumn(
        header: 'Name',
        width: 20,
        accessor: (row) => row['name'] as String,
      ),
      TableColumn(
        header: 'Age',
        width: 5,
        accessor: (row) => row['age'].toString(),
        alignment: 'right',
      ),
      TableColumn(
        header: 'Email',
        width: 30,
        accessor: (row) => row['email'] as String,
      ),
    ],
    data: [
      {'name': 'Alice', 'age': 30, 'email': 'alice@example.com'},
      {'name': 'Bob', 'age': 25, 'email': 'bob@example.com'},
      {'name': 'Charlie', 'age': 35, 'email': 'charlie@example.com'},
    ],
    x: 2,
    y: 2,
    width: 60,
    height: 10,
  );

  try {
    console.hideCursor();
    var running = true;

    while (running) {
      console.clearScreen();
      table.render(console);

      console.writeAt('[↑↓] Navigate  [s] Sort  [q] Quit', 0, console.windowHeight - 1);

      final key = console.readKey();

      if (key.char == 'q') {
        running = false;
      } else {
        table.handleKey(key);
      }
    }
  } finally {
    console.clearScreen();
    console.showCursor();
  }
}
```

This table:
- Renders with proper borders
- Handles navigation with arrow keys
- Supports column alignment
- Scrolls for long data
- Highlights selected row
- Truncates overflow with "..."

## Tree Views

For hierarchical data like file systems or JSON:

```dart
class TreeNode {
  final String label;
  final List<TreeNode> children;
  bool expanded;
  final dynamic data;

  TreeNode({
    required this.label,
    this.children = const [],
    this.expanded = false,
    this.data,
  });
}

class TreeView extends Component {
  final TreeNode root;
  int selectedIndex = 0;
  late List<_FlatNode> _flatTree;

  TreeView({
    required this.root,
    required super.x,
    required super.y,
    required super.width,
    required super.height,
  }) {
    _rebuildFlatTree();
  }

  void _rebuildFlatTree() {
    _flatTree = [];
    _flattenTree(root, 0);
  }

  void _flattenTree(TreeNode node, int depth) {
    _flatTree.add(_FlatNode(node: node, depth: depth));

    if (node.expanded) {
      for (final child in node.children) {
        _flattenTree(child, depth + 1);
      }
    }
  }

  @override
  void render(Console console) {
    final visibleLines = height - 2;

    // Header
    console.writeAt('┌─ Tree View ', x, y);
    console.writeAt('─' * (width - 14), x + 13, y);
    console.writeAt('┐', x + width - 1, y);

    // Render visible nodes
    for (var i = 0; i < visibleLines && i < _flatTree.length; i++) {
      final flatNode = _flatTree[i];
      final node = flatNode.node;
      final depth = flatNode.depth;
      final isSelected = i == selectedIndex;

      final indent = '  ' * depth;
      final icon = node.children.isEmpty
          ? '• '
          : (node.expanded ? '▼ ' : '▶ ');

      final text = '$indent$icon${node.label}';
      final displayText = text.length > width - 4
          ? text.substring(0, width - 7) + '...'
          : text.padRight(width - 4);

      if (isSelected) {
        console.writeAt('│ $displayText │', x, y + 1 + i, TextStyle.inverse);
      } else {
        console.writeAt('│ $displayText │', x, y + 1 + i);
      }
    }

    // Fill empty lines
    for (var i = _flatTree.length; i < visibleLines; i++) {
      console.writeAt('│${' ' * (width - 2)}│', x, y + 1 + i);
    }

    // Footer
    console.writeAt('└', x, y + height - 1);
    console.writeAt('─' * (width - 2), x + 1, y + height - 1);
    console.writeAt('┘', x + width - 1, y + height - 1);
  }

  @override
  bool handleKey(Key key) {
    if (key.isControl) {
      switch (key.controlChar) {
        case ControlCharacter.arrowUp:
          if (selectedIndex > 0) selectedIndex--;
          return true;

        case ControlCharacter.arrowDown:
          if (selectedIndex < _flatTree.length - 1) selectedIndex++;
          return true;

        case ControlCharacter.arrowRight:
        case ControlCharacter.enter:
          final node = _flatTree[selectedIndex].node;
          if (node.children.isNotEmpty) {
            node.expanded = true;
            _rebuildFlatTree();
          }
          return true;

        case ControlCharacter.arrowLeft:
          final node = _flatTree[selectedIndex].node;
          if (node.expanded) {
            node.expanded = false;
            _rebuildFlatTree();
          }
          return true;

        default:
          return false;
      }
    }

    return false;
  }

  TreeNode? getSelected() {
    if (_flatTree.isEmpty) return null;
    return _flatTree[selectedIndex].node;
  }
}

class _FlatNode {
  final TreeNode node;
  final int depth;

  _FlatNode({required this.node, required this.depth});
}

// Usage
void main() {
  final tree = TreeNode(
    label: 'Root',
    children: [
      TreeNode(
        label: 'Folder 1',
        children: [
          TreeNode(label: 'File 1.1'),
          TreeNode(label: 'File 1.2'),
        ],
      ),
      TreeNode(
        label: 'Folder 2',
        children: [
          TreeNode(label: 'File 2.1'),
          TreeNode(
            label: 'Subfolder',
            children: [
              TreeNode(label: 'File 2.2.1'),
            ],
          ),
        ],
      ),
      TreeNode(label: 'File 3'),
    ],
  );

  final console = Console();
  final treeView = TreeView(
    root: tree,
    x: 5,
    y: 2,
    width: 40,
    height: 15,
  );

  try {
    console.hideCursor();
    var running = true;

    while (running) {
      console.clearScreen();
      treeView.render(console);

      console.writeAt('[↑↓] Navigate  [→] Expand  [←] Collapse  [q] Quit',
          0, console.windowHeight - 1);

      final key = console.readKey();

      if (key.char == 'q') {
        running = false;
      } else {
        treeView.handleKey(key);
      }
    }
  } finally {
    console.clearScreen();
    console.showCursor();
  }
}
```

## Split Panes

Display multiple views side by side:

```dart
class SplitPane extends Component {
  final Component left;
  final Component right;
  final double splitRatio;  // 0.0 to 1.0
  Component? focused;

  SplitPane({
    required this.left,
    required this.right,
    this.splitRatio = 0.5,
    required super.x,
    required super.y,
    required super.width,
    required super.height,
  }) {
    focused = left;
    _layoutChildren();
  }

  void _layoutChildren() {
    final leftWidth = (width * splitRatio).floor();
    final rightWidth = width - leftWidth - 1;  // -1 for divider

    left.x = x;
    left.y = y;
    left.width = leftWidth;
    left.height = height;

    right.x = x + leftWidth + 1;
    right.y = y;
    right.width = rightWidth;
    right.height = height;
  }

  @override
  void render(Console console) {
    left.render(console);
    right.render(console);

    // Draw vertical divider
    final dividerX = x + left.width;
    for (var i = 0; i < height; i++) {
      console.writeAt('│', dividerX, y + i);
    }

    // Highlight focused pane
    if (focused == left) {
      console.writeAt('◀', dividerX - 1, y + height ~/ 2);
    } else {
      console.writeAt('▶', dividerX + 1, y + height ~/ 2);
    }
  }

  @override
  bool handleKey(Key key) {
    // Tab switches focus
    if (key.controlChar == ControlCharacter.tab) {
      focused = (focused == left) ? right : left;
      return true;
    }

    // Pass to focused component
    return focused?.handleKey(key) ?? false;
  }
}

// Usage: Show file browser on left, file preview on right
void main() {
  final console = Console();

  final leftPane = FileList(/* ... */);
  final rightPane = FilePreview(/* ... */);

  final splitPane = SplitPane(
    left: leftPane,
    right: rightPane,
    splitRatio: 0.4,
    x: 0,
    y: 0,
    width: console.windowWidth,
    height: console.windowHeight - 1,
  );

  // ... event loop
}
```

## Modal Dialogs

Overlay dialogs on top of the main UI:

```dart
class Dialog {
  final String title;
  final String message;
  final List<String> buttons;
  int selectedButton = 0;

  Dialog({
    required this.title,
    required this.message,
    this.buttons = const ['OK'],
  });

  /// Returns the index of the selected button
  int show(Console console) {
    final width = 50;
    final height = 10;
    final x = (console.windowWidth - width) ~/ 2;
    final y = (console.windowHeight - height) ~/ 2;

    var running = true;

    while (running) {
      _render(console, x, y, width, height);

      final key = console.readKey();

      if (key.isControl) {
        switch (key.controlChar) {
          case ControlCharacter.arrowLeft:
            if (selectedButton > 0) selectedButton--;
            break;

          case ControlCharacter.arrowRight:
            if (selectedButton < buttons.length - 1) selectedButton++;
            break;

          case ControlCharacter.enter:
            running = false;
            break;

          default:
            break;
        }
      } else if (key.char == 'q' || key.controlChar == ControlCharacter.escape) {
        selectedButton = -1;
        running = false;
      }
    }

    return selectedButton;
  }

  void _render(Console console, int x, int y, int width, int height) {
    // Draw shadow
    for (var i = 0; i < height; i++) {
      console.writeAt('  ', x + width, y + i + 1, TextStyle.dim);
    }
    console.writeAt(' ' * width, x + 2, y + height, TextStyle.dim);

    // Draw box
    console.writeAt('┌', x, y);
    console.writeAt('─' * (width - 2), x + 1, y);
    console.writeAt('┐', x + width - 1, y);

    // Title
    console.writeAt(' $title ', x + 2, y, TextStyle.bold);

    // Sides
    for (var i = 1; i < height - 1; i++) {
      console.writeAt('│', x, y + i);
      console.writeAt(' ' * (width - 2), x + 1, y + i);
      console.writeAt('│', x + width - 1, y + i);
    }

    // Message (word-wrapped)
    final lines = _wordWrap(message, width - 4);
    for (var i = 0; i < lines.length && i < height - 5; i++) {
      console.writeAt(lines[i], x + 2, y + 2 + i);
    }

    // Buttons
    final buttonY = y + height - 3;
    final totalButtonWidth = buttons.map((b) => b.length + 4).reduce((a, b) => a + b);
    var buttonX = x + (width - totalButtonWidth) ~/ 2;

    for (var i = 0; i < buttons.length; i++) {
      final button = ' ${buttons[i]} ';

      if (i == selectedButton) {
        console.writeAt('[ $button ]', buttonX, buttonY, TextStyle.inverse);
      } else {
        console.writeAt('[ $button ]', buttonX, buttonY);
      }

      buttonX += button.length + 6;
    }

    // Bottom border
    console.writeAt('└', x, y + height - 1);
    console.writeAt('─' * (width - 2), x + 1, y + height - 1);
    console.writeAt('┘', x + width - 1, y + height - 1);
  }

  List<String> _wordWrap(String text, int width) {
    final words = text.split(' ');
    final lines = <String>[];
    var currentLine = '';

    for (final word in words) {
      if (currentLine.length + word.length + 1 <= width) {
        currentLine += (currentLine.isEmpty ? '' : ' ') + word;
      } else {
        if (currentLine.isNotEmpty) lines.add(currentLine);
        currentLine = word;
      }
    }

    if (currentLine.isNotEmpty) lines.add(currentLine);
    return lines;
  }
}

// Usage
void main() {
  final console = Console();

  final dialog = Dialog(
    title: 'Confirm Delete',
    message: 'Are you sure you want to delete this file? This action cannot be undone.',
    buttons: ['Cancel', 'Delete'],
  );

  final choice = dialog.show(console);

  if (choice == 1) {
    print('File deleted!');
  } else {
    print('Cancelled');
  }
}
```

## Keyboard Binding System

Map keys to actions flexibly:

```dart
class KeyBinding {
  final String key;
  final String description;
  final void Function() action;

  KeyBinding({
    required this.key,
    required this.description,
    required this.action,
  });
}

class KeyboardHandler {
  final Map<String, KeyBinding> bindings = {};

  void bind(String key, String description, void Function() action) {
    bindings[key] = KeyBinding(
      key: key,
      description: description,
      action: action,
    );
  }

  bool handle(Key key) {
    String? keyString;

    if (key.isControl) {
      keyString = _controlToString(key.controlChar);
    } else {
      keyString = key.char;
    }

    if (keyString != null && bindings.containsKey(keyString)) {
      bindings[keyString]!.action();
      return true;
    }

    return false;
  }

  String? _controlToString(ControlCharacter ctrl) {
    return switch (ctrl) {
      ControlCharacter.arrowUp => 'UP',
      ControlCharacter.arrowDown => 'DOWN',
      ControlCharacter.arrowLeft => 'LEFT',
      ControlCharacter.arrowRight => 'RIGHT',
      ControlCharacter.enter => 'ENTER',
      ControlCharacter.escape => 'ESC',
      ControlCharacter.tab => 'TAB',
      ControlCharacter.ctrlC => 'CTRL+C',
      _ => null,
    };
  }

  List<String> getHelpText() {
    return bindings.values
        .map((b) => '${b.key.padRight(10)} ${b.description}')
        .toList();
  }
}

// Usage
void main() {
  final kbd = KeyboardHandler();

  kbd.bind('j', 'Move down', () => print('Down!'));
  kbd.bind('k', 'Move up', () => print('Up!'));
  kbd.bind('h', 'Move left', () => print('Left!'));
  kbd.bind('l', 'Move right', () => print('Right!'));
  kbd.bind('DOWN', 'Move down', () => print('Down!'));
  kbd.bind('UP', 'Move up', () => print('Up!'));
  kbd.bind('q', 'Quit', () => exit(0));

  // In event loop:
  final key = console.readKey();
  kbd.handle(key);

  // Show help:
  print('Available keys:');
  for (final line in kbd.getHelpText()) {
    print(line);
  }
}
```

## Themes and Styling

Consistent colors and styles:

```dart
class Theme {
  final String primaryFg;
  final String primaryBg;
  final String secondaryFg;
  final String secondaryBg;
  final String accentFg;
  final String errorFg;
  final String successFg;

  const Theme({
    required this.primaryFg,
    required this.primaryBg,
    required this.secondaryFg,
    required this.secondaryBg,
    required this.accentFg,
    required this.errorFg,
    required this.successFg,
  });

  static const dark = Theme(
    primaryFg: 'white',
    primaryBg: 'black',
    secondaryFg: 'gray',
    secondaryBg: 'darkGray',
    accentFg: 'cyan',
    errorFg: 'red',
    successFg: 'green',
  );

  static const light = Theme(
    primaryFg: 'black',
    primaryBg: 'white',
    secondaryFg: 'darkGray',
    secondaryBg: 'lightGray',
    accentFg: 'blue',
    errorFg: 'red',
    successFg: 'green',
  );
}

// Use with components
class ThemedComponent extends Component {
  final Theme theme;

  ThemedComponent({
    required this.theme,
    required super.x,
    required super.y,
    required super.width,
    required super.height,
  });

  @override
  void render(Console console) {
    // Use theme.accentFg for highlights, etc.
  }
}
```

## Complete Example: Log Viewer TUI

Putting it all together:

```dart
import 'dart:io';
import 'package:dart_console/dart_console.dart';

class LogEntry {
  final DateTime timestamp;
  final String level;
  final String message;

  LogEntry(this.timestamp, this.level, this.message);
}

class LogViewer {
  final Console console;
  final List<LogEntry> logs;
  final KeyboardHandler keyboard = KeyboardHandler();

  int selectedIndex = 0;
  int scrollOffset = 0;
  String filterLevel = 'ALL';

  LogViewer(this.console, this.logs) {
    _setupKeyBindings();
  }

  void _setupKeyBindings() {
    keyboard.bind('j', 'Move down', () {
      if (selectedIndex < _filteredLogs.length - 1) {
        selectedIndex++;
        _adjustScroll();
      }
    });

    keyboard.bind('k', 'Move up', () {
      if (selectedIndex > 0) {
        selectedIndex--;
        _adjustScroll();
      }
    });

    keyboard.bind('DOWN', 'Move down', keyboard.bindings['j']!.action);
    keyboard.bind('UP', 'Move up', keyboard.bindings['k']!.action);

    keyboard.bind('f', 'Filter', _showFilterDialog);
    keyboard.bind('q', 'Quit', () => exit(0));
  }

  List<LogEntry> get _filteredLogs {
    if (filterLevel == 'ALL') return logs;
    return logs.where((log) => log.level == filterLevel).toList();
  }

  void _adjustScroll() {
    final visibleLines = console.windowHeight - 4;

    if (selectedIndex < scrollOffset) {
      scrollOffset = selectedIndex;
    } else if (selectedIndex >= scrollOffset + visibleLines) {
      scrollOffset = selectedIndex - visibleLines + 1;
    }
  }

  void _showFilterDialog() {
    final dialog = Dialog(
      title: 'Filter Logs',
      message: 'Choose log level to display',
      buttons: ['ALL', 'ERROR', 'WARN', 'INFO', 'DEBUG'],
    );

    final choice = dialog.show(console);
    if (choice >= 0) {
      filterLevel = dialog.buttons[choice];
      selectedIndex = 0;
      scrollOffset = 0;
    }
  }

  void run() {
    console.hideCursor();

    try {
      while (true) {
        render();

        final key = console.readKey();
        keyboard.handle(key);
      }
    } finally {
      console.clearScreen();
      console.showCursor();
    }
  }

  void render() {
    console.clearScreen();

    // Header
    console.writeAt('═' * console.windowWidth, 0, 0);
    console.writeAt(' Log Viewer ', 2, 0, TextStyle.bold);
    console.writeAt('Filter: $filterLevel ', console.windowWidth - 20, 0);

    // Log entries
    final filtered = _filteredLogs;
    final visibleLines = console.windowHeight - 4;
    final endIndex = (scrollOffset + visibleLines).clamp(0, filtered.length);

    for (var i = scrollOffset; i < endIndex; i++) {
      final log = filtered[i];
      final y = 2 + (i - scrollOffset);
      final isSelected = i == selectedIndex;

      final time = log.timestamp.toString().substring(11, 19);
      final level = log.level.padRight(5);
      final message = log.message.length > console.windowWidth - 20
          ? log.message.substring(0, console.windowWidth - 23) + '...'
          : log.message;

      final line = '$time [$level] $message';

      final style = _getLogStyle(log.level);

      if (isSelected) {
        console.writeAt(line, 0, y, TextStyle.inverse);
      } else {
        console.writeAt(line, 0, y, style);
      }
    }

    // Footer
    final footerY = console.windowHeight - 1;
    console.writeAt('─' * console.windowWidth, 0, footerY - 1);
    console.writeAt('[j/k/↑↓] Navigate  [f] Filter  [q] Quit', 2, footerY);
    console.writeAt('${selectedIndex + 1}/${filtered.length}',
        console.windowWidth - 15, footerY);
  }

  TextStyle _getLogStyle(String level) {
    return switch (level) {
      'ERROR' => TextStyle.red,
      'WARN' => TextStyle.yellow,
      'INFO' => TextStyle.cyan,
      _ => TextStyle(),
    };
  }
}

void main() {
  // Sample log data
  final logs = [
    LogEntry(DateTime.now(), 'INFO', 'Application started'),
    LogEntry(DateTime.now(), 'DEBUG', 'Loading configuration'),
    LogEntry(DateTime.now(), 'INFO', 'Connected to database'),
    LogEntry(DateTime.now(), 'WARN', 'High memory usage detected'),
    LogEntry(DateTime.now(), 'ERROR', 'Failed to process request'),
    // ... more logs
  ];

  final console = Console();
  final viewer = LogViewer(console, logs);
  viewer.run();
}
```

This log viewer has:
- Vim-style navigation (hjkl + arrows)
- Color-coded log levels
- Filtering by level
- Modal dialog for filter selection
- Scrolling for long logs
- Status footer with position

## Best Practices for Complex TUIs

1. **Use components** — break UI into reusable pieces
2. **Separate rendering from logic** — keep state separate from display
3. **Implement proper focus management** — track which component is active
4. **Handle resize gracefully** — rebuild layout when terminal changes size
5. **Provide keyboard shortcuts** — both vim-style and arrow keys
6. **Show help** — always display available keys
7. **Use themes** — make colors configurable
8. **Test on different terminals** — iTerm, Terminal.app, Windows Terminal, etc.
9. **Profile performance** — avoid unnecessary redraws
10. **Document your keybindings** — users need to know what's available

## What's Next?

You now have the tools to build sophisticated TUIs! You've learned:

- Component architecture
- Tables with sorting and navigation
- Tree views for hierarchical data
- Split panes for multi-view layouts
- Modal dialogs
- Keyboard binding systems
- Themes and styling

In the next chapter, we'll tackle an often-overlooked topic: **testing CLI apps**. How do you test something that interacts with stdin, stdout, and the terminal?

Time to make your code bulletproof!

---

**[← Previous: Building a TUI](09-building-a-tui.md)** | **[Next: Testing CLI Apps →](11-testing-cli-apps.md)**
