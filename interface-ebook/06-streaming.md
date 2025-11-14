# 6장: 스트리밍 인터페이스

스트리밍(Streaming)은 LLM의 응답을 **전체가 완성될 때까지 기다리지 않고 실시간으로 조금씩 받아오는** 방식입니다. ChatGPT 웹 인터페이스처럼 텍스트가 타이핑되듯 나타나는 효과를 구현할 수 있습니다.

## 왜 스트리밍이 필요한가?

### 일괄 응답 (Non-streaming)

```python
response = client.chat.completions.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "긴 이야기를 써주세요"}]
)

# ⏳ 10초 대기...

print(response.choices[0].message.content)
# 전체 응답이 한 번에 출력
```

**문제점**:
- ❌ 사용자가 오랫동안 기다려야 함
- ❌ 응답이 멈춘 것처럼 보임
- ❌ UX 저하

### 스트리밍 응답

```python
stream = client.chat.completions.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "긴 이야기를 써주세요"}],
    stream=True
)

for chunk in stream:
    if chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="", flush=True)

# "옛" "날" " " "옛" "적" "에" " " ...
# 실시간으로 단어가 출력됨!
```

**장점**:
- ✅ 즉각적인 피드백
- ✅ 더 나은 사용자 경험
- ✅ 긴 응답에도 반응성 유지
- ✅ 중간에 취소 가능

## OpenAI 스트리밍

### 기본 사용법

```python
from openai import OpenAI

client = OpenAI()

stream = client.chat.completions.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "Python이 뭐야?"}],
    stream=True
)

for chunk in stream:
    if chunk.choices[0].delta.content is not None:
        print(chunk.choices[0].delta.content, end="")
```

### 청크(Chunk) 구조

각 청크는 다음과 같은 구조를 가집니다:

```python
{
  "id": "chatcmpl-abc123",
  "object": "chat.completion.chunk",  # "chunk"임에 주목
  "created": 1699564800,
  "model": "gpt-4-0613",
  "choices": [
    {
      "index": 0,
      "delta": {
        "role": "assistant",     # 첫 청크에만
        "content": "Python은"     # 실제 텍스트 조각
      },
      "finish_reason": null      # 마지막 청크에서만 "stop" 등
    }
  ]
}
```

**청크 시퀀스 예시**:

```python
# Chunk 1
{"delta": {"role": "assistant", "content": ""}, "finish_reason": null}

# Chunk 2
{"delta": {"content": "Python"}, "finish_reason": null}

# Chunk 3
{"delta": {"content": "은"}, "finish_reason": null}

# Chunk 4
{"delta": {"content": " 프로그래밍"}, "finish_reason": null}

# ...

# 마지막 Chunk
{"delta": {}, "finish_reason": "stop"}
```

### 전체 응답 재구성

```python
stream = client.chat.completions.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "안녕"}],
    stream=True
)

full_response = ""
for chunk in stream:
    if chunk.choices[0].delta.content:
        content = chunk.choices[0].delta.content
        full_response += content
        print(content, end="", flush=True)

print(f"\n\n전체 응답: {full_response}")
```

### finish_reason 처리

```python
for chunk in stream:
    choice = chunk.choices[0]

    if choice.delta.content:
        print(choice.delta.content, end="", flush=True)

    if choice.finish_reason == "stop":
        print("\n✓ 정상 완료")
    elif choice.finish_reason == "length":
        print("\n⚠ max_tokens 도달")
    elif choice.finish_reason == "function_call":
        print("\n🔧 함수 호출 요청")
    elif choice.finish_reason == "content_filter":
        print("\n🚫 컨텐츠 필터링")
```

## Function Calling과 스트리밍

스트리밍 모드에서도 함수 호출이 가능합니다!

