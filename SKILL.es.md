description: Traduce sin complicaciones código entre lenguajes de programación mientras preserva la lógica y la semántica
version: 1.0.0
author: OpenClaw Team
tags: [polyglot, transpilation, code-conversion]
maintainer: dev@openclaw.io
---

# Propósito

La habilidad Polyglot Transmogrifier permite a los desarrolladores convertir código fuente de un lenguaje de programación a otro con alta fidelidad. Se dirige a escenarios del mundo real como:

- **Legacy modernization**: Convertir lógica de negocio COBOL a Java o C# para facilitar el mantenimiento.
- **Cross-platform adaptation**: Traducir código Swift de iOS a Kotlin para Android mientras se preservan los algoritmos.
- **Educational tool**: Mostrar a los estudiantes cómo se expresa el mismo algoritmo en diferentes paradigmas (por ejemplo, Haskell funcional vs. Java orientado a objetos).
- **Prototype porting**: Mover rápidamente un script de procesamiento de datos de Python a Go para mejorar el rendimiento.
- **API compatibility**: Convertir interfaces de TypeScript a anotaciones de tipo de Python para microservicios poliglotas.

NO es un compilador completo; se centra en preservar la semántica del código de aplicación típico (flujo de control, estructuras de datos, patrones de E/S) y reconoce que algunos idioms son específicos del lenguaje y pueden requerir revisión manual.

# Alcance

## Comandos

- `polyglot-transmogrifier convert <source-lang> <target-lang> [options] <input-files>`
  - Traduce uno o más archivos fuente del lenguaje origen al lenguaje destino.
  - Opciones:
    - `--output <dir>`: Escribir los archivos traducidos en el directorio (predeterminado: directorio de trabajo actual).
    - `--preserve-comments`: Intentar conservar los comentarios originales (funciona para lenguajes compatibles).
    - `--style <auto|pep8|google|...>`: Aplicar convenciones de estilo de código para el lenguaje destino.
    - `--dry-run`: Mostrar el plan de traducción sin escribir archivos.
    - `--diff`: Salida de diff unificado en lugar de archivos.
    - `--map <spec>`: Proporcionar mapeos de tipos personalizados (ej., `--map "std::vector~List"`).
    - `--exclude <pattern>`: Excluir archivos que coincidan con el patrón glob.
    - `--verbose`: Mostrar pasos de conversión detallados.
    - `--no-verify`: Omitir la verificación posterior a la traducción.

- `polyglot-transmogrifier detect <file>`
  - Detecta el lenguaje de programación de un archivo dado (usado internamente pero puede invocarse).
  - Salida: identificador de lenguaje (ej., `python`, `javascript`, `rust`).

- `polyglot-transmogrifier list-languages`
  - Lista todos los lenguajes origen y destino compatibles con soporte de versión.

- `polyglot-transmogrifier validate <file>`
  - Valida un archivo traducido contra la sintaxis del lenguaje destino (usando servidor de lenguaje o compilador si está disponible).

- `polyglot-transmogrifier --help`
  - Muestra la ayuda.

## Lenguajes Compatibles (a partir de v1.0.0)
- Origen → Destino: Python, JavaScript/TypeScript, Java, C#, C++, Go, Rust, Ruby, Swift, Kotlin, PHP, Haskell, Lua, Bash.

