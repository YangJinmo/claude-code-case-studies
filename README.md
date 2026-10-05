# Claude Code Case Studies

Claude Code를 실제로 어떻게 쓰고 있는지, **세션 로그로 확인된 사례만** 정리한
레포입니다. 설정 파일과 직접 만든 Skills/Commands는
[superclaude-config](https://github.com/YangJinmo/superclaude-config)에 있습니다.

사례의 호출 횟수·서브에이전트 수는 모두 세션 로그(`~/.claude/projects/*.jsonl`)의
`tool_use` 기록을 집계한 값이고, 설정만 해두고 호출 기록이 없는 기능은 사례에서
뺐습니다.

## 실증된 사용 사례

### 1. 구조화된 작업 관리 (Task 도구)
**2026-07-22 · PartMoney (Swift iOS 앱)**

모놀리식 구조를 Clean Architecture + MVVM으로 리팩터링하는 작업을 27개 태스크로
쪼개 진행 (`TaskCreate`/`TaskUpdate` 79회).

- 계층(Domain → Data → DI) 단위로 나눠 각 단계마다 빌드 가능한 상태를 유지
- 리팩터링 후 실사용 중 발견된 버그 7건 수정 (알림 race condition, 연도 경계 계산 오류 등)
- Domain/UseCase 유닛 테스트 추가

### 2. 멀티 에이전트 병렬 코드 리뷰
**2026-08-07 · PartMoney 외 1개 프로젝트**

`/code-review`로 정확성·단순화·효율 등 8개 관점의 서브에이전트를 병렬로 돌려 후보를
찾고, 검증 서브에이전트로 교차 확인해 결과를 취합 (세션당 서브에이전트 15개).
유튜브 자막 추출 스크립트 리뷰에서 실제 버그 10건을 찾았고, 예를 들면:

- 자막 언어 코드를 검증하지 않아 잘못된 언어 자막을 반환할 수 있음
- `subprocess.run`에 `timeout`이 없어 외부 프로세스가 멈추면 무한 대기

### 3. 커스텀 스킬 제작과 실사용 기반 개선
**2026-08-07 ~ 2026-10-05**

- **`tube-info`**: 직접 작성한 유튜브 정보·자막 추출 스크립트를 `skill-creator`로
  재사용 가능한 스킬로 전환
- **`app-mockup`** (9개 세션): 앱 스크린샷을 기기 프레임에 합성해 투명 배경
  1920x1080 목업으로 만드는 스킬. 쓰다가 나온 문제를 바로 스킬에 반영해 다듬어 옴:
  - 녹화 표시를 지우다 근처 빨간 앱 로고까지 뭉개짐 → 모양 기준 탐지로 스크립트 수정
  - 마젠타 키잉으로 만든 투명 배경에 핑크 얼룩 → 헤드리스 Chrome의 실제 알파 렌더링으로 교체
  - 프레임 안쪽 모서리에 회색 라인 → 픽셀 값을 찍어 흰 배경의 안티앨리어싱 번짐이
    원인임을 확인하고 수정

### 4. Playwright MCP 브라우저 자동화
**2026-08-27**

코드를 고친 뒤 로컬 개발 서버를 실제 브라우저로 열어 클릭·콘솔 확인을 반복하며
동작을 검증 (`browser_*` 25회).

### 5. 설계 → 계획 → 서브에이전트 구현 사이클
**2026-09-12 · [SecureAuthKit](https://github.com/YangJinmo/SecureAuthKit) (iOS 인증 SDK 예제)**

Superpowers 스킬(`brainstorming` → `writing-plans` → `subagent-driven-development`)로
아이디어부터 PR까지 한 세션에서 진행.

- 설계 스펙과 8개 태스크 구현 계획을 문서로 남긴 뒤, 태스크마다 구현·리뷰
  서브에이전트를 붙여 진행 (서브에이전트 19개, iOS 시뮬레이터 도구 66회)
- 마지막 전체 리뷰에서 태스크별 리뷰로는 안 보이던 통합 버그 9건을 찾아 수정
  (재실행 후 토큰 리프레시 실패, 빠져나갈 수 없는 생체인증 잠금 화면 등)
- SDK 테스트 27/27 통과, 시뮬레이터 시나리오별 스크린샷과 함께
  [PR #1](https://github.com/YangJinmo/SecureAuthKit/pull/1)로 정리

## 구성은 됐지만 아직 사용 안 한 기능

매 세션 로드되지만 세션 로그상 실제로 쓰인 기록이 없는 기능입니다.

**Behavioral Modes** — [Introspection](https://github.com/SuperClaude-Org/SuperClaude_Framework/blob/master/src/superclaude/modes/MODE_Introspection.md),
[Deep Research](https://github.com/SuperClaude-Org/SuperClaude_Framework/blob/master/src/superclaude/modes/MODE_DeepResearch.md),
[Token Efficiency](https://github.com/SuperClaude-Org/SuperClaude_Framework/blob/master/src/superclaude/modes/MODE_Token_Efficiency.md),
[Orchestration](https://github.com/SuperClaude-Org/SuperClaude_Framework/blob/master/src/superclaude/modes/MODE_Orchestration.md),
[Business Panel](https://github.com/SuperClaude-Org/SuperClaude_Framework/blob/master/src/superclaude/modes/MODE_Business_Panel.md),
[Brainstorming](https://github.com/SuperClaude-Org/SuperClaude_Framework/blob/master/src/superclaude/modes/MODE_Brainstorming.md)
(같은 역할은 Superpowers `brainstorming` 스킬로 쓰는 중 — 사례 5)

**MCP 서버** — [Context7](https://github.com/SuperClaude-Org/SuperClaude_Framework/blob/master/src/superclaude/mcp/MCP_Context7.md),
[Sequential](https://github.com/SuperClaude-Org/SuperClaude_Framework/blob/master/src/superclaude/mcp/MCP_Sequential.md),
[Serena](https://github.com/SuperClaude-Org/SuperClaude_Framework/blob/master/src/superclaude/mcp/MCP_Serena.md),
[Morphllm](https://github.com/SuperClaude-Org/SuperClaude_Framework/blob/master/src/superclaude/mcp/MCP_Morphllm.md),
[Magic](https://github.com/SuperClaude-Org/SuperClaude_Framework/blob/master/src/superclaude/mcp/MCP_Magic.md),
[Tavily](https://github.com/SuperClaude-Org/SuperClaude_Framework/blob/master/src/superclaude/mcp/MCP_Tavily.md)
