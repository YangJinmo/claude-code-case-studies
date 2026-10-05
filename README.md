# Claude Code Workflow

Claude Code를 매 세션 실제로 불러오는 설정과, 세션 로그로 검증한 실사용
사례를 함께 정리했습니다. 프레임워크 원본 파일([SuperClaude Framework](https://github.com/SuperClaude-Org/SuperClaude_Framework))은
이 레포에 복제해두지 않고 원본 링크로 대체했고, 직접 만든 `Skills/`,
`Commands/`만 실제 파일로 들어 있습니다.

## 실증된 사용 사례

### 1. 구조화된 작업 관리 (Task 도구)
**2026-07-22 · PartMoney (Swift iOS 앱)**

`TaskCreate`/`TaskUpdate` 79회 호출. 모놀리식 구조를 Clean Architecture로
리팩터링하는 과정을 27개 태스크로 쪼개 순서대로 진행:

1. **안전망 확보**: Git init + 리팩터링 전 상태 baseline 커밋
2. **계층 구축**: Domain(Entities/Interfaces/UseCases) → Data(Repository 구현체) → DI 컨테이너
3. **기능별 MVVM 전환**: Home → Calendar → Presets → Earnings → Settings 순으로 5개 기능을 하나씩 이관
4. **정리**: 기존 모놀리식 파일 삭제, 시뮬레이터 스모크 테스트, 커밋
5. **하드닝**: UserDefaults 키 중복 제거, 저장 실패 시 알림 노출, 페이데이 알림 race condition 수정,
   리마인더 리드타임 변경 시 알림이 사라지는 버그 수정, 격주 페이데이 연도 경계 버그 수정 등
   실사용 중 발견된 버그 7건 수정
6. **검증**: Domain/UseCase 유닛 테스트 추가, 빌드·실행·커밋 반복

큰 리팩터링을 계층 단위/기능 단위로 쪼개고, 각 단계마다 빌드 가능한 상태를
유지하며 진행한 사례입니다.

### 2. 멀티 에이전트 병렬 코드 리뷰
**2026-08-07 · PartMoney 외 1개 프로젝트**

`/code-review` high-effort 실행. "정확성 A/B/C" 3개 앵글과
"제거된 동작/교차 함수/재사용/단순화/효율/추상화 레벨/컨벤션" 5개 앵글,
총 8개 관점의 서브에이전트를 병렬로 디스패치해 각자 후보를 찾게 하고,
1-vote 검증 서브에이전트 3배치로 교차 검증한 뒤 결과를 취합하는 방식으로 진행
(세션당 서브에이전트 15개 디스패치).

같은 방식으로 별도 프로젝트(유튜브 자막 추출 스크립트)를 리뷰했을 때
실제로 잡아낸 버그 10건 중 일부:
- 자막 다운로드 시 언어 코드를 검증하지 않고 임시 폴더에서 첫 번째 매칭 파일을 반환 → 잘못된 언어 자막 반환 가능
- 챕터 제목/시작시간이 `null`로 명시된 경우 `dict.get()`의 기본값이 적용되지 않아 `TypeError` 발생
- `subprocess.run` 3곳 중 2곳에 `timeout`과 예외 처리 누락 → 외부 프로세스 행 시 무한 대기
- 클립보드 복사 시 Windows `clip.exe`는 UTF-8이 아닌 콘솔 코드페이지를 쓰는데 항상 UTF-8로 인코딩

### 3. 커스텀 스킬 제작 및 활용
**2026-08-07 ~ 2026-08-30**

- **`tube-info` (08-07)**: 유튜브 URL을 넣으면 제목/조회수/자막/타임라인을 추출하는
  스크립트를 직접 작성한 뒤, `skill-creator`로 재사용 가능한 스킬로 전환
- **`app-mockup` (08-28, 08-30)**: 앱 스크린샷을 아이폰/갤럭시 기기 프레임에 합성해
  배경 투명 1920x1080 이미지로 만드는 스킬. Somvely 프로젝트에서 홈/필터/상품/장바구니/결제
  화면 5장을 이 스킬로 일괄 처리
- **`karpathy-guidelines` (08-26)**: 과설계 방지 가이드라인 스킬이 의도대로 트리거되는지
  동작 테스트 (실제 코드 리뷰 적용 사례는 아직 없음)

### 4. Playwright MCP 브라우저 자동화
**2026-08-27**

로컬 개발 서버(`http://127.0.0.1:4899`)를 대상으로 하루 동안 3차례에 걸쳐
`browser_navigate` → `browser_snapshot`/`browser_console_messages` → `browser_click` →
`browser_evaluate` 흐름을 반복 실행 (총 25회 호출). 코드 수정 후 실제 브라우저에서
동작을 재현·확인하며 반복 검증하는 용도로 사용.

## 구성

`CLAUDE.md`는 Claude Code가 세션 시작 시 읽는 진입점 파일로, 아래 설정
파일들을 전부 import합니다. 파일 자체는 [SuperClaude Framework](https://github.com/SuperClaude-Org/SuperClaude_Framework)
원본을 그대로 쓰고 있어서 이 레포에 복제해두지 않았고, 링크는 전부
원본 레포를 가리킵니다. 실제로 매 세션 로드되는 설정이지 데모용이
아닙니다.

### Behavioral Modes

| 모드 | 용도 |
|---|---|
| [Brainstorming](https://github.com/SuperClaude-Org/SuperClaude_Framework/blob/master/src/superclaude/modes/MODE_Brainstorming.md) | 모호한 요청에 대한 소크라테스식 질문 |
| [Task Management](https://github.com/SuperClaude-Org/SuperClaude_Framework/blob/master/src/superclaude/modes/MODE_Task_Management.md) | 다단계 작업의 계층적 계획·메모리 관리 |
| [Orchestration](https://github.com/SuperClaude-Org/SuperClaude_Framework/blob/master/src/superclaude/modes/MODE_Orchestration.md) | 작업별 최적 도구/MCP 서버 선택 |
| [Introspection](https://github.com/SuperClaude-Org/SuperClaude_Framework/blob/master/src/superclaude/modes/MODE_Introspection.md) | 에러·복잡한 판단 이후 메타인지적 자기 점검 |
| [Deep Research](https://github.com/SuperClaude-Org/SuperClaude_Framework/blob/master/src/superclaude/modes/MODE_DeepResearch.md) | 근거 기반, 출처 명시 리서치 |
| [Token Efficiency](https://github.com/SuperClaude-Org/SuperClaude_Framework/blob/master/src/superclaude/modes/MODE_Token_Efficiency.md) | 컨텍스트 압박 시 기호 기반 압축 커뮤니케이션 |
| [Business Panel](https://github.com/SuperClaude-Org/SuperClaude_Framework/blob/master/src/superclaude/modes/MODE_Business_Panel.md) | Christensen·Porter 등 9명의 경영 프레임워크로 전략 문서를 다각도로 분석 |

### MCP 서버 라우팅 규칙

아래 문서들은 각 서버를 언제 선택해야 하는지 정의한 규칙입니다
(예: 공식 문서 조회는 Context7, 다단계 추론은 Sequential, 브라우저 테스트는
Playwright). **다만 이 중 실제 호출 이력이 확인된 것은 Playwright뿐입니다**
(위 "실증된 사용 사례" 4번). 나머지는 라우팅 규칙만 구성해뒀고 실사용 검증은
아직입니다.

### Core Rules & Principles

- [`RULES.md`](https://github.com/SuperClaude-Org/SuperClaude_Framework/blob/master/src/superclaude/core/RULES.md) — 세션 워크플로우, Git 안전 수칙, 스코프 규율
- [`PRINCIPLES.md`](https://github.com/SuperClaude-Org/SuperClaude_Framework/blob/master/src/superclaude/core/PRINCIPLES.md) — SOLID, DRY/KISS/YAGNI, 근거 기반 의사결정
- [`FLAGS.md`](https://github.com/SuperClaude-Org/SuperClaude_Framework/blob/master/src/superclaude/core/FLAGS.md) — 모드/깊이/MCP 선택을 수동으로 override하는 플래그
- [`RESEARCH_CONFIG.md`](https://github.com/SuperClaude-Org/SuperClaude_Framework/blob/master/src/superclaude/core/RESEARCH_CONFIG.md) — 딥리서치 워크플로우 기본값

### Skills

요청이 스킬 설명과 매칭되면 자동으로 트리거되는 커스텀 스킬:

| Skill | 용도 |
|---|---|
| [Skills/tube-info](Skills/tube-info/SKILL.md) | TubeAlfred MCP로 유튜브 채널/영상 정보·챕터·전체 자막 요약 |
| [Skills/app-mockup](Skills/app-mockup/SKILL.md) | 앱 스크린샷을 아이폰/갤럭시 기기 목업 프레임에 합성 (프레임 PNG는 라이선스상 저장소 미포함) |

### Commands

스킬에 `/이름` 형태의 진입점을 붙인 얇은 래퍼 (로직 중복 없음):

| Command | 호출 대상 |
|---|---|
| [Commands/tube-info.md](Commands/tube-info.md) | `/tube-info <url>` → Skills/tube-info |
| [Commands/app-mockup.md](Commands/app-mockup.md) | `/app-mockup` → Skills/app-mockup |

### 사용법

프레임워크 설정 파일은 [SuperClaude Framework](https://github.com/SuperClaude-Org/SuperClaude_Framework)를
설치하면 따라옵니다. `~/.claude/`(전역) 또는 `.claude/`(프로젝트별)에 두고
자신의 `CLAUDE.md`에서 `@파일명.md`로 import하면 됩니다. 이 레포에서 실제로
가져다 쓸 수 있는 건 `Skills/`, `Commands/`뿐입니다 — Skill은
`~/.claude/skills/<name>/SKILL.md`, Command는 `~/.claude/commands/<name>.md`에
넣습니다.

## 로드 방식: 자동 vs 조건부 vs 명시적 호출

"설정은 됐는데 왜 실행 이력이 없는 항목이 있는가"를 이해하려면 이 레포의
파일들이 세 가지 다른 방식으로 작동한다는 걸 구분해야 합니다.

### 1. 항상 자동으로 로드됨 — 별도 호출 불필요

`CLAUDE.md`가 세션 시작 시 나머지 파일을 전부 `@import`하기 때문에,
아래는 매 세션 시작할 때마다 자동으로 컨텍스트에 실립니다. 사용자가 명령어를
치거나 파일을 지정할 필요가 없습니다.

- `RULES.md`, `PRINCIPLES.md`, `FLAGS.md`, `RESEARCH_CONFIG.md`
- `MODE_*.md` 전부
- `MCP_*.md` 전부

이 안에서도 성격이 두 가지로 나뉩니다.

**항상 적용되는 규칙** — `RULES.md`, `PRINCIPLES.md`. "Git status 먼저 확인",
"쓰기 전에 읽기", "근거 없는 주장 금지" 같이 조건 없이 매 세션 지키려고
하는 행동 규칙입니다. 특정 트리거가 있어야 켜지는 게 아니라 항상 배경에서
작동합니다.

**조건이 맞으면 자동으로 전환되는 모드/라우팅** — `MODE_*.md`, `MCP_*.md`.
각 파일에는 "Activation Triggers"가 정의돼 있고 (예: "모호한 요청 →
Brainstorming", "브라우저 테스트 필요 → Playwright"), 대화 맥락에서 그
조건이 감지되면 Claude가 스스로 판단해서 해당 모드/서버로 전환합니다.
사용자가 "Brainstorming 모드 켜줘"라고 호출하는 게 아니라, 트리거 조건이
항상 감시되고 있다가 맞으면 자동으로 적용되는 방식입니다. 위 "구성은
됐지만 아직 사용 안 한 기능" 목록에 있는 항목들도 이 트리거 자체는 매
세션 감시되고 있었지만, 실제로 조건이 뚜렷하게 걸린 적이 없었던 것뿐입니다.

### 2. 명시적으로 호출해야 작동함

- **Skills** (`Skills/tube-info`, `Skills/app-mockup`) — 컨텍스트에 자동으로
  실리지 않습니다. 요청 내용이 스킬 설명과 매칭되거나 `/이름`을 직접 입력해야
  Claude가 해당 스킬 파일을 불러와 그 지침을 따릅니다.
- **Commands** (`Commands/tube-info.md`, `Commands/app-mockup.md`) — 슬래시
  명령을 직접 입력해야 실행됩니다.
- **MCP 서버의 실제 도구 호출** — `MCP_*.md`의 라우팅 규칙은 항상 로드돼
  있지만, 실제로 그 서버의 도구를 호출할지는 매번 그때그때 판단하는
  런타임 결정이고, 서버가 연결돼 있어야 합니다 (연결이 끊기면 라우팅
  규칙이 있어도 호출 자체가 불가능).

## 구성은 됐지만 아직 사용 안 한 기능

레포에 설정은 돼 있지만 세션 로그상 실행 이력이 없는 기능입니다. 뭘 더
써볼 수 있는지 참고용으로 남겨둡니다.

### Behavioral Modes

| 모드 | 용도 |
|---|---|
| [Brainstorming](https://github.com/SuperClaude-Org/SuperClaude_Framework/blob/master/src/superclaude/modes/MODE_Brainstorming.md) | 모호한 요청에 질문을 던져 요구사항을 구체화 |
| [Introspection](https://github.com/SuperClaude-Org/SuperClaude_Framework/blob/master/src/superclaude/modes/MODE_Introspection.md) | 에러나 복잡한 판단 이후 스스로의 추론 과정을 되짚어봄 |
| [Deep Research](https://github.com/SuperClaude-Org/SuperClaude_Framework/blob/master/src/superclaude/modes/MODE_DeepResearch.md) | 여러 출처를 병렬 검색·신뢰도 채점해 근거 기반으로 종합 |
| [Token Efficiency](https://github.com/SuperClaude-Org/SuperClaude_Framework/blob/master/src/superclaude/modes/MODE_Token_Efficiency.md) | 컨텍스트가 부족할 때 기호·축약어로 압축해 커뮤니케이션 |
| [Orchestration](https://github.com/SuperClaude-Org/SuperClaude_Framework/blob/master/src/superclaude/modes/MODE_Orchestration.md) | 작업 유형에 따라 최적 도구/MCP 서버를 자동으로 선택 |
| [Business Panel](https://github.com/SuperClaude-Org/SuperClaude_Framework/blob/master/src/superclaude/modes/MODE_Business_Panel.md) | 9명의 경영 프레임워크로 전략 문서·사업 아이디어를 다각도로 분석 (discussion/debate/socratic) |

### MCP 서버

| 서버 | 용도 |
|---|---|
| [Context7](https://github.com/SuperClaude-Org/SuperClaude_Framework/blob/master/src/superclaude/mcp/MCP_Context7.md) | 공식 라이브러리 문서를 버전에 맞게 조회 |
| [Sequential](https://github.com/SuperClaude-Org/SuperClaude_Framework/blob/master/src/superclaude/mcp/MCP_Sequential.md) | 복잡한 다단계 추론이 필요한 디버깅·아키텍처 분석 |
| [Serena](https://github.com/SuperClaude-Org/SuperClaude_Framework/blob/master/src/superclaude/mcp/MCP_Serena.md) | 심볼 단위 코드 탐색, 세션 간 메모리 유지 |
| [Morphllm](https://github.com/SuperClaude-Org/SuperClaude_Framework/blob/master/src/superclaude/mcp/MCP_Morphllm.md) | 여러 파일에 걸친 패턴 기반 대량 수정 |
| [Magic](https://github.com/SuperClaude-Org/SuperClaude_Framework/blob/master/src/superclaude/mcp/MCP_Magic.md) | 21st.dev 패턴 기반 UI 컴포넌트 생성 |
| [Tavily](https://github.com/SuperClaude-Org/SuperClaude_Framework/blob/master/src/superclaude/mcp/MCP_Tavily.md) | 실시간 웹 검색 |

## 왜 이렇게 구성했는가
반복적인 컨텍스트 손실, 임시방편적 코드 수정, 불필요한 verbose 출력
같은 문제를 겪은 뒤, 작업을 태스크 단위로 쪼개고 검증 가능한 방식으로
진행하는 것을 우선했습니다. 위 항목들은 실제 세션 로그에 남은 사용
이력을 기준으로 정리한 것이며, 설정만 해두고 실행 기록이 없는 기능
(예: 일부 MCP 서버, 특정 리서치/분석 스킬)은 포함하지 않았습니다.
