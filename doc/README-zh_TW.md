# SPEC — 辣椒-tradutor-go

**軟體版本：** 2.1.20 (2026-02-01)
**網址：** https://chililinux.com
**儲存庫：** https://github.com/chililinux/chili-tradutor-go
**作者：** Vilmar Catafesta <vcatafesta@gmail.com>
**許可證/版權：** 版權所有 (C) 2019-2026 Vilmar Catafesta

---

## 1. 概述

`chili-translator-go` 是一個用 Go 編寫的命令列包裝器，它使用外部翻譯引擎（透過 `translate-shell`）和磁碟快取系統，自動將不同格式的檔案（腳本、文件、結構化資料、手冊頁）同時翻譯成多種語言，以避免重複重新翻譯。

這個程式主要用於在地化軟體專案：它透過「gettext」/「xgettext」從原始程式碼中提取可翻譯的字串，產生「.po」/「.mo」文件，並且還可以直接翻譯文件（「.md」、「.txt」、「.json」、「.yaml」、「.html」、手冊頁），而無需透過 gettext 串流。

## 2. 目標

- 自動將一個或多個檔案翻譯成可設定的語言清單。
- 透過重複使用已經進行的翻譯（持久性快取）來最大限度地減少網路呼叫。
- 支援經典的 gettext 流（`.po`/`.mo`，用於 `i18n` 應用程式）以及文件和資料的直接翻譯。
- 並行處理多種語言，並在終端機上即時顯示進度。
- 自動偵測檔案類型（透過副檔名或 shebang），無需手動設定。

## 3. 功能範圍

### 3.1 支援的輸入格式

|擴充/標準 |偵測到型別 |翻譯流程|
|---|---|---|
| sem extensão、com shebang（`#!/usr/bin/env python` 等）|腳本（python、php、perl、ruby、javascript、shell）| gettext(`.pot`/`.po`/`.mo`) |
|沒有擴展，沒有shebang |純文字|取得文字 |
| `.1` 到 `.9` |手冊頁 |具有 roff 巨集保護的逐行翻譯 |
| `.sh .py .php .c .cpp .go .pl .rb` |原始碼 | gettext(`.pot`/`.po`/`.mo`) |
| `.html .htm` | HTML |帶有標籤保護的逐行翻譯 |
| `.md .markdown` |降價|具有程式碼區塊保護的逐行翻譯，保留前綴（`#`、`-`、`1.`） |
| `.txt` |純文字|逐行翻譯|
| `.json` | JSON |將字串值遞歸翻譯為映射 |
| `.yaml.yml` | yaml |遞歸翻譯（透過 JSON 解析器）|
| `.pot` |範本 gettext |複製到 `pot/` 並作為 PO | 處理
|任何其他擴充 |後備|被視為 shell/gettext |

### 3.2 執行流程（每個文件）

1. 檢查文件是否存在。
2. 偵測類型（`detectFileType`）並準備對應的輸出目錄（`pot/`、`doc/`、`txt/`、`json/`、`yml/`、`html/`、`man/`）。
3. 對於 gettext 流：運行 `xgettext` 來提取字串並產生標準化的 POT 標頭 (`stampPotHeader`)。
4. 檢查是否有實際要翻譯的內容（`hasActualContent`）；如果沒有，它會清除空工件併中止文件並發出警告。
5. 為每種目標語言觸發一個 goroutine，受大小為“jobs”的信號量（“-j”，預設為 8）限制。
6. 每個 goroutine 都會呼叫特定於格式的翻譯程式（`translateManPage`、`translateHTML`、`translateMarkdown`、`translatePlaintext`、`translateJSON` 或 gettext 流的 `prepareMsginit`/`translateFile`/`writeMsmtgfToMo` 三重奏）。
7. 每個字串/行/msgid 都透過「callUniversalTranslator」傳遞，其中：
   - 在任何網路呼叫之前標準化並查詢本地快取；
   - 在傳送到翻譯引擎之前保護變數、格式化佔位符、連結和 URL（“protectVariables”/“restoreVariables”）；
   - 呼叫「trans」（translate-shell），最多嘗試 3 次並進行漸進式退避；
   - 
