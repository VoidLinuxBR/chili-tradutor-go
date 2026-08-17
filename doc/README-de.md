# SPEC – chili-tradutor-go

**Softwareversion:** 2.1.20 (01.02.2026)
**Site:** https://chililinux.com
**Repository:** https://github.com/chililinux/chili-tradutor-go
**Autor:** Vilmar Catafesta <vcatafesta@gmail.com>
**Lizenz/Copyright:** Copyright (C) 2019-2026 Vilmar Catafesta

---

## 1. Übersicht

„chili-translator-go“ ist ein in Go geschriebener Befehlszeilen-Wrapper, der die Übersetzung von Dateien unterschiedlicher Formate (Skripte, Dokumentation, strukturierte Daten, Manpages) in mehrere Sprachen gleichzeitig automatisiert und dabei externe Übersetzungs-Engines (über „translate-shell“) und ein Caching-System auf der Festplatte verwendet, um wiederholte Neuübersetzungen zu vermeiden.

Das Programm ist in erster Linie für die Lokalisierung von Softwareprojekten konzipiert: Es extrahiert übersetzbare Zeichenfolgen aus dem Quellcode über „gettext“/„xgettext“, generiert „.po“/„.mo“-Dateien und übersetzt auch direkt Dokumente („.md“, „.txt“, „.json“, „.yaml“, „.html“, Manpages), ohne den Gettext-Stream zu durchlaufen.

## 2. Ziele

- Übersetzen Sie eine oder mehrere Dateien automatisch in eine konfigurierbare Liste von Sprachen.
- Minimieren Sie Netzwerkaufrufe, indem Sie bereits vorgenommene Übersetzungen wiederverwenden (persistenter Cache).
- Unterstützt sowohl den klassischen Gettext-Stream („.po“/„.mo“, zur Verwendung in „i18n“-Anwendungen) als auch die direkte Übersetzung von Dokumenten und Daten.
- Verarbeiten Sie mehrere Sprachen parallel, mit visuellem Fortschritt in Echtzeit auf dem Terminal.
- Automatische Erkennung des Dateityps (durch Erweiterung oder Shebang), ohne dass eine manuelle Konfiguration erforderlich ist.

## 3. Funktionsumfang

### 3.1 Unterstützte Eingabeformate

| Erweiterung/Kriterium | Typ erkannt | Übersetzungsfluss |
|---|---|---|
| ohne Erweiterung (`#!/usr/bin/env python`, etc.) | Skript (Python, PHP, Perl, Ruby, Javascript, Shell) | gettext (`.pot`/`.po`/`.mo`) |
| keine Erweiterung, kein Scherz | Klartext | gettext |
| `.1` bis `.9` | Manpage | Zeile für Zeile Übersetzung mit Roff-Makroschutz |
| `.sh .py .php .c .cpp .go .pl .rb` | Quellcode | gettext (`.pot`/`.po`/`.mo`) |
| `.html .htm` | HTML | Zeile für Zeile Übersetzung mit Tag-Schutz |
| `.md .markdown` | Abschlag | Zeile-für-Zeile-Übersetzung mit Codeblockschutz unter Beibehaltung von Präfixen (`#`, `-`, `1.`) |
| `.txt` | Klartext | Zeile für Zeile Übersetzung |
| `.json` | JSON | rekursive Übersetzung von String-Werten in Karten |
| `.yaml .yml` | YAML | rekursive Übersetzung (über JSON-Parser) |
| `.pot` | Vorlage gettext | nach „pot/“ kopiert und als PO | verarbeitet
| jede andere Erweiterung | Rückfall | wird als Shell/gettext | behandelt

### 3.2 Ausführungsablauf (pro Datei)

1. Überprüft, ob die Datei vorhanden ist.
2. Erkennt den Typ (`detectFileType`) und bereitet das entsprechende Ausgabeverzeichnis vor (`pot/`, `doc/`, `txt/`, `json/`, `yml/`, `html/`, `man/`).
3. Für gettext-Stream: Führt „xgettext“ aus, um Zeichenfolgen zu extrahieren und generiert einen standardisierten POT-Header („stampPotHeader“).
4. Prüft, ob tatsächlich zu übersetzender Inhalt vorhanden ist („hasActualContent“); Wenn keine vorhanden sind, werden leere Artefakte bereinigt und die Datei wird mit einer Warnung abgebrochen.
5. Löst eine Goroutine pro Zielsprache aus, begrenzt durch ein Semaphor der Größe „Jobs“ („-j“, Standard 8).
6. Jede Goroutine ruft die formatspezifische Übersetzungsroutine auf („translateManPage“, „translateHTML“, „translateMarkdown“, „translatePlaintext“, „translateJSON“ oder das Trio „prepareMsginit“/„translateFile“/„writeMsgfmtToMo“ für den gettext-Stream).
7. Jeder String/jede Zeile/msgid wird durch „callUniversalTranslator“ übergeben, der:
   - normalisiert und fragt den lokalen Cache vor Netzwerkaufrufen ab;
   - schützt Variablen, Formatierungsplatzhalter, Links und URLs vor dem Senden an die Übersetzungsmaschine („protectVariables“/„restoreVariables“);
   - Rufen Sie „trans“ (translate-shell) mit bis zu 3 Versuchen und progressivem Backoff auf;
   - schreibt das Ergebnis in den Cache (`~/.cache/chili-tradutor-go/cache.json`).
