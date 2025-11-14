# 11장: 부록 - 전체 레퍼런스

## A. 전체 Chat Completion 스키마

### 요청 (Request) 전체 파라미터

```typescript
interface ChatCompletionRequest {
  // 필수 파라미터
  model: string;                    // 모델 ID
  messages: Message[];              // 메시지 배열

  // 응답 제어
  max_tokens?: number;              // 최대 생성 토큰 수
  n?: number;                       // 생성할 응답 개수 (기본: 1)
  stop?: string | string[];         // 중단 시퀀스
  stream?: boolean;                 // 스트리밍 모드 (기본: false)

  // 샘플링 파라미터
  temperature?: number;             // 0~2 (기본: 1)
  top_p?: number;                   // 0~1 (기본: 1)
  presence_penalty?: number;        // -2~2 (기본: 0)
  frequency_penalty?: number;       // -2~2 (기본: 0)
  logit_bias?: Record<string, number>; // 토큰별 바이어스

  // 재현성
  seed?: number;                    // 시드 값

  // 출력 형식
  response_format?: {
    type: "text" | "json_object" | "json_schema";
    json_schema?: object;
  };

  // 고급 기능
  tools?: Tool[];                   // 함수/도구 정의
  tool_choice?: "none" | "auto" | {type: "function", function: {name: string}};
  logprobs?: boolean;               // 로그 확률 반환
  top_logprobs?: number;            // 상위 N개 로그 확률

  // 메타데이터
  user?: string;                    // 사용자 식별자

  // 멀티모달 (GPT-4 Vision)
  // messages[].content에 이미지 포함 가능
}

interface Message {
  role: "system" | "user" | "assistant" | "tool";
  content: string | MessageContent[];
  name?: string;
  tool_calls?: ToolCall[];
  tool_call_id?: string;            // tool 롤 메시지용
}

interface MessageContent {
  type: "text" | "image_url";
  text?: string;
  image_url?: {
    url: string;
    detail?: "low" | "high" | "auto";
  };
}

interface Tool {
  type: "function";
  function: {
    name: string;
    description: string;
    parameters: JSONSchema;
  };
}
```

### 응답 (Response) 전체 스키마

```typescript
interface ChatCompletionResponse {
  id: string;                       // 요청 ID
  object: "chat.completion";
  created: number;                  // Unix timestamp
  model: string;                    // 실제 사용된 모델
  system_fingerprint?: string;      // 백엔드 설정 식별자

  choices: Choice[];
  usage: Usage;
}

interface Choice {
  index: number;
  message: Message;
  finish_reason: "stop" | "length" | "tool_calls" | "content_filter" | null;
  logprobs?: Logprobs;
}

interface Message {
  role: "assistant";
  content: string | null;
  tool_calls?: ToolCall[];
}

interface ToolCall {
  id: string;
  type: "function";
  function: {
    name: string;
    arguments: string;              // JSON 문자열
  };
}

interface Usage {
  prompt_tokens: number;
  completion_tokens: number;
  total_tokens: number;
}

interface Logprobs {
  content: TokenLogprob[];
}

interface TokenLogprob {
  token: string;
  logprob: number;
  bytes: number[];
  top_logprobs: {token: string, logprob: number}[];
}
```

## B. 에러 코드 레퍼런스

### HTTP 상태 코드

| 코드 | 에러 타입 | 의미 | 대응 방법 |
|------|----------|------|-----------|
| **400** | `invalid_request_error` | 잘못된 요청 | 요청 파라미터 확인 |
| **401** | `authentication_error` | 인증 실패 | API 키 확인 |
| **403** | `permission_error` | 권한 없음 | 조직/프로젝트 설정 확인 |
| **404** | `not_found_error` | 리소스 없음 | 모델 ID 확인 |
| **429** | `rate_limit_error` | 요청 제한 초과 | 재시도 (backoff) |
| **500** | `api_error` | 서버 에러 | 재시도 |
| **503** | `service_unavailable` | 서비스 불가 | 잠시 후 재시도 |

### 세부 에러 코드

```json
{
  "error": {
    "message": "에러 메시지",
    "type": "에러 타입",
    "param": "문제가 있는 파라미터",
    "code": "에러 코드"
  }
}
```

**주요 에러 코드**:

