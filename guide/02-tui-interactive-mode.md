# 시나리오 2: 대화형 TUI 모드

## 시나리오 개요

사용자가 인증을 완료하고 `App::run`에 진입한 후, 실제 대화형 터미널 UI에서 프롬프트를 입력하고 AI 에이전트의 응답을 실시간으로 받는 과정을 추적합니다.

## 이야기로 따라가는 실행 흐름

### 1단계: TUI 초기화

`App::run`이 호출되면 Ratatui 기반 전체 화면 UI가 시작됩니다.

**진입점**: `codex-rs/tui/src/app.rs::App::run()`

```rust
impl App {
    pub async fn run(
        tui: &mut Tui,
        auth_manager: Arc<AuthManager>,
        config: Config,
        active_profile: Option<String>,
        prompt: Option<String>,
        images: Vec<PathBuf>,
        resume_selection: ResumeSelection,
        feedback: CodexFeedback,
    ) -> color_eyre::Result<AppExitInfo> {
        // ConversationManager 생성
        let conversation_manager = ConversationManager::new(
            auth_manager.clone(),
            SessionSource::Interactive,
        );

        // Resume 또는 새 대화 시작
        let NewConversation {
            conversation_id,
            conversation,
            session_configured,
        } = match resume_selection {
            ResumeSelection::Resume(path) => {
                conversation_manager
                    .resume_conversation_from_rollout(config.clone(), path, auth_manager.clone())
                    .await?
            }
            ResumeSelection::StartFresh => {
                conversation_manager.new_conversation(config.clone()).await?
            }
            _ => unreachable!(),
        };

        // App 상태 초기화
        let mut app = App {
            config,
            conversation,
            // ... 기타 필드
        };

        // 이벤트 루프 시작
        app.run_event_loop(tui).await
    }
}
```

**동작**:
1. `ConversationManager` 생성 - 세션 소스는 `Interactive`
2. Resume 또는 새 대화 생성:
   - **Resume**: Rollout 파일에서 히스토리 로드
   - **StartFresh**: 빈 히스토리로 시작
3. `App` 구조체 초기화 - UI 상태 관리
4. 이벤트 루프 진입

**위치**: `codex-rs/tui/src/app.rs`

### 2단계: 대화 생성 - Codex::spawn

`ConversationManager::new_conversation()`은 내부적으로 `Codex::spawn`을 호출합니다.

**코드**: `codex-rs/core/src/codex.rs::Codex::spawn()`

```rust
pub async fn spawn(
    config: Config,
    auth_manager: Arc<AuthManager>,
    conversation_history: InitialHistory,
    session_source: SessionSource,
) -> CodexResult<CodexSpawnOk> {
    // 비동기 채널 생성
    let (tx_sub, rx_sub) = async_channel::bounded(SUBMISSION_CHANNEL_CAPACITY);
    let (tx_event, rx_event) = async_channel::unbounded();

    // 사용자 지시사항 로드 (CODEX.md, AGENTS.md 등)
    let user_instructions = get_user_instructions(&config).await;

    // Session 생성
    let session = Session::new(
        session_configuration,
        config.clone(),
        auth_manager.clone(),
        tx_event.clone(),
        conversation_history,
        session_source,
    )
    .await?;

    // 백그라운드 이벤트 루프 시작
    tokio::spawn(session.run(rx_sub, tx_event));

    // Codex 인스턴스 반환
    Ok(CodexSpawnOk {
        codex: Codex {
            next_id: AtomicU64::new(1),
            tx_sub,
            rx_event,
        },
        conversation_id: session.conversation_id,
    })
}
```

**동작**:
1. **채널 생성**:
   - `tx_sub/rx_sub`: UI → 에이전트 (Op 전송)
   - `tx_event/rx_event`: 에이전트 → UI (Event 수신)

2. **Session 초기화**:
   - 설정 읽기
   - MCP 서버 연결
   - Rollout recorder 설정

3. **백그라운드 태스크**:
   - `session.run()`을 별도 태스크로 실행
   - 계속 rx_sub에서 Op를 기다림

4. **Codex 인스턴스 반환**:
   - `next_id`: submission ID 생성용 카운터
   - `tx_sub`: Op 전송용 채널
   - `rx_event`: Event 수신용 채널

**위치**: `codex-rs/core/src/codex.rs:156-250`

### 3단계: Session 초기화

`Session::new()`는 세션의 모든 서비스를 초기화합니다.

**코드**: `codex-rs/core/src/state/mod.rs`

```rust
impl Session {
    pub async fn new(
        config: SessionConfiguration,
        // ...
    ) -> CodexResult<Self> {
        // Rollout recorder 초기화
        let rollout_recorder = RolloutRecorder::new(
            RolloutRecorderParams {
                codex_home: original_config.codex_home.clone(),
                config: &config,
                conversation_history,
            },
        )
        .await?;

        // MCP 연결 매니저 초기화
        let mcp_manager = McpConnectionManager::new(
            original_config.mcp_servers.clone(),
            DEFAULT_STARTUP_TIMEOUT,
        );

        // ToolRouter 생성
        let tool_router = ToolRouter::new(tools_config);

        // SessionServices 생성
        let services = Arc::new(SessionServices {
            auth_manager,
            rollout_recorder,
            mcp_manager,
            tool_router,
            // ...
        });

        // 첫 이벤트: SessionConfigured 전송
        tx_event.send(Event {
            id: INITIAL_SUBMIT_ID.to_string(),
            msg: EventMsg::SessionConfigured(SessionConfiguredEvent {
                config: config.clone(),
                rollout_path: services.rollout_recorder.path().to_path_buf(),
                // ...
            }),
        }).await?;

        Ok(Session {
            state: SessionState::Idle,
            services,
            // ...
        })
    }
}
```

**동작**:
1. **RolloutRecorder 초기화**:
   - `~/.codex/sessions/<uuid>/` 디렉토리 생성
   - 히스토리가 있으면 로드

2. **McpConnectionManager 초기화**:
   - config에 정의된 MCP 서버들에 연결 시도
   - stdio 또는 SSE transport

3. **ToolRouter 생성**:
   - 내장 도구 등록 (Bash, Read, Write, Edit, Glob, Grep 등)
   - MCP 도구 등록

4. **SessionConfigured 이벤트 전송**:
   - UI에 세션 정보 알림
   - rollout 경로 포함

**위치**: `codex-rs/core/src/state/mod.rs`

### 4단계: 초기 화면 렌더링

`App::run_event_loop()`에서 TUI를 처음 렌더링합니다.

```rust
async fn run_event_loop(&mut self, tui: &mut Tui) -> color_eyre::Result<AppExitInfo> {
    // 초기 렌더링
    tui.draw(|f| self.render(f, f.area()))?;

    loop {
        tokio::select! {
            // 에이전트 이벤트
            Some(event) = self.conversation.next_event() => {
                self.handle_agent_event(event);
                tui.draw(|f| self.render(f, f.area()))?;
            }
            // 키보드 입력
            Some(Ok(crossterm_event)) = tui.events.next() => {
                self.handle_key_event(crossterm_event);
                tui.draw(|f| self.render(f, f.area()))?;
            }
            // ...
        }

        if self.should_exit {
            break;
        }
    }

    Ok(self.get_exit_info())
}
```

**TUI 레이아웃** (Ratatui):
```
┌─────────────────────────────────────────────────────┐
│ Codex CLI - GPT-4 Turbo                             │  <- 헤더
├─────────────────────────────────────────────────────┤
│                                                     │
│  [Assistant] Let me help you with that.            │  <- 대화 영역
│                                                     │
│  [You] Write a hello world program in Python       │
│                                                     │
│  [Assistant] I'll create a hello world program.    │
│  ┌─────────────────────────────────────────┐      │
│  │ bash -c "echo 'print(\"Hello\")' > h.py"│      │  <- 도구 호출
│  └─────────────────────────────────────────┘      │
│  Done! Created hello.py                            │
│                                                     │
├─────────────────────────────────────────────────────┤
│ > _                                                 │  <- 입력 영역
├─────────────────────────────────────────────────────┤
│ Ready | Tokens: 1234                                │  <- 상태 바
└─────────────────────────────────────────────────────┘
```

