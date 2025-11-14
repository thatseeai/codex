# 8장: 고급 파라미터와 설정

LLM API의 파라미터들은 **출력의 품질, 창의성, 일관성**을 제어하는 핵심 도구입니다. 이 장에서는 각 파라미터가 실제로 어떻게 작동하는지 깊이 있게 다룹니다.

## Temperature (온도)

### 개념

Temperature는 **출력의 무작위성(랜덤성)**을 제어합니다.

```python
# Temperature 범위: 0.0 ~ 2.0
# - 0.0: 가장 결정적 (deterministic)
# - 1.0: 기본값 (balanced)
# - 2.0: 매우 창의적/무작위적
```

### 작동 원리

```python
# LLM이 다음 토큰을 선택하는 과정

# 1. 각 토큰의 로짓(logit) 계산
logits = {
    "cat": 2.5,
    "dog": 2.3,
    "bird": 1.8,
    "fish": 1.2
}

# 2. Temperature 적용
# 조정된 로짓 = 원래 로짓 / temperature

## Temperature = 0.5 (낮음 → 더 결정적)
adjusted_logits = {
    "cat": 2.5 / 0.5 = 5.0,   # 상위 선택지 강화
    "dog": 2.3 / 0.5 = 4.6,
    "bird": 1.8 / 0.5 = 3.6,
    "fish": 1.2 / 0.5 = 2.4
}
# → "cat" 선택 확률 매우 높음

## Temperature = 2.0 (높음 → 더 무작위적)
adjusted_logits = {
    "cat": 2.5 / 2.0 = 1.25,  # 차이 완화
    "dog": 2.3 / 2.0 = 1.15,
    "bird": 1.8 / 2.0 = 0.9,
    "fish": 1.2 / 2.0 = 0.6
}
# → 모든 선택지가 비슷한 확률

# 3. Softmax로 확률 변환
# 4. 확률 분포에서 샘플링
```

### 실전 예제

```python
prompts = [
    {"role": "user", "content": "고양이에 대한 짧은 시를 써주세요"}
]

# Temperature = 0.0 (매우 결정적)
response_0 = client.chat.completions.create(
    model="gpt-4",
    messages=prompts,
    temperature=0.0
)

# Temperature = 0.7 (균형)
response_07 = client.chat.completions.create(
    model="gpt-4",
    messages=prompts,
    temperature=0.7
)

# Temperature = 1.5 (매우 창의적)
response_15 = client.chat.completions.create(
    model="gpt-4",
    messages=prompts,
    temperature=1.5
)
```

**결과 비교**:

**Temperature = 0.0**:
```
고양이

부드러운 털, 날렵한 몸
창가에 앉아 햇살을 받네
조용히 눈을 감고
평화로운 오후를 즐기네
```

**Temperature = 0.7**:
```
고양이의 오후

햇살 가득한 창턱 위
꼬리를 감싼 작은 털뭉치
꿈속에서 날아다니는
나비를 쫓는 작은 사냥꾼
```

**Temperature = 1.5**:
```
양자 고양이

슈뢰딩거의 상자를 탈출한
존재와 비존재 사이를 배회하는
파동과 입자의 이중주
우주의 모든 가능성을 품은 눈동자
```

### 사용 가이드

| Temperature | 사용 사례 | 예시 |
|-------------|----------|------|
| **0.0 - 0.3** | 정확성이 중요한 작업 | - 코드 생성<br>- 번역<br>- 요약<br>- 분류 |
| **0.4 - 0.7** | 균형 잡힌 작업 | - 일반 Q&A<br>- 설명<br>- 교육 콘텐츠 |
| **0.8 - 1.2** | 창의적 작업 | - 스토리 작성<br>- 브레인스토밍<br>- 마케팅 카피 |
| **1.3 - 2.0** | 매우 창의적/실험적 | - 예술 작품<br>- 시<br>- 기발한 아이디어 |

## Top-p (Nucleus Sampling)

### 개념

Top-p는 **누적 확률이 p에 도달할 때까지의 토큰만 고려**합니다.