8. Der Fortschritt wird in Echtzeit per Sprache angezeigt, indem ANSI-Escape-Codes verwendet werden, um den Cursor in einem mehrzeiligen Bereich des Terminals neu zu positionieren.
9. Am Ende jeder Datei werden schnelle Statistiken angezeigt (Zeit, Cache-Treffer, Netzwerkaufrufe).
10. Am Ende aller Dateien (falls mehr als eine) wird eine globale Zusammenfassung angezeigt.

### 3.3 Cache-System

- Lokal: „$HOME/.cache/chili-tradutor-go/cache.json“.
- Struktur: `map[Sprache]map[textoNormalisiert]CacheEntry{Value, LastUsed}`.
- Einmal am Anfang geladen (`loadCache`) und einmal am Ende der normalen Ausführung gespeichert (`saveCache`, über `defer`).
- „--force“ ignoriert vorhandene Cache-Einträge und erzwingt eine Neuübersetzung.
- „--clean-cache“ entfernt Einträge, die länger als 30 Tage nicht verwendet wurden.

### 3.4 Schutz nicht übersetzbarer Inhalte

Die Funktion „protectVariables“ ersetzt durch Platzhalter („CHILI_REF_N_CHILI“), bevor Text an die Übersetzungsmaschine gesendet wird, und stellt ihn dann wieder her („restoreVariables“):
- Shell-Varianten: „$VAR“, „${VAR}“.
- Einfache Formatierungsspezifizierer: „%s“, „%d“ (nur Kleinbuchstaben).
- Links und Bilder Markdown: „[texto](url)“, „![alt](url)“.
- URLs (`http://`, `https://`).