**위치**: `codex-rs/tui/src/app.rs`, `codex-rs/tui/src/render.rs`

### 5단계: 사용자 입력 처리

사용자가 입력 영역에 텍스트를 입력하고 Enter를 누릅니다.

**코드**: `codex-rs/tui/src/app.rs::handle_key_event()`

```rust
fn handle_key_event(&mut self, event: crossterm::event::Event) {
    match event {
        Event::Key(KeyEvent {
            code: KeyCode::Enter,
            modifiers: KeyModifiers::NONE,
            ..
        }) if self.input_mode == InputMode::Editing => {
            // 입력된 텍스트 가져오기
            let text = self.input_widget.take_text();

            // Op::UserTurn 생성 및 전송
            let items = vec![UserInput::Text { text }];
            self.conversation.submit(Op::UserTurn {
                items,
                cwd: self.config.cwd.clone(),
                approval_policy: self.config.approval_policy,
                sandbox_policy: self.config.sandbox_policy.clone(),
                model: self.config.model.clone(),
                effort: self.config.model_reasoning_effort,
                summary: self.config.model_reasoning_summary,
                final_output_json_schema: None,
            }).await;

            self.input_mode = InputMode::Waiting;
        }
        // ... 기타 키 처리
    }
}
```

**동작**:
1. Enter 키 감지
2. 입력 위젯에서 텍스트 추출
3. `Op::UserTurn` 생성:
   - `items`: 사용자 입력 (텍스트, 이미지 등)
   - `cwd`: 현재 작업 디렉토리
   - `approval_policy`: 승인 정책
   - `sandbox_policy`: 샌드박스 정책
   - `model`: 사용할 모델
4. `conversation.submit()` 호출 - tx_sub 채널로 전송
5. 입력 모드를 `Waiting`으로 전환

**위치**: `codex-rs/tui/src/app.rs`

### 6단계: Session이 Op 처리

백그라운드의 `session.run()` 루프가 Op를 받습니다.

**코드**: `codex-rs/core/src/state/mod.rs::Session::run()`

```rust
pub async fn run(mut self, rx_sub: Receiver<Submission>, tx_event: Sender<Event>) {
    loop {
        tokio::select! {
            // UI로부터 submission 수신
            Ok(submission) = rx_sub.recv() => {
                match submission.op {
                    Op::UserTurn { items, cwd, approval_policy, sandbox_policy, model, ... } => {
                        self.handle_user_turn(
                            submission.id,
                            items,
                            cwd,
                            approval_policy,
                            sandbox_policy,
                            model,
                            // ...
                        ).await;
                    }
                    Op::Interrupt => {
                        self.handle_interrupt().await;
                    }
                    Op::Shutdown => {
                        break;
                    }
                    // ... 기타 Op
                }
            }
            // 다른 이벤트들...
        }
    }
}
```

**동작**:
1. `rx_sub.recv()` 대기
2. `Op::UserTurn` 수신
3. `handle_user_turn()` 호출

**위치**: `codex-rs/core/src/state/mod.rs`

### 7단계: LLM 호출 준비

`handle_user_turn()`은 LLM 호출을 준비합니다.

