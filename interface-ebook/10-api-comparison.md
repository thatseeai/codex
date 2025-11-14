# 10장: API 비교 및 마이그레이션

주요 LLM 플랫폼들의 API를 비교하고, 플랫폼 간 마이그레이션 전략을 다룹니다.

## 주요 LLM API 개요

| 플랫폼 | 주요 모델 | 강점 | 약점 |
|--------|-----------|------|------|
| **OpenAI** | GPT-4, GPT-3.5 | 범용성, 생태계 | 높은 비용 |
| **Anthropic** | Claude 3 | 긴 컨텍스트, 안전성 | 제한적 생태계 |
| **Google** | Gemini Pro | 멀티모달, 무료 티어 | 새로운 플랫폼 |
| **Cohere** | Command R+ | 기업용 특화 | 인지도 낮음 |
| **Ollama** | Llama, Mistral | 로컬 실행, 무료 | 성능 제한 |

## API 구조 비교

### Chat Completion 요청

#### OpenAI
```python
from openai import OpenAI
client = OpenAI()

response = client.chat.completions.create(
    model="gpt-4",
    messages=[
        {"role": "system", "content": "You are helpful"},
        {"role": "user", "content": "Hello"}
    ],
    temperature=0.7,
    max_tokens=100
)

print(response.choices[0].message.content)
```

#### Anthropic
```python
import anthropic
client = anthropic.Anthropic()

message = client.messages.create(
    model="claude-3-5-sonnet-20241022",
    max_tokens=1024,  # 필수!
    system="You are helpful",  # 별도 파라미터
    messages=[
        {"role": "user", "content": "Hello"}
    ]
)

print(message.content[0].text)
```

#### Google Gemini
```python
import google.generativeai as genai
genai.configure(api_key="...")

model = genai.GenerativeModel('gemini-pro')
response = model.generate_content("Hello")

print(response.text)
```

#### Cohere
```python
import cohere
co = cohere.Client('...')

response = co.chat(
    message="Hello",
    model="command-r-plus",
    temperature=0.7
)

print(response.text)
```

### 주요 차이점

| 특성 | OpenAI | Anthropic | Google | Cohere |
|------|--------|-----------|--------|--------|
| **시스템 메시지** | messages 배열 내 | system 파라미터 | 별도 설정 | preamble 파라미터 |
| **max_tokens** | 선택적 | **필수** | 선택적 | max_tokens |
| **응답 경로** | choices[0].message.content | content[0].text | text | text |
| **스트리밍** | stream=True | stream=True | stream=True | stream=True |

## Function Calling 비교

### OpenAI Function Calling

```python
tools = [{
    "type": "function",
    "function": {
        "name": "get_weather",
        "description": "Get weather",
        "parameters": {
            "type": "object",
            "properties": {
                "location": {"type": "string"}
            },
            "required": ["location"]
        }
    }
}]

response = client.chat.completions.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "Weather in Seoul?"}],
    tools=tools
)

if response.choices[0].message.tool_calls:
    tool_call = response.choices[0].message.tool_calls[0]
    function_name = tool_call.function.name
    arguments = tool_call.function.arguments
```

### Anthropic Tool Use

```python
tools = [{
    "name": "get_weather",
    "description": "Get weather",
    "input_schema": {
        "type": "object",
        "properties": {
            "location": {"type": "string"}
        },
        "required": ["location"]
    }
}]

message = client.messages.create(
    model="claude-3-5-sonnet-20241022",
    max_tokens=1024,
    tools=tools,
    messages=[{"role": "user", "content": "Weather in Seoul?"}]
)

if message.stop_reason == "tool_use":
    tool_use = next(c for c in message.content if c.type == "tool_use")
    function_name = tool_use.name
    arguments = tool_use.input
```

### Cohere Tools

```python
tools = [{
    "name": "get_weather",
    "description": "Get weather",
    "parameter_definitions": {
        "location": {
            "description": "City name",
            "type": "str",
            "required": True
        }
    }
}]

response = co.chat(
    message="Weather in Seoul?",
    tools=tools
)

if response.tool_calls:
    tool_call = response.tool_calls[0]
    function_name = tool_call.name
    arguments = tool_call.parameters
```

## Vision 비교

### OpenAI Vision

