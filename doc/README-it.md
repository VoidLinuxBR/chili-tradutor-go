# SPEC — chili-tradutor-go

**Versione software:** 2.1.20 (01-02-2026)
**Sito:** https://chililinux.com
**Archivio:** https://github.com/chililinux/chili-tradutor-go
**Autore:** Vilmar Catafesta <vcatafesta@gmail.com>
**Licenza/Copyright:** Copyright (C) 2019-2026 Vilmar Catafesta

---

## 1. Panoramica

`chili-translator-go` è un wrapper a riga di comando scritto in Go che automatizza la traduzione di file di diversi formati (script, documentazione, dati strutturati, pagine man) in più lingue contemporaneamente, utilizzando motori di traduzione esterni (tramite `translate-shell`) e un sistema di caching su disco per evitare ripetute ritraduzioni.

Il programma è progettato principalmente per la localizzazione di progetti software: estrae stringhe traducibili dal codice sorgente tramite `gettext`/`xgettext`, genera file `.po`/`.mo` e traduce direttamente anche documenti (`.md`, `.txt`, `.json`, `.yaml`, `.html`, pagine man) senza passare attraverso il flusso gettext.

## 2. Obiettivi

- Traduci automaticamente uno o più file in un elenco configurabile di lingue.
- Minimizzare le chiamate di rete riutilizzando le traduzioni già effettuate (cache persistente).
- Supporta sia il classico flusso gettext (`.po`/`.mo`, da utilizzare nelle applicazioni "i18n") sia la traduzione diretta di documenti e dati.
- Elabora più lingue in parallelo, con avanzamento visivo in tempo reale sul terminale.
- Rilevamento automatico del tipo di file (per estensione o shebang) senza richiedere la configurazione manuale.

## 3. Ambito funzionale

### 3.1 Formati di input supportati

| Estensione/criterio | Tipo rilevato | Flusso di traduzione |
|---|---|---|
| senza estensione, come (`#!/usr/bin/env python`, ecc.) | script (python, php, perl, ruby, javascript, shell) | gettext(`.pot`/`.po`/`.mo`) |
| nessuna estensione, niente shebang | testo semplice | gettesto |
| da `.1` a `.9` | pagina man | traduzione riga per riga con protezione macro roff |
| `.sh .py .php .c .cpp .go .pl .rb` | codice sorgente | gettext(`.pot`/`.po`/`.mo`) |
| `.html .htm` | HTML | traduzione riga per riga con protezione tag |
| `.md .markdown` | Ribasso | traduzione riga per riga con protezione del blocco di codice, preservando i prefissi (`#`, `-`, `1.`) |
| `.txt` | testo semplice | traduzione riga per riga |
| `.json` | JSON | traduzione ricorsiva di valori di stringa in mappe |
| `.yaml .yml` | YAML | traduzione ricorsiva (tramite parser JSON) |
| `.pot` | modello gettesto | copiato in `pot/` ed elaborato come PO |
| qualsiasi altra estensione | ripiego | trattato come shell/gettext |

### 3.2 Flusso di esecuzione (per file)

1. Controlla se il file esiste.
2. Rileva il tipo (`detectFileType`) e prepara la directory di output corrispondente (`pot/`, `doc/`, `txt/`, `json/`, `yml/`, `html/`, `man/`).
3. Per il flusso gettext: esegue `xgettext` per estrarre le stringhe e genera un'intestazione POT standardizzata (`stampPotHeader`).
4. Controlla se c'è del contenuto effettivo da tradurre (`hasActualContent`); se non ce ne sono, ripulisce gli artefatti vuoti e interrompe il file con un avviso.
5. Attiva una goroutine per lingua di destinazione, limitata da un semaforo di dimensione `jobs` (`-j`, default 8).
6. Ogni goroutine chiama la routine di traduzione specifica del formato (`translateManPage`, `translateHTML`, `translateMarkdown`, `translatePlaintext`, `translateJSON` o il trio `prepareMsginit`/`translateFile`/`writeMsgfmtToMo` per il flusso gettext).
7. Ogni stringa/riga/msgid viene passata attraverso `callUniversalTranslator`, che:
   - normalizza e interroga la cache locale prima di qualsiasi chiamata di rete;
   - protegge le variabili, formattando segnaposti, collegamenti e URL prima dell'invio al motore di traduzione (`protectVariables`/`restoreVariables`);
   - invocare `trans` (translate-shell) con un massimo di 3 tentativi e backoff progressivo;
   - scrive il risultato nella cache (`~/.cache/chili-tradutor-go/cache.json`).
