# Chapter 7: Configuration Files

**Where to hide your secrets (and settings)**

Hard-coding configuration is a path to pain. Want to change the database host? Recompile. Want different settings for dev vs production? Too bad. Want to share your tool without sharing your API keys? Hope you like `git reset --hard`.

Configuration files solve this. They let users customize behavior, store credentials, and maintain different profiles without touching code.

This chapter covers everything about config files: formats (YAML, JSON, TOML), locations (where config files should live), precedence (system vs user vs environment), and secrets management.

Let's make your tools configurable!

## Configuration File Formats

Three main formats dominate the config file world:

### JSON: The Universal Format

**Pros**:
- Everyone knows it
- Dart has built-in support
- Easy to parse and generate

**Cons**:
- No comments (!!!!)
- Trailing commas are errors
- Verbose for nested structures

Example:
```json
{
  "server": {
    "host": "localhost",
    "port": 8080,
    "ssl": true
  },
  "database": {
    "url": "postgresql://localhost/mydb",
    "pool_size": 10
  }
}
```

Parsing in Dart:
```dart
import 'dart:convert';
import 'dart:io';

Future<Map<String, dynamic>> loadJsonConfig(String path) async {
  final file = File(path);
  final contents = await file.readAsString();
  return jsonDecode(contents) as Map<String, dynamic>;
}

void main() async {
  final config = await loadJsonConfig('config.json');
  final host = config['server']['host'];
  final port = config['server']['port'];

  print('Server: $host:$port');
}
```

### YAML: The Human-Friendly Format

**Pros**:
- Super readable
- Supports comments
- Less verbose than JSON
- Industry standard (Docker, Kubernetes, GitHub Actions)

**Cons**:
- Whitespace-sensitive (indentation matters)
- Sometimes *too* flexible

Example:
```yaml
# Server configuration
server:
  host: localhost
  port: 8080
  ssl: true

# Database settings
database:
  url: postgresql://localhost/mydb
  pool_size: 10

# Feature flags
features:
  - authentication
  - caching
  - monitoring
```

Using the `yaml` package:

```bash
$ dart pub add yaml
```

```dart
import 'dart:io';
import 'package:yaml/yaml.dart';

Future<Map<String, dynamic>> loadYamlConfig(String path) async {
  final file = File(path);
  final contents = await file.readAsString();
  final yaml = loadYaml(contents);

  // Convert YamlMap to regular Map
  return _yamlToMap(yaml);
}

dynamic _yamlToMap(dynamic value) {
  if (value is YamlMap) {
    return {for (var entry in value.entries) entry.key: _yamlToMap(entry.value)};
  } else if (value is YamlList) {
    return value.map(_yamlToMap).toList();
  }
  return value;
}

void main() async {
  final config = await loadYamlConfig('config.yaml');
  final host = config['server']['host'];
  final features = config['features'] as List;

  print('Server: $host');
  print('Features: ${features.join(', ')}');
}
```

### TOML: The Best of Both Worlds?

**Pros**:
- Readable like YAML
- Unambiguous (less magic)
- Great for configuration

**Cons**:
- Less common than JSON/YAML
- Dart support via third-party packages

Example:
```toml
# Server configuration
[server]
host = "localhost"
port = 8080
ssl = true

# Database settings
[database]
url = "postgresql://localhost/mydb"
pool_size = 10

# Feature flags
features = ["authentication", "caching", "monitoring"]
```

Using the `toml` package:

```bash
$ dart pub add toml
```

```dart
import 'dart:io';
import 'package:toml/toml.dart';

Future<Map<String, dynamic>> loadTomlConfig(String path) async {
  final file = File(path);
  final contents = await file.readAsString();
  final document = TomlDocument.parse(contents);
  return document.toMap();
}
```

