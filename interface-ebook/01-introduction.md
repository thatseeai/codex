# 1장: 소개 및 개요

## LLM 인터페이스란?

Large Language Model (LLM) 인터페이스는 인간과 AI 모델 간의 소통 방식을 정의하는 **프로토콜**입니다. 웹 개발에서 REST API나 GraphQL이 클라이언트와 서버 간 통신 방식을 정의하듯이, LLM 인터페이스는 사용자(또는 애플리케이션)와 AI 모델 간의 입출력 형식을 규정합니다.

### 왜 인터페이스가 중요한가?

```
사용자 입력 → [인터페이스] → LLM 모델 → [인터페이스] → 사용자 출력
              ↑ 보이는 부분                    ↑ 보이는 부분
              ↓ 숨겨진 부분                    ↓ 숨겨진 부분
         시스템 프롬프트                     메타데이터
         전처리 로직                         후처리 로직
         토큰화                              토큰 사용량
```

인터페이스를 이해하면:
- ✅ 더 효과적인 프롬프트 작성 가능
- ✅ 비용 최적화 (토큰 사용량 제어)
- ✅ 에러 디버깅 및 문제 해결
- ✅ 고급 기능 활용 (함수 호출, 멀티모달 등)
- ✅ 플랫폼 간 마이그레이션 용이

## 주요 개념과 용어

### 1. 토큰 (Token)
LLM이 텍스트를 처리하는 기본 단위입니다.

```python
# 예시: "Hello, world!"는 약 3-4개의 토큰
# - "Hello"
# - ","
# - " world"
# - "!"

# 한글의 경우 토큰 수가 더 많을 수 있음
# "안녕하세요" → 약 5-8개 토큰 (모델에 따라 다름)
```

**토큰의 중요성**:
- 비용 계산 단위 (입력 토큰 + 출력 토큰)
- 컨텍스트 윈도우 제한 (예: GPT-4: 128K 토큰)
- 처리 속도에 영향

### 2. 프롬프트 (Prompt)
모델에게 보내는 입력 텍스트입니다.

```
[User Prompt] → "Python으로 피보나치 수열을 구현해줘"
[System Prompt] → "당신은 친절한 코딩 어시스턴트입니다." (숨겨져 있음)
```

### 3. 컴플리션 (Completion)
모델이 생성한 출력 텍스트입니다.

```python
{
  "choices": [
    {
      "message": {
        "role": "assistant",
        "content": "물론이죠! 피보나치 수열 구현 코드입니다:\n\ndef fibonacci(n):\n    ..."
      },
      "finish_reason": "stop"
    }
  ]
}
```

### 4. 롤 (Role)
대화에서 각 메시지의 발화자를 나타냅니다.

| Role | 설명 | 예시 |
|------|------|------|
| `system` | 모델의 동작 방식 지정 (숨겨진 지시사항) | "당신은 전문 번역가입니다." |
| `user` | 사용자의 입력 | "이 문장을 영어로 번역해줘" |
| `assistant` | 모델의 응답 | "Sure, here's the translation..." |
| `function`/`tool` | 함수/도구 실행 결과 | `{"result": "계산 완료"}` |

### 5. 컨텍스트 윈도우 (Context Window)
모델이 한 번에 처리할 수 있는 최대 토큰 수입니다.

```
┌─────────────────────────────────────────┐
│     Context Window (예: 128K 토큰)      │
├─────────────────────────────────────────┤
│ [System Prompt]          (500 토큰)     │
│ [이전 대화 기록]         (10,000 토큰)  │
│ [현재 사용자 입력]       (200 토큰)     │
│ [모델 응답 공간]         (최대 4,096)   │
└─────────────────────────────────────────┘
```

## 인터페이스 유형 분류

### 1. 입력 방식에 따른 분류

#### A. 텍스트 전용 인터페이스
```json
// Text Completion (레거시)
{
  "prompt": "Once upon a time",
  "max_tokens": 100
}

// Chat Completion (현대적)
{
  "messages": [
    {"role": "user", "content": "Tell me a story"}
  ]
}
```

#### B. 멀티모달 인터페이스
```json
// Vision (이미지 + 텍스트)
{
  "messages": [
    {
      "role": "user",
      "content": [
        {"type": "text", "text": "이 이미지에 뭐가 있나요?"},
        {"type": "image_url", "image_url": {"url": "data:image/jpeg;base64,..."}}
      ]
    }
  ]
}

// Audio (음성)
{
  "input": "audio_base64_data",
  "voice": "alloy"
}
```

### 2. 출력 방식에 따른 분류

