# 시나리오 3: Exec 비대화형 모드

## 시나리오 개요

사용자가 `codex exec "fix all type errors"` 명령을 실행하여 자동화된 워크플로우에서 Codex를 사용하는 시나리오입니다. TUI 대신 표준 출력으로 결과를 받고, CI/CD나 스크립트에서 사용하기 적합한 형식을 제공합니다.

## 이야기로 따라가는 실행 흐름

### 1단계: Exec 서브커맨드 라우팅

사용자가 `codex exec "fix all type errors"` 명령을 실행합니다.

**코드**: `codex-rs/cli/src/main.rs::cli_main()`

```rust
match subcommand {
    Some(Subcommand::Exec(mut exec_cli)) => {
        prepend_config_flags(&mut exec_cli.config_overrides, root_config_overrides.clone());
        codex_exec::run_main(exec_cli, codex_linux_sandbox_exe).await?;
    }
    // ... 기타 서브커맨드
}
```

**Exec CLI 옵션**:
```rust
pub struct Cli {
    pub prompt: Option<String>,           // 프롬프트 (또는 stdin)
    pub images: Vec<PathBuf>,             // 이미지 파일들
    pub model: Option<String>,            // 모델 override
    pub oss: bool,                        // OSS 모델 사용
    pub full_auto: bool,                  // 자동 승인
    pub sandbox_mode: Option<SandboxModeCliArg>,
    pub json: bool,                       // JSONL 출력 모드
    pub color: Color,                     // 컬러 출력 제어
    pub last_message_file: Option<PathBuf>, // 마지막 메시지 저장
    pub output_schema: Option<PathBuf>,   // JSON 출력 스키마
    // ...
}
```

**위치**: `codex-rs/cli/src/main.rs:419-424`

### 2단계: Exec 모드 초기화

`codex_exec::run_main()`이 호출됩니다.

**코드**: `codex-rs/exec/src/lib.rs::run_main()`

```rust
pub async fn run_main(cli: Cli, codex_linux_sandbox_exe: Option<PathBuf>) -> anyhow::Result<()> {
    // 프롬프트 읽기 (인자 또는 stdin)
    let prompt = match cli.prompt {
        Some(p) if p != "-" => p,
        _ => {
            // stdin이 TTY면 에러
            if std::io::stdin().is_terminal() {
                eprintln!("No prompt provided. Specify as argument or pipe to stdin.");
                std::process::exit(1);
            }
            // stdin에서 읽기
            let mut buffer = String::new();
            std::io::stdin().read_to_string(&mut buffer)?;
            buffer
        }
    };

    // ANSI 컬러 지원 확인
    let (stdout_with_ansi, stderr_with_ansi) = match cli.color {
        Color::Always => (true, true),
        Color::Never => (false, false),
        Color::Auto => (
            supports_color::on(Stream::Stdout).is_some(),
            supports_color::on(Stream::Stderr).is_some(),
        ),
    };

    // 샌드박스 모드 결정
    let sandbox_mode = if cli.full_auto {
        Some(SandboxMode::WorkspaceWrite)
    } else if cli.dangerously_bypass_approvals_and_sandbox {
        Some(SandboxMode::DangerFullAccess)
    } else {
        cli.sandbox_mode.map(Into::into)
    };

    // Config 로드
    let config = Config::load_with_cli_overrides(cli_kv_overrides, overrides).await?;

    // ...
}
```

**동작**:
1. **프롬프트 소스 결정**:
   - 명령줄 인자
   - stdin (파이프)
   - `-` 명시적 stdin

2. **컬러 출력 설정**:
   - `--color=always/never/auto`
   - TTY 여부 자동 감지

3. **샌드박스 정책**:
   - `--full-auto`: WorkspaceWrite + Never 승인
   - `--dangerously-bypass-approvals-and-sandbox`: FullAccess

4. **Config 로드**: TUI와 동일한 로직

**위치**: `codex-rs/exec/src/lib.rs:50-194`

### 3단계: EventProcessor 선택

출력 형식에 따라 EventProcessor를 선택합니다.

```rust
let mut event_processor: Box<dyn EventProcessor> = match cli.json {
    true => Box::new(EventProcessorWithJsonOutput::new(cli.last_message_file.clone())),
    false => Box::new(EventProcessorWithHumanOutput::create_with_ansi(
        stdout_with_ansi,
        &config,
        cli.last_message_file.clone(),
    )),
};
```

**EventProcessor 인터페이스**:
```rust
pub trait EventProcessor {
    fn process_event(&mut self, event: Event) -> CodexStatus;
    fn print_config_summary(&self, config: &Config, prompt: &str);
    fn print_final_output(&self);
}
```

