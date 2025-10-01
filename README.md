# Writing CLI Apps with Dart

**By Claude Code Sonnet 4.5**

> "Why are you writing CLI apps in Dart?" — Every developer, before reading this book
> "Oh, that's actually really nice!" — Every developer, after reading this book

## What This Book Is About

You've probably used Dart for Flutter. Maybe you've built beautiful mobile apps with smooth animations and pixel-perfect UIs. That's great! But did you know Dart is also secretly excellent for writing command-line tools?

This book will teach you how to build CLI applications in Dart — from simple argument-parsing utilities to complex Terminal User Interface (TUI) applications that would make `htop` jealous. We'll cover:

- 🎯 Argument parsing that doesn't make you cry
- 📁 File I/O and Unix pipe etiquette
- 🌈 Making your terminal output *fabulous*
- ⌨️  Interactive prompts and user input
- ⏳ Progress bars and spinners (the good kind)
- 🎨 Full-blown TUI applications with widgets and layouts
- ✅ Testing strategies that actually work
- 📦 Packaging and distributing your masterpiece

## Who This Book Is For

This book assumes you:

- Know how to write code (in any language)
- Are at least somewhat familiar with Dart syntax
- Have written CLI applications in other languages
- Want to learn how to build sophisticated terminal interfaces
- Enjoy the occasional joke in technical writing

If you've never touched Dart before, that's okay! Check out [dart.dev](https://dart.dev) for a quick syntax primer, then come back. We'll wait.

## Why Dart for CLI?

Great question! Skip to Chapter 1 for the full answer, but here's the TL;DR:

- **Fast startup**: Compiled Dart starts faster than you can say "JVM warmup"
- **Easy distribution**: Single binary, no runtime required
- **Great tooling**: The Dart ecosystem has excellent packages for CLI work
- **Type safety**: Catch errors before your users do
- **Actually fun**: Hot reload works for CLI apps too!

## Prerequisites

To follow along with this book, you'll need:

- **Dart SDK** (version 3.0 or later): [Install it here](https://dart.dev/get-dart)
- A **terminal emulator** you enjoy using
- A **text editor or IDE** (VS Code, IntelliJ, or even `vim` if you're feeling spicy)
- **Basic command-line skills**: You should know what `cd`, `ls`, and pipes are
- **Curiosity** and a **sense of humor** (non-negotiable)

## How This Book Is Structured

Each chapter builds on the previous ones, gradually increasing in complexity:

**Chapters 1-4**: Foundation — arguments, I/O, and making things pretty
**Chapters 5-8**: Intermediate — interactivity, configuration, and error handling
**Chapters 9-10**: Advanced — Full TUI applications with complex layouts
**Chapters 11-12**: Production — Testing, packaging, and shipping

You can jump around if you're already familiar with certain topics, but the examples do build on each other. When in doubt, read sequentially.

## Code Examples

All code examples in this book are:

- **Runnable**: Copy-paste and they work
- **Practical**: Based on real-world use cases
- **Progressively complex**: Starting simple, building to sophisticated
- **Documented**: Comments explain the "why," not just the "what"

You'll find complete example projects in each chapter. Don't just read — type them out, break them, fix them, and make them your own.

## A Note on Style

This book aims to be **entertaining while teaching**. You'll find:

- Real-world examples and war stories
- Occasional humor and pop culture references
- Honest discussions of trade-offs and pitfalls
- Strong opinions, weakly held

If you prefer dry, academic technical writing... well, you might be in the wrong place. But give it a shot anyway — you might enjoy yourself.

## Table of Contents

See [Table of Contents](00-table-of-contents.md) for the full chapter list.

## Contributing

Spot a typo? Have a suggestion? Found a better way to do something? Great! This book is open source. File an issue or submit a pull request.

## License

This book is licensed under the Creative Commons Attribution-ShareAlike 4.0 International License. See the [LICENSE](LICENSE) file for details.

---

Ready to build some awesome CLI tools? Let's go!

**[Start with Chapter 1: Why Dart for CLI? →](01-why-dart-for-cli.md)**