```python
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "날씨 정보 조회",
            "parameters": {
                "type": "object",
                "properties": {
                    "location": {"type": "string"}
                },
                "required": ["location"]
            }
        }
    }
]

stream = client.chat.completions.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "서울 날씨는?"}],
    tools=tools,
    stream=True
)

function_name = ""
function_args = ""

for chunk in stream:
    delta = chunk.choices[0].delta

    # 함수 호출 정보가 조각나서 옴
    if delta.tool_calls:
        for tool_call in delta.tool_calls:
            if tool_call.function.name:
                function_name += tool_call.function.name
            if tool_call.function.arguments:
                function_args += tool_call.function.arguments

    if chunk.choices[0].finish_reason == "tool_calls":
        print(f"함수 호출: {function_name}({function_args})")
        # 함수 실행 로직...
```

## Server-Sent Events (SSE)

스트리밍은 내부적으로 **Server-Sent Events** 프로토콜을 사용합니다.

### SSE 형식

```
data: {"id":"chatcmpl-123","object":"chat.completion.chunk","choices":[{"delta":{"content":"Hello"},"finish_reason":null}]}

data: {"id":"chatcmpl-123","object":"chat.completion.chunk","choices":[{"delta":{"content":" world"},"finish_reason":null}]}

data: {"id":"chatcmpl-123","object":"chat.completion.chunk","choices":[{"delta":{},"finish_reason":"stop"}]}

data: [DONE]
```

### 직접 SSE 처리 (고급)

```python
import requests
import json

response = requests.post(
    "https://api.openai.com/v1/chat/completions",
    headers={
        "Authorization": f"Bearer {api_key}",
        "Content-Type": "application/json"
    },
    json={
        "model": "gpt-4",
        "messages": [{"role": "user", "content": "Hello"}],
        "stream": True
    },
    stream=True
)

for line in response.iter_lines():
    if line:
        line = line.decode('utf-8')
        if line.startswith("data: "):
            data = line[6:]  # "data: " 제거
            if data == "[DONE]":
                break
            chunk = json.loads(data)
            if chunk["choices"][0]["delta"].get("content"):
                print(chunk["choices"][0]["delta"]["content"], end="")
```

## Anthropic 스트리밍

### 기본 사용법

```python
import anthropic

client = anthropic.Anthropic()

with client.messages.stream(
    model="claude-3-5-sonnet-20241022",
    max_tokens=1024,
    messages=[{"role": "user", "content": "안녕하세요"}]
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)
```

### 이벤트 기반 처리

```python
with client.messages.stream(
    model="claude-3-5-sonnet-20241022",
    max_tokens=1024,
    messages=[{"role": "user", "content": "안녕하세요"}]
) as stream:
    for event in stream:
        if event.type == "content_block_start":
            print("응답 시작")
        elif event.type == "content_block_delta":
            print(event.delta.text, end="")
        elif event.type == "content_block_stop":
            print("\n응답 완료")
        elif event.type == "message_stop":
            print(f"토큰 사용: {stream.message.usage}")
```

## 웹 애플리케이션 통합

### FastAPI + SSE

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
from openai import OpenAI
import json

app = FastAPI()
client = OpenAI()

@app.get("/chat")
async def chat_stream(message: str):
    def generate():
        stream = client.chat.completions.create(
            model="gpt-4",
            messages=[{"role": "user", "content": message}],
            stream=True
        )

        for chunk in stream:
            if chunk.choices[0].delta.content:
                # SSE 형식으로 전송
                data = {
                    "content": chunk.choices[0].delta.content,
                    "done": False
                }
                yield f"data: {json.dumps(data)}\n\n"

        # 완료 신호
        yield f"data: {json.dumps({'done': True})}\n\n"

    return StreamingResponse(
        generate(),
        media_type="text/event-stream"
    )
```

### JavaScript 클라이언트

```javascript
// EventSource API 사용
const eventSource = new EventSource('/chat?message=안녕하세요');

eventSource.onmessage = (event) => {
    const data = JSON.parse(event.data);

    if (data.done) {
        eventSource.close();
        console.log('응답 완료');
    } else {
        // 화면에 텍스트 추가
        document.getElementById('response').textContent += data.content;
    }
};