```python
response = client.chat.completions.create(
    model="gpt-4-vision-preview",
    messages=[{
        "role": "user",
        "content": [
            {"type": "text", "text": "What's in this image?"},
            {"type": "image_url", "image_url": {"url": "..."}}
        ]
    }]
)
```

### Anthropic Vision

```python
import base64

with open("image.jpg", "rb") as f:
    image_data = base64.b64encode(f.read()).decode()

message = client.messages.create(
    model="claude-3-5-sonnet-20241022",
    max_tokens=1024,
    messages=[{
        "role": "user",
        "content": [
            {"type": "text", "text": "What's in this image?"},
            {
                "type": "image",
                "source": {
                    "type": "base64",
                    "media_type": "image/jpeg",
                    "data": image_data
                }
            }
        ]
    }]
)
```

### Google Gemini Vision

```python
import PIL.Image

model = genai.GenerativeModel('gemini-pro-vision')
img = PIL.Image.open('image.jpg')

response = model.generate_content([
    "What's in this image?",
    img
])

print(response.text)
```

## 가격 비교 (2024년 기준)

### 텍스트 모델

| 모델 | 입력 (per 1M tokens) | 출력 (per 1M tokens) |
|------|---------------------|---------------------|
| **GPT-4 Turbo** | $10 | $30 |
| **GPT-3.5 Turbo** | $0.50 | $1.50 |
| **Claude 3.5 Sonnet** | $3 | $15 |
| **Claude 3 Haiku** | $0.25 | $1.25 |
| **Gemini Pro** | 무료 (제한) / $0.50 | 무료 (제한) / $1.50 |
| **Command R+** | $3 | $15 |

### Vision 모델

| 모델 | 가격 |
|------|------|
| **GPT-4 Vision** | $10 입력 + 이미지당 $0.00765 (high detail) |
| **Claude 3.5 Sonnet (Vision)** | $3 입력 + 이미지당 ~$0.0048 |
| **Gemini Pro Vision** | 무료 (제한) / 유료 플랜 |

### 비용 예시 (10,000 요청)

**시나리오**: 평균 입력 500 토큰, 출력 150 토큰

```python
# GPT-3.5 Turbo
input_cost = (500 * 10000 / 1_000_000) * 0.50   # $2.50
output_cost = (150 * 10000 / 1_000_000) * 1.50  # $2.25
total = input_cost + output_cost  # $4.75

# Claude 3 Haiku
input_cost = (500 * 10000 / 1_000_000) * 0.25   # $1.25
output_cost = (150 * 10000 / 1_000_000) * 1.25  # $1.875
total = input_cost + output_cost  # $3.125

# → Claude 3 Haiku가 34% 저렴
```

## 마이그레이션 전략

### 패턴 1: 추상화 레이어

```python
from abc import ABC, abstractmethod

class LLMProvider(ABC):
    @abstractmethod
    def chat(self, messages, **kwargs):
        pass

class OpenAIProvider(LLMProvider):
    def __init__(self, api_key):
        self.client = OpenAI(api_key=api_key)

    def chat(self, messages, **kwargs):
        response = self.client.chat.completions.create(
            model=kwargs.get("model", "gpt-4"),
            messages=messages,
            temperature=kwargs.get("temperature", 0.7)
        )
        return response.choices[0].message.content

class AnthropicProvider(LLMProvider):
    def __init__(self, api_key):
        self.client = anthropic.Anthropic(api_key=api_key)

    def chat(self, messages, **kwargs):
        # 시스템 메시지 분리
        system = next((m["content"] for m in messages if m["role"] == "system"), None)
        user_messages = [m for m in messages if m["role"] != "system"]

        message = self.client.messages.create(
            model=kwargs.get("model", "claude-3-5-sonnet-20241022"),
            max_tokens=kwargs.get("max_tokens", 1024),
            system=system,
            messages=user_messages
        )
        return message.content[0].text

# 사용
provider = OpenAIProvider(api_key="...")  # 또는 AnthropicProvider

response = provider.chat(
    messages=[
        {"role": "system", "content": "You are helpful"},
        {"role": "user", "content": "Hello"}
    ],
    temperature=0.7
)
```

### 패턴 2: LiteLLM (통합 라이브러리)

