# SPEC — 칠리-tradutor-go

**소프트웨어 버전:** 2.1.20(2026-02-01)
**사이트:** https://chililinux.com
**저장소:** https://github.com/chililinux/chili-tradutor-go
**저자:** 빌마르 카타페스타 <vcatafesta@gmail.com>
**라이센스/저작권:** 저작권 (C) 2019-2026 Vilmar Catafesta

---

## 1. 개요

`chili-translator-go`는 반복적인 재번역을 피하기 위해 외부 번역 엔진(`translate-shell`을 통해)과 온디스크 캐싱 시스템을 사용하여 다양한 형식(스크립트, 문서, 구조화된 데이터, 매뉴얼 페이지)의 파일을 여러 언어로 동시에 자동 번역하는 Go로 작성된 명령줄 래퍼입니다.

이 프로그램은 주로 소프트웨어 프로젝트 현지화를 위해 설계되었습니다. `gettext`/`xgettext`를 통해 소스 코드에서 번역 가능한 문자열을 추출하고 `.po`/`.mo` 파일을 생성하며 gettext 스트림을 거치지 않고 문서(`.md`, `.txt`, `.json`, `.yaml`, `.html`, 매뉴얼 페이지)를 직접 번역합니다.

## 2. 목표

- 하나 이상의 파일을 구성 가능한 언어 목록으로 자동 번역합니다.
- 이미 수행된 번역을 재사용하여 네트워크 호출을 최소화합니다(영구 캐시).
- 클래식 gettext 스트림(`i18n` 애플리케이션에서 사용하기 위한 `.po`/`.mo`)과 문서 및 데이터의 직접 번역을 모두 지원합니다.
- 터미널에서 실시간 시각적 진행을 통해 여러 언어를 병렬로 처리합니다.
- 수동 구성 없이 파일 형식(확장자 또는 shebang 기준)을 자동 감지합니다.

## 3. 기능적 범위

### 3.1 지원되는 입력 형식

| 확장/기준 | 유형이 감지됨 | 번역 흐름 |
|---|---|---|
| 확장명, com shebang(`#!/usr/bin/env python` 등) | 스크립트(파이썬, PHP, Perl, 루비, 자바스크립트, 쉘) | gettext (`.pot`/`.po`/`.mo`) |
| 확장 없음, shebang 없음 | 일반 텍스트 | gettext |
| `.1` 에서 `.9` | 매뉴얼 페이지 | roff 매크로 보호를 사용한 한 줄씩 번역 |
| `.sh .py .php .c .cpp .go .pl .rb` | 소스 코드 | gettext (`.pot`/`.po`/`.mo`) |
| `.html .htm` | HTML | 태그 보호를 통한 한 줄씩 번역 |
| `.md .markdown` | 마크다운 | 코드 블록 보호 기능을 갖춘 한 줄씩 번역, 접두사(`#`, `-`, `1.`) 유지 |
| `.txt` | 일반 텍스트 | 한 줄씩 번역 |
| `.json` | JSON | 문자열 값을 맵으로 재귀적으로 변환 |
| `.yaml .yml` | YAML | 재귀 번역(JSON 파서를 통해) |
| `.pot` | 템플릿 gettext | `pot/`에 복사되어 PO로 처리됨 |
| 다른 확장 | 대체 | 쉘/gettext로 처리 |

### 3.2 실행 흐름(파일별)

1. 파일이 존재하는지 확인합니다.
2. 유형(`DetectFileType`)을 감지하고 해당 출력 디렉터리(`pot/`, `doc/`, `txt/`, `json/`, `yml/`, `html/`, `man/`)를 준비합니다.
3. gettext 스트림의 경우: `xgettext`를 실행하여 문자열을 추출하고 표준화된 POT 헤더(`stampPotHeader`)를 생성합니다.
4. 번역할 실제 콘텐츠가 있는지 확인합니다(`hasActualContent`). 아무것도 없으면 빈 아티팩트를 정리하고 경고와 함께 파일을 중단합니다.
5. `jobs` 크기(`-j`, 기본값 8)의 세마포어에 의해 제한되는 대상 언어당 하나의 고루틴을 트리거합니다.
6. 각 고루틴은 형식별 번역 루틴(`translateManPage`, `translateHTML`, `translateMarkdown`, `translatePlaintext`, `translateJSON` 또는 gettext 스트림의 `prepareMsginit`/`translateFile`/`writeMsgfmtToMo` 트리오)을 호출합니다.
7. 각 문자열/줄/msgid는 `callUniversalTranslator`를 통해 전달됩니다.
   - 네트워크 호출 전에 로컬 캐시를 정규화하고 쿼리합니다.
   - 번역 엔진(`protectVariables`/`restoreVariables`)으로 보내기 전에 변수, 서식 지정 자리 표시자, 링크 및 URL을 보호합니다.
   - 최대 3번의 시도와 점진적인 백오프로 'trans'(translate-shell)를 호출합니다.
   - 결과를 캐시(`~/.cache/chili-tradutor-go/cache.json`)에 씁니다.
