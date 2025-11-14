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