```rust
async fn handle_user_turn(
    &mut self,
    sub_id: String,
    items: Vec<UserInput>,
    cwd: PathBuf,
    // ...
) {
    // TaskStarted 이벤트 전송
    self.tx_event.send(Event {
        id: sub_id.clone(),
        msg: EventMsg::TaskStarted(TaskStartedEvent {
            task_id: sub_id.clone(),
        }),
    }).await;

    // 사용자 입력을 히스토리에 추가
    self.services.rollout_recorder.append_user_input(items.clone()).await;

    // 컨텍스트 구성
    let context = self.build_context(cwd, items).await;

    // 프롬프트 구성
    let prompt = self.build_prompt(context).await;

    // ModelClient로 LLM 호출
    let response_stream = self.model_client
        .chat_completions_stream(prompt)
        .await?;

    // 응답 스트림 처리
    self.process_response_stream(sub_id, response_stream).await;
}
```

**프롬프트 구성**:
1. **System Message**:
   - Base instructions
   - Developer instructions (AGENTS.md)
   - User instructions (CODEX.md)
   - 환경 컨텍스트 (OS, cwd, git info)
   - 도구 목록

2. **History**:
   - 이전 대화 내역 (rollout에서 로드)
   - 사용자 메시지
   - 어시스턴트 메시지
   - 도구 호출 결과

3. **Current Turn**:
   - 사용자의 새 입력

**위치**: `codex-rs/core/src/state/session_task.rs`

### 8단계: LLM 응답 스트리밍

ModelClient가 OpenAI API를 호출하고 응답을 스트리밍합니다.

**코드**: `codex-rs/core/src/client/mod.rs`

```rust
pub async fn chat_completions_stream(
    &self,
    prompt: Prompt,
) -> CodexResult<impl Stream<Item = CodexResult<ResponseEvent>>> {
    // HTTP 요청 구성
    let request = self.build_request(prompt);

    // SSE 스트림 열기
    let response = self.http_client
        .post(&self.endpoint)
        .json(&request)
        .send()
        .await?;

    // SSE 이벤트 파싱
    let stream = response
        .bytes_stream()
        .map(|chunk| parse_sse_chunk(chunk))
        .filter_map(|event| async move {
            match event {
                Ok(ResponseEvent::ContentDelta { text }) => Some(Ok(ResponseEvent::ContentDelta { text })),
                Ok(ResponseEvent::ToolCall { name, args }) => Some(Ok(ResponseEvent::ToolCall { name, args })),
                Ok(ResponseEvent::Done) => Some(Ok(ResponseEvent::Done)),
                Err(e) => Some(Err(e)),
                _ => None,
            }
        });

    Ok(stream)
}
```

**ResponseEvent 종류**:
- `ContentDelta`: 텍스트 스트리밍
- `ToolCall`: 도구 호출 시작
- `ToolCallDelta`: 도구 인자 스트리밍
- `Done`: 응답 완료

**위치**: `codex-rs/core/src/client/`

### 9단계: 응답 처리 및 UI 업데이트

Session이 응답 스트림을 처리하고 Event를 UI에 전송합니다.

```rust
async fn process_response_stream(
    &mut self,
    sub_id: String,
    mut stream: impl Stream<Item = CodexResult<ResponseEvent>>,
) {
    while let Some(event) = stream.next().await {
        match event? {
            ResponseEvent::ContentDelta { text } => {
                // UI에 텍스트 델타 전송
                self.tx_event.send(Event {
                    id: sub_id.clone(),
                    msg: EventMsg::AgentMessageContentDelta(AgentMessageContentDeltaEvent {
                        delta: text,
                    }),
                }).await;
            }
            ResponseEvent::ToolCall { name, args } => {
                // 도구 실행
                let result = self.execute_tool(name, args).await?;

                // 도구 결과를 히스토리에 추가
                self.services.rollout_recorder.append_tool_result(result.clone()).await;

                // UI에 도구 결과 전송
                self.tx_event.send(Event {
                    id: sub_id.clone(),
                    msg: EventMsg::ToolResult(result),
                }).await;
            }
            ResponseEvent::Done => {
                // 턴 완료 이벤트 전송
                self.tx_event.send(Event {
                    id: sub_id.clone(),
                    msg: EventMsg::TurnComplete(TurnCompleteEvent {
                        token_usage: self.token_usage,
                    }),
                }).await;
                break;
            }
        }
    }
}
```

