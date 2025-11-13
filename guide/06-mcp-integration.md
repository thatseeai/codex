# 시나리오 6: MCP 통합

## 시나리오 개요

Model Context Protocol (MCP)을 통해 외부 도구와 리소스를 Codex에 통합하는 과정을 추적합니다. MCP 서버 연결, 도구 등록, 도구 호출, 리소스 읽기까지의 전체 흐름을 다룹니다.

## MCP 개요

**Model Context Protocol (MCP)**는 AI 모델에 외부 도구와 데이터 소스를 연결하기 위한 프로토콜입니다.

**주요 개념**:
- **MCP Server**: 도구와 리소스를 제공하는 서버
- **MCP Client**: Codex (서버에 연결하여 도구 호출)
- **Tools**: 실행 가능한 함수
- **Resources**: 읽을 수 있는 데이터 (파일, DB 쿼리 결과 등)
- **Prompts**: 재사용 가능한 프롬프트 템플릿

## 이야기로 따라가는 실행 흐름

### 1단계: MCP 서버 설정

사용자가 `~/.codex/config.toml`에 MCP 서버를 설정합니다.

**설정 예시**:
```toml
[[mcp_servers]]
name = "database"
command = "npx"
args = ["-y", "@modelcontextprotocol/server-postgres"]
env = { DATABASE_URL = "postgresql://localhost/mydb" }

[[mcp_servers]]
name = "filesystem"
transport = { stdio = { command = "mcp-server-filesystem", args = ["/home/user/docs"] } }

[[mcp_servers]]
name = "slack"
transport = { sse = { url = "http://localhost:3000/mcp" } }
```

**Transport 종류**:
1. **stdio**: 표준 입력/출력으로 통신
2. **SSE (Server-Sent Events)**: HTTP 스트리밍

**위치**: 설정 파일 `~/.codex/config.toml`

### 2단계: McpConnectionManager 초기화

Session 초기화 시 McpConnectionManager가 생성됩니다.

**코드**: `codex-rs/core/src/mcp_connection_manager.rs`

```rust
pub struct McpConnectionManager {
    connections: Arc<RwLock<HashMap<String, McpConnection>>>,
    server_configs: Vec<McpServerConfig>,
}

impl McpConnectionManager {
    pub fn new(
        server_configs: Vec<McpServerConfig>,
        startup_timeout: Duration,
    ) -> Self {
        let manager = Self {
            connections: Arc::new(RwLock::new(HashMap::new())),
            server_configs,
        };

        // 백그라운드에서 모든 서버 연결 시작
        for config in &manager.server_configs {
            let manager_clone = manager.clone();
            let config_clone = config.clone();
            tokio::spawn(async move {
                if let Err(e) = manager_clone.connect_server(config_clone, startup_timeout).await {
                    tracing::error!("Failed to connect to MCP server {}: {}", config_clone.name, e);
                }
            });
        }

        manager
    }
}
```

**동작**:
1. 설정에서 모든 MCP 서버 읽기
2. 각 서버에 대해 연결 태스크 시작 (병렬)
3. 연결 실패는 로그만 남기고 계속 진행

**위치**: `codex-rs/core/src/mcp_connection_manager.rs`

### 3단계: MCP 서버 연결 (stdio)

stdio transport를 사용하는 서버에 연결합니다.

**코드**: `codex-rs/rmcp-client/src/client.rs`

```rust
pub async fn connect_stdio(
    command: &str,
    args: &[String],
    env: &HashMap<String, String>,
) -> Result<McpClient> {
    // 프로세스 스폰
    let mut child = tokio::process::Command::new(command)
        .args(args)
        .envs(env)
        .stdin(Stdio::piped())
        .stdout(Stdio::piped())
        .stderr(Stdio::inherit())
        .spawn()?;

    let stdin = child.stdin.take().unwrap();
    let stdout = child.stdout.take().unwrap();

    // JSON-RPC 트랜스포트 생성
    let transport = JsonRpcTransport::new(stdin, stdout);

    // Initialize 요청 전송
    transport
        .request("initialize", json!({
            "protocolVersion": "2024-11-05",
            "capabilities": {
                "roots": { "listChanged": true },
                "sampling": {}
            },
            "clientInfo": {
                "name": "codex",
                "version": env!("CARGO_PKG_VERSION")
            }
        }))
        .await?;

    // initialized 알림 전송
    transport.notify("initialized", json!({})).await?;

    Ok(McpClient {
        transport: Arc::new(transport),
        server_info: None,
    })
}
```

