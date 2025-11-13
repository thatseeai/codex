# Codex CLI 프로젝트 개요

## Codex CLI란?

**Codex CLI**는 OpenAI의 공식 코딩 에이전트로, 사용자의 컴퓨터에서 로컬로 실행됩니다. npm(`@openai/codex`) 및 Homebrew를 통해 배포되는 강력한 AI 기반 개발 도우미로, ChatGPT Plus, Pro, Team, Edu, Enterprise 사용자를 위해 설계되었습니다.

### 주요 기능

- 샌드박스 환경에서 코드 및 명령어 실행
- 코드베이스의 파일 변경 및 수정
- Git 저장소와 상호작용
- Model Context Protocol (MCP) 서버 연결
- 대화형 TUI (Terminal User Interface) 모드 제공
- 자동화를 위한 비대화형 exec 모드 지원
- TypeScript SDK를 통한 프로그래밍 방식 임베딩 가능

## 저장소 구조

**모노레포** 구조로, Rust/TypeScript 하이브리드 아키텍처를 사용합니다:

```
/home/user/codex/
├── codex-rs/           # Rust 워크스페이스 (메인 구현)
│   ├── cli/            # 메인 CLI 바이너리 진입점
│   ├── core/           # 비즈니스 로직 및 핵심 기능
│   ├── tui/            # 터미널 UI (Ratatui 기반)
│   ├── exec/           # 비대화형/헤드리스 실행 모드
│   ├── protocol/       # 통신 프로토콜 정의
│   ├── common/         # 공유 유틸리티
│   ├── app-server/     # IDE 통합을 위한 로컬 서버
│   ├── backend-client/ # 백엔드 API 클라이언트
│   ├── mcp-server/     # MCP 서버 구현
│   ├── mcp-types/      # MCP 타입 정의
│   ├── login/          # 인증 로직
│   ├── sandbox/        # 플랫폼별 샌드박싱
│   │   ├── linux-sandbox/
│   │   └── windows-sandbox-rs/
│   └── utils/          # 다양한 유틸리티 크레이트
├── codex-cli/          # NPM 래퍼 패키지
├── sdk/typescript/     # 프로그래밍 방식 사용을 위한 TypeScript SDK
├── docs/               # 사용자 문서
├── scripts/            # 빌드 및 릴리스 스크립트
└── .github/            # CI/CD 워크플로우
```

### Rust 워크스페이스 구조

`codex-rs/` 디렉토리는 41개의 워크스페이스 멤버 크레이트를 포함합니다:

**핵심 컴포넌트:**
- `cli` - 메인 CLI 바이너리 진입점 (`codex-rs/cli/src/main.rs`)
- `core` - 비즈니스 로직, 도구 실행, MCP 클라이언트, 설정 관리
- `tui` - 전체 화면 대화형 터미널 인터페이스 (Ratatui 기반)
- `exec` - 자동화를 위한 헤드리스 모드
- `protocol` - 프로토콜 정의

**보안 및 샌드박싱:**
- `linux-sandbox` - Linux 샌드박싱 (Landlock, seccomp)
- `windows-sandbox-rs` - Windows 샌드박싱 (제한된 토큰)
- macOS 샌드박싱은 Seatbelt 사용 (Core Foundation)
- `process-hardening` - 프로세스 강화 유틸리티

**통합:**
- `mcp-server` - MCP 서버 구현
- `mcp-types` - MCP 타입 정의
- `rmcp-client` - MCP 클라이언트
- `app-server` - IDE 통합을 위한 로컬 WebSocket 서버
- `backend-client` - 백엔드 API 클라이언트

**유틸리티:**
- `utils/git` - Git 작업
- `utils/cache` - 캐싱 유틸리티
- `utils/image` - 이미지 처리
- `utils/pty` - PTY 유틸리티
- `utils/tokenizer` - 토큰화
- `file-search` - 파일 검색
- `apply-patch` - 패치 적용

