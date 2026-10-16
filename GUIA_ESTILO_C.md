# Guía de estilo y estándar de ingeniería para C

**Versión:** 1.0.0
**Ámbito principal:** aplicaciones, bibliotecas y herramientas nativas para Windows de escritorio
**Lenguaje:** C17
**Arquitectura principal:** x86-64

Esta guía define cómo escribir, revisar y mantener código C del proyecto. Su objetivo no es imponer
preferencias arbitrarias, sino reducir errores, hacer explícitos los contratos y mantener el código
legible durante años.

Las palabras **DEBE**, **NO DEBE**, **RECOMENDADO** y **PERMITIDO** tienen significado normativo.
Cuando una regla no pueda aplicarse, la excepción debe quedar localizada y justificada.

> Esta guía no es una guía de C++. Un proyecto C++ debe adoptar un estándar específico para C++.

---

## 1. Principios

El código debe priorizar, en este orden:

1. Corrección y comportamiento definido.
2. Claridad y facilidad de revisión.
3. Contratos explícitos de ownership, lifetime y errores.
4. Simplicidad del diseño.
5. Mantenibilidad.
6. Rendimiento cuando sea relevante y esté demostrado mediante medición.

El código ingenioso que oculta precondiciones, ownership o control de flujo es peor que una versión
algo más larga pero evidente.

La legibilidad, claridad, corrección y mantenibilidad tienen prioridad sobre la microoptimización
manual, salvo que exista un requisito explícito de rendimiento o una medición que demuestre que una
optimización es necesaria.

Se debe escribir primero la versión más clara y obviamente correcta que tenga una complejidad
algorítmica apropiada. Se permite al compilador realizar optimizaciones mecánicas como propagación de
constantes, eliminación de trabajo redundante, vectorización o reconocimiento de patrones. No se debe
sacrificar legibilidad intentando anticipar manualmente cada transformación posible.

Por ejemplo, para un `uint32_t n`, cuya suma completa cabe en `uint64_t`, un programador puede
escribir con claridad:

```c
uint64_t sum = 0;

for (uint64_t i = 0; i <= n; ++i) {
  sum += i;
}
```

Determinados compiladores, niveles de optimización y condiciones pueden reconocer un patrón
equivalente y generar una forma cerrada u otra implementación más eficiente. No se debe afirmar ni
suponer que esa transformación concreta ocurrirá siempre.

Esta regla tampoco autoriza algoritmos evidentemente inadecuados. No se escribe deliberadamente una
solución O(n²) cuando existe una solución O(n) igual de clara, ni se confía en que el compilador
transforme arbitrariamente un algoritmo ineficiente. Entre dos implementaciones razonablemente
eficientes se prefiere la más clara hasta que perfiles o benchmarks justifiquen otra.

El proyecto debe evitar:

- comportamiento indefinido;
- conversiones silenciosas peligrosas;
- dependencias implícitas;
- estado mutable global sin propietario;
- optimizaciones no medidas;
- abstracciones de portabilidad que no correspondan a plataformas soportadas.

---

## 2. Plataformas soportadas

### 2.1 Windows como plataforma principal

La plataforma principal es **Windows 11 de escritorio, x86-64**. El código debe usar interfaces
públicas y documentadas para conservar una ruta razonable de compatibilidad con futuras versiones de
Windows de escritorio.

No se debe depender de:

- APIs internas o no documentadas;
- comportamiento accidental de una versión concreta del sistema;
- rutas fijas del SDK, Visual Studio o Windows;
- tamaños de estructuras del sistema no garantizados por la documentación;
- `MAX_PATH` como límite universal del proyecto.

El proyecto debe fijar una versión mínima de Windows mediante sus manifiestos, macros de destino y
documentación de compilación.

### 2.2 Linux es opt-in

Linux no se considera soportado por accidente. Un proyecto o módulo solo soporta Linux cuando lo
declara expresamente.

Cuando exista soporte Linux:

- el código específico debe aislarse en una capa de plataforma;
- los módulos comunes no deben incluir cabeceras Win32 ni POSIX directamente;
- las diferencias no deben dispersarse mediante docenas de `#ifdef` dentro de la lógica;
- la API común debe representar capacidades reales compartidas, no fingir equivalencias falsas.

Organización recomendada:

```text
src/
  core/
  platform/
    windows/
    linux/       # Solo si el proyecto declara soporte Linux.
```

Una compilación auxiliar en Linux para analizadores o sanitizers puede existir sin convertir Linux
en plataforma de producto soportada, siempre que se documente claramente como build de análisis.

---

## 3. Lenguaje y toolchain

### 3.1 Estándar de C

Todo el código C debe compilar como **C17**.

```text
clang -std=c17
cl /std:c17
clang-cl /std:c17
```

Las extensiones del compilador se permiten únicamente cuando:

- resuelven una necesidad real;
- están aisladas detrás de una macro o helper;
- tienen una alternativa o error claro para toolchains incompatibles;
- no sustituyen una solución estándar igual de clara.

Los VLAs están prohibidos. Deben usarse tamaños estáticos, heap, arena o una estrategia explícita.

### 3.2 Toolchains de desarrollo

Para desarrollo, prototipos, herramientas internas y builds rápidos se permite:

```text
clang + lld-link
```

También se permite `clang-cl + lld-link` cuando convenga usar flags y ABI de estilo MSVC.

### 3.3 Toolchains de entrega final en Windows

La build oficial final ordinaria de una aplicación, DLL o producto completo debe usar una de estas
dos cadenas:

```text
cl + link
clang-cl + link
```

Ambas son válidas. El proyecto debe escoger una como toolchain oficial y fijar sus versiones. No se
debe alternar silenciosamente entre ambas para artefactos de release.

`lld-link` continúa siendo válido para desarrollo, CI rápida y herramientas no distribuidas, pero no
es el enlazador de la build final oficial salvo excepción documentada del proyecto.

Una variante adicional para CPUs Intel debe usar `icx + link` conforme a §3.4. Esta variante no
convierte `icx` en la toolchain oficial ordinaria ni sustituye el artefacto normal.

### 3.4 Variante Intel oneAPI (`icx`)

Para todo proyecto C compilable que adopte esta guía, el sistema o script oficial de build DEBE
contemplar una variante Intel cuando la build se ejecute sobre una CPU Intel x86-64 compatible y
`icx` esté disponible. Esta política se aplica con independencia de que se use CMake, Make, NMake,
Ninja, MSBuild, `.ps1`, `.bat`, `.cmd` u otro sistema razonable.

El sistema de build debe distinguir dos condiciones:

1. El fabricante de la CPU de la máquina que realiza la build.
2. La disponibilidad efectiva de `icx` en el entorno configurado.

Si la CPU no es Intel, la build genera únicamente el artefacto normal mediante la toolchain oficial
del proyecto. No es obligatorio buscar ni ejecutar `icx`.

Si la CPU es Intel, la build debe intentar ofrecer dos variantes:

```text
program.exe
program_intel.exe
```

La primera usa la toolchain oficial ordinaria, por ejemplo:

```text
clang-cl + link
cl + link
```

La segunda usa:

```text
icx + link
```

El sufijo `_intel` se coloca inmediatamente antes de la extensión:

```text
renderer.exe
renderer_intel.exe

compiler.exe
compiler_intel.exe

mylibrary.dll
mylibrary_intel.dll
```

La misma convención puede aplicarse a otros artefactos cuando tenga sentido.

Si la CPU es Intel pero `icx` no está instalado o no está disponible en el entorno:

- la build normal DEBE continuar funcionando;
- la ausencia de la variante Intel NO DEBE convertirse por sí sola en un error fatal;
- el sistema de build DEBE emitir un mensaje claro indicando que no se generó la variante `_intel`
  porque `icx` no está disponible;
- se puede documentar cómo instalar o habilitar Intel oneAPI.

