# 시나리오 7: 파일 작업 및 Rollout

## 시나리오 개요

Codex가 파일을 읽고, 쓰고, 편집하는 과정과 모든 변경사항이 Rollout 시스템에 어떻게 추적되고 저장되는지 살펴봅니다. Diff 생성, 변경 추적, Git 통합까지 포함합니다.

## 이야기로 따라가는 실행 흐름

### 1단계: 파일 읽기 (Read 도구)

LLM이 파일을 읽기 위해 Read 도구를 호출합니다.

**LLM 요청**:
```json
{
  "tool_calls": [{
    "name": "Read",
    "arguments": {
      "file_path": "/home/user/project/src/main.rs"
    }
  }]
}
```

**코드**: `codex-rs/core/src/tools/read.rs`

```rust
pub struct ReadTool;

#[async_trait]
impl Tool for ReadTool {
    async fn execute(&self, args: Value, context: &ToolContext) -> CodexResult<ToolResult> {
        let params: ReadParams = serde_json::from_value(args)?;

        // 경로 검증
        let path = validate_path(&params.file_path, &context.cwd)?;

        // 샌드박스 정책 확인
        if !context.sandbox_policy.allows_read(&path) {
            return Err(CodexErr::SandboxViolation(format!(
                "Read access denied: {}",
                path.display()
            )));
        }

        // 파일 읽기
        let content = tokio::fs::read_to_string(&path).await?;

        // 줄 번호 추가
        let numbered_content = add_line_numbers(&content);

        Ok(ToolResult {
            content: numbered_content,
            metadata: json!({
                "lines": content.lines().count(),
                "size_bytes": content.len()
            }),
        })
    }
}

fn add_line_numbers(content: &str) -> String {
    content
        .lines()
        .enumerate()
        .map(|(i, line)| format!("{:4} | {}", i + 1, line))
        .collect::<Vec<_>>()
        .join("\n")
}
```

**결과**:
```
   1 | fn main() {
   2 |     println!("Hello, World!");
   3 | }
```

**LLM에 전달**:
- 줄 번호가 포함된 내용
- 편집 시 정확한 위치 지정 가능

**위치**: `codex-rs/core/src/tools/read.rs`

### 2단계: 파일 쓰기 (Write 도구)

새 파일을 생성하거나 기존 파일을 덮어씁니다.

**LLM 요청**:
```json
{
  "tool_calls": [{
    "name": "Write",
    "arguments": {
      "file_path": "/home/user/project/hello.py",
      "content": "print('Hello, World!')\n"
    }
  }]
}
```

**코드**: `codex-rs/core/src/tools/write.rs`

```rust
pub struct WriteTool;

#[async_trait]
impl Tool for WriteTool {
    async fn execute(&self, args: Value, context: &ToolContext) -> CodexResult<ToolResult> {
        let params: WriteParams = serde_json::from_value(args)?;

        let path = validate_path(&params.file_path, &context.cwd)?;

        // 쓰기 권한 확인
        if !context.sandbox_policy.allows_write(&path) {
            return Err(CodexErr::SandboxViolation(format!(
                "Write access denied: {}",
                path.display()
            )));
        }

        // 기존 파일 읽기 (diff 생성용)
        let old_content = if path.exists() {
            Some(tokio::fs::read_to_string(&path).await?)
        } else {
            None
        };

        // 부모 디렉토리 생성
        if let Some(parent) = path.parent() {
            tokio::fs::create_dir_all(parent).await?;
        }

        // 파일 쓰기
        tokio::fs::write(&path, &params.content).await?;

        // Diff 생성 및 저장
        let diff = generate_diff(&path, old_content.as_deref(), &params.content);
        let diff_path = context.save_diff(&diff).await?;

        // TurnDiffTracker에 변경사항 기록
        context.turn_diff_tracker.add_file_change(FileChange {
            path: path.clone(),
            diff_path,
            operation: if old_content.is_some() {
                FileOperation::Modified
            } else {
                FileOperation::Created
            },
        });

        Ok(ToolResult {
            content: format!("Successfully wrote to {}", path.display()),
            metadata: json!({
                "operation": if old_content.is_some() { "modified" } else { "created" },
                "bytes_written": params.content.len()
            }),
        })
    }
}
```