eventSource.onerror = (error) => {
    console.error('SSE 에러:', error);
    eventSource.close();
};
```

### Fetch API + Streaming

```javascript
async function streamChat(message) {
    const response = await fetch('/chat', {
        method: 'POST',
        headers: {'Content-Type': 'application/json'},
        body: JSON.stringify({message})
    });

    const reader = response.body.getReader();
    const decoder = new TextDecoder();

    while (true) {
        const {done, value} = await reader.read();
        if (done) break;

        const chunk = decoder.decode(value);
        const lines = chunk.split('\n');

        for (const line of lines) {
            if (line.startsWith('data: ')) {
                const data = JSON.parse(line.slice(6));
                if (data.content) {
                    document.getElementById('response').textContent += data.content;
                }
            }
        }
    }
}
```

## React 통합 예제

```typescript
import { useState } from 'react';
import OpenAI from 'openai';

export function ChatComponent() {
    const [response, setResponse] = useState('');
    const [isStreaming, setIsStreaming] = useState(false);

    const handleSubmit = async (message: string) => {
        setIsStreaming(true);
        setResponse('');

        const client = new OpenAI({
            apiKey: process.env.OPENAI_API_KEY,
            dangerouslyAllowBrowser: true  // 프로덕션에서는 백엔드 사용!
        });

        const stream = await client.chat.completions.create({
            model: 'gpt-4',
            messages: [{role: 'user', content: message}],
            stream: true
        });

        for await (const chunk of stream) {
            const content = chunk.choices[0]?.delta?.content || '';
            setResponse(prev => prev + content);
        }

        setIsStreaming(false);
    };

    return (
        <div>
            <div className="response">
                {response}
                {isStreaming && <span className="cursor">▋</span>}
            </div>
            <button onClick={() => handleSubmit('안녕하세요')}>
                전송
            </button>
        </div>
    );
}
```

## 에러 처리 및 재연결

### Timeout 처리

```python
import signal

class TimeoutException(Exception):
    pass

def timeout_handler(signum, frame):
    raise TimeoutException()

# 타임아웃 설정 (30초)
signal.signal(signal.SIGALRM, timeout_handler)
signal.alarm(30)

try:
    stream = client.chat.completions.create(
        model="gpt-4",
        messages=[{"role": "user", "content": "긴 응답 요청"}],
        stream=True
    )

    for chunk in stream:
        if chunk.choices[0].delta.content:
            print(chunk.choices[0].delta.content, end="")

    signal.alarm(0)  # 타임아웃 해제

except TimeoutException:
    print("\n⏱ 응답 시간 초과")
except Exception as e:
    print(f"\n❌ 에러 발생: {e}")
```

### 재시도 로직

```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=2, max=10)
)
def stream_with_retry(messages):
    stream = client.chat.completions.create(
        model="gpt-4",
        messages=messages,
        stream=True
    )

    response = ""
    for chunk in stream:
        if chunk.choices[0].delta.content:
            content = chunk.choices[0].delta.content
            response += content
            yield content

    return response

# 사용
try:
    for content in stream_with_retry([{"role": "user", "content": "안녕"}]):
        print(content, end="")
except Exception as e:
    print(f"최종 실패: {e}")
```

## 성능 최적화

### 버퍼링

```python
import time

class StreamBuffer:
    def __init__(self, flush_interval=0.1):
        self.buffer = []
        self.flush_interval = flush_interval
        self.last_flush = time.time()

    def add(self, text):
        self.buffer.append(text)

        # 일정 시간마다 또는 버퍼가 차면 플러시
        if time.time() - self.last_flush > self.flush_interval or len(self.buffer) > 10:
            self.flush()

    def flush(self):
        if self.buffer:
            print(''.join(self.buffer), end='', flush=True)
            self.buffer = []
            self.last_flush = time.time()

# 사용
buffer = StreamBuffer()

stream = client.chat.completions.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "긴 응답"}],
    stream=True
)

for chunk in stream:
    if chunk.choices[0].delta.content:
        buffer.add(chunk.choices[0].delta.content)

buffer.flush()  # 마지막 버퍼 비우기
```

## 중간 취소

```python
import threading

class CancellableStream:
    def __init__(self):
        self.cancelled = False

    def cancel(self):
        self.cancelled = True

    def stream(self, messages):
        stream = client.chat.completions.create(
            model="gpt-4",
            messages=messages,
            stream=True
        )

        for chunk in stream:
            if self.cancelled:
                print("\n⏹ 스트리밍 취소됨")
                break

            if chunk.choices[0].delta.content:
                print(chunk.choices[0].delta.content, end="")

