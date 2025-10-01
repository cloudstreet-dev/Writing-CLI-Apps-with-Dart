# Chapter 12: Packaging and Distribution

**Getting your masterpiece into users' hands**

You've built an amazing CLI tool. It's tested, polished, and ready. Now what?

This final chapter covers the entire journey from source code to a tool users can install with a single command:

- Compiling to native executables
- Cross-platform builds
- Publishing to pub.dev
- Creating installers (Homebrew, Chocolatey, etc.)
- Versioning and changelogs
- Auto-update mechanisms
- Distribution strategies

Let's ship this thing!

## Compiling to Native Executables

Dart can compile your CLI app to a standalone native executable — no runtime required.

### Basic Compilation

```bash
# Compile to executable
$ dart compile exe bin/myapp.dart -o myapp

# The output is a native binary
$ ls -lh myapp
-rwxr-xr-x  1 user  staff   6.2M Oct  1 12:00 myapp

# Run it
$ ./myapp --help
```

The executable includes the Dart runtime, so users don't need Dart installed.

### Compilation Options

```bash
# Optimize for size
$ dart compile exe bin/myapp.dart -o myapp --target-os=macos

# Optimize for speed (default)
$ dart compile exe bin/myapp.dart -o myapp

# For a specific platform
$ dart compile exe bin/myapp.dart -o myapp --target-os=linux
$ dart compile exe bin/myapp.dart -o myapp --target-os=windows
$ dart compile exe bin/myapp.dart -o myapp --target-os=macos
```

### Cross-Compilation Limitations

**Important**: Dart doesn't support true cross-compilation. To build for Linux, you need to run the compile command on Linux. To build for Windows, you need Windows.

Solutions:
1. **Use CI/CD** — build on multiple platforms in parallel
2. **Use GitHub Actions** — provides Linux, macOS, and Windows runners
3. **Use Docker** — for Linux builds on any platform

## Building for Multiple Platforms

### Using GitHub Actions

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v*'  # Trigger on version tags (v1.0.0, v2.1.3, etc.)

jobs:
  build:
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
        include:
          - os: ubuntu-latest
            output: myapp-linux
            target: linux
          - os: macos-latest
            output: myapp-macos
            target: macos
          - os: windows-latest
            output: myapp-windows.exe
            target: windows

    runs-on: ${{ matrix.os }}

    steps:
      - uses: actions/checkout@v3

      - uses: dart-lang/setup-dart@v1
        with:
          sdk: stable

      - name: Install dependencies
        run: dart pub get

      - name: Run tests
        run: dart test

      - name: Compile executable
        run: dart compile exe bin/myapp.dart -o ${{ matrix.output }}

      - name: Upload artifact
        uses: actions/upload-artifact@v3
        with:
          name: ${{ matrix.output }}
          path: ${{ matrix.output }}

      - name: Create release
        if: startsWith(github.ref, 'refs/tags/')
        uses: softprops/action-gh-release@v1
        with:
          files: ${{ matrix.output }}
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

Now when you push a tag:

```bash
$ git tag v1.0.0
$ git push origin v1.0.0
```

GitHub Actions will:
1. Build for Linux, macOS, and Windows
2. Run tests on all platforms
3. Create a GitHub Release with all binaries

Users can download the binary for their platform directly from GitHub Releases!

## Publishing to pub.dev

For Dart developers, publish to pub.dev:

### Prepare Your Package

```yaml
# pubspec.yaml
name: myapp
description: A fantastic CLI tool that does amazing things
version: 1.0.0
repository: https://github.com/yourname/myapp

environment:
  sdk: ^3.0.0

executables:
  myapp: myapp  # Allows 'dart pub global activate myapp'

dependencies:
  args: ^2.4.0
  # ... other dependencies

dev_dependencies:
  test: ^1.24.0
```

The `executables` section tells pub that this package provides a CLI command.

### Publish

```bash
# First time: do a dry-run
$ dart pub publish --dry-run

# Review the output, then publish for real
$ dart pub publish
```

Now users can install via:

```bash
$ dart pub global activate myapp
$ myapp --help
```

### Update

When you release a new version:

```bash
# 1. Update version in pubspec.yaml
version: 1.1.0

# 2. Update CHANGELOG.md

# 3. Commit and tag
$ git commit -am "Release v1.1.0"
$ git tag v1.1.0
$ git push origin main --tags

# 4. Publish
$ dart pub publish
```

Users update with:

```bash
$ dart pub global activate myapp  # Gets latest version
```

## Creating a Homebrew Formula (macOS/Linux)

