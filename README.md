# SPEC — chili-tradutor-go

**Versão do software:** 2.1.20 (2026-02-01)
**Site:** https://chililinux.com
**Repositório:** https://github.com/chililinux/chili-tradutor-go
**Autor:** Vilmar Catafesta <vcatafesta@gmail.com>
**Licença/Copyright:** Copyright (C) 2019-2026 Vilmar Catafesta

---

## 1. Visão geral

`chili-tradutor-go` é um wrapper de linha de comando escrito em Go que automatiza a tradução de arquivos de diversos formatos (scripts, documentação, dados estruturados, man pages) para múltiplos idiomas simultaneamente, usando motores de tradução externos (via `translate-shell`) e um sistema de cache em disco para evitar retraduções repetidas.

O programa foi projetado principalmente para localizar projetos de software: extrai strings traduzíveis de código-fonte via `gettext`/`xgettext`, gera arquivos `.po`/`.mo`, e também traduz diretamente documentos (`.md`, `.txt`, `.json`, `.yaml`, `.html`, man pages) sem passar pelo fluxo gettext.

## 2. Objetivos

- Traduzir automaticamente um ou mais arquivos para uma lista configurável de idiomas.
- Minimizar chamadas de rede reutilizando traduções já feitas (cache persistente).
- Suportar tanto o fluxo clássico gettext (`.po`/`.mo`, para uso em `i18n` de aplicações) quanto tradução direta de documentos e dados.
- Processar múltiplos idiomas em paralelo, com progresso visual em tempo real no terminal.
- Autodetectar o tipo de arquivo (por extensão ou shebang) sem exigir configuração manual.

## 3. Escopo funcional

### 3.1 Formatos de entrada suportados

| Extensão/critério | Tipo detectado | Fluxo de tradução |
|---|---|---|
| sem extensão, com shebang (`#!/usr/bin/env python`, etc.) | script (python, php, perl, ruby, javascript, shell) | gettext (`.pot`/`.po`/`.mo`) |
| sem extensão, sem shebang | texto simples | gettext |
| `.1` a `.9` | man page | tradução linha a linha com proteção de macros roff |
| `.sh .py .php .c .cpp .go .pl .rb` | código-fonte | gettext (`.pot`/`.po`/`.mo`) |
| `.html .htm` | HTML | tradução linha a linha com proteção de tags |
| `.md .markdown` | Markdown | tradução linha a linha com proteção de blocos de código, preservando prefixos (`#`, `-`, `1.`) |
| `.txt` | texto simples | tradução linha a linha |
| `.json` | JSON | tradução recursiva de valores string em mapas |
| `.yaml .yml` | YAML | tradução recursiva (via parser JSON) |
| `.pot` | template gettext | copiado para `pot/` e processado como PO |
| qualquer outra extensão | fallback | tratado como shell/gettext |

### 3.2 Fluxo de execução (por arquivo)

1. Verifica se o arquivo existe.
2. Detecta tipo (`detectFileType`) e prepara diretório de saída correspondente (`pot/`, `doc/`, `txt/`, `json/`, `yml/`, `html/`, `man/`).
3. Para o fluxo gettext: executa `xgettext` para extrair strings e gera cabeçalho POT padronizado (`stampPotHeader`).
4. Verifica se há conteúdo real a traduzir (`hasActualContent`); se não houver, limpa artefatos vazios e aborta o arquivo com aviso.
5. Dispara uma goroutine por idioma-alvo, limitada por um semáforo de tamanho `jobs` (`-j`, padrão 8).
6. Cada goroutine chama a rotina de tradução específica do formato (`translateManPage`, `translateHTML`, `translateMarkdown`, `translatePlaintext`, `translateJSON`, ou o trio `prepareMsginit`/`translateFile`/`writeMsgfmtToMo` para o fluxo gettext).
7. Cada string/linha/msgid é passada por `callUniversalTranslator`, que:
   - normaliza e consulta o cache local antes de qualquer chamada de rede;
   - protege variáveis, placeholders de formatação, links e URLs antes de enviar ao motor de tradução (`protectVariables`/`restoreVariables`);
   - invoca `trans` (translate-shell) com até 3 tentativas e backoff progressivo;
   - grava o resultado no cache (`~/.cache/chili-tradutor-go/cache.json`).