## Dependencias
- **Runtime**: Python 3.9+ (usado por el motor Polyglot subyacente).
- **Herramientas externas** (opcionales pero recomendadas para verificación):
  - `node` (para validación JavaScript/TypeScript)
  - `python3` (para validación Python)
  - `go` (para validación Go)
  - `rustc` (para validación Rust)
  - `javac` (para validación Java)
  - `dotnet` (para validación C#)
- **Configuración**: archivo JSON `.polyglotrc` en la raíz del proyecto para personalizar mapeos, convenciones de nomenclatura, etc.

## Variables de Entorno
- `POLYGLOT_CACHE_DIR`: Sobrescribir la caché predeterminada para AST analizados (predeterminado: `~/.cache/polyglot`).
- `POLYGLOT_DISABLE_TELEMETRY`: Establecer a `1` para deshabilitar el reporte de uso.
- `POLYGLOT_VERBOSE`: Establecer a `1` para equivalente de `--verbose`.

# Proceso de Trabajo

1. **Recopilación de Entrada**: Recopilar archivos mediante argumentos CLI o leer de STDIN si se usa `-`.
2. **Detección de Lenguaje**: Para cada archivo, determinar el lenguaje origen usando heurísticas (extensión, shebang, detección AST). Sobrescribir con `--from` si es necesario.
3. **Análisis**: Analizar el archivo fuente en una representación intermedia (IR) - el Polyglot IR (PIR). Esto incluye AST, tabla de símbolos e inferencia de tipos donde sea posible.
4. **Análisis Semántico**: Resolver tipos, manejar importaciones y construir referencias cruzadas de símbolos. Este paso puede requerir stubs para bibliotecas estándar.
5. **Transformación**: Aplicar transformaciones específicas del lenguaje destino. Esto incluye:
   - Mapeo de tipos primitivos (ej., Python `int` → Go `int64`).
   - Conversión de estructuras de control (ej., Python `for item in list` → Rust `for item in list.iter()`).
   - Manejo de paradigmas de manejo de errores (excepciones vs. tipos de resultado).
   - Ajuste de gestión de memoria (GC vs. manual/RAII).
6. **Generación de Código**: Emitir código fuente del lenguaje destino con formato apropiado. Incorporar mapeos personalizados de la configuración.
7. **Post-Procesamiento**: Aplicar estilo de código (si se solicitó), insertar boilerplate necesario (ej., declaraciones de paquete, importaciones).
8. **Verificación** (a menos que `--no-verify`): Ejecutar verificación de sintaxis usando la cadena de herramientas del lenguaje destino. Fallar si se encuentran errores de sintaxis.
9. **Salida**: Escribir archivos en el directorio de destino, preservando rutas relativas. Si `--diff`, salida diff unificado a stdout.
10. **Reporte**: Imprimir resumen de conversiones, cualquier advertencia (ej., características no compatibles que fueron aproximadas) y estado de verificación.

# Reglas de Oro

1. **Copia de Seguridad Primero**: Siempre confirmar o hacer copia de seguridad de los archivos fuente antes de conversión masiva. Usar control de versiones.
2. **Revisar Mapeos**: Los mapeos de tipos personalizados deben ser validados; mapeos incorrectos pueden introducir errores sutiles.
3. **Verificar la Salida**: Incluso si la verificación pasa, revisar manualmente secciones críticas (ej., concurrencia, E/S, manejo de errores) porque la semántica puede divergir.
4. **Migración Incremental**: Para proyectos grandes, convertir un módulo a la vez e integrar antes de proceder.
5. **Preservar Pruebas**: Convertir suites de pruebas junto con el código para validar comportamiento; no omitir pruebas.
6. **Manejar Código Específico de Plataforma**: Reconocer que llamadas al sistema, rutas de archivo e interacciones de entorno pueden no traducirse directamente; estas requieren adaptación manual.
7. **Usar Versionado Explícito**: Si el código fuente usa características de una versión específica del lenguaje, especificar con los flags `--source-version` y `--target-version`.
8. **Evitar One-Liners**: Expresiones complejas pueden dividirse en múltiples líneas para legibilidad en el lenguaje destino; no forzar one-liners.
9. **Reservar Ediciones Manuales**: Después de la conversión, marcar secciones editadas manualmente con `// POLYGLOT: MANUAL REVIEW` o equivalente para evitar sobrescribir durante la retraducción.
10. **Mantenerse Dentro de Constructos Compatibles**: Evitar usar características oscuras o obsoletas; el Polyglot IR tiene cobertura limitada para metaprogramación muy avanzada o reflexión.

# Ejemplos

## Convertir un módulo de utilidades de Python a Go

Comando de entrada:
```bash
polyglot-transmogrifier convert python go utils.py --output ./go_utils --style=go fmt --preserve-comments
```

`utils.py` contiene:
```python
def format_name(first: str, last: str) -> str:
    """Return full name, title-cased."""
    return f"{first.title()} {last.title()}"
```

Salida `./go_utils/utils.go`:
```go
package utils

// format_name returns full name, title-cased.
func FormatName(first, last string) string {
    return strings.Title(first) + " " + strings.Title(last)
}
```

Nota: Se agregó automáticamente la importación del paquete `strings`.

## Detectar lenguaje de un archivo

```bash
polyglot-transmogrifier detect script.sh
# Salida: bash
```

## Listar lenguajes compatibles

```bash
polyglot-transmogrifier list-languages
# Salida incluye:
#   python: 3.6-3.11
#   go: 1.16-1.22
#   rust: 2018+, 2021
```

## Convertir una clase de C++ a Rust con mapeo personalizado

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

Comando:
```bash
polyglot-transmogrifier convert cpp rust vector_math.h \
  --map "Vec3~Vec3f" \
  --map "std::vector~Vec" \
  --output ./rust_math
```

Generado `rust_math/vector_math.rs`:
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

## Dry-run para ver qué se convertirá

```bash
polyglot-transmogrifier convert typescript python src/ --dry-run
# Salida:
# [DRY RUN] Se convertirían 12 archivos de TypeScript a Python
#   src/utils.ts -> src/utils.py
#   src/api.ts -> src/api.py
# ...
```

# Rollback

- **Rollback inmediato de un solo archivo**: Si no ha modificado el original después de la conversión, simplemente restaure desde control de versiones (ej., `git checkout -- utils.py`). Si sobrescribió el original (usando `--in-place`), Polyglot no admite sobrescribir por defecto; la salida siempre va a archivos/directorio separados.
- **Rollback masivo**: Si convirtió un directorio completo y desea revertir:
  - Eliminar el directorio traducido y restaurar desde copia de seguridad:
    ```bash
    rm -rf go_utils/
    git checkout -- utils.py
    ```
  - Alternativamente, use el log de rollback de Polyglot: `polyglot-transmogrifier` escribe un `.polyglot_manifest.json` en el directorio de salida listando archivos fuente y sus rutas originales. Puede alimentar esto para restaurar:
    ```bash
    cat go_utils/.polyglot_manifest.json | jq -r '.mappings[] | "\(.source) -> \(.target)"' | while read src tgt; do
      mv "$tgt" "$src"
    done
    ```
- **Reconvertir con diferentes opciones**: Vuelva a ejecutar la conversión con nuevos flags; la salida antigua debe eliminarse primero para evitar mezclar.
- **Deshacer en control de versiones**: La mejor práctica es confirmar la fuente antes de la conversión, luego puede revertir el commit:
    ```bash
    git revert <conversion-commit>
    ```

# Solución de Problemas

- **Errores de "Característica no compatible"**: Algunas construcciones de lenguaje (ej., decoradores de Python en funciones locales, metaprogramación de plantillas C++, proxies JavaScript) pueden no traducirse completamente. La herramienta emitirá una advertencia y generará código aproximado con marcadores `// TODO: POLYGLOT`. Revise estos manualmente.
- **Importaciones faltantes**: El código generado puede carecer de importaciones requeridas si la herramienta no puede inferir dependencias. Use `--preserve-comments` para ver declaraciones de importación originales, luego agregue manualmente.
- **Fallas de inferencia de tipos**: Lenguajes dinámicos (Python, Ruby, JS) pueden producir `any` u `Object` en el destino. Use `--map` para anular tipos específicos.
- **Verificación fallida**: Los errores de sintaxis a menudo surgen de un desajuste de versión del lenguaje destino. Use `--target-version` para especificar.
- **Regresiones de rendimiento**: El código traducido puede ser menos performante, especialmente en cuanto a asignación de memoria o concurrencia. Perfile y optimice manualmente.
- **Errores de resolución de símbolos**: Si su proyecto usa un sistema de construcción complejo, la herramienta puede no resolver importaciones correctamente. Proporcione archivos stub o use `--include-path` para ayudar.
- **Colisiones de nombres**: Los identificadores traducidos pueden chocar con palabras clave del lenguaje destino. La herramienta normalmente agrega guiones bajos, pero verifique con `--verbose`.
- **Problemas de codificación**: Asegúrese de que todos los archivos fuente sean UTF-8; los no-UTF-8 pueden causar errores de análisis.

# Configuración

Polyglot Transmogrifier lee configuración opcional de `.polyglotrc` en la raíz del proyecto (o cualquier directorio padre). Ejemplo:

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

Esto permite personalizar el comportamiento por proyecto.

# Pasos de Verificación

1. **Comprobación de Sintaxis**: Ejecute `polyglot-transmogrifier validate` en cada archivo de salida, o confíe en la verificación automática después de la conversión.
2. **Ejecutar Pruebas**: Ejecute la suite de pruebas del lenguaje destino (ej., `go test ./...`, `pytest`) para capturar diferencias semánticas.
3. **Análisis Estático**: Use linters (ej., `eslint`, `flake8`, `cargo clippy`) para detectar problemas de estilo.
4. **Ejecución de Ejemplo**: Ejecute algunas funciones representativas con entradas conocidas para verificar que la salida coincida con el comportamiento original.
5. **Revisión de Diferencias**: Use `--diff` para revisar cambios antes de confirmar; busque marcadores `// POLYGLOT: MANUAL REVIEW`.

# Características Avanzadas

- **Conversión parcial**: Convierta solo funciones seleccionadas vía flag `--functions` (lista separada por comas).
- **Procesamiento por lotes**: Proporcione un archivo manifiesto (JSON) con pares de archivos explícitos para conversión uno a uno.
- **Modo interactivo**: `polyglot-transmogrifier interactive` lanza una TUI para ajustar mapeos sobre la marcha.
- **API Web**: Ejecute como microservicio: `polyglot-transmogrifier serve --port 8080` para aceptar solicitudes JSON (útil para integración IDE).

# Notas

- Polyglot Transmogrifier no es un compilador completo; casos límite como macros, reflexión o genéricos avanzados pueden necesitar intervención manual.
- Legal: Asegúrese de tener derechos para convertir el código fuente; respete las licencias.
- Rendimiento: La traducción consume mucha CPU para bases de código grandes; considere usar caché con `--cache-dir`.