# 시나리오 4: 대화 생명주기

## 시나리오 개요

사용자가 대화를 시작하고, 중단하고, 나중에 재개하거나, 특정 시점으로 되돌아가는(fork) 전체 생명주기를 추적합니다. Rollout 메커니즘과 히스토리 관리가 어떻게 작동하는지 살펴봅니다.

## 이야기로 따라가는 실행 흐름

### 1단계: 새 대화 생성

사용자가 `codex`를 실행하여 새 대화를 시작합니다.

**코드**: `codex-rs/core/src/conversation_manager.rs::new_conversation()`

```rust
pub async fn new_conversation(&self, config: Config) -> CodexResult<NewConversation> {
    self.spawn_conversation(config, self.auth_manager.clone()).await
}

async fn spawn_conversation(
    &self,
    config: Config,
    auth_manager: Arc<AuthManager>,
) -> CodexResult<NewConversation> {
    // Codex::spawn 호출
    let CodexSpawnOk {
        codex,
        conversation_id,
    } = Codex::spawn(
        config,
        auth_manager,
        InitialHistory::New,  // 빈 히스토리
        self.session_source.clone(),
    )
    .await?;

    self.finalize_spawn(codex, conversation_id).await
}
```

**동작**:
1. `Codex::spawn` 호출 with `InitialHistory::New`
2. 내부에서 `Session::new()` 호출
3. 첫 이벤트 (`SessionConfigured`) 수신
4. `CodexConversation` 래퍼 생성

**위치**: `codex-rs/core/src/conversation_manager.rs:59-80`

### 2단계: Session ID 생성

`Codex::spawn` 내부에서 고유한 대화 ID를 생성합니다.

**코드**: `codex-rs/core/src/codex.rs`

```rust
pub async fn spawn(
    // ...
) -> CodexResult<CodexSpawnOk> {
    // UUID 생성
    let conversation_id = ConversationId::new();

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

    // ...

    Ok(CodexSpawnOk {
        codex,
        conversation_id,
    })
}
```

**ConversationId**:
- UUID v4 형식: `f47ac10b-58cc-4372-a567-0e02b2c3d479`
- 모든 대화마다 고유
- Rollout 디렉토리 이름으로 사용

**위치**: `codex-rs/protocol/src/lib.rs`

### 3단계: Rollout 디렉토리 초기화

`Session::new()`에서 `RolloutRecorder`를 초기화합니다.

**코드**: `codex-rs/core/src/rollout/mod.rs`

```rust
impl RolloutRecorder {
    pub async fn new(params: RolloutRecorderParams) -> CodexResult<Self> {
        let RolloutRecorderParams {
            codex_home,
            config,
            conversation_history,
        } = params;

        // Rollout 디렉토리 경로 결정
        let rollout_dir = if let InitialHistory::Resume { path } = &conversation_history {
            // Resume인 경우 기존 경로 사용
            path.clone()
        } else {
            // 새 대화인 경우 새 디렉토리 생성
            let session_id = ConversationId::new();
            let dir = codex_home
                .join("sessions")
                .join(session_id.to_string());

            // 디렉토리 생성
            tokio::fs::create_dir_all(&dir).await?;

            dir
        };

        // rollout.jsonl 파일 열기
        let rollout_file = OpenOptions::new()
            .create(true)
            .append(true)
            .open(rollout_dir.join("rollout.jsonl"))
            .await?;

        // metadata.json 작성
        let metadata = SessionMetadata {
            created_at: Utc::now(),
            model: config.model.clone(),
            model_provider: config.provider.name.clone(),
            cwd: config.cwd.clone(),
            session_source: config.session_source.clone(),
        };
        tokio::fs::write(
            rollout_dir.join("metadata.json"),
            serde_json::to_string_pretty(&metadata)?,
        )
        .await?;

        Ok(RolloutRecorder {
            path: rollout_dir,
            file: Arc::new(Mutex::new(rollout_file)),
            // ...
        })
    }
}
```

**디렉토리 구조**:
```
~/.codex/sessions/f47ac10b-58cc-4372-a567-0e02b2c3d479/
├── rollout.jsonl       # 대화 히스토리
├── metadata.json       # 세션 메타데이터
└── diffs/              # 파일 변경 diff들
    ├── 001.diff
    ├── 002.diff
    └── ...
```

**위치**: `codex-rs/core/src/rollout/mod.rs`

