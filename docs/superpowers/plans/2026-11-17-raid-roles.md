# 무스펠 성역 역할 배분 사이트 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 공대원 10명을 입력하면 1네임드 드리블·2네임드 신호등 자리를 자동 배정하고 파티별 한 줄 문구를 복사하는 웹 페이지를 만든다.

**Architecture:** 파일 하나(`index.html`). 스크립트를 두 부분으로 나눈다 — 화면과 무관한 배정 계산(순수 함수)과 화면 그리기. `?selftest=1`로 열면 배정 계산을 시험 입력으로 돌려 결과를 화면에 보여준다.

**Tech Stack:** HTML, CSS, JavaScript(라이브러리 없음), SVG, localStorage.

## Global Constraints

- 설계 문서: `docs/superpowers/specs/2026-11-17-raid-roles-design.md`. 수치·문구는 그 문서가 기준.
- 직업 9종: 수호성, 검성, 살성, 권성, 궁성, 마도성, 정령성, 치유성, 호법성.
- 드리블 우선: 궁성 → 마도성 → 정령성 → 그 외(입력 순서).
- 1파티 신호등 복사 순서 12·11·9·7시(2줄 자리 11·7시), 2파티 1·3·5·6시(2줄 자리 1·5시).
- 1네임드 경로: 1파티 1,2번 `3->3`, 3,4번 `9->12` / 2파티 1,2번 `12->9`, 3,4번 `6->6`.
- 복사 문구는 쉼표 뒤 공백 없음. 예: `12시:가가,11시:나나,9시:다다,7시:라라,깍두기:마마`.
- git 저장소가 아니므로 커밋 단계는 없다.

---

### Task 1: 배정 계산과 자체 시험

**Files:** Create `aion2-raid-roles/index.html` (스크립트의 계산 부분과 selftest)

**Interfaces (Produces):**
- 상태 `st = { rows: [{name, job, two}]×10, reserve: [null|0..4, null|0..4], dribble: [[null|i, null|i], [null|i, null|i]], slots: [null|[i,i,i,i], null|[i,i,i,i]], tab: 'b1'|'b2' }` — 색인 i는 파티 안의 줄 번호(0~4).
- `autoReserve(rows) → i`
- `autoDribbler(rows, [i, j]) → i`
- `autoSlots(rows, mem, p) → [i,i,i,i]` (복사 순서)
- `compute(st, p) → {ok:false, missing} | {ok:true, rows, reserve, mem, pairs, dribblers, slots, short}`
- `text1(st, p) → string|null`, `text2(st, p) → string|null`
- `runSelfTest() → [{name, pass, got, want}]`

- [ ] 시험 먼저 작성 (기준 입력 — 1파티: 가가 궁성, 나나 검성 ✓, 다다 마도성, 라라 살성 ✓, 마마 치유성 / 2파티: 바바 정령성 ✓, 사사 수호성, 아아 궁성 ✓, 자자 권성, 차차 치유성. ✓ = 2줄 가능)

| 시험 | 기대값 |
|---|---|
| 1넴 1파티 | `가가:3->3,다다:9->12` |
| 1넴 2파티 | `바바:12->9,아아:6->6` |
| 2넴 1파티 | `12시:가가,11시:나나,9시:다다,7시:라라,깍두기:마마` |
| 2넴 2파티 | `1시:바바,3시:사사,5시:아아,6시:자자,깍두기:차차` |
| 치유성 없음, 2줄 [✓,-,✓,-,-] | 깍두기 4번 줄(색인 4) |
| 치유성 없음, 2줄 [-,-,✓,✓,✓] | 깍두기 색인 1 |
| 치유성 없음, 전원 ✓ | 깍두기 색인 4 |
| 치유성 둘(색인 1 ✓, 색인 3 -) | 깍두기 색인 3 |
| 치유성 둘(둘 다 -) | 깍두기 색인 1 |
| 치유성 하나가 ✓ | 그 치유성이 깍두기 |
| 1파티 2줄 0명 | `12시:다다,11시:가가,9시:라라,7시:나나,깍두기:마마`, short=true |
| 1파티 2줄 1명(라라) | `12시:나나,11시:라라,9시:다다,7시:가가,깍두기:마마`, short=true |
| 1파티 2줄 3명(가가·나나·다다) | `12시:다다,11시:가가,9시:라라,7시:나나,깍두기:마마`, short=false |
| 조(검성, 살성) | 담당 = 앞사람 |
| 조(정령성, 궁성) | 담당 = 궁성 |
| 깍두기 직접 지정 = 가가 | 1넴 `다다:3->3,라라:9->12`, 2넴 `12시:다다,11시:나나,9시:마마,7시:라라,깍두기:가가` |
| 드리블 직접 지정(1조 = 나나) | `나나:3->3,다다:9->12` |
| 자리 직접 지정 [1,0,2,3] | `12시:나나,11시:가가,9시:다다,7시:라라,깍두기:마마` |
| 이름 하나 비움 | ok=false, missing=1 |

- [ ] 계산 함수 작성
- [ ] `index.html?selftest=1`을 열어 전부 통과하는지 확인 (문서 제목이 `SELFTEST PASS n/n`)

### Task 2: 입력·탭·복사 화면

**Files:** Modify `aion2-raid-roles/index.html`

**Interfaces (Consumes):** Task 1의 `compute`, `text1`, `text2`.

- [ ] 파티 카드 2개(줄마다 이름, 직업 선택, 2줄 가능 체크, 깍두기 라디오), 파티별 안내 줄("N명 더 입력 필요", 2줄 부족 안내)
- [ ] 탭 2개와 파티별 결과(1넴: 조별 담당 선택 버튼 / 2넴: 자리 버튼), 복사될 문구 미리보기, "1파티 복사"·"2파티 복사"
- [ ] 입력·깍두기가 바뀌면 그 파티의 `dribble`, `slots`를 비운다
- [ ] 브라우저에서 기준 입력을 넣고 복사 문구가 Task 1 표와 같은지 확인

### Task 3: 배치도·저장·자리 바꾸기

**Files:** Modify `aion2-raid-roles/index.html`

- [ ] 2넴 자리 버튼 두 개를 차례로 누르면 같은 파티 안에서 맞바꾸기, "자동 배정으로 되돌리기", "초기화"
- [ ] SVG 배치도 2종(1넴 경로 화살표 + 담당 이름, 2넴 여덟 자리 + 이름)
- [ ] localStorage 저장·불러오기(`try/catch`로 감싼다)
- [ ] 확인: 자리 바꾼 뒤 그림과 문구가 함께 바뀌는지, 새로 고침 뒤 입력이 남는지, 좁은 화면에서 가로로 넘치지 않는지