Bestimmte Formate fügen ihren eigenen Schutz hinzu, bevor sie an „callUniversalTranslator“ delegieren:
- **Manpages:** Roff-Makros (Zeilen, die mit „.“ beginnen) enthalten nur den Text nach dem übersetzten Makro; Kommentare („\“`) bleiben erhalten.
- **HTML:** Tags (`<...>`) werden vor der Zeilenübersetzung durch Platzhalter (`CHILI_HTML_N_CHILI`) ersetzt.
- **Markdown:** Blöcke, die durch „````````` getrennt sind, werden nicht übersetzt; Titel-/Listen-/Nummerierungspräfixe bleiben außerhalb der Übersetzung erhalten.

### 3.5 Selbsttests („--self-test“)

Führt eine vereinfachte Reihe interner Prüfungen durch (Abhängigkeiten, „protectVariables“/„restoreVariables“-Roundtrip) und druckt einen OK/FAIL-Bericht an das Terminal.

### 3.6 „--self“-Modus

Spezialisierter Modus zum Extrahieren und Übersetzen eigener Zeichenfolgen aus der „chili-translator-go“-Binärdatei (verwendet „T“/„TN“-Extraktionsschlüsselwörter aus dem Quellcode selbst über „xgettext“).

## 4. Befehlszeilenschnittstelle

```
chili-tradutor-go -i <arquivo> [opções]
```

| Kurze Flagge | Lange Flagge | Beschreibung | Standard |
|---|---|---|---|
| `-i` | `--inputfile` | Quelldatei(en) (akzeptiert Vielfache, auch über Positionsargumente) | — |
| `-l` | `--sprache` | Liste der Redewendungen-alvo (z. B. „pt_BR,en“) oder „all“ | `pt_BR,en,es,it,de,fr,ru,zh_CN,zh_TW,ja,ko` |
| `-e` | `--engine` | Übersetzungsmaschine: „google“, „bing“, „yandex“ | `google` |
| `-j` | `--jobs` | Anzahl der Simultanübersetzungen (Parallelität pro Sprache) | `8` |
| `-s` | `--source` | Ausgangssprache | `auto` |
| `-f` | `--force` | Cache ignorieren, neue Übersetzung erzwingen | „falsch“ |
| — | `--self` | Spezialisierte Extraktion für die Binärdatei selbst | „falsch“ |
| — | `--self-test` | Führt einen Integritätsselbsttest durch | „falsch“ |
| — | `--clean-cache` | Cache-Einträge entfernen, die 30 Tage lang nicht verwendet wurden | „falsch“ |
| `-q` | `--quiet` | Stiller Modus (teilweise – siehe Einschränkungen) | „falsch“ |
| `-v` | `--verbose` | Ausführlicher Modus (derzeit nicht implementiert) | „falsch“ |
| `-V` | `--version` | Zeigt die Programmversion | an — |

Unterstützte Sprachen in „--other language“: „ar bg cs da de el en es et fa fi fr he hi hr hu is it ja ko nl no pl pt_PT pt_BR ro ru sk sv tr uk zh_CN zh_TW“.

## 5. Externe Abhängigkeiten

| Binär | Paket | Verwendung |
|---|---|---|
| `xgettext` | gettext | String-Extraktion aus Quellcode |
| `msginit` | gettext | „.po“-Dateiinitialisierung nach Sprache |
| `msgfmt` | gettext | Kompilierung „.po“ → „.mo“ |
| `gettext` / `ngettext` | gettext | Übersetzung der Programmschnittstelle selbst (`T`/`TN`) |
| `trans` | Translate-Shell | Übersetzungsausführung über externe Engine |

Das Programm prüft beim Start das Vorhandensein dieser Binärdateien („checkDependencies“) und bietet eine automatische Installation über den erkannten Paketmanager („pacman“, „xbps-install“, „apt“, „dnf“) entsprechend der in „/etc/os-release“ angegebenen Distribution an.

Überprüft außerdem die Internetkonnektivität zu Beginn der Ausführung („checkInternet“, TCP-Test gegen „8.8.8.8:53“); Wenn er offline ist, wird der Cache weiterhin abgefragt, nicht zwischengespeicherter Text wird jedoch unübersetzt zurückgegeben.

## 6. Generierte Ausgaben

| Eintragstyp | Ausgabeverzeichnis | Namensmuster |
|---|---|---|
| gettext (Code) | `pot/`, `usr/share/locale/<lang>/LC_MESSAGES/` | `<pot>.pot`, `<base>-<lang>.po`, `<base>.mo` |
| Manpage | `Mann/` | `<base>-<lang>.<n>` |
| HTML | `html/` | `<Basis>-<Sprache>.html` |
| Abschlag | `doc/` | `<base>-<lang>.md` |
| Einfacher Text | `txt/` | `<Basis>-<Sprache>.txt` |
| JSON | `json/` | `<base>-<lang>.json` |
| YAML | `yml/` | `<base>-<lang>.yml` |

## 7. Terminalausgang

- Header mit Name/Version, erkanntem Dateityp, Engine, Quellsprache, Anzahl der Jobs und Cache-Pfad.
- Erste Liste der Zielsprachen mit Status „[Warten...]“.
- Fortschrittsbalken nach Sprache, direkt über ANSI-Escape-Codes aktualisiert (`\033[nA`, `\033[K`, `\033[nB`), mit Anzeige von Sprache, Prozentbalken und Formatsuffix (`MD`, `TXT`, `HTML`, `MAN`, `PO`, `JSON`, `OK`).
- Schnellstatistiken pro Datei: verstrichene Zeit, Cache-Treffer (%), Netzwerkaufrufe (%), insgesamt.
- Abschließende Zusammenfassung (nur wenn mehr als eine Datei verarbeitet wurde): Gesamtzeit, Cache-Treffer, Netzwerkaufrufe, Fehler (falls vorhanden).
- Farbverwendung über „github.com/fatih/color“: Cyan (Hervorhebung), Grün (Erfolg), Gelb (Warnung/Status), Rot (Fehler), Blau (sekundäre Informationen).

## 8. Wettbewerb

- Ein „sync.WaitGroup“ + Semaphor-Kanal („chan struct{}, jobs“) begrenzt, wie viele Sprachen gleichzeitig pro Datei übersetzt werden.
- `sync.Mutex` (`mu`) schützt den Zugriff auf die gemeinsam genutzte Cache-Map.
- „sync.Mutex“ („muConsole“) serialisiert das Schreiben auf das Terminal zwischen Goroutinen.
- Der Zähler für die abgeschlossene Sprache („langsDone“) verwendet „sync/atomic“.

## 9. Bekannte Einschränkungen (v2.1.20)

- „.yaml“-/„.yml“-Dateien werden mit „encoding/json“ deserialisiert und funktionieren nur für YAML, das mit der JSON-Syntax kompatibel ist.
- `translateMap` durchläuft keine Arrays (`[]interface{}`), sondern nur Karten.
- `<script>`/`<style>`-Blöcke in HTML und Inline-Codefragmente („code“ ``) in Markdown sind nicht durch Übersetzung geschützt.
- Das Flag „--verbose“ ist in der CLI vorhanden, hat aber keine Auswirkung auf das aktuelle Verhalten.
- „--quiet“ unterdrückt nur Fortschrittsbalken, keine anderen Header-/Zusammenfassungsmeldungen.
- Keine Unterstützung für andere Übersetzungs-Engines als die von „translate-shell“ unterstützten („google“, „bing“, „yandex“).
- Keine Signalverarbeitung („SIGINT“/„SIGTERM“) für Cache-Flush bei manuellem Interrupt.

## 10. Umgebungsanforderungen

- Go 1.x (Build), Linux-System (Verwendung von „/etc/os-release“, „LC_ALL=C“ für die Lokalisierung von Unterprozessen).
- Internetzugang für Übersetzungen (Offline-Modus funktioniert nur mit vorgefülltem Cache).
- Schreibberechtigung für „$HOME/.cache/chili-tradutor-go/“ und das aktuelle Arbeitsverzeichnis (für „pot/“, „doc/“, „txt/“, „json/“, „yml/“, „html/“, „man/“, „usr/“.