```python
# Top-p 범위: 0.0 ~ 1.0
# - 0.1: 상위 10% 확률의 토큰만
# - 0.9: 상위 90% 확률의 토큰들 (기본값)
# - 1.0: 모든 토큰 고려
```

### 작동 원리

```python
# 토큰별 확률 (Softmax 후)
probabilities = {
    "cat": 0.35,
    "dog": 0.25,
    "bird": 0.20,
    "fish": 0.10,
    "snake": 0.05,
    "lion": 0.03,
    "bear": 0.02
}

## Top-p = 0.5
# 누적 확률:
# "cat": 0.35
# "dog": 0.35 + 0.25 = 0.60 (> 0.5, 여기서 중단)
# → "cat", "dog"만 고려

## Top-p = 0.9
# 누적 확률:
# "cat": 0.35
# "dog": 0.60
# "bird": 0.80
# "fish": 0.90 (= 0.9, 여기서 중단)
# → "cat", "dog", "bird", "fish" 고려
```

### Temperature vs Top-p

```python
# Temperature와 Top-p는 함께 사용 가능하지만,
# 보통 하나만 조정하는 것을 권장

# 패턴 1: Temperature만 조정
response = client.chat.completions.create(
    model="gpt-4",
    messages=[...],
    temperature=0.8,
    top_p=1.0  # 기본값
)

# 패턴 2: Top-p만 조정
response = client.chat.completions.create(
    model="gpt-4",
    messages=[...],
    temperature=1.0,  # 기본값
    top_p=0.95
)

# ❌ 둘 다 동시에 낮추면 과도하게 제한적
response = client.chat.completions.create(
    model="gpt-4",
    messages=[...],
    temperature=0.3,
    top_p=0.1  # 너무 제한적!
)
```

### 사용 가이드

| Top-p | 효과 | 사용 사례 |
|-------|------|----------|
| **0.1 - 0.5** | 매우 보수적 선택 | 정확성이 극도로 중요한 경우 |
| **0.6 - 0.9** | 균형 잡힌 다양성 | 대부분의 작업 (권장) |
| **0.95 - 1.0** | 최대 다양성 | 창의적 작업 |

## Top-k (상위 K 샘플링)

일부 모델에서 지원 (예: Google Gemini):

```python
# Top-k: 확률 상위 k개 토큰만 고려

# k = 5
# 상위 5개 토큰만 선택 가능
# 나머지는 확률 0으로 설정
```

## Max Tokens (최대 토큰 수)

### 기본 개념

```python
response = client.chat.completions.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "긴 이야기를 써주세요"}],
    max_tokens=100  # 최대 100 토큰만 생성
)

# 100 토큰에 도달하면 생성 중단
# finish_reason: "length"
```

### 토큰 계산

```python
from tiktoken import encoding_for_model

enc = encoding_for_model("gpt-4")

# 입력 토큰 계산
prompt = "Write a story about a cat"
input_tokens = len(enc.encode(prompt))
print(f"입력: {input_tokens} 토큰")  # 6 토큰

# 모델 컨텍스트 윈도우: 8,192 토큰
# 남은 공간 = 8,192 - 6 = 8,186 토큰

# 실제 사용 가능한 max_tokens
max_tokens = 8192 - input_tokens
```

### 비용 최적화

```python
# 짧은 답변만 필요한 경우
response = client.chat.completions.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "대한민국의 수도는?"}],
    max_tokens=10  # "서울입니다"만 필요
)
# 출력 비용 절감!

# 긴 답변 필요
response = client.chat.completions.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "AI의 역사를 자세히 설명해주세요"}],
    max_tokens=2000
)
```

## Presence Penalty & Frequency Penalty

### Presence Penalty (출현 패널티)

**새로운 주제를 언급하도록 장려**