### 4단계: 대화 내역 기록

사용자 메시지와 어시스턴트 응답이 실시간으로 `rollout.jsonl`에 기록됩니다.

**코드**: `codex-rs/core/src/rollout/mod.rs`

```rust
impl RolloutRecorder {
    pub async fn append_user_input(&self, items: Vec<UserInput>) -> CodexResult<()> {
        let rollout_item = RolloutItem::UserInput { items };
        self.append_item(rollout_item).await
    }

    pub async fn append_assistant_message(&self, content: String) -> CodexResult<()> {
        let rollout_item = RolloutItem::AssistantMessage { content };
        self.append_item(rollout_item).await
    }

    pub async fn append_tool_call(
        &self,
        name: String,
        args: Value,
        result: ToolResult,
    ) -> CodexResult<()> {
        let rollout_item = RolloutItem::ToolCall { name, args, result };
        self.append_item(rollout_item).await
    }

    async fn append_item(&self, item: RolloutItem) -> CodexResult<()> {
        let json = serde_json::to_string(&item)?;
        let mut file = self.file.lock().await;

        // JSONL 형식으로 한 줄 추가
        file.write_all(json.as_bytes()).await?;
        file.write_all(b"\n").await?;

        // 즉시 디스크에 쓰기 (fsync)
        file.flush().await?;

        Ok(())
    }
}
```

**RolloutItem 종류**:
```rust
pub enum RolloutItem {
    UserInput { items: Vec<UserInput> },
    AssistantMessage { content: String },
    ToolCall { name: String, args: Value, result: ToolResult },
    FileChange { path: PathBuf, diff_path: PathBuf },
    TokenUsage { usage: TokenUsage },
}
```

**rollout.jsonl 예시**:
```jsonl
{"type":"user_input","items":[{"type":"text","text":"create a hello world"}]}
{"type":"assistant_message","content":"I'll create a hello world script."}
{"type":"tool_call","name":"Write","args":{"file_path":"hello.py","content":"print('Hello')"},"result":{"success":true}}
{"type":"assistant_message","content":"Done!"}
{"type":"token_usage","total":345,"input":234,"output":111}
```

**위치**: `codex-rs/core/src/rollout/mod.rs`

### 5단계: 세션 목록 관리

RolloutRecorder는 모든 세션의 메타데이터를 관리합니다.

**코드**: `codex-rs/core/src/rollout/mod.rs`

```rust
impl RolloutRecorder {
    pub async fn list_conversations(
        codex_home: &Path,
        limit: usize,
        cursor: Option<String>,
        session_sources: &[SessionSource],
        provider_filter: Option<&[String]>,
        default_provider: &str,
    ) -> CodexResult<ConversationPage> {
        let sessions_dir = codex_home.join("sessions");

        // 모든 세션 디렉토리 읽기
        let mut entries = tokio::fs::read_dir(&sessions_dir).await?;
        let mut conversations = Vec::new();

        while let Some(entry) = entries.next_entry().await? {
            let path = entry.path();
            if !path.is_dir() {
                continue;
            }

            // metadata.json 읽기
            let metadata_path = path.join("metadata.json");
            if let Ok(metadata_str) = tokio::fs::read_to_string(&metadata_path).await {
                if let Ok(metadata) = serde_json::from_str::<SessionMetadata>(&metadata_str) {
                    // 필터 적용
                    if !session_sources.is_empty() && !session_sources.contains(&metadata.session_source) {
                        continue;
                    }
                    if let Some(providers) = provider_filter {
                        if !providers.contains(&metadata.model_provider) {
                            continue;
                        }
                    }

                    conversations.push(ConversationInfo {
                        id: ConversationId::from_string(entry.file_name().to_string_lossy().to_string())?,
                        path: path.clone(),
                        created_at: metadata.created_at,
                        model: metadata.model,
                        preview: Self::get_preview(&path).await,
                    });
                }
            }
        }

        // 최신순 정렬
        conversations.sort_by(|a, b| b.created_at.cmp(&a.created_at));

        // 페이징
        let items = conversations.into_iter().take(limit).collect();

        Ok(ConversationPage { items, next_cursor: None })
    }

    async fn get_preview(path: &Path) -> String {
        // rollout.jsonl의 첫 사용자 메시지 읽기
        if let Ok(file) = File::open(path.join("rollout.jsonl")).await {
            let reader = BufReader::new(file);
            let mut lines = reader.lines();

            while let Ok(Some(line)) = lines.next_line().await {
                if let Ok(item) = serde_json::from_str::<RolloutItem>(&line) {
                    if let RolloutItem::UserInput { items } = item {
                        for input in items {
                            if let UserInput::Text { text } = input {
                                return text.chars().take(100).collect();
                            }
                        }
                    }
                }
            }
        }
        "No preview available".to_string()
    }
}
```