**동작**:
1. 스트림에서 이벤트 읽기
2. **ContentDelta**: 텍스트를 UI에 즉시 전송
3. **ToolCall**: 도구 실행하고 결과 전송
4. **Done**: 턴 완료 이벤트 전송

**위치**: `codex-rs/core/src/response_processing.rs`

### 10단계: TUI가 Event 처리

App의 이벤트 루프가 Event를 받아 UI를 업데이트합니다.

```rust
async fn handle_agent_event(&mut self, event: Event) {
    match event.msg {
        EventMsg::AgentMessageContentDelta(delta_event) => {
            // 현재 메시지에 텍스트 추가
            self.current_message.push_str(&delta_event.delta);
            // 화면 다시 그리기는 이벤트 루프에서 자동으로 호출됨
        }
        EventMsg::ToolResult(result) => {
            // 도구 결과를 히스토리에 추가
            self.history.push(HistoryItem::ToolCall {
                name: result.name,
                args: result.args,
                result: result.output,
            });
        }
        EventMsg::TurnComplete(complete_event) => {
            // 현재 메시지를 히스토리에 추가
            self.history.push(HistoryItem::Assistant {
                text: std::mem::take(&mut self.current_message),
            });
            // 입력 모드로 전환
            self.input_mode = InputMode::Editing;
        }
        EventMsg::ExecApprovalRequest(approval_request) => {
            // 승인 팝업 표시
            self.show_approval_dialog(approval_request);
        }
        // ... 기타 이벤트
    }
}
```

**UI 업데이트 흐름**:
1. Event 수신
2. App 상태 업데이트
3. `tui.draw()` 호출 - Ratatui가 변경된 부분만 다시 그림
4. 터미널에 출력

**실시간 스트리밍 효과**:
- ContentDelta 이벤트가 연속으로 오면 텍스트가 한 글자씩 나타남
- Ratatui의 효율적인 diff 알고리즘으로 부드러운 렌더링

**위치**: `codex-rs/tui/src/app.rs`, `codex-rs/tui/src/app_event.rs`

### 11단계: 도구 실행 (예: Bash)

LLM이 도구 호출을 요청하면 ToolRouter가 처리합니다.

**코드**: `codex-rs/core/src/tools/router.rs`

```rust
pub async fn execute_tool(
    &self,
    name: String,
    args: Value,
    context: &ToolContext,
) -> CodexResult<ToolResult> {
    match name.as_str() {
        "Bash" => {
            let params: BashParams = serde_json::from_value(args)?;

            // 승인 확인
            if context.approval_policy.requires_approval(&params) {
                self.request_approval(params.clone()).await?;
            }

            // 샌드박스에서 실행
            let output = crate::exec::process_exec_tool_call(
                ExecParams {
                    command: params.command,
                    cwd: context.cwd.clone(),
                    timeout_ms: params.timeout,
                    env: HashMap::new(),
                    with_escalated_permissions: None,
                    justification: None,
                    arg0: None,
                },
                context.sandbox_type,
                &context.sandbox_policy,
                &context.sandbox_cwd,
                &context.codex_linux_sandbox_exe,
                Some(stdout_stream),
            )
            .await?;

            Ok(ToolResult {
                name: "Bash".to_string(),
                output: output.stdout,
                exit_code: output.exit_code,
            })
        }
        // ... 기타 도구
    }
}
```

**도구 실행 단계**:
1. 인자 역직렬화
2. 승인 정책 확인 - 필요시 사용자에게 물어봄
3. 샌드박스 매니저에 전달
4. 플랫폼별 샌드박스에서 실행
5. 결과 반환

**위치**: `codex-rs/core/src/tools/router.rs`, 다음 시나리오에서 자세히 다룸

### 12단계: Rollout 저장

모든 대화 내역은 Rollout 파일에 저장됩니다.

