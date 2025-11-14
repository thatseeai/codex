# 5장: 멀티모달 인터페이스

멀티모달(Multimodal) 인터페이스는 **텍스트뿐만 아니라 이미지, 음성, 파일 등 다양한 형식의 입력을 처리**할 수 있게 합니다. GPT-4 Vision, Claude 3, Gemini Pro Vision 등이 이를 지원합니다.

## 멀티모달이란?

```
[텍스트만] → "고양이를 설명해주세요"
            → "고양이는 털이 있고 네 발로 걷는 동물입니다..."

[멀티모달] → [이미지: 고양이 사진] + "이 이미지를 설명해주세요"
            → "이 이미지에는 주황색 털을 가진 고양이가 창가에 앉아 있습니다..."
```

## Vision (이미지 입력)

### 입력 형식

#### 방법 1: Base64 인코딩

```json
{
  "model": "gpt-4-vision-preview",
  "messages": [
    {
      "role": "user",
      "content": [
        {
          "type": "text",
          "text": "이 이미지에 뭐가 있나요?"
        },
        {
          "type": "image_url",
          "image_url": {
            "url": "data:image/jpeg;base64,/9j/4AAQSkZJRg..."
          }
        }
      ]
    }
  ],
  "max_tokens": 300
}
```

**Python 예제**:
```python
import base64
from openai import OpenAI

client = OpenAI()

# 이미지를 Base64로 인코딩
def encode_image(image_path):
    with open(image_path, "rb") as image_file:
        return base64.b64encode(image_file.read()).decode('utf-8')

image_base64 = encode_image("cat.jpg")

response = client.chat.completions.create(
    model="gpt-4-vision-preview",
    messages=[
        {
            "role": "user",
            "content": [
                {"type": "text", "text": "이 이미지를 설명해주세요"},
                {
                    "type": "image_url",
                    "image_url": {
                        "url": f"data:image/jpeg;base64,{image_base64}"
                    }
                }
            ]
        }
    ],
    max_tokens=300
)

print(response.choices[0].message.content)
```

#### 방법 2: URL 직접 전달

```python
response = client.chat.completions.create(
    model="gpt-4-vision-preview",
    messages=[
        {
            "role": "user",
            "content": [
                {"type": "text", "text": "이 이미지의 주요 색상은?"},
                {
                    "type": "image_url",
                    "image_url": {
                        "url": "https://example.com/image.jpg"
                    }
                }
            ]
        }
    ]
)
```

### 이미지 상세도 제어 (detail)

```python
{
    "type": "image_url",
    "image_url": {
        "url": "https://example.com/image.jpg",
        "detail": "high"  # "low", "high", "auto"
    }
}
```

| detail 값 | 토큰 사용 | 처리 방식 |
|-----------|-----------|-----------|
| `low` | 85 토큰 (고정) | 저해상도 (512x512) |
| `high` | 가변 (129~1105 토큰) | 고해상도 타일 처리 |
| `auto` | 자동 선택 | 이미지 크기 기반 |

**비용 최적화**:
```python
# 간단한 분류 작업 → low detail
response = client.chat.completions.create(
    model="gpt-4-vision-preview",
    messages=[{
        "role": "user",
        "content": [
            {"type": "text", "text": "이 이미지가 고양이인가요 개인가요?"},
            {"type": "image_url", "image_url": {"url": url, "detail": "low"}}
        ]
    }]
)  # 85 토큰만 사용

# 세밀한 분석 필요 → high detail
response = client.chat.completions.create(
    model="gpt-4-vision-preview",
    messages=[{
        "role": "user",
        "content": [
            {"type": "text", "text": "이 건축 도면의 모든 치수를 읽어주세요"},
            {"type": "image_url", "image_url": {"url": url, "detail": "high"}}
        ]
    }]
)  # 500+ 토큰 사용 가능
```

### 다중 이미지 입력

```python
response = client.chat.completions.create(
    model="gpt-4-vision-preview",
    messages=[
        {
            "role": "user",
            "content": [
                {"type": "text", "text": "이 두 이미지의 차이점을 설명해주세요"},
                {"type": "image_url", "image_url": {"url": "https://example.com/before.jpg"}},
                {"type": "image_url", "image_url": {"url": "https://example.com/after.jpg"}}
            ]
        }
    ]
)
```

### 출력 형식

Vision 응답은 일반 Chat Completion과 동일합니다:

```json
{
  "id": "chatcmpl-abc123",
  "choices": [
    {
      "message": {
        "role": "assistant",
        "content": "이 이미지에는 주황색 고양이가 창가에 앉아 밖을 바라보고 있습니다. 고양이는 긴 털을 가지고 있으며, 배경에는 푸른 하늘과 나무가 보입니다."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 1150,  # 이미지 토큰 포함
    "completion_tokens": 45,
    "total_tokens": 1195
  }
}
```

**주의**: `prompt_tokens`에 이미지 처리 토큰이 포함됩니다!

## Audio (음성 입력/출력)

