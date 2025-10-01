# Table of Contents

## Part I: Foundations

### [Chapter 1: Why Dart for CLI?](01-why-dart-for-cli.md)
**In which we justify our life choices**

Dart is usually associated with Flutter and mobile apps. So why on earth would you use it for command-line tools? This chapter makes the case: blazing-fast startup times, single-binary distribution, excellent tooling, and a type system that actually helps instead of getting in your way. You'll learn about Dart's compilation modes, compare it to other popular CLI languages, and understand why Dart might just be the secret weapon in your terminal toolkit.

**Topics**: AOT compilation, Dart VM, startup performance, ecosystem overview, when to use (and not use) Dart for CLI

---

### [Chapter 2: Hello, Arguments!](02-hello-arguments.md)
**Parsing arguments without losing your mind**

Every CLI app starts with parsing arguments. We'll use the excellent `args` package to handle flags, options, commands, and subcommands. You'll build a real argument parser that handles `--help`, validates input, and provides helpful error messages. By the end, you'll never write `if (args[0] == '--help')` again.

**Topics**: The `args` package, flags vs options, positional arguments, subcommands, help text generation, validation

**Example Project**: `greeter` — A friendly CLI tool with multiple greeting styles and configuration options

---

### [Chapter 3: Files and Pipes](03-files-and-pipes.md)
**Being a good Unix citizen**

The Unix philosophy: do one thing well, and play nicely with others. This chapter covers reading and writing files, working with stdin/stdout/stderr, handling pipes, and respecting Unix conventions. You'll learn how to build tools that integrate seamlessly into shell pipelines and respect the user's data.

**Topics**: File I/O, `dart:io`, stdin/stdout/stderr, pipes and redirection, buffering, text encoding, stream processing

**Example Project**: `wordcount` — A `wc` clone that demonstrates proper pipe handling

---

### [Chapter 4: Pretty Colors](04-pretty-colors.md)
**Making your terminal output fabulous**

Monochrome terminal output is so 1985. Learn how to add colors, bold text, underlining, and more using ANSI escape codes. We'll use packages like `ansi_styles` and `tint` to make output that's both beautiful and readable. Plus: how to detect terminal capabilities and respect `NO_COLOR`.

**Topics**: ANSI escape codes, the `ansi_styles` package, terminal capability detection, color schemes, semantic colors, accessibility

**Example Project**: `log-viewer` — A colorful log file viewer with syntax highlighting

---

## Part II: Interactivity

### [Chapter 5: Interactive Prompts](05-interactive-prompts.md)
**Talking to humans (scary, I know)**

Sometimes you need to ask the user questions. This chapter covers interactive prompts: yes/no questions, text input, selections, multi-select, and confirmation dialogs. You'll learn how to build UIs that are intuitive, forgiving, and don't make users want to throw their keyboards.

**Topics**: The `interact` and `cli_dialog` packages, input validation, default values, masking passwords, list selections, confirmation prompts

**Example Project**: `project-init` — An interactive project scaffolding tool (like `npm init`)

---

### [Chapter 6: Progress and Spinners](06-progress-and-spinners.md)
**The art of "please wait..."**

Long-running operations need feedback. Learn to implement progress bars, spinners, and status indicators that keep users informed without being annoying. We'll cover determinate and indeterminate progress, multi-stage operations, and how to handle errors gracefully.

**Topics**: The `cli_util` and `progress_bar` packages, spinners, progress bars, ETAs, multi-step progress, updating messages

**Example Project**: `downloader` — A file downloader with beautiful progress display

---

### [Chapter 7: Configuration Files](07-configuration-files.md)
**Where to hide your secrets (and settings)**

CLI tools need configuration. This chapter covers reading and writing config files in YAML, JSON, and TOML formats. You'll learn about the XDG Base Directory specification, configuration precedence (system vs user vs environment), and how to provide sensible defaults.

**Topics**: YAML, JSON, TOML parsing, XDG directories, config precedence, validation, schema, environment variables, secrets management

**Example Project**: `task-manager` — A task tracker with persistent configuration

---

### [Chapter 8: Error Handling](08-error-handling.md)
**Failing gracefully like a professional**

Things go wrong. Disks fill up, networks fail, users provide invalid input. This chapter is about handling errors gracefully: meaningful error messages, proper exit codes, stack traces (when appropriate), logging, and recovery strategies. Your users will thank you.

**Topics**: Exit codes, stderr usage, stack traces, the `Result` pattern, exceptions vs errors, logging packages, user-friendly error messages

**Example Project**: `backup-tool` — A backup utility with comprehensive error handling

---

## Part III: Terminal User Interfaces

### [Chapter 9: Building a TUI](09-building-a-tui.md)
**Moving beyond line-based output**

Time to level up: full-screen terminal UIs. This chapter introduces TUI concepts: raw terminal mode, cursor control, keyboard input, screen layouts, and event loops. You'll build your first interactive TUI application with widgets and layouts.

**Topics**: Raw mode, terminal control sequences, `dart_console` package, event loops, keyboard input, mouse support, layouts, widgets, screen management

**Example Project**: `file-browser` — A simple interactive file browser (like `ranger` or `midnight commander`)

---

### [Chapter 10: Advanced TUI Patterns](10-advanced-tui.md)
**Complex interfaces and sophisticated interactions**

Now that you know the basics, let's build something impressive. This chapter covers advanced TUI patterns: tables with sorting, tree views, modal dialogs, split panes, tabs, vim-style keybindings, and themes. You'll learn architectural patterns for complex TUI applications.

**Topics**: Component architecture, state management, tables and trees, modals and overlays, keybinding systems, themes and styling, performance optimization

**Example Project**: `log-analyzer` — A sophisticated log analysis TUI with multiple views and filters

---

## Part IV: Production

### [Chapter 11: Testing CLI Apps](11-testing-cli-apps.md)
**How to test the "untestable"**

CLI apps interact with stdin, stdout, files, and the terminal. How do you test that? This chapter covers unit testing, integration testing, snapshot testing for output, mocking the filesystem, testing interactive prompts, and CI/CD strategies for CLI tools.

**Topics**: Unit testing with `test`, mocking `dart:io`, testing output, snapshot testing, integration tests, testing interactive features, CI/CD, test coverage

**Example Project**: Comprehensive test suite for the projects built in previous chapters

---

### [Chapter 12: Packaging and Distribution](12-packaging-and-distribution.md)
**Getting your masterpiece into users' hands**

You've built something awesome — now what? This chapter covers compiling to native binaries, creating installers, publishing to package managers (Homebrew, apt, Chocolatey), auto-updates, and making installation painless. Plus: versioning, changelogs, and user documentation.

**Topics**: `dart compile`, cross-compilation, standalone executables, installer creation, Homebrew formulas, apt/deb packages, Chocolatey, pub.dev publishing, auto-update mechanisms, semantic versioning

**Example Project**: Full distribution setup for a production-ready CLI tool

---

## Appendices

### Appendix A: Quick Reference
Common patterns, escape codes, and package cheat sheets

### Appendix B: Recommended Packages
Curated list of the best Dart packages for CLI development

### Appendix C: Resources
Links to documentation, communities, and further reading

---

## About the Author

**Claude Code Sonnet 4.5** is an AI assistant who has written exactly one book (this one). Despite being made of math and linear algebra, they have strong opinions about terminal color schemes and believe that every CLI tool deserves a good `--help` message. They enjoy long walks through codebases and arguing about whether tabs or spaces are superior (spaces, obviously).

---

**Ready to start?** → [Chapter 1: Why Dart for CLI?](01-why-dart-for-cli.md)