Una build rota con `icx` después de que este se haya detectado debe informar el fallo de la variante
Intel sin destruir ni invalidar un artefacto normal ya generado correctamente. CI puede elevar ese
fallo a error en jobs que declaren expresamente obligatoria la validación de `icx`.

La detección de CPU y compilador debe estar centralizada y ser revisable. Se permite un override
explícito y documentado para CI o cross-compilation, pero no una detección basada en estado oculto.

La existencia de `program_intel.exe` no demuestra que sea más rápido. `icx` no garantiza por sí solo
una mejora; el resultado depende del programa, los flags, el runtime y el hardware. La variante se
genera para ofrecer de forma consistente una build alternativa con la toolchain Intel.

Para módulos o productos extremadamente sensibles al rendimiento, los benchmarks deben comparar al
menos:

- la toolchain oficial;
- la variante `icx`;
- cualquier otra variante relevante para el despliegue.

En esos componentes se permite compilar únicamente los translation units críticos con `icx` y
enlazar sus objetos con el resto de la aplicación, siempre que la compatibilidad binaria, el runtime,
los tests, los warnings y el hardening se verifiquen en CI.

No se deben usar opciones equivalentes a `-march=native`, `-xHost` o targets ligados accidentalmente
a la CPU exacta del desarrollador cuando el binario vaya a distribuirse a otras CPUs Intel. Para una
distribución general debe existir una baseline razonable, dispatch por ISA o variantes explícitas.

`icx` complementa la toolchain oficial y no sustituye el principio de **medir antes de optimizar**.
La build normal sigue siendo el artefacto de referencia salvo que el proyecto documente otra política
de distribución respaldada por pruebas.

### 3.5 Builds mínimas

Todo proyecto no trivial debe ofrecer como mínimo:

- `debug`: sin optimización o con optimización mínima, símbolos completos y assertions;
- `release`: optimización estable, símbolos conservados y hardening;
- `analyze`: análisis estático y clang-tidy;
- `sanitized`: ASan/UBSan donde el toolchain lo soporte correctamente;
- `test`: ejecución de tests con los mismos contratos que producción.

Los flags exactos pertenecen al sistema de build. No se deben copiar listas sin comprobar que el
compilador y el runtime seleccionados las soportan.

### 3.6 Warnings

El código propio debe compilar sin warnings. CI debe tratar como error los warnings del proyecto, pero
no convertir indiscriminadamente en error los diagnostics de cabeceras de terceros o del SDK.

Con Clang se recomienda como base:

```text
-Wall -Wextra -Wpedantic -Werror
-Wshadow -Wconversion -Wsign-conversion
-Wformat=2 -Wundef -Wstrict-prototypes
-Wmissing-prototypes -Wmissing-declarations -Wvla
```

Con `cl` o `clang-cl` se recomienda como base:

```text
/W4 /WX
```

No se debe usar `/Wall` como sustituto automático de una política de warnings revisada: incluye
avisos informativos y de cabeceras que suelen requerir una selección explícita.

### 3.7 Sistema de build

El sistema de build debe ser comprensible sin depender de magia oculta. Para proyectos pequeños se
recomiendan scripts `.ps1` o `.bat`; para proyectos mayores se permiten MSBuild, Ninja, `nmake`, Make
o CMake cuando la complejidad real lo justifique.

No se prohíbe una herramienta por identidad: se prohíben configuraciones difíciles de depurar,
generación innecesaria y dependencia de estado no documentado. Toda build oficial debe poder
reproducirse desde una terminal configurada y debe fijar toolchain, SDK y configuración.

### 3.8 Codificación del código fuente

Los ficheros fuente, cabeceras, scripts y ensamblador del proyecto deben almacenarse en UTF-8. La
build oficial DEBE fijar explícitamente la codificación de entrada para no depender de la code page
activa de la máquina.

Con `cl` y `clang-cl`, cuando la versión fijada lo soporte, se usa:

```text
/utf-8
```

Con otras toolchains se usa la opción equivalente verificada para esa versión. La codificación del
fuente y la representación de texto en runtime son contratos distintos; esta regla no modifica la
política UTF-8/UTF-16 de §14.1.

---

## 4. Formato

`.clang-format` es la autoridad para el formato de C. Si la prosa y el fichero discrepan en una
decisión puramente visual, prevalece `.clang-format`.

### 4.1 Reglas principales

- Sangría: **2 espacios**.
- Tabs: prohibidos en C y ensamblador, salvo donde una sintaxis externa los exija.
- Límite blando: **100 columnas**.
- Llaves: estilo attach/K&R moderno.
- Llaves obligatorias en todos los cuerpos de control.
- Punteros: alineación a la izquierda, `void* ptr`.
- Una declaración por sentencia.
- Variables declaradas cerca de su primer uso útil.
- Espacio antes del paréntesis de control, no en llamadas.
- Dos espacios antes de un comentario final `//`.
- main debe estar arriba del todo, el resto de funciones van abajo y declaradas arriba con prototipos

Ejemplo:

```c
static ErrorCode load_file(const wchar_t* path, Buffer* out_buffer) {
  if (path == NULL || out_buffer == NULL) {
    return ERROR_INVALID_ARGUMENT;
  }

  FILE* file = NULL;
  errno_t error = _wfopen_s(&file, path, L"rb");
  if (error != 0 || file == NULL) {
    return ERROR_IO;
  }

  if (fclose(file) != 0) {
    return ERROR_IO;
  }

  return ERROR_OK;
}
```

### 4.2 Comentarios en línea

Cuando un comentario comparte línea con código, debe haber **exactamente dos espacios** antes de `//`:

```c
puts("listo");  // Informa de que la operación terminó.
```

Esto es incorrecto:

```c
puts("listo"); // Falta un espacio.
```

Un comentario que ocupa su propia línea sigue la sangría normal del bloque:

```c
if (is_ready) {
  // Se publica el estado después de inicializar todos los campos.
  publish_state();
}
```

### 4.3 Llaves

Correcto:

```c
if (condition) {
  run();
} else {
  recover();
}
```

Incorrecto:

```c
if (condition)
  run();
```

La obligación de usar llaves la comprueba clang-tidy; clang-format no debe insertar o eliminar llaves
por sí solo.

### 4.4 Punteros

El proyecto usa:

```c
const char* text;
void* memory;
unsigned char* bytes;
```

La grafía `type* name` es una convención visual. En la gramática de C, `*` pertenece al declarator;
por eso sigue estando prohibido declarar varios objetos en la misma sentencia:

```c
int* first;
int second;
```

Nunca:

```c
int* first, second;
```

---

## 5. Ficheros, cabeceras e includes

### 5.1 Nombres

- C: `snake_case.c`, `snake_case.h`.
- NASM: `snake_case.asm` o `snake_case.nasm`.
- MASM: `snake_case.asm`.
- Tests: `module_test.c`.
- Cabecera interna: `module_internal.h`.

Se deben evitar nombres vagos como `misc.c`, `stuff.c`, `utils.c` o sufijos temporales como
`parser_new.c`.

Un fichero se divide por responsabilidad, no para satisfacer un número. A partir de unas 800-1000
líneas de lógica manual debe revisarse su cohesión. Tablas generadas, datos constantes y código
generado no se evalúan con el mismo criterio.

### 5.2 `#pragma once`

Las cabeceras del proyecto deben usar:

```c
#pragma once
```

Esta decisión es deliberadamente Windows/Clang/MSVC-oriented. Si un target futuro requiere un
compilador que no lo soporte, la excepción debe decidirse a nivel de proyecto.

### 5.3 Cabeceras autocontenidas

Toda cabecera pública debe compilar al incluirse sola en una translation unit mínima.

```c
#include "module.h"
```

No se debe depender de includes transitivos.

### 5.4 Orden de includes

Orden recomendado en un `.c`:

1. Cabecera pareja del módulo.
2. Cabeceras estándar de C.
3. Otras cabeceras entre `<...>`: Windows SDK, sistema objetivo y terceros.
4. Otras cabeceras del proyecto entre `"..."`, públicas o internas.

Dentro de cada bloque, `.clang-format` aplica orden case-sensitive. No se fuerzan subgrupos que el
fichero de formato volvería a unir.

```c
#include "network_client.h"

#include <stddef.h>
#include <stdint.h>

#include <Winsock2.h>
#include <Ws2Tcpip.h>

#include "network_internal.h"
#include "project/log.h"
```

### 5.5 Capitalización de cabeceras Win32

La capitalización de las cabeceras del Windows SDK es normativa para el proyecto. No se debe
convertir todo a minúsculas ni aplicar una regla simplista de «solo la primera letra en mayúscula».

Grafías canónicas del proyecto, entre otras:

```c
#include <Windows.h>
#include <Winsock2.h>
#include <Ws2Tcpip.h>
```

La grafía debe copiarse de la lista canónica del proyecto, fijada a partir del Windows SDK objetivo.
Aunque NTFS suela resolver nombres sin distinguir mayúsculas, el repositorio debe conservar una forma
única y revisable.

Cuando se use Winsock, `Winsock2.h` debe aparecer antes que cualquier inclusión que pueda arrastrar
Winsock 1.1. Normalmente no es necesario incluir también `Windows.h`.

### 5.6 Configuración de `Windows.h`

Estas macros deben definirse en el sistema de build o en una única cabecera central, nunca repetirse
arbitrariamente por el código:

```c
#define WIN32_LEAN_AND_MEAN
#define NOMINMAX
#define UNICODE
#define _UNICODE
#include <Windows.h>
```

El proyecto usa APIs `W` explícitas en las fronteras Win32; no usa `TCHAR` como abstracción interna.

---

## 6. Nombres

| Elemento                      | Convención                     | Ejemplo             |
| ----------------------------- | ------------------------------ | ------------------- |
| Funciones                     | `snake_case`                   | `parse_packet`      |
| Variables y parámetros        | `snake_case`                   | `buffer_size`       |
| Tipos                         | `PascalCase`                   | `ParseResult`       |
| Macros                        | `UPPER_SNAKE_CASE`             | `ARRAY_COUNT`       |
| Enumeradores                  | `UPPER_SNAKE_CASE` con prefijo | `TOKEN_EOF`         |
| Estado estático de fichero    | `s_`                           | `s_instance_count`  |
| Valores inmutables de fichero | `k_`                           | `k_default_timeout` |
| Parámetros de salida          | `out_`                         | `out_result`        |
| Parámetros de entrada/salida  | `inout_`                       | `inout_cursor`      |

Los símbolos públicos deben llevar prefijo de proyecto o subsistema:

```c
net_socket_open();
net_socket_close();
image_decode();
```

No se deben exportar símbolos genéricos como `init`, `open`, `run`, `create` o `destroy` sin prefijo.

Los booleanos deben leerse como predicados:

```c
bool is_ready;
bool has_error;
bool owns_memory;
```

---

## 7. Tipos y declaraciones

### 7.1 Tipos estándar

- `size_t` para tamaños, cantidades e índices de arrays.
- `ptrdiff_t` para diferencias firmadas entre punteros.
- `uint32_t`, `int64_t`, etc. cuando el ancho exacto importa.
- `uintptr_t` e `intptr_t` para conversiones justificadas entre punteros y enteros.
- `int` para APIs que lo exigen y valores naturalmente pequeños sin requisito de ancho.
- `unsigned char` para almacenamiento byte a byte cuando esa semántica sea necesaria.
- `char` para texto estrecho y buffers de cadenas C.

`uint8_t` debe reservarse para un entero sin signo exactamente de 8 bits, no como sustituto automático
de cualquier bloque de memoria.

### 7.2 `typedef`

`typedef` se usa cuando aporta semántica o encapsulación:

```c
typedef uint32_t EntityId;
typedef struct Database Database;
typedef ErrorCode (*PacketHandler)(Context* context, const Packet* packet);
```

No se usa para inventar nombres alternativos de primitivas sin significado adicional:

```c
typedef unsigned int uint;  // Prohibido.
typedef int i32;  // Prohibido.
```

### 7.3 `const`

Los parámetros puntero de solo lectura deben usar `const`:

```c
ErrorCode parse_data(const unsigned char* data, size_t size, Result* out_result);
```

`const` en C expresa inmutabilidad a través de ese acceso, pero no convierte una variable en una
integer constant expression. Para constantes enteras necesarias en contextos de compilación se usan
enums o macros justificadas:

```c
enum {
  MAX_CONNECTIONS = 1024,
};
```

### 7.4 Declaraciones y alcance

Las variables deben declararse en el alcance más pequeño razonable y cerca de su primer uso.

```c
ErrorCode parse(const char* text) {
  if (text == NULL) {
    return ERROR_INVALID_ARGUMENT;
  }

  size_t length = strlen(text);
  if (length == 0) {
    return ERROR_INVALID_FORMAT;
  }

  return parse_nonempty(text, length);
}
```

No se exige agrupar todas las declaraciones al principio del bloque.

### 7.5 Conversiones

Toda conversión que pueda perder información debe validarse en runtime cuando dependa de datos
externos. Para un valor de origen firmado:

```c
if (value < 0 || value > UINT8_MAX) {
  return ERROR_OVERFLOW;
}

uint8_t byte = (uint8_t)value;
```

Una assertion no sustituye la validación de datos externos, porque puede desaparecer en release.

Las conversiones integer-to-pointer o pointer-to-integer deben pasar por `uintptr_t`/`intptr_t` y
tener una justificación clara.

---

## 8. Estructuras, enums, unions y datos binarios

### 8.1 Layout

Los campos se agrupan primero por significado. En estructuras medidas como hot path, se permite
reordenar por localidad de datos si la mejora está demostrada.

Cuando el layout forme parte de un ABI o formato binario, se debe documentar y comprobar:

```c
_Static_assert(sizeof(FileHeader) == 32, "FileHeader size mismatch");
_Static_assert(offsetof(FileHeader, magic) == 0, "FileHeader.magic offset mismatch");
_Static_assert(offsetof(FileHeader, version) == 4, "FileHeader.version offset mismatch");
```

No basta con comprobar únicamente `sizeof`: deben comprobarse también los offsets y alineamientos
relevantes para el contrato.

No se deben serializar, hashear ni comparar mediante `memcmp` estructuras C ordinarias que puedan
contener padding. Solo se permite cuando el layout y el valor de todos los bytes, incluido cualquier
padding, estén definidos e inicializados explícitamente por el contrato.

### 8.2 Estructuras packed

Solo se permiten para layouts externos: ficheros, protocolos, hardware o firmware.

La decodificación explícita desde bytes suele ser preferible a superponer una estructura packed sobre
datos externos. Una estructura packed solo se usa cuando existe una razón clara y cuando el mecanismo
de packing de cada toolchain está aislado y verificado.

No se debe trabajar directamente con miembros potencialmente desalineados. Se decodifican mediante
helpers de bytes o se copian con `memcpy` a objetos normales correctamente alineados antes de usarlos.
Las comprobaciones de `sizeof`, `offsetof`, endianness y alineamiento siguen siendo obligatorias cuando
formen parte del layout contractual.

### 8.3 Bitfields

Los bitfields no deben usarse para protocolos, ficheros, registros hardware o ABI. Su layout es
dependiente de implementación. Se prefieren máscaras y shifts sobre enteros de ancho fijo.

### 8.4 Type punning y alineamiento

No se reinterpretan objetos mediante casts de puntero incompatibles:

```c
static uint32_t float_bits(float value) {
  uint32_t bits;
  _Static_assert(sizeof bits == sizeof value, "float and uint32_t size mismatch");
  memcpy(&bits, &value, sizeof bits);
  return bits;
}
```