**사용 예시**:
```rust
// 최근 10개 세션 가져오기
let page = RolloutRecorder::list_conversations(
    &codex_home,
    10,
    None,
    &[SessionSource::Interactive],
    None,
    "openai",
)
.await?;

for conv in page.items {
    println!("{}: {}", conv.id, conv.preview);
}
```

**위치**: `codex-rs/core/src/rollout/mod.rs`

### 6단계: 대화 재개 (Resume)

사용자가 `codex resume <session-id>` 명령으로 이전 대화를 재개합니다.

**코드**: `codex-rs/core/src/conversation_manager.rs`

```rust
pub async fn resume_conversation_from_rollout(
    &self,
    config: Config,
    rollout_path: PathBuf,
    auth_manager: Arc<AuthManager>,
) -> CodexResult<NewConversation> {
    // Rollout 파일에서 히스토리 로드
    let initial_history = RolloutRecorder::get_rollout_history(&rollout_path).await?;

    self.resume_conversation_with_history(config, initial_history, auth_manager)
        .await
}

pub async fn resume_conversation_with_history(
    &self,
    config: Config,
    initial_history: InitialHistory,
    auth_manager: Arc<AuthManager>,
) -> CodexResult<NewConversation> {
    // 히스토리와 함께 Codex spawn
    let CodexSpawnOk {
        codex,
        conversation_id,
    } = Codex::spawn(
        config,
        auth_manager,
        initial_history,  // 기존 히스토리 전달
        self.session_source.clone(),
    )
    .await?;

    self.finalize_spawn(codex, conversation_id).await
}
```

**get_rollout_history() 동작**:
```rust
impl RolloutRecorder {
    pub async fn get_rollout_history(path: &Path) -> CodexResult<InitialHistory> {
        let rollout_file = path.join("rollout.jsonl");
        let file = File::open(&rollout_file).await?;
        let reader = BufReader::new(file);
        let mut lines = reader.lines();

        let mut items = Vec::new();

        while let Ok(Some(line)) = lines.next_line().await {
            if let Ok(item) = serde_json::from_str::<RolloutItem>(&line) {
                items.push(item);
            }
        }

        Ok(InitialHistory::Resume {
            path: path.to_path_buf(),
            items,
        })
    }
}
```

**Resume 시 동작**:
1. `rollout.jsonl` 파일을 전부 읽음
2. 각 라인을 `RolloutItem`으로 파싱
3. `InitialHistory::Resume`으로 감싸서 반환
4. `Session::new()`에서 이 히스토리를 사용
5. LLM에 전체 히스토리 전달하여 컨텍스트 복원

**위치**: `codex-rs/core/src/conversation_manager.rs:128-156`

### 7단계: 대화 포크 (Fork)

사용자가 대화의 특정 시점으로 되돌아가 새 분기를 만듭니다.

**코드**: `codex-rs/core/src/conversation_manager.rs`

```rust
pub async fn fork_conversation(
    &self,
    nth_user_message: usize,  // 몇 번째 사용자 메시지 이전으로 자르기
    config: Config,
    path: PathBuf,
) -> CodexResult<NewConversation> {
    // Rollout에서 히스토리 로드
    let history = RolloutRecorder::get_rollout_history(&path).await?;

    // n번째 사용자 메시지 이전까지만 자르기
    let history = truncate_before_nth_user_message(history, nth_user_message);

    // 새 대화 생성 (새 ID로)
    let auth_manager = self.auth_manager.clone();
    let CodexSpawnOk {
        codex,
        conversation_id,
    } = Codex::spawn(config, auth_manager, history, self.session_source.clone()).await?;

    self.finalize_spawn(codex, conversation_id).await
}
```

