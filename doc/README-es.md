# SPEC — chili-tradutor-go

**Versión de software:** 2.1.20 (2026-02-01)
**Sitio:** https://chililinux.com
**Repositorio:** https://github.com/chililinux/chili-tradutor-go
**Autor:** Vilmar Catafesta <vcatafesta@gmail.com>
**Licencia/Copyright:** Copyright (C) 2019-2026 Vilmar Catafesta

---

## 1. Descripción general

`chili-translator-go` es un contenedor de línea de comandos escrito en Go que automatiza la traducción de archivos de diferentes formatos (scripts, documentación, datos estructurados, páginas de manual) a varios idiomas simultáneamente, utilizando motores de traducción externos (a través de `translate-shell`) y un sistema de almacenamiento en caché en disco para evitar retraducciones repetidas.

El programa está diseñado principalmente para localizar proyectos de software: extrae cadenas traducibles del código fuente a través de `gettext`/`xgettext`, genera archivos `.po`/`.mo` y también traduce directamente documentos (`.md`, `.txt`, `.json`, `.yaml`, `.html`, páginas man) sin pasar por el flujo de gettext.

## 2. Objetivos

- Traduce automáticamente uno o más archivos a una lista configurable de idiomas.
- Minimiza las llamadas de red reutilizando traducciones ya realizadas (caché persistente).
- Admite tanto el flujo clásico de gettext (`.po`/`.mo`, para usar en aplicaciones `i18n`) como la traducción directa de documentos y datos.
- Procese múltiples idiomas en paralelo, con progreso visual en tiempo real en el terminal.
- Detecta automáticamente el tipo de archivo (por extensión o shebang) sin necesidad de configuración manual.

## 3. Alcance funcional

### 3.1 Formatos de entrada admitidos

| Extensión/criterio | Tipo detectado | flujo traducción |
|---|---|---|
| sin extensión, con shebang (`#!/usr/bin/env python`, etc.) | script (python, php, perl, ruby, javascript, shell) | gettext (`.pot`/`.po`/`.mo`) |
| sin extensión, sin tinglados | texto plano | obtener texto |
| `.1` a `.9` | página de manual | traducción línea por línea con protección macro de roff |
| `.sh .py .php .c .cpp .go .pl .rb` | código fuente | gettext (`.pot`/`.po`/`.mo`) |
| `.html .htm` | HTML | traducción línea por línea con protección de etiquetas |
| `.md .rebaja` | Rebaja | traducción línea por línea con protección de bloque de código, preservando los prefijos (`#`, `-`, `1.`) |
| `.txt` | texto plano | línea por línea traducción |
| `.json` | JSON | traducción recursiva de valores de cadenas a mapas |
| `.yaml .yml` | YAML | traducción recursiva (a través del analizador JSON) |
| `.pot` | plantilla obtener texto | copiado a `pot/` y procesado como PO |
| cualquier otra extensión | respaldo | tratado como shell/gettext |

### 3.2 Flujo de ejecución (por archivo)

1. Comprueba si el archivo existe.
2. Detecta el tipo (`detectFileType`) y prepara el directorio de salida correspondiente (`pot/`, `doc/`, `txt/`, `json/`, `yml/`, `html/`, `man/`).
3. Para flujo gettext: ejecuta `xgettext` para extraer cadenas y genera un encabezado POT estandarizado (`stampPotHeader`).
4. Comprueba si hay contenido real para traducir (`hasActualContent`); si no hay ninguno, limpia los artefactos vacíos y cancela el archivo con una advertencia.
5. Activa una rutina por idioma de destino, limitada por un semáforo de tamaño `jobs` (`-j`, predeterminado 8).
6. Cada gorutina llama a la rutina de traducción específica del formato (`translateManPage`, `translateHTML`, `translateMarkdown`, `translatePlaintext`, `translateJSON` o el trío `prepareMsginit`/`translateFile`/`writeMsgfmtToMo` para el flujo gettext).
7. Cada cadena/línea/msgid se pasa a través de `callUniversalTranslator`, que:
   - normaliza y consulta el caché local antes de cualquier llamada de red;
   - protege las variables, formateando marcadores de posición, enlaces y URL antes de enviarlos al motor de traducción (`protectVariables`/`restoreVariables`);
   - invocar `trans` (translate-shell) con hasta 3 intentos y retroceso progresivo;
   - escribe el resultado en el caché (`~/.cache/chili-tradutor-go/cache.json`).