### Text-to-Speech (TTS)

```python
from openai import OpenAI
client = OpenAI()

# 텍스트 → 음성
response = client.audio.speech.create(
    model="tts-1",  # 또는 "tts-1-hd" (고품질)
    voice="alloy",  # "echo", "fable", "onyx", "nova", "shimmer"
    input="안녕하세요, 저는 AI 어시스턴트입니다."
)

# 파일로 저장
response.stream_to_file("output.mp3")
```

**Voice 옵션**:
| Voice | 특징 |
|-------|------|
| `alloy` | 중성적, 균형잡힌 |
| `echo` | 남성적 |
| `fable` | 영국식 억양 |
| `onyx` | 깊은 남성 목소리 |
| `nova` | 밝은 여성 목소리 |
| `shimmer` | 부드러운 여성 목소리 |

### Speech-to-Text (STT / Whisper)

```python
# 음성 → 텍스트
audio_file = open("speech.mp3", "rb")

transcript = client.audio.transcriptions.create(
    model="whisper-1",
    file=audio_file,
    language="ko"  # 선택적: 언어 지정
)

print(transcript.text)
# 출력: "안녕하세요, 저는 AI 어시스턴트입니다."
```

**고급 옵션**:
```python
transcript = client.audio.transcriptions.create(
    model="whisper-1",
    file=audio_file,
    response_format="verbose_json",  # "json", "text", "srt", "vtt"
    timestamp_granularities=["word"]  # 단어별 타임스탬프
)

print(transcript.words)
# [
#   {"word": "안녕하세요", "start": 0.0, "end": 0.5},
#   {"word": "저는", "start": 0.6, "end": 0.8},
#   ...
# ]
```

### Translation (번역)

```python
# 외국어 음성 → 영어 텍스트
audio_file = open("korean_speech.mp3", "rb")

translation = client.audio.translations.create(
    model="whisper-1",
    file=audio_file
)

print(translation.text)
# 한국어 음성이었어도 영어로 번역됨
```

## Anthropic Vision

### Claude 3 Vision 입력

```python
import anthropic
import base64

client = anthropic.Anthropic()

# 이미지 읽기
with open("image.jpg", "rb") as f:
    image_data = base64.standard_b64encode(f.read()).decode("utf-8")

message = client.messages.create(
    model="claude-3-5-sonnet-20241022",
    max_tokens=1024,
    messages=[
        {
            "role": "user",
            "content": [
                {
                    "type": "image",
                    "source": {
                        "type": "base64",
                        "media_type": "image/jpeg",
                        "data": image_data
                    }
                },
                {
                    "type": "text",
                    "text": "이 이미지를 설명해주세요"
                }
            ]
        }
    ]
)

print(message.content[0].text)
```

**OpenAI와의 차이**:
- `image_url` → `image`
- `url` → `source` 객체
- `source.type`: `"base64"` 또는 `"url"`
- `media_type` 명시 필요

## Document Processing (PDF, 파일)

### PDF 처리 (Claude)

```python
import anthropic
import base64

client = anthropic.Anthropic()

# PDF를 Base64로 인코딩
with open("document.pdf", "rb") as f:
    pdf_data = base64.standard_b64encode(f.read()).decode("utf-8")

message = client.messages.create(
    model="claude-3-5-sonnet-20241022",
    max_tokens=1024,
    messages=[
        {
            "role": "user",
            "content": [
                {
                    "type": "document",
                    "source": {
                        "type": "base64",
                        "media_type": "application/pdf",
                        "data": pdf_data
                    }
                },
                {
                    "type": "text",
                    "text": "이 PDF의 주요 내용을 요약해주세요"
                }
            ]
        }
    ]
)
```

**지원 형식**:
- `application/pdf`
- `text/plain`
- `text/csv`
- `application/json`

## 실전 예제

### 예제 1: 이미지 기반 Q&A

```python
def analyze_product_image(image_url):
    response = client.chat.completions.create(
        model="gpt-4-vision-preview",
        messages=[
            {
                "role": "user",
                "content": [
                    {
                        "type": "text",
                        "text": """이 제품 이미지를 분석하여 다음 정보를 JSON 형식으로 반환해주세요:
                        - category: 제품 카테고리
                        - color: 주요 색상
                        - brand: 브랜드 (보이는 경우)
                        - condition: 제품 상태 (새것/중고)
                        - description: 간단한 설명"""
                    },
                    {
                        "type": "image_url",
                        "image_url": {"url": image_url}
                    }
                ]
            }
        ],
        response_format={"type": "json_object"},
        max_tokens=300
    )

    return json.loads(response.choices[0].message.content)

# 사용
result = analyze_product_image("https://example.com/product.jpg")
print(result)
# {
#   "category": "신발",
#   "color": "흰색/검정",
#   "brand": "Nike",
#   "condition": "새것",
#   "description": "나이키 에어맥스 운동화, 흰색 바탕에 검정 로고"
# }
```

### 예제 2: OCR (텍스트 추출)

