# 9장: 디자인 패턴과 베스트 프랙티스

실전에서 LLM API를 효과적으로 사용하기 위한 검증된 패턴과 전략들을 다룹니다.

## 프롬프트 템플릿 패턴

### 패턴 1: 역할 기반 템플릿

```python
class PromptTemplate:
    def __init__(self, role, task_description):
        self.role = role
        self.task_description = task_description

    def format(self, **kwargs):
        system_prompt = f"You are a {self.role}. {self.task_description}"
        user_prompt = self.user_template.format(**kwargs)
        return {
            "system": system_prompt,
            "user": user_prompt
        }

# 사용
translator = PromptTemplate(
    role="professional translator",
    task_description="Translate text accurately while preserving tone and context."
)
translator.user_template = "Translate the following to {target_language}:\n\n{text}"

prompts = translator.format(
    target_language="Korean",
    text="Hello, world!"
)

response = client.chat.completions.create(
    model="gpt-4",
    messages=[
        {"role": "system", "content": prompts["system"]},
        {"role": "user", "content": prompts["user"]}
    ]
)
```

### 패턴 2: Few-Shot 템플릿

```python
class FewShotTemplate:
    def __init__(self, task, examples):
        self.task = task
        self.examples = examples

    def build_messages(self, query):
        messages = [
            {"role": "system", "content": self.task}
        ]

        # Few-shot 예제 추가
        for example in self.examples:
            messages.append({"role": "user", "content": example["input"]})
            messages.append({"role": "assistant", "content": example["output"]})

        # 실제 질문
        messages.append({"role": "user", "content": query})
        return messages

# 사용: 감정 분석
sentiment_classifier = FewShotTemplate(
    task="Classify the sentiment of text as Positive, Negative, or Neutral.",
    examples=[
        {"input": "I love this product!", "output": "Positive"},
        {"input": "This is terrible.", "output": "Negative"},
        {"input": "It's okay.", "output": "Neutral"}
    ]
)

messages = sentiment_classifier.build_messages("The service was amazing!")
response = client.chat.completions.create(model="gpt-4", messages=messages)
```

### 패턴 3: Chain-of-Thought (사고 사슬)

```python
def chain_of_thought_prompt(problem):
    return f"""Let's solve this step by step:

Problem: {problem}

Step 1: Understand the problem
Step 2: Identify key information
Step 3: Apply relevant concepts
Step 4: Calculate or reason through
Step 5: Verify the answer

Solution:"""

# 사용
response = client.chat.completions.create(
    model="gpt-4",
    messages=[{
        "role": "user",
        "content": chain_of_thought_prompt("If a train travels 120 km in 2 hours, what is its average speed?")
    }]
)

# 모델이 단계별로 사고 과정을 보여줌
# → 더 정확한 답변
```

## 에러 핸들링

### 패턴 1: 재시도 with Exponential Backoff

```python
import time
from openai import OpenAI, APIError, RateLimitError, APIConnectionError

def chat_with_retry(messages, max_retries=5):
    client = OpenAI()
    base_delay = 1  # 초

    for attempt in range(max_retries):
        try:
            response = client.chat.completions.create(
                model="gpt-4",
                messages=messages,
                timeout=30.0
            )
            return response

        except RateLimitError as e:
            if attempt == max_retries - 1:
                raise

            # Exponential backoff with jitter
            delay = base_delay * (2 ** attempt) + random.uniform(0, 1)
            print(f"Rate limited. Retrying in {delay:.1f}s...")
            time.sleep(delay)

        except APIConnectionError as e:
            if attempt == max_retries - 1:
                raise

            delay = base_delay * (2 ** attempt)
            print(f"Connection error. Retrying in {delay:.1f}s...")
            time.sleep(delay)

        except APIError as e:
            # 서버 에러 (5xx)는 재시도
            if e.status_code >= 500:
                if attempt == max_retries - 1:
                    raise
                delay = base_delay * (2 ** attempt)
                print(f"Server error {e.status_code}. Retrying...")
                time.sleep(delay)
            else:
                # 클라이언트 에러 (4xx)는 즉시 실패
                raise

    raise Exception("Max retries exceeded")
```

### 패턴 2: Fallback 모델

```python
def chat_with_fallback(messages):
    models = [
        "gpt-4",
        "gpt-3.5-turbo",
        "gpt-3.5-turbo-16k"
    ]

    last_error = None

    for model in models:
        try:
            response = client.chat.completions.create(
                model=model,
                messages=messages
            )
            return response, model

        except Exception as e:
            print(f"{model} failed: {e}")
            last_error = e
            continue

    raise last_error or Exception("All models failed")

# 사용
response, used_model = chat_with_fallback(messages)
print(f"Used model: {used_model}")
```

### 패턴 3: Circuit Breaker