```python
# 범위: -2.0 ~ 2.0
# 양수: 이미 나온 토큰의 확률 감소 (새 주제 장려)
# 음수: 이미 나온 토큰의 확률 증가 (반복 장려)

response = client.chat.completions.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "AI의 응용 분야를 나열해주세요"}],
    presence_penalty=0.6  # 다양한 주제 언급
)

# 결과: "의료, 교육, 금융, 제조, 운송, 엔터테인먼트, ..."
# (같은 분야 반복 없이 다양하게)
```

### Frequency Penalty (빈도 패널티)

**반복을 억제**

```python
# 범위: -2.0 ~ 2.0
# 양수: 자주 나온 토큰일수록 확률 더 많이 감소
# 음수: 자주 나온 토큰일수록 확률 증가

response = client.chat.completions.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "고양이에 대한 글을 써주세요"}],
    frequency_penalty=0.5  # 단어 반복 억제
)

# Frequency penalty 없음:
# "고양이는 귀여운 동물입니다. 고양이는 털이 부드럽습니다. 고양이는..."
# (고양이 반복)

# Frequency penalty 0.5:
# "고양이는 귀여운 동물입니다. 이들은 털이 부드럽고 독립적인 성격을 가지고 있습니다..."
# (대명사 사용 등으로 반복 회피)
```

### 비교

| 파라미터 | 효과 | 사용 사례 |
|----------|------|----------|
| **Presence Penalty** | 새로운 주제/단어 장려 | 브레인스토밍, 다양한 아이디어 나열 |
| **Frequency Penalty** | 반복 억제 | 자연스러운 글쓰기, 대화 |

## Stop Sequences (중단 시퀀스)

### 기본 사용

```python
response = client.chat.completions.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "1부터 10까지 세어주세요"}],
    stop=["5"]  # "5"가 나오면 중단
)

# 출력: "1, 2, 3, 4, "
# finish_reason: "stop"
```

### 다중 Stop Sequences

```python
response = client.chat.completions.create(
    model="gpt-4",
    messages=[{
        "role": "user",
        "content": "Q: What is Python?\nA:"
    }],
    stop=["\n", "Q:", "User:"]  # 여러 중단 조건
)

# "A: Python is a programming language"
# (다음 줄바꿈이나 "Q:"가 나오기 전까지만)
```

### 사용 사례

```python
# 1. Q&A 형식 제어
stop=["\nQ:", "\n\n"]

# 2. 코드 블록 제어
stop=["```", "# End of code"]

# 3. 대화 턴 제어
stop=["User:", "Human:", "\n\n"]

# 4. JSON 생성 제어
stop=["}"]  # 첫 객체만
```

## Seed (재현성)

### 결정적 출력

```python
# 동일한 seed = 동일한 출력 (대부분의 경우)

response1 = client.chat.completions.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "Random story"}],
    seed=12345,
    temperature=0.7  # seed는 temperature > 0에서도 작동
)

response2 = client.chat.completions.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "Random story"}],
    seed=12345,
    temperature=0.7
)

# response1 == response2 (높은 확률로)
```

### system_fingerprint 확인

```python
response = client.chat.completions.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "Hello"}],
    seed=12345
)

print(response.system_fingerprint)
# "fp_abc123xyz"

# 동일한 seed + 동일한 fingerprint = 재현 가능
```

### 사용 사례

```python
# A/B 테스트
def generate_variations(prompt, n=5):
    variations = []
    for i in range(n):
        response = client.chat.completions.create(
            model="gpt-4",
            messages=[{"role": "user", "content": prompt}],
            seed=i,  # 다른 seed로 다양한 결과
            temperature=0.8
        )
        variations.append(response.choices[0].message.content)
    return variations

# 재현 가능한 벤치마크
response = client.chat.completions.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "Solve this problem"}],
    seed=42,  # 고정 seed
    temperature=0
)
```

## Logprobs (로그 확률)

### 토큰별 확률 정보

```python
response = client.chat.completions.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "The sky is"}],
    logprobs=True,
    top_logprobs=3  # 상위 3개 후보 표시
)

logprobs = response.choices[0].logprobs.content

