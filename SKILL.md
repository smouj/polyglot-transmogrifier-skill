name: Polyglot Transmogrifier
description: Seamlessly translates code between programming languages while preserving logic and semantics
version: 1.0.0
author: OpenClaw Team
tags: [polyglot, transpilation, code-conversion]
maintainer: dev@openclaw.io
---

# Purpose

The Polyglot Transmogrifier skill enables developers to convert source code from one programming language to another with high fidelity. It targets real-world scenarios such as:

- **Legacy modernization**: Converting COBOL business logic to Java or C# for maintainability.
- **Cross-platform adaptation**: Translating Swift iOS code to Kotlin for Android while preserving algorithms.
- **Educational tool**: Showing students how the same algorithm is expressed in different paradigms (e.g., functional Haskell vs. object-oriented Java).
- **Prototype porting**: Quickly moving a Python data processing script to Go for performance.
- **API compatibility**: Converting TypeScript interfaces to Python type hints for polyglot microservices.

It is NOT a full compiler; it focuses on preserving semantics for typical application code (control flow, data structures, I/O patterns) and recognizes that some idioms are language-specific and may require manual review.

# Scope

## Commands

- `polyglot-transmogrifier convert <source-lang> <target-lang> [options] <input-files>`
  - Translates one or more source files from source language to target language.
  - Options:
    - `--output <dir>`: Write translated files to directory (default: current working directory).
    - `--preserve-comments`: Attempt to retain original comments (works for supported languages).
    - `--style <auto|pep8|google|...>`: Apply code style conventions for target language.
    - `--dry-run`: Show translation plan without writing files.
    - `--diff`: Output unified diff instead of files.
    - `--map <spec>`: Provide custom type mappings (e.g., `--map "std::vector~List"`).
    - `--exclude <pattern>`: Exclude files matching glob pattern.
    - `--verbose`: Show detailed conversion steps.
    - `--no-verify`: Skip post-translation verification.

- `polyglot-transmogrifier detect <file>`
  - Detect the programming language of a given file (used internally but can be invoked).
  - Outputs language identifier (e.g., `python`, `javascript`, `rust`).

- `polyglot-transmogrifier list-languages`
  - List all supported source and target languages with version support.

- `polyglot-transmogrifier validate <file>`
  - Validate a translated file against the target language's syntax (using language server or compiler if available).

- `polyglot-transmogrifier --help`
  - Show help.

## Supported Languages (as of v1.0.0)
- Source → Target: Python, JavaScript/TypeScript, Java, C#, C++, Go, Rust, Ruby, Swift, Kotlin, PHP, Haskell, Lua, Bash.