**파일**: `~/.codex/sessions/<uuid>/rollout.jsonl`

```jsonl
{"type":"user_input","content":[{"type":"text","text":"Write a hello world program"}]}
{"type":"assistant_message","content":"I'll create a hello world program for you."}
{"type":"tool_call","name":"Bash","args":{"command":["echo","print('Hello, World!')",">","hello.py"]}}
{"type":"tool_result","name":"Bash","output":"","exit_code":0}
{"type":"assistant_message","content":"Done! I've created hello.py"}
```

**RolloutRecorder 동작**:
- 각 이벤트를 JSONL 형식으로 추가
- 파일은 항상 최신 상태 유지 (fsync)
- 재개 시 이 파일을 읽어 히스토리 복원

**위치**: `codex-rs/core/src/rollout/`

## 주요 데이터 흐름 다이어그램

```
사용자 입력
    │
    ▼
┌─────────────────────┐
│ TUI Event Handler   │
│ (handle_key_event)  │
└──────────┬──────────┘
           │ Op::UserTurn
           ▼
┌─────────────────────┐
│ tx_sub (channel)    │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ Session::run        │
│ (백그라운드 루프)   │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ handle_user_turn    │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ ModelClient         │
│ (OpenAI API 호출)   │
└──────────┬──────────┘
           │ ResponseEvent stream
           ▼
┌─────────────────────┐
│ process_response    │
│ - ContentDelta      │
│ - ToolCall          │
│ - Done              │
└──────────┬──────────┘
           │ Event
           ▼
┌─────────────────────┐
│ tx_event (channel)  │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ App event loop      │
│ (handle_agent_event)│
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ TUI Render          │
│ (Ratatui draw)      │
└─────────────────────┘
```

## 주요 타입 및 구조체

### Op (Operation)
```rust
pub enum Op {
    UserTurn {
        items: Vec<UserInput>,
        cwd: PathBuf,
        approval_policy: AskForApproval,
        sandbox_policy: SandboxPolicy,
        model: String,
        // ...
    },
    ApprovalResponse {
        approved: bool,
    },
    Interrupt,
    Shutdown,
}
```

### Event
```rust
pub struct Event {
    pub id: String,  // submission ID
    pub msg: EventMsg,
}

pub enum EventMsg {
    TaskStarted(TaskStartedEvent),
    AgentMessageContentDelta(AgentMessageContentDeltaEvent),
    ToolCallStarted(ToolCallStartedEvent),
    ToolCallResult(ToolCallResultEvent),
    TurnComplete(TurnCompleteEvent),
    ExecApprovalRequest(ExecApprovalRequestEvent),
    Error(ErrorEvent),
    // ... 많은 이벤트 타입들
}
```

### SessionState
```rust
pub enum SessionState {
    Idle,
    ProcessingUserTurn {
        task_id: String,
        cancellation_token: CancellationToken,
    },
    WaitingForApproval {
        pending_tool_call: PendingToolCall,
    },
}
```

## 성능 최적화

### 1. 채널 기반 비동기
- UI와 에이전트가 독립적인 태스크로 실행
- 논블로킹 메시지 전달

### 2. 스트리밍
- LLM 응답을 실시간으로 표시
- 사용자는 기다리지 않고 즉시 피드백 확인

### 3. Ratatui 효율적 렌더링
- 변경된 영역만 다시 그림
- 터미널 I/O 최소화

### 4. Tokio 멀티스레드
- CPU 바운드 작업은 `spawn_blocking`
- I/O 바운드 작업은 비동기

## 다음 단계

- **[Exec 비대화형 모드](03-exec-non-interactive-mode.md)**: 자동화 워크플로우
- **[도구 실행 및 샌드박싱](05-tool-execution-and-sandboxing.md)**: Bash, 파일 작업, 보안
- **[대화 생명주기](04-conversation-lifecycle.md)**: Resume, Fork, 히스토리 관리
