# SPEC — chili-tradutor-go

**Version du logiciel :** 2.1.20 (01/02/2026)
**Site :** https://chililinux.com
**Dépôt :** https://github.com/chililinux/chili-tradutor-go
**Auteur :** Vilmar Catafesta <vcatafesta@gmail.com>
**Licence/Copyright :** Copyright (C) 2019-2026 Vilmar Catafesta

---

## 1. Aperçu

`chili-translator-go` est un wrapper de ligne de commande écrit en Go qui automatise la traduction de fichiers de différents formats (scripts, documentation, données structurées, pages de manuel) dans plusieurs langues simultanément, en utilisant des moteurs de traduction externes (via `translate-shell`) et un système de mise en cache sur disque pour éviter les retraductions répétées.

Le programme est principalement conçu pour localiser des projets logiciels : il extrait les chaînes traduisibles du code source via `gettext`/`xgettext`, génère des fichiers `.po`/`.mo`, et traduit également directement les documents (`.md`, `.txt`, `.json`, `.yaml`, `.html`, pages de manuel) sans passer par le flux gettext.

## 2. Objectifs

- Traduisez automatiquement un ou plusieurs fichiers dans une liste configurable de langues.
- Minimisez les appels réseau en réutilisant les traductions déjà effectuées (cache persistant).
- Prend en charge à la fois le flux gettext classique (`.po`/`.mo`, pour une utilisation dans les applications `i18n`) et la traduction directe de documents et de données.
- Traitez plusieurs langues en parallèle, avec une progression visuelle en temps réel sur le terminal.
- Détection automatique du type de fichier (par extension ou shebang) sans nécessiter de configuration manuelle.

## 3. Portée fonctionnelle

### 3.1 Formats d'entrée pris en charge

