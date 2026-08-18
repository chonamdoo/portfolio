# trading-journal

크립토 선물 거래일지 웹 서비스.

**코드** [github.com/chonamdoo/trading-journal](https://github.com/chonamdoo/trading-journal)
**규모** TypeScript 235파일 27,377줄 · 테스트 135개 · 커밋 156
**구성** API 라우트 50개 · 페이지 13개 · 기능 모듈 6개
**운영** Vercel 배포 중 · Next.js App Router + Supabase

---

## 무엇을 하는가

선물 거래 기록을 남기고, 그 기록에서 자기 매매 습관을 찾아내는 서비스입니다.

손익만 보는 게 아니라 **감정 태그, 시간대, 요일, 종목, 복기 태그별로 승률을 쪼갭니다.**
"새벽에 낸 거래의 승률이 낮다" 같은 패턴은 거래를 쌓아야만 보입니다.

| 화면 | 내용 |
|---|---|
| `/` | 대시보드 — KPI 그리드, 에쿼티 커브, 오픈 포지션 |
| `/trades` | 거래 목록 + 필터, 상단 KPI (월 손익·MDD·평균 보유시간·승률) |
| `/analysis` | 에쿼티 / Trading Score / 요일별 / 월간 캘린더 / 종목별 / 복기 태그별 |
| `/analysis/report` | AI 리포트 — Master Score Ring, 행동 패턴, 시간 히트맵, 감정별 승률 |
| `/settings` | 거래소 연동 5종, 초기 자산, 목표, 입금, 데이터 관리 |

---

## 아키텍처

Feature module + Clean Architecture (ADR-0001).

```
src/features/
├── assets/           자산
├── capital-targets/  자본·목표
├── exchange-import/  거래소 동기화
├── reports/          리포트
├── trades/           거래
└── user-profile/     프로필·인증 경계
```

각 모듈이 `domain` / `data` 경계를 갖고, API 라우트는 어댑터 역할만 합니다.
이 구조는 처음부터 있던 게 아니라 8개 슬라이스로 나눠 점진적으로 리팩터링한
결과입니다. 단계별 기록이 `.agent-flow/handoffs/`에 남아 있습니다.

```
slice-1 safety-foundation      slice-5 reports-feature-module
slice-2 trades-tracer          slice-6 user-profile-auth-boundary
slice-3 trades-route-adapter   slice-7 assets-capital-targets
slice-4 trades-lifecycle       slice-8 exchange-import-boundary
```

한 번에 갈아엎지 않고 슬라이스마다 테스트가 통과하는 상태를 유지했습니다.
[agent-flow](agent-flow.md)의 `slice-plan` 단계가 이걸 강제합니다.

---

## 보안 — 남의 거래소 키를 맡는다는 것

거래소 5종(Binance, Bybit, OKX, Bitget, Flipster)의 API 키를 사용자로부터 받아
거래 내역을 동기화합니다. 남의 자산에 접근할 수 있는 키를 대신 보관하는 일이라
여기가 이 프로젝트에서 가장 조심한 부분입니다.

**키 암호화**
`AES-256-GCM`으로 암호화해 저장합니다 (`src/lib/exchange/crypto.ts`).
키 ID(`kid`)를 함께 기록해 **키 링 회전**을 지원합니다 — 암호화 키를 교체해도
과거 데이터를 이전 키로 복호화할 수 있습니다.

**RLS 전면 적용**
14개 테이블 **전부** Row Level Security를 켰고 정책은 46개입니다. 미적용 테이블은
없습니다. 서버 코드에서 `service_role`을 쓰지 않고 anon key + RLS 경로만 사용해,
애플리케이션 버그가 있어도 DB가 최종 방어선으로 남도록 했습니다.

**CI 게이트** (`web-guard.yml`)
```
tsc --noEmit            타입 검사
npm run lint            린트
Supabase Migration      마이그레이션 정합성
Secret Scan             시크릿 스캔
```

---

## AI 개발 흔적이 남아 있는 곳

이 저장소는 `.agent-flow/` 디렉터리를 커밋에 포함하고 있습니다. 지웠다면 더 깔끔했겠지만,
**절차가 실제로 돌았다는 증거**라서 남겼습니다.

- `handoffs/` — 슬라이스별 인계 문서 (설계 판단과 남은 일)
- `prompts/` — 단계별 프롬프트 정본 (`red`, `green`, `gates`, `fix-loop`, `ddd-design` 등)

커밋 로그에도 그대로 보입니다. `[autofix] Address review feedback` 같은 커밋은
리뷰 서브에이전트가 `request-changes`를 내고 `fix-loop` 단계로 되돌아간 흔적입니다.

---

## 남은 것

- `/onboarding` 라우트가 빈 파일입니다. 화면은 있는데 구현이 안 됐습니다.
- ADR이 1건뿐입니다. aitrading은 17건인데 이쪽은 설계 결정을 문서로 남기는 습관이
  늦게 붙었습니다.