8. El progreso se muestra en tiempo real por idioma mediante códigos de escape ANSI para reposicionar el cursor en un área de varias líneas del terminal.
9. Al final de cada archivo, muestra estadísticas rápidas (tiempo, aciertos de caché, llamadas de red).
10. Al final de todos los archivos (si hay más de uno), se muestra un resumen ejecutivo global.

### 3.3 Sistema de caché

- Local: `$HOME/.cache/chili-tradutor-go/cache.json`.
- Estructura: `mapa[idioma]mapa[textoNormalizado]CacheEntry{Valor, ÚltimoUsado}`.
- Se carga una vez al principio (`loadCache`) y se guarda una vez al final de la ejecución normal (`saveCache`, mediante `defer`).
- `--force` ignora las entradas de caché existentes y fuerza la retraducción.
- `--clean-cache` elimina las entradas que no se utilizan durante más de 30 días.

### 3.4 Protección de contenidos no traducibles

La función `protectVariables` reemplaza con marcadores de posición (`CHILI_REF_N_CHILI`) antes de enviar el texto al motor de traducción y luego lo restaura (`restoreVariables`):
- Variantes de shell: `$VAR`, `${VAR}`.
- Especificadores de formato simples: `%s`, `%d` (solo letras minúsculas).
- Enlaces e imágenes Markdown: `[texto](url)`, `![alt](url)`.
- URL (`http://`, `https://`).

Los formatos específicos agregan su propia protección antes de delegar a `callUniversalTranslator`:
- **Páginas de manual:** las macros de roff (líneas que comienzan con `.`) tienen solo el texto después de la macro traducida; Los comentarios (`\"`) se conservan intactos.
- **HTML:** las etiquetas (`<...>`) se reemplazan por marcadores de posición (`CHILI_HTML_N_CHILI`) antes de la traducción de línea.
- **Markdown:** los bloques delimitados por ``` ``` ``` no se traducen; Los prefijos de título/lista/numeración se conservan fuera de la traducción.

### 3.5 Autopruebas (`--self-test')

Ejecuta una batería simplificada de comprobaciones internas (dependencias, `protectVariables`/`restoreVariables` ida y vuelta) e imprime un informe OK/FAIL en el terminal.

### 3.6 modo `--self`

Modo especializado para extraer y traducir cadenas propias del binario `chili-translator-go` (usa palabras clave de extracción `T`/`TN` del propio código fuente a través de `xgettext`).

## 4. Interfaz de línea de comando

```
chili-tradutor-go -i <arquivo> [opções]
```

| Bandera corta | Bandera larga | Descripción | Estándar |
|---|---|---|---|
| `-yo` | `--archivo de entrada` | Archivo(s) fuente (acepta múltiplos, también mediante argumentos posicionales) | — |
| `-l` | `--idioma` | Lista de modismos-alvo (ej: `pt_BR,en`) o `all` | `pt_BR,en,es,it,de,fr,ru,zh_CN,zh_TW,ja,ko` |
| `-e` | `--motor` | Motor de traducción: `google`, `bing`, `yandex` | `google` |
| `-j` | `--trabajos` | Número de traducciones simultáneas (paralelismo por lengua) | `8` |
| `-s` | `--fuente` | Idioma de origen | `auto` |
| `-f` | `--fuerza` | ignorar caché, forzar nueva traducción | `falso` |
| — | `--yo` | Extracción especializada para el propio binario | `falso` |
| — | `--autoprueba` | Realiza autoprueba de integridad | `falso` |
| — | `--clean-cache` | Eliminar las entradas de caché que no se utilizan durante 30 días | `falso` |
| `-q` | `--tranquilo` | Modo silencioso (parcial - ver limitaciones) | `falso` |
| `-v` | `--detallado` | Modo detallado (no implementado actualmente) | `falso` |
| `-V` | `--versión` | Muestra la versión del programa | — |

Idiomas admitidos en `--other language`: `ar bg cs da de el en es et fa fi fr he hi hr hu is it ja ko nl no pl pt_PT pt_BR ro ru sk sv tr uk zh_CN zh_TW`.

## 5. Dependencias externas