| Extension/critère | Type détecté | Flux de traduction |
|---|---|---|
| sans extension, avec shebang (`#!/usr/bin/env python`, etc.) | script (python, php, perl, ruby, javascript, shell) | gettext (`.pot`/`.po`/`.mo`) |
| pas d'extension, pas de shebang | texte brut | obtenir le texte |
| `.1` à `.9` | page de manuel | traduction ligne par ligne avec protection macro roff |
| `.sh .py .php .c .cpp .go .pl .rb` | code source | gettext (`.pot`/`.po`/`.mo`) |
| `.html .htm` | HTML | traduction ligne par ligne avec protection par balise |
| `.md .markdown` | Démarquage | traduction ligne par ligne avec protection des blocs de code, préservant les préfixes (`#`, `-`, `1.`) |
| `.txt` | texte brut | Traduction ligne par ligne |
| `.json` | JSON | traduction récursive des valeurs de chaîne en cartes |
| `.yaml .yml` | YAML | Traduction récursive (via l'analyseur JSON) |
| `.pot` | modèle gettext | copié dans `pot/` et traité comme PO |
| toute autre extension | repli | traité comme shell/gettext |

### 3.2 Flux d'exécution (par fichier)

1. Vérifie si le fichier existe.
2. Détecte le type (`detectFileType`) et prépare le répertoire de sortie correspondant (`pot/`, `doc/`, `txt/`, `json/`, `yml/`, `html/`, `man/`).
3. Pour le flux gettext : exécute `xgettext` pour extraire les chaînes et génère un en-tête POT standardisé (`stampPotHeader`).
4. Vérifie s'il y a du contenu réel à traduire (`hasActualContent`) ; s'il n'y en a pas, il nettoie les artefacts vides et abandonne le fichier avec avertissement.
5. Déclenche une goroutine par langue cible, limitée par un sémaphore de taille `jobs` (`-j`, par défaut 8).
6. Chaque goroutine appelle la routine de traduction spécifique au format (`translateManPage`, `translateHTML`, `translateMarkdown`, `translatePlaintext`, `translateJSON` ou le trio `prepareMsginit`/`translateFile`/`writeMsgfmtToMo` pour le flux gettext).
7. Chaque chaîne/ligne/msgid est transmis via `callUniversalTranslator`, qui :
   - normalise et interroge le cache local avant tout appel réseau ;
   - protège les variables, formatant les espaces réservés, les liens et les URL avant de les envoyer au moteur de traduction (`protectVariables`/`restoreVariables`) ;
   - invoquez « trans » (translate-shell) avec jusqu'à 3 tentatives et une interruption progressive ;
   - écrit le résultat dans le cache (`~/.cache/chili-tradutor-go/cache.json`).
8. La progression est affichée en temps réel par langue à l'aide de codes d'échappement ANSI pour repositionner le curseur dans une zone multiligne du terminal.
9. À la fin de chaque fichier, il affiche des statistiques rapides (durée, accès au cache, appels réseau).
10. À la fin de tous les fichiers (s'il y en a plusieurs), affiche un résumé global.

### 3.3 Système de cache

- Local : `$HOME/.cache/chili-tradutor-go/cache.json`.
- Structure : `map[langue]map[textoNormalizado]CacheEntry{Value, LastUsed}`.
- Chargé une fois au début (`loadCache`) et enregistré une fois à la fin de l'exécution normale (`saveCache`, via `defer`).
- `--force` ignore les entrées de cache existantes et force la retraduction.
- `--clean-cache` supprime les entrées non utilisées depuis plus de 30 jours.

### 3.4 Protection des contenus non traduisibles

La fonction `protectVariables` remplace par des espaces réservés (`CHILI_REF_N_CHILI`) avant d'envoyer le texte au moteur de traduction, puis le restaure (`restoreVariables`) :
- Variantes du shell : `$VAR`, `${VAR}`.
- Spécificateurs de formatage simples : `%s`, `%d` (lettres minuscules uniquement).
- Liens et images Markdown : `[texto](url)`, `![alt](url)`.
- URL (`http://`, `https://`).

Des formats spécifiques ajoutent leur propre protection avant de déléguer à `callUniversalTranslator` :
- **Pages de manuel :** les macros roff (lignes commençant par `.`) n'ont que le texte après la macro traduite ; les commentaires (`\"`) sont conservés intacts.
- **HTML :** les balises (`<...>`) sont remplacées par des espaces réservés (`CHILI_HTML_N_CHILI`) avant la traduction de la ligne.
- **Markdown :** les blocs délimités par ``` ``` ``` ne sont pas traduits ; les préfixes de titre/liste/numérotation sont conservés en dehors de la traduction.

### 3.5 Autotests («--autotest»)

Exécute une batterie simplifiée de vérifications internes (dépendances, aller-retour `protectVariables`/`restoreVariables`) et imprime un rapport OK/FAIL sur le terminal.

### 3.6 Mode `--soi`

Mode spécialisé pour extraire et traduire ses propres chaînes à partir du binaire `chili-translator-go` (utilise les mots-clés d'extraction `T`/`TN` du code source lui-même via `xgettext`).

## 4. Interface de ligne de commande

```
chili-tradutor-go -i <arquivo> [opções]
```

| Drapeau court | Drapeau long | Descriptif | Norme |
|---|---|---|---|
| `-je` | `--fichier d'entrée` | Fichier(s) source (accepte les multiples, également via des arguments de position) | — |
| `-l` | `--langue` | Liste des idiomes-alvo (ex : `pt_BR,en`) ou `all` | `pt_BR,en,es,it,de,fr,ru,zh_CN,zh_TW,ja,ko` |
| `-e` | `--moteur` | Moteur de traduction : `google`, `bing`, `yandex` | `google` |
| `-j` | `--emplois` | Nombre de traductions simultanées (parallélisme par langue) | '8' |
| `-s` | `--source` | Langue source | `auto` |
| `-f` | `--force` | Traduction ignorer le cache, forcer le nouveau | `faux` |
| — | `--soi` | Extraction spécialisée pour le binaire lui-même | `faux` |
| — | `--auto-test` | Effectue un auto-test d'intégrité | `faux` |
| — | `--clean-cache` | Supprimer les entrées de cache non utilisées depuis 30 jours | `faux` |
| `-q` | `--calme` | Mode silencieux (partiel — voir limitations) | `faux` |
| `-v` | `--verbeux` | Mode verbeux (non implémenté actuellement) | `faux` |
| `-V` | `--version` | Affiche la version du programme | — |

Langues prises en charge dans `--autre langue` : `ar bg cs da de el en es et fa fi fr he hi hr hu is it ja ko nl no pl pt_PT pt_BR ro ru sk sv tr uk zh_CN zh_TW`.

## 5. Dépendances externes

| Binaire | Forfait | Utilisation |
|---|---|---|
| `xgettext` | obtenir le texte | extraction de chaîne du code source |
| `msginit` | obtenir le texte | Initialisation du fichier `.po` par langue |
| `msgfmt` | obtenir le texte | compilation `.po` → `.mo` |
| `gettext` / `ngettext` | obtenir le texte | traduction de l'interface du programme elle-même (`T`/`TN`) |
| `trans` | traduire-shell | exécution de la traduction via un moteur externe |

Le programme vérifie la présence de ces binaires au démarrage (`checkDependencies`) et propose une installation automatique via le gestionnaire de paquets détecté (`pacman`, `xbps-install`, `apt`, `dnf`), selon la distribution identifiée dans `/etc/os-release`.

Vérifie également la connectivité Internet au début de l'exécution (`checkInternet`, test TCP par rapport à `8.8.8.8:53`) ; en cas de connexion hors ligne, le cache est toujours consulté, mais le texte non mis en cache est renvoyé non traduit.

## 6. Résultats générés

| Type d'entrée | Répertoire de sortie | Modèle de nom |
|---|---|---|
| gettext (code) | `pot/`, `usr/share/locale/<lang>/LC_MESSAGES/` | `<pot>.pot`, `<base>-<lang>.po`, `<base>.mo` |
| Page de manuel | `homme/` | `<base>-<lang>.<n>` |
| HTML | `html/` | `<base>-<lang>.html` |
| Démarquage | `doc/` | `<base>-<lang>.md` |
| Texte simple | `txt/` | `<base>-<lang>.txt` |
| JSON | `json/` | `<base>-<lang>.json` |
| YAML | `yml/` | `<base>-<lang>.yml` |

## 7. Sortie terminale

- En-tête avec nom/version, type de fichier détecté, moteur, langue source, nombre de tâches et chemin du cache.
- Liste initiale des langues cibles avec le statut "[En attente...]".
- Barre de progression par langue, mise à jour sur place via les codes d'échappement ANSI (`\033[nA`, `\033[K`, `\033[nB`), affichant la langue, la barre de pourcentage et le suffixe de format (`MD`, `TXT`, `HTML`, `MAN`, `PO`, `JSON`, `OK`).
- Statistiques rapides par fichier : temps écoulé, accès au cache (%), appels réseau (%), total.
- Résumé final (uniquement si plusieurs fichiers sont traités) : durée totale, accès au cache, appels réseau, échecs (le cas échéant).
- Utilisation des couleurs via `github.com/fatih/color` : cyan (surbrillance), vert (succès), jaune (avertissement/statut), rouge (erreur), bleu (informations secondaires).

## 8. Concurrence

- Un canal `sync.WaitGroup` + sémaphore (`chan struct{}, jobs`) limite le nombre de langues traduites simultanément par fichier.
- `sync.Mutex` (`mu`) protège l'accès à la carte de cache partagée.
- `sync.Mutex` (`muConsole`) sérialise l'écriture sur le terminal entre les goroutines.
- Le compteur de langage terminé (`langsDone`) utilise `sync/atomic`.

## 9. Limitations connues (v2.1.20)

- Les fichiers `.yaml`/`.yml` sont désérialisés avec `encoding/json`, fonctionnant uniquement pour YAML compatible avec la syntaxe JSON.
- `translateMap` ne traverse pas les tableaux (`[]interface{}`), uniquement les cartes.
- Les blocs `<script>`/`<style>` en HTML et les extraits de code en ligne (`` `code` ``) dans Markdown ne sont pas protégés contre la traduction.
- L'indicateur `--verbose` est présent dans la CLI mais n'a aucun effet sur le comportement actuel.
- `--quiet` supprime uniquement les barres de progression, pas les autres messages d'en-tête/résumé.
- Aucune prise en charge des moteurs de traduction autres que ceux pris en charge par `translate-shell` (`google`, `bing`, `yandex`).
- Aucune gestion du signal (`SIGINT`/`SIGTERM`) pour le vidage du cache lors d'une interruption manuelle.

## 10. Exigences environnementales

- Passez à 1.x (build), système Linux (utilisation de `/etc/os-release`, `LC_ALL=C` pour l'isolation des paramètres régionaux dans les sous-processus).
- Accès Internet pour la traduction (le mode hors ligne fonctionne uniquement avec un cache pré-rempli).
- Autorisation d'écriture sur `$HOME/.cache/chili-tradutor-go/` et sur le répertoire de travail actuel (pour `pot/`, `doc/`, `txt/`, `json/`, `yml/`, `html/`, `man/`, `usr/`).