Homebrew is the most popular package manager for macOS (and Linux).

### Step 1: Create a Tap

A "tap" is a third-party Homebrew repository:

```bash
# Create a repo named homebrew-tap
$ mkdir homebrew-tap
$ cd homebrew-tap
$ git init
```

### Step 2: Write a Formula

```ruby
# Formula/myapp.rb
class Myapp < Formula
  desc "A fantastic CLI tool that does amazing things"
  homepage "https://github.com/yourname/myapp"
  url "https://github.com/yourname/myapp/archive/v1.0.0.tar.gz"
  sha256 "abc123..."  # SHA256 of the tarball
  license "MIT"

  depends_on "dart-lang/dart/dart" => :build

  def install
    # Install dependencies
    system "dart", "pub", "get"

    # Compile executable
    system "dart", "compile", "exe", "bin/myapp.dart", "-o", "myapp"

    # Install to bin
    bin.install "myapp"
  end

  test do
    assert_match "myapp version 1.0.0", shell_output("#{bin}/myapp --version")
  end
end
```

Calculate SHA256:

```bash
$ curl -L https://github.com/yourname/myapp/archive/v1.0.0.tar.gz | shasum -a 256
```

### Step 3: Push to GitHub

```bash
$ git add Formula/myapp.rb
$ git commit -m "Add myapp formula"
$ git push origin main
```

### Step 4: Users Install

```bash
# Add your tap
$ brew tap yourname/tap

# Install your app
$ brew install myapp

# Run it
$ myapp --help
```

### Automated Updates with GitHub Actions

```yaml
# .github/workflows/update-homebrew.yml
name: Update Homebrew Formula

on:
  release:
    types: [published]

jobs:
  update-formula:
    runs-on: ubuntu-latest
    steps:
      - name: Update Homebrew formula
        uses: dawidd6/action-homebrew-bump-formula@v3
        with:
          token: ${{ secrets.TAP_GITHUB_TOKEN }}
          formula: myapp
          tap: yourname/homebrew-tap
```

Now your Homebrew formula auto-updates when you create a GitHub Release!

## Creating a Chocolatey Package (Windows)

Chocolatey is the Homebrew of Windows.

### Step 1: Install Chocolatey Locally

```powershell
Set-ExecutionPolicy Bypass -Scope Process -Force
iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))
```

### Step 2: Create Package

```bash
$ choco new myapp
```

This creates:
```
myapp/
├── myapp.nuspec
└── tools/
    └── chocolateyinstall.ps1
```

### Step 3: Edit nuspec File

```xml
<!-- myapp.nuspec -->
<?xml version="1.0"?>
<package>
  <metadata>
    <id>myapp</id>
    <version>1.0.0</version>
    <title>MyApp</title>
    <authors>Your Name</authors>
    <description>A fantastic CLI tool</description>
    <projectUrl>https://github.com/yourname/myapp</projectUrl>
    <tags>cli tool awesome</tags>
    <licenseUrl>https://github.com/yourname/myapp/blob/main/LICENSE</licenseUrl>
  </metadata>
</package>
```

### Step 4: Edit Install Script

```powershell
# tools/chocolateyinstall.ps1
$ErrorActionPreference = 'Stop'

$packageName = 'myapp'
$url = 'https://github.com/yourname/myapp/releases/download/v1.0.0/myapp-windows.exe'
$checksum = 'ABC123...'  # SHA256 checksum

Install-ChocolateyPackage `
  -PackageName $packageName `
  -FileType 'exe' `
  -Url $url `
  -Checksum $checksum `
  -ChecksumType 'sha256'
```

### Step 5: Pack and Publish

```powershell
# Create package
choco pack

# Test locally
choco install myapp -source .