8. L'avanzamento viene visualizzato in tempo reale per lingua utilizzando i codici di escape ANSI per riposizionare il cursore in un'area multilinea del terminale.
9. Alla fine di ogni file vengono visualizzate statistiche rapide (tempo, riscontri nella cache, chiamate di rete).
10. Alla fine di tutti i file (se più di uno), viene visualizzato un riepilogo esecutivo globale.

### 3.3 Sistema de cache

- Locale: `$HOME/.cache/chili-tradutor-go/cache.json`.
- Struttura: `map[lingua]map[textoNormalizado]CacheEntry{Value, LastUsed}`.
- Caricato una volta all'inizio (`loadCache`) e salvato una volta alla fine della normale esecuzione (`saveCache`, tramite `defer`).
- "--force" ignora le voci della cache esistenti e forza la ritraduzione.
- `--clean-cache` rimuove le voci non utilizzate per più di 30 giorni.

### 3.4 Tutela dei contenuti non traducibili

La funzione `protectVariables` sostituisce con segnaposto (`CHILI_REF_N_CHILI`) prima di inviare il testo al motore di traduzione, quindi lo ripristina (`restoreVariables`):
- Varianti di shell: `$VAR`, `${VAR}`.
- Specificatori di formattazione semplici: `%s`, `%d` (solo lettere minuscole).
- Link e immagini Markdown: `[texto](url)`, `![alt](url)`.
- URL (`http://`, `https://`).

Formati specifici aggiungono la propria protezione prima di delegare a `callUniversalTranslator`:
- **Pagine man:** le macro roff (righe che iniziano con `.`) hanno solo il testo dopo la macro tradotta; i commenti (`\"`) vengono conservati intatti.
- **HTML:** i tag (`<...>`) vengono sostituiti da segnaposto (`CHILI_HTML_N_CHILI`) prima della traduzione della riga.
- **Markdown:** i blocchi delimitati da ``` ``` ``` non vengono tradotti; i prefissi titolo/elenco/numerazione vengono conservati al di fuori della traduzione.

### 3.5 Autotest (`--autotest')

Esegue una batteria semplificata di controlli interni (dipendenze, andata e ritorno `protectVariables`/`restoreVariables`) e stampa un rapporto OK/FAIL sul terminale.

### 3.6 Modalità "--self".

Modalità specializzata per estrarre e tradurre le proprie stringhe dal binario `chili-translator-go` (utilizza parole chiave di estrazione `T`/`TN` dal codice sorgente stesso tramite `xgettext`).

## 4. Interfaccia della riga di comando

```
chili-tradutor-go -i <arquivo> [opções]
```

| Bandiera corta | Bandiera lunga | Descrizione | Norma |
|---|---|---|---|
| `-i` | `--fileinput` | File sorgente (accetta multipli, anche tramite argomenti posizionali) | — |
| `-l` | `--language` | Lista de idiomas-alvo (ex: `pt_BR,en`) ou `all` | `pt_BR,en,es,it,de,fr,ru,zh_CN,zh_TW,ja,ko` |
| `-e` | `--motore` | Motore di traduzione: `google`, `bing`, `yandex` | "google" |
| `-j` | `--lavori` | Numero di traduzioni simultanee (parallelismo per lingua) | "8" |
| `-s` | `--fonte` | Lingua di partenza | "auto" |
| `-f` | `--forza` | Ignora cache, forza nuova traduzione | `falso` |
| — | `--sé` | Estrazione specializzata per il binario stesso | `falso` |
| — | `--autotest` | Esegue l'autotest dell'integrità | `falso` |
| — | `--pulisci-cache` | Rimuovi le voci della cache non utilizzate per 30 giorni | `falso` |
| `-q` | `--tranquillo` | Modalità silenziosa (parziale – vedi limitazioni) | `falso` |
| `-v` | `--verboso` | Modalità dettagliata (non attualmente implementata) | `falso` |
| `-V` | `--versione` | Mostra la versione del programma | — |

Lingue supportate in `--altra lingua`: `ar bg cs da de el en es et fa fi fr he hi hr hu is it ja ko nl no pl pt_PT pt_BR ro ru sk sv tr uk zh_CN zh_TW`.

## 5. Dipendenze esterne

