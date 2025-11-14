# 3장: Text Completion 인터페이스

Text Completion은 **LLM의 가장 원시적인 인터페이스**입니다. GPT-3 시대에 주로 사용되었으며, 현재는 대부분 Chat Completion으로 대체되었지만, 그 원리를 이해하는 것은 여전히 중요합니다.

## Text Completion이란?

**기본 개념**: 주어진 텍스트를 이어서 생성하는 단순한 방식

```
입력 (Prompt): "Once upon a time"
출력 (Completion): "there was a beautiful princess who lived in a castle..."
```

모델은 단순히 **"다음에 올 가능성이 높은 텍스트"**를 예측하여 생성합니다.

## Chat vs Text Completion 비교

### 구조적 차이

```python
# Text Completion (구식)
{
  "prompt": "User: 안녕하세요\nAssistant:",
  "max_tokens": 100
}

# Chat Completion (현대식)
{
  "messages": [
    {"role": "user", "content": "안녕하세요"}
  ],
  "max_tokens": 100
}
```

### 핵심 차이점

| 특성 | Text Completion | Chat Completion |
|------|----------------|-----------------|
| **구조** | 단일 문자열 | 구조화된 메시지 배열 |
| **역할 구분** | 수동 (문자열에 포함) | 자동 (role 필드) |
| **대화 관리** | 수동 형식 지정 필요 | 자동 관리 |
| **시스템 지시** | 프롬프트에 포함 | 별도 system 메시지 |
| **함수 호출** | 지원 안 함 | 네이티브 지원 |
| **사용 사례** | 단순 텍스트 생성, 코드 자동완성 | 대화형 AI, 복잡한 상호작용 |
| **현재 상태** | 레거시 (deprecated) | 표준 |

## 입력 형식: Request Schema

### 기본 구조

```json
{
  "model": "gpt-3.5-turbo-instruct",
  "prompt": "Write a haiku about programming:\n\n",
  "max_tokens": 50,
  "temperature": 0.7
}
```

### 필수 필드

| 필드 | 타입 | 설명 |
|------|------|------|
| `model` | string | 모델 ID (예: "gpt-3.5-turbo-instruct") |
| `prompt` | string or array | 생성할 텍스트의 시작점 |

**참고**: GPT-4는 Text Completion을 지원하지 않습니다. `gpt-3.5-turbo-instruct` 또는 이전 모델만 사용 가능합니다.

### 선택적 필드

| 필드 | 타입 | 기본값 | 설명 |
|------|------|--------|------|
| `max_tokens` | int | 16 | 생성할 최대 토큰 수 |
| `temperature` | float | 1.0 | 무작위성 (0~2) |
| `top_p` | float | 1.0 | Nucleus sampling |
| `n` | int | 1 | 생성할 완성본 개수 |
| `stream` | bool | false | 스트리밍 모드 |
| `stop` | string/array | null | 중단 시퀀스 |
| `presence_penalty` | float | 0 | 새 토픽 장려 |
| `frequency_penalty` | float | 0 | 반복 억제 |
| `logprobs` | int | null | 로그 확률 반환 개수 (0~5) |
| `echo` | bool | false | 프롬프트 에코 여부 |
| `suffix` | string | null | 삽입 모드 접미사 |

### 고급: 배열 형식 프롬프트

```json
{
  "model": "gpt-3.5-turbo-instruct",
  "prompt": [
    "1 + 1 = 2\n2 + 2 = 4\n3 + 3 = ",
    "The capital of France is Paris.\nThe capital of Germany is Berlin.\nThe capital of Italy is "
  ],
  "max_tokens": 10
}
```

배치 처리로 여러 프롬프트를 한 번에 요청할 수 있습니다.

## 출력 형식: Response Schema

### 기본 응답 구조

```json
{
  "id": "cmpl-8VXKz9f3JQ2Xr7ZG1",
  "object": "text_completion",
  "created": 1699564800,
  "model": "gpt-3.5-turbo-instruct",
  "choices": [
    {
      "text": "Code flows like streams,\nBugs hide in silent forests,\nDebug, find your peace.",
      "index": 0,
      "logprobs": null,
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 8,
    "completion_tokens": 20,
    "total_tokens": 28
  }
}
```

### Chat Completion과의 차이

| 필드 | Text Completion | Chat Completion |
|------|----------------|-----------------|
| 응답 내용 | `choices[].text` (문자열) | `choices[].message` (객체) |
| 역할 정보 | 없음 | `message.role` |
| 객체 타입 | `"text_completion"` | `"chat.completion"` |

## 실전 예제

### 예제 1: 간단한 텍스트 생성

**요청**:
```python
import openai

response = openai.completions.create(
    model="gpt-3.5-turbo-instruct",
    prompt="Write three tips for learning Python:\n\n1.",
    max_tokens=150,
    temperature=0.7
)

print(response.choices[0].text)
```

**응답**:
```
1. Practice regularly: The more you practice, the better you'll become at Python programming.
2. Read documentation: Understanding the official Python documentation will help you learn best practices.
3. Build projects: Apply what you learn by creating real-world projects.
```

### 예제 2: Stop Sequences 사용

**요청**:
```python
response = openai.completions.create(
    model="gpt-3.5-turbo-instruct",
    prompt="Q: What is the capital of France?\nA:",
    max_tokens=50,
    stop=["\n", "Q:"]  # 줄바꿈이나 다음 질문이 나오면 중단
)

print(response.choices[0].text)
```

**응답**:
```
 Paris
```

**finish_reason**: `"stop"` (stop sequence에 의해 중단됨)

### 예제 3: 코드 자동완성

**요청**:
```python
prompt = """def fibonacci(n):
    \"\"\"Generate fibonacci sequence up to n terms\"\"\"
    """

response = openai.completions.create(
    model="gpt-3.5-turbo-instruct",
    prompt=prompt,
    max_tokens=200,
    temperature=0.2,  # Low temperature for consistent code
    stop=["def ", "\n\n\n"]  # Stop at next function or triple newline
)

print(prompt + response.choices[0].text)
```

**응답**:
```python
def fibonacci(n):
    """Generate fibonacci sequence up to n terms"""
    if n <= 0:
        return []
    elif n == 1:
        return [0]
    elif n == 2:
        return [0, 1]

    fib = [0, 1]
    for i in range(2, n):
        fib.append(fib[i-1] + fib[i-2])
    return fib
```

### 예제 4: Multiple Completions (n > 1)

**요청**:
```python
response = openai.completions.create(
    model="gpt-3.5-turbo-instruct",
    prompt="Write a creative name for a coffee shop:",
    max_tokens=10,
    n=5,  # 5개의 다른 완성본 생성
    temperature=1.2
)

for i, choice in enumerate(response.choices, 1):
    print(f"{i}. {choice.text.strip()}")
```

**응답**:
```
1. The Cozy Bean
2. Brew Haven
3. Java Dreams Café
4. Percolate & Pause
5. The Roasted Muse
```

### 예제 5: Logprobs 사용 (확률 정보 포함)

**요청**:
```python
response = openai.completions.create(
    model="gpt-3.5-turbo-instruct",
    prompt="The sky is",
    max_tokens=1,
    logprobs=3  # 상위 3개 토큰의 로그 확률 반환
)

print(response.choices[0].logprobs)
```

**응답**:
```json
{
  "tokens": [" blue"],
  "token_logprobs": [-0.0023],
  "top_logprobs": [
    {
      " blue": -0.0023,
      " clear": -6.1234,
      " gray": -7.8901
    }
  ],
  "text_offset": [11]
}
```

**해석**:
- " blue"가 가장 높은 확률 (log prob: -0.0023 ≈ 99.77% 확률)
- " clear"는 두 번째로 높음
- " gray"는 세 번째

## Chat Completion으로 변환하기

Text Completion을 Chat Completion으로 마이그레이션하는 방법:

### 변환 패턴 1: 단순 프롬프트

**Before (Text Completion)**:
```python
response = openai.completions.create(
    model="gpt-3.5-turbo-instruct",
    prompt="Translate to French: Hello, how are you?",
    max_tokens=50
)
```

**After (Chat Completion)**:
```python
response = openai.chat.completions.create(
    model="gpt-3.5-turbo",
    messages=[
        {"role": "user", "content": "Translate to French: Hello, how are you?"}
    ],
    max_tokens=50
)
```

### 변환 패턴 2: 시스템 지시사항 분리

**Before**:
```python
prompt = """You are a helpful Python tutor.

User: How do I create a list?
Assistant:"""

response = openai.completions.create(
    model="gpt-3.5-turbo-instruct",
    prompt=prompt
)
```