#### A. 일괄 응답 (Batch)
```python
# 전체 응답이 한 번에 반환됨
response = client.chat.completions.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "안녕하세요"}]
)
print(response.choices[0].message.content)
# 출력: "안녕하세요! 무엇을 도와드릴까요?"
```

#### B. 스트리밍 응답 (Streaming)
```python
# 응답이 실시간으로 조각나서 전달됨
stream = client.chat.completions.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "안녕하세요"}],
    stream=True
)
for chunk in stream:
    print(chunk.choices[0].delta.content, end="")
# 출력: "안" "녕" "하" "세" "요" "!" " " "무" "엇" ...
```

### 3. 기능에 따른 분류

#### A. 기본 대화
- 단순 질의응답
- 텍스트 생성

#### B. 함수 호출 (Function Calling)
```json
{
  "messages": [...],
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "get_weather",
        "description": "현재 날씨 조회",
        "parameters": {
          "type": "object",
          "properties": {
            "location": {"type": "string"}
          }
        }
      }
    }
  ]
}
```

#### C. 구조화된 출력
```json
{
  "messages": [...],
  "response_format": {
    "type": "json_object"
  }
}
```

## 사용자에게 보이는 것 vs 숨겨진 것

### 사용자에게 보이는 것 (Visible)

```json
// 요청
{
  "model": "gpt-4",
  "messages": [
    {"role": "user", "content": "2+2는?"}
  ]
}

// 응답
{
  "choices": [
    {
      "message": {
        "role": "assistant",
        "content": "2+2는 4입니다."
      }
    }
  ]
}
```

### 사용자에게 숨겨진 것 (Hidden)

```json
// 실제로 모델에게 전달되는 전체 컨텍스트
{
  "system_prompt": "You are a helpful assistant. Always be polite and accurate.",
  "safety_filters": {
    "hate_speech": true,
    "violence": true,
    "sexual_content": true
  },
  "preprocessing": {
    "tokenization": "cl100k_base",
    "normalization": true
  },
  "user_messages": [
    {"role": "user", "content": "2+2는?"}
  ],
  "model_config": {
    "temperature": 0.7,
    "top_p": 0.9,
    "frequency_penalty": 0.0
  }
}

// 응답과 함께 오는 메타데이터
{
  "id": "chatcmpl-abc123",
  "object": "chat.completion",
  "created": 1699564800,
  "model": "gpt-4-0613",
  "usage": {
    "prompt_tokens": 12,
    "completion_tokens": 8,
    "total_tokens": 20
  },
  "system_fingerprint": "fp_xxx",
  // ... 실제 응답 내용
}
```

## 인터페이스 계층 구조

```
┌──────────────────────────────────────────────────┐
│          사용자 애플리케이션 레이어               │
│  (ChatGPT UI, Custom App, API Integration)      │
└────────────────┬─────────────────────────────────┘
                 │
┌────────────────▼─────────────────────────────────┐
│          API 추상화 레이어 (SDK)                  │
│  (openai-python, anthropic-sdk, langchain)       │
└────────────────┬─────────────────────────────────┘
                 │
┌────────────────▼─────────────────────────────────┐
│          HTTP/REST API 레이어                     │
│  (JSON over HTTPS, Authentication, Rate Limits)  │
└────────────────┬─────────────────────────────────┘
                 │
┌────────────────▼─────────────────────────────────┐
│          플랫폼 레이어                            │
│  (Routing, Load Balancing, Safety Filters)       │
└────────────────┬─────────────────────────────────┘
                 │
┌────────────────▼─────────────────────────────────┐
│          모델 추론 레이어                         │
│  (Actual LLM Model - GPT-4, Claude, Gemini)     │
└──────────────────────────────────────────────────┘
```

## 다음 단계

다음 장에서는 가장 일반적으로 사용되는 **Chat Completion 인터페이스**를 자세히 살펴보겠습니다. 입력 형식, 출력 형식, 그리고 실전 예제를 통해 완벽하게 이해해봅시다.

➡️ [2장: Chat Completion 인터페이스](./02-chat-completion.md)

---

## 핵심 요약

- ✅ LLM 인터페이스는 입출력 프로토콜을 정의
- ✅ 토큰, 프롬프트, 롤, 컨텍스트 윈도우 등 핵심 개념 이해 필수
- ✅ 텍스트/멀티모달, 일괄/스트리밍, 기본/고급 기능으로 분류
- ✅ 보이는 부분과 숨겨진 부분(시스템 프롬프트, 메타데이터) 모두 중요
- ✅ 여러 계층의 추상화가 존재함을 인지