| Binario | Pacchetto | Utilizzo |
|---|---|---|
| `xgettext` | gettesto | estrazione di stringhe dal codice sorgente |
| `msginit` | gettesto | Inizializzazione del file `.po` per lingua |
| `msgfmt` | gettesto | compilazione `.po` → `.mo` |
| `gettext` / `ngettext` | gettesto | traduzione dell'interfaccia del programma stesso (`T`/`TN`) |
| "trans" | tradurre-shell | esecuzione della traduzione tramite motore esterno |

Il programma verifica la presenza di questi binari all'avvio (`checkDependencies`) e offre l'installazione automatica tramite il gestore dei pacchetti rilevati (`pacman`, `xbps-install`, `apt`, `dnf`), secondo la distribuzione identificata in `/etc/os-release`.

Controlla anche la connettività Internet all'inizio dell'esecuzione (`checkInternet`, test TCP contro `8.8.8.8:53`); se offline, la cache viene comunque consultata, ma il testo non memorizzato nella cache viene restituito non tradotto.

## 6. Risultati generati

| Tipo di voce | Directory di output | Modello nome |
|---|---|---|
| gettext (codice) | `pot/`, `usr/share/locale/<lang>/LC_MESSAGES/` | `<pot>.pot`, `<base>-<lang>.po`, `<base>.mo` |
| Pagina man | `uomo/` | `<base>-<lingua>.<n>` |
| HTML | `html/` | `<base>-<lingua>.html` |
| Ribasso | `doc/` | `<base>-<lingua>.md` |
| Testo semplice | `txt/` | `<base>-<lingua>.txt` |
| JSON | `json/` | `<base>-<lingua>.json` |
| YAML | `yml/` | `<base>-<lingua>.yml` |

## 7. Uscita terminale

- Intestazione con nome/versione, tipo di file rilevato, motore, lingua di origine, numero di lavori e percorso della cache.
- Elenco iniziale delle lingue di destinazione con stato "[In attesa...]".
- Barra di avanzamento per lingua, aggiornata sul posto tramite codici escape ANSI (`\033[nA`, `\033[K`, `\033[nB`), che mostra lingua, barra percentuale e suffisso formato (`MD`, `TXT`, `HTML`, `MAN`, `PO`, `JSON`, `OK`).
- Statistiche rapide per file: tempo trascorso, riscontri nella cache (%), chiamate di rete (%), totale.
- Riepilogo esecutivo finale (solo se vengono elaborati più file): tempo totale, riscontri nella cache, chiamate di rete, errori (se presenti).
- Utilizzo del colore tramite `github.com/fatih/color`: ciano (evidenziazione), verde (successo), giallo (avviso/stato), rosso (errore), blu (informazioni secondarie).

## 8. Concorrenza

- Un canale `sync.WaitGroup` + semaforo (`chan struct{}, jobs`) limita il numero di lingue tradotte simultaneamente per file.
- `sync.Mutex` (`mu`) protegge l'accesso alla mappa della cache condivisa.
- `sync.Mutex` (`muConsole`) serializza la scrittura sul terminale tra goroutine.
- Il contatore di completamento della lingua (`langsDone`) utilizza "sync/atomic".

## 9. Limitazioni note (v2.1.20)

- I file `.yaml`/`.yml` vengono deserializzati con `encoding/json`, funzionando solo per YAML compatibile con la sintassi JSON.
- `translateMap` non attraversa gli array (`[]interface{}`), solo le mappe.
- I blocchi `<script>`/`<style>` in HTML e gli snippet di codice in linea (`` `code` ``) in Markdown non sono protetti dalla traduzione.
- Il flag "--verbose" è presente nella CLI ma non ha alcun effetto sul comportamento corrente.
- `--quiet` sopprime solo le barre di avanzamento, non altri messaggi di intestazione/riepilogo.
- Nessun supporto per motori di traduzione diversi da quelli supportati da `translate-shell` (`google`, `bing`, `yandex`).
- Nessuna gestione del segnale (`SIGINT`/`SIGTERM`) per lo svuotamento della cache in caso di interruzione manuale.

## 10. Requisiti ambientali

- Go 1.x (build), sistema Linux (uso di `/etc/os-release`, `LC_ALL=C` per l'isolamento locale nei sottoprocessi).
- Accesso a Internet per la traduzione (la modalità offline funziona solo con la cache precompilata).
- Autorizzazione di scrittura su `$HOME/.cache/chili-tradutor-go/` e sulla directory di lavoro corrente (per `pot/`, `doc/`, `txt/`, `json/`, `yml/`, `html/`, `man/`, `usr/`).