**위치**: `codex-rs/core/src/tools/write.rs`

### 3단계: Diff 생성

파일 변경사항을 unified diff 형식으로 생성합니다.

**코드**: `codex-rs/core/src/diff.rs`

```rust
pub fn generate_diff(
    file_path: &Path,
    old_content: Option<&str>,
    new_content: &str,
) -> String {
    use similar::TextDiff;

    let old = old_content.unwrap_or("");
    let new = new_content;

    let diff = TextDiff::from_lines(old, new);

    let mut output = String::new();

    // Diff 헤더
    output.push_str(&format!("--- a/{}\n", file_path.display()));
    output.push_str(&format!("+++ b/{}\n", file_path.display()));

    // Hunks
    for hunk in diff.unified_diff().iter_hunks() {
        output.push_str(&format!(
            "@@ -{},{} +{},{} @@\n",
            hunk.old_range().start + 1,
            hunk.old_range().len(),
            hunk.new_range().start + 1,
            hunk.new_range().len()
        ));

        for change in hunk.iter_changes() {
            match change.tag() {
                similar::ChangeTag::Delete => output.push_str(&format!("-{}", change)),
                similar::ChangeTag::Insert => output.push_str(&format!("+{}", change)),
                similar::ChangeTag::Equal => output.push_str(&format!(" {}", change)),
            }
        }
    }

    output
}
```

**Diff 예시**:
```diff
--- a/hello.py
+++ b/hello.py
@@ -1,1 +1,3 @@
-print('Hello')
+def greet():
+    print('Hello, World!')
+greet()
```

**위치**: `codex-rs/core/src/diff.rs`

### 4단계: TurnDiffTracker에 변경사항 기록

각 턴의 파일 변경사항을 추적합니다.

**코드**: `codex-rs/core/src/turn_diff_tracker.rs`

```rust
pub struct TurnDiffTracker {
    changes: Arc<Mutex<Vec<FileChange>>>,
}

impl TurnDiffTracker {
    pub fn new() -> Self {
        Self {
            changes: Arc::new(Mutex::new(Vec::new())),
        }
    }

    pub fn add_file_change(&self, change: FileChange) {
        let mut changes = self.changes.lock().unwrap();
        changes.push(change);
    }

    pub async fn get_changes(&self) -> Vec<FileChange> {
        let changes = self.changes.lock().unwrap();
        changes.clone()
    }

    pub async fn generate_turn_diff(&self) -> String {
        let changes = self.get_changes().await;
        let mut output = String::new();

        for change in changes {
            let diff_content = tokio::fs::read_to_string(&change.diff_path)
                .await
                .unwrap_or_default();

            output.push_str(&format!(
                "\n{} {}\n",
                match change.operation {
                    FileOperation::Created => "Created:",
                    FileOperation::Modified => "Modified:",
                    FileOperation::Deleted => "Deleted:",
                },
                change.path.display()
            ));
            output.push_str(&diff_content);
            output.push_str("\n");
        }

        output
    }
}

pub struct FileChange {
    pub path: PathBuf,
    pub diff_path: PathBuf,
    pub operation: FileOperation,
}

pub enum FileOperation {
    Created,
    Modified,
    Deleted,
}
```

**위치**: `codex-rs/core/src/turn_diff_tracker.rs`

### 5단계: Rollout에 변경사항 저장

턴이 완료되면 Rollout에 변경사항을 저장합니다.

**코드**: `codex-rs/core/src/rollout/mod.rs`