Un objeto no tiene que inicializarse a cero si va a ser escrito completamente antes de leerse. La
regla real es: **nunca leer un valor indeterminado**.

### 8.5 Endianness

Todo formato persistente o transmitido debe declarar su endianness y usar helpers de lectura/escritura.
Que la arquitectura x86-64 sea little-endian no convierte el formato externo en implícitamente
little-endian.

---

## 9. Funciones y control de flujo

### 9.1 Tamaño y cohesión

Una función debe tener una responsabilidad reconocible.

- Recomendación: hasta 60 líneas lógicas.
- Revisión especial: más de 100 líneas lógicas.
- Anidamiento recomendado: hasta 3 niveles.
- Revisión especial: 4 o más niveles.
- Parámetros recomendados: hasta 5.
- Revisión especial: 7 o más.

Estos números activan revisión, no una división automática. Una función cohesiva de 110 líneas puede
ser mejor que diez helpers artificiales.

### 9.2 Guard clauses

Se prefieren guard clauses para reducir anidamiento:

```c
if (context == NULL) {
  return ERROR_INVALID_ARGUMENT;
}

if (!context->is_ready) {
  return ERROR_INVALID_STATE;
}
```

### 9.3 `goto`

`goto` está permitido para cleanup y unwinding lineal de recursos:

```c
ErrorCode result = ERROR_OK;
FILE* file = NULL;
void* memory = NULL;

if (_wfopen_s(&file, path, L"rb") != 0) {
  result = ERROR_IO;
  goto cleanup;
}

if (size != 0) {
  memory = malloc(size);
  if (memory == NULL) {
    result = ERROR_OUT_OF_MEMORY;
    goto cleanup;
  }
}

cleanup:
  free(memory);
  if (file != NULL) {
    if (fclose(file) != 0 && result == ERROR_OK) {
      result = ERROR_IO;
    }
  }
  return result;
```

No se permite usar `goto` para construir bucles o saltar entre responsabilidades no relacionadas.

### 9.4 Recursión

La recursión está permitida cuando el límite de profundidad es pequeño, demostrable y adecuado para
el stack. Parsers o recorridos controlados por entrada externa deben imponer un límite explícito o
usar una pila iterativa.

### 9.5 Bucles

Todo bucle debe tener una condición de terminación evidente o documentada. No se exige un límite
estático artificial para bucles naturales de aplicaciones, pero los bucles sobre datos no confiables
deben estar acotados por tamaños validados.

### 9.6 `switch`

- Los `case` se indentan un nivel.
- El fallthrough debe ser explícito mediante un comentario reconocido por el analizador, como
  `/* fall through */`, o una macro de proyecto que aísle un atributo soportado por la toolchain.
- Un `case` que declara variables debe usar bloque.
- Los enums deben revisarse para valores desconocidos recibidos desde ABI, ficheros o red.

`[[fallthrough]]` no forma parte de C17 y no se usa directamente en fuentes que deban cumplir esta
guía.

---

## 10. Ownership, lifetime y memoria

### 10.1 Vocabulario

- **Borrowed:** el receptor no adquiere ownership.
- **Owned/acquired:** el receptor debe liberar el recurso.
- **Transferred:** el ownership cambia de propietario.
- **Consumed:** la función puede almacenar, modificar o destruir el recurso.

Las APIs no triviales deben dejar claro quién libera cada recurso y hasta cuándo es válido un puntero.

### 10.2 Asignación

Toda asignación puede fallar y debe comprobarse.

No se debe llamar a `realloc` con tamaño cero. La liberación se expresa explícitamente mediante
`free`. Todo helper de realloc debe resolver `new_size == 0` antes de llamar a la CRT y documentar si
libera el bloque y publica `NULL` o si devuelve un error.

Nunca se sobrescribe el puntero original antes de validar `realloc`. Para un `new_size` ya comprobado
como mayor que cero:

```c
void* new_memory = realloc(memory, new_size);
if (new_memory == NULL) {
  return ERROR_OUT_OF_MEMORY;
}

memory = new_memory;
```

La función que asigna y la que libera deben pertenecer al mismo allocator o contrato.

### 10.3 Estrategias

- Stack para objetos pequeños y lifetime léxico.
- Heap para ownership independiente.
- Arenas para grupos de objetos con el mismo lifetime.
- Pools para objetos homogéneos y frecuentes.
- `VirtualAlloc` para reservas grandes o control explícito de memoria virtual en Windows.

No se debe elegir arena, pool o memoria virtual por moda; debe corresponder al lifetime y patrón de uso.

### 10.4 Destructores explícitos

Todo tipo que posea recursos debe tener una operación de cleanup coherente, normalmente `_destroy`,
`_close` o `_release`.

Se recomienda diseñar el cleanup para que sea seguro sobre el estado cero y sobre estados
parcialmente inicializados. Cuando ese contrato exista, cada miembro adquirido se publica en el
objeto inmediatamente y el destructor ignora miembros todavía vacíos.

Una función que construya un objeto con múltiples recursos debe elegir y documentar uno de estos
contratos:

1. Es transaccional: en error libera internamente todo lo adquirido y no deja ownership pendiente.
2. Puede dejar un objeto parcial: el caller DEBE ejecutar el destructor seguro antes de abandonar el
   path de error.

No se debe retornar directamente desde una construcción fallida si el contrato permite ownership
parcial y todavía no se ha ejecutado su cleanup. El out-parameter final solo se publica cuando el
objeto está en un estado válido, salvo que la API documente expresamente otra semántica.

---

## 11. Errores, assertions y logging

### 11.1 Assertions

Las assertions verifican errores de programación e invariantes internas. No validan:

- entrada del usuario;
- ficheros;
- red;
- dispositivos;
- fallos de asignación;
- errores normales del sistema operativo.

No existe una cuota de assertions. Se añade una assertion cuando expresa un invariante útil y barato.

### 11.2 Errores recuperables

Los fallos recuperables deben devolverse mediante:

- un `ErrorCode` del dominio;
- un valor especial inequívoco;
- un resultado más out-parameter;
- un objeto de resultado documentado.

No se deben perder detalles esenciales al convertir errores Win32. `GetLastError()` y
`WSAGetLastError()` deben capturarse inmediatamente después del fallo correspondiente.

### 11.3 Logging

Los logs deben ser accionables y no sustituir la propagación de errores.

Nunca deben contener secretos, tokens, contraseñas o buffers arbitrarios de entrada no confiable.

Una función de librería no debe terminar el proceso por un error recuperable salvo que su contrato sea
explícitamente fatal.

---

## 12. Seguridad, UB y aritmética

Están prohibidos:

- overflow signed;
- accesos fuera de límites;
- use-after-free y double free;
- shifts por una cantidad igual o superior al ancho;
- división por cero;
- desreferencia nula;
- violaciones de strict aliasing;
- acceso desalineado no permitido;
- lectura de valores indeterminados;
- data races;
- llamadas mediante function pointers incompatibles.

La aritmética de tamaños debe comprobar overflow antes de asignar:

```c
if (element_size != 0 && count > SIZE_MAX / element_size) {
  return ERROR_OVERFLOW;
}

size_t total_size = count * element_size;
```

También deben comprobarse sumas y redondeos usados para tamaños:

- `a + b`;
- `count * element_size`;
- `offset + length`;
- aritmética de alineamiento.

Para validar un rango dentro de un buffer se prefiere una forma que no calcule primero una suma
potencialmente desbordada:

```c
if (offset > buffer_size || length > buffer_size - offset) {
  return ERROR_OUT_OF_BOUNDS;
}
```

Un proyecto que repita estas operaciones puede proporcionar helpers pequeños:

```c
static bool size_add(size_t a, size_t b, size_t* out_result) {
  if (out_result == NULL || b > SIZE_MAX - a) {
    return false;
  }

  *out_result = a + b;
  return true;
}

static bool size_mul(size_t a, size_t b, size_t* out_result) {
  if (out_result == NULL || (a != 0 && b > SIZE_MAX / a)) {
    return false;
  }

  *out_result = a * b;
  return true;
}

static bool size_align_up(size_t value, size_t alignment, size_t* out_result) {
  if (out_result == NULL || alignment == 0 || (alignment & (alignment - 1)) != 0) {
    return false;
  }

  size_t mask = alignment - 1;
  if (value > SIZE_MAX - mask) {
    return false;
  }

  *out_result = (value + mask) & ~mask;
  return true;
}
```

En este ejemplo, `size_align_up` acepta únicamente alineamientos potencia de dos. Un proyecto pequeño
no está obligado a crear estos helpers si la comprobación local es igual de clara.

Las rotaciones deben normalizar el shift para evitar desplazar por el ancho completo:

```c
static inline uint32_t rotate_left_u32(uint32_t value, unsigned shift) {
  shift &= 31u;
  return (value << shift) | (value >> ((32u - shift) & 31u));
}
```

---

## 13. CRT y entrada segura

«Seguro» no significa únicamente usar un nombre terminado en `_s`. Debe elegirse la operación que
mejor limite y valide la entrada.

### 13.1 Prioridad para entrada textual

Orden preferido:

1. `fgets`/`fgetws` con buffer acotado y parser explícito.
2. Helpers del proyecto que devuelvan errores detallados.
3. `scanf_s`/`sscanf_s` para herramientas pequeñas y formatos muy controlados.
4. `scanf`/`sscanf` únicamente cuando el formato esté completamente acotado y revisado.

`scanf_s` añade requisitos de tamaño para conversiones de caracteres y cadenas, por lo que es más
segura que `scanf` en esos casos; aun así, no resuelve por sí sola recuperación de entrada, trailing
data, rangos ni formatos ambiguos.

Ejemplo recomendado:

```c
static ErrorCode parse_int32(const char* text, int32_t* out_value) {
  if (text == NULL || out_value == NULL) {
    return ERROR_INVALID_ARGUMENT;
  }

  errno = 0;
  char* end = NULL;
  long parsed = strtol(text, &end, 10);

  if (end == text) {
    return ERROR_INVALID_FORMAT;
  }

  if (errno == ERANGE) {
    return ERROR_OVERFLOW;
  }

  while (*end != '\0' && isspace((unsigned char)*end)) {
    ++end;
  }

  if (*end != '\0') {
    return ERROR_INVALID_FORMAT;
  }

  if (parsed < (long)INT32_MIN || parsed > (long)INT32_MAX) {
    return ERROR_OVERFLOW;
  }

  *out_value = (int32_t)parsed;
  return ERROR_OK;
}
```

Este ejemplo acepta whitespace inicial mediante `strtol` y whitespace final mediante `isspace`, pero
rechaza `123abc`. Si la gramática no permite espacios, debe exigirse que `end` apunte directamente al
terminador. El argumento de `isspace` se convierte a `unsigned char` para evitar comportamiento
indefinido con valores `char` negativos.

Cuando el texto proceda de `fgets`, el código debe detectar una línea truncada por falta de espacio en
el buffer antes de tratar el fragmento como una entrada completa.

### 13.2 Funciones prohibidas o desaconsejadas

Prohibidas:

```text
gets, strcpy, strcat, sprintf, vsprintf, tmpnam
```

Desaconsejadas salvo contrato muy claro:

```text
strncpy, strncat, strtok, atoi, atof, scanf, fscanf, sscanf
```

Alternativas habituales:

- `snprintf` o `_snprintf_s` para salida acotada;
- `strcpy_s`/`strcat_s` o helpers propios con tamaño explícito;
- `strtol`, `strtoll`, `strtoull`, `strtod` con validación completa;
- tokenizer explícito o `strtok_s` cuando su semántica sea suficiente;
- `_wfopen_s` o Win32 `CreateFileW` para paths Windows Unicode.

Las funciones `_s` pueden invocar el invalid-parameter handler ante contratos inválidos. Sus valores
de retorno deben comprobarse y no se deben usar como decoración.

No se define globalmente `_CRT_SECURE_NO_WARNINGS` para ocultar problemas. Una excepción para código
legacy debe quedar limitada a un wrapper pequeño.

### 13.3 Formatos

Una cadena no confiable nunca se pasa como format string:

```c
printf("%s", user_text);
```

Nunca:

```c
printf(user_text);
```

### 13.4 Formatos y tipos

Cada argumento de una función `printf` debe corresponder exactamente con su format specifier. Se usan
los especificadores estándar y las macros de `<inttypes.h>`:

```c
printf("size=%zu delta=%td count=%" PRIu64 " address=0x%" PRIxPTR "\n",
       size,
       delta,
       count,
       address);
```

Correspondencias relevantes:

- `size_t`: `%zu`;
- `ptrdiff_t`: `%td`;
- `uint64_t`: `PRIu64`;
- `uintptr_t`: `PRIxPTR`, `PRIuPTR` u otra macro del formato deseado.

No se introducen casts únicamente para adaptar un valor a un especificador incorrecto. Windows x64
usa el modelo LLP64: `long` y `unsigned long` son normalmente de 32 bits aunque los punteros sean de
64 bits.

---

## 14. Windows y Win32

### 14.1 Texto y Unicode

- Texto interno: UTF-8, salvo que un subsistema tenga una razón documentada.
- Frontera Win32: UTF-16 mediante APIs `W`.
- No se usan APIs `A` para paths o texto del usuario.
- La conversión ocurre en helpers centralizados y valida secuencias inválidas.

### 14.2 Handles

Cada API Win32 define su propio valor inválido. No se debe asumir que todos los `HANDLE` fallan con
`NULL` ni que todos fallan con `INVALID_HANDLE_VALUE`.

Todo handle válido debe cerrarse con la función correspondiente, no necesariamente `CloseHandle`:

- `CloseHandle` para muchos kernel handles;
- `closesocket` para `SOCKET`;
- `RegCloseKey` para `HKEY`;
- `FindClose` para handles de búsqueda;
- funciones específicas para GDI, COM y otros subsistemas.

### 14.3 COM en C

En C se usa la forma C del vtable o los macros proporcionados por las cabeceras:

```c
IUnknown_Release(object);
```

No se escribe sintaxis C++ `object->Release()` dentro de código C.

Cada thread que use COM debe inicializarlo y equilibrar correctamente su finalización.

### 14.4 `DllMain`

`DllMain` debe ser mínimo. No debe iniciar threads, cargar DLLs arbitrarias, adquirir locks externos ni
realizar trabajo que pueda bloquear bajo el loader lock.

### 14.5 Paths

Los paths Win32 deben representarse como UTF-16 en la frontera. El proyecto debe habilitar y probar
long paths cuando sean requisito; no debe diseñar buffers suponiendo que 260 caracteres bastan.

### 14.6 Hardening de release

La build final debe habilitar las mitigaciones compatibles con el compilador y linker fijados. Como
base con `cl`/`clang-cl` + `link`:

```text
/GS                # Cookie y comprobaciones de seguridad del compilador.
/guard:cf          # Compilación con CFG.
/GUARD:CF          # Imagen enlazada con CFG.
/DYNAMICBASE       # ASLR.
/NXCOMPAT          # DEP.
/HIGHENTROPYVA     # ASLR de alta entropía en imágenes x64.
```

`/GS` no debe deshabilitarse sin una excepción localizada y revisada. Si el proyecto sustituye los
entry points del runtime, debe cumplir también el contrato de inicialización requerido por la
mitigación.

Cuando se compile y enlace por pasos separados, las opciones de CFG deben aplicarse en ambos pasos.
`/HIGHENTROPYVA` solo tiene sentido para imágenes de 64 bits y requiere las opciones de imagen
compatibles, incluidas `/DYNAMICBASE` y `/LARGEADDRESSAWARE`.

