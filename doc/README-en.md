# SPEC — chili-tradutor-go

**Software version:** 2.1.20 (2026-02-01)
**Site:** https://chililinux.com
**Repository:** https://github.com/chililinux/chili-tradutor-go
**Author:** Vilmar Catafesta <vcatafesta@gmail.com>
**License/Copyright:** Copyright (C) 2019-2026 Vilmar Catafesta

---

## 1. Overview

`chili-translator-go` is a command line wrapper written in Go that automates the translation of files of different formats (scripts, documentation, structured data, man pages) into multiple languages simultaneously, using external translation engines (via `translate-shell`) and an on-disk caching system to avoid repeated retranslations.

The program is primarily designed for localizing software projects: it extracts translatable strings from source code via `gettext`/`xgettext`, generates `.po`/`.mo` files, and also directly translates documents (`.md`, `.txt`, `.json`, `.yaml`, `.html`, man pages) without going through the gettext stream.

## 2. Objectives

- Automatically translate one or more files into a configurable list of languages.
- Minimize network calls by reusing translations already made (persistent cache).
- Support both the classic gettext stream (`.po`/`.mo`, for use in `i18n` applications) and direct translation of documents and data.
- Process multiple languages in parallel, with real-time visual progress on the terminal.
- Autodetect file type (by extension or shebang) without requiring manual configuration.

## 3. Functional scope

### 3.1 Supported input formats

| Extension/criterion | Type detected | Translation flow |
|---|---|---|
| sem extensão, com shebang (`#!/usr/bin/env python`, etc.) | script (python, php, perl, ruby, javascript, shell) | gettext (`.pot`/`.po`/`.mo`) |
| no extension, no shebang | plain text | gettext |
| `.1` to `.9` | man page | line-by-line translation with roff macro protection |
| `.sh .py .php .c .cpp .go .pl .rb` | source code | gettext (`.pot`/`.po`/`.mo`) |
| `.html .htm` | HTML | line-by-line translation with tag protection |
| `.md .markdown` | Markdown | line-by-line translation with code block protection, preserving prefixes (`#`, `-`, `1.`) |
| `.txt` | plain text | line by line translation |
| `.json` | JSON | recursive translation of string values into maps |
| `.yaml .yml` | YAML | recursive translation (via JSON parser) |
| `.pot` | template gettext | copied to `pot/` and processed as PO |
| any other extension | fallback | treated as shell/gettext |

### 3.2 Execution flow (per file)

1. Checks if the file exists.
2. Detects type (`detectFileType`) and prepares corresponding output directory (`pot/`, `doc/`, `txt/`, `json/`, `yml/`, `html/`, `man/`).
3. For gettext stream: runs `xgettext` to extract strings and generates standardized POT header (`stampPotHeader`).
4. Checks if there is actual content to be translated (`hasActualContent`); if there are none, it cleans up empty artifacts and aborts the file with warning.
5. Triggers one goroutine per target language, limited by a semaphore of size `jobs` (`-j`, default 8).
6. Each goroutine calls the format-specific translation routine (`translateManPage`, `translateHTML`, `translateMarkdown`, `translatePlaintext`, `translateJSON`, or the `prepareMsginit`/`translateFile`/`writeMsgfmtToMo` trio for the gettext stream).
7. Each string/line/msgid is passed through `callUniversalTranslator`, which:
   - normalizes and queries the local cache before any network calls;
   - protects variables, formatting placeholders, links and URLs before sending to the translation engine (`protectVariables`/`restoreVariables`);
   - invoke `trans` (translate-shell) with up to 3 attempts and progressive backoff;
   - writes the result to the cache (`~/.cache/chili-tradutor-go/cache.json`).
8. Progress is displayed in real time by language using ANSI escape codes to reposition the cursor in a multi-line area of the terminal.
9. At the end of each file, it displays quick statistics (time, cache hits, network calls).
10. At the end of all files (if more than one), displays a global executive summary.

### 3.3 Cache system

- Local: `$HOME/.cache/chili-tradutor-go/cache.json`.
- Structure: `map[language]map[textoNormalizado]CacheEntry{Value, LastUsed}`.
- Loaded once at the beginning (`loadCache`) and saved once at the end of normal execution (`saveCache`, via `defer`).
- `--force` ignores existing cache entries and forces retranslation.
- `--clean-cache` removes entries not used for more than 30 days.

### 3.4 Protection of non-translatable content

The `protectVariables` function replaces with placeholders (`CHILI_REF_N_CHILI`) before sending text to the translation engine, and then restores it (`restoreVariables`):
- Variáveis de shell: `$VAR`, `${VAR}`.
- Simple formatting specifiers: `%s`, `%d` (lowercase letters only).
- Links e imagens Markdown: `[texto](url)`, `![alt](url)`.
- URLs (`http://`, `https://`).

Specific formats add their own protection before delegating to `callUniversalTranslator`:
- **Man pages:** roff macros (lines starting with `.`) have only the text after the translated macro; comments (`\"`) are preserved intact.
- **HTML:** tags (`<...>`) are replaced by placeholders (`CHILI_HTML_N_CHILI`) before line translation.
- **Markdown:** blocks delimited by ``` ``` ``` are not translated; title/list/numbering prefixes are preserved outside of translation.

### 3.5 Self-tests (`--self-test')

