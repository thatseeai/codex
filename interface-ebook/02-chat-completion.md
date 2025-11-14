# 2장: Chat Completion 인터페이스

Chat Completion은 **현대적인 LLM 인터페이스의 표준**입니다. OpenAI GPT-4, Anthropic Claude, Google Gemini 등 대부분의 LLM이 이 형식을 채택하고 있습니다.

## 왜 Chat Completion인가?

기존 Text Completion과 비교:

```python
# 구식: Text Completion (단방향)
Prompt: "User: 안녕하세요\nAssistant:"
→ 모델이 이어서 생성

# 현대적: Chat Completion (구조화된 대화)
Messages: [
  {"role": "user", "content": "안녕하세요"}
]
→ 명확한 역할 구분과 대화 컨텍스트
```

**장점**:
- ✅ 명확한 역할 구분 (user, assistant, system)
- ✅ 대화 히스토리 관리 용이
- ✅ 멀티턴 대화 지원
- ✅ 시스템 지시사항 분리 가능
- ✅ 함수 호출 등 고급 기능 지원

## 입력 형식: Request Schema

### 기본 구조

```json
{
  "model": "gpt-4",
  "messages": [
    {"role": "system", "content": "당신은 도움이 되는 어시스턴트입니다."},
    {"role": "user", "content": "Python에서 리스트를 정렬하는 방법은?"}
  ],
  "temperature": 0.7,
  "max_tokens": 1000
}
```

### 필수 필드

| 필드 | 타입 | 설명 |
|------|------|------|
| `model` | string | 사용할 모델 ID (예: "gpt-4", "gpt-3.5-turbo") |
| `messages` | array | 대화 메시지 배열 |

### messages 배열 구조

각 메시지 객체:

```typescript
interface Message {
  role: "system" | "user" | "assistant" | "function" | "tool";
  content: string | null;
  name?: string;           // 선택적: 발화자 이름
  function_call?: object;  // 함수 호출 시
  tool_calls?: array;      // 도구 호출 시
}
```

#### Role별 설명

**1. system 롤**
```json
{
  "role": "system",
  "content": "당신은 전문 Python 개발자입니다. 코드 예제를 제공할 때는 항상 주석을 포함하세요."
}
```
- 모델의 동작 방식, 페르소나, 제약사항 지정
- 일반적으로 대화의 맨 처음에 위치
- **사용자에게 보이지 않는 경우가 많음** (채팅 UI에서 숨겨짐)

**2. user 롤**
```json
{
  "role": "user",
  "content": "리스트 정렬 예제를 보여주세요"
}
```
- 사용자의 실제 입력
- 항상 사용자에게 보임

**3. assistant 롤**
```json
{
  "role": "assistant",
  "content": "물론입니다! 다음은 리스트 정렬 예제입니다:\n\n```python\n# 오름차순 정렬\nnumbers = [3, 1, 4, 1, 5]\nnumbers.sort()\nprint(numbers)  # [1, 1, 3, 4, 5]\n```"
}
```
- AI 어시스턴트의 이전 응답
- 대화 히스토리를 구성할 때 사용

### 선택적 필드 (파라미터)

| 필드 | 타입 | 기본값 | 설명 |
|------|------|--------|------|
| `temperature` | float | 1.0 | 출력 무작위성 (0~2, 낮을수록 일관적) |
| `max_tokens` | int | inf | 생성할 최대 토큰 수 |
| `top_p` | float | 1.0 | Nucleus sampling (0~1) |
| `n` | int | 1 | 생성할 응답 개수 |
| `stream` | bool | false | 스트리밍 모드 활성화 |
| `stop` | string/array | null | 생성 중단 시퀀스 |
| `presence_penalty` | float | 0 | 새로운 주제 언급 장려 (-2~2) |
| `frequency_penalty` | float | 0 | 반복 억제 (-2~2) |
| `user` | string | - | 사용자 식별자 (모니터링용) |

## 출력 형식: Response Schema

### 기본 응답 구조

```json
{
  "id": "chatcmpl-8VXKz9f3JQ2Xr7ZG1",
  "object": "chat.completion",
  "created": 1699564800,
  "model": "gpt-4-0613",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "Python에서 리스트를 정렬하는 방법은 두 가지가 있습니다:\n\n1. `sort()` 메서드 (in-place 정렬)\n```python\nnumbers = [3, 1, 4, 1, 5]\nnumbers.sort()\nprint(numbers)  # [1, 1, 3, 4, 5]\n```\n\n2. `sorted()` 함수 (새 리스트 반환)\n```python\nnumbers = [3, 1, 4, 1, 5]\nsorted_numbers = sorted(numbers)\nprint(sorted_numbers)  # [1, 1, 3, 4, 5]\nprint(numbers)  # [3, 1, 4, 1, 5] - 원본 유지\n```"
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 28,
    "completion_tokens": 145,
    "total_tokens": 173
  },
  "system_fingerprint": "fp_eeff13170a"
}
```

### 응답 필드 설명

#### 최상위 필드

| 필드 | 설명 |
|------|------|
| `id` | 고유 요청 ID (디버깅, 로깅용) |
| `object` | 응답 타입 (`"chat.completion"`) |
| `created` | Unix 타임스탬프 |
| `model` | 실제 사용된 모델 (스냅샷 포함) |
| `choices` | 생성된 응답 배열 |
| `usage` | 토큰 사용량 통계 |
| `system_fingerprint` | 백엔드 설정 식별자 |

#### choices 배열

```typescript
interface Choice {
  index: number;
  message: Message;
  finish_reason: "stop" | "length" | "function_call" | "content_filter" | "null";
  logprobs?: object;  // logprobs=true일 때만
}
```

**finish_reason 값**:

| 값 | 의미 |
|---|------|
| `stop` | 정상 완료 (자연스러운 종료 또는 stop 시퀀스 도달) |
| `length` | max_tokens 제한에 도달하여 중단 |
| `function_call` | 함수 호출 요청으로 중단 |
| `content_filter` | 안전 필터에 의해 차단됨 |
| `null` | 스트리밍 중 (아직 완료되지 않음) |

#### usage 객체

```json
{
  "prompt_tokens": 28,      // 입력 토큰 수
  "completion_tokens": 145, // 출력 토큰 수
  "total_tokens": 173       // 총 토큰 수 (비용 계산 기준)
}
```

**비용 계산 예시** (GPT-4 기준):
```python
# GPT-4 가격 (2024년 기준)
# 입력: $0.03 / 1K tokens
# 출력: $0.06 / 1K tokens

prompt_cost = (28 / 1000) * 0.03      # $0.00084
completion_cost = (145 / 1000) * 0.06  # $0.0087
total_cost = prompt_cost + completion_cost  # $0.00954 (약 1센트)
```

## 실전 예제

### 예제 1: 기본 단일 질문

**요청**:
```python
import openai

response = openai.chat.completions.create(
    model="gpt-4",
    messages=[
        {"role": "user", "content": "대한민국의 수도는?"}
    ]
)

print(response.choices[0].message.content)
```

**응답**:
```json
{
  "id": "chatcmpl-xyz",
  "object": "chat.completion",
  "created": 1699564800,
  "model": "gpt-4-0613",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "대한민국의 수도는 서울입니다."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 10,
    "completion_tokens": 8,
    "total_tokens": 18
  }
}
```

**출력**:
```
대한민국의 수도는 서울입니다.
```

### 예제 2: 멀티턴 대화 (컨텍스트 유지)

**요청**:
```python
messages = [
    {"role": "system", "content": "당신은 친절한 수학 선생님입니다."},
    {"role": "user", "content": "2 + 2는 얼마인가요?"},
    {"role": "assistant", "content": "2 + 2는 4입니다."},
    {"role": "user", "content": "그럼 그 결과에 3을 곱하면?"}
]

response = openai.chat.completions.create(
    model="gpt-4",
    messages=messages
)

print(response.choices[0].message.content)
```

**응답**:
```
4에 3을 곱하면 12가 됩니다. 즉, 2 + 2 = 4이고, 4 × 3 = 12입니다.
```

**핵심**: 모델은 이전 대화 컨텍스트("4")를 기억하고 활용합니다.

### 예제 3: 시스템 프롬프트로 동작 제어

**요청**:
```python
# 전문가 모드
response_expert = openai.chat.completions.create(
    model="gpt-4",
    messages=[
        {"role": "system", "content": "당신은 컴퓨터 과학 박사입니다. 항상 기술 용어를 사용하고 깊이 있게 설명하세요."},
        {"role": "user", "content": "정렬 알고리즘을 설명해주세요"}
    ]
)

# 초보자 모드
response_beginner = openai.chat.completions.create(
    model="gpt-4",
    messages=[
        {"role": "system", "content": "당신은 초등학생을 가르치는 선생님입니다. 쉬운 말과 비유로 설명하세요."},
        {"role": "user", "content": "정렬 알고리즘을 설명해주세요"}
    ]
)
```

**전문가 모드 응답**:
```
정렬 알고리즘은 주어진 데이터 집합을 특정 순서(일반적으로 오름차순 또는 내림차순)로 재배열하는
알고리즘입니다. 시간 복잡도에 따라 O(n²) 알고리즘(버블 정렬, 삽입 정렬)과 O(n log n)
알고리즘(병합 정렬, 퀵 정렬, 힙 정렬)으로 분류됩니다. 실용적인 관점에서 퀵 정렬이 평균적으로
가장 빠르지만 최악의 경우 O(n²)이며, 병합 정렬은 안정적인 O(n log n)을 보장합니다...
```

**초보자 모드 응답**:
```
정렬은 마치 키 순서대로 줄을 서는 것과 같아요! 숫자나 글자들이 뒤죽박죽 섞여있을 때,
작은 것부터 큰 순서로 또는 큰 것부터 작은 순서로 차례차례 정리하는 거예요.

예를 들어 [5, 2, 8, 1, 9]라는 숫자들이 있다면, 정렬하면 [1, 2, 5, 8, 9]가 되는
거죠. 마치 친구들이 키 순서대로 줄을 서는 것처럼요!
```

### 예제 4: Temperature 조정

**요청**:
```python
# Low temperature (일관적, 결정적)
response_low = openai.chat.completions.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "고양이에 대한 짧은 시를 써주세요"}],
    temperature=0.2
)

# High temperature (창의적, 다양함)
response_high = openai.chat.completions.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "고양이에 대한 짧은 시를 써주세요"}],
    temperature=1.5
)
```

**Low Temperature (0.2) 응답**:
```
고양이

부드러운 털, 날렵한 몸
창가에 앉아 햇살을 받네
조용히 눈을 감고
평화로운 오후를 즐기네
```

**High Temperature (1.5) 응답**:
```
우주 고양이의 꿈

은하수 사이를 뛰어다니는 털뭉치
별똥별을 쫓다 달나라에 착륙
크레이터 위에서 그루밍하며
지구인들의 참치캔을 꿈꾸네
야옹~ 우주의 메아리
```

## 사용자에게 보이는 것 vs 숨겨진 것

### 사용자 관점 (ChatGPT UI 예시)

**보이는 것**:
```
[User]
Python으로 파일을 읽는 방법을 알려주세요

[Assistant]
Python에서 파일을 읽는 방법은 다음과 같습니다:

```python
# 방법 1: with 문 사용 (권장)
with open('file.txt', 'r', encoding='utf-8') as f:
    content = f.read()
    print(content)
```
```

**숨겨진 것** (실제 API 호출):
```json
{
  "model": "gpt-4",
  "messages": [
    {
      "role": "system",
      "content": "You are ChatGPT, a large language model trained by OpenAI, based on the GPT-4 architecture.\nKnowledge cutoff: 2023-04\nCurrent date: 2024-11-14\n\nImage input capabilities: Enabled\nPersonality: v2"
    },
    {
      "role": "user",
      "content": "Python으로 파일을 읽는 방법을 알려주세요"
    }
  ],
  "temperature": 1,
  "top_p": 1,
  "presence_penalty": 0,
  "frequency_penalty": 0,
  "max_tokens": 4096
}
```

**응답 메타데이터** (숨겨짐):
```json
{
  "id": "chatcmpl-AXK...",
  "created": 1731586234,
  "model": "gpt-4-0613",
  "usage": {
    "prompt_tokens": 87,
    "completion_tokens": 156,
    "total_tokens": 243
  },
  "system_fingerprint": "fp_abc123"
}
```

### 개발자 관점 (API 직접 사용)

**전체 가시성**:
```python
import openai
import json

response = openai.chat.completions.create(
    model="gpt-4",
    messages=[
        {"role": "user", "content": "Hello"}
    ]
)

# 전체 응답 객체 확인
print(json.dumps(response.model_dump(), indent=2))

# 개별 필드 접근
print(f"모델: {response.model}")
print(f"ID: {response.id}")
print(f"토큰 사용: {response.usage.total_tokens}")
print(f"내용: {response.choices[0].message.content}")
print(f"종료 이유: {response.choices[0].finish_reason}")
```

## 에러 처리

### 일반적인 에러 응답

**1. 인증 실패 (401)**:
```json
{
  "error": {
    "message": "Incorrect API key provided",
    "type": "invalid_request_error",
    "param": null,
    "code": "invalid_api_key"
  }
}
```

**2. 요청 제한 초과 (429)**:
```json
{
  "error": {
    "message": "Rate limit reached for requests",
    "type": "rate_limit_error",
    "param": null,
    "code": "rate_limit_exceeded"
  }
}
```

**3. 토큰 제한 초과 (400)**:
```json
{
  "error": {
    "message": "This model's maximum context length is 8192 tokens. However, your messages resulted in 10543 tokens.",
    "type": "invalid_request_error",
    "param": "messages",
    "code": "context_length_exceeded"
  }
}
```

**4. 컨텐츠 필터링 (400)**:
```json
{
  "error": {
    "message": "Your request was rejected as a result of our safety system.",
    "type": "invalid_request_error",
    "param": null,
    "code": "content_filter"
  }
}
```

### 에러 처리 패턴

```python
from openai import OpenAI, OpenAIError, RateLimitError, APIError

client = OpenAI()

def safe_chat_completion(messages, max_retries=3):
    for attempt in range(max_retries):
        try:
            response = client.chat.completions.create(
                model="gpt-4",
                messages=messages,
                timeout=30.0
            )
            return response

        except RateLimitError:
            wait_time = 2 ** attempt  # Exponential backoff
            print(f"Rate limit hit. Waiting {wait_time}s...")
            time.sleep(wait_time)

        except APIError as e:
            if e.status_code >= 500:  # Server error
                print(f"Server error: {e}. Retrying...")
                time.sleep(2)
            else:
                raise  # Client error, don't retry

        except OpenAIError as e:
            print(f"OpenAI error: {e}")
            raise

    raise Exception("Max retries exceeded")

# 사용
try:
    response = safe_chat_completion([
        {"role": "user", "content": "Hello"}
    ])
    print(response.choices[0].message.content)
except Exception as e:
    print(f"Failed: {e}")
```

## 다음 단계

Chat Completion의 기본을 마스터했습니다! 다음 장에서는 레거시 **Text Completion 인터페이스**를 살펴보고 Chat Completion과의 차이점을 비교해봅시다.

➡️ [3장: Text Completion 인터페이스](./03-text-completion.md)

---

## 핵심 요약

- ✅ Chat Completion은 현대 LLM의 표준 인터페이스
- ✅ `messages` 배열로 구조화된 대화 관리
- ✅ `role` (system/user/assistant)로 명확한 역할 구분
- ✅ 응답의 `usage` 필드로 토큰 사용량 추적
- ✅ `finish_reason`으로 생성 종료 원인 파악
- ✅ 시스템 프롬프트는 사용자에게 숨겨지는 경우가 많음
- ✅ 에러 처리 및 재시도 로직 필수