## Dependencies
- **Runtime**: Python 3.9+ (used by the underlying Polyglot engine).
- **External tools** (optional but recommended for verification):
  - `node` (for JavaScript/TypeScript validation)
  - `python3` (for Python validation)
  - `go` (for Go validation)
  - `rustc` (for Rust validation)
  - `javac` (for Java validation)
  - `dotnet` (for C# validation)
- **Configuration**: `.polyglotrc` JSON file in project root to customize mappings, naming conventions, etc.

## Environment Variables
- `POLYGLOT_CACHE_DIR`: Override default cache for parsed ASTs (default: `~/.cache/polyglot`).
- `POLYGLOT_DISABLE_TELEMETRY`: Set to `1` to disable usage reporting.
- `POLYGLOT_VERBOSE`: Set to `1` for equivalent of `--verbose`.

# Work Process

1. **Input Collection**: Gather files via CLI arguments or read from STDIN if `-` is used.
2. **Language Detection**: For each file, determine source language using heuristics (extension, shebang, AST sniff). Override with explicit `--from` if needed.
3. **Parsing**: Parse source file into an intermediate representation (IR) - the Polyglot IR (PIR). This includes AST, symbol table, and type inference where possible.
4. **Semantic Analysis**: Resolve types, handle imports, and build symbol cross-references. This step may require stubs for standard libraries.
5. **Transformation**: Apply target-language-specific transformations. This includes:
   - Mapping primitive types (e.g., Python `int` → Go `int64`).
   - Converting control structures (e.g., Python `for item in list` → Rust `for item in list.iter()`).
   - Handling error handling paradigms (exceptions vs. result types).
   - Adjusting memory management (GC vs. manual/RAII).
6. **Code Generation**: Emit target source code with appropriate formatting. Incorporate custom mappings from config.
7. **Post-Processing**: Apply code style (if requested), insert necessary boilerplate (e.g., package declarations, imports).
8. **Verification** (unless `--no-verify`): Run syntax check using target language toolchain. Fail if syntax errors found.
9. **Output**: Write files to destination directory, preserving relative paths. If `--diff`, output unified diff to stdout.
10. **Report**: Print summary of conversions, any warnings (e.g., unsupported features that were approximated), and verification status.

# Golden Rules

1. **Backup First**: Always commit or backup source files before bulk conversion. Use version control.
2. **Review Mappings**: Custom type mappings must be validated; incorrect mappings can introduce subtle bugs.
3. **Verify Output**: Even if verification passes, manually review critical sections (e.g., concurrency, I/O, error handling) because semantics may diverge.
4. **Incremental Migration**: For large projects, convert one module at a time and integrate before proceeding.
5. **Preserve Tests**: Convert test suites along with code to validate behavior; do not skip tests.
6. **Handle Platform-Specific Code**: Recognize that system calls, file paths, and environment interactions may not translate directly; these require manual adaptation.
7. **Use Explicit Versioning**: If source uses language features from a specific version, specify with `--source-version` and `--target-version` flags.
8. **Avoid One-Liners**: Complex expressions may be split across lines for readability in target language; don't force one-liners.
9. **Reserve Manual Edits**: After conversion, mark manually edited sections with `// POLYGLOT: MANUAL REVIEW` or equivalent to avoid overwriting during re-translation.
10. **Stay Within Supported Constructs**: Avoid using obscure or deprecated features; the Polyglot IR has limited coverage for very advanced metaprogramming or reflection.

# Examples

## Convert a Python utility module to Go

Input command:
```bash
polyglot-transmogrifier convert python go utils.py --output ./go_utils --style=go fmt --preserve-comments
```

`utils.py` contains:
```python
def format_name(first: str, last: str) -> str:
    """Return full name, title-cased."""
    return f"{first.title()} {last.title()}"
```

Output `./go_utils/utils.go`:
```go
package utils

// format_name returns full name, title-cased.
func FormatName(first, last string) string {
    return strings.Title(first) + " " + strings.Title(last)
}
```

Note: Automatic import of `strings` package was added.

## Detect language of a file

```bash
polyglot-transmogrifier detect script.sh
# Output: bash
```

## List supported languages

```bash
polyglot-transmogrifier list-languages
# Output includes:
#   python: 3.6-3.11
#   go: 1.16-1.22
#   rust: 2018+, 2021
```

## Convert a C++ class to Rust with custom mapping

`vector_math.h`:
```cpp
#include <vector>
#include <cmath>

class Vec3 {
public:
    double x, y, z;
    Vec3(double x, double y, double z) : x(x), y(y), z(z) {}
    double length() const { return std::sqrt(x*x + y*y + z*z); }
};
```

Command:
```bash
polyglot-transmogrifier convert cpp rust vector_math.h \
  --map "Vec3~Vec3f" \
  --map "std::vector~Vec" \
  --output ./rust_math
```

Generated `rust_math/vector_math.rs`:
```rust
/// Vec3 represents a 3D vector with f64 precision.
pub struct Vec3 {
    pub x: f64,
    pub y: f64,
    pub z: f64,
}

impl Vec3 {
    pub fn new(x: f64, y: f64, z: f64) -> Self {
        Self { x, y, z }
    }

    pub fn length(&self) -> f64 {
        (self.x * self.x + self.y * self.y + self.z * self.z).sqrt()
    }
}
```

## Dry-run to see what will be converted

```bash
polyglot-transmogrifier convert typescript python src/ --dry-run
# Output:
# [DRY RUN] Would convert 12 files from TypeScript to Python
#   src/utils.ts -> src/utils.py
#   src/api.ts -> src/api.py
# ...
```

# Rollback

- **Immediate rollback of single file**: If you haven't modified the original after conversion, simply restore from version control (e.g., `git checkout -- utils.py`). If you wrote over original (using `--in-place`), Polyglot does not support overwriting by default; output always to separate files/directory.
- **Bulk rollback**: If you converted a whole directory and want to revert:
  - Remove translated directory and restore from backup:
    ```bash
    rm -rf go_utils/
    git checkout -- utils.py
    ```
  - Alternatively, use the Polyglot rollback log: `polyglot-transmogrifier` writes a `.polyglot_manifest.json` in the output directory listing source files and their original paths. You can feed this back to restore:
    ```bash
    cat go_utils/.polyglot_manifest.json | jq -r '.mappings[] | "\(.source) -> \(.target)"' | while read src tgt; do
      mv "$tgt" "$src"
    done
    ```
- **Re-convert with different options**: Re-run conversion with new flags; old output should be deleted first to avoid mixing.
- **Undo in version control**: Best practice is to commit source before conversion, then you can revert commit:
    ```bash
    git revert <conversion-commit>
    ```

# Troubleshooting

- **"Unsupported feature" errors**: Some language constructs (e.g., Python decorators on local functions, C++ template metaprogramming, JavaScript proxies) may not be fully translated. The tool will emit a warning and generate approximate code with `// TODO: POLYGLOT` markers. Review these manually.
- **Missing imports**: Generated code may lack required imports if the tool cannot infer dependencies. Use `--preserve-comments` to see original import statements, then manually add.
- **Type inference failures**: Dynamic languages (Python, Ruby, JS) may produce `any` or `Object` in target. Use `--map` to override specific types.
- **Verification fails**: Syntax errors often arise from target language version mismatch. Use `--target-version` to specify.
- **Performance regressions**: Translated code may be less performant, especially regarding memory allocation or concurrency. Profile and optimize manually.
- **Symbol resolution errors**: If your project uses a complex build system, the tool may not resolve imports correctly. Provide stub files or use `--include-path` to help.
- **Name collisions**: Translated identifiers may clash with target language keywords. The tool typically appends underscores, but verify with `--verbose`.
- **Encoding issues**: Ensure all source files are UTF-8; non-UTF-8 may cause parse errors.

# Configuration

Polyglot Transmogrifier reads optional configuration from `.polyglotrc` in the project root (or any parent directory). Example:

```json
{
  "default_style": "pep8",
  "type_mappings": {
    "std::string": "String",
    "std::vector": "ArrayList"
  },
  "exclude": ["**/test_*", "**/*_test.go"],
  "naming": {
    "function": "camelCase",
    "class": "PascalCase"
  },
  "imports": {
    "auto_add": true,
    "preferred": "absolute"
  }
}
```

This allows customizing behavior per project.

# Verification Steps

1. **Syntax Check**: Run `polyglot-transmogrifier validate` on each output file, or rely on automatic verification after conversion.
2. **Run Tests**: Execute the target language's test suite (e.g., `go test ./...`, `pytest`) to catch semantic differences.
3. **Static Analysis**: Use linters (e.g., `eslint`, `flake8`, `cargo clippy`) to spot idiomatic issues.
4. **Sample Execution**: Run a few representative functions with known inputs to check output matches original behavior.
5. **Diff Review**: Use `--diff` to review changes before committing; look for `// POLYGLOT: MANUAL REVIEW` markers.

# Advanced Features

- **Partial conversion**: Convert only selected functions via `--functions` flag (comma-separated).
- **Batch processing**: Provide a manifest file (JSON) with explicit file pairs for one-to-one conversion.
- **Interactive mode**: `polyglot-transmogrifier interactive` launches a TUI to adjust mappings on the fly.
- **Web API**: Run as a microservice: `polyglot-transmogrifier serve --port 8080` to accept JSON requests (useful for IDE integration).

# Notes

- Polyglot Transmogrifier is not a full compiler; corner cases like macros, reflection, or advanced generics may need manual intervention.
- Legal: Ensure you have rights to convert the source code; respect licensing.
- Performance: Translation is CPU-intensive for large codebases; consider caching with `--cache-dir`.