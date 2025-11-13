# 시나리오 5: 도구 실행 및 샌드박싱

## 시나리오 개요

LLM이 `Bash` 도구를 호출하여 시스템 명령을 실행하는 과정을 추적합니다. 도구 라우팅, 승인 정책, 샌드박스 보안, 플랫폼별 구현까지 전체 흐름을 다룹니다.

## 이야기로 따라가는 실행 흐름

### 1단계: LLM이 도구 호출 요청

LLM이 응답 스트림에서 도구 호출을 요청합니다.

**ResponseEvent**:
```json
{
  "type": "tool_call",
  "name": "Bash",
  "arguments": {
    "command": ["ls", "-la"],
    "description": "List files in current directory"
  }
}
```

**코드**: `codex-rs/core/src/response_processing.rs`

```rust
async fn process_response_stream(
    session: &mut Session,
    mut stream: impl Stream<Item = CodexResult<ResponseEvent>>,
) {
    while let Some(event) = stream.next().await {
        match event? {
            ResponseEvent::ToolCall { name, args } => {
                // 도구 실행
                let result = session.execute_tool(name, args).await?;
                // ...
            }
            // ...
        }
    }
}
```

**위치**: `codex-rs/core/src/response_processing.rs`

### 2단계: ToolRouter로 라우팅

Session이 ToolRouter에 도구 실행을 요청합니다.

**코드**: `codex-rs/core/src/tools/router.rs`

```rust
pub struct ToolRouter {
    tools: HashMap<String, Box<dyn Tool>>,
    mcp_manager: Arc<McpConnectionManager>,
}

impl ToolRouter {
    pub async fn execute(
        &self,
        name: &str,
        args: Value,
        context: &ToolContext,
    ) -> CodexResult<ToolResult> {
        // 내장 도구 확인
        if let Some(tool) = self.tools.get(name) {
            return tool.execute(args, context).await;
        }

        // MCP 도구 확인
        if let Some(mcp_result) = self.mcp_manager.try_execute(name, args).await? {
            return Ok(mcp_result);
        }

        Err(CodexErr::ToolNotFound(name.to_string()))
    }
}
```

**ToolContext**:
```rust
pub struct ToolContext {
    pub cwd: PathBuf,
    pub approval_policy: AskForApproval,
    pub sandbox_policy: SandboxPolicy,
    pub sandbox_type: SandboxType,
    pub codex_linux_sandbox_exe: Option<PathBuf>,
    pub stdout_stream: Option<StdoutStream>,
}
```

**위치**: `codex-rs/core/src/tools/router.rs`

### 3단계: Bash 도구 실행

BashTool이 호출됩니다.

**코드**: `codex-rs/core/src/tools/bash.rs`

```rust
pub struct BashTool;

#[async_trait]
impl Tool for BashTool {
    async fn execute(&self, args: Value, context: &ToolContext) -> CodexResult<ToolResult> {
        let params: BashParams = serde_json::from_value(args)?;

        // ExecParams 생성
        let exec_params = ExecParams {
            command: params.command,
            cwd: context.cwd.clone(),
            timeout_ms: params.timeout.or(Some(10_000)),
            env: HashMap::new(),
            with_escalated_permissions: None,
            justification: None,
            arg0: None,
        };

        // 실행
        let output = crate::exec::process_exec_tool_call(
            exec_params,
            context.sandbox_type,
            &context.sandbox_policy,
            &context.cwd,
            &context.codex_linux_sandbox_exe,
            context.stdout_stream.clone(),
        )
        .await?;

        Ok(ToolResult {
            stdout: output.stdout,
            stderr: output.stderr,
            exit_code: output.exit_code,
            duration_ms: output.duration.as_millis() as u64,
        })
    }
}
```

**위치**: `codex-rs/core/src/tools/bash.rs`

### 4단계: 승인 정책 확인

실행 전에 승인 정책을 확인합니다.

**코드**: `codex-rs/core/src/tools/sandboxing/approval.rs`

```rust
pub struct ApprovalStore {
    policy: AskForApproval,
    pending_requests: HashMap<String, oneshot::Sender<bool>>,
}

impl ApprovalStore {
    pub async fn check_approval(
        &mut self,
        command: &ExecParams,
        tx_event: &Sender<Event>,
    ) -> CodexResult<bool> {
        match self.policy {
            AskForApproval::Never => {
                // 항상 승인
                Ok(true)
            }
            AskForApproval::OnRequest => {
                // 항상 물어봄
                self.request_approval(command, tx_event).await
            }
            AskForApproval::OnDanger => {
                // 위험한 명령만 물어봄
                if is_dangerous_command(&command.command) {
                    self.request_approval(command, tx_event).await
                } else {
                    Ok(true)
                }
            }
        }
    }

    async fn request_approval(
        &mut self,
        command: &ExecParams,
        tx_event: &Sender<Event>,
    ) -> CodexResult<bool> {
        let request_id = uuid::Uuid::new_v4().to_string();
        let (tx, rx) = oneshot::channel();

        self.pending_requests.insert(request_id.clone(), tx);

        // UI에 승인 요청 이벤트 전송
        tx_event
            .send(Event {
                id: request_id.clone(),
                msg: EventMsg::ExecApprovalRequest(ExecApprovalRequestEvent {
                    request_id: request_id.clone(),
                    command: command.command.join(" "),
                    cwd: command.cwd.clone(),
                    justification: command.justification.clone(),
                }),
            })
            .await?;

        // 사용자 응답 대기
        let approved = rx.await?;

        self.pending_requests.remove(&request_id);

        Ok(approved)
    }
}

fn is_dangerous_command(command: &[String]) -> bool {
    let dangerous_keywords = ["rm", "del", "format", "dd", "mkfs", "sudo", "su"];
    command
        .first()
        .map(|cmd| dangerous_keywords.iter().any(|kw| cmd.contains(kw)))
        .unwrap_or(false)
}
```

**승인 요청 UI (TUI)**:
```
┌─ Approval Required ──────────────────────────────────────┐
│                                                           │
│ Command: rm -rf temp/                                     │
│ Working Directory: /home/user/project                     │
│                                                           │
│ Reason: Cleaning up temporary files                      │
│                                                           │
│ [A]pprove  [D]eny  [O]nce  [Always for this session]    │
└───────────────────────────────────────────────────────────┘
```

**위치**: `codex-rs/core/src/tools/sandboxing/approval.rs`

### 5단계: SandboxManager로 샌드박스 설정

승인되면 SandboxManager가 플랫폼별 샌드박스를 설정합니다.

**코드**: `codex-rs/core/src/sandboxing/mod.rs`

```rust
pub struct SandboxManager;

impl SandboxManager {
    pub fn new() -> Self {
        Self
    }

    pub fn transform(
        &self,
        spec: &CommandSpec,
        policy: &SandboxPolicy,
        sandbox_type: SandboxType,
        sandbox_cwd: &Path,
        linux_sandbox_exe: Option<&PathBuf>,
    ) -> Result<ExecEnv> {
        match sandbox_type {
            SandboxType::None => {
                // 샌드박스 없음
                Ok(ExecEnv {
                    command: spec.program.clone(),
                    args: spec.args.clone(),
                    cwd: spec.cwd.clone(),
                    env: spec.env.clone(),
                    timeout_ms: spec.timeout_ms,
                    sandbox: None,
                    // ...
                })
            }
            SandboxType::LinuxSeccomp => {
                self.transform_linux(spec, policy, sandbox_cwd, linux_sandbox_exe)
            }
            SandboxType::MacosSeatbelt => {
                self.transform_macos(spec, policy, sandbox_cwd)
            }
            SandboxType::WindowsRestrictedToken => {
                self.transform_windows(spec, policy, sandbox_cwd)
            }
        }
    }
}
```

**ExecEnv**:
```rust
pub struct ExecEnv {
    pub command: Vec<String>,  // 실행할 명령 (샌드박스 래퍼 포함 가능)
    pub cwd: PathBuf,
    pub env: HashMap<String, String>,
    pub timeout_ms: Option<u64>,
    pub sandbox: Option<SandboxInfo>,
    // ...
}

pub struct SandboxInfo {
    pub sandbox_type: SandboxType,
    pub allowed_paths: Vec<PathBuf>,
    pub network_enabled: bool,
}
```

**위치**: `codex-rs/core/src/sandboxing/mod.rs`

### 6단계: Linux 샌드박스 (Landlock + seccomp)

Linux에서는 Landlock과 seccomp을 사용합니다.

**코드**: `codex-rs/sandbox/linux-sandbox/src/main.rs`

```rust
fn main() -> Result<()> {
    // CLI 파싱
    let args = Args::parse();

    // Landlock 규칙 설정
    let mut ruleset = Ruleset::new()
        .handle_fs_access(AccessFs::Execute)
        .handle_fs_access(AccessFs::ReadFile)
        .handle_fs_access(AccessFs::ReadDir);

    // 정책에 따라 쓰기 권한 추가
    match args.policy {
        Policy::ReadOnly => {
            // 읽기 전용
        }
        Policy::WorkspaceWrite => {
            // 워크스페이스에만 쓰기
            ruleset = ruleset
                .handle_fs_access(AccessFs::WriteFile)
                .handle_fs_access(AccessFs::MakeDir);

            for path in &args.allowed_write_paths {
                ruleset = ruleset.add_rule(PathBeneath::new(path, AccessFs::WriteFile))?;
            }
        }
        Policy::FullAccess => {
            // 제한 없음
            ruleset = ruleset.handle_fs_access(AccessFs::WriteFile);
        }
    }

    // Landlock 적용
    ruleset.restrict_self()?;

    // seccomp BPF 필터 설정
    let mut filter = SeccompFilter::new();

    // 허용할 시스템 콜 목록
    let allowed_syscalls = [
        "read", "write", "open", "close", "stat", "fstat",
        "mmap", "mprotect", "munmap", "brk",
        "execve", "exit", "exit_group",
        "getpid", "getppid", "getuid", "geteuid",
        // ...
    ];

    for syscall in &allowed_syscalls {
        filter.allow_syscall(syscall);
    }

    // 네트워크 시스템 콜 제어
    if !args.allow_network {
        filter.deny_syscall("socket");
        filter.deny_syscall("connect");
        filter.deny_syscall("bind");
    }

    // seccomp 적용
    filter.apply()?;

    // 실제 명령 실행
    let err = std::process::Command::new(&args.command[0])
        .args(&args.command[1..])
        .current_dir(&args.cwd)
        .envs(&args.env)
        .exec();

    Err(err.into())
}
```

**Landlock**:
- 파일 시스템 접근 제어
- 경로별 세밀한 권한 설정
- 커널 5.13+ 필요

**seccomp**:
- 시스템 콜 화이트리스트
- 네트워크 접근 제어
- 프로세스 격리

**위치**: `codex-rs/sandbox/linux-sandbox/src/main.rs`

### 7단계: macOS 샌드박스 (Seatbelt)

macOS에서는 Seatbelt 프로파일을 사용합니다.

**코드**: `codex-rs/core/src/seatbelt.rs`

```rust
pub fn apply_seatbelt_profile(
    policy: &SandboxPolicy,
    sandbox_cwd: &Path,
) -> Result<()> {
    let profile = generate_seatbelt_profile(policy, sandbox_cwd);

    unsafe {
        // sandbox_init C 함수 호출
        let profile_cstr = CString::new(profile)?;
        let ret = sandbox_init(
            profile_cstr.as_ptr(),
            SANDBOX_NAMED,
            std::ptr::null_mut(),
        );

        if ret != 0 {
            return Err(CodexErr::SandboxError(format!(
                "sandbox_init failed: {}",
                std::io::Error::last_os_error()
            )));
        }
    }

    Ok(())
}

fn generate_seatbelt_profile(
    policy: &SandboxPolicy,
    sandbox_cwd: &Path,
) -> String {
    match policy {
        SandboxPolicy::ReadOnly => format!(
            r#"
            (version 1)
            (deny default)
            (allow process*)
            (allow file-read*)
            (allow sysctl-read)
            (allow mach-lookup)
            "#
        ),
        SandboxPolicy::WorkspaceWrite { allowed_paths } => {
            let mut profile = r#"
            (version 1)
            (deny default)
            (allow process*)
            (allow file-read*)
            (allow sysctl-read)
            (allow mach-lookup)
            "#.to_string();

            for path in allowed_paths {
                profile.push_str(&format!(
                    r#"(allow file-write* (subpath "{}"))
"#,
                    path.display()
                ));
            }

            profile
        }
        SandboxPolicy::DangerFullAccess => {
            "(version 1)\n(allow default)\n".to_string()
        }
    }
}
```

**Seatbelt 프로파일 예시**:
```scheme
(version 1)
(deny default)
(allow process*)
(allow file-read*)
(allow file-write* (subpath "/Users/user/project"))
(allow sysctl-read)
(allow mach-lookup)
```

**위치**: `codex-rs/core/src/seatbelt.rs`

### 8단계: Windows 샌드박스 (Restricted Token)

Windows에서는 Restricted Token을 사용합니다.

**코드**: `codex-rs/windows-sandbox/src/lib.rs`

```rust
pub fn run_windows_sandbox_capture(
    policy_str: &str,
    sandbox_cwd: &Path,
    command: Vec<String>,
    cwd: &Path,
    env: HashMap<String, String>,
    timeout_ms: Option<u64>,
    logs_dir: Option<PathBuf>,
) -> std::io::Result<(String, String, i32)> {
    unsafe {
        // 현재 프로세스 토큰 가져오기
        let mut token: HANDLE = std::ptr::null_mut();
        if OpenProcessToken(
            GetCurrentProcess(),
            TOKEN_ALL_ACCESS,
            &mut token,
        ) == 0 {
            return Err(std::io::Error::last_os_error());
        }

        // Restricted Token 생성
        let mut restricted_token: HANDLE = std::ptr::null_mut();

        // 제거할 권한 설정
        let sids_to_disable = match policy_str {
            "read-only" => {
                vec![
                    well_known_sid(WinBuiltinAdministratorsSid),
                    well_known_sid(WinWriteRestrictedCodeSid),
                ]
            }
            "workspace-write" => {
                vec![well_known_sid(WinBuiltinAdministratorsSid)]
            }
            _ => vec![],
        };

        CreateRestrictedToken(
            token,
            DISABLE_MAX_PRIVILEGE,
            sids_to_disable.len() as u32,
            sids_to_disable.as_ptr(),
            0,
            std::ptr::null(),
            0,
            std::ptr::null(),
            &mut restricted_token,
        );

        // 프로세스 생성
        let mut startup_info: STARTUPINFOW = std::mem::zeroed();
        startup_info.cb = std::mem::size_of::<STARTUPINFOW>() as u32;

        let mut process_info: PROCESS_INFORMATION = std::mem::zeroed();

        let command_line = command.join(" ");
        let mut command_line_wide = to_wide_string(&command_line);

        CreateProcessAsUserW(
            restricted_token,
            std::ptr::null(),
            command_line_wide.as_mut_ptr(),
            std::ptr::null_mut(),
            std::ptr::null_mut(),
            0,
            CREATE_NO_WINDOW,
            std::ptr::null_mut(),
            to_wide_string(&cwd.to_string_lossy()).as_ptr(),
            &mut startup_info,
            &mut process_info,
        );

        // 프로세스 대기
        WaitForSingleObject(process_info.hProcess, INFINITE);

        // Exit code 가져오기
        let mut exit_code: u32 = 0;
        GetExitCodeProcess(process_info.hProcess, &mut exit_code);

        CloseHandle(process_info.hProcess);
        CloseHandle(process_info.hThread);
        CloseHandle(restricted_token);
        CloseHandle(token);

        Ok(("".to_string(), "".to_string(), exit_code as i32))
    }
}
```

**Restricted Token**:
- SID (Security Identifier) 제거
- 권한 축소
- 파일 시스템 ACL 준수

**위치**: `codex-rs/windows-sandbox/src/lib.rs`

### 9단계: 명령 실행

샌드박스 설정이 완료되면 실제 명령을 실행합니다.

**코드**: `codex-rs/core/src/exec.rs::exec()`

