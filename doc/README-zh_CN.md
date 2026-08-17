# SPEC — 辣椒-tradutor-go

**软件版本：** 2.1.20 (2026-02-01)
**网站：** https://chililinux.com
**存储库：** https://github.com/chililinux/chili-tradutor-go
**作者：** Vilmar Catafesta <vcatafesta@gmail.com>
**许可证/版权：** 版权所有 (C) 2019-2026 Vilmar Catafesta

---

## 1. 概述

`chili-translator-go` 是一个用 Go 编写的命令行包装器，它使用外部翻译引擎（通过 `translate-shell`）和磁盘缓存系统，自动将不同格式的文件（脚本、文档、结构化数据、手册页）同时翻译成多种语言，以避免重复重新翻译。

该程序主要用于本地化软件项目：它通过“gettext”/“xgettext”从源代码中提取可翻译的字符串，生成“.po”/“.mo”文件，并且还可以直接翻译文档（“.md”、“.txt”、“.json”、“.yaml”、“.html”、手册页），而无需通过 gettext 流。

## 2. 目标

- 自动将一个或多个文件翻译成可配置的语言列表。
- 通过重用已经进行的翻译（持久缓存）来最大限度地减少网络调用。
- 支持经典的 gettext 流（`.po`/`.mo`，用于 `i18n` 应用程序）以及文档和数据的直接翻译。
- 并行处理多种语言，并在终端上实时显示进度。
- 自动检测文件类型（通过扩展名或 shebang），无需手动配置。

## 3. 功能范围

### 3.1 支持的输入格式

|扩展/标准 |检测到类型 |翻译流程|
|---|---|---|
| sem extensão、com shebang（`#!/usr/bin/env python` 等）|脚本（python、php、perl、ruby、javascript、shell）| gettext(`.pot`/`.po`/`.mo`) |
|没有扩展，没有shebang |纯文本|获取文本 |
| `.1` 到 `.9` |手册页 |具有 roff 宏保护的逐行翻译 |
| `.sh .py .php .c .cpp .go .pl .rb` |源代码 | gettext(`.pot`/`.po`/`.mo`) |
| `.html .htm` | HTML |带有标签保护的逐行翻译 |
| `.md .markdown` |降价|具有代码块保护的逐行翻译，保留前缀（`#`、`-`、`1.`） |
| `.txt` |纯文本|逐行翻译|
| `.json` | JSON |将字符串值递归翻译为映射 |
| `.yaml.yml` | yaml |递归翻译（通过 JSON 解析器）|
| `.pot` |模板 gettext |复制到 `pot/` 并作为 PO | 处理
|任何其他扩展 |后备|被视为 shell/gettext |

### 3.2 执行流程（每个文件）

1. 检查文件是否存在。
2. 检测类型（`detectFileType`）并准备相应的输出目录（`pot/`、`doc/`、`txt/`、`json/`、`yml/`、`html/`、`man/`）。
3. 对于 gettext 流：运行 `xgettext` 来提取字符串并生成标准化的 POT 标头 (`stampPotHeader`)。
4. 检查是否有实际要翻译的内容（`hasActualContent`）；如果没有，它会清除空工件并中止文件并发出警告。
5. 为每种目标语言触发一个 goroutine，受大小为“jobs”的信号量（“-j”，默认为 8）限制。
6. 每个 goroutine 都会调用特定于格式的翻译例程（`translateManPage`、`translateHTML`、`translateMarkdown`、`translatePlaintext`、`translateJSON` 或 gettext 流的 `prepareMsginit`/`translateFile`/`writeMsgfmtToMo` 三重奏）。
7. 每个字符串/行/msgid 都通过“callUniversalTranslator”传递，其中：
   - 在任何网络调用之前标准化并查询本地缓存；
   - 在发送到翻译引擎之前保护变量、格式化占位符、链接和 URL（“protectVariables”/“restoreVariables”）；
   - 调用“trans”（translate-shell），最多尝试 3 次并进行渐进式退避；
   - 将结果写入缓存（`~/.cache/chili-tradutor-go/cache.json`）。