| code | 의미 |
|------|------|
| `invalid_api_key` | API 키가 유효하지 않음 |
| `insufficient_quota` | 크레딧 부족 |
| `model_not_found` | 모델을 찾을 수 없음 |
| `context_length_exceeded` | 컨텍스트 길이 초과 |
| `invalid_request_error` | 요청 형식 오류 |
| `content_filter` | 안전 필터에 의해 차단 |
| `rate_limit_exceeded` | Rate limit 초과 |

## C. 모델 레퍼런스

### OpenAI 모델

| 모델 ID | 컨텍스트 | 지식 컷오프 | 비용 (입력/출력) |
|---------|----------|------------|-----------------|
| `gpt-4-turbo` | 128K | 2023-12 | $10 / $30 |
| `gpt-4-turbo-preview` | 128K | 2023-12 | $10 / $30 |
| `gpt-4` | 8K | 2021-09 | $30 / $60 |
| `gpt-4-32k` | 32K | 2021-09 | $60 / $120 |
| `gpt-3.5-turbo` | 16K | 2021-09 | $0.50 / $1.50 |
| `gpt-3.5-turbo-16k` | 16K | 2021-09 | $3 / $4 |
| `gpt-4-vision-preview` | 128K | 2023-04 | $10 + 이미지 |

### Anthropic 모델

| 모델 ID | 컨텍스트 | 비용 |
|---------|----------|------|
| `claude-3-5-sonnet-20241022` | 200K | $3 / $15 |
| `claude-3-opus-20240229` | 200K | $15 / $75 |
| `claude-3-sonnet-20240229` | 200K | $3 / $15 |
| `claude-3-haiku-20240307` | 200K | $0.25 / $1.25 |

### Google 모델

| 모델 ID | 컨텍스트 | 비용 |
|---------|----------|------|
| `gemini-pro` | 32K | 무료 티어 / $0.50 |
| `gemini-pro-vision` | 16K | 무료 티어 / $0.50 |
| `gemini-ultra` | 32K | TBA |

## D. Rate Limits

### OpenAI Rate Limits (Tier별)

| Tier | RPM | TPM | 조건 |
|------|-----|-----|------|
| **Free** | 3 | 40K | 가입 직후 |
| **Tier 1** | 500 | 90K | $5+ 결제 |
| **Tier 2** | 5,000 | 450K | $50+ 결제, 7일+ |
| **Tier 3** | 10,000 | 1M | $1,000+ 결제, 30일+ |
| **Tier 4** | 30,000 | 5M | $5,000+ 결제, 30일+ |
| **Tier 5** | 80,000 | 10M | $50,000+ 결제, 30일+ |

**RPM**: Requests Per Minute
**TPM**: Tokens Per Minute

### Rate Limit 헤더

```
x-ratelimit-limit-requests: 500
x-ratelimit-limit-tokens: 90000
x-ratelimit-remaining-requests: 499
x-ratelimit-remaining-tokens: 89950
x-ratelimit-reset-requests: 12s
x-ratelimit-reset-tokens: 6m0s
```

## E. 토큰 계산 도구

### Python - tiktoken

```python
import tiktoken

# GPT-4용 인코더
enc = encoding_for_model("gpt-4")

# 텍스트 → 토큰
text = "Hello, world!"
tokens = enc.encode(text)
print(f"토큰 수: {len(tokens)}")  # 4
print(f"토큰: {tokens}")  # [9906, 11, 1917, 0]

# 토큰 → 텍스트
decoded = enc.decode(tokens)
print(decoded)  # "Hello, world!"

# 메시지 토큰 계산
def count_message_tokens(messages, model="gpt-4"):
    enc = encoding_for_model(model)
    tokens = 0

    for message in messages:
        tokens += 4  # 메시지 오버헤드
        tokens += len(enc.encode(message["content"]))
        if message.get("name"):
            tokens += len(enc.encode(message["name"])) - 1

    tokens += 2  # 응답 시작 토큰
    return tokens

messages = [
    {"role": "system", "content": "You are helpful"},
    {"role": "user", "content": "Hello"}
]

print(count_message_tokens(messages))  # 약 15-20 토큰
```

### JavaScript - tiktoken

```javascript
import {get_encoding} from "tiktoken";

const enc = get_encoding("cl100k_base");  // GPT-4

const text = "Hello, world!";
const tokens = enc.encode(text);

console.log(`토큰 수: ${tokens.length}`);
console.log(`토큰: ${tokens}`);

enc.free();  // 메모리 해제
```

