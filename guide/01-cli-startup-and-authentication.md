# 시나리오 1: CLI 시작 및 인증

## 시나리오 개요

사용자가 터미널에서 `codex` 명령을 처음 실행할 때 어떤 일이 일어나는지 추적합니다. 프로그램 진입점부터 설정 로딩, 인증 확인, 그리고 온보딩 화면(필요한 경우)까지의 전체 흐름을 다룹니다.

## 이야기로 따라가는 실행 흐름

### 1단계: 프로그램 시작

사용자가 터미널에서 `codex "write a hello world program"` 명령을 실행합니다.

**진입점**: `codex-rs/cli/src/main.rs::main()`

```rust
fn main() -> anyhow::Result<()> {
    arg0_dispatch_or_else(|codex_linux_sandbox_exe| async move {
        cli_main(codex_linux_sandbox_exe).await?;
        Ok(())
    })
}
```

**동작**:
1. `arg0_dispatch_or_else`는 Linux에서 샌드박스 바이너리로 재실행되었는지 확인
2. 일반 실행이면 `cli_main`으로 진행

**위치**: `codex-rs/cli/src/main.rs:391-396`

### 2단계: 명령줄 파싱

`cli_main` 함수에서 clap을 사용하여 명령줄 인자를 파싱합니다.

```rust
async fn cli_main(codex_linux_sandbox_exe: Option<PathBuf>) -> anyhow::Result<()> {
    let MultitoolCli {
        config_overrides: mut root_config_overrides,
        feature_toggles,
        mut interactive,
        subcommand,
    } = MultitoolCli::parse();

    // feature toggles를 config overrides로 변환
    let toggle_overrides = feature_toggles.to_overrides()?;
    root_config_overrides.raw_overrides.extend(toggle_overrides);

    match subcommand {
        None => {
            // 서브커맨드가 없으면 대화형 TUI 모드
            prepend_config_flags(&mut interactive.config_overrides, root_config_overrides.clone());
            let exit_info = codex_tui::run_main(interactive, codex_linux_sandbox_exe).await?;
            handle_app_exit(exit_info)?;
        }
        Some(Subcommand::Exec(mut exec_cli)) => { /* ... */ }
        Some(Subcommand::Login(mut login_cli)) => { /* ... */ }
        // ... 기타 서브커맨드
    }
    Ok(())
}
```

**동작**:
1. `MultitoolCli` 구조체로 모든 CLI 옵션 파싱
2. `--enable`/`--disable` 플래그를 config override로 변환
3. 서브커맨드 유무에 따라 라우팅:
   - 없으면: 대화형 TUI 모드 (`codex_tui::run_main`)
   - `exec`: 비대화형 모드
   - `login`: 인증 관리
   - 기타: 각 서브커맨드 핸들러

**위치**: `codex-rs/cli/src/main.rs:398-419`

### 3단계: TUI 모드 시작 - 설정 로딩

대화형 모드가 선택되었으므로 `codex_tui::run_main`으로 진입합니다.

**진입점**: `codex-rs/tui/src/lib.rs::run_main()`

```rust
pub async fn run_main(
    mut cli: Cli,
    codex_linux_sandbox_exe: Option<PathBuf>,
) -> std::io::Result<AppExitInfo> {
    // CLI 플래그 처리 (full_auto, sandbox_mode 등)
    let (sandbox_mode, approval_policy) = if cli.full_auto {
        (Some(SandboxMode::WorkspaceWrite), Some(AskForApproval::OnRequest))
    } else if cli.dangerously_bypass_approvals_and_sandbox {
        (Some(SandboxMode::DangerFullAccess), Some(AskForApproval::Never))
    } else {
        (
            cli.sandbox_mode.map(Into::<SandboxMode>::into),
            cli.approval_policy.map(Into::into),
        )
    };

    // Config overrides 구성
    let overrides = ConfigOverrides {
        model,
        sandbox_mode,
        approval_policy,
        cwd,
        // ... 기타 옵션
    };

    // Config 로드
    let config = load_config_or_exit(cli_kv_overrides.clone(), overrides.clone()).await;

    // ...
}
```