**After**:
```python
response = openai.chat.completions.create(
    model="gpt-3.5-turbo",
    messages=[
        {"role": "system", "content": "You are a helpful Python tutor."},
        {"role": "user", "content": "How do I create a list?"}
    ]
)
```

### 변환 패턴 3: Few-shot Learning

**Before**:
```python
prompt = """Classify sentiment:

Text: I love this product!
Sentiment: Positive

Text: This is terrible.
Sentiment: Negative

Text: It's okay, nothing special.
Sentiment:"""

response = openai.completions.create(
    model="gpt-3.5-turbo-instruct",
    prompt=prompt,
    max_tokens=5
)
```

**After**:
```python
response = openai.chat.completions.create(
    model="gpt-3.5-turbo",
    messages=[
        {"role": "system", "content": "Classify the sentiment of the given text."},
        {"role": "user", "content": "I love this product!"},
        {"role": "assistant", "content": "Positive"},
        {"role": "user", "content": "This is terrible."},
        {"role": "assistant", "content": "Negative"},
        {"role": "user", "content": "It's okay, nothing special."}
    ],
    max_tokens=5
)
```

## 특수 기능: Insert Mode

Text Completion만의 고유 기능으로, 텍스트 중간에 삽입할 수 있습니다.

**요청**:
```python
response = openai.completions.create(
    model="gpt-3.5-turbo-instruct",
    prompt="The quick brown",
    suffix="jumps over the lazy dog.",
    max_tokens=10
)
```

**응답**:
```
 fox
```

**전체 결과**: "The quick brown fox jumps over the lazy dog."

**사용 사례**:
- 코드 에디터 자동완성 (커서 위치 기준 양방향)
- 문서 편집 (중간 부분 생성)

## 언제 Text Completion을 사용할까?

### 적합한 경우

1. **레거시 시스템 유지보수**
   - 기존 코드가 Text Completion 사용 중

2. **단순 텍스트 생성**
   - 블로그 글 이어쓰기
   - 스토리 생성

3. **Insert Mode 필요**
   - 코드 자동완성 IDE
   - 텍스트 편집기 플러그인

4. **비용 최적화** (일부 상황)
   - `gpt-3.5-turbo-instruct`가 더 저렴할 수 있음

### 부적합한 경우 (Chat Completion 권장)

1. **대화형 애플리케이션**
   - 챗봇, 가상 어시스턴트

2. **멀티턴 대화**
   - 컨텍스트 유지 필요

3. **함수 호출**
   - API 통합, 도구 사용

4. **최신 모델 사용**
   - GPT-4, GPT-4 Turbo는 Chat Completion만 지원

## 마이그레이션 체크리스트

Text Completion → Chat Completion으로 전환 시:

- [ ] 모델명 변경 (`gpt-3.5-turbo-instruct` → `gpt-3.5-turbo` 또는 `gpt-4`)
- [ ] API 엔드포인트 변경 (`/v1/completions` → `/v1/chat/completions`)
- [ ] `prompt` → `messages` 배열로 변환
- [ ] 시스템 지시사항을 `system` 롤로 분리
- [ ] Few-shot 예제를 `user`/`assistant` 메시지 쌍으로 변환
- [ ] 응답 파싱 변경 (`choices[].text` → `choices[].message.content`)
- [ ] `suffix` 기능 사용 시 대안 고려 (Chat Completion은 미지원)
- [ ] 테스트 및 검증

## 다음 단계

기본적인 텍스트 인터페이스를 마스터했습니다. 이제 **Function Calling과 Tool Use**를 통해 LLM이 외부 도구와 상호작용하는 방법을 배워봅시다!

➡️ [4장: Function Calling & Tool Use](./04-function-calling.md)

---

## 핵심 요약

- ✅ Text Completion은 레거시 인터페이스 (단순 텍스트 이어쓰기)
- ✅ Chat Completion과 달리 구조화되지 않은 단일 문자열 프롬프트 사용
- ✅ Insert Mode (suffix)는 Text Completion만의 고유 기능
- ✅ GPT-4는 Text Completion 미지원 (Chat Completion만 지원)
- ✅ 대부분의 경우 Chat Completion으로 마이그레이션 권장
- ✅ Logprobs로 토큰별 확률 정보 확인 가능