### Online 도구

- **OpenAI Tokenizer**: https://platform.openai.com/tokenizer
- **Anthropic Token Counter**: https://docs.anthropic.com/claude/reference/tokens

## F. SDK 및 라이브러리

### 공식 SDK

| 플랫폼 | Python | JavaScript/TypeScript | Go | Ruby |
|--------|--------|---------------------|-----|------|
| **OpenAI** | ✅ `openai` | ✅ `openai` | ✅ | ✅ |
| **Anthropic** | ✅ `anthropic` | ✅ `@anthropic-ai/sdk` | ✅ | - |
| **Google** | ✅ `google-generativeai` | ✅ `@google/generative-ai` | ✅ | - |
| **Cohere** | ✅ `cohere` | ✅ `cohere-ai` | ✅ | ✅ |

### 통합 라이브러리

**LiteLLM** (권장)
```bash
pip install litellm
```

```python
from litellm import completion

# 모든 프로바이더를 동일한 인터페이스로
response = completion(
    model="gpt-4",  # 또는 "claude-3-opus", "gemini-pro"
    messages=[{"role": "user", "content": "Hello"}]
)
```

**LangChain**
```bash
pip install langchain langchain-openai
```

```python
from langchain_openai import ChatOpenAI
from langchain.schema import HumanMessage

llm = ChatOpenAI(model="gpt-4")
response = llm([HumanMessage(content="Hello")])
```

**LlamaIndex**
```bash
pip install llama-index
```

```python
from llama_index.llms import OpenAI

llm = OpenAI(model="gpt-4")
response = llm.complete("Hello")
```

## G. 유용한 도구

### 프롬프트 엔지니어링

- **PromptPerfect**: https://promptperfect.jina.ai/
- **Prompt Engineering Guide**: https://www.promptingguide.ai/
- **OpenAI Cookbook**: https://github.com/openai/openai-cookbook

### 모니터링 및 디버깅

- **Langfuse**: 오픈소스 LLM 옵저버빌리티
- **PromptLayer**: 프롬프트 버저닝 및 추적
- **Helicone**: LLM 모니터링 및 캐싱
- **LangSmith**: LangChain 공식 디버깅 도구

### 벡터 데이터베이스 (RAG용)

- **Pinecone**: 관리형 벡터 DB
- **Weaviate**: 오픈소스 벡터 DB
- **Chroma**: 임베딩 데이터베이스
- **Qdrant**: 고성능 벡터 검색

### UI 프레임워크

- **Streamlit**: 빠른 프로토타이핑
- **Gradio**: 데모 및 인터페이스
- **Chainlit**: ChatGPT 스타일 UI

## H. 환경 변수 설정

### OpenAI

```bash
# .env
OPENAI_API_KEY=sk-...
OPENAI_ORG_ID=org-...  # 선택적
OPENAI_PROJECT_ID=proj-...  # 선택적
```

```python
from dotenv import load_dotenv
load_dotenv()

# 자동으로 환경 변수에서 로드됨
client = OpenAI()
```

### Anthropic

```bash
# .env
ANTHROPIC_API_KEY=sk-ant-...
```

```python
import os
from anthropic import Anthropic

client = Anthropic(api_key=os.getenv("ANTHROPIC_API_KEY"))
```

### 여러 프로바이더

```bash
# .env
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
GOOGLE_API_KEY=...
COHERE_API_KEY=...
```

## I. 보안 베스트 프랙티스

### 1. API 키 보호

```python
# ❌ 절대 하지 말 것
api_key = "sk-abc123..."  # 하드코딩

# ✅ 환경 변수 사용
import os
api_key = os.getenv("OPENAI_API_KEY")

# ✅ 시크릿 관리자 사용 (프로덕션)
from google.cloud import secretmanager
# 또는 AWS Secrets Manager, Azure Key Vault 등
```

### 2. 사용자 입력 검증

```python
# ❌ 검증 없이 사용
user_input = request.form['message']
response = client.chat.completions.create(
    messages=[{"role": "user", "content": user_input}]
)

# ✅ 검증 및 제한
MAX_LENGTH = 10000

def validate_input(text):
    if len(text) > MAX_LENGTH:
        raise ValueError("Input too long")

    # XSS 방지
    import html
    text = html.escape(text)

    return text

user_input = validate_input(request.form['message'])
```