8. Progresso é exibido em tempo real por idioma usando códigos de escape ANSI para reposicionar o cursor em uma área multi-linha do terminal.
9. Ao final de cada arquivo, exibe estatísticas rápidas (tempo, hits de cache, chamadas de rede).
10. Ao final de todos os arquivos (se mais de um), exibe um resumo executivo global.

### 3.3 Sistema de cache

- Local: `$HOME/.cache/chili-tradutor-go/cache.json`.
- Estrutura: `map[idioma]map[textoNormalizado]CacheEntry{Value, LastUsed}`.
- Carregado uma vez no início (`loadCache`) e salvo uma vez ao final da execução normal (`saveCache`, via `defer`).
- `--force` ignora entradas de cache existentes e força retradução.
- `--clean-cache` remove entradas não usadas há mais de 30 dias.

### 3.4 Proteção de conteúdo não traduzível

A função `protectVariables` substitui por placeholders (`CHILI_REF_N_CHILI`) antes de enviar texto ao motor de tradução, e restaura depois (`restoreVariables`):
- Variáveis de shell: `$VAR`, `${VAR}`.
- Especificadores de formatação simples: `%s`, `%d` (letras minúsculas apenas).
- Links e imagens Markdown: `[texto](url)`, `![alt](url)`.
- URLs (`http://`, `https://`).