```python
class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60):
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.last_failure_time = None
        self.state = "CLOSED"  # CLOSED, OPEN, HALF_OPEN

    def call(self, func, *args, **kwargs):
        if self.state == "OPEN":
            if time.time() - self.last_failure_time > self.timeout:
                self.state = "HALF_OPEN"
            else:
                raise Exception("Circuit breaker is OPEN")

        try:
            result = func(*args, **kwargs)
            if self.state == "HALF_OPEN":
                self.state = "CLOSED"
                self.failure_count = 0
            return result

        except Exception as e:
            self.failure_count += 1
            self.last_failure_time = time.time()

            if self.failure_count >= self.failure_threshold:
                self.state = "OPEN"

            raise

# 사용
breaker = CircuitBreaker()

def api_call():
    return client.chat.completions.create(
        model="gpt-4",
        messages=[{"role": "user", "content": "Hello"}]
    )

try:
    response = breaker.call(api_call)
except Exception as e:
    print(f"Call failed: {e}")
```

## 컨텍스트 윈도우 관리

### 패턴 1: 슬라이딩 윈도우

```python
from tiktoken import encoding_for_model

class ConversationManager:
    def __init__(self, model="gpt-4", max_tokens=8000):
        self.model = model
        self.max_tokens = max_tokens
        self.encoder = encoding_for_model(model)
        self.messages = []

    def count_tokens(self, messages):
        total = 0
        for message in messages:
            # 메시지 오버헤드: 약 4 토큰
            total += 4
            total += len(self.encoder.encode(message["content"]))
        return total

    def add_message(self, role, content):
        self.messages.append({"role": role, "content": content})
        self._trim_messages()

    def _trim_messages(self):
        # 시스템 메시지는 항상 유지
        system_messages = [m for m in self.messages if m["role"] == "system"]
        other_messages = [m for m in self.messages if m["role"] != "system"]

        # 토큰 수가 제한을 초과하면 오래된 메시지부터 제거
        while self.count_tokens(system_messages + other_messages) > self.max_tokens:
            if len(other_messages) > 1:
                other_messages.pop(0)  # 가장 오래된 메시지 제거
            else:
                break

        self.messages = system_messages + other_messages

    def get_response(self):
        response = client.chat.completions.create(
            model=self.model,
            messages=self.messages
        )
        assistant_message = response.choices[0].message.content
        self.add_message("assistant", assistant_message)
        return assistant_message

# 사용
conv = ConversationManager()
conv.add_message("system", "You are a helpful assistant")
conv.add_message("user", "Hello")
response = conv.get_response()
```

### 패턴 2: 요약 기반 컨텍스트 압축

```python
class SummarizingConversation:
    def __init__(self, summary_threshold=10):
        self.messages = []
        self.summary_threshold = summary_threshold
        self.conversation_summary = ""

    def add_and_manage(self, role, content):
        self.messages.append({"role": role, "content": content})

        # 메시지가 임계값을 넘으면 요약
        if len(self.messages) > self.summary_threshold:
            self._summarize_old_messages()

    def _summarize_old_messages(self):
        # 오래된 메시지 절반을 요약
        to_summarize = self.messages[:len(self.messages)//2]

        summary_response = client.chat.completions.create(
            model="gpt-3.5-turbo",
            messages=[
                {"role": "system", "content": "Summarize the following conversation concisely."},
                {"role": "user", "content": str(to_summarize)}
            ]
        )

        self.conversation_summary = summary_response.choices[0].message.content
        self.messages = self.messages[len(self.messages)//2:]

    def get_full_context(self):
        if self.conversation_summary:
            return [
                {"role": "system", "content": f"Previous conversation summary: {self.conversation_summary}"}
            ] + self.messages
        return self.messages
```

## 비용 최적화

### 패턴 1: 캐싱

```python
import hashlib
import json
from functools import lru_cache

class LLMCache:
    def __init__(self):
        self.cache = {}

    def _hash_request(self, model, messages, **kwargs):
        # 요청을 해시화하여 캐시 키로 사용
        request_str = json.dumps({
            "model": model,
            "messages": messages,
            **kwargs
        }, sort_keys=True)
        return hashlib.md5(request_str.encode()).hexdigest()

    def get_or_create(self, model, messages, **kwargs):
        cache_key = self._hash_request(model, messages, **kwargs)

        if cache_key in self.cache:
            print("Cache HIT")
            return self.cache[cache_key]

        print("Cache MISS")
        response = client.chat.completions.create(
            model=model,
            messages=messages,
            **kwargs
        )

        self.cache[cache_key] = response
        return response

# 사용
cache = LLMCache()

# 첫 번째 호출: API 요청
response1 = cache.get_or_create(
    model="gpt-4",
    messages=[{"role": "user", "content": "What is Python?"}],
    temperature=0
)

# 두 번째 호출 (동일): 캐시에서 반환
response2 = cache.get_or_create(
    model="gpt-4",
    messages=[{"role": "user", "content": "What is Python?"}],
    temperature=0
)
# → 비용 절감!
```

### 패턴 2: 배치 처리

