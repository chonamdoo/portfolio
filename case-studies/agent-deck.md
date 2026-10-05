# AgentDeck

Herdr에서 돌아가는 AI 코딩 에이전트들을 한 창에서 고르고, 각 에이전트의 터미널과 변경 내역을 함께 보는
네이티브 macOS 앱.

**코드** [github.com/chonamdoo/AgentDeck](https://github.com/chonamdoo/AgentDeck)  
**규모** Swift 27,529줄 (앱 21,076 + 테스트 6,453) · XCTest 264개 · 커밋 44  
**상태** v0.1.4 pre-release · Apple Silicon, macOS 14 이상 · Apache-2.0

![AgentDeck 디자인 프로토타입 — 왼쪽 에이전트 목록, 가운데 터미널 pane, 오른쪽 에이전트가 고친 파일](../assets/agentdeck-preview.png)

README에 실린 HTML 프로토타입 화면입니다. 데이터는 목업입니다.

---

## 문제

[Herdr](https://github.com/herdrdev/herdr)는 코딩 에이전트 여러 개를 pane으로 나눠 띄우는 오픈소스
터미널 멀티플렉서입니다. AgentDeck은 그 클라이언트입니다. Herdr를 앱에 넣지 않고, Herdr 서버를 자동으로
시작하거나 확인 없이 진행 중인 작업을 끝내지 않습니다.

Herdr만 쓸 때 부족한 점이 셋 있었습니다.

- **에이전트 상태** — pane마다 어떤 모델과 추론 강도로 돌고 있고 컨텍스트를 얼마나 썼는지 보이지
  않습니다. Herdr에는 이 값의 출처가 없습니다.
- **실제로 고치는 폴더** — [agent-flow](agent-flow.md)는 모든 작업을 격리된 git worktree 안에서 하게
  만듭니다. 그래서 pane은 main 폴더에서 돌고, 수정은 다른 worktree에서 일어납니다. pane 폴더만 보면
  변경이 0으로 나옵니다.
- **한국어 입력** — 한국어 입력원에서 Ctrl+C를 누르면, Kitty 키보드 방식으로 키를 보내는 터미널에서는
  Herdr 0.9.3이 `c` 대신 `ㅊ`을 입력합니다([herdr#1079](https://github.com/herdrdev/herdr/issues/1079)).
  다른 pane을 클릭할 때 조합 중이던 글자도 원래 pane에 확정해야 합니다.

여기에 조건이 하나 더 있었습니다. 터미널 옆에 띄우는 앱이라 추가로 쓰는 메모리가 작아야 했습니다.

---

## 무엇을 하나

- Herdr 세션·에이전트·pane 목록과, pane마다 붙는 터미널 (SwiftTerm 기반)
- pane 아래 정보 줄에 모델, 추론 강도, 컨텍스트 사용률 표시 (Claude Code, Codex, OMP, pi)
- 선택한 pane의 worktree, 변경 파일, diff. 에이전트가 다른 worktree를 고치고 있으면 그쪽을 먼저 보여 줌
- 프로젝트 탭(⌘T, ⌘1~9), 파일 탐색, 문서 미리보기, Herdr 설치·업데이트

---

## 측정해서 정하고, 확정은 미뤘다

### 터미널 엔진

pane마다 터미널 view를 하나씩 띄우는 구조라, 엔진이 메모리와 한국어 입력을 좌우합니다. SwiftTerm과
Ghostty(libghostty)를 같은 실험 앱에 붙여 비교했습니다. 메모리는 앱 프로세스 트리 전체의 값이고,
3회 측정의 중앙값입니다. 같은 조건에서 Terminal.app은 pane 1개에 39.1MiB였습니다.

| 항목 | SwiftTerm | Ghostty |
|---|---|---|
| 앱 쪽 메모리, pane 1개 | 51.5MiB | 185.5MiB |
| pane 6개 동시 연결 | 77.8MiB | 240.3MiB |
| 셸에서 한글 조합 뒤 Enter | 3/3 통과 | 0/3 |
| 셸에서 한글 조합 중 Backspace | 3/3 통과 | 0/2 |
| 빌드 | SwiftPM만 | Zig + Metal 툴체인 + xcframework |

Ghostty 수치는 실험용 어댑터에서 나온 값입니다. 여러 pane 메모리는 pane마다 런타임을 따로 만든 조건이라
런타임을 공유하는 원래 Ghostty 앱보다 클 수 있습니다. 한글 입력 실패도 원인을 아직 몰라 Ghostty 자체의
결함으로 판단하지 않았습니다.

SwiftTerm을 임시로 쓰고 있지만 엔진은 확정하지 않았습니다. 판정 순서의 1순위는 한국어 입력인데, 실제
Claude Code·Codex pane에서는 두 엔진 모두 4건 중 3건만 통과했습니다. 사람이 15분 동안 직접 써 보는
확인도 아직 하지 않았습니다. 엔진 결정 기록(ADR)에는 이렇게 적었습니다.

> 상위 기준 미확정을 하위 메모리·빌드 수치로 덮지 않는다.

### 보이는 pane만 연결한다

숨긴 pane까지 모두 연결해 두면 pane 40개에서 앱 쪽 메모리가 275.0MiB였습니다. 보이는 pane과 직전에 본
1개만 연결하면 106.1MiB입니다. 스크롤과 상태는 Herdr 서버가 갖고 있어서, 숨긴 pane을 떼어도 잃는 것이
없습니다. 대가는 돌아갈 때 다시 연결하는 시간이고, 중앙값 35ms(첫 회 205ms)입니다.

이 절의 수치는 모두 작성자의 Mac(macOS 26.6.2, Apple M5 Pro)에서 Herdr 0.9.3 격리 세션으로 잰
값입니다. 출시 앱이 아니라 엔진 비교용 실험 앱으로 쟀고, 측정 문서도 "이 수치는 엔진 비교용이고 최종
제품의 합격 판정이 아니다"라고 적고 있습니다.

---

## Herdr에 없는 값은 직접 모으고, 확실하지 않은 값은 보여 주지 않는다

Herdr가 모델과 컨텍스트 값을 주지 않으므로, AgentDeck은 실행될 때마다 이 Mac에 설치된 에이전트에 보고
스크립트를 넣습니다. 스크립트가 값을 Herdr pane 메타데이터로 보내면 앱이 읽어 정보 줄에 표시합니다.

사용자의 에이전트 설정을 고치는 일이라 규칙을 정했습니다.

- **Claude Code** — 컨텍스트 창 크기와 사용률을 주는 곳이 상태 줄뿐이라 `statusLine`을 AgentDeck
  스크립트로 바꿉니다. 원래 상태 줄은 따로 보관해 그대로 출력합니다. 사용자가 AgentDeck 상태 줄을 지우면
  다시 넣지 않습니다.
- **Codex** — `hooks.json`에 hook을 더합니다. 설정에서 hook을 꺼 두었다면 아무것도 설치하지 않습니다.
  Codex는 여러 터미널이 함께 쓰는 데몬 안에서 hook을 실행하므로, 값이 어느 pane에서 왔는지 직접 찾아야
  합니다. 같은 폴더에 Codex pane이 둘 이상이면 잘못 붙일 수 있어 아무것도 표시하지 않습니다.
- **공통** — 보고 스크립트는 표준 출력에 아무것도 쓰지 않고 항상 정상 종료합니다. 에이전트가 보여
  주거나 결정하는 내용을 바꾸지 않기 위해서입니다. hook이 모델·컨텍스트를 주지 않는 에이전트(Gemini CLI
  등)는 표시하지 않습니다.

---

## 에이전트가 고치는 곳을 먼저 보여 준다

pane의 작업 위치는 다음 순서로 정합니다.

1. 에이전트가 마지막으로 파일을 고친 worktree
2. agent-flow가 그 세션에 묶어 둔 worktree
3. pane이 실행 중인 폴더

2번은 보고 스크립트가 agent-flow의 세션 기록(`<git-common>/agent-flow/host-sessions/`)을 읽어
찾습니다. agent-flow의 worktree 격리 규칙 때문에 생긴 "변경 0" 착시를 여기서 풀었습니다. pane 폴더
기준으로 바꿔 볼 수도 있습니다.

---

## 검증

### 테스트

XCTest 264개입니다. 데이터 계층(HerdDataTests) 22개 파일, UI 계층(HerdUITests) 14개 파일입니다. diff와
worktree는 테스트 안에서 실제 git 저장소를 만들어 확인하고, 보고 스크립트는 실제 `python3`·`bun`으로
실행해 결과를 검사합니다.

CI는 수동으로 실행하는 DMG 빌드만 하고, 테스트는 돌리지 않습니다.

### 남은 확인이 끝나야 정식 릴리스를 낸다

정식 릴리스 전에 끝내야 할 확인을 `macos/Herd/release-gates.txt`에 두었습니다. 사람이 해야 하거나 오래
재야 하는 확인입니다.

```
user-ime-session pending
memory-matrix pending
```

`user-ime-session`은 사람이 실제 agent pane에서 15분 동안 한국어로 입력해 보는 확인이고, `memory-matrix`는
Terminal.app + Herdr 대비 메모리 비교(시나리오 4종 × 3회)입니다.

배포 스크립트(`publish-release.sh`)는 이 파일을 읽고, 열린 항목이 있으면 정식 릴리스(`--final`)를
거부합니다. 통과 기록도 그 커밋 이후 코드가 바뀌지 않았을 때만 인정하고, 이후 코드가 바뀌면 다시
pre-release로 돌아갑니다. 두 항목은 v0.1.0 때부터 계속 pending이라, v0.1.0부터 v0.1.4까지 다섯 릴리스가
모두 pre-release입니다.

측정 문서에도 같은 기준을 적었습니다.

> 자동 바이트 비교 시험으로 사용자의 수동 IME 확인을 대체하지 않는다.

### agent-flow로 개발

개발에는 [agent-flow](agent-flow.md)를 썼습니다. 요구사항 원장의 SPEC 항목별로 수동 증거를 승인해 CLI로
기록했고, 원래 기준의 메모리 비교가 끝나지 않은 SPEC-8은 완료 승인을 기록하지 않았습니다.

---

## 한계

- 정식 출시 전입니다. 위 두 확인 항목이 남아 있습니다.
- 무료 ad-hoc 서명이고 Apple 공증을 받지 않아, 처음 실행할 때 macOS에서 직접 허용해야 합니다.
- Apple Silicon 전용입니다.
- Herdr 0.9.3의 `terminal attach` 경로로는 터미널 인라인 이미지(Kitty graphics)가 전달되지 않습니다.
  두 엔진 모두 같았고, ADR은 이것을 엔진이 아닌 Herdr 전송 경로의 제한으로 기록했습니다.
- 한국어 입력원의 Ctrl 보정은 SwiftTerm 내부 동작에 기대는 우회입니다. SwiftTerm을 올릴 때마다 IME 검사
  스크립트를 다시 돌려야 합니다.
- Codex 값은 새로 시작한 세션부터 보입니다. 같은 폴더에서 Herdr 밖의 Codex가 같은 데몬을 쓰면 값이 섞일
  수 있습니다.

---

## 수치 기준

2026-10-05, v0.1.4(`a57d5e6`) 기준입니다. Swift 줄 수는 `git ls-files`로 추적되는 직접 작성한 파일만
`wc -l`로 셌습니다.

| 포함 | 줄 |
|---|---|
| 앱 소스 `macos/Herd/Sources` | 19,806 |
| 앱이 함께 빌드하는 터미널 모듈 (HerdTerminalKit, SpikeCore) | 1,270 |
| 테스트 `macos/Herd/Tests` | 6,453 |

함께 들어 있는 SwiftTerm(MIT, 102,887줄), 컴파일하지 않는 디자인 참고 코드(9,502줄), 엔진 비교용 실험
코드와 그 테스트는 뺐습니다. XCTest 수는 `func test…()` 메서드를 센 값입니다.