```rust
impl RolloutRecorder {
    pub async fn save_turn_changes(
        &self,
        turn_diff_tracker: &TurnDiffTracker,
    ) -> CodexResult<()> {
        let changes = turn_diff_tracker.get_changes().await;

        for change in changes {
            // Diff 파일을 rollout의 diffs/ 디렉토리로 복사
            let diff_filename = format!(
                "{:03}.diff",
                self.get_next_diff_number().await?
            );
            let dest_diff_path = self.path.join("diffs").join(&diff_filename);

            tokio::fs::create_dir_all(self.path.join("diffs")).await?;
            tokio::fs::copy(&change.diff_path, &dest_diff_path).await?;

            // rollout.jsonl에 기록
            self.append_item(RolloutItem::FileChange {
                path: change.path.clone(),
                diff_path: dest_diff_path,
                operation: change.operation,
            })
            .await?;
        }

        Ok(())
    }

    async fn get_next_diff_number(&self) -> CodexResult<usize> {
        let diffs_dir = self.path.join("diffs");
        if !diffs_dir.exists() {
            return Ok(1);
        }

        let mut entries = tokio::fs::read_dir(&diffs_dir).await?;
        let mut max_num = 0;

        while let Some(entry) = entries.next_entry().await? {
            if let Some(filename) = entry.file_name().to_str() {
                if let Some(num_str) = filename.strip_suffix(".diff") {
                    if let Ok(num) = num_str.parse::<usize>() {
                        max_num = max_num.max(num);
                    }
                }
            }
        }

        Ok(max_num + 1)
    }
}
```

**Rollout 구조**:
```
~/.codex/sessions/f47ac10b-58cc-4372-a567-0e02b2c3d479/
├── rollout.jsonl
├── metadata.json
└── diffs/
    ├── 001.diff  # 첫 번째 턴의 변경사항
    ├── 002.diff  # 두 번째 턴의 변경사항
    └── 003.diff
```

**rollout.jsonl 항목**:
```jsonl
{"type":"file_change","path":"/home/user/project/hello.py","diff_path":"diffs/001.diff","operation":"Created"}
```

**위치**: `codex-rs/core/src/rollout/mod.rs`

### 6단계: Edit 도구 (정교한 편집)

기존 파일의 일부만 수정할 때 Edit 도구를 사용합니다.

**LLM 요청**:
```json
{
  "tool_calls": [{
    "name": "Edit",
    "arguments": {
      "file_path": "/home/user/project/src/main.rs",
      "old_string": "println!(\"Hello, World!\");",
      "new_string": "println!(\"Hello, Codex!\");"
    }
  }]
}
```

**코드**: `codex-rs/core/src/tools/edit.rs`

```rust
pub struct EditTool;

#[async_trait]
impl Tool for EditTool {
    async fn execute(&self, args: Value, context: &ToolContext) -> CodexResult<ToolResult> {
        let params: EditParams = serde_json::from_value(args)?;

        let path = validate_path(&params.file_path, &context.cwd)?;

        // 파일 읽기
        let content = tokio::fs::read_to_string(&path).await?;

        // old_string 찾기
        let occurrences = content.matches(&params.old_string).count();

        if occurrences == 0 {
            return Err(CodexErr::EditFailed(
                "old_string not found in file".to_string()
            ));
        }

        if occurrences > 1 && !params.replace_all.unwrap_or(false) {
            return Err(CodexErr::EditFailed(format!(
                "old_string found {} times. Use replace_all=true or provide more context.",
                occurrences
            )));
        }

        // 교체
        let new_content = if params.replace_all.unwrap_or(false) {
            content.replace(&params.old_string, &params.new_string)
        } else {
            content.replacen(&params.old_string, &params.new_string, 1)
        };

        // 파일 쓰기
        tokio::fs::write(&path, &new_content).await?;

        // Diff 생성
        let diff = generate_diff(&path, Some(&content), &new_content);
        let diff_path = context.save_diff(&diff).await?;

        context.turn_diff_tracker.add_file_change(FileChange {
            path: path.clone(),
            diff_path,
            operation: FileOperation::Modified,
        });

        Ok(ToolResult {
            content: format!(
                "Successfully edited {} ({} replacement{})",
                path.display(),
                occurrences,
                if occurrences > 1 { "s" } else { "" }
            ),
            metadata: json!({
                "replacements": occurrences
            }),
        })
    }
}
```

**장점**:
- 전체 파일을 다시 쓰지 않아도 됨
- 정확한 위치 지정
- 작은 변경에 효율적

**위치**: `codex-rs/core/src/tools/edit.rs`

### 7단계: ApplyPatch 도구 (Git diff 적용)

Git 형식의 diff를 적용합니다.

**LLM 요청**:
```json
{
  "tool_calls": [{
    "name": "ApplyPatch",
    "arguments": {
      "patch": "--- a/hello.py\n+++ b/hello.py\n@@ -1,1 +1,3 @@\n-print('Hello')\n+def greet():\n+    print('Hello, World!')\n+greet()\n"
    }
  }]
}
```