**동작**:
1. CLI 플래그를 `ConfigOverrides`로 변환
2. `load_config_or_exit`로 설정 로드:
   - `~/.codex/config.toml` 읽기
   - 프로필 병합
   - CLI override 적용
3. 설정 로딩 실패 시 에러 메시지 출력하고 종료

**위치**: `codex-rs/tui/src/lib.rs:98-176`

### 4단계: 로깅 및 관찰성 설정

설정이 로드되면 로깅과 OpenTelemetry를 초기화합니다.

```rust
// 로그 파일 생성 (~/.codex/logs/codex-tui.log)
let log_dir = codex_core::config::log_dir(&config)?;
std::fs::create_dir_all(&log_dir)?;
let log_file = log_file_opts.open(log_dir.join("codex-tui.log"))?;

// tracing subscriber 설정
let (non_blocking, _guard) = non_blocking(log_file);
let file_layer = tracing_subscriber::fmt::layer()
    .with_writer(non_blocking)
    .with_filter(env_filter());

// OpenTelemetry 설정
let otel = codex_core::otel_init::build_provider(&config, env!("CARGO_PKG_VERSION"));
```

**동작**:
1. `~/.codex/logs/` 디렉토리 생성
2. `codex-tui.log` 파일 생성 (권한 0600)
3. tracing subscriber 레이어 설정:
   - 파일 레이어 (로그 파일)
   - 피드백 레이어 (UI 표시용)
   - OpenTelemetry 레이어 (원격 관찰)

**위치**: `codex-rs/tui/src/lib.rs:193-268`

### 5단계: 인증 상태 확인

TUI 초기화 전에 인증 상태를 확인합니다.

```rust
let auth_manager = AuthManager::shared(
    initial_config.codex_home.clone(),
    false,
    initial_config.cli_auth_credentials_store_mode,
);

let login_status = get_login_status(&initial_config);

fn get_login_status(config: &Config) -> LoginStatus {
    if config.model_provider.requires_openai_auth {
        let codex_home = config.codex_home.clone();
        match CodexAuth::from_auth_storage(&codex_home, config.cli_auth_credentials_store_mode) {
            Ok(Some(auth)) => LoginStatus::AuthMode(auth.mode),
            Ok(None) => LoginStatus::NotAuthenticated,
            Err(err) => {
                error!("Failed to read auth.json: {err}");
                LoginStatus::NotAuthenticated
            }
        }
    } else {
        LoginStatus::NotAuthenticated
    }
}
```

**동작**:
1. `AuthManager::shared()` 생성 - 인증 매니저 싱글톤
2. `~/.codex/auth.json` 파일 읽기 시도
3. 인증 상태 판별:
   - `AuthMode::ChatGptSso`: ChatGPT SSO 인증됨
   - `AuthMode::ApiKey`: API Key 인증됨
   - `NotAuthenticated`: 인증 안됨

**위치**: `codex-rs/tui/src/lib.rs:329-515`

### 6단계: 온보딩 화면 결정

인증 상태와 기타 요소를 바탕으로 온보딩 화면 표시 여부를 결정합니다.

```rust
let should_show_trust_screen = should_show_trust_screen(&initial_config);
let should_show_windows_wsl_screen =
    cfg!(target_os = "windows") && !initial_config.windows_wsl_setup_acknowledged;

let should_show_onboarding = should_show_onboarding(
    login_status,
    &initial_config,
    should_show_trust_screen,
    should_show_windows_wsl_screen,
);

fn should_show_trust_screen(config: &Config) -> bool {
    if cfg!(target_os = "windows") && get_platform_sandbox().is_none() {
        return false; // Windows에서 샌드박스 비활성화시 스킵
    }
    if config.did_user_set_custom_approval_policy_or_sandbox_mode {
        return false; // 사용자가 명시적 설정한 경우 스킵
    }
    !config.active_project.is_trusted() // 프로젝트가 신뢰되지 않은 경우만
}
```