```python
def extract_text_from_image(image_path):
    image_base64 = encode_image(image_path)

    response = client.chat.completions.create(
        model="gpt-4-vision-preview",
        messages=[
            {
                "role": "user",
                "content": [
                    {
                        "type": "text",
                        "text": "이 이미지의 모든 텍스트를 정확히 추출해주세요. 레이아웃을 유지하세요."
                    },
                    {
                        "type": "image_url",
                        "image_url": {
                            "url": f"data:image/jpeg;base64,{image_base64}",
                            "detail": "high"  # OCR에는 high detail 권장
                        }
                    }
                ]
            }
        ],
        max_tokens=1000
    )

    return response.choices[0].message.content

# 명함, 영수증, 문서 스캔 등에 활용
text = extract_text_from_image("receipt.jpg")
```

### 예제 3: 차트/그래프 분석

```python
response = client.chat.completions.create(
    model="gpt-4-vision-preview",
    messages=[
        {
            "role": "user",
            "content": [
                {
                    "type": "text",
                    "text": "이 그래프를 분석하여 다음을 알려주세요:\n1. 그래프 유형\n2. 주요 트렌드\n3. 가장 높은/낮은 값\n4. 인사이트"
                },
                {
                    "type": "image_url",
                    "image_url": {"url": "https://example.com/sales_chart.png"}
                }
            ]
        }
    ]
)
```

### 예제 4: 음성 대화 시스템

```python
def voice_conversation():
    # 1. 사용자 음성 녹음
    audio_file = record_audio()  # 실제 녹음 함수

    # 2. STT: 음성 → 텍스트
    with open(audio_file, "rb") as f:
        transcript = client.audio.transcriptions.create(
            model="whisper-1",
            file=f
        )

    user_text = transcript.text
    print(f"사용자: {user_text}")

    # 3. LLM 응답 생성
    response = client.chat.completions.create(
        model="gpt-4",
        messages=[
            {"role": "user", "content": user_text}
        ]
    )

    assistant_text = response.choices[0].message.content
    print(f"어시스턴트: {assistant_text}")

    # 4. TTS: 텍스트 → 음성
    speech = client.audio.speech.create(
        model="tts-1",
        voice="nova",
        input=assistant_text
    )

    # 5. 음성 재생
    speech.stream_to_file("response.mp3")
    play_audio("response.mp3")  # 실제 재생 함수
```

## 제한사항 및 주의사항

### Vision 제한사항

1. **이미지 크기**
   - 최대: 20MB (Base64 인코딩 전)
   - 권장: 512x512 ~ 2048x2048

2. **파일 형식**
   - 지원: PNG, JPEG, WebP, GIF (non-animated)

3. **민감한 컨텐츠**
   - 사람 얼굴 감지 제한
   - 개인정보 포함 이미지 주의

4. **정확도**
   - 작은 텍스트는 OCR 정확도 낮을 수 있음
   - 의료/법률 문서는 검증 필요

### Audio 제한사항

1. **파일 크기**
   - Whisper: 최대 25MB

2. **파일 형식**
   - 지원: mp3, mp4, mpeg, mpga, m4a, wav, webm

3. **언어**
   - Whisper: 99개 언어 지원
   - Translation: 항상 영어로 출력

## 비용 계산

### Vision 비용 (GPT-4 Vision)

```python
# Low detail
cost_per_image_low = 85 tokens

# High detail (예: 2048x2048 이미지)
# 계산: 기본 85 + (타일 수 * 170)
# 2048x2048 → 4 타일 → 85 + (4 * 170) = 765 tokens

# 입력 비용 (예: $0.01 / 1K tokens)
cost = (765 / 1000) * 0.01  # $0.00765 per image
```

### Audio 비용

```python
# TTS
# $0.015 / 1K characters (tts-1)
# $0.030 / 1K characters (tts-1-hd)

text = "안녕하세요" * 100  # 500자
cost_tts = (500 / 1000) * 0.015  # $0.0075

# Whisper
# $0.006 / minute
audio_duration_minutes = 5
cost_whisper = 5 * 0.006  # $0.03
```

## 다음 단계

멀티모달 입력을 마스터했습니다! 다음은 **스트리밍 인터페이스**에서 실시간 응답을 구현하는 방법을 배워봅시다.

➡️ [6장: 스트리밍 인터페이스](./06-streaming.md)

---

## 핵심 요약

- ✅ Vision: `image_url` 타입으로 이미지 입력 (Base64 또는 URL)
- ✅ `detail` 파라미터로 비용 최적화 (low: 85토큰, high: 가변)
- ✅ TTS: 텍스트 → 음성 (`tts-1`, `tts-1-hd`)
- ✅ Whisper: 음성 → 텍스트 (99개 언어 지원)
- ✅ Claude는 PDF 문서 처리 지원
- ✅ 이미지 토큰이 `prompt_tokens`에 포함되어 비용 발생
- ✅ OCR, 차트 분석, 제품 이미지 분석 등 다양한 활용 가능