**JSON-RPC 예시**:
```json
// Request
{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05"}}

// Response
{"jsonrpc":"2.0","id":1,"result":{"protocolVersion":"2024-11-05","capabilities":{"tools":{}},"serverInfo":{"name":"postgres-server","version":"1.0.0"}}}
```

**위치**: `codex-rs/rmcp-client/src/client.rs`

### 4단계: 도구 목록 가져오기

연결 후 서버가 제공하는 도구 목록을 가져옵니다.

**코드**: `codex-rs/rmcp-client/src/client.rs`

```rust
impl McpClient {
    pub async fn list_tools(&self) -> Result<Vec<ToolInfo>> {
        let response: ListToolsResult = self
            .transport
            .request("tools/list", json!({}))
            .await?;

        Ok(response.tools)
    }
}
```

**응답 예시**:
```json
{
  "tools": [
    {
      "name": "database__query",
      "description": "Execute a SQL query",
      "inputSchema": {
        "type": "object",
        "properties": {
          "query": { "type": "string" }
        },
        "required": ["query"]
      }
    },
    {
      "name": "database__list_tables",
      "description": "List all tables in the database",
      "inputSchema": {
        "type": "object",
        "properties": {}
      }
    }
  ]
}
```

**위치**: `codex-rs/rmcp-client/src/client.rs`

### 5단계: 도구를 ToolRouter에 등록

McpConnectionManager가 모든 도구를 ToolRouter에 등록합니다.

**코드**: `codex-rs/core/src/mcp_connection_manager.rs`

```rust
impl McpConnectionManager {
    pub async fn get_all_tools(&self) -> Vec<ToolDefinition> {
        let connections = self.connections.read().await;
        let mut all_tools = Vec::new();

        for (server_name, connection) in connections.iter() {
            match connection.client.list_tools().await {
                Ok(tools) => {
                    for tool in tools {
                        all_tools.push(ToolDefinition {
                            name: format!("{}@_{}", server_name, tool.name),
                            description: tool.description,
                            parameters: tool.input_schema,
                        });
                    }
                }
                Err(e) => {
                    tracing::error!("Failed to list tools from {}: {}", server_name, e);
                }
            }
        }

        all_tools
    }
}
```

**도구 이름 규칙**:
- 서버 이름을 접두사로 사용: `{server_name}@_{tool_name}`
- 예: `database@_query`, `slack@_send_message`

**LLM에 전달되는 도구 정의**:
```json
{
  "type": "function",
  "function": {
    "name": "database@_query",
    "description": "Execute a SQL query",
    "parameters": {
      "type": "object",
      "properties": {
        "query": { "type": "string" }
      },
      "required": ["query"]
    }
  }
}
```

**위치**: `codex-rs/core/src/mcp_connection_manager.rs`

### 6단계: LLM이 MCP 도구 호출

LLM이 MCP 도구를 호출합니다.

**LLM 응답**:
```json
{
  "tool_calls": [
    {
      "id": "call_abc123",
      "type": "function",
      "function": {
        "name": "database@_query",
        "arguments": "{\"query\":\"SELECT * FROM users LIMIT 10\"}"
      }
    }
  ]
}
```

**ToolRouter 처리**:
```rust
impl ToolRouter {
    pub async fn execute(
        &self,
        name: &str,
        args: Value,
        context: &ToolContext,
    ) -> CodexResult<ToolResult> {
        // MCP 도구인지 확인
        if name.contains("@_") {
            let parts: Vec<&str> = name.splitn(2, "@_").collect();
            let server_name = parts[0];
            let tool_name = parts[1];

            return self
                .mcp_manager
                .call_tool(server_name, tool_name, args)
                .await;
        }

        // 내장 도구 처리
        // ...
    }
}
```

**위치**: `codex-rs/core/src/tools/router.rs`

### 7단계: MCP 도구 실행

McpConnectionManager가 실제 MCP 서버에 요청을 전송합니다.

**코드**: `codex-rs/rmcp-client/src/client.rs`

```rust
impl McpClient {
    pub async fn call_tool(
        &self,
        name: &str,
        arguments: Value,
    ) -> Result<CallToolResult> {
        let response: CallToolResult = self
            .transport
            .request(
                "tools/call",
                json!({
                    "name": name,
                    "arguments": arguments
                }),
            )
            .await?;

        Ok(response)
    }
}
```

**JSON-RPC 요청**:
```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "query",
    "arguments": {
      "query": "SELECT * FROM users LIMIT 10"
    }
  }
}
```

**JSON-RPC 응답**:
```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "id | name | email\n1 | Alice | alice@example.com\n2 | Bob | bob@example.com"
      }
    ]
  }
}
```

**위치**: `codex-rs/rmcp-client/src/client.rs`

### 8단계: MCP 리소스 읽기

MCP는 도구 외에도 리소스를 제공할 수 있습니다.

**리소스 목록 가져오기**:
```rust
impl McpClient {
    pub async fn list_resources(&self) -> Result<Vec<ResourceInfo>> {
        let response: ListResourcesResult = self
            .transport
            .request("resources/list", json!({}))
            .await?;

        Ok(response.resources)
    }
}
```

**응답 예시**:
```json
{
  "resources": [
    {
      "uri": "file:///home/user/docs/README.md",
      "name": "README",
      "description": "Project README",
      "mimeType": "text/markdown"
    },
    {
      "uri": "postgres://localhost/mydb/users",
      "name": "users_table",
      "description": "Users table schema"
    }
  ]
}
```

**리소스 읽기**:
```rust
impl McpClient {
    pub async fn read_resource(&self, uri: &str) -> Result<ResourceContents> {
        let response: ReadResourceResult = self
            .transport
            .request(
                "resources/read",
                json!({
                    "uri": uri
                }),
            )
            .await?;

        Ok(response.contents)
    }
}
```

**컨텍스트에 자동 추가**:
- 사용자가 "check the database schema" 요청
- Codex가 관련 리소스 자동 감지
- 프롬프트에 리소스 내용 추가

**위치**: `codex-rs/rmcp-client/src/client.rs`

### 9단계: SSE Transport (웹 서버)

SSE를 통해 HTTP로 MCP 서버에 연결합니다.

**코드**: `codex-rs/rmcp-client/src/sse.rs`

```rust
pub async fn connect_sse(url: &str) -> Result<McpClient> {
    // SSE 엔드포인트에 연결
    let client = reqwest::Client::new();
    let response = client
        .get(url)
        .header("Accept", "text/event-stream")
        .send()
        .await?;

    let mut stream = response.bytes_stream();

    // SSE 이벤트 파싱
    let (tx_request, rx_request) = mpsc::channel(100);
    let (tx_response, rx_response) = mpsc::channel(100);

    tokio::spawn(async move {
        while let Some(chunk) = stream.next().await {
            let chunk = chunk?;
            let text = String::from_utf8_lossy(&chunk);

            for line in text.lines() {
                if line.starts_with("data: ") {
                    let data = &line[6..];
                    let event: Value = serde_json::from_str(data)?;

                    if event.get("id").is_some() {
                        // Response
                        tx_response.send(event).await?;
                    } else {
                        // Notification
                        // Handle notification
                    }
                }
            }
        }
        Ok::<(), anyhow::Error>(())
    });

    // POST 요청으로 JSON-RPC 전송
    let transport = SseTransport {
        url: url.to_string(),
        client,
        tx_request,
        rx_response,
    };

    // Initialize
    let result = transport
        .request("initialize", json!({"protocolVersion": "2024-11-05"}))
        .await?;

    Ok(McpClient {
        transport: Arc::new(transport),
        server_info: Some(result),
    })
}
```

**SSE 이벤트 예시**:
```
data: {"jsonrpc":"2.0","id":1,"result":{"protocolVersion":"2024-11-05"}}

data: {"jsonrpc":"2.0","method":"notifications/resources/updated","params":{}}
```

**위치**: `codex-rs/rmcp-client/src/sse.rs`

### 10단계: Codex as MCP Server

Codex 자체도 MCP 서버로 동작하여 다른 클라이언트가 사용할 수 있습니다.

**실행**:
```bash
codex mcp-server
```

**제공하는 도구**:
- `read_file`: 파일 읽기
- `write_file`: 파일 쓰기
- `execute_command`: 명령 실행
- `search_files`: 파일 검색

**코드**: `codex-rs/mcp-server/src/main.rs`