모든 크레이트 이름은 `codex-` 접두사를 사용합니다 (예: `core` 폴더의 크레이트는 `codex-core`).

## 기술 스택

### Rust 스택

**언어 및 에디션:**
- Rust 1.90.0 (2024 edition)
- `codex-rs/rust-toolchain.toml` 참조

**핵심 의존성:**
- **TUI 프레임워크**: Ratatui 0.29.0 (터미널 UI 렌더링)
- **터미널**: Crossterm 0.28.1 (크로스 플랫폼 터미널 조작)
- **비동기 런타임**: Tokio (멀티스레드 비동기 런타임)
- **HTTP 클라이언트**: Reqwest 0.12 (비동기 HTTP)
- **직렬화**: Serde, serde_json
- **설정**: TOML (toml 0.9.5, toml_edit 0.23.4)
- **MCP**: rmcp 0.8.5 (Model Context Protocol)
- **관찰성**: OpenTelemetry 0.30.0, tracing
- **샌드박싱**:
  - Linux: Landlock 0.4.1, seccompiler 0.5.0
  - macOS: Core Foundation (Seatbelt)
  - Windows: 사용자 정의 제한 토큰 구현

**테스트:**
- cargo-nextest (빠른 테스트 러너)
- insta 1.43.2 (스냅샷 테스트, 특히 TUI용)
- wiremock 0.6 (HTTP 모킹)
- pretty_assertions 1.4.1 (더 나은 assertion diff)

### TypeScript/Node.js 스택

**패키지 매니저 및 런타임:**
- **pnpm**: 10.8.1 (워크스페이스 패키지 매니저)
- **Node.js**: >=22
- **빌드 도구**: tsup 8.5.0 (TypeScript 번들러)
- **테스트**: Jest 29.7.0
- **린팅**: ESLint 9.36.0
- **포매팅**: Prettier 3.5.3
- **MCP SDK**: @modelcontextprotocol/sdk 1.20.2

### 빌드 도구

- **Just**: 태스크 러너 (`codex-rs/justfile` 참조)
- **cargo-nextest**: 빠른 병렬 테스트 실행
- **cargo-insta**: 스냅샷 테스트 도구
- **cargo-shear**: 사용하지 않는 의존성 감지
- **sccache**: CI를 위한 빌드 캐싱
- **GitHub Actions**: CI/CD 자동화

## 개발 워크플로우

### 빌드

**Rust:**
```bash
cd codex-rs
just codex          # CLI 실행
just fmt            # 코드 포매팅 (변경 후 자동 실행)
just fix -p <crate> # 특정 크레이트에 대한 린터 문제 수정
just test           # nextest로 테스트 실행
```

**TypeScript:**
```bash
pnpm install        # 의존성 설치
pnpm format         # 포매팅 확인
pnpm format:fix     # 포매팅 수정
```

### 테스트

**Rust 테스트:**
1. 특정 프로젝트 테스트 실행: `cargo test -p codex-<crate>`
2. common/core/protocol 변경 시: `cargo test --all-features`
3. 빠른 실행을 위해 `cargo nextest run` 사용
4. 스냅샷 테스트: `cargo insta accept -p codex-tui` (변경 사항 검토 후)

**테스트 유틸리티:**
- 통합 테스트는 `core_test_support::responses` 사용
- TUI 테스트는 스냅샷 테스트(insta) 사용
- `mount_sse*` 헬퍼로 SSE 응답 모킹
- 더 나은 diff를 위해 `pretty_assertions::assert_eq` 사용

**중요 테스트 참고사항:**
- 테스트는 네트워크 테스트를 건너뛰기 위해 `CODEX_SANDBOX_NETWORK_DISABLED_ENV_VAR` 확인
- Seatbelt 테스트는 중첩 샌드박싱을 피하기 위해 `CODEX_SANDBOX=seatbelt` 확인
- 이러한 환경 변수 검사를 절대 수정하지 말 것

### 코드 스타일