8. 使用 ANSI 轉義碼將遊標重新定位在終端機的多行區域中，並按語言即時顯示進度。
9. 在每個文件的末尾，它顯示快速統計資訊（時間、快取命中、網路呼叫）。
10. 在所有文件（如果多個文件）的末尾，顯示全域執行摘要。

### 3.3 快取系統

- 本機：`$HOME/.cache/chili-tradutor-go/cache.json`。
- 結構：`map[語言]map[textoNormalizado]CacheEntry{Value, LastUsed}`。
- 在開始時載入一次（“loadCache”）並在正常執行結束時儲存一次（“saveCache”，透過“defer”）。
- `--force` 忽略現有的快取條目並強制重新翻譯。
- `--clean-cache` 刪除超過 30 天未使用的項目。

### 3.4 保護不可翻譯內容

`protectVariables` 函數在將文字傳送到翻譯引擎之前替換為佔位符 (`CHILI_REF_N_CHILI`)，然後恢復它 (`restoreVariables`)：
- shell 變數：`$VAR`、`${VAR}`。
- 簡單格式說明符：「%s」、「%d」（僅限小寫字母）。
- Markdown 影像連結：`[texto](url)`、`![alt](url)`。
- 網址 (`http://`, `https://`).

特定格式在委託給「callUniversalTranslator」之前加入自己的保護：
- **手冊頁：** roff 巨集（以「.」開頭的行）僅包含翻譯後的巨集之後的文字；註解（`\"`）原樣保留。
- **HTML：** 標籤 (`<...>`) 在行翻譯之前被佔位符 (`CHILI_HTML_N_CHILI`) 取代。
- **Markdown:** 由 ``` ``` ``` 分隔的區塊不會被翻譯；標題/列表/編號前綴在翻譯之外保留。

### 3.5 自測（`--自測'）

執行一系列簡化的內部檢查（依賴項、「protectVariables」/「restoreVariables」往返）並向終端列印 OK/FAIL 報告。

### 3.6 `--self`模式

用於從“chili-translator-go”二進位檔案中提取和翻譯自己的字串的專用模式（透過“xgettext”從原始程式碼本身中使用“T”/“TN”提取關鍵字）。

## 4. 命令列介面

```
chili-tradutor-go -i <arquivo> [opções]
```

|短旗|長旗|描述 |標準|
|---|---|---|---|
| `-i` | `--輸入檔案` |原始檔（接受多個，也透過位置參數）| — |
| `-l` | `--語言` |習語列表-alvo（例如：`pt_BR,en`）或`all` | `pt_BR，en，es，it，de，fr，ru，zh_CN，zh_TW，ja，ko` |
| `-e` | `--引擎` |翻譯引擎：`google`、`bing`、`yandex` | `Google` |
| `-j` | `--工作` |同聲傳譯數量（每種語言的並行度）| `8` |
| `-s` | `--來源` |來源語言 | `自動` |
| `-f` | `--force` |忽略緩存，強制新翻譯 | `假` |
| — | `--自我` |針對二進位檔案本身的專門提取 | `假` |
| — | `--自測` |執行完整性自我檢測 | `假` |
| — | `--clean-cache` |刪除 30 天內未使用的快取項目 | `假` |
| `-q` | `--安靜` |靜音模式（部分 - 請參閱限制）| `假` |
| `-v` | `--詳細` |詳細模式（目前未實現）| `假` |
| `-V` | `--版本` |顯示程式版本 | — |

`--other language`中支援的語言：`ar bg cs da de el en es et fa fi fr he hi hr hu is it ja ko nl no pl pt_PT pt_BR ro ru sk sv tr uk zh_CN zh_TW`。

## 5. 外部依賴