**truncate_before_nth_user_message() 동작**:
```rust
fn truncate_before_nth_user_message(history: InitialHistory, n: usize) -> InitialHistory {
    let items = history.get_rollout_items();

    // 사용자 메시지 인덱스 찾기
    let user_message_indices: Vec<usize> = items
        .iter()
        .enumerate()
        .filter_map(|(i, item)| match item {
            RolloutItem::UserInput { .. } => Some(i),
            _ => None,
        })
        .collect();

    // n번째 이전까지 자르기
    let cut_index = user_message_indices.get(n).copied().unwrap_or(items.len());
    let truncated_items: Vec<RolloutItem> = items.into_iter().take(cut_index).collect();

    InitialHistory::New { items: truncated_items }
}
```

**예시 시나리오**:
```
원본 대화:
1. User: "create a web server"
2. Assistant: "I'll create an Express server"
3. Tool: Write server.js
4. User: "add authentication"  <- 여기서 포크하고 싶음
5. Assistant: "Adding JWT auth..."
6. Tool: Write auth.js

fork_conversation(nth_user_message=1)을 호출하면:
- 새 대화가 4번 메시지 이전까지만 포함
- 5, 6번은 버림
- 새 UUID로 새 세션 생성
- 사용자가 다른 지시사항 입력 가능
```

**위치**: `codex-rs/core/src/conversation_manager.rs:173-224`

### 8단계: Resume Picker (TUI)

TUI 모드에서 `codex resume` 명령 시 대화 선택 UI가 표시됩니다.

**코드**: `codex-rs/tui/src/resume_picker.rs`

```rust
pub async fn run_resume_picker(
    tui: &mut Tui,
    codex_home: &Path,
    model_provider_id: &str,
) -> color_eyre::Result<ResumeSelection> {
    // 최근 세션 목록 가져오기
    let conversations = RolloutRecorder::list_conversations(
        codex_home,
        20,  // 최근 20개
        None,
        INTERACTIVE_SESSION_SOURCES,
        Some(&[model_provider_id.to_string()]),
        model_provider_id,
    )
    .await?;

    if conversations.items.is_empty() {
        return Ok(ResumeSelection::StartFresh);
    }

    // 선택 UI 표시
    let mut selected_index = 0;

    loop {
        tui.draw(|f| {
            render_conversation_list(f, f.area(), &conversations.items, selected_index);
        })?;

        // 키 입력 처리
        if let Some(Ok(event)) = tui.events.next().await {
            match event {
                Event::Key(KeyEvent { code: KeyCode::Up, .. }) => {
                    selected_index = selected_index.saturating_sub(1);
                }
                Event::Key(KeyEvent { code: KeyCode::Down, .. }) => {
                    selected_index = (selected_index + 1).min(conversations.items.len() - 1);
                }
                Event::Key(KeyEvent { code: KeyCode::Enter, .. }) => {
                    let selected = &conversations.items[selected_index];
                    return Ok(ResumeSelection::Resume(selected.path.clone()));
                }
                Event::Key(KeyEvent { code: KeyCode::Esc, .. }) => {
                    return Ok(ResumeSelection::Exit);
                }
                Event::Key(KeyEvent { code: KeyCode::Char('n'), .. }) => {
                    return Ok(ResumeSelection::StartFresh);
                }
                _ => {}
            }
        }
    }
}
```

**UI 레이아웃**:
```
┌─ Select a conversation to resume ────────────────────────────┐
│ > 2024-01-15 14:32 | create a web server with authentication  │
│   2024-01-15 12:18 | fix TypeScript type errors              │
│   2024-01-14 09:45 | implement user profile page             │
│   2024-01-13 16:22 | add database migrations                 │
│                                                                │
│ ↑/↓: Navigate | Enter: Resume | n: New conversation | Esc: Exit │
└───────────────────────────────────────────────────────────────┘
```

**위치**: `codex-rs/tui/src/resume_picker.rs`

### 9단계: 대화 삭제

세션을 삭제하려면 디렉토리를 제거합니다.

```rust
impl RolloutRecorder {
    pub async fn delete_conversation(codex_home: &Path, conversation_id: &ConversationId) -> CodexResult<()> {
        let session_dir = codex_home
            .join("sessions")
            .join(conversation_id.to_string());

        tokio::fs::remove_dir_all(&session_dir).await?;
        Ok(())
    }
}
```

**CLI 명령** (가상):
```bash
codex delete <session-id>
```

### 10단계: 세션 내보내기/가져오기

세션을 다른 기계로 이동하거나 백업할 수 있습니다.