8. 터미널의 여러 줄 영역에 커서를 재배치하기 위해 ANSI 이스케이프 코드를 사용하여 진행 상황을 언어별로 실시간으로 표시합니다.
9. 각 파일 끝에는 빠른 통계(시간, 캐시 적중, 네트워크 호출)가 표시됩니다.
10. 모든 파일 끝에(두 개 이상인 경우) 전체 요약 내용이 표시됩니다.

### 3.3 캐시 시스템

- 로컬: `$HOME/.cache/chili-tradutor-go/cache.json`.
- 구조: `map[언어]map[textoNormalizado]CacheEntry{값, LastUsed}`.
- 처음에 한 번 로드되고(`loadCache`) 일반 실행이 끝날 때 한 번 저장됩니다(`saveCache`, `defer`를 통해).
- `--force`는 기존 캐시 항목을 무시하고 재번역을 강제합니다.
- `--clean-cache`는 30일 이상 사용되지 않은 항목을 제거합니다.

### 3.4 번역 불가능한 콘텐츠의 보호

`protectVariables` 함수는 번역 엔진에 텍스트를 보내기 전에 자리 표시자(`CHILI_REF_N_CHILI`)로 대체한 다음 복원합니다(`restoreVariables`).
- 쉘 버전: `$VAR`, `${VAR}`.
- 단순 형식 지정자: `%s`, `%d`(소문자만 해당).
- 링크 및 이미지 마크다운: `[texto](url)`, `![alt](url)`.
- URL(`http://`,`https://`).

특정 형식은 `callUniversalTranslator`에 위임하기 전에 자체 보호를 추가합니다.
- **맨 페이지:** roff 매크로(`.`로 시작하는 줄)에는 번역된 매크로 다음의 텍스트만 있습니다. 주석(`\"`)은 그대로 유지됩니다.
- **HTML:** 태그(`<...>`)는 줄 번역 전에 자리 표시자(`CHILI_HTML_N_CHILI`)로 대체됩니다.
- **마크다운:** ``` ``` ```로 구분된 블록은 번역되지 않습니다. 제목/목록/번호 매기기 접두사는 번역 외부에서 유지됩니다.

### 3.5 자가 테스트(`--self-test')

내부 검사(종속성, `protectVariables`/`restoreVariables` 왕복)의 단순화된 배터리를 실행하고 OK/FAIL 보고서를 터미널에 인쇄합니다.

### 3.6 `--self` 모드

`chili-translator-go` 바이너리에서 자체 문자열을 추출하고 번역하기 위한 특수 모드(`xgettext`를 통해 소스 코드 자체에서 `T`/`TN` 추출 키워드 사용)

## 4. 명령줄 인터페이스

```
chili-tradutor-go -i <arquivo> [opções]
```

| 짧은 플래그 | 긴 깃발 | 설명 | 표준 |
|---|---|---|---|
| `-i` | `--입력파일` | 소스 파일(위치 인수를 통해서도 배수 허용) | — |
| `-l` | `--언어` | 관용어 목록-alvo(예: `pt_BR,en`) 또는 `all` | `pt_BR,en,es,it,de,fr,ru,zh_CN,zh_TW,ja,ko` |
| `-e` | `--엔진` | 번역 엔진: `google`, `bing`, `yandex` | '구글' |
| `-j` | `--작업` | 동시번역수(언어별 병렬도) | `8` |
| `-s` | `--소스` | 소스 언어 | '자동' |
| `-f` | `--force` | 캐시를 무시하고 새로운 번역을 강제합니다 | '거짓' |
| — | `--self` | 바이너리 자체에 대한 특화된 추출 | '거짓' |
| — | `--자체 테스트` | 무결성 자체 테스트 수행 | '거짓' |
| — | `--clean-cache` | 30일 동안 사용되지 않은 캐시 항목 제거 | '거짓' |
| `-q` | `--조용` | 자동 모드(부분 - 제한 사항 참조) | '거짓' |
| `-v` | `--상세` | Verbose 모드(현재 구현되지 않음) | '거짓' |
| `-V` | `--버전` | 프로그램 버전 표시 | — |

`--다른 언어`에서 지원되는 언어: `ar bg cs da de el en es et fa fi fr he hi hr hu is it ja ko nl no pl pt_PT pt_BR ro ru sk sv tr uk zh_CN zh_TW`.

## 5. 외부 의존성

