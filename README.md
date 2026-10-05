# Claude Code Case Studies

Claude Code를 실제로 어떻게 쓰고 있는지, **세션 로그로 확인된 사례만** 정리한
레포입니다. 설정만 해두고 실행 기록이 없는 기능은 사례에 넣지 않고, 아래
"구성은 됐지만 아직 사용 안 한 기능"에 따로 모아뒀습니다.

매 세션 로드되는 설정 파일(`CLAUDE.md`, 규칙·모드·MCP 라우팅 문서)과 직접 만든
Skills/Commands는 [superclaude-config](https://github.com/YangJinmo/superclaude-config)에
있습니다. 이 레포는 그 설정을 **어떻게 썼는가**만 다룹니다.

## 검증 방법

Claude Code는 세션마다 대화 전체를 `~/.claude/projects/<프로젝트 경로>/<세션 ID>.jsonl`에
남깁니다. 각 줄의 `tool_use` 블록(도구 이름·입력)을 세서 "실제로 호출됐는가"를
판단했고, 사례에 적은 호출 횟수·서브에이전트 수·커밋 내용은 모두 이 로그에서
집계한 값입니다. 설정 문서에 적혀 있다는 것만으로는 사용 사례로 치지 않았습니다.

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

### 3. 커스텀 스킬 제작과 실사용 기반 개선
**2026-08-07 ~ 2026-10-05**

- **`tube-info` (08-07)**: 유튜브 URL을 넣으면 제목/조회수/자막/타임라인을 추출하는
  스크립트를 직접 작성한 뒤, `skill-creator`로 재사용 가능한 스킬로 전환
- **`karpathy-guidelines` (08-26)**: 과설계 방지 가이드라인 스킬이 의도대로 트리거되는지
  동작 테스트 (실제 코드 리뷰 적용 사례는 아직 없음)
- **`app-mockup` (08-26 ~ 10-05, 9개 세션)**: 앱 스크린샷을 아이폰/갤럭시 기기 프레임에
  합성해 배경 투명 1920x1080 이미지로 만드는 스킬. Somvely, 엄선, Hoogi, BuyeoTravel 등
  여러 앱의 홍보용 목업을 이 스킬로 만들었고, **쓰다가 나온 문제를 그때그때 스킬 문서와
  스크립트에 되먹임**하는 식으로 다듬어 왔습니다:
  - iOS 화면 녹화 빨간 캡슐을 지우다가 바로 아래 빨간 앱 로고("엄선") 위쪽까지 같이
    뭉개짐 → 색만 보지 않고 연결요소 분석으로 "화면 맨 위에 붙은 알약 모양" 덩어리만
    지우도록 스크립트 수정, 처리 후 로고 영역까지 확대 비교하는 검증 단계 추가
  - 투명 배경을 마젠타 키잉으로 만들었더니 둥근 모서리·그림자 경계에 핑크 얼룩이 남음 →
    Chrome 헤드리스의 진짜 알파 채널 렌더링(`--default-background-color=00000000`)으로 교체
  - "프레임 적용 안 함"을 골랐는데 둥근 모서리·그림자가 들어가 재작업 → 이 옵션의 기본값을
    "어떤 스타일링도 추가하지 않음"으로 문서에 명시
  - (10-05) CSS 프레임 안쪽 모서리를 따라 밝은 회색 라인 발생 → 렌더 결과 픽셀 값을
    직접 찍어 화면 영역의 흰 배경이 둥근 경계의 안티앨리어싱 픽셀로 섞여 비치는 것이
    원인임을 확인, 배경을 베젤 색으로 바꾸고 모든 치수를 정수 px로 반올림해 해결
  - 실물 기기 프레임 PNG(Design at Meta)의 라이선스를 확인해 재배포 금지 조항 때문에
    공개 저장소에는 올리지 않도록 `.gitignore`로 제외

### 4. Playwright MCP 브라우저 자동화
**2026-08-27**

로컬 개발 서버(`http://127.0.0.1:4899`)를 대상으로 하루 동안 3차례에 걸쳐
`browser_navigate` → `browser_snapshot`/`browser_console_messages` → `browser_click` →
`browser_evaluate` 흐름을 반복 실행 (총 25회 호출). 코드 수정 후 실제 브라우저에서
동작을 재현·확인하며 반복 검증하는 용도로 사용.

### 5. 설계 → 계획 → 서브에이전트 구현까지 이어지는 개발 사이클
**2026-09-12 · [SecureAuthKit](https://github.com/YangJinmo/SecureAuthKit) (iOS 인증 SDK 예제)**

Superpowers 스킬 체인(`brainstorming` → `writing-plans` → `using-git-worktrees` →
`subagent-driven-development` → `finishing-a-development-branch`)으로 아이디어
단계부터 PR까지 한 세션에서 진행. 생체인증 + Keychain 토큰 저장 SDK와 SwiftUI 데모 앱.

1. **설계**: 질문으로 방향을 좁혀 A안 선택 → 설계 스펙 문서 작성·커밋
2. **계획**: 8개 태스크(모델 → Keychain 저장소 → Mock 프로바이더 → 생체인증 →
   AuthSession 파사드 → 데모 앱 스캐폴드 → 뷰 레이어 → README)로 구현 계획 작성.
   이 과정에서 스펙에는 있지만 계획에서 빠진 `canAuthenticate()` 가드를 발견해 계획에 반영
3. **구현**: git worktree에서 태스크마다 구현 서브에이전트 1개 + 리뷰 서브에이전트 1개
   (스펙 일치·코드 품질)를 붙여 진행, 마지막에 브랜치 전체 리뷰 → 수정 → 재리뷰
   (서브에이전트 총 19개 디스패치, iOS 시뮬레이터 도구 66회 호출)
4. **최종 리뷰에서 잡은 것**: 태스크 단위 리뷰로는 안 보이던 교차 태스크 통합 버그 9건.
   재실행 후 토큰 리프레시 실패, 생체인증 미등록 시 빠져나갈 수 없는 잠금 화면,
   뷰모델 메모리 누수 등. 초기에 "정상 처리"로 기록했던 잠금 화면 동작이 실제로는
   탈출구 없는 버그였다는 것도 이 단계에서 정정됨
5. **결과**: SDK 테스트 27/27 통과, 서드파티 런타임 의존성 없음. 시뮬레이터에서
   로그인 → 잘못된 비밀번호 → 재실행 시 생체인증 잠금 → 로그아웃 흐름을 직접 눌러보며
   스크린샷으로 남기고, 자동화가 안 되는 시나리오(Face ID 매칭, 토큰 만료)는
   재현 절차를 문서화해 [PR #1](https://github.com/YangJinmo/SecureAuthKit/pull/1)로 올림

## 로드 방식: 자동 vs 조건부 vs 명시적 호출

"설정은 됐는데 왜 실행 이력이 없는 항목이 있는가"를 이해하려면
[superclaude-config](https://github.com/YangJinmo/superclaude-config)의 파일들이
세 가지 다른 방식으로 작동한다는 걸 구분해야 합니다.

### 1. 항상 자동으로 로드됨 — 별도 호출 불필요

`CLAUDE.md`가 세션 시작 시 나머지 파일을 전부 `@import`하기 때문에,
아래는 매 세션 시작할 때마다 자동으로 컨텍스트에 실립니다.

- `RULES.md`, `PRINCIPLES.md`, `FLAGS.md`, `RESEARCH_CONFIG.md`
- `MODE_*.md` 전부
- `MCP_*.md` 전부

이 안에서도 성격이 두 가지로 나뉩니다.

**항상 적용되는 규칙** — `RULES.md`, `PRINCIPLES.md`. "Git status 먼저 확인",
"쓰기 전에 읽기", "근거 없는 주장 금지" 같이 조건 없이 매 세션 지키려고
하는 행동 규칙입니다.

**조건이 맞으면 자동으로 전환되는 모드/라우팅** — `MODE_*.md`, `MCP_*.md`.
각 파일에는 "Activation Triggers"가 정의돼 있고 (예: "모호한 요청 →
Brainstorming", "브라우저 테스트 필요 → Playwright"), 대화 맥락에서 그
조건이 감지되면 Claude가 스스로 판단해서 해당 모드/서버로 전환합니다.
아래 "구성은 됐지만 아직 사용 안 한 기능"의 항목들도 트리거 자체는 매 세션
감시되고 있었지만, 실제로 조건이 뚜렷하게 걸린 적이 없었던 것뿐입니다.

### 2. 명시적으로 호출해야 작동함

- **Skills** — 컨텍스트에 자동으로 실리지 않습니다. 요청 내용이 스킬 설명과
  매칭되거나 `/이름`을 직접 입력해야 Claude가 해당 스킬 파일을 불러와 그
  지침을 따릅니다.
- **Commands** — 슬래시 명령을 직접 입력해야 실행됩니다.
- **MCP 서버의 실제 도구 호출** — `MCP_*.md`의 라우팅 규칙은 항상 로드돼
  있지만, 실제로 그 서버의 도구를 호출할지는 매번 그때그때 판단하는
  런타임 결정이고, 서버가 연결돼 있어야 합니다 (연결이 끊기면 라우팅
  규칙이 있어도 호출 자체가 불가능).

## 구성은 됐지만 아직 사용 안 한 기능

설정은 돼 있지만 세션 로그상 실행 이력이 없는 기능입니다. 뭘 더 써볼 수
있는지 참고용으로 남겨둡니다.

### Behavioral Modes

| 모드 | 용도 |
|---|---|
| [Brainstorming](https://github.com/SuperClaude-Org/SuperClaude_Framework/blob/master/src/superclaude/modes/MODE_Brainstorming.md) | 모호한 요청에 질문을 던져 요구사항을 구체화 (같은 역할을 Superpowers `brainstorming` 스킬이 대신 수행 — 사례 5) |
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

## 왜 이렇게 정리했는가

반복적인 컨텍스트 손실, 임시방편적 코드 수정, 불필요한 verbose 출력
같은 문제를 겪은 뒤, 작업을 태스크 단위로 쪼개고 검증 가능한 방식으로
진행하는 것을 우선했습니다. 설정을 많이 붙여두는 것과 실제로 쓰는 것은
다르기 때문에, 이 레포는 설정 목록이 아니라 세션 로그에 남은 사용 이력을
기준으로 정리합니다.