8. 使用 ANSI 转义码将光标重新定位在终端的多行区域中，按语言实时显示进度。
9. 在每个文件的末尾，它显示快速统计信息（时间、缓存命中、网络调用）。
10. 在所有文件（如果多个文件）的末尾，显示全局执行摘要。

### 3.3 缓存系统

- 本地：`$HOME/.cache/chili-tradutor-go/cache.json`。
- 结构：`map[语言]map[textoNormalizado]CacheEntry{Value, LastUsed}`。
- 在开始时加载一次（“loadCache”）并在正常执行结束时保存一次（“saveCache”，通过“defer”）。
- `--force` 忽略现有的缓存条目并强制重新翻译。
- `--clean-cache` 删除超过 30 天未使用的条目。

### 3.4 保护不可翻译内容

`protectVariables` 函数在将文本发送到翻译引擎之前替换为占位符 (`CHILI_REF_N_CHILI`)，然后恢复它 (`restoreVariables`)：
- shell 变量：`$VAR`、`${VAR}`。
- 简单格式说明符：“%s”、“%d”（仅限小写字母）。
- Markdown 图像链接：`[texto](url)`、`![alt](url)`。
- 网址 (`http://`, `https://`).

特定格式在委托给“callUniversalTranslator”之前添加自己的保护：
- **手册页：** roff 宏（以“.”开头的行）仅包含翻译后的宏之后的文本；注释（`\"`）原样保留。
- **HTML：** 标签 (`<...>`) 在行翻译之前被占位符 (`CHILI_HTML_N_CHILI`) 替换。
- **Markdown:** 由 ``` ``` ``` 分隔的块不会被翻译；标题/列表/编号前缀在翻译之外保留。

### 3.5 自测试（`--自测试'）

运行一系列简化的内部检查（依赖项、“protectVariables”/“restoreVariables”往返）并向终端打印 OK/FAIL 报告。

### 3.6 `--self`模式

用于从“chili-translator-go”二进制文件中提取和翻译自己的字符串的专用模式（通过“xgettext”从源代码本身中使用“T”/“TN”提取关键字）。

## 4. 命令行界面

```
chili-tradutor-go -i <arquivo> [opções]
```

|短旗|长旗|描述 |标准|
|---|---|---|---|
| `-i` | `--输入文件` |源文件（接受多个，也通过位置参数）| — |
| `-l` | `--语言` |习语列表-alvo（例如：`pt_BR,en`）或`all` | `pt_BR，en，es，it，de，fr，ru，zh_CN，zh_TW，ja，ko` |
| `-e` | `--引擎` |翻译引擎：`google`、`bing`、`yandex` | `谷歌` |
| `-j` | `--工作` |同声传译数量（每种语言的并行度）| `8` |
| `-s` | `--来源` |源语言 | `自动` |
| `-f` | `--force` |忽略缓存，强制新翻译 | `假` |
| — | `--自我` |针对二进制文件本身的专门提取 | `假` |
| — | `--自测试` |执行完整性自检 | `假` |
| — | `--clean-cache` |删除 30 天内未使用的缓存条目 | `假` |
| `-q` | `--安静` |静音模式（部分 - 请参阅限制）| `假` |
| `-v` | `--详细` |详细模式（当前未实现）| `假` |
| `-V` | `--版本` |显示程序版本 | — |

`--other language`中支持的语言：`ar bg cs da de el en es et fa fi fr he hi hr hu is it ja ko nl no pl pt_PT pt_BR ro ru sk sv tr uk zh_CN zh_TW`。

## 5. 外部依赖