**두 가지 구현**:

1. **EventProcessorWithHumanOutput** (기본):
   - 사람이 읽기 좋은 형식
   - ANSI 컬러 지원
   - 실시간 스트리밍 출력

2. **EventProcessorWithJsonOutput** (`--json`):
   - JSONL 형식 (한 줄에 하나의 JSON 객체)
   - 파싱하기 쉬움
   - 자동화 워크플로우에 적합

**위치**: `codex-rs/exec/src/lib.rs:225-232`

### 4단계: ConversationManager 생성 및 대화 시작

TUI 모드와 유사하게 대화를 시작합니다.

```rust
let auth_manager = AuthManager::shared(
    config.codex_home.clone(),
    true,  // headless mode
    config.cli_auth_credentials_store_mode,
);

let conversation_manager = ConversationManager::new(
    auth_manager.clone(),
    SessionSource::Exec,  // 소스는 Exec
);

// 새 대화 또는 Resume
let NewConversation {
    conversation_id: _,
    conversation,
    session_configured,
} = if let Some(ExecCommand::Resume(args)) = cli.command {
    // Resume 처리
    let resume_path = resolve_resume_path(&config, &args).await?;
    if let Some(path) = resume_path {
        conversation_manager
            .resume_conversation_from_rollout(config.clone(), path, auth_manager.clone())
            .await?
    } else {
        conversation_manager.new_conversation(config.clone()).await?
    }
} else {
    // 새 대화
    conversation_manager.new_conversation(config.clone()).await?
};

// 설정 요약 출력
event_processor.print_config_summary(&config, &prompt, &session_configured);
```

**차이점 (TUI vs Exec)**:
- `SessionSource::Exec`: 메트릭/로그에서 구분
- `headless: true`: 브라우저 열기 등 비활성화
- 설정 요약을 stdout에 출력

**위치**: `codex-rs/exec/src/lib.rs:252-284`

### 5단계: 이벤트 루프 시작

백그라운드에서 이벤트를 받는 태스크를 시작합니다.

```rust
let (tx, mut rx) = tokio::sync::mpsc::unbounded_channel::<Event>();

{
    let conversation = conversation.clone();
    tokio::spawn(async move {
        loop {
            tokio::select! {
                // Ctrl+C 처리
                _ = tokio::signal::ctrl_c() => {
                    tracing::debug!("Keyboard interrupt");
                    conversation.submit(Op::Interrupt).await.ok();
                    break;
                }
                // 이벤트 수신
                res = conversation.next_event() => match res {
                    Ok(event) => {
                        debug!("Received event: {event:?}");
                        let is_shutdown = matches!(event.msg, EventMsg::ShutdownComplete);
                        if let Err(e) = tx.send(event) {
                            error!("Error sending event: {e:?}");
                            break;
                        }
                        if is_shutdown {
                            info!("Received shutdown event, exiting.");
                            break;
                        }
                    },
                    Err(e) => {
                        error!("Error receiving event: {e:?}");
                        break;
                    }
                }
            }
        }
    });
}
```

**동작**:
1. 비동기 채널 생성
2. 백그라운드 태스크에서 `conversation.next_event()` 대기
3. Ctrl+C 시그널 처리 - `Op::Interrupt` 전송
4. 이벤트를 메인 루프로 전달

**위치**: `codex-rs/exec/src/lib.rs:287-324`

### 6단계: 사용자 턴 제출

초기 프롬프트를 제출합니다.

```rust
// 이미지와 텍스트를 UserInput으로 변환
let mut items: Vec<UserInput> = cli.images
    .into_iter()
    .map(|path| UserInput::LocalImage { path })
    .collect();
items.push(UserInput::Text { text: prompt });

// Op::UserTurn 제출
let initial_prompt_task_id = conversation
    .submit(Op::UserTurn {
        items,
        cwd: config.cwd.clone(),
        approval_policy: config.approval_policy,
        sandbox_policy: config.sandbox_policy.clone(),
        model: config.model.clone(),
        effort: config.model_reasoning_effort,
        summary: config.model_reasoning_summary,
        final_output_json_schema: output_schema,
    })
    .await?;

info!("Sent prompt with event ID: {initial_prompt_task_id}");
```

**동작**:
- TUI 모드와 동일한 `Op::UserTurn`
- 이미지 지원
- `output_schema` 지정 가능 (구조화된 JSON 출력)

**위치**: `codex-rs/exec/src/lib.rs:326-344`

### 7단계: 이벤트 처리 (Human 출력)

메인 루프에서 이벤트를 받아 EventProcessor로 처리합니다.