**온보딩 화면 표시 조건**:
1. **Windows WSL 화면**: Windows에서 아직 WSL 설정 안내를 보지 않음
2. **Trust 화면**: 현재 디렉토리가 신뢰되지 않고, 사용자가 샌드박스 정책을 명시하지 않음
3. **Login 화면**: 인증되지 않았고, 모델 프로바이더가 OpenAI auth 필요

**위치**: `codex-rs/tui/src/lib.rs:534-572`

### 7단계: 온보딩 화면 실행 (필요한 경우)

온보딩이 필요하면 `run_onboarding_app`을 실행합니다.

```rust
if should_show_onboarding {
    let onboarding_result = run_onboarding_app(
        OnboardingScreenArgs {
            show_login_screen: should_show_login_screen(login_status, &initial_config),
            show_windows_wsl_screen,
            show_trust_screen,
            login_status,
            auth_manager: auth_manager.clone(),
            config: initial_config.clone(),
        },
        &mut tui,
    )
    .await?;

    if onboarding_result.should_exit {
        restore();
        return Ok(AppExitInfo::default());
    }

    // Trust 결정이 있으면 config 재로드
    if onboarding_result.directory_trust_decision.is_some() {
        load_config_or_exit(cli_kv_overrides, overrides).await
    } else {
        initial_config
    }
}
```

**온보딩 화면 흐름**:

#### a. Windows WSL 화면 (Windows only)
- WSL 사용 권장 메시지
- "Continue Anyway" 또는 "Exit" 선택
- 선택을 `~/.codex/config.toml`에 저장

#### b. Login 화면
1. **ChatGPT SSO** (기본 권장):
   - 브라우저 URL 표시
   - 사용자가 브라우저에서 인증
   - 폴링하여 토큰 수신
   - `~/.codex/auth.json`에 저장

2. **Device Code Flow**:
   - Device code 표시
   - 사용자가 코드 입력
   - 폴링하여 토큰 수신

3. **API Key**:
   - stdin에서 API key 읽기
   - `~/.codex/auth.json`에 저장

#### c. Trust 화면
- 현재 디렉토리 정보 표시
- Git 저장소 여부 확인
- 선택지:
  - **Trust this directory**: `~/.codex/config.toml`에 추가
  - **Ask for each action**: approval_policy 설정
  - **Exit**: 프로그램 종료

**위치**: `codex-rs/tui/src/onboarding/`

### 8단계: 인증 토큰 저장 구조

인증이 완료되면 다음 형식으로 저장됩니다.

**파일**: `~/.codex/auth.json`

```json
{
  "mode": "ChatGptSso",
  "data": {
    "access_token": "eyJhbGc...",
    "refresh_token": "...",
    "expires_at": 1234567890
  }
}
```

또는 API Key 모드:

```json
{
  "mode": "ApiKey",
  "data": {
    "api_key": "sk-..."
  }
}
```

**AuthManager 동작**:
- `AuthManager::shared()`: 프로세스당 하나의 인스턴스
- 토큰 만료 시 자동 갱신 (ChatGPT SSO)
- 스레드 안전 (Arc + RwLock)

**위치**: `codex-rs/core/src/auth/mod.rs`

### 9단계: 설정 최종화

온보딩 완료 후 최종 설정이 확정됩니다.

```rust
// Config 최종 로드 (trust 결정 반영)
let config = if should_show_onboarding {
    // ... onboarding 결과에 따라 reload 또는 initial_config 사용
} else {
    initial_config
};

// 설정 검증
if let Err(err) = enforce_login_restrictions(&config).await {
    eprintln!("{err}");
    std::process::exit(1);
}
```

**최종 Config 내용**:
- `model`: 사용할 모델 (예: "gpt-4-turbo")
- `model_provider_id`: 프로바이더 ID (예: "openai")
- `approval_policy`: 승인 정책 (OnRequest, OnDanger, Never)
- `sandbox_policy`: 샌드박스 정책 (ReadOnly, WorkspaceWrite, FullAccess)
- `cwd`: 작업 디렉토리
- `codex_home`: `~/.codex` 경로
- `features`: 기능 플래그 맵
- `mcp_servers`: MCP 서버 설정 배열

**위치**: `codex-rs/tui/src/lib.rs:382-394`