for token_data in logprobs:
    print(f"토큰: {token_data.token}")
    print(f"로그 확률: {token_data.logprob}")
    print(f"실제 확률: {math.exp(token_data.logprob):.2%}")
    print("대안 토큰:")
    for alt in token_data.top_logprobs:
        print(f"  {alt.token}: {math.exp(alt.logprob):.2%}")
```

**출력**:
```
토큰:  blue
로그 확률: -0.002
실제 확률: 99.80%
대안 토큰:
  clear: 0.15%
  gray: 0.03%
  dark: 0.02%
```

### 활용 사례

```python
# 1. 모델 신뢰도 측정
def get_confidence(response):
    logprobs = response.choices[0].logprobs.content
    avg_logprob = sum(t.logprob for t in logprobs) / len(logprobs)
    confidence = math.exp(avg_logprob)
    return confidence

# 2. 불확실한 답변 감지
if get_confidence(response) < 0.5:
    print("모델이 불확실해 보입니다. 다시 확인하세요.")

# 3. 다양성 측정
top_prob = math.exp(logprobs[0].logprob)
if top_prob > 0.95:
    print("매우 결정적인 답변")
else:
    print("여러 가능성이 있는 답변")
```

## Response Format (구조화된 출력)

### JSON 모드

```python
response = client.chat.completions.create(
    model="gpt-4",
    messages=[
        {"role": "system", "content": "Extract user info as JSON"},
        {"role": "user", "content": "My name is John, I'm 30 years old"}
    ],
    response_format={"type": "json_object"}
)

# 보장: 항상 유효한 JSON 반환
result = json.loads(response.choices[0].message.content)
print(result)
# {"name": "John", "age": 30}
```

### JSON Schema (GPT-4 Turbo)

```python
schema = {
    "type": "object",
    "properties": {
        "name": {"type": "string"},
        "age": {"type": "integer"},
        "email": {"type": "string", "format": "email"}
    },
    "required": ["name", "age"]
}

response = client.chat.completions.create(
    model="gpt-4-turbo",
    messages=[{"role": "user", "content": "John, 30, john@example.com"}],
    response_format={
        "type": "json_schema",
        "json_schema": {
            "name": "user_info",
            "strict": True,
            "schema": schema
        }
    }
)

# 스키마를 엄격히 준수하는 JSON 반환 보장
```

## 파라미터 조합 가이드

### 정확성이 중요한 작업

```python
response = client.chat.completions.create(
    model="gpt-4",
    messages=[...],
    temperature=0.0,      # 결정적
    top_p=1.0,            # 기본값
    presence_penalty=0.0,
    frequency_penalty=0.0,
    seed=42               # 재현 가능
)
```

### 창의적 작업

```python
response = client.chat.completions.create(
    model="gpt-4",
    messages=[...],
    temperature=1.2,      # 창의적
    top_p=0.95,
    presence_penalty=0.6, # 다양한 주제
    frequency_penalty=0.5 # 반복 억제
)
```

### 비용 최적화

```python
response = client.chat.completions.create(
    model="gpt-3.5-turbo",  # 저렴한 모델
    messages=[...],
    max_tokens=100,        # 토큰 제한
    temperature=0.3        # 빠른 수렴
)
```

## 다음 단계

파라미터를 마스터했습니다! 다음은 **디자인 패턴과 베스트 프랙티스**에서 실전 활용법을 배워봅시다.

➡️ [9장: 디자인 패턴과 베스트 프랙티스](./09-patterns-best-practices.md)

---

## 핵심 요약

- ✅ Temperature: 무작위성 제어 (0=결정적, 2=창의적)
- ✅ Top-p: 누적 확률 기반 샘플링 (0.9 권장)
- ✅ Max Tokens: 출력 길이 및 비용 제어
- ✅ Presence Penalty: 새 주제 장려
- ✅ Frequency Penalty: 반복 억제
- ✅ Stop Sequences: 특정 패턴에서 중단
- ✅ Seed: 재현 가능한 출력
- ✅ Logprobs: 모델 신뢰도 측정
- ✅ Response Format: JSON 출력 보장