**코드**: `codex-rs/exec/src/event_processor_with_human_output.rs`

```rust
impl EventProcessor for EventProcessorWithHumanOutput {
    fn process_event(&mut self, event: Event) -> CodexStatus {
        match &event.msg {
            EventMsg::AgentMessageContentDelta(delta) => {
                // 실시간 텍스트 출력
                print!("{}", delta.delta);
                std::io::stdout().flush().ok();
                CodexStatus::Running
            }
            EventMsg::ExecCommandOutputDelta(output_delta) => {
                // 명령 출력 스트리밍
                let text = match output_delta.stream {
                    ExecOutputStream::Stdout => &output_delta.output,
                    ExecOutputStream::Stderr => &output_delta.output,
                };
                print!("{}", text.dim());
                std::io::stdout().flush().ok();
                CodexStatus::Running
            }
            EventMsg::ToolCallStarted(tool_call) => {
                // 도구 호출 표시
                println!("\n┌─ {} ─┐", tool_call.name.bold());
                println!("│ {}", format_args(&tool_call.args));
                CodexStatus::Running
            }
            EventMsg::ToolCallResult(result) => {
                // 도구 결과 표시
                println!("└─ {} ─┘\n", "Done".green());
                CodexStatus::Running
            }
            EventMsg::TurnComplete(_) => {
                println!("\n");
                CodexStatus::InitiateShutdown
            }
            EventMsg::Error(err) => {
                eprintln!("\n{}: {}", "Error".red().bold(), err.message);
                CodexStatus::InitiateShutdown
            }
            // ... 기타 이벤트
            _ => CodexStatus::Running,
        }
    }
}
```

**출력 예시**:
```
$ codex exec "create a hello world python script"

I'll create a hello world Python script for you.

┌─ Bash ─┐
│ echo 'print("Hello, World!")' > hello.py
└─ Done ─┘

Created hello.py with a simple hello world program.

Token usage: total=234 input=123 output=111
```

**위치**: `codex-rs/exec/src/event_processor_with_human_output.rs`

### 8단계: 이벤트 처리 (JSON 출력)

`--json` 플래그 사용 시 JSONL 형식으로 출력합니다.

**코드**: `codex-rs/exec/src/event_processor_with_jsonl_output.rs`

```rust
impl EventProcessor for EventProcessorWithJsonOutput {
    fn process_event(&mut self, event: Event) -> CodexStatus {
        // 모든 이벤트를 JSON으로 직렬화
        let json_event = serde_json::json!({
            "type": event_type_name(&event.msg),
            "id": event.id,
            "data": event.msg,
        });

        // stdout에 한 줄씩 출력
        println!("{}", serde_json::to_string(&json_event).unwrap());

        // 종료 이벤트 확인
        match &event.msg {
            EventMsg::TurnComplete(_) => CodexStatus::InitiateShutdown,
            EventMsg::Error(_) => CodexStatus::InitiateShutdown,
            EventMsg::ShutdownComplete => CodexStatus::Shutdown,
            _ => CodexStatus::Running,
        }
    }
}
```

**JSONL 출력 예시**:
```jsonl
{"type":"task_started","id":"0","data":{"task_id":"0"}}
{"type":"agent_message_content_delta","id":"0","data":{"delta":"I'll"}}
{"type":"agent_message_content_delta","id":"0","data":{"delta":" create"}}
{"type":"tool_call_started","id":"0","data":{"name":"Bash","args":{"command":["echo","..."]}}}
{"type":"tool_call_result","id":"0","data":{"output":"","exit_code":0}}
{"type":"turn_complete","id":"0","data":{"token_usage":{"total":234,"input":123,"output":111}}}
```

**장점**:
- 파싱하기 쉬움
- 스트리밍 처리 가능
- jq 등 도구로 필터링 가능

**위치**: `codex-rs/exec/src/event_processor_with_jsonl_output.rs`

### 9단계: 자동 승인 (--full-auto)

`--full-auto` 플래그 사용 시 모든 도구 호출을 자동 승인합니다.

```rust
let approval_policy = if cli.full_auto {
    Some(AskForApproval::Never)
} else {
    None
};
```

**동작**:
- `AskForApproval::Never`: 승인 없이 모든 도구 실행
- `AskForApproval::OnRequest`: 요청 시 승인 필요 (기본)
- `AskForApproval::OnDanger`: 위험한 작업만 승인

**Exec 모드의 승인 처리**:
```rust
EventMsg::ExecApprovalRequest(approval) => {
    // Exec 모드에서는 승인 요청이 오면 자동으로 거부
    eprintln!("Approval required but running in non-interactive mode. Denying.");
    conversation.submit(Op::ApprovalResponse { approved: false }).await;
    CodexStatus::InitiateShutdown
}
```