| Binario | Paquete | Uso |
|---|---|---|
| `xgettext` | obtener texto | extracción de cadenas del código fuente |
| `msginit` | obtener texto | Inicialización de archivo `.po` por idioma |
| `msgfmt` | obtener texto | compilación `.po` → `.mo` |
| `gettext` / `ngettext` | obtener texto | traducción de la propia interfaz del programa (`T`/`TN`) |
| `trans` | traducir-shell | ejecución de traducciones mediante motor externo |

El programa comprueba la presencia de estos binarios al inicio (`checkDependencies`) y ofrece instalación automática a través del administrador de paquetes detectados (`pacman`, `xbps-install`, `apt`, `dnf`), según la distribución identificada en `/etc/os-release`.

También verifica la conectividad a Internet al inicio de la ejecución (`checkInternet`, prueba TCP contra `8.8.8.8:53`); si está fuera de línea, se sigue consultando el caché, pero el texto no almacenado en caché se devuelve sin traducir.

## 6. Productos generados

| Tipo de entrada | Directorio de salida | Patrón de nombre |
|---|---|---|
| gettext (código) | `pot/`, `usr/share/locale/<idioma>/LC_MESSAGES/` | `<pot>.pot`, `<base>-<lang>.po`, `<base>.mo` |
| Página de manual | `hombre/` | `<base>-<idioma>.<n>` |
| HTML | `html/` | `<base>-<idioma>.html` |
| Rebaja | `doc/` | `<base>-<idioma>.md` |
| Texto sencillo | `txt/` | `<base>-<idioma>.txt` |
| JSON | `json/` | `<base>-<idioma>.json` |
| YAML | `yml/` | `<base>-<idioma>.yml` |

## 7. Salida terminal

- Encabezado con nombre/versión, tipo de archivo detectado, motor, idioma de origen, número de trabajos y ruta de caché.
- Lista inicial de idiomas de destino con estado "[En espera...]".
- Barra de progreso por idioma, actualizada in situ mediante códigos de escape ANSI (`\033[nA`, `\033[K`, `\033[nB`), que muestra el idioma, la barra de porcentaje y el sufijo de formato (`MD`, `TXT`, `HTML`, `MAN`, `PO`, `JSON`, `OK`).
- Estadísticas rápidas por archivo: tiempo transcurrido, aciertos de caché (%), llamadas de red (%), total.
- Resumen ejecutivo final (solo si se procesó más de un archivo): tiempo total, aciertos de caché, llamadas de red, fallas (si las hubiera).
- Uso del color a través de `github.com/fatih/color`: cian (resaltado), verde (éxito), amarillo (advertencia/estado), rojo (error), azul (información secundaria).

## 8. Competencia

- Un `sync.WaitGroup` + canal de semáforo (`chan struct{}, jobs`) limita cuántos idiomas se traducen simultáneamente por archivo.
- `sync.Mutex` (`mu`) protege el acceso al mapa de caché compartido.
- `sync.Mutex` (`muConsole`) serializa la escritura en el terminal entre gorutinas.
- El contador de idioma completado (`langsDone`) usa `sync/atomic`.

## 9. Limitaciones conocidas (v2.1.20)

- Los archivos `.yaml`/`.yml` se deserializan con `encoding/json` y funcionan solo para YAML compatible con la sintaxis JSON.
- `translateMap` no atraviesa matrices (`[]interface{}`), solo mapas.
- Los bloques `<script>`/`<style>` en HTML y los fragmentos de código en línea (`` `code` ``) en Markdown no están protegidos contra la traducción.
- El indicador `--verbose` está presente en la CLI pero no tiene ningún efecto en el comportamiento actual.
- `--quiet` solo suprime las barras de progreso, no otros mensajes de encabezado/resumen.
- No hay soporte para motores de traducción distintos de los soportados por `translate-shell` (`google`, `bing`, `yandex`).
- Sin manejo de señales (`SIGINT`/`SIGTERM`) para vaciado de caché en interrupción manual.

## 10. Requisitos de ambiente

- Vaya a 1.x (compilación), sistema Linux (uso de `/etc/os-release`, `LC_ALL=C` para aislamiento local en subprocesos).
- Acceso a Internet para traducción (el modo fuera de línea funciona solo con caché previamente completado).
- Permiso de escritura para `$HOME/.cache/chili-tradutor-go/` y el directorio de trabajo actual (para `pot/`, `doc/`, `txt/`, `json/`, `yml/`, `html/`, `man/`, `usr/`).