**My recommendation**: Use YAML for user-facing configs, JSON for machine-generated configs, and TOML if you like it (it's great!).

## Where to Put Config Files

Don't scatter config files randomly! Follow platform conventions:

### The XDG Base Directory Specification (Linux/macOS)

```dart
import 'dart:io';
import 'package:path/path.dart' as path;

class ConfigPaths {
  static String getConfigDir(String appName) {
    // $XDG_CONFIG_HOME or ~/.config
    final configHome = Platform.environment['XDG_CONFIG_HOME'] ??
        path.join(Platform.environment['HOME']!, '.config');

    return path.join(configHome, appName);
  }

  static String getDataDir(String appName) {
    // $XDG_DATA_HOME or ~/.local/share
    final dataHome = Platform.environment['XDG_DATA_HOME'] ??
        path.join(Platform.environment['HOME']!, '.local', 'share');

    return path.join(dataHome, appName);
  }

  static String getCacheDir(String appName) {
    // $XDG_CACHE_HOME or ~/.cache
    final cacheHome = Platform.environment['XDG_CACHE_HOME'] ??
        path.join(Platform.environment['HOME']!, '.cache');

    return path.join(cacheHome, appName);
  }
}

void main() {
  print('Config: ${ConfigPaths.getConfigDir('mytool')}');
  // Linux: /home/user/.config/mytool
  // macOS: /Users/user/.config/mytool

  print('Data: ${ConfigPaths.getDataDir('mytool')}');
  // Linux: /home/user/.local/share/mytool
  // macOS: /Users/user/.local/share/mytool

  print('Cache: ${ConfigPaths.getCacheDir('mytool')}');
  // Linux: /home/user/.cache/mytool
  // macOS: /Users/user/.cache/mytool
}
```

### Windows Paths

Windows has its own conventions:

```dart
String getWindowsConfigDir(String appName) {
  final appData = Platform.environment['APPDATA'];
  if (appData != null) {
    return path.join(appData, appName);
  }

  // Fallback
  final home = Platform.environment['USERPROFILE'];
  return path.join(home!, appName);
}
```

### Cross-Platform Helper

```dart
String getConfigPath(String appName, String filename) {
  if (Platform.isWindows) {
    final appData = Platform.environment['APPDATA']!;
    return path.join(appData, appName, filename);
  } else {
    // Linux/macOS
    final configHome = Platform.environment['XDG_CONFIG_HOME'] ??
        path.join(Platform.environment['HOME']!, '.config');
    return path.join(configHome, appName, filename);
  }
}

void main() {
  final configPath = getConfigPath('mytool', 'config.yaml');
  print('Config file: $configPath');

  // Windows: C:\Users\username\AppData\Roaming\mytool\config.yaml
  // Linux:   /home/username/.config/mytool/config.yaml
  // macOS:   /Users/username/.config/mytool/config.yaml
}
```

## Configuration Precedence

Real tools support multiple config sources with clear precedence:

1. **Command-line arguments** (highest priority)
2. **Environment variables**
3. **User config file** (`~/.config/mytool/config.yaml`)
4. **System config file** (`/etc/mytool/config.yaml`)
5. **Built-in defaults** (lowest priority)

Example:

```dart
import 'dart:io';
import 'package:args/args.dart';
import 'package:yaml/yaml.dart';
import 'package:path/path.dart' as path;

class Config {
  final String host;
  final int port;
  final bool debug;

  Config({
    required this.host,
    required this.port,
    required this.debug,
  });

  static Future<Config> load(List<String> args) async {
    // 1. Defaults
    var host = 'localhost';
    var port = 8080;
    var debug = false;

    // 2. System config
    final systemConfig = File('/etc/mytool/config.yaml');
    if (await systemConfig.exists()) {
      final data = await _loadYaml(systemConfig);
      host = data['host'] ?? host;
      port = data['port'] ?? port;
      debug = data['debug'] ?? debug;
    }

    // 3. User config
    final userConfigPath = getConfigPath('mytool', 'config.yaml');
    final userConfig = File(userConfigPath);
    if (await userConfig.exists()) {
      final data = await _loadYaml(userConfig);
      host = data['host'] ?? host;
      port = data['port'] ?? port;
      debug = data['debug'] ?? debug;
    }

    // 4. Environment variables
    host = Platform.environment['MYTOOL_HOST'] ?? host;
    port = int.tryParse(Platform.environment['MYTOOL_PORT'] ?? '') ?? port;
    debug = Platform.environment['MYTOOL_DEBUG'] == 'true' || debug;

    // 5. Command-line arguments (highest priority)
    final parser = ArgParser()
      ..addOption('host')
      ..addOption('port')
      ..addFlag('debug');

    final results = parser.parse(args);

    host = results['host'] ?? host;
    port = int.tryParse(results['port'] ?? '') ?? port;
    debug = results['debug'] || debug;

    return Config(host: host, port: port, debug: debug);
  }

  static Future<Map<String, dynamic>> _loadYaml(File file) async {
    final contents = await file.readAsString();
    final yaml = loadYaml(contents);
    return {for (var entry in yaml.entries) entry.key: entry.value};
  }
}

String getConfigPath(String appName, String filename) {
  if (Platform.isWindows) {
    return path.join(Platform.environment['APPDATA']!, appName, filename);
  }
  final configHome = Platform.environment['XDG_CONFIG_HOME'] ??
      path.join(Platform.environment['HOME']!, '.config');
  return path.join(configHome, appName, filename);
}

void main(List<String> args) async {
  final config = await Config.load(args);

  print('Host:  ${config.host}');
  print('Port:  ${config.port}');
  print('Debug: ${config.debug}');
}
```

Now users can configure your tool in any way:

```bash
# Use defaults
$ mytool

# Override via config file
$ echo "host: example.com" > ~/.config/mytool/config.yaml
$ mytool

# Override via environment
$ MYTOOL_PORT=3000 mytool

# Override via CLI args (highest priority)
$ mytool --host production.com --port 443 --debug
```

## Type-Safe Configuration with Classes

Parsing raw maps is error-prone. Use classes:

```dart
class ServerConfig {
  final String host;
  final int port;
  final bool ssl;

  ServerConfig({
    required this.host,
    required this.port,
    required this.ssl,
  });

  factory ServerConfig.fromMap(Map<String, dynamic> map) {
    return ServerConfig(
      host: map['host'] as String? ?? 'localhost',
      port: map['port'] as int? ?? 8080,
      ssl: map['ssl'] as bool? ?? false,
    );
  }

  Map<String, dynamic> toMap() {
    return {
      'host': host,
      'port': port,
      'ssl': ssl,
    };
  }
}

class DatabaseConfig {
  final String url;
  final int poolSize;

  DatabaseConfig({required this.url, required this.poolSize});

  factory DatabaseConfig.fromMap(Map<String, dynamic> map) {
    return DatabaseConfig(
      url: map['url'] as String,
      poolSize: map['pool_size'] as int? ?? 10,
    );
  }
}

class AppConfig {
  final ServerConfig server;
  final DatabaseConfig database;

  AppConfig({required this.server, required this.database});

  factory AppConfig.fromMap(Map<String, dynamic> map) {
    return AppConfig(
      server: ServerConfig.fromMap(map['server'] as Map<String, dynamic>),
      database: DatabaseConfig.fromMap(map['database'] as Map<String, dynamic>),
    );
  }

  static Future<AppConfig> loadFromFile(String path) async {
    final file = File(path);
    final contents = await file.readAsString();
    final data = loadYaml(contents);
    return AppConfig.fromMap({for (var e in data.entries) e.key: e.value});
  }
}

void main() async {
  final config = await AppConfig.loadFromFile('config.yaml');

  print('Server: ${config.server.host}:${config.server.port}');
  print('Database: ${config.database.url}');
}
```

Benefits:
- Type safety
- Autocomplete in your IDE
- Validation in one place
- Easy testing

## Validation

Always validate config:

```dart
class Config {
  final String host;
  final int port;

  Config({required this.host, required this.port}) {
    _validate();
  }

  void _validate() {
    if (host.isEmpty) {
      throw ConfigException('host cannot be empty');
    }

    if (port < 1 || port > 65535) {
      throw ConfigException('port must be between 1 and 65535');
    }
  }
}

class ConfigException implements Exception {
  final String message;
  ConfigException(this.message);

  @override
  String toString() => 'ConfigException: $message';
}

void main() {
  try {
    final config = Config(host: '', port: 99999);
  } on ConfigException catch (e) {
    stderr.writeln('Configuration error: ${e.message}');
    exit(1);
  }
}
```

## Secrets Management

**Never commit secrets to version control!**

### Option 1: Environment Variables

```dart
class Config {
  final String apiKey;
  final String dbPassword;

  Config._({required this.apiKey, required this.dbPassword});

  static Config load() {
    final apiKey = Platform.environment['API_KEY'];
    if (apiKey == null) {
      throw ConfigException('API_KEY environment variable not set');
    }

    final dbPassword = Platform.environment['DB_PASSWORD'];
    if (dbPassword == null) {
      throw ConfigException('DB_PASSWORD environment variable not set');
    }

    return Config._(apiKey: apiKey, dbPassword: dbPassword);
  }
}
```

Usage:
```bash
$ export API_KEY=your-secret-key
$ export DB_PASSWORD=your-password
$ mytool
```

### Option 2: Separate Secrets File

Config file:
```yaml
# config.yaml (committed to git)
server:
  host: localhost
  port: 8080

secrets_file: ./secrets.yaml  # Not committed!
```

Secrets file:
```yaml
# secrets.yaml (in .gitignore!)
api_key: your-secret-key
db_password: your-password
```

`.gitignore`:
```
secrets.yaml
*.secrets.yaml
```

Loading:
```dart
Future<Map<String, dynamic>> loadWithSecrets(String configPath) async {
  final config = await loadYaml(File(configPath).readAsStringSync());

  // Load secrets file if specified
  final secretsPath = config['secrets_file'];
  if (secretsPath != null) {
    final secretsFile = File(secretsPath);
    if (await secretsFile.exists()) {
      final secrets = await loadYaml(secretsFile.readAsStringSync());
      config['secrets'] = secrets;
    } else {
      stderr.writeln('Warning: Secrets file not found: $secretsPath');
    }
  }

  return config;
}
```

### Option 3: Encrypted Secrets

For extra security, encrypt secrets:

```dart
import 'dart:convert';
import 'package:encrypt/encrypt.dart';

// Encrypt secrets
String encryptSecret(String plaintext, String password) {
  final key = Key.fromUtf8(password.padRight(32).substring(0, 32));
  final iv = IV.fromLength(16);
  final encrypter = Encrypter(AES(key));

  return encrypter.encrypt(plaintext, iv: iv).base64;
}

// Decrypt secrets
String decryptSecret(String encrypted, String password) {
  final key = Key.fromUtf8(password.padRight(32).substring(0, 32));
  final iv = IV.fromLength(16);
  final encrypter = Encrypter(AES(key));

  return encrypter.decrypt64(encrypted, iv: iv);
}
```

**Note**: This is simplified. For production, use proper key derivation (PBKDF2, Argon2) and random IVs.

## Creating Default Config Files

Help users by generating default configs:

```dart
Future<void> initConfig(String appName) async {
  final configPath = getConfigPath(appName, 'config.yaml');
  final configFile = File(configPath);

  if (await configFile.exists()) {
    final overwrite = Confirm(
      prompt: 'Config file already exists. Overwrite?',
      defaultValue: false,
    ).interact();

    if (!overwrite) {
      print('Cancelled.');
      return;
    }
  }

  // Create directory if needed
  await configFile.parent.create(recursive: true);

  // Write default config
  final defaultConfig = '''
# Configuration for $appName
# Edit this file to customize settings

server:
  host: localhost
  port: 8080
  ssl: false

# Database settings
database:
  url: postgresql://localhost/mydb
  pool_size: 10

# Feature flags
features:
  debug: false
  verbose: false
''';

  await configFile.writeAsString(defaultConfig);

  print('✓ Created config file: $configPath');
  print('Edit this file to customize your settings.');
}

void main() async {
  await initConfig('mytool');
}
```

Usage:
```bash
$ mytool init
✓ Created config file: /home/user/.config/mytool/config.yaml
Edit this file to customize your settings.
```

## Watching for Config Changes

For long-running tools, reload config when it changes:

```dart
import 'dart:async';
import 'dart:io';

class ConfigWatcher {
  final String configPath;
  final void Function(Map<String, dynamic>) onConfigChanged;

  StreamSubscription? _subscription;

  ConfigWatcher({
    required this.configPath,
    required this.onConfigChanged,
  });

  void start() {
    final file = File(configPath);

    _subscription = file.watch().listen((event) {
      if (event.type == FileSystemEvent.modify) {
        _reloadConfig();
      }
    });

    print('Watching config file for changes: $configPath');
  }

  Future<void> _reloadConfig() async {
    try {
      final contents = await File(configPath).readAsString();
      final data = loadYaml(contents);
      final config = {for (var e in data.entries) e.key: e.value};

      onConfigChanged(config);
      print('Config reloaded');
    } catch (e) {
      stderr.writeln('Error reloading config: $e');
    }
  }

  void stop() {
    _subscription?.cancel();
  }
}

void main() async {
  final watcher = ConfigWatcher(
    configPath: 'config.yaml',
    onConfigChanged: (config) {
      print('New config: $config');
      // Update app settings here
    },
  );

  watcher.start();

  // Keep running
  await Future.delayed(Duration(hours: 1));
  watcher.stop();
}
```

## Migration: Updating Old Configs

When you change config format, help users migrate:

```dart
Future<Map<String, dynamic>> migrateConfig(Map<String, dynamic> config) async {
  final version = config['version'] as int? ?? 1;

  if (version == 1) {
    // Migrate v1 -> v2
    config = _migrateV1ToV2(config);
  }

  if (version == 2) {
    // Migrate v2 -> v3
    config = _migrateV2ToV3(config);
  }

  return config;
}

Map<String, dynamic> _migrateV1ToV2(Map<String, dynamic> config) {
  print('Migrating config from v1 to v2...');

  // V1 had 'server_host' and 'server_port' as top-level keys
  // V2 nests them under 'server'
  return {
    'version': 2,
    'server': {
      'host': config['server_host'] ?? 'localhost',
      'port': config['server_port'] ?? 8080,
    },
    ...config,
  }..remove('server_host')..remove('server_port');
}

Map<String, dynamic> _migrateV2ToV3(Map<String, dynamic> config) {
  print('Migrating config from v2 to v3...');
  // ... migration logic
  return config;
}
```

## Example: Complete Config System

Let's put it all together:

```dart
// lib/config.dart
import 'dart:io';
import 'package:yaml/yaml.dart';
import 'package:path/path.dart' as path;

class AppConfig {
  final String host;
  final int port;
  final bool debug;
  final String apiKey;

  AppConfig({
    required this.host,
    required this.port,
    required this.debug,
    required this.apiKey,
  });

  static Future<AppConfig> load({
    List<String>? args,
    String? configPath,
  }) async {
    // Defaults
    var host = 'localhost';
    var port = 8080;
    var debug = false;
    String? apiKey;

    // Load from config file
    final configFile = configPath ?? _getDefaultConfigPath();
    if (await File(configFile).exists()) {
      final data = await _loadYamlFile(configFile);
      host = data['host'] ?? host;
      port = data['port'] ?? port;
      debug = data['debug'] ?? debug;
      apiKey = data['api_key'];
    }

    // Override from environment
    host = Platform.environment['APP_HOST'] ?? host;
    port = int.tryParse(Platform.environment['APP_PORT'] ?? '') ?? port;
    debug = Platform.environment['APP_DEBUG'] == 'true' || debug;
    apiKey = Platform.environment['API_KEY'] ?? apiKey;

    // Validate
    if (apiKey == null || apiKey.isEmpty) {
      throw ConfigException('API key not configured. Set API_KEY environment variable or add api_key to config file.');
    }

    if (port < 1 || port > 65535) {
      throw ConfigException('Invalid port: $port');
    }

    return AppConfig(
      host: host,
      port: port,
      debug: debug,
      apiKey: apiKey,
    );
  }

  static String _getDefaultConfigPath() {
    if (Platform.isWindows) {
      return path.join(
        Platform.environment['APPDATA']!,
        'myapp',
        'config.yaml',
      );
    }

    final configHome = Platform.environment['XDG_CONFIG_HOME'] ??
        path.join(Platform.environment['HOME']!, '.config');

    return path.join(configHome, 'myapp', 'config.yaml');
  }

  static Future<Map<String, dynamic>> _loadYamlFile(String path) async {
    final contents = await File(path).readAsString();
    final yaml = loadYaml(contents);
    return {for (var e in yaml.entries) e.key: e.value};
  }
}

class ConfigException implements Exception {
  final String message;
  ConfigException(this.message);

  @override
  String toString() => message;
}
```

## Best Practices

1. **Follow platform conventions** for config file locations
2. **Support multiple config sources** (files, env vars, CLI args)
3. **Use clear precedence** (CLI > env > user config > system config > defaults)
4. **Never commit secrets** — use environment variables or separate files
5. **Validate early** — catch config errors before starting
6. **Provide defaults** — tools should work with zero configuration
7. **Use typed config classes** — avoid raw map access
8. **Generate default configs** — `mytool init` is helpful
9. **Support comments** — YAML or TOML, not JSON
10. **Version your config format** — makes migration easier

## What's Next?

Your tools can now be properly configured! You've learned:

- Config file formats (JSON, YAML, TOML)
- Platform-specific config locations
- Configuration precedence
- Secrets management
- Type-safe config classes
- Validation strategies
- Config file generation

In the next chapter, we tackle the inevitable: **errors**. Because things go wrong, and when they do, we need to handle them gracefully.

Time to fail with dignity!

---

**[← Previous: Progress and Spinners](06-progress-and-spinners.md)** | **[Next: Error Handling →](08-error-handling.md)**