**코드**: `codex-rs/apply-patch/src/lib.rs`

```rust
pub async fn apply_patch(
    patch_content: &str,
    cwd: &Path,
    dry_run: bool,
) -> Result<ApplyPatchResult> {
    // Patch 파싱
    let patches = parse_unified_diff(patch_content)?;

    let mut applied_files = Vec::new();
    let mut failed_files = Vec::new();

    for patch in patches {
        let file_path = cwd.join(&patch.file_path);

        // 파일 읽기
        let content = if file_path.exists() {
            tokio::fs::read_to_string(&file_path).await?
        } else {
            String::new()
        };

        // Patch 적용
        match apply_hunks(&content, &patch.hunks) {
            Ok(new_content) => {
                if !dry_run {
                    tokio::fs::write(&file_path, new_content).await?;
                }
                applied_files.push(patch.file_path);
            }
            Err(e) => {
                failed_files.push((patch.file_path, e.to_string()));
            }
        }
    }

    Ok(ApplyPatchResult {
        applied_files,
        failed_files,
    })
}

fn apply_hunks(content: &str, hunks: &[Hunk]) -> Result<String> {
    let mut lines: Vec<&str> = content.lines().collect();

    for hunk in hunks {
        let start = hunk.old_start - 1;

        // 컨텍스트 확인
        if !verify_context(&lines, start, &hunk.context_before) {
            return Err(anyhow::anyhow!("Context mismatch"));
        }

        // 제거할 라인 삭제
        for _ in &hunk.removed_lines {
            lines.remove(start);
        }

        // 추가할 라인 삽입
        for (i, line) in hunk.added_lines.iter().enumerate() {
            lines.insert(start + i, line);
        }
    }

    Ok(lines.join("\n"))
}
```

**위치**: `codex-rs/apply-patch/src/lib.rs`

### 8단계: 턴 종료 시 TurnDiffEvent 전송

턴이 완료되면 모든 변경사항을 요약한 이벤트를 전송합니다.

**코드**: `codex-rs/core/src/state/session_task.rs`

```rust
async fn finalize_turn(
    session: &mut Session,
    turn_diff_tracker: &TurnDiffTracker,
) -> CodexResult<()> {
    // 턴 diff 생성
    let turn_diff = turn_diff_tracker.generate_turn_diff().await;

    // UI에 이벤트 전송
    session
        .tx_event
        .send(Event {
            id: session.current_turn_id.clone(),
            msg: EventMsg::TurnDiff(TurnDiffEvent {
                diff: turn_diff.clone(),
                files_changed: turn_diff_tracker.get_changes().await.len(),
            }),
        })
        .await?;

    // Rollout에 저장
    session
        .services
        .rollout_recorder
        .save_turn_changes(turn_diff_tracker)
        .await?;

    Ok(())
}
```

**TurnDiffEvent UI 표시**:
```
┌─ Changes in this turn ──────────────────────────────┐
│ Modified: hello.py                                   │
│ --- a/hello.py                                       │
│ +++ b/hello.py                                       │
│ @@ -1,1 +1,3 @@                                      │
│ -print('Hello')                                      │
│ +def greet():                                        │
│ +    print('Hello, World!')                          │
│ +greet()                                             │
│                                                      │
│ 1 file changed                                       │
└──────────────────────────────────────────────────────┘
```

**위치**: `codex-rs/core/src/state/session_task.rs`

### 9단계: Git 통합

Git 저장소에서 작업 중이면 변경사항을 자동으로 추적합니다.

**코드**: `codex-rs/utils/git/src/lib.rs`