|二进制|套餐 |用途 |
|---|---|---|
| `xgettext` |获取文本 |从源代码中提取字符串 |
| `msginit` |获取文本 |按语言初始化 `.po` 文件 |
| `msgfmt` |获取文本 |编译 `.po` → `.mo` |
| `gettext` / `ngettext` |获取文本 |程序接口本身的翻译 (`T`/`TN`) |
| `反式` |翻译外壳 |通过外部引擎执行翻译 |

该程序在启动时检查这些二进制文件是否存在（“checkDependency”），并根据“/etc/os-release”中标识的发行版，通过检测到的包管理器（“pacman”、“xbps-install”、“apt”、“dnf”）提供自动安装。

还在执行开始时检查互联网连接（“checkInternet”，针对“8.8.8.8:53”的 TCP 测试）；如果离线，仍会查阅缓存，但未缓存的文本会以未翻译的形式返回。

## 6. 生成的输出

|条目类型|输出目录|姓名图案|
|---|---|---|
| gettext（código）| `pot/`、`usr/share/locale/<lang>/LC_MESSAGES/` | `<pot>.pot`、`<base>-<lang>.po`、`<base>.mo` |
|手册页 | `男人/` | `<base>-<lang>.<n>` |
| HTML | `html/` | `<base>-<lang>.html` |
|降价| `文档/` | `<base>-<lang>.md` |
|简单文字| `txt/` | `<base>-<lang>.txt` |
| JSON | `json/` | `<base>-<lang>.json` |
| yaml | `yml/` | `<base>-<lang>.yml` |

## 7. 端子输出

- 标头包含名称/版本、检测到的文件类型、引擎、源语言、作业数量和缓存路径。
- 状态为“[等待...]”的目标语言初始列表。
- 按语言显示的进度条，通过 ANSI 转义码（`\033[nA`、`\033[K`、`\033[nB`）就地更新，显示语言、百分比条和格式后缀（`MD`、`TXT`、`HTML`、`MAN`、`PO`、`JSON`、`OK`）。
- 每个文件的快速统计信息：已用时间、缓存命中 (%)、网络调用 (%)、总计。
- 最终执行摘要（仅当处理多个文件时）：总时间、缓存命中、网络调用、失败（如果有）。
- 通过`github.com/fatih/color`使用颜色：青色（突出显示）、绿色（成功）、黄色（警告/状态）、红色（错误）、蓝色（辅助信息）。

## 8. 竞争

- `sync.WaitGroup` + 信号量通道（`chan struct{}, jobs`）限制每个文件同时翻译的语言数量。
- `sync.Mutex` (`mu`) 保护对共享缓存映射的访问。
- `sync.Mutex` (`muConsole`) 在 goroutine 之间序列化对终端的写入。
- 语言完成计数器（“langsDone”）使用“sync/atomic”。

## 9.已知限制 (v2.1.20)

- `.yaml`/`.yml` 文件使用 `encoding/json` 进行反序列化，仅适用于与 JSON 语法兼容的 YAML。
- `translateMap` 不遍历数组（`[]interface{}`），仅遍历映射。
- HTML 中的 `<script>`/`<style>` 块和 Markdown 中的内联代码片段 (`` `code` ``) 不受翻译保护。
- CLI 中存在标志“--verbose”，但对当前行为没有影响。
- `--quiet` 仅抑制进度条，而不抑制其他标题/摘要消息。
- 除“translate-shell”支持的翻译引擎（“google”、“bing”、“yandex”）之外，不支持其他翻译引擎。
- 手动中断时没有用于缓存刷新的信号处理（“SIGINT”/“SIGTERM”）。

## 10、环境要求

- Go 1.x（构建），Linux 系统（使用 `/etc/os-release`、`LC_ALL=C` 在子进程中进行区域设置隔离）。
- 访问互联网进行翻译（离线模式仅适用于预先填充的缓存）。
- 对 `$HOME/.cache/chili-tradutor-go/` 和当前工作目录的写入权限（对于 `pot/`、`doc/`、`txt/`、`json/`、`yml/`、`html/`、`man/`、`usr/`）。