```rust
pub async fn run_main(
    codex_linux_sandbox_exe: Option<PathBuf>,
    config_overrides: CliConfigOverrides,
) -> anyhow::Result<()> {
    // stdin/stdout로 JSON-RPC 통신
    let stdin = tokio::io::stdin();
    let stdout = tokio::io::stdout();

    let mut server = McpServer::new(stdin, stdout);

    // 도구 등록
    server.register_tool(
        "read_file",
        json!({
            "type": "object",
            "properties": {
                "path": {"type": "string"}
            },
            "required": ["path"]
        }),
        |args| async move {
            let path = args["path"].as_str().unwrap();
            let content = tokio::fs::read_to_string(path).await?;
            Ok(json!({ "content": content }))
        },
    );

    server.register_tool(
        "execute_command",
        json!({
            "type": "object",
            "properties": {
                "command": {"type": "array", "items": {"type": "string"}}
            },
            "required": ["command"]
        }),
        |args| async move {
            let command: Vec<String> = serde_json::from_value(args["command"].clone())?;
            let output = process_exec_tool_call(/* ... */).await?;
            Ok(json!({
                "stdout": output.stdout,
                "stderr": output.stderr,
                "exit_code": output.exit_code
            }))
        },
    );

    // 이벤트 루프 시작
    server.run().await?;

    Ok(())
}
```

**사용 예시**:
```toml
# 다른 AI 도구의 설정
[[mcp_servers]]
name = "codex"
transport = { stdio = { command = "codex", args = ["mcp-server"] } }
```

**위치**: `codex-rs/mcp-server/src/main.rs`

## MCP 아키텍처 다이어그램

```
┌─────────────────────────────────────────────────────────┐
│ Codex (MCP Client)                                      │
│                                                         │
│  ┌───────────────────────────────────────────────────┐ │
│  │ McpConnectionManager                              │ │
│  │                                                   │ │
│  │  ┌─────────────┐  ┌─────────────┐  ┌──────────┐ │ │
│  │  │ Connection  │  │ Connection  │  │Connection│ │ │
│  │  │ "database"  │  │ "filesystem"│  │ "slack"  │ │ │
│  │  └─────────────┘  └─────────────┘  └──────────┘ │ │
│  └───────────────────────────────────────────────────┘ │
│                                                         │
│  ┌───────────────────────────────────────────────────┐ │
│  │ ToolRouter                                        │ │
│  │  - database@_query                                │ │
│  │  - database@_list_tables                          │ │
│  │  - filesystem@_read                               │ │
│  │  - slack@_send_message                            │ │
│  └───────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
                           │
                           │ JSON-RPC
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
        ▼                  ▼                  ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ MCP Server   │  │ MCP Server   │  │ MCP Server   │
│ (stdio)      │  │ (stdio)      │  │ (SSE)        │
│              │  │              │  │              │
│ Postgres     │  │ Filesystem   │  │ Slack        │
│ Tool         │  │ Tool         │  │ Tool         │
└──────────────┘  └──────────────┘  └──────────────┘
```

## 주요 코드 위치 요약

| 기능 | 파일 | 위치 |
|-----|------|------|
| MCP 연결 관리 | `McpConnectionManager` | `codex-rs/core/src/mcp_connection_manager.rs` |
| MCP 클라이언트 | `McpClient` | `codex-rs/rmcp-client/src/client.rs` |
| stdio 트랜스포트 | `connect_stdio()` | `codex-rs/rmcp-client/src/client.rs` |
| SSE 트랜스포트 | `connect_sse()` | `codex-rs/rmcp-client/src/sse.rs` |
| MCP 서버 구현 | `run_main()` | `codex-rs/mcp-server/src/main.rs` |
| 도구 호출 | `call_tool()` | `codex-rs/rmcp-client/src/client.rs` |
| 리소스 읽기 | `read_resource()` | `codex-rs/rmcp-client/src/client.rs` |

## MCP 서버 예시

### 커스텀 MCP 서버 (Python)

```python
from mcp.server import Server, Tool
import asyncio

app = Server("my-custom-server")

@app.tool()
async def search_docs(query: str) -> str:
    """Search documentation"""
    # 실제 검색 로직
    return f"Search results for: {query}"

@app.resource("docs://readme")
async def get_readme() -> str:
    """Get README content"""
    with open("README.md") as f:
        return f.read()

if __name__ == "__main__":
    asyncio.run(app.run_stdio())
```

**Codex 설정**:
```toml
[[mcp_servers]]
name = "docs"
transport = { stdio = { command = "python", args = ["mcp_server.py"] } }
```

## 다음 단계

- **[파일 작업 및 Rollout](07-file-operations-and-rollout.md)**: 파일 변경 추적 및 저장