```python
from litellm import completion

# OpenAI
response = completion(
    model="gpt-4",
    messages=[{"role": "user", "content": "Hello"}]
)

# Anthropic (동일한 인터페이스!)
response = completion(
    model="claude-3-5-sonnet-20241022",
    messages=[{"role": "user", "content": "Hello"}]
)

# Gemini (동일한 인터페이스!)
response = completion(
    model="gemini-pro",
    messages=[{"role": "user", "content": "Hello"}]
)

# 모든 응답이 동일한 형식
print(response.choices[0].message.content)
```

### 패턴 3: 멀티-프로바이더 Fallback

```python
class MultiProviderLLM:
    def __init__(self, providers):
        self.providers = providers

    def chat(self, messages, **kwargs):
        last_error = None

        for provider in self.providers:
            try:
                return provider.chat(messages, **kwargs)
            except Exception as e:
                print(f"{provider.__class__.__name__} failed: {e}")
                last_error = e
                continue

        raise last_error or Exception("All providers failed")

# 사용
multi_llm = MultiProviderLLM([
    OpenAIProvider(api_key="..."),
    AnthropicProvider(api_key="..."),
    GeminiProvider(api_key="...")
])

response = multi_llm.chat(messages=[...])
# → OpenAI 실패 시 자동으로 Anthropic 시도
```

## 로컬 모델 (Ollama)

### 설치 및 사용

```bash
# Ollama 설치
curl -fsSL https://ollama.com/install.sh | sh

# 모델 다운로드
ollama pull llama3
ollama pull mistral
```

```python
import ollama

# Chat
response = ollama.chat(
    model='llama3',
    messages=[
        {'role': 'user', 'content': 'Hello'}
    ]
)

print(response['message']['content'])
```

### OpenAI 호환 API

```python
from openai import OpenAI

# Ollama의 OpenAI 호환 엔드포인트 사용
client = OpenAI(
    base_url='http://localhost:11434/v1',
    api_key='ollama'  # 더미 키
)

response = client.chat.completions.create(
    model="llama3",
    messages=[{"role": "user", "content": "Hello"}]
)

print(response.choices[0].message.content)
```

### 장단점

**장점**:
- ✅ 완전 무료
- ✅ 데이터 프라이버시 (로컬 실행)
- ✅ 오프라인 사용 가능
- ✅ 무제한 요청

**단점**:
- ❌ 성능 제한 (하드웨어 의존)
- ❌ 클라우드 모델 대비 낮은 품질
- ❌ GPU 필요 (빠른 추론)
- ❌ 모델 업데이트 수동 관리

## 마이그레이션 체크리스트

### OpenAI → Anthropic

- [ ] `messages` 배열에서 `system` 메시지 분리
- [ ] `max_tokens` 필수 파라미터 추가
- [ ] 응답 파싱: `choices[0].message.content` → `content[0].text`
- [ ] `finish_reason` → `stop_reason`
- [ ] Tool calling: `tool_calls` → `content` 배열 내 `tool_use` 타입
- [ ] 함수 스키마: `parameters` → `input_schema`

### OpenAI → Google Gemini

- [ ] SDK 변경: `openai` → `google.generativeai`
- [ ] API 구조 변경: `chat.completions.create` → `GenerativeModel.generate_content`
- [ ] 메시지 형식 단순화 (Gemini는 더 간단)
- [ ] 응답 파싱: `choices[0].message.content` → `text`

### 클라우드 → 로컬 (Ollama)

- [ ] 모델 다운로드 및 설치
- [ ] GPU 메모리 요구사항 확인
- [ ] API 엔드포인트 변경: `https://api.openai.com` → `http://localhost:11434`
- [ ] 성능 테스트 및 벤치마크
- [ ] 품질 허용 범위 검증

## 다음 단계

API 비교를 마쳤습니다! 마지막으로 **부록**에서 전체 레퍼런스와 유용한 리소스를 확인하세요.

➡️ [11장: 부록 - 전체 레퍼런스](./11-appendix.md)

---

## 핵심 요약

- ✅ OpenAI: 가장 범용적, 강력한 생태계
- ✅ Anthropic: 긴 컨텍스트, 안전성 강조
- ✅ Google Gemini: 멀티모달 강점, 무료 티어
- ✅ Ollama: 로컬 실행, 프라이버시
- ✅ 추상화 레이어로 플랫폼 독립성 확보
- ✅ LiteLLM 등 통합 라이브러리 활용
- ✅ 비용, 성능, 프라이버시 트레이드오프 고려