| 바이너리 | 패키지 | 사용법 |
|---|---|---|
| `xgettext` | gettext | 소스 코드에서 문자열 추출 |
| `msginit` | gettext | 언어별 `.po` 파일 초기화 |
| `msgfmt` | gettext | 컴파일 `.po` → `.mo` |
| `gettext` / `ngettext` | gettext | 프로그램 인터페이스 자체의 번역(`T`/`TN`) |
| `트랜스` | 번역 쉘 | 외부 엔진을 통한 번역 실행 |

프로그램은 시작 시 이러한 바이너리의 존재를 확인하고(`checkDependency`) `/etc/os-release`에서 식별된 배포판에 따라 감지된 패키지 관리자(`pacman`, `xbps-install`, `apt`, `dnf`)를 통해 자동 설치를 제공합니다.

또한 실행 시작 시 인터넷 연결을 확인합니다(`checkInternet`, `8.8.8.8:53`에 대한 TCP 테스트). 오프라인인 경우 캐시는 계속 참조되지만 캐시되지 않은 텍스트는 번역되지 않은 채 반환됩니다.

## 6. 생성된 출력

| 응모유형 | 출력 디렉터리 | 이름 패턴 |
|---|---|---|
| gettext(코드) | `pot/`, `usr/share/locale/<lang>/LC_MESSAGES/` | `<pot>.pot`, `<base>-<lang>.po`, `<base>.mo` |
| 맨 페이지 | `남자/` | `<베이스>-<언어>.<n>` |
| HTML | `html/` | `<base>-<lang>.html` |
| 마크다운 | `문서/` | `<베이스>-<언어>.md` |
| 간단한 텍스트 | `txt/` | `<베이스>-<lang>.txt` |
| JSON | `json/` | `<베이스>-<lang>.json` |
| YAML | `yml/` | `<베이스>-<lang>.yml` |

## 7. 터미널 출력

- 이름/버전, 감지된 파일 형식, 엔진, 소스 언어, 작업 수 및 캐시 경로가 포함된 헤더입니다.
- 상태가 "[대기 중...]"인 대상 언어의 초기 목록입니다.
- 언어별 진행률 표시줄, ANSI 이스케이프 코드(`\033[nA`, `\033[K`, `\033[nB`)를 통해 내부 업데이트되어 언어, 백분율 막대 및 형식 접미사(`MD`, `TXT`, `HTML`, `MAN`, `PO`, `JSON`, `OK`)를 표시합니다.
- 파일당 빠른 통계: 경과 시간, 캐시 적중률(%), 네트워크 호출(%), 총계.
- 최종 요약(처리된 파일이 두 개 이상인 경우에만): 총 시간, 캐시 적중, 네트워크 호출, 실패(있는 경우).
- `github.com/fatih/color`를 통한 색상 사용: 청록색(강조 표시), 녹색(성공), 노란색(경고/상태), 빨간색(오류), 파란색(보조 정보).

## 8. 경쟁

- `sync.WaitGroup` + 세마포어 채널(`chan struct{}, jobs`)은 파일당 동시에 번역되는 언어 수를 제한합니다.
- `sync.Mutex`(`mu`)는 공유 캐시 맵에 대한 액세스를 보호합니다.
- `sync.Mutex`(`muConsole`)는 고루틴 사이의 터미널에 쓰기를 직렬화합니다.
- 언어 완료 카운터(`langsDone`)는 `sync/atomic`을 사용합니다.

## 9. 알려진 제한사항(v2.1.20)

- `.yaml`/`.yml` 파일은 `encoding/json`을 사용하여 역직렬화되며 JSON 구문과 호환되는 YAML에서만 작동합니다.
- `translateMap`은 배열(`[]interface{}`)을 순회하지 않고 지도만 순회합니다.
- HTML의 `<script>`/`<style>` 블록과 Markdown의 인라인 코드 조각(`` `code` ``)은 번역이 보호되지 않습니다.
- '--verbose' 플래그는 CLI에 있지만 현재 동작에는 영향을 주지 않습니다.
- `--quiet`은 진행률 표시줄만 표시하고 다른 헤더/요약 메시지는 표시하지 않습니다.
- `translate-shell`(`google`, `bing`, `yandex`)에서 지원하는 번역 엔진 이외의 번역 엔진은 지원하지 않습니다.
- 수동 인터럽트 시 캐시 플러시에 대한 신호 처리(`SIGINT`/`SIGTERM`)가 없습니다.

## 10. 환경 요구사항

- Go 1.x(빌드), Linux 시스템(하위 프로세스의 로캘 격리를 위해 `/etc/os-release`, `LC_ALL=C` 사용)
- 번역을 위한 인터넷 액세스(오프라인 모드는 미리 채워진 캐시에서만 작동함)
- `$HOME/.cache/chili-tradutor-go/` 및 현재 작업 디렉터리(`pot/`, `doc/`, `txt/`, `json/`, `yml/`, `html/`, `man/`, `usr/`의 경우)에 대한 쓰기 권한입니다.