```rust
async fn exec(
    params: ExecParams,
    sandbox: Option<SandboxInfo>,
    sandbox_policy: &SandboxPolicy,
    stdout_stream: Option<StdoutStream>,
) -> Result<RawExecToolCallOutput> {
    let timeout = params.timeout_duration();

    // 프로세스 스폰
    let mut child = spawn_child_async(
        &params.command,
        &params.cwd,
        &params.env,
        StdioPolicy::Piped,
    )
    .await?;

    // stdout/stderr 읽기 태스크 시작
    let stdout_task = tokio::spawn(stream_output(
        child.stdout.take().unwrap(),
        ExecOutputStream::Stdout,
        stdout_stream.clone(),
    ));

    let stderr_task = tokio::spawn(stream_output(
        child.stderr.take().unwrap(),
        ExecOutputStream::Stderr,
        stdout_stream.clone(),
    ));

    // 타임아웃과 함께 프로세스 대기
    let exit_status = tokio::select! {
        status = child.wait() => status?,
        _ = tokio::time::sleep(timeout) => {
            // 타임아웃 - 프로세스 kill
            child.kill().await?;
            return Err(CodexErr::ExecTimeout);
        }
    };

    // 출력 수집
    let stdout = stdout_task.await??;
    let stderr = stderr_task.await??;

    Ok(RawExecToolCallOutput {
        stdout,
        stderr,
        exit_code: exit_status.code().unwrap_or(-1),
    })
}

async fn stream_output(
    reader: impl AsyncRead + Unpin,
    stream: ExecOutputStream,
    stdout_stream: Option<StdoutStream>,
) -> Result<String> {
    let mut reader = BufReader::new(reader);
    let mut buffer = String::new();
    let mut accumulated = String::new();

    while reader.read_line(&mut buffer).await? > 0 {
        accumulated.push_str(&buffer);

        // UI에 실시간 전송
        if let Some(ref stream_info) = stdout_stream {
            stream_info
                .tx_event
                .send(Event {
                    id: stream_info.sub_id.clone(),
                    msg: EventMsg::ExecCommandOutputDelta(ExecCommandOutputDeltaEvent {
                        output: buffer.clone(),
                        stream,
                    }),
                })
                .await?;
        }

        buffer.clear();
    }

    Ok(accumulated)
}
```

**실행 과정**:
1. `tokio::process::Command`로 프로세스 스폰
2. stdout/stderr를 비동기로 읽기
3. 실시간으로 UI에 전송 (스트리밍)
4. 타임아웃 처리
5. exit code 수집

**위치**: `codex-rs/core/src/exec.rs:137-400`

### 10단계: 결과 반환

실행이 완료되면 결과를 LLM에 반환합니다.

**ToolResult**:
```rust
pub struct ToolResult {
    pub stdout: String,
    pub stderr: String,
    pub exit_code: i32,
    pub duration_ms: u64,
}
```

**LLM에 전달되는 형식**:
```json
{
  "role": "tool",
  "tool_call_id": "call_abc123",
  "content": "hello.py\nREADME.md\n",
  "metadata": {
    "exit_code": 0,
    "duration_ms": 123
  }
}
```

## 샌드박스 정책 비교

| 정책 | 파일 읽기 | 파일 쓰기 | 네트워크 | 프로세스 생성 |
|-----|----------|----------|---------|-------------|
| ReadOnly | ✓ | ✗ | ✗ | ✓ |
| WorkspaceWrite | ✓ | ✓ (워크스페이스만) | ✗ | ✓ |
| DangerFullAccess | ✓ | ✓ | ✓ | ✓ |

## 도구 목록

### 내장 도구

- **Bash**: 명령 실행
- **Read**: 파일 읽기
- **Write**: 파일 쓰기
- **Edit**: 파일 편집 (diff 기반)
- **Glob**: 파일 검색 (패턴)
- **Grep**: 텍스트 검색
- **ApplyPatch**: Git diff 적용
- **WebFetch**: URL 가져오기 (MCP 대체 가능)

### MCP 도구

- 설정에 따라 동적으로 추가
- 예: `database__query`, `slack__send_message`

## 다음 단계

- **[MCP 통합](06-mcp-integration.md)**: 외부 도구 통합
- **[파일 작업 및 Rollout](07-file-operations-and-rollout.md)**: 파일 변경 추적