**위치**: `codex-rs/exec/src/lib.rs:140-147`

### 10단계: 마지막 메시지 저장

`--last-message-file` 옵션으로 마지막 어시스턴트 메시지를 파일에 저장할 수 있습니다.

```rust
impl EventProcessorWithHumanOutput {
    fn process_event(&mut self, event: Event) -> CodexStatus {
        match &event.msg {
            EventMsg::AgentMessageContentDelta(delta) => {
                // 현재 메시지에 추가
                self.current_message.push_str(&delta.delta);
                // ...
            }
            EventMsg::TurnComplete(_) => {
                // 파일에 저장
                if let Some(ref path) = self.last_message_file {
                    std::fs::write(path, &self.current_message).ok();
                }
                CodexStatus::InitiateShutdown
            }
            // ...
        }
    }
}
```

**사용 예시**:
```bash
codex exec "summarize README.md" --last-message-file summary.txt
cat summary.txt
```

**위치**: `codex-rs/exec/src/event_processor_with_human_output.rs`

### 11단계: Resume 지원

Exec 모드도 이전 세션을 재개할 수 있습니다.

```bash
# 마지막 세션 재개
codex exec resume --last "continue the implementation"

# 특정 세션 재개
codex exec resume <session-id> "add error handling"
```

**코드**:
```rust
async fn resolve_resume_path(
    config: &Config,
    args: &ResumeArgs,
) -> anyhow::Result<Option<PathBuf>> {
    if args.last {
        // 가장 최근 세션 찾기
        let conversations = RolloutRecorder::list_conversations(
            &config.codex_home,
            1,
            None,
            &[],
            Some(&[config.model_provider_id.clone()]),
            &config.model_provider_id,
        )
        .await?;
        Ok(conversations.items.first().map(|it| it.path.clone()))
    } else if let Some(id_str) = &args.session_id {
        // 세션 ID로 찾기
        find_conversation_path_by_id_str(&config.codex_home, id_str).await
    } else {
        Ok(None)
    }
}
```

**위치**: `codex-rs/exec/src/lib.rs:373-401`

### 12단계: 종료 처리

이벤트 루프가 완료되면 정리하고 종료합니다.

```rust
let mut error_seen = false;
while let Some(event) = rx.recv().await {
    if matches!(event.msg, EventMsg::Error(_)) {
        error_seen = true;
    }

    let status = event_processor.process_event(event);
    match status {
        CodexStatus::Running => continue,
        CodexStatus::InitiateShutdown => {
            conversation.submit(Op::Shutdown).await?;
        }
        CodexStatus::Shutdown => {
            break;
        }
    }
}

// 최종 출력
event_processor.print_final_output();

// 에러가 있었으면 exit code 1
if error_seen {
    std::process::exit(1);
}

Ok(())
```

**Exit Code**:
- `0`: 성공
- `1`: 에러 발생 (자동화 워크플로우에서 감지 가능)

**위치**: `codex-rs/exec/src/lib.rs:349-371`

## 사용 예시

### 1. 기본 사용
```bash
codex exec "fix all TypeScript type errors"
```

### 2. Stdin에서 프롬프트 읽기
```bash
echo "create a Dockerfile for this project" | codex exec
```

### 3. JSON 출력 (파싱용)
```bash
codex exec "list all TODO comments" --json | jq '.data.delta' -r
```

### 4. 자동 승인 (CI/CD)
```bash
codex exec "run tests and fix failures" --full-auto
```

### 5. 구조화된 출력
```bash
cat schema.json
# {"type": "object", "properties": {"summary": {"type": "string"}}}

codex exec "summarize this code" --output-schema schema.json
```

### 6. 마지막 메시지 저장
```bash
codex exec "write documentation" --last-message-file docs.md
```

## 주요 차이점 요약

| 특징 | TUI 모드 | Exec 모드 |
|-----|---------|-----------|
| UI | 전체 화면 Ratatui | stdout/stderr |
| 입력 | 키보드 상호작용 | CLI 인자/stdin |
| 출력 | 실시간 렌더링 | 텍스트 스트림 또는 JSONL |
| 승인 | 대화형 팝업 | 자동 거부 (또는 --full-auto) |
| 세션 소스 | Interactive | Exec |
| 사용 사례 | 개발자 대화 | 자동화, CI/CD |

## 다음 단계

- **[대화 생명주기](04-conversation-lifecycle.md)**: ConversationManager 상세
- **[도구 실행 및 샌드박싱](05-tool-execution-and-sandboxing.md)**: 도구 실행 내부 동작