Formatos específicos adicionam proteção própria antes de delegar para `callUniversalTranslator`:
- **Man pages:** macros roff (linhas iniciadas por `.`) têm apenas o texto após a macro traduzido; comentários (`\"`) são preservados intactos.
- **HTML:** tags (`<...>`) são substituídas por placeholders (`CHILI_HTML_N_CHILI`) antes da tradução da linha.
- **Markdown:** blocos delimitados por ``` ``` ``` não são traduzidos; prefixos de título/lista/numeração são preservados fora da tradução.

### 3.5 Auto-teste (`--self-test`)

Executa uma bateria simplificada de verificações internas (dependências, ida-e-volta de `protectVariables`/`restoreVariables`) e imprime um relatório OK/FALHA no terminal.

### 3.6 Modo `--self`

Modo especializado para extrair e traduzir as próprias strings do binário `chili-tradutor-go` (usa palavras-chave de extração `T`/`TN` do próprio código-fonte via `xgettext`).

## 4. Interface de linha de comando

```
chili-tradutor-go -i <arquivo> [opções]
```

| Flag curta | Flag longa | Descrição | Padrão |
|---|---|---|---|
| `-i` | `--inputfile` | Arquivo(s) fonte (aceita múltiplos, também via argumentos posicionais) | — |
| `-l` | `--language` | Lista de idiomas-alvo (ex: `pt_BR,en`) ou `all` | `pt_BR,en,es,it,de,fr,ru,zh_CN,zh_TW,ja,ko` |
| `-e` | `--engine` | Motor de tradução: `google`, `bing`, `yandex` | `google` |
| `-j` | `--jobs` | Número de traduções simultâneas (paralelismo por idioma) | `8` |
| `-s` | `--source` | Idioma de origem | `auto` |
| `-f` | `--force` | Ignora cache, força nova tradução | `false` |
| — | `--self` | Extração especializada para o próprio binário | `false` |
| — | `--self-test` | Executa auto-teste de integridade | `false` |
| — | `--clean-cache` | Remove entradas de cache não usadas há 30 dias | `false` |
| `-q` | `--quiet` | Modo silencioso (parcial — ver limitações) | `false` |
| `-v` | `--verbose` | Modo detalhado (não implementado atualmente) | `false` |
| `-V` | `--version` | Mostra a versão do programa | — |

Idiomas suportados em `--language all`: `ar bg cs da de el en es et fa fi fr he hi hr hu is it ja ko nl no pl pt_PT pt_BR ro ru sk sv tr uk zh_CN zh_TW`.

## 5. Dependências externas

| Binário | Pacote | Uso |
|---|---|---|
| `xgettext` | gettext | extração de strings do código-fonte |
| `msginit` | gettext | inicialização de arquivo `.po` por idioma |
| `msgfmt` | gettext | compilação `.po` → `.mo` |
| `gettext` / `ngettext` | gettext | tradução da própria interface do programa (`T`/`TN`) |
| `trans` | translate-shell | execução da tradução via motor externo |

O programa verifica a presença desses binários na inicialização (`checkDependencies`) e oferece instalação automática via gerenciador de pacotes detectado (`pacman`, `xbps-install`, `apt`, `dnf`), conforme distribuição identificada em `/etc/os-release`.

Também verifica conectividade com a internet no início da execução (`checkInternet`, teste TCP contra `8.8.8.8:53`); se offline, o cache ainda é consultado, mas textos sem entrada em cache são retornados sem tradução.

## 6. Saídas geradas

| Tipo de entrada | Diretório de saída | Padrão de nome |
|---|---|---|
| gettext (código) | `pot/`, `usr/share/locale/<lang>/LC_MESSAGES/` | `<pot>.pot`, `<base>-<lang>.po`, `<base>.mo` |
| Man page | `man/` | `<base>-<lang>.<n>` |
| HTML | `html/` | `<base>-<lang>.html` |
| Markdown | `doc/` | `<base>-<lang>.md` |
| Texto simples | `txt/` | `<base>-<lang>.txt` |
| JSON | `json/` | `<base>-<lang>.json` |
| YAML | `yml/` | `<base>-<lang>.yml` |

## 7. Saída de terminal

- Cabeçalho com nome/versão, tipo de arquivo detectado, motor, idioma de origem, número de jobs e caminho do cache.
- Lista inicial de idiomas-alvo com status "[Aguardando...]".
- Barra de progresso por idioma, atualizada in-place via escape codes ANSI (`\033[nA`, `\033[K`, `\033[nB`), mostrando idioma, barra percentual e sufixo do formato (`MD`, `TXT`, `HTML`, `MAN`, `PO`, `JSON`, `OK`).
- Estatísticas rápidas por arquivo: tempo decorrido, hits de cache (%), chamadas de rede (%), total.
- Resumo executivo final (somente se mais de um arquivo processado): tempo total, cache hits, chamadas de rede, falhas (se houver).
- Uso de cores via `github.com/fatih/color`: ciano (destaque), verde (sucesso), amarelo (aviso/status), vermelho (erro), azul (info secundária).

## 8. Concorrência

- Um `sync.WaitGroup` + canal semáforo (`chan struct{}, jobs`) limitam quantos idiomas são traduzidos simultaneamente por arquivo.
- `sync.Mutex` (`mu`) protege acesso ao mapa de cache compartilhado.
- `sync.Mutex` (`muConsole`) serializa escrita no terminal entre goroutines.
- Contador de idiomas concluídos (`langsDone`) usa `sync/atomic`.

## 9. Limitações conhecidas (v2.1.20)

- Arquivos `.yaml`/`.yml` são desserializados com `encoding/json`, funcionando apenas para YAML compatível com sintaxe JSON.
- `translateMap` não percorre arrays (`[]interface{}`), apenas mapas.
- Blocos `<script>`/`<style>` em HTML e trechos de código inline (`` `código` ``) em Markdown não são protegidos contra tradução.
- Flag `--verbose` está presente na CLI mas sem efeito no comportamento atual.
- `--quiet` suprime apenas as barras de progresso, não as demais mensagens de cabeçalho/resumo.
- Sem suporte a engines de tradução além dos aceitos pelo `translate-shell` (`google`, `bing`, `yandex`).
- Sem tratamento de sinais (`SIGINT`/`SIGTERM`) para flush do cache em interrupção manual.

## 10. Requisitos de ambiente

- Go 1.x (build), sistema Linux (uso de `/etc/os-release`, `LC_ALL=C` para isolamento de locale nos subprocessos).
- Acesso à internet para tradução (modo offline funciona apenas com cache pré-populado).
- Permissão de escrita em `$HOME/.cache/chili-tradutor-go/` e no diretório de trabalho atual (para `pot/`, `doc/`, `txt/`, `json/`, `yml/`, `html/`, `man/`, `usr/`).