```rust
pub struct GitInfo {
    pub repo_root: PathBuf,
    pub current_branch: String,
    pub has_uncommitted_changes: bool,
    pub remote_url: Option<String>,
}

pub fn get_git_info(cwd: &Path) -> Option<GitInfo> {
    let repo_root = get_git_repo_root(cwd)?;

    let branch = std::process::Command::new("git")
        .args(["rev-parse", "--abbrev-ref", "HEAD"])
        .current_dir(&repo_root)
        .output()
        .ok()?
        .stdout;
    let current_branch = String::from_utf8_lossy(&branch).trim().to_string();

    let status = std::process::Command::new("git")
        .args(["status", "--porcelain"])
        .current_dir(&repo_root)
        .output()
        .ok()?
        .stdout;
    let has_uncommitted_changes = !status.is_empty();

    let remote = std::process::Command::new("git")
        .args(["config", "--get", "remote.origin.url"])
        .current_dir(&repo_root)
        .output()
        .ok()
        .and_then(|output| {
            if output.status.success() {
                Some(String::from_utf8_lossy(&output.stdout).trim().to_string())
            } else {
                None
            }
        });

    Some(GitInfo {
        repo_root,
        current_branch,
        has_uncommitted_changes,
        remote_url: remote,
    })
}

pub async fn get_git_diff(repo_root: &Path) -> Result<String> {
    let output = tokio::process::Command::new("git")
        .args(["diff", "HEAD"])
        .current_dir(repo_root)
        .output()
        .await?;

    Ok(String::from_utf8_lossy(&output.stdout).to_string())
}
```

**환경 컨텍스트에 포함**:
```rust
let git_info = get_git_info(&config.cwd);

let env_context = format!(
    r#"
    Working directory: {}
    Git branch: {}
    Uncommitted changes: {}
    "#,
    config.cwd.display(),
    git_info.as_ref().map(|g| g.current_branch.as_str()).unwrap_or("N/A"),
    git_info.as_ref().map(|g| g.has_uncommitted_changes).unwrap_or(false)
);
```

**위치**: `codex-rs/utils/git/src/lib.rs`

### 10단계: Apply 커맨드 (CLI에서 diff 적용)

사용자가 Codex 세션 후 `codex apply` 명령으로 변경사항을 적용할 수 있습니다.

**사용법**:
```bash
# 가장 최근 세션의 변경사항 적용
codex apply

# 특정 세션의 변경사항 적용
codex apply <session-id>

# Dry run (미리보기)
codex apply --dry-run
```

**코드**: `codex-rs/chatgpt/src/apply_command.rs`

```rust
pub async fn run_apply_command(
    cli: ApplyCommand,
    codex_home: Option<PathBuf>,
) -> anyhow::Result<()> {
    let codex_home = codex_home.unwrap_or_else(|| {
        dirs::home_dir()
            .unwrap()
            .join(".codex")
    });

    // 세션 ID 결정
    let session_id = if let Some(id) = cli.session_id {
        id
    } else {
        // 가장 최근 세션 찾기
        let conversations = RolloutRecorder::list_conversations(
            &codex_home,
            1,
            None,
            &[],
            None,
            "openai",
        )
        .await?;

        conversations
            .items
            .first()
            .ok_or_else(|| anyhow::anyhow!("No recent sessions found"))?
            .id
            .clone()
    };

    // Rollout 로드
    let rollout_path = codex_home.join("sessions").join(session_id.to_string());
    let history = RolloutRecorder::get_rollout_history(&rollout_path).await?;

    // 모든 diff 수집
    let mut all_diffs = String::new();

    for item in history.get_rollout_items() {
        if let RolloutItem::FileChange { diff_path, .. } = item {
            let diff = tokio::fs::read_to_string(diff_path).await?;
            all_diffs.push_str(&diff);
            all_diffs.push('\n');
        }
    }

    if all_diffs.is_empty() {
        println!("No file changes found in session.");
        return Ok(());
    }

    // Dry run이면 diff만 출력
    if cli.dry_run {
        println!("{}", all_diffs);
        return Ok(());
    }

    // Patch 적용
    let result = apply_patch(&all_diffs, &std::env::current_dir()?, false).await?;

    println!("Applied changes to {} files:", result.applied_files.len());
    for file in &result.applied_files {
        println!("  ✓ {}", file);
    }

    if !result.failed_files.is_empty() {
        eprintln!("\nFailed to apply changes to {} files:", result.failed_files.len());
        for (file, error) in &result.failed_files {
            eprintln!("  ✗ {}: {}", file, error);
        }
    }

    Ok(())
}
```

**위치**: `codex-rs/chatgpt/src/apply_command.rs`

## 파일 작업 흐름 다이어그램