**Rust 관례:**
- if 문 축약 (clippy 규칙)
- 가능한 경우 format! 인자 인라인화
- 클로저보다 메서드 참조 사용
- 음수가 아니어도 부호 없는 정수는 절대 사용하지 않음
- 필드별 비교보다 전체 객체 비교 선호
- 모든 린트는 `codex-rs/Cargo.toml`의 `[workspace.lints.clippy]`에 설정

**TUI 스타일링 (Ratatui):**
- Stylize 트레잇 헬퍼 사용: `.dim()`, `.bold()`, `.cyan()`, `.italic()`
- 단순 span의 경우 `"text".into()` 선호
- 하드코딩된 `.white()` 피하기 - 기본 전경색 사용
- 텍스트 줄바꿈에 `textwrap::wrap` 사용
- 전체 규칙은 `codex-rs/tui/styles.md` 참조

## 설정

**사용자 설정:**
- 위치: `~/.codex/config.toml`
- 풍부한 TOML 기반 설정 시스템
- 전체 옵션은 `docs/config.md` 참조
- 예제: `docs/example-config.md`

**빌드 설정:**
- `codex-rs/Cargo.toml` - 워크스페이스 의존성 및 설정
- `codex-rs/clippy.toml` - Clippy 린팅 규칙
- `codex-rs/rustfmt.toml` - 포매팅 규칙
- `codex-rs/justfile` - 일반 빌드 명령어

**CI/CD 설정:**
- `.github/workflows/rust-ci.yml` - Rust CI 파이프라인
- `.github/workflows/rust-release.yml` - 릴리스 자동화
- `.github/workflows/sdk.yml` - TypeScript SDK CI

## 배포 및 릴리스

### 빌드 프로파일

**릴리스 프로파일** (`[profile.release]`):
- LTO: "fat" (링크 타임 최적화)
- Strip: "symbols" (디버그 심볼 제거)
- codegen-units: 1 (최적화를 위한 단일 코드젠 유닛)

**CI 테스트 프로파일** (`[profile.ci-test]`):
- debug: 1 (축소된 디버그 심볼)
- opt-level: 0 (빠른 컴파일)

### 지원 플랫폼

Codex는 8개 타겟 플랫폼용으로 빌드됩니다:
- **macOS**: aarch64-apple-darwin, x86_64-apple-darwin
- **Linux**: x86_64-unknown-linux-musl, x86_64-unknown-linux-gnu, aarch64-unknown-linux-musl, aarch64-unknown-linux-gnu
- **Windows**: x86_64-pc-windows-msvc, aarch64-pc-windows-msvc

### 배포 채널

1. **npm**: `@openai/codex` (네이티브 바이너리 래핑)
   - 설치: `npm install -g @openai/codex`
   - NPM 래퍼는 `codex-cli/`에 위치

2. **Homebrew**: `codex` (cask)
   - 설치: `brew install --cask codex`

3. **GitHub Releases**: 직접 바이너리 다운로드
   - `rust-v*.*.*` 형식으로 태그

### 릴리스 프로세스

1. `rust-v*.*.*`로 태그 (예: `rust-v0.1.0`)
2. CI가 태그와 `Cargo.toml` 버전 일치 확인
3. 8개 플랫폼 모두에 대한 크로스 컴파일 빌드
4. GitHub Releases에 바이너리 업로드
5. NPM 패키지 스테이징 및 게시

## 주요 기능

### 샌드박싱 모드

- **read-only**: 워크스페이스에 대한 읽기 전용 액세스
- **workspace-write**: 워크스페이스에만 쓰기 액세스
- **full-access**: 전체 시스템 액세스

플랫폼별 구현:
- Linux: Landlock LSM + seccomp BPF
- macOS: Seatbelt 샌드박스 프로파일
- Windows: 제한된 토큰

### 인증 방법

- ChatGPT SSO (권장)
- OpenAI API 키
- 디바이스 코드 플로우
- `docs/authentication.md` 참조

### MCP 통합

