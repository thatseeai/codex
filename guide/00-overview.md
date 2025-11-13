# Codex CLI 내부 아키텍처 가이드 - 개요

## 문서 소개

이 가이드는 Codex CLI의 내부 동작 방식을 시나리오 기반 이야기 형식으로 설명합니다. 각 시나리오는 실제 사용 케이스를 따라가며, 코드가 어떻게 실행되고 컴포넌트들이 어떻게 상호작용하는지 보여줍니다.

## 아키텍처 개요

Codex CLI는 Rust/TypeScript 하이브리드 모노레포 구조로, 다음과 같은 핵심 계층으로 구성됩니다:

```
사용자 인터페이스 계층
    ↓
대화 관리 계층
    ↓
에이전트 실행 계층
    ↓
도구 실행 계층
    ↓
샌드박싱 계층
```

### 주요 컴포넌트

1. **CLI 진입점** (`codex-rs/cli/src/main.rs`)
   - 명령줄 파싱 및 라우팅
   - 서브커맨드 디스패치

2. **TUI (Terminal User Interface)** (`codex-rs/tui/`)
   - Ratatui 기반 전체 화면 대화형 인터페이스
   - 실시간 이벤트 스트리밍 및 렌더링

3. **Exec 모드** (`codex-rs/exec/`)
   - 비대화형 헤드리스 모드
   - 자동화 워크플로우 지원

4. **Core** (`codex-rs/core/`)
   - **Codex**: 메인 에이전트 로직
   - **ConversationManager**: 대화 생명주기 관리
   - **Session**: 세션 상태 및 서비스 관리
   - **ToolRouter**: 도구 실행 라우팅

5. **샌드박싱** (`codex-rs/sandbox/`)
   - Linux: Landlock + seccomp
   - macOS: Seatbelt
   - Windows: Restricted Token

6. **MCP 통합** (`codex-rs/mcp-server/`, `codex-rs/rmcp-client/`)
   - MCP 클라이언트/서버 구현
   - 외부 도구 및 리소스 통합

## 주요 데이터 흐름

### 1. 명령 실행 흐름

```
사용자 입력 (CLI/TUI)
    ↓
Op (Operation) 생성
    ↓
Codex Session 처리
    ↓
ModelClient를 통한 LLM 호출
    ↓
ResponseItem 스트리밍
    ↓
ToolRouter를 통한 도구 실행
    ↓
SandboxManager를 통한 샌드박스 실행
    ↓
결과를 Event로 반환
    ↓
UI 업데이트 (TUI) 또는 출력 (Exec)
```

### 2. 대화 상태 관리

```
InitialHistory (새 대화 또는 재개)
    ↓
Codex::spawn
    ↓
Session 초기화
    ↓
RolloutRecorder 설정 (히스토리 저장)
    ↓
대화 진행 중 상태 업데이트
    ↓
Rollout 파일에 지속적 저장
```

### 3. 인증 흐름

```
CLI 시작
    ↓
Config 로드
    ↓
AuthManager 초기화
    ↓
인증 상태 확인
    - ChatGPT SSO
    - OpenAI API Key
    - Device Code Flow
    ↓
필요시 온보딩 화면 표시
    ↓
토큰 갱신 (자동)
```

## 시나리오별 가이드

각 시나리오는 별도의 문서로 제공됩니다:

1. **[CLI 시작 및 인증](01-cli-startup-and-authentication.md)**
   - 프로그램 진입점부터 인증 완료까지

2. **[대화형 TUI 모드](02-tui-interactive-mode.md)**
   - TUI 초기화, 사용자 입력 처리, 실시간 렌더링

3. **[Exec 비대화형 모드](03-exec-non-interactive-mode.md)**
   - 헤드리스 실행, JSONL 출력, 자동화 워크플로우

4. **[대화 생명주기](04-conversation-lifecycle.md)**
   - 대화 생성, 재개, 포크, 히스토리 관리

5. **[도구 실행 및 샌드박싱](05-tool-execution-and-sandboxing.md)**
   - Bash 실행, 파일 읽기/쓰기, 샌드박스 보안

6. **[MCP 통합](06-mcp-integration.md)**
   - MCP 서버 연결, 도구/리소스 통합

7. **[파일 작업 및 Rollout](07-file-operations-and-rollout.md)**
   - 파일 변경 추적, Rollout 저장, 세션 복원

## 코드 탐색 가이드

### 주요 진입점

- **CLI 메인**: `codex-rs/cli/src/main.rs::cli_main()`
- **TUI 메인**: `codex-rs/tui/src/lib.rs::run_main()`
- **Exec 메인**: `codex-rs/exec/src/lib.rs::run_main()`
- **Codex 스폰**: `codex-rs/core/src/codex.rs::Codex::spawn()`

### 핵심 타입

- **Event**: 에이전트에서 UI로의 이벤트 (`codex-rs/protocol/src/protocol.rs`)
- **Op**: UI에서 에이전트로의 작업 (`codex-rs/protocol/src/protocol.rs`)
- **Config**: 설정 (`codex-rs/core/src/config/mod.rs`)
- **Session**: 세션 상태 (`codex-rs/core/src/state/`)

### 테스트

- 단위 테스트: 각 크레이트의 `tests/` 디렉토리
- 통합 테스트: `codex-rs/core/tests/`
- TUI 스냅샷 테스트: `codex-rs/tui/tests/` (insta 사용)

## 다음 단계

각 시나리오 문서를 순서대로 읽으면서 Codex의 내부 동작을 이해하세요. 각 문서는 실제 사용자 워크플로우를 따라가며, 코드 레퍼런스와 함께 상세한 설명을 제공합니다.