```
┌────────────────────────────────────────────────────────┐
│ LLM 도구 호출                                          │
│ (Read/Write/Edit/ApplyPatch)                           │
└────────────────┬───────────────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────────┐
│ Tool 실행                                              │
│ - 파일 읽기/쓰기                                       │
│ - Diff 생성                                            │
└────────────────┬───────────────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────────┐
│ TurnDiffTracker                                        │
│ - 변경사항 누적                                        │
│ - FileChange 객체 저장                                 │
└────────────────┬───────────────────────────────────────┘
                 │
                 │ 턴 종료 시
                 ▼
┌────────────────────────────────────────────────────────┐
│ TurnDiffEvent 전송                                     │
│ - UI에 변경사항 요약 표시                              │
└────────────────┬───────────────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────────┐
│ RolloutRecorder                                        │
│ - diffs/ 디렉토리에 diff 파일 저장                     │
│ - rollout.jsonl에 FileChange 항목 추가                │
└────────────────────────────────────────────────────────┘
```

## 주요 코드 위치 요약

| 기능 | 파일 | 위치 |
|-----|------|------|
| Read 도구 | `ReadTool` | `codex-rs/core/src/tools/read.rs` |
| Write 도구 | `WriteTool` | `codex-rs/core/src/tools/write.rs` |
| Edit 도구 | `EditTool` | `codex-rs/core/src/tools/edit.rs` |
| Diff 생성 | `generate_diff()` | `codex-rs/core/src/diff.rs` |
| 변경사항 추적 | `TurnDiffTracker` | `codex-rs/core/src/turn_diff_tracker.rs` |
| Rollout 저장 | `RolloutRecorder::save_turn_changes()` | `codex-rs/core/src/rollout/mod.rs` |
| Patch 적용 | `apply_patch()` | `codex-rs/apply-patch/src/lib.rs` |
| Git 통합 | `get_git_info()` | `codex-rs/utils/git/src/lib.rs` |
| Apply 커맨드 | `run_apply_command()` | `codex-rs/chatgpt/src/apply_command.rs` |

## Rollout 파일 구조 상세

### 디렉토리 구조
```
~/.codex/sessions/<session-uuid>/
├── rollout.jsonl       # 전체 대화 히스토리
├── metadata.json       # 세션 메타데이터
└── diffs/              # 파일 변경 diff들
    ├── 001.diff        # 첫 번째 턴
    ├── 002.diff        # 두 번째 턴
    └── 003.diff        # 세 번째 턴
```

### rollout.jsonl 항목 종류
```jsonl
{"type":"user_input","items":[{"Text":{"text":"create hello.py"}}]}
{"type":"assistant_message","content":"I'll create hello.py"}
{"type":"tool_call","name":"Write","args":{"file_path":"hello.py","content":"print('Hello')\n"},"result":{"success":true}}
{"type":"file_change","path":"hello.py","diff_path":"diffs/001.diff","operation":"Created"}
{"type":"token_usage","total":234,"input":123,"output":111}
```

### diffs/001.diff 내용
```diff
--- a/hello.py
+++ b/hello.py
@@ -0,0 +1,1 @@
+print('Hello')
```

## 다음 단계

모든 시나리오 가이드를 완료했습니다! 다음을 참고하세요:

- **[개요](00-overview.md)**: 전체 아키텍처 요약
- **[CLI 시작 및 인증](01-cli-startup-and-authentication.md)**: 프로그램 시작 흐름
- **[대화형 TUI 모드](02-tui-interactive-mode.md)**: 실시간 대화 처리
- **[Exec 비대화형 모드](03-exec-non-interactive-mode.md)**: 자동화 워크플로우
- **[대화 생명주기](04-conversation-lifecycle.md)**: Resume, Fork, 히스토리 관리
- **[도구 실행 및 샌드박싱](05-tool-execution-and-sandboxing.md)**: 보안 실행
- **[MCP 통합](06-mcp-integration.md)**: 외부 도구 통합

코드베이스 기여 시:
1. `AGENTS.md` 읽기 - 개발 가이드라인
2. `docs/contributing.md` 참고 - 기여 프로세스
3. 각 시나리오 가이드로 내부 동작 이해