# Publish to Chocolatey.org
choco push myapp.1.0.0.nupkg --source https://push.chocolatey.org/ --api-key YOUR_KEY
```

Users install with:

```powershell
choco install myapp
```

## Versioning and Changelogs

Follow [Semantic Versioning](https://semver.org/):

- **Major** (1.0.0 → 2.0.0): Breaking changes
- **Minor** (1.0.0 → 1.1.0): New features, backwards compatible
- **Patch** (1.0.0 → 1.0.1): Bug fixes

### CHANGELOG.md

Keep a changelog following [Keep a Changelog](https://keepachangelog.com/):

```markdown
# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/),
and this project adheres to [Semantic Versioning](https://semver.org/).

## [Unreleased]

### Added
- Feature in development

## [1.1.0] - 2024-01-15

### Added
- New `--verbose` flag for detailed output
- Support for YAML configuration files

### Changed
- Improved error messages
- Updated dependencies

### Fixed
- Fixed crash when processing empty files
- Fixed progress bar flicker on Windows

## [1.0.0] - 2024-01-01

### Added
- Initial release
- Basic file processing
- Colorized output
- Configuration file support

[Unreleased]: https://github.com/yourname/myapp/compare/v1.1.0...HEAD
[1.1.0]: https://github.com/yourname/myapp/compare/v1.0.0...v1.1.0
[1.0.0]: https://github.com/yourname/myapp/releases/tag/v1.0.0
```

### Automatic Versioning

Use a tool like `cider`:

```bash
$ dart pub global activate cider

# Add a change
$ cider log added "New feature"

# Bump version
$ cider bump minor

# Generate changelog
$ cider release
```

## Auto-Update Mechanism

Let your CLI tool update itself:

```dart
// lib/updater.dart
import 'dart:io';
import 'package:http/http.dart' as http;
import 'package:pub_semver/pub_semver.dart';

class Updater {
  final String currentVersion;
  final String repoOwner;
  final String repoName;

  Updater({
    required this.currentVersion,
    required this.repoOwner,
    required this.repoName,
  });

  Future<String?> checkForUpdates() async {
    try {
      final url = 'https://api.github.com/repos/$repoOwner/$repoName/releases/latest';
      final response = await http.get(Uri.parse(url));

      if (response.statusCode != 200) return null;

      final json = jsonDecode(response.body);
      final latestVersion = json['tag_name'].toString().replaceAll('v', '');

      final current = Version.parse(currentVersion);
      final latest = Version.parse(latestVersion);

      if (latest > current) {
        return latestVersion;
      }

      return null;  // Already up to date
    } catch (e) {
      stderr.writeln('Failed to check for updates: $e');
      return null;
    }
  }

  Future<void> performUpdate(String version) async {
    final platform = _getPlatform();
    final url = 'https://github.com/$repoOwner/$repoName/releases/download/v$version/myapp-$platform';

    print('Downloading version $version...');

    final response = await http.get(Uri.parse(url));
    if (response.statusCode != 200) {
      stderr.writeln('Failed to download update');
      return;
    }

    // Get current executable path
    final currentExe = Platform.resolvedExecutable;
    final tempPath = '$currentExe.new';

    // Write new version
    await File(tempPath).writeAsBytes(response.bodyBytes);

    // Make executable (Unix)
    if (!Platform.isWindows) {
      await Process.run('chmod', ['+x', tempPath]);
    }

    print('Update downloaded. Restart to apply.');
    print('Run: mv $tempPath $currentExe');
  }

  String _getPlatform() {
    if (Platform.isLinux) return 'linux';
    if (Platform.isMacOS) return 'macos';
    if (Platform.isWindows) return 'windows.exe';
    throw UnsupportedError('Platform not supported');
  }
}

// Usage in your CLI
void main(List<String> args) async {
  if (args.contains('update')) {
    final updater = Updater(
      currentVersion: '1.0.0',
      repoOwner: 'yourname',
      repoName: 'myapp',
    );

    final newVersion = await updater.checkForUpdates();

    if (newVersion != null) {
      print('New version available: $newVersion');
      print('Current version: ${updater.currentVersion}');

      final confirm = Confirm(
        prompt: 'Update now?',
        defaultValue: true,
      ).interact();

      if (confirm) {
        await updater.performUpdate(newVersion);
      }
    } else {
      print('Already up to date!');
    }

    return;
  }

  // Regular app logic...
}
```

Users update with:

```bash
$ myapp update
New version available: 1.1.0
Current version: 1.0.0
Update now? (Y/n): y
Downloading version 1.1.0...
Update downloaded. Restart to apply.
```

## Distribution Strategies

### 1. GitHub Releases (Simple)

**Pros**: Free, simple, works for all platforms
**Cons**: Manual downloads, not in package managers

Best for: Small projects, early releases

### 2. pub.dev (Dart Developers)

**Pros**: Easy for Dart devs, automatic updates
**Cons**: Requires Dart SDK, only Dart community

Best for: Tools for Dart/Flutter developers

### 3. Homebrew (macOS/Linux)

**Pros**: Most popular on macOS, trusted
**Cons**: macOS/Linux only, formula maintenance

Best for: Developer tools, Unix-y tools

### 4. Chocolatey (Windows)

**Pros**: De facto Windows package manager
**Cons**: Windows only, less popular than Homebrew

Best for: Tools targeting Windows users

### 5. All of the Above!

Use GitHub Actions to automate releases to all platforms:

```yaml
name: Multi-Platform Release

on:
  release:
    types: [published]

jobs:
  # Build binaries
  build:
    # ... (matrix build for Linux/macOS/Windows)

  # Publish to pub.dev
  publish-pub:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: dart pub publish --force

  # Update Homebrew
  update-homebrew:
    runs-on: ubuntu-latest
    steps:
      - uses: dawidd6/action-homebrew-bump-formula@v3

  # Update Chocolatey
  update-chocolatey:
    runs-on: windows-latest
    steps:
      # ... chocolatey push steps
```

## Documentation for Distribution

### README.md

```markdown
# MyApp

A fantastic CLI tool that does amazing things.

## Installation

### macOS/Linux (Homebrew)

\`\`\`bash
brew tap yourname/tap
brew install myapp
\`\`\`

### Windows (Chocolatey)

\`\`\`powershell
choco install myapp
\`\`\`

### Dart Developers

\`\`\`bash
dart pub global activate myapp
\`\`\`

### Pre-built Binaries

Download from [GitHub Releases](https://github.com/yourname/myapp/releases)

### From Source

\`\`\`bash
git clone https://github.com/yourname/myapp.git
cd myapp
dart pub get
dart compile exe bin/myapp.dart -o myapp
\`\`\`

## Usage

\`\`\`bash
myapp --help
\`\`\`

## Documentation

See [docs/](docs/) for full documentation.

## License

MIT
```

## Security Considerations

### Code Signing

For production tools, sign your executables:

**macOS**:
```bash
# Get Developer ID certificate from Apple
codesign --force --sign "Developer ID Application: Your Name" myapp

# Notarize for Gatekeeper
xcrun notarytool submit myapp.zip --apple-id you@example.com --wait
```

**Windows**:
```bash
# Get code signing certificate
signtool sign /f certificate.pfx /p password myapp.exe
```

### Dependency Auditing

Check for vulnerabilities:

```bash
# Analyze dependencies
$ dart pub outdated

# Update to latest secure versions
$ dart pub upgrade
```

### Checksums

Always provide SHA256 checksums:

```bash
# Generate checksum
$ sha256sum myapp-linux > myapp-linux.sha256

# Publish alongside binary
```

Users verify:

```bash
$ sha256sum -c myapp-linux.sha256
myapp-linux: OK
```

## Final Checklist

Before shipping 1.0:

- [ ] Tests pass on all platforms
- [ ] Documentation is complete
- [ ] CHANGELOG.md is up to date
- [ ] Version number follows semver
- [ ] LICENSE file exists
- [ ] README has installation instructions
- [ ] Binaries are code-signed (if applicable)
- [ ] Checksums are provided
- [ ] GitHub Release is created
- [ ] Package managers are updated

## You Did It!

Congratulations! You've built, tested, and shipped a professional CLI tool in Dart.

You now know:

- ✅ Why Dart is great for CLI apps
- ✅ Argument parsing like a pro
- ✅ Files, pipes, and Unix philosophy
- ✅ Colorful, beautiful output
- ✅ Interactive prompts
- ✅ Progress bars and spinners
- ✅ Configuration management
- ✅ Graceful error handling
- ✅ Building full TUI applications
- ✅ Advanced TUI patterns
- ✅ Testing CLI apps thoroughly
- ✅ Packaging and distribution

## What's Next?

Keep building! Ideas for your next CLI project:

- **Developer tools**: Build tools, linters, code generators
- **System utilities**: Log analyzers, file processors, backup tools
- **Productivity apps**: Task managers, note-taking, time trackers
- **TUI applications**: File browsers, system monitors, chat clients
- **Data tools**: CSV/JSON processors, API clients, data converters

The terminal is your canvas. Go create something awesome!

---

## Resources

- [Dart documentation](https://dart.dev)
- [pub.dev packages](https://pub.dev)
- [dart_console package](https://pub.dev/packages/dart_console)
- [args package](https://pub.dev/packages/args)
- [GitHub Actions documentation](https://docs.github.com/en/actions)
- [Homebrew formula docs](https://docs.brew.sh/Formula-Cookbook)
- [Chocolatey package docs](https://docs.chocolatey.org/en-us/create/create-packages)

## Thank You!

Thanks for reading **Writing CLI Apps with Dart**! I hope you enjoyed the journey and learned something useful.

If you build something cool, I'd love to hear about it. Share your creations!

Happy coding! 🎉

---

**[← Previous: Testing CLI Apps](11-testing-cli-apps.md)** | **[Back to Table of Contents](00-table-of-contents.md)**