### 3. Rate Limiting (애플리케이션 레벨)

```python
from flask_limiter import Limiter

limiter = Limiter(
    app,
    key_func=lambda: request.remote_addr,
    default_limits=["100 per hour"]
)

@app.route("/chat")
@limiter.limit("10 per minute")
def chat():
    # LLM API 호출
    pass
```

### 4. 비용 제한

```python
# 사용자별 일일 토큰 제한
class UsageLimiter:
    def __init__(self, daily_token_limit=100000):
        self.limit = daily_token_limit
        self.usage = {}  # user_id -> tokens_used

    def check_and_update(self, user_id, tokens):
        today = datetime.now().date()
        key = f"{user_id}:{today}"

        current_usage = self.usage.get(key, 0)
        if current_usage + tokens > self.limit:
            raise Exception("Daily limit exceeded")

        self.usage[key] = current_usage + tokens

limiter = UsageLimiter()
```

## J. 성능 최적화 팁

### 1. 병렬 처리

```python
import asyncio
from openai import AsyncOpenAI

client = AsyncOpenAI()

async def process_batch(items):
    tasks = [
        client.chat.completions.create(
            model="gpt-3.5-turbo",
            messages=[{"role": "user", "content": item}]
        )
        for item in items
    ]
    return await asyncio.gather(*tasks)

items = ["Query 1", "Query 2", "Query 3"]
results = asyncio.run(process_batch(items))
```

### 2. 캐싱 전략

```python
from functools import lru_cache
import hashlib

@lru_cache(maxsize=1000)
def cached_completion(prompt_hash, model):
    # 실제 API 호출
    response = client.chat.completions.create(
        model=model,
        messages=[{"role": "user", "content": prompt_hash}],
        temperature=0  # 결정적 출력
    )
    return response.choices[0].message.content

def get_completion(prompt, model="gpt-4"):
    # 프롬프트 해시
    prompt_hash = hashlib.md5(prompt.encode()).hexdigest()
    return cached_completion(prompt_hash, model)
```

### 3. 프롬프트 압축

```python
# ❌ 비효율적
prompt = f"""
당신은 전문 번역가입니다.
다음 텍스트를 한국어로 번역해주세요.
번역할 때는 자연스럽게 번역하고,
문맥을 고려하여 의역하세요.
또한 전문 용어는 정확하게 번역하세요.

텍스트: {text}
"""

# ✅ 효율적
prompt = f"Translate to Korean (natural, contextual): {text}"
# → 토큰 절약!
```

## K. 문제 해결 (Troubleshooting)

### 문제: "Rate limit exceeded"

**원인**: 요청 빈도 제한 초과

**해결**:
```python
import time

def retry_with_backoff(func, max_retries=5):
    for i in range(max_retries):
        try:
            return func()
        except RateLimitError:
            wait = 2 ** i
            time.sleep(wait)
    raise Exception("Max retries exceeded")
```

### 문제: "Context length exceeded"

**원인**: 입력 + 출력 토큰이 모델 제한 초과

**해결**:
```python
# 1. 토큰 수 확인
tokens = count_tokens(messages)
if tokens > 7000:  # GPT-4: 8K 모델
    # 메시지 축약
    messages = messages[-5:]  # 최근 5개만

# 2. max_tokens 조정
max_output = 8192 - tokens - 100  # 여유분
response = client.chat.completions.create(
    model="gpt-4",
    messages=messages,
    max_tokens=min(max_output, 1000)
)
```

### 문제: "Invalid API key"

**원인**: API 키 오류

**해결**:
```python
# 1. 환경 변수 확인
import os
print(os.getenv("OPENAI_API_KEY"))

# 2. API 키 형식 확인
# OpenAI: sk-...
# Anthropic: sk-ant-...

# 3. 프로젝트 키 vs 시크릿 키 확인
```

## 마치며

이 ebook을 통해 LLM 인터페이스의 모든 측면을 깊이 있게 다뤘습니다. 실전에서 활용하며 더 많은 경험을 쌓으시기 바랍니다!

**피드백 및 기여**:
- GitHub Issues: [프로젝트 링크]
- 이메일: [연락처]

---

**라이센스**: MIT
**버전**: 1.0.0
**최종 업데이트**: 2024-11-14