# 사용
cs = CancellableStream()

# 별도 스레드에서 스트리밍
thread = threading.Thread(
    target=cs.stream,
    args=([{"role": "user", "content": "매우 긴 이야기를 써주세요"}],)
)
thread.start()

# 사용자가 취소 버튼을 누르면
# cs.cancel()
```

## Codex CLI에서의 실제 구현

Codex CLI는 **모든 요청을 스트리밍으로 처리**합니다. 일괄 응답 모드는 사용하지 않습니다.

### 스트리밍 요청 설정

**위치**: `codex-rs/core/src/chat_completions.rs:331-370`

```rust
// 페이로드에 stream: true 설정
let payload = json!({
    "model": model_family.slug,
    "messages": messages,
    "stream": true,  // 항상 스트리밍
    "tools": tools_json,
});

// SSE (Server-Sent Events) 수락
let req_builder = provider
    .create_request_builder(client, &None)
    .await?
    .header(reqwest::header::ACCEPT, "text/event-stream")  // SSE
    .json(&payload);

let response = req_builder.send().await?;
```

**포인트**:
- `Accept: text/event-stream` 헤더로 SSE 요청
- 모든 Chat Completion 호출이 스트리밍

### 스트림 처리 아키텍처

**위치**: `codex-rs/core/src/chat_completions.rs:374-388`

```rust
if resp.status().is_success() {
    // 채널 생성 (비동기 메시지 전달)
    let (tx_event, rx_event) = mpsc::channel::<Result<ResponseEvent>>(1600);

    // 바이트 스트림 생성
    let stream = resp.bytes_stream().map_err(|e| {
        CodexErr::ResponseStreamFailed(ResponseStreamFailed {
            source: e,
            request_id: None,
        })
    });

    // 백그라운드 태스크에서 SSE 처리
    tokio::spawn(process_chat_sse(
        stream,
        tx_event,
        provider.stream_idle_timeout(),
        otel_event_manager.clone(),
    ));

    // 스트림 반환 (비동기 채널 수신 측)
    return Ok(ResponseStream { rx_event });
}
```

**아키텍처**:
```
HTTP Response
     ↓
Bytes Stream ──→ [Background Task: process_chat_sse]
                          ↓
                   Parse SSE Events
                          ↓
                   Parse JSON Chunks
                          ↓
                   mpsc::channel (tx_event)
                          ↓
     ┌────────────────────┘
     ↓
