# agent-flow

AI 코딩 에이전트를 검증 가능한 개발 절차 위에서 동작시키는 CLI 워크플로 도구.

**코드** [github.com/chonamdoo/agent-flow](https://github.com/chonamdoo/agent-flow)
**규모** Python 60파일 31,801줄 · 테스트 55파일 · 커밋 425
**구성** 워크플로 5개 · 스킬 49개 · eval 케이스 3종

---

## 문제

AI 코딩 도구를 실무에 붙이면서 반복해서 부딪힌 문제가 셋이었습니다.

**① 산출물은 나오는데 검증이 안 된다.** "테스트를 작성했습니다"라고 보고하지만 실제로
실행됐는지는 알 수 없습니다. 리뷰를 했다고 하지만 무엇을 기준으로 봤는지 남지 않습니다.

**② 대화가 길어지면 요구사항이 증발한다.** 초반에 지시한 값(간격, 색상, 임계치)이 컨텍스트
압축 과정에서 사라집니다. 구현 단계에 도달했을 때는 이미 원래 요구와 다른 것이 만들어져
있습니다.

**③ 에이전트가 범위를 조용히 늘린다.** "이것도 필요할 것 같아서" 하며 요구하지 않은
리팩터링, 모듈 분리, 성능 최적화가 섞여 들어옵니다.

세 문제 모두 프롬프트를 개선하는 방식으로는 해결되지 않았습니다. 개인이 잘 쓰는 요령이
아니라 절차 자체를 고정해야 하는 문제였습니다.

---

## 설계 원칙: 에이전트의 자기 보고를 신뢰하지 않는다

### 판정 권한을 에이전트에서 떼어낸다

워크플로 YAML은 단계 **정의**일 뿐이고, 다음 단계를 결정하는 것은 런너입니다.
에이전트는 런너가 출력한 `next_command`를 그대로 따라야 하며 스스로 단계를 건너뛸 수
없습니다.

### 산출물 파일이 있다고 완료가 아니다

`required_markers`를 선언한 단계는 산출물 문서 끝의 `## Completion Gate` 블록에 그 마커가
전부 있어야 다음 단계로 넘어갑니다.

```
## Completion Gate
usecase-interface: required|optional|n/a
usecase-composition: none|domain-service|application-service|orchestrator|justified
cache-required: yes|no
cache-invalidation-policy: <policy or n/a>
solid-dip-dependency-direction: <summary>
```

설계 단계에서만 20개 넘는 마커를 요구합니다. 계층 경계, 의존성 방향, UseCase 포트,
Repository 어댑터, 캐시 정책, 매핑 경계, 합성 루트, SOLID 각 항목까지 명시적으로 답하지
않으면 통과하지 못합니다.

마커 값도 본문과 대조합니다. `spec-items:`에 개수만 적으면 막히고, 실제 항목 ID 목록과
일치해야 합니다. `design-values: none`을 적었는데 본문에 값이 기록되어 있으면 그 모순을
런너가 잡습니다.

### 주장이 아니라 관찰로 판정한다

TDD의 red 단계에서 테스트를 실제로 돌렸는지는 에이전트의 보고가 아니라 훅이 기록한
명령 실행 로그로 판정합니다.

> The run itself is observed by the `record-command-run.py` PostToolUse hook,
> so "I wrote a test" without a test command in this phase is rejected
> regardless of what you record.
>
> A test that passes on the first run is not a red phase.

마커에 무엇을 적든 런너가 로그를 직접 읽습니다.

### 요구사항에 검증 방식을 하나씩 붙인다

사용자의 모든 지시를 순번이 매겨진 항목으로 기록하고, 각 항목마다 검증 방식을 붙입니다.

```
SPEC-<n>: <requirement>
verify: test:<name> | symbol:<symbol>=<value> | manual
```

사용자가 제공한 구체적 값(간격, 크기, 색상, 지속시간, 임계치)은 `Design Values`로 따로
기록합니다. 이미지나 디자인 링크에서 읽은 값이면 "내가 읽어낸 값이며 원본이 아니다"를
전제로 기록하고, 표로 되읽어 사용자에게 확인받습니다.

이 원장은 대화가 압축되어도 살아남는 유일한 전달자입니다. **리뷰어가 approve해도 미충족
항목이 있으면 단계가 완료되지 않습니다.**

### 같은 세션이 만든 코드를 같은 세션이 리뷰하지 못하게 한다

구현 리뷰와 아키텍처 리뷰 두 단계 모두 독립된 서브에이전트 2개 이상이 나눠서 리뷰합니다.

- 각 리뷰어 섹션에 `reviewer-source: sub-agent` 필수
- 한 명이라도 `request-changes`면 전체 판정은 `request-changes`
- 리뷰어 프로세스가 실패하면 컨트롤러 세션이 대신 리뷰할 수 없음 (차단)

### 범위 확장을 막는다

작업 도중 SPEC이 추가·변경·삭제되면 변경분만 사용자에게 보고하고 확인을 받은 뒤
진행합니다. 주석 정리 단계에도 같은 제약이 걸립니다.

```
comment-scope: final-pass-only
refactor-scope: none
performance-optimization: none
module-split: none
```

### 자동화 도구가 메인 브랜치를 훼손할 수 없게 한다

- 모든 작업을 격리된 git worktree 안에서만 수행 (브랜치만 만드는 것은 불가)
- `guard-protected-branch.sh` 훅으로 보호 브랜치 커밋·푸시 차단
- `worktree-tripwire.py`로 리더 체크아웃 이탈 감지
- 머지 직전 사용자 승인 단계를 별도로 배치

### 승인 경로 자체를 공격 표면으로 본다

승인 훅이 타는 유일한 실행 경로이므로 런처를 방어합니다.

- `LD_PRELOAD`, `LD_AUDIT`, `DYLD_INSERT_LIBRARIES` 등 로더 주입 환경변수 전부 해제
- Python 실행 시 `sys.path`에서 현재 디렉터리 제거 — 프로젝트에 놓인 `argparse.py` 하나가
  승인 프로세스 안에서 실행되는 경로를 차단
- 런처 파일 다이제스트를 run-start 훅이 대조

### 문서의 숫자도 검사 대상이다

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

`red → green → refactor`가 절차에 박혀 있습니다. TDD를 권장이 아니라 게이트로 강제한
것이고, 앞서 적은 대로 red 단계는 훅 로그로 검증됩니다.

---

## 결과

이 절차 위에서 실제 서비스 두 개를 만들어 운영하고 있습니다.

- [aitrading](aitrading.md) — Python 53,226줄, 테스트 함수 1,215개, 커밋 428
- [trading-journal](trading-journal.md) — TypeScript 27,377줄, 커밋 156

두 프로젝트 모두 `.agent-flow/` 디렉터리에 단계별 산출물(handoff, slice-plan, review 기록)이
남아 있습니다. 절차가 실제로 돌았다는 증거입니다.

---

## 한계

개인 도구에 머물러 있습니다. 제가 만든 기준으로 제 프로젝트에 적용한 상태이고, 조직이
함께 합의하고 신뢰하는 단계로 가는 방법은 아직 모릅니다.

풀리지 않은 것은 넷입니다.

- **검증 기준을 조직이 합의하는 절차** — 마커 목록을 누가 정하고 어떻게 바꾸는가.
  지금은 제가 정했기 때문에 제게만 맞습니다.
- **팀원마다 다른 숙련도** — 절차를 이해하고 쓰는 사람과, 막히는 지점에서 우회로를 찾는
  사람에게 같은 게이트가 같은 의미인지 확인하지 못했습니다.
- **절차가 만드는 마찰과 얻는 신뢰의 손익 분기점** — 게이트가 많아 혼자 쓸 땐 감당되지만,
  팀에 넣으면 "왜 이렇게 오래 걸리냐"는 반발이 먼저 올 것입니다.
- **리뷰 서브에이전트 비용의 경제성** — 리뷰어를 병렬로 띄우면 토큰 비용이 배로 늘어납니다.
  팀 규모로 곱해졌을 때의 손익을 계산해 본 적이 없습니다.

앞의 셋은 기술 문제가 아니라 사람과 조직의 문제입니다. 자동화를 늘리는 일이 곧 신뢰를
얻는 일은 아니라는 것이, 이 도구를 쓰면서 가장 분명해진 것입니다.