Runs a simplified battery of internal checks (dependencies, `protectVariables`/`restoreVariables` roundtrip) and prints an OK/FAIL report to the terminal.

### 3.6 `--self` mode

Specialized mode for extracting and translating own strings from the `chili-translator-go` binary (uses `T`/`TN` extraction keywords from the source code itself via `xgettext`).

## 4. Command Line Interface

```
chili-tradutor-go -i <arquivo> [opções]
```

| Short flag | Long flag | Description | Standard |
|---|---|---|---|
| `-i` | `--inputfile` | Source file(s) (accepts multiples, also via positional arguments) | — |
| `-l` | `--language` | List of idioms-alvo (ex: `pt_BR,en`) or `all` | `pt_BR,en,es,it,de,fr,ru,zh_CN,zh_TW,ja,ko` |
| `-e` | `--engine` | Translation engine: `google`, `bing`, `yandex` | `google` |
| `-j` | `--jobs` | Number of simultaneous translations (parallelism per language) | `8` |
| `-s` | `--source` | Source language | `auto` |
| `-f` | `--force` | Ignore cache, force new translation | `false` |
| — | `--self` | Specialized extraction for the binary itself | `false` |
| — | `--self-test` | Performs integrity self-test | `false` |
| — | `--clean-cache` | Remove cache entries not used for 30 days | `false` |
| `-q` | `--quiet` | Silent mode (partial — see limitations) | `false` |
| `-v` | `--verbose` | Verbose mode (not currently implemented) | `false` |
| `-V` | `--version` | Shows the program version | — |

Supported languages in `--other language`: `ar bg cs da de el en es et fa fi fr he hi hr hu is it ja ko nl no pl pt_PT pt_BR ro ru sk sv tr uk zh_CN zh_TW`.

## 5. External dependencies

| Binary | Package | Usage |
|---|---|---|
| `xgettext` | gettext | string extraction from source code |
| `msginit` | gettext | `.po` file initialization by language |
| `msgfmt` | gettext | compilation `.po` → `.mo` |
| `gettext` / `ngettext` | gettext | translation of the program interface itself (`T`/`TN`) |
| `trans` | translate-shell | translation execution via external engine |

The program checks the presence of these binaries at startup (`checkDependencies`) and offers automatic installation via the detected package manager (`pacman`, `xbps-install`, `apt`, `dnf`), according to the distribution identified in `/etc/os-release`.

Also checks internet connectivity at the start of execution (`checkInternet`, TCP test against `8.8.8.8:53`); if offline, the cache is still consulted, but uncached text is returned untranslated.

## 6. Generated outputs

| Entry type | Output Directory | Name pattern |
|---|---|---|
| gettext (código) | `pot/`, `usr/share/locale/<lang>/LC_MESSAGES/` | `<pot>.pot`, `<base>-<lang>.po`, `<base>.mo` |
| Man page | `man/` | `<base>-<lang>.<n>` |
| HTML | `html/` | `<base>-<lang>.html` |
| Markdown | `doc/` | `<base>-<lang>.md` |
| Simple text | `txt/` | `<base>-<lang>.txt` |
| JSON | `json/` | `<base>-<lang>.json` |
| YAML | `yml/` | `<base>-<lang>.yml` |

## 7. Terminal output

- Header with name/version, detected file type, engine, source language, number of jobs and cache path.
- Initial list of target languages with status "[Waiting...]".
- Progress bar by language, updated in-place via ANSI escape codes (`\033[nA`, `\033[K`, `\033[nB`), showing language, percentage bar and format suffix (`MD`, `TXT`, `HTML`, `MAN`, `PO`, `JSON`, `OK`).
- Quick stats per file: elapsed time, cache hits (%), network calls (%), total.
- Final executive summary (only if more than one file processed): total time, cache hits, network calls, failures (if any).
- Color usage via `github.com/fatih/color`: cyan (highlight), green (success), yellow (warning/status), red (error), blue (secondary info).

## 8. Competition

- A `sync.WaitGroup` + semaphore channel (`chan struct{}, jobs`) limit how many languages are translated simultaneously per file.
- `sync.Mutex` (`mu`) protects access to the shared cache map.
- `sync.Mutex` (`muConsole`) serializes writing to the terminal between goroutines.
- Language completed counter (`langsDone`) uses `sync/atomic`.

## 9. Known limitations (v2.1.20)

- `.yaml`/`.yml` files are deserialized with `encoding/json`, working only for YAML compatible with JSON syntax.
- `translateMap` does not traverse arrays (`[]interface{}`), only maps.
- `<script>`/`<style>` blocks in HTML and inline code snippets (`` `code` ``) in Markdown are not translation protected.
- Flag `--verbose` is present in the CLI but has no effect on the current behavior.
- `--quiet` only suppresses progress bars, not other header/summary messages.
- No support for translation engines other than those supported by `translate-shell` (`google`, `bing`, `yandex`).
- No signal handling (`SIGINT`/`SIGTERM`) for cache flush on manual interrupt.

## 10. Environment requirements

- Go 1.x (build), Linux system (use of `/etc/os-release`, `LC_ALL=C` for locale isolation in subprocesses).
- Internet access for translation (offline mode works only with pre-populated cache).
- Write permission to `$HOME/.cache/chili-tradutor-go/` and the current working directory (for `pot/`, `doc/`, `txt/`, `json/`, `yml/`, `html/`, `man/`, `usr/`).