```python
async def batch_process(items, batch_size=10):
    import asyncio
    from openai import AsyncOpenAI

    client = AsyncOpenAI()
    results = []

    # 배치로 나누기
    for i in range(0, len(items), batch_size):
        batch = items[i:i + batch_size]

        # 병렬 처리
        tasks = [
            client.chat.completions.create(
                model="gpt-3.5-turbo",
                messages=[{"role": "user", "content": item}]
            )
            for item in batch
        ]

        batch_results = await asyncio.gather(*tasks)
        results.extend(batch_results)

        # Rate limit 방지
        if i + batch_size < len(items):
            await asyncio.sleep(1)

    return results

# 사용
items = ["Translate: Hello", "Translate: World", ...]
results = asyncio.run(batch_process(items))
```

### 패턴 3: 모델 선택 전략

```python
def choose_model(task_complexity, max_cost=0.01):
    """작업 복잡도에 따라 적절한 모델 선택"""

    models = [
        {"name": "gpt-4", "capability": 10, "cost": 0.03},
        {"name": "gpt-3.5-turbo", "capability": 7, "cost": 0.002},
        {"name": "gpt-3.5-turbo-16k", "capability": 7, "cost": 0.003}
    ]

    # 비용 제약 내에서 가장 강력한 모델 선택
    suitable = [m for m in models if m["cost"] <= max_cost]
    if not suitable:
        raise ValueError("No model within budget")

    # 복잡도 요구사항 충족 모델 중 가장 저렴한 것
    suitable = [m for m in suitable if m["capability"] >= task_complexity]
    if not suitable:
        # 복잡도 요구사항 충족 못하면 가장 강력한 모델
        suitable = models

    return min(suitable, key=lambda m: m["cost"])

# 사용
task = "Explain quantum physics"  # 복잡도 9
model = choose_model(task_complexity=9, max_cost=0.01)
print(f"Selected: {model['name']}")
```

## 프롬프트 검증

### 패턴 1: 입력 검증

```python
def validate_prompt(prompt, max_length=10000):
    """프롬프트 유효성 검사"""

    # 길이 체크
    if len(prompt) > max_length:
        raise ValueError(f"Prompt too long: {len(prompt)} > {max_length}")

    # 유해 콘텐츠 체크
    moderation = client.moderations.create(input=prompt)
    if moderation.results[0].flagged:
        raise ValueError("Prompt contains inappropriate content")

    # 개인정보 감지 (간단한 예시)
    import re
    if re.search(r'\b\d{3}-\d{2}-\d{4}\b', prompt):  # SSN 패턴
        raise ValueError("Prompt may contain PII")

    return True

# 사용
try:
    validate_prompt(user_input)
    response = client.chat.completions.create(...)
except ValueError as e:
    print(f"Invalid prompt: {e}")
```

### 패턴 2: 출력 검증

```python
def validate_response(response, expected_format="text"):
    """응답 유효성 검사"""

    content = response.choices[0].message.content

    if expected_format == "json":
        try:
            data = json.loads(content)
            return data
        except json.JSONDecodeError:
            raise ValueError("Response is not valid JSON")

    elif expected_format == "code":
        # 코드 블록 체크
        if "```" not in content:
            raise ValueError("Response does not contain code block")

    # 길이 체크
    if len(content) < 10:
        raise ValueError("Response too short, likely failed")

    return content

# 사용
response = client.chat.completions.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "Return JSON with user info"}],
    response_format={"type": "json_object"}
)

try:
    data = validate_response(response, expected_format="json")
    print(data)
except ValueError as e:
    print(f"Invalid response: {e}")
    # 재시도 로직...
```

## 멀티모달 베스트 프랙티스

### 이미지 최적화

```python
from PIL import Image
import base64
from io import BytesIO

def optimize_image(image_path, max_size=(2048, 2048), quality=85):
    """이미지 크기 및 품질 최적화"""

    img = Image.open(image_path)

    # 크기 조정
    img.thumbnail(max_size, Image.Resampling.LANCZOS)

    # Base64 인코딩
    buffered = BytesIO()
    img.save(buffered, format="JPEG", quality=quality, optimize=True)
    img_str = base64.b64encode(buffered.getvalue()).decode()

    return f"data:image/jpeg;base64,{img_str}"

# 사용
optimized_image = optimize_image("large_photo.jpg")

response = client.chat.completions.create(
    model="gpt-4-vision-preview",
    messages=[{
        "role": "user",
        "content": [
            {"type": "text", "text": "What's in this image?"},
            {"type": "image_url", "image_url": {"url": optimized_image, "detail": "low"}}
        ]
    }]
)
# → 비용 절감 + 빠른 응답
```

## 다음 단계

실전 패턴을 마스터했습니다! 다음은 **API 비교 및 마이그레이션**에서 다양한 LLM 플랫폼을 비교해봅시다.

➡️ [10장: API 비교 및 마이그레이션](./10-api-comparison.md)

---

## 핵심 요약

- ✅ 템플릿 패턴으로 프롬프트 재사용성 향상
- ✅ Exponential backoff로 안정적인 재시도
- ✅ 슬라이딩 윈도우로 컨텍스트 관리
- ✅ 캐싱으로 비용 절감
- ✅ 배치 처리로 효율성 향상
- ✅ 입력/출력 검증으로 품질 보장
- ✅ 이미지 최적화로 멀티모달 비용 절감