**내보내기**:
```rust
pub async fn export_conversation(
    codex_home: &Path,
    conversation_id: &ConversationId,
    output_path: &Path,
) -> CodexResult<()> {
    let session_dir = codex_home.join("sessions").join(conversation_id.to_string());

    // TAR 아카이브 생성
    let tar_gz = File::create(output_path).await?;
    let encoder = GzipEncoder::new(tar_gz);
    let mut archive = tar::Builder::new(encoder);

    archive.append_dir_all(".", &session_dir)?;
    archive.finish()?;

    Ok(())
}
```

**가져오기**:
```rust
pub async fn import_conversation(
    codex_home: &Path,
    archive_path: &Path,
) -> CodexResult<ConversationId> {
    let sessions_dir = codex_home.join("sessions");

    // TAR 압축 해제
    let tar_gz = File::open(archive_path).await?;
    let decoder = GzipDecoder::new(tar_gz);
    let mut archive = tar::Archive::new(decoder);

    archive.unpack(&sessions_dir)?;

    // metadata.json에서 ID 읽기
    // ...

    Ok(conversation_id)
}
```

## 대화 상태 다이어그램

```
┌─────────────┐
│ 새 대화     │ <─ codex (서브커맨드 없음)
│ (New)       │
└──────┬──────┘
       │ 사용자 입력
       ▼
┌─────────────┐
│ 진행 중     │
│ (Active)    │
└──────┬──────┘
       │
       ├─ Ctrl+C 또는 exit
       │    ▼
       │  ┌─────────────┐
       │  │ 일시정지    │
       │  │ (Paused)    │
       │  └──────┬──────┘
       │         │ codex resume <id>
       │         └──────────────┐
       │                        │
       ▼                        ▼
┌─────────────┐          ┌─────────────┐
│ 완료        │          │ 재개        │
│ (Complete)  │          │ (Resumed)   │
└──────┬──────┘          └──────┬──────┘
       │                        │
       │ codex resume <id>      │ fork 가능
       └────────────┬───────────┘
                    │
                    ▼
              ┌─────────────┐
              │ 포크        │ <─ 새 ID 생성
              │ (Forked)    │
              └─────────────┘
```

## Rollout 파일 형식 상세

### rollout.jsonl
각 라인은 하나의 JSON 객체:
```jsonl
{"type":"user_input","items":[{"Text":{"text":"hello"}}]}
{"type":"assistant_message","content":"Hi there!"}
{"type":"tool_call","name":"Bash","args":{"command":["ls"]},"result":{"stdout":"file.txt\n","exit_code":0}}
```

### metadata.json
세션 메타정보:
```json
{
  "created_at": "2024-01-15T14:32:18Z",
  "updated_at": "2024-01-15T14:45:32Z",
  "model": "gpt-4-turbo",
  "model_provider": "openai",
  "cwd": "/home/user/project",
  "session_source": "Interactive",
  "total_tokens": 5432
}
```

### diffs/ 디렉토리
파일 변경 diff들 (unified diff 형식):
```
diffs/001.diff:
--- a/hello.py
+++ b/hello.py
@@ -1,1 +1,3 @@
-print("Hello")
+def greet():
+    print("Hello, World!")
+greet()
```

## 주요 코드 위치 요약

| 기능 | 함수/파일 | 위치 |
|-----|-----------|------|
| 새 대화 생성 | `ConversationManager::new_conversation()` | `codex-rs/core/src/conversation_manager.rs:59` |
| 대화 재개 | `ConversationManager::resume_conversation_from_rollout()` | `codex-rs/core/src/conversation_manager.rs:128` |
| 대화 포크 | `ConversationManager::fork_conversation()` | `codex-rs/core/src/conversation_manager.rs:173` |
| Rollout 초기화 | `RolloutRecorder::new()` | `codex-rs/core/src/rollout/mod.rs` |
| 히스토리 로드 | `RolloutRecorder::get_rollout_history()` | `codex-rs/core/src/rollout/mod.rs` |
| 세션 목록 | `RolloutRecorder::list_conversations()` | `codex-rs/core/src/rollout/mod.rs` |
| Resume Picker | `run_resume_picker()` | `codex-rs/tui/src/resume_picker.rs` |

## 다음 단계

- **[도구 실행 및 샌드박싱](05-tool-execution-and-sandboxing.md)**: 도구 실행 내부 메커니즘
- **[파일 작업 및 Rollout](07-file-operations-and-rollout.md)**: 파일 변경 추적 상세
