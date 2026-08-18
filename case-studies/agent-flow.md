# agent-flow

AI 코딩 에이전트를 검증 가능한 개발 절차 위에서 동작시키는 CLI 워크플로 도구.

**코드** [github.com/chonamdoo/agent-flow](https://github.com/chonamdoo/agent-flow)
**규모** Python 88,320줄 (src+tests) · 테스트 55파일 · 커밋 425
**구성** 워크플로 5개 · 스킬 49개 · eval 케이스 3종

---

## 문제

AI 코딩 도구를 실무에 붙이면서 세 가지 문제에 반복해서 부딪혔습니다.

**검증 부재** — "테스트를 작성했습니다"라고 보고하지만 실제로 실행했는지는 알 수 없습니다.
리뷰했다고 하지만 무엇을 기준으로 봤는지 남지 않습니다.

**요구사항 증발** — 초반에 지시한 값(간격, 색상, 임계치)이 컨텍스트 압축 과정에서
사라집니다. 구현 단계에 오면 이미 원래 요구와 다른 것이 만들어져 있습니다.

**범위 확장** — "이것도 필요할 것 같아서" 하며 요구하지 않은 리팩터링, 모듈 분리, 성능
최적화가 섞여 들어옵니다.

셋 다 프롬프트를 고쳐서는 해결되지 않았습니다. 개인이 잘 쓰는 요령이 아니라 절차를
고정해야 하는 문제였습니다.

---

## 전제: 에이전트의 자기 보고를 신뢰하지 않는다

### 판정 권한 분리

워크플로 YAML은 단계 정의일 뿐이고, 다음 단계는 런너가 정합니다. 에이전트는 런너가
출력한 `next_command`를 그대로 따라야 하며 스스로 단계를 건너뛸 수 없습니다.

### 산출물 파일이 있다고 완료가 아니다

`required_markers`를 선언한 단계는 산출물 문서 끝의 `## Completion Gate` 블록에 그 마커가
전부 있어야 넘어갑니다.

```
## Completion Gate
usecase-interface: required|optional|n/a
usecase-composition: none|domain-service|application-service|orchestrator|justified
cache-required: yes|no
cache-invalidation-policy: <policy or n/a>
solid-dip-dependency-direction: <summary>
```

설계 단계에서만 20개 넘는 마커를 요구합니다. 계층 경계, 의존성 방향, UseCase 포트,
Repository 어댑터, 캐시 정책, 매핑 경계, 합성 루트, SOLID 각 항목까지 명시적으로 답해야
합니다.

마커 값도 본문과 대조합니다. `spec-items:`에 개수만 적으면 막히고 실제 항목 ID 목록과
일치해야 합니다. `design-values: none`을 적었는데 본문에 값이 있으면 런너가 그 모순을
잡습니다.

### 주장이 아니라 관찰로 판정

TDD의 red 단계에서 테스트를 실제로 돌렸는지는 에이전트의 보고가 아니라 훅이 기록한 명령
실행 로그로 판정합니다.

> The run itself is observed by the `record-command-run.py` PostToolUse hook,
> so "I wrote a test" without a test command in this phase is rejected
> regardless of what you record.
>
> A test that passes on the first run is not a red phase.

마커에 무엇을 적든 런너가 로그를 직접 읽습니다.

### 요구사항마다 검증 방식을 붙인다

사용자의 모든 지시를 순번이 매겨진 항목으로 기록하고, 항목마다 검증 방식을 붙입니다.

```
SPEC-<n>: <requirement>
verify: test:<name> | symbol:<symbol>=<value> | manual
```

사용자가 준 구체적 값(간격, 크기, 색상, 지속시간, 임계치)은 `Design Values`로 따로
기록합니다. 이미지나 디자인 링크에서 읽은 값이면 "내가 읽어낸 값이며 원본이 아니다"를
전제로 적고, 표로 되읽어 확인받습니다.

이 원장은 대화가 압축되어도 살아남는 유일한 전달자입니다. 리뷰어가 approve해도 미충족
항목이 남으면 단계가 완료되지 않습니다.

### 리뷰어 격리

구현 리뷰와 아키텍처 리뷰 두 단계 모두 독립된 서브에이전트 2개 이상이 나눠서 봅니다.

- 각 리뷰어 섹션에 `reviewer-source: sub-agent` 필수
- 한 명이라도 `request-changes`면 전체 판정은 `request-changes`
- 리뷰어 프로세스가 실패하면 컨트롤러 세션이 대신 리뷰할 수 없음 (차단)

### 범위 확장 차단

작업 도중 SPEC이 추가·변경·삭제되면 변경분만 보고하고 확인을 받은 뒤 진행합니다. 주석
정리 단계에도 같은 제약을 겁니다.

```
comment-scope: final-pass-only
refactor-scope: none
performance-optimization: none
module-split: none
```

### 메인 브랜치 보호

- 모든 작업을 격리된 git worktree 안에서만 수행. 브랜치만 만드는 것은 불가
- `guard-protected-branch.sh` 훅으로 보호 브랜치 커밋·푸시 차단
- `worktree-tripwire.py`로 리더 체크아웃 이탈 감지
- 머지 직전에 사용자가 승인하는 단계를 따로 배치

### 승인 경로를 공격 표면으로 본다

승인 훅이 타는 유일한 실행 경로이므로 런처를 방어합니다.

- `LD_PRELOAD`, `LD_AUDIT`, `DYLD_INSERT_LIBRARIES` 등 로더 주입 환경변수 전부 해제
- Python 실행 시 `sys.path`에서 현재 디렉터리 제거. 프로젝트에 놓인 `argparse.py` 하나가
  승인 프로세스 안에서 실행되는 경로를 차단
- 런처 파일 다이제스트를 run-start 훅이 대조

### 문서의 숫자도 검사 대상

README에 적힌 워크플로 단계 수, 프로파일 이름, 스킬 수를 `npm run parity:check`가 정본
파일과 대조합니다. 문서가 낡으면 검사가 깨집니다. 자기 보고를 신뢰하지 않는다는 원칙을
문서에도 적용한 것입니다.

---

## 워크플로

작업 크기로 고릅니다. 정본은 `src/agent_flow/workflows/<name>.yaml` 한 벌뿐입니다.

| 워크플로 | 쓰는 때 |
|---|---|
| `review` | 코드 변경 없는 리뷰 |
| `bugfix` | 버그 수정 |
| `development` | 작은 변경 |
| `default` | 일반 기능 |
| `full-feature` | 신규 기능 전체 |

`full-feature`는 24단계입니다.

```
domain-grill → product-brief → prd → slice-plan → plan-review → ddd-design
→ worktree → run-start → red → green → refactor → comment-authoring
→ multi-review → architecture-review → gates ↔ fix-loop
→ commit → push-pr → pr-watch ↔ pr-comment-fix / pr-ci-fix
→ merge-approval → merge → handoff
```

`red → green → refactor`가 절차에 박혀 있습니다. TDD를 권장이 아니라 게이트로 만든
것이고, red 단계는 앞서 적은 대로 훅 로그로 검증합니다.

---

## 결과

이 절차 위에서 서비스 두 개를 만들어 운영하고 있습니다.

- [aitrading](aitrading.md) — Python 118,273줄, 테스트 2,265개, 커밋 428
- [trading-journal](trading-journal.md) — TypeScript 30,655줄, 커밋 156

두 저장소 모두 `.agent-flow/` 디렉터리에 단계별 산출물이 남아 있습니다. handoff 문서,
slice-plan, 리뷰 기록이 들어 있고 절차가 실제로 돌았다는 증거입니다.

---

## 한계

개인 도구에 머물러 있습니다. 제가 만든 기준을 제 프로젝트에 적용한 상태이고, 조직이
함께 합의하고 신뢰하는 단계로 가는 방법은 아직 모릅니다.

풀리지 않은 것은 넷입니다.

- **검증 기준의 합의 절차** — 마커 목록을 누가 정하고 어떻게 바꾸는가. 지금은 제가
  정했기 때문에 제게만 맞습니다.
- **팀원마다 다른 숙련도** — 절차를 이해하고 쓰는 사람과, 막히는 지점에서 우회로를 찾는
  사람에게 같은 게이트가 같은 의미인지 확인하지 못했습니다.
- **마찰과 신뢰의 손익 분기점** — 게이트가 많아 혼자 쓸 땐 감당되지만, 팀에 넣으면 "왜
  이렇게 오래 걸리냐"는 반발이 먼저 옵니다. 어디까지가 지불할 만한 마찰인지 기준이
  없습니다.
- **리뷰 서브에이전트 비용** — 리뷰어를 병렬로 띄우면 토큰 비용이 배로 늘어납니다. 팀
  규모로 곱했을 때의 손익을 계산해 본 적이 없습니다.

앞의 셋은 기술 문제가 아니라 사람과 조직의 문제입니다. 자동화를 늘리는 일이 곧 신뢰를
얻는 일은 아니라는 것이, 이 도구를 쓰면서 가장 분명해졌습니다.