|二進位|套餐 |用途 |
|---|---|---|
| `xgettext` |取得文字 |從原始碼中提取字串 |
| `msginit` |取得文字 |按語言初始化 `.po` 檔案 |
| `msgfmt` |取得文字 |編譯 `.po` → `.mo` |
| `gettext` / `ngettext` |取得文字 |程式介面本身的翻譯 (`T`/`TN`) |
| `反式` |翻譯外殼 |透過外部引擎執行翻譯 |

程式在啟動時檢查這些二進位檔案是否存在（「checkDependency」），並根據「/etc/os-release」中標識的發行版，透過偵測到的套件管理器（「pacman」、「xbps-install」、「apt」、「dnf」）提供自動安裝。

在執行開始時也檢查網際網路連線（“checkInternet”，針對「8.8.8.8:53」的 TCP 測試）；如果離線，仍會查閱緩存，但未快取的文字會以未翻譯的形式傳回。

## 6. 產生的輸出

|條目類型|輸出目錄|姓名圖案|
|---|---|---|
| gettext（código）| `pot/`、`usr/share/locale/<lang>/LC_MESSAGES/` | `<pot>.pot`、`<base>-<lang>.po`、`<base>.mo` |
|手冊頁 | `男人/` | `<base>-<lang>.<n>` |
| HTML | `html/` | `<base>-<lang>.html` |
|降價| `文檔/` | `<base>-<lang>.md` |
|簡單文字| `txt/` | `<base>-<lang>.txt` |
| JSON | `json/` | `<base>-<lang>.json` |
| yaml | `yml/` | `<base>-<lang>.yml` |

## 7. 端子輸出

- 標頭包含名稱/版本、偵測到的檔案類型、引擎、來源語言、作業數量和快取路徑。
- 狀態為「[等待...]」的目標語言初始清單。
- 依語言顯示的進度條，透過 ANSI 轉義碼（`\033[nA`、`\033[K`、`\033[nB`）就地更新，顯示語言、百分比條和格式後綴（`MD`、`TXT`、`HTML`、`MAN`、`PO`、`JSON`、`OK`）。
- 每個檔案的快速統計資料：已使用時間、快取命中 (%)、網路呼叫 (%)、總計。
- 最終執行摘要（僅處理多個檔案時）：總時間、快取命中、網路呼叫、失敗（如果有）。
- 透過`github.com/fatih/color`使用顏色：青色（突出顯示）、綠色（成功）、黃色（警告/狀態）、紅色（錯誤）、藍色（輔助資訊）。

## 8. 競爭

- `sync.WaitGroup` + 信號量通道（`chan struct{}, jobs`）限制每個檔案同時翻譯的語言數量。
- `sync.Mutex` (`mu`) 保護對共享快取映射的存取。
- `sync.Mutex` (`muConsole`) 在 goroutine 之間序列化對終端的寫入。
- 語言完成計數器（“langsDone”）使用“sync/atomic”。

## 9.已知限制 (v2.1.20)

- `.yaml`/`.yml` 檔案使用 `encoding/json` 進行反序列化，僅適用於與 JSON 語法相容的 YAML。
- `translateMap` 不遍歷陣列（`[]interface{}`），僅遍歷映射。
- HTML 中的 `<script>`/`<style>` 區塊和 Markdown 中的內聯程式碼片段 (`` `code` ``) 不受翻譯保護。
- CLI 中存在標誌“--verbose”，但對當前行為沒有影響。
- `--quiet` 僅抑制進度條，而不抑制其他標題/摘要訊息。
- 除「translate-shell」支援的翻譯引擎（「google」、「bing」、「yandex」）之外，不支援其他翻譯引擎。
- 手動中斷時沒有用於快取刷新的訊號處理（“SIGINT”/“SIGTERM”）。

## 10、環境要求

- Go 1.x（建置），Linux 系統（使用 `/etc/os-release`、`LC_ALL=C` 在子進程中進行區域設定隔離）。
- 訪問互聯網進行翻譯（離線模式僅適用於預先填充的快取）。
- 對 `$HOME/.cache/chili-tradutor-go/` 和目前工作目錄的寫入權限（對於 `pot/`、`doc/`、`txt/`、`json/`、`yml/`、`html/`、`man/`、`usr/`）。