Se recomienda evaluar:

```text
/CETCOMPAT
```

para artefactos x64 cuando el linker y todas las dependencias relevantes sean compatibles con CET
Shadow Stack y el producto se pruebe bajo esa mitigación. Opciones como `/guard:ehcont` se habilitan
solo cuando el toolset y el target fijados las soportan y la build verifica su interacción con el
código generado, ensamblador, JITs, callbacks y dependencias.

Los flags exactos DEBEN verificarse contra el compilador, linker, arquitectura y versión fijados. La
lista anterior no autoriza a copiar opciones incompatibles, redundantes o no probadas.

Los binarios de release deben conservar PDB archivados junto a la versión exacta del artefacto.

---

## 15. Concurrencia

- Toda variable compartida mutable debe tener un propietario y mecanismo de sincronización.
- Las data races son bugs, no optimizaciones.
- `volatile` no es sincronización.
- Los atomics deben usar un orden de memoria correcto y explícito. Se prefiere el modelo más fácil de
  razonar; debilitarlo por rendimiento requiere explicación y medición cuando empeore la claridad.
- El orden de adquisición de múltiples locks debe documentarse.
- No se debe llamar código externo o callbacks arbitrarios mientras se mantiene un lock, salvo contrato
  explícito.

El código común de Windows que deba compilar con `cl`, `clang-cl`, `clang` e `icx` no debe asumir que
`<stdatomic.h>` de C17 tiene el mismo grado de soporte, representación o interoperabilidad práctica en
todas las toolchains fijadas.

Cuando sea necesaria compatibilidad entre toolchains, se debe elegir una de estas estrategias:

- locks Win32 para invariantes que abarcan varios valores;
- operaciones `Interlocked*` para estados simples y correctamente alineados;
- wrappers pequeños del proyecto que oculten la diferencia de toolchain;
- `<stdatomic.h>` únicamente cuando la matriz concreta lo soporte y esté verificada.

No se debe exponer `_Atomic` ni un tipo atómico específico del compilador como parte de una ABI pública
estable. Tampoco se debe acceder al mismo objeto atómico desde módulos compilados con implementaciones
incompatibles sin un contrato probado. Los wrappers deben especificar operación, ancho, alineamiento y
orden de memoria; no deben convertirse en una jerarquía abstracta si un lock o una llamada
`Interlocked*` resuelven el caso con más claridad.

En Windows se prefieren primitivas adecuadas al caso: `SRWLOCK`, `CRITICAL_SECTION`, condition
variables, eventos, semáforos e InitOnce.

TSan no es un requisito de CI nativa Windows. Puede ejecutarse en una build auxiliar Linux compatible
con el código común. En Windows se complementa con tests de estrés, revisión, Application Verifier y
herramientas de diagnóstico adecuadas.

---

## 16. APIs, ABI y librerías

### 16.1 Superficie pública

La API pública debe ser pequeña y usar prefijos. Los detalles internos permanecen `static` o detrás de
tipos opacos.

```c
typedef struct Database Database;

ErrorCode database_open(const wchar_t* path, Database** out_database);
void database_close(Database* database);
```

### 16.2 ABI estable

Se distinguen tres niveles:

1. ABI interna dentro de un mismo ejecutable o conjunto de objetos de una build.
2. ABI entre módulos compilados con la misma toolchain, opciones y versión fijadas.
3. ABI pública estable entre versiones y, cuando se declare, entre toolchains.

Una decisión válida para el primer nivel no se considera automáticamente segura para el tercero.

Para ABI estable se prefieren handles opacos. Una estructura pública extensible debe incluir tamaño y,
cuando proceda, versión:

```c
typedef struct {
  uint32_t struct_size;
  uint32_t version;
  uint32_t flags;
} LibraryConfig;
```

Añadir campos al final no es automáticamente ABI-safe: depende de quién asigna la estructura, cómo se
pasa y qué tamaño espera cada lado.

Cuando el ancho o la representación formen parte del contrato se prefieren tipos explícitos como
`uint32_t` e `int64_t`. En una ABI pública estable deben revisarse especialmente:

- `enum`: el tipo compatible y su representación no se tratan como contrato estable entre toolchains;
- `bool`: su tamaño y representación de ABI no se presuponen fuera de la matriz fijada;
- `long`: Windows x64 usa LLP64 y mantiene `long` normalmente en 32 bits;
- `wchar_t`: es apropiado en fronteras Win32 `W`, pero no representa por sí solo un contrato Unicode
  universal ni idéntico entre todas las plataformas y opciones de compilación;
- padding, bitfields y alineamiento de estructuras;
- estructuras y unions pasadas o devueltas por valor;
- calling convention y reglas de retorno de agregados.

No se pasan estructuras inestables por valor a través de una frontera DLL. Si una estructura pública
es inevitable, su tamaño, alineamiento, offsets, ownership y dirección de compatibilidad deben quedar
documentados y comprobados. Los bitfields no forman parte de una ABI pública estable.

No se serializa, hashea ni compara con `memcmp` una estructura C ordinaria que pueda contener padding,
conforme a §8.1.

### 16.3 Exportaciones

Las exportaciones deben centralizarse en una macro de API. El calling convention forma parte del ABI y
no debe cambiar sin bump mayor. La macro debe expandirse de forma coherente en todas las toolchains
soportadas y no depender de defaults distintos entre translation units.

---

## 17. SIMD, intrinsics y rendimiento

El rendimiento se optimiza después de perfilar. Primero se escribe una implementación clara,
obviamente correcta y con complejidad algorítmica apropiada. Se permite al compilador realizar las
transformaciones mecánicas que pueda demostrar correctas; no se depende de él para reparar una
estructura de datos o un algoritmo inadecuados.

Solo se introducen optimizaciones manuales que empeoren la legibilidad cuando una medición demuestre
que son necesarias o exista un requisito explícito de rendimiento. Entre dos implementaciones
razonablemente eficientes se prefiere la más clara mientras no exista evidencia en contra.

En código marcado expresamente como hot path, kernel numérico, SIMD, codec, crypto, procesamiento en
tiempo real o sistema de muy baja latencia se permite priorizar rendimiento de forma más agresiva.
Estas excepciones deben conservar tests de corrección, benchmarks reproducibles, documentación de la
baseline y una explicación `PERF` cuando la decisión no sea evidente.

La variante `icx` de §3.4 existe para todos los proyectos aplicables. En componentes extremadamente
sensibles al rendimiento, la toolchain oficial y `icx` deben compararse sobre hardware representativo;
no se atribuye una mejora a la marca del compilador sin mediciones reproducibles.

- Mantener una implementación escalar correcta.
- Aislar implementaciones SIMD por ISA.
- Hacer dispatch una vez, no comprobar CPUID dentro del hot loop.
- Comprobar tanto capacidades de CPU como soporte del OS para el estado extendido.
- Para AVX/AVX2 no basta con leer solo el bit AVX2: deben comprobarse AVX, OSXSAVE y XCR0.
- No asumir AVX-512 como baseline.

Los hints de branch, prefetches y forced inline requieren benchmark que demuestre beneficio.

---

## 18. Ensamblador

Esta es la única sección normativa de ensamblador de la guía.

### 18.1 Herramientas y sintaxis

En Windows se permiten:

- **NASM**, sintaxis Intel, formato `win64`;
- **MASM**, sintaxis Microsoft x64.

En Linux, cuando el target esté declarado, se permite NASM con formato `elf64`. MASM es exclusivo de
la build Windows.

No se mezclan sintaxis NASM y MASM dentro del mismo fichero.

### 18.2 Formato

- Sangría: 2 espacios.
- Labels principales: columna 0.
- Instrucciones: 2 espacios.
- Comentario final: 2 espacios antes de `;`.
- Comentario en línea propia: sigue la sangría del bloque.

NASM:

```nasm
copy_byte:
  ; RCX: origen, RDX: tamaño disponible, R8: destino.
  ; Leaf; destruye RAX. Con RDX != 0, RCX y R8 apuntan al menos a un byte válido.
  test rdx, rdx
  jz .done

  mov al, [rcx]  ; Lee un byte del origen.
  mov [r8], al  ; Escribe el byte en el destino.

.done:
  ret
```

Comentario independiente:

```nasm
copy_byte:
  ; La función es leaf y no modifica registros no volátiles.
  mov al, [rcx]
  ret
```

MASM:

```asm
copy_byte PROC
  ; RCX: origen, RDX: tamaño disponible, R8: destino.
  ; Leaf; destruye RAX. Con RDX != 0, RCX y R8 apuntan al menos a un byte válido.
  test rdx, rdx
  jz copy_byte_done

  mov al, [rcx]  ; Lee un byte del origen.
  mov [r8], al  ; Escribe el byte en el destino.

copy_byte_done:
  ret
copy_byte ENDP
```

### 18.3 ABI de Windows x64

Los cuatro primeros argumentos enteros o de puntero se reciben, según su posición, en:

```text
RCX
RDX
R8
R9
```

Los argumentos de coma flotante de las cuatro primeras posiciones usan `XMM0`-`XMM3` conforme a las
reglas del ABI. Los argumentos restantes se pasan por stack. Las funciones variádicas y los agregados
tienen reglas adicionales y no deben implementarse por intuición.

Registros volátiles:

```text
RAX
RCX
RDX
R8
R9
R10
R11
XMM0-XMM5
```

El caller debe asumir que una llamada destruye esos registros.

Registros no volátiles:

```text
RBX
RBP
RSI
RDI
RSP
R12-R15
XMM6-XMM15
```

Toda función que modifique un registro no volátil DEBE restaurarlo exactamente antes de retornar y
describir su ubicación en la metadata de unwind cuando corresponda.

La preservación de `XMM6`-`XMM15` cubre su estado de 128 bits. Las partes superiores de
`YMM0`-`YMM15` y `ZMM0`-`ZMM15` son volátiles. `ZMM16`-`ZMM31` también son volátiles; cuando
AVX-512VL permite sus formas estrechas, lo son igualmente `XMM16`-`XMM31` y `YMM16`-`YMM31`. El
código que use AVX, AVX-512 u otro estado extendido debe comprobar tanto la ISA de la CPU como el
soporte del sistema operativo conforme a §17.

### 18.4 Stack y llamadas

El caller DEBE reservar 32 bytes de shadow space para toda llamada, aunque el callee tenga menos de
cuatro parámetros. Ese espacio pertenece temporalmente al callee y no se usa para conservar valores
del caller a través de la llamada.

En una frame function, `RSP` debe estar alineado a 16 bytes en las regiones que no sean prólogo o
epílogo y, en particular, inmediatamente antes de ejecutar `call`. Al entrar en una función, la
dirección de retorno ya ocupa 8 bytes; cualquier reserva debe tenerlo en cuenta junto con el shadow
space y los argumentos de stack.

Toda función DEBE restaurar `RSP` exactamente al valor que tenía a su entrada antes de ejecutar
`ret`. No existe red zone estilo System V en el ABI Windows x64. La memoria por debajo del `RSP`
actual no se trata como almacenamiento privado persistente: primero se ajusta `RSP` mediante un
prólogo válido y solo después se accede al frame reservado.

### 18.5 Leaf functions y frame functions

Una leaf function:

- no llama a ninguna otra función;
- no modifica `RSP`;
- no modifica registros no volátiles;
- no crea un stack frame;
- no necesita metadata de unwind porque puede desenrollarse simulando un `ret`.

Una función que llama a otra función, modifica registros no volátiles, reserva stack, crea un frame,
usa manejo de excepciones o necesita almacenamiento local no trivial debe tratarse como frame function
cuando corresponda. Una frame function requiere prólogo, epílogo y metadata de unwind coherentes con
las operaciones reales.

No se etiqueta una función como leaf únicamente porque la llamada esté en un path infrecuente o
porque el ensamblador no genere la metadata automáticamente.

### 18.6 Prólogos, epílogos y unwind metadata

Toda función assembly Windows x64 no trivial que modifique el stack, guarde registros no volátiles,
llame a otra función o pueda participar en stack walking DEBE cumplir las restricciones de prólogo y
epílogo y generar la metadata `.pdata`/`.xdata` apropiada.

Con MASM se usan, según las operaciones reales:

```text
PROC FRAME
.pushreg
.allocstack
.savereg
.savexmm128
.setframe
.endprolog
```

Estas directivas describen operaciones; no sustituyen las instrucciones que guardan registros,
reservan stack o establecen el frame pointer. La descripción y el código DEBEN coincidir exactamente.
El orden concreto de directivas se ajusta a la versión de unwind y de `ml64` fijada por el proyecto;
no se mezclan reglas de formatos experimentales con la metadata ordinaria.

Ejemplo MASM de una frame function que reserva shadow space, mantiene el alineamiento y llama a otra
función:

```asm
EXTERN target_function:PROC
PUBLIC call_target

_TEXT SEGMENT

call_target PROC FRAME
  sub rsp, 28h
  .allocstack 28h
  .endprolog

  call target_function

  add rsp, 28h
  ret
call_target ENDP

_TEXT ENDS
END
```

Los epílogos deben usar únicamente las formas reconocidas por el unwinder para restaurar `RSP`,
registros no volátiles y retornar o realizar un tail jump permitido. No se insertan instrucciones
arbitrarias dentro de un epílogo.

NASM continúa siendo una toolchain soportada. Para una frame function NASM, la build debe generar y
validar explícitamente la metadata PE `.pdata`/`.xdata` equivalente, ya sea en el propio ensamblador o
mediante un paso de integración controlado. La ausencia de soporte automático no autoriza a omitir
unwind; si no existe un mecanismo correcto, ese fichero NASM se limita a leaf functions o se usa MASM
para la función afectada.

### 18.7 Stack probing y `__chkstk`

No se debe reducir `RSP` en una cantidad grande de una sola vez si con ello pueden saltarse guard
pages del stack. Cuando la reserva alcance el umbral de probing de la toolchain y el target, el prólogo
debe usar `__chkstk` o el mecanismo documentado equivalente antes de materializar la reserva.

El patrón de llamada, registros destruidos, umbral y metadata deben seguir la versión fijada del
toolchain; no se copia una secuencia de otra arquitectura. La reserva final debe continuar descrita por
la metadata de unwind. Para almacenamiento grande o de tamaño no confiable se debe considerar heap,
arena o `VirtualAlloc` en lugar del stack.

### 18.8 Inline assembly e intrinsics

En el target principal Windows x64 con MSVC no se depende del inline assembly tradicional, que no
está soportado para x64.

Se prefieren:

- intrinsics cuando representan claramente la operación y permiten al compilador comprenderla;
- ficheros NASM o MASM separados cuando se requiere ensamblador explícito.

El inline assembly ofrecido como extensión por otra toolchain debe quedar aislado y no formar parte
del camino común que también deba compilar con `cl`. Un intrinsic no elimina la obligación de
comprobar ISA, alineamiento, rangos y efectos laterales.

### 18.9 Direction Flag y estado de control

Una función assembly que ejecute `STD` DEBE restaurar `DF = 0` mediante `CLD` antes de retornar o
realizar cualquier llamada que espere el ABI normal. No puede existir un path de salida que deje el
Direction Flag activo.

Una función que modifique partes no volátiles de `MXCSR` o el control de la FPU debe restaurarlas antes
de retornar, salvo que el propósito documentado de la función sea precisamente cambiar ese estado.
Las instrucciones que alteren estado global del procesador deben formar parte del contrato.

### 18.10 Contrato de funciones assembly

Cada función assembly no trivial debe documentar, cuando corresponda:

- parámetros y su ubicación;
- valor de retorno;
- registros clobbered;
- registros preservados;
- stack utilizado y shadow space;
- alineamiento requerido;
- precondiciones y postcondiciones;
- si es leaf o frame function;
- ISA mínima;
- instrucciones o estado especial utilizados;
- ownership y lifetime si maneja buffers o recursos;
- requisitos y mecanismo de unwind.

El ensamblador se usa solo para una necesidad real: ABI bridge, instrucciones no expresables,
context-switch, crypto o hot path medido. No se usa para reemplazar C claro sin beneficio demostrado.

---

## 19. Testing y herramientas

### 19.1 Definition of done

Una modificación no trivial está terminada cuando:

- compila en debug y release;
- pasa clang-format;
- pasa clang-tidy y el analizador configurado;
- sus errores y cleanup paths están probados;
- no introduce warnings;
- añade tests para regresiones y límites relevantes;
- documenta ownership e invariantes no evidentes;
- cuando `icx` está disponible en una CPU Intel, la variante aplicable compila y pasa los tests
  funcionales correspondientes.

### 19.2 Sanitizers

- ASan para OOB, use-after-free y errores de heap.
- UBSan para operaciones indefinidas cubiertas por el runtime disponible.
- TSan solo en plataformas/toolchains donde esté soportado.
- Fuzzing para parsers y decoders de entrada no confiable.

La ausencia de findings no demuestra ausencia de bugs; estas herramientas complementan diseño, tests y
review.

### 19.3 Formato y análisis

Comandos típicos:

```text
clang-format --style=file -i src/module.c include/module.h
clang-format --style=file --dry-run --Werror src/module.c
clang-tidy -p build src/module.c
```

CI debe usar una versión fijada de LLVM. Un cambio de versión de clang-format o clang-tidy debe
revisarse como cambio de tooling y no mezclarse silenciosamente con una feature.

---

## 20. Preprocesador

El preprocesador se limita a:

- includes;
- selección de plataforma o feature;
- macros de atributos/compiler abstraction;
- constantes que realmente requieren preprocesador;
- helpers muy pequeños que no pueden ser funciones.

Las macros con varias sentencias usan `do { ... } while (0)`. Todo argumento debe evaluarse una sola
vez o el contrato debe prohibir explícitamente side effects.

Una macro genérica de intercambio que referencia varias veces sus argumentos no puede garantizar
evaluación única en C17 y no se usa como helper general. Se prefieren funciones `static inline`
tipadas. En este ejemplo, ambos parámetros deben apuntar a objetos `size_t` válidos:

```c
static inline void size_swap(size_t* left, size_t* right) {
  size_t temporary = *left;
  *left = *right;
  *right = temporary;
}
```

Si una macro que evalúa varias veces un argumento resulta inevitable, su contrato DEBE exigir lvalues
simples sin efectos laterales. Expresiones como `array[i++]`, `*ptr++` o llamadas a funciones están
prohibidas como argumentos de esa macro.

Las macros públicas deben llevar prefijo de proyecto.

---

## 21. Comentarios y documentación

Los comentarios explican:

- por qué existe una decisión;
- ownership y lifetime;
- invariantes;
- límites de seguridad;
- workarounds del sistema;
- referencias a formatos o especificaciones;
- resultados medidos de optimizaciones.

No deben repetir literalmente el código.

Tags admitidos:

```text
TODO(nombre): trabajo pendiente concreto.
FIXME(nombre): bug conocido.
HACK(nombre): workaround correcto pero no ideal.
SAFETY: razón por la que una operación delicada es segura.
PERF(nombre): decisión de rendimiento respaldada por medición.
NOTE(nombre): hecho no evidente.
DEPRECATED(nombre): migración y fecha/versión de retirada.
```

Un `FIXME` debe estar registrado. Un `TODO` vago como `TODO: mejorar` no es aceptable.

---

## 22. Patrones prohibidos

Salvo excepción localizada y documentada:

- VLAs;
- cuerpos de control sin llaves;
- varias declaraciones en una sentencia;
- casts para ocultar warnings;
- type punning mediante punteros incompatibles;
- unchecked allocation;
- unchecked size arithmetic;
- `realloc(ptr, 0)`;
- macros con doble evaluación;
- fallthrough oculto;
- estado global mutable sin propietario;
- `_Atomic` o layouts frágiles expuestos en una ABI pública estable;
- serialización o comparación byte a byte de structs con padding no definido;
- frame functions assembly Windows x64 sin unwind metadata válida;
- `gets`, `strcpy`, `strcat`, `sprintf`, `vsprintf`, `tmpnam`;
- format strings controladas por entrada;
- APIs `A` de Win32 para texto o paths del usuario;
- sintaxis C++ en fuentes C;
- `Windows.h` o cabeceras del SDK con grafía distinta de la forma canónica del proyecto;
- supresión global de warnings para hacer verde la build;
- optimización de bajo nivel sin benchmark.

---

## 23. Checklist de review

### Corrección

- [ ] No hay lecturas fuera de límites ni valores indeterminados.
- [ ] Las conversiones y la aritmética de tamaños están validadas.
- [ ] Todos los recursos se liberan en cada path.
- [ ] Los errores externos se manejan también en release.

### API y ownership

- [ ] Se entiende quién posee cada recurso.
- [ ] Los out-parameters solo se publican en estado válido.
- [ ] Los símbolos públicos llevan prefijo.
- [ ] Las fronteras DLL no exponen layouts frágiles.

### Windows

- [ ] Se usan APIs `W` para texto y paths.
- [ ] Los headers del SDK usan grafía canónica.
- [ ] Cada handle usa su valor inválido y destructor correctos.
- [ ] `GetLastError`/`WSAGetLastError` se captura inmediatamente.

### Concurrencia

- [ ] El estado compartido tiene lock o atomic documentado.
- [ ] No hay callbacks arbitrarios bajo lock sin contrato.
- [ ] El memory order está justificado.
- [ ] Los atomics son compatibles con todas las toolchains que acceden al mismo objeto.

### Ensamblador

- [ ] Cada función respeta registros volátiles/no volátiles, shadow space y alineamiento.
- [ ] Toda frame function tiene prólogo, epílogo y unwind metadata válidos.
- [ ] `RSP`, `DF` y el estado no volátil modificado se restauran en todos los paths.
- [ ] La ISA mínima y los clobbers están documentados.

### Tooling

- [ ] clang-format no produce cambios.
- [ ] clang-tidy y analyzer están limpios o tienen suppressions locales justificadas.
- [ ] Los tests cubren límites y errores nuevos.
- [ ] Los cambios de rendimiento incluyen medición.
- [ ] En CPU Intel, la build intenta generar `_intel` o informa claramente de que falta `icx`.
- [ ] La ausencia de `icx` no rompe la build normal.

---

## 24. Resumen visual

```c
#include "packet_parser.h"

#include <stddef.h>

static const size_t k_max_packet_size = 64u * 1024u;

// out_packet debe apuntar a almacenamiento vacío y solo se modifica en caso de éxito.
// packet_parse_impl puede dejar ownership parcial; packet_destroy acepta estado cero o parcial.
ErrorCode packet_parse(const unsigned char* data, size_t size, Packet* out_packet) {
  if (data == NULL || out_packet == NULL) {
    return ERROR_INVALID_ARGUMENT;
  }

  if (size > k_max_packet_size) {
    return ERROR_OVERFLOW;
  }

  Packet packet = {0};
  ErrorCode error = packet_parse_impl(data, size, &packet);
  if (error != ERROR_OK) {
    packet_destroy(&packet);
    return error;
  }

  *out_packet = packet;
  return ERROR_OK;
}
```

Código C bueno suele parecer aburrido. Es una virtud: los contratos son visibles, los errores tienen
camino y el lector no necesita adivinar qué intentaba hacer el autor.