### 10단계: 메인 App으로 진입

모든 초기화가 완료되면 실제 대화형 앱으로 진입합니다.

```rust
let app_result = App::run(
    &mut tui,
    auth_manager,
    config,
    active_profile,
    prompt,
    images,
    resume_selection,
    feedback,
)
.await;

restore();
session_log::log_session_end();
app_result
```

**App::run으로 전달되는 것**:
- `tui`: 터미널 인터페이스
- `auth_manager`: 인증 매니저
- `config`: 최종 설정
- `prompt`: CLI에서 전달된 초기 프롬프트 (있는 경우)
- `resume_selection`: 재개할 세션 (있는 경우)

**이후 흐름**:
- `App::run`에서 `ConversationManager` 생성
- 새 대화 시작 또는 기존 대화 재개
- 사용자 입력 처리 루프 진입

**위치**: `codex-rs/tui/src/lib.rs:462-478`

## 주요 코드 위치 요약

| 단계 | 함수 | 파일 | 라인 |
|-----|------|------|------|
| 진입점 | `main()` | `codex-rs/cli/src/main.rs` | 391-396 |
| CLI 파싱 | `cli_main()` | `codex-rs/cli/src/main.rs` | 398-595 |
| TUI 시작 | `run_main()` | `codex-rs/tui/src/lib.rs` | 98-280 |
| 설정 로딩 | `Config::load_with_cli_overrides()` | `codex-rs/core/src/config/mod.rs` | - |
| 인증 확인 | `get_login_status()` | `codex-rs/tui/src/lib.rs` | 499-515 |
| 온보딩 | `run_onboarding_app()` | `codex-rs/tui/src/onboarding/onboarding_screen.rs` | - |
| Auth 저장 | `AuthManager::shared()` | `codex-rs/core/src/auth/mod.rs` | - |

## 다이어그램: 전체 흐름

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. 프로그램 시작 (main)                                         │
│    - arg0_dispatch_or_else 체크                                 │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. CLI 파싱 (cli_main)                                          │
│    - MultitoolCli::parse()                                      │
│    - 서브커맨드 라우팅                                          │
└────────────────────────┬────────────────────────────────────────┘
                         │ (대화형 모드 선택)
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. TUI 초기화 (codex_tui::run_main)                            │
│    - ConfigOverrides 생성                                       │
│    - Config 로드 (~/.codex/config.toml)                        │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. 로깅 설정                                                    │
│    - 로그 파일 생성                                             │
│    - tracing subscriber 설정                                    │
│    - OpenTelemetry 초기화                                       │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. 인증 상태 확인                                               │
│    - AuthManager::shared()                                      │
│    - auth.json 읽기                                             │
│    - LoginStatus 결정                                           │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 6. 온보딩 필요 여부 판단                                        │
│    - should_show_trust_screen?                                  │
│    - should_show_windows_wsl_screen?                            │
│    - should_show_login_screen?                                  │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
        ┌────────────────┴────────────────┐
        │ 온보딩 필요?                    │
        └────────┬──────────────────┬─────┘
                YES               NO
                 │                 │
                 ▼                 ▼
┌────────────────────────────┐    │
│ 7. 온보딩 화면 실행        │    │
│    - Windows WSL 안내      │    │
│    - Login 화면            │    │
│    - Trust 화면            │    │
│    - 설정 업데이트         │    │
└────────────┬───────────────┘    │
             │                    │
             └────────┬───────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────────┐
│ 8. 설정 최종화                                                  │
│    - Config reload (필요시)                                     │
│    - enforce_login_restrictions()                               │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 9. App::run 진입                                                │
│    - ConversationManager 생성                                   │
│    - 대화 시작/재개                                             │
└─────────────────────────────────────────────────────────────────┘
```

## 다음 단계

이제 CLI가 성공적으로 초기화되었습니다. 다음 시나리오에서는:
- **[대화형 TUI 모드](02-tui-interactive-mode.md)**: App::run 내부에서 일어나는 일
- **[대화 생명주기](04-conversation-lifecycle.md)**: ConversationManager와 Codex::spawn의 동작