- MCP 클라이언트로 동작 (MCP 서버에 연결)
- MCP 서버로 동작 (다른 MCP 클라이언트가 사용 가능)
- `~/.codex/config.toml`에서 설정
- `docs/advanced.md#model-context-protocol-mcp` 참조

### 작동 모드

1. **대화형 TUI**: 전체 화면 터미널 인터페이스
   - 실행: `codex`
   - 위치: `codex-rs/tui/`

2. **Exec 모드**: 비대화형 자동화
   - 실행: `codex exec "<prompt>"`
   - 참조: `docs/exec.md`, `codex-rs/exec/`

3. **TypeScript SDK**: 프로그래밍 방식 API
   - 패키지: `@openai/codex-sdk`
   - 위치: `sdk/typescript/`

4. **App Server**: IDE 통합을 위한 로컬 WebSocket 서버
   - 위치: `codex-rs/app-server/`

## 중요 파일 및 진입점

### Rust 진입점

- `codex-rs/cli/src/main.rs` - 메인 CLI 바이너리 진입점
- `codex-rs/tui/src/main.rs` - TUI 진입점
- `codex-rs/exec/src/lib.rs` - Exec 모드 라이브러리

### TypeScript 진입점

- `sdk/typescript/src/index.ts` - SDK 공개 API
- `codex-cli/bin/codex.js` - NPM 래퍼 스크립트

### 문서

- `README.md` - 메인 프로젝트 README
- `AGENTS.md` - 개발 가이드라인 및 관례
- `docs/getting-started.md` - 시작 가이드
- `docs/config.md` - 설정 레퍼런스
- `docs/sandbox.md` - 샌드박스 및 승인
- `docs/authentication.md` - 인증 방법
- `docs/exec.md` - 비대화형 모드
- `docs/contributing.md` - 기여 가이드라인
- `docs/install.md` - 설치 및 빌드 지침
- `docs/faq.md` - 자주 묻는 질문

### 빌드 스크립트

- `scripts/stage_npm_packages.py` - NPM 패키지 스테이징
- `codex-cli/scripts/build_npm_package.py` - NPM 배포 빌드

## 개발 가이드라인 요약

전체 가이드라인은 `AGENTS.md` 참조. 핵심 사항:

- 모든 크레이트 이름에 `codex-` 접두사 사용
- 코드 변경 후 `just fmt` 실행 (승인 불필요)
- 변경 완료 전 `just fix -p <project>` 실행
- 프로젝트별 테스트에 `cargo test -p codex-<crate>` 사용
- 스냅샷 업데이트에 `cargo insta accept -p codex-<crate>` 사용
- 샌드박스 환경 변수 검사를 절대 수정하지 말 것
- Rust 2024 edition 관례 준수
- TUI 스타일링에 Ratatui Stylize 헬퍼 사용
- TUI의 텍스트 줄바꿈에 `textwrap` 사용

## 외부 리소스

- **메인 웹사이트**: https://chatgpt.com/codex
- **IDE 통합**: https://developers.openai.com/codex/ide
- **GitHub 저장소**: https://github.com/openai/codex
- **npm 패키지**: https://www.npmjs.com/package/@openai/codex
- **문서**: `docs/` 폴더 참조
- **이슈**: https://github.com/openai/codex/issues

## 아키텍처 참고사항

이것은 다음을 갖춘 프로덕션급 엔터프라이즈 도구입니다:
- 플랫폼별 정교한 보안 샌드박싱
- 포괄적인 테스트 커버리지 (단위, 통합, 스냅샷)
- 멀티 플랫폼 지원 (macOS, Linux, Windows, ARM)
- 모던 비동기 Rust 아키텍처 (Tokio)
- Ratatui를 사용한 풍부한 TUI
- 확장 가능한 MCP 통합
- 임베딩을 위한 TypeScript SDK
- 전문적인 릴리스 자동화

코드베이스는 엄격한 clippy 규칙, UI용 스냅샷 테스트, 포괄적인 CI/CD 워크플로우를 통해 Rust 개발 모범 사례를 따릅니다.