ResponseStream (rx_event) ──→ Main Application
```

**포인트**:
- 별도 Tokio 태스크에서 SSE 파싱
- `mpsc::channel`로 메인 스레드와 통신
- 버퍼 크기 1600으로 백프레셔 관리

### SSE 파싱 (실제 구현)

**위치**: `codex-rs/core/src/chat_completions.rs:430+` (process_chat_sse 함수)

```rust
async fn process_chat_sse(
    stream: impl Stream<Item = Result<Bytes>>,
    tx_event: mpsc::Sender<Result<ResponseEvent>>,
    idle_timeout: Duration,
    otel_manager: OtelEventManager,
) {
    let mut event_stream = stream.eventsource();

    loop {
        // 타임아웃으로 스트림 읽기
        let event = match timeout(idle_timeout, event_stream.next()).await {
            Ok(Some(Ok(event))) => event,
            Ok(Some(Err(e))) => {
                // 스트림 에러
                let _ = tx_event.send(Err(parse_error(e))).await;
                break;
            }
            Ok(None) | Err(_) => {
                // 스트림 종료 또는 타임아웃
                break;
            }
        };

        // "[DONE]" 체크
        if event.data == "[DONE]" {
            let _ = tx_event.send(Ok(ResponseEvent::Done)).await;
            break;
        }

        // JSON 파싱
        let chunk: ChatCompletionChunk = match serde_json::from_str(&event.data) {
            Ok(c) => c,
            Err(e) => {
                let _ = tx_event.send(Err(CodexErr::ParseError(e))).await;
                continue;
            }
        };

        // 델타 처리
        if let Some(delta) = chunk.choices[0].delta.content {
            // 텍스트 델타를 이벤트로 전송
            let _ = tx_event.send(Ok(ResponseEvent::TextDelta(delta))).await;
        }
    }
}
```

**포인트**:
- `eventsource-stream` crate로 SSE 파싱
- Idle timeout으로 무한 대기 방지
- 각 청크를 `ResponseEvent`로 변환

### Idle Timeout 처리

**위치**: `codex-rs/core/src/model_provider_info.rs` (추정)

```rust
impl ModelProviderInfo {
    pub fn stream_idle_timeout(&self) -> Duration {
        // 스트림이 60초 동안 데이터 없으면 타임아웃
        Duration::from_secs(60)
    }
}
```

**동작**:
```rust
let event = timeout(idle_timeout, event_stream.next()).await;
```

- 60초 동안 청크가 오지 않으면 스트림 중단
- 느린 생성이나 네트워크 문제 감지

### 재시도와 스트리밍

**위치**: `codex-rs/core/src/chat_completions.rs:390-430`

```rust
// 429 (Rate Limit) 또는 5xx 에러 시
Ok(res) if status == StatusCode::TOO_MANY_REQUESTS
         || status.is_server_error() => {

    if attempt > max_retries {
        return Err(CodexErr::RetryLimit(...));
    }

    // Retry-After 헤더 확인
    let retry_after_secs = res
        .headers()
        .get(reqwest::header::RETRY_AFTER)
        .and_then(|v| v.to_str().ok())
        .and_then(|s| s.parse::<u64>().ok());

    // Retry-After 또는 exponential backoff
    let delay = retry_after_secs
        .map(|s| Duration::from_millis(s * 1_000))
        .unwrap_or_else(|| backoff(attempt));

    tokio::time::sleep(delay).await;
    // 재시도...
}
```

**포인트**:
- 스트리밍 시작 전 에러는 재시도 가능
- 스트리밍 시작 후 에러는 재시도 불가 (별도 처리)
- `Retry-After` 헤더 우선 사용

### 실시간 이벤트 전달

```rust
// TUI에서 스트림 소비
let mut stream = codex.send_message(user_input).await?;

while let Some(event) = stream.next().await {
    match event? {
        ResponseEvent::TextDelta(text) => {
            // 화면에 즉시 출력
            print!("{}", text);
            stdout().flush()?;
        }
        ResponseEvent::Done => {
            println!("\n[완료]");
            break;
        }
        _ => {}
    }
}
```

**사용자 경험**:
- ChatGPT처럼 타이핑 효과
- 즉각적인 피드백
- 취소 가능 (Ctrl+C)

**학습 포인트**:
- ✅ **스트리밍 우선 설계**: 모든 요청이 스트리밍
- ✅ **비동기 아키텍처**: Tokio + mpsc 채널
- ✅ **타임아웃 관리**: Idle timeout으로 무한 대기 방지
- ✅ **재시도 로직**: 스트림 시작 전 에러만 재시도
- ✅ **Retry-After 지원**: 서버 지시 존중
- ✅ **에러 처리**: 파싱 에러, 네트워크 에러 등 모두 처리

## 다음 단계

스트리밍을 마스터했습니다! 이제 **보이는 것과 숨겨진 것**에서 시스템 프롬프트, 메타데이터 등 인터페이스의 숨겨진 레이어를 파헤쳐봅시다.

➡️ [7장: 보이는 것과 숨겨진 것](./07-visible-vs-hidden.md)

---

## 핵심 요약

- ✅ 스트리밍은 `stream=True`로 활성화
- ✅ 응답이 청크(chunk) 단위로 실시간 전달됨
- ✅ `delta.content`에 텍스트 조각 포함
- ✅ `finish_reason`으로 완료 상태 확인
- ✅ Server-Sent Events (SSE) 프로토콜 사용
- ✅ 함수 호출도 스트리밍 가능
- ✅ 웹앱 통합 시 EventSource 또는 Fetch API 사용
- ✅ 타임아웃, 재시도, 취소 로직 구현 권장
