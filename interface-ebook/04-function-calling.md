# 4장: Function Calling & Tool Use

Function Calling은 LLM이 **외부 도구, API, 데이터베이스와 상호작용**할 수 있게 하는 혁명적인 기능입니다. LLM을 단순한 대화 모델에서 **액션을 수행할 수 있는 에이전트**로 변화시킵니다.

## Function Calling이란?

```
[User] → "서울의 현재 날씨는?"

[LLM] → 날씨 정보가 필요함을 인식
     → get_weather(location="서울") 호출 요청

[App] → 실제 날씨 API 호출
     → {"temp": 15, "condition": "맑음"} 반환

[LLM] → "서울의 현재 날씨는 15도이며 맑습니다."
```

**핵심**: LLM이 직접 API를 호출하는 것이 아니라, **어떤 함수를 어떤 파라미터로 호출해야 하는지 알려주는 것**입니다.

## 왜 Function Calling이 필요한가?

### 문제: 환각(Hallucination)

```python
# Function Calling 없이
user: "애플 주식 현재가는?"
assistant: "애플(AAPL)의 현재 주가는 약 $175.43입니다."
# ❌ 실시간 데이터가 아니라 추측한 값!
```

### 해결: 실제 데이터 연결

```python
# Function Calling 사용
user: "애플 주식 현재가는?"
→ LLM: get_stock_price(symbol="AAPL") 호출 요청
→ App: 실제 API 호출 → $182.31
→ LLM: "애플(AAPL)의 현재 주가는 $182.31입니다."
# ✅ 실제 데이터 기반!
```

## 플랫폼별 용어

| 플랫폼 | 용어 | 개념 |
|--------|------|------|
| **OpenAI** | Function Calling (구), Tools (신) | 함수 정의 및 호출 |
| **Anthropic** | Tool Use | 도구 사용 |
| **Google** | Function Calling | 함수 호출 |
| **Cohere** | Tool Use | 도구 사용 |

**이 장에서는 OpenAI 형식을 기준으로 설명하되, Anthropic과의 차이도 다룹니다.**

## OpenAI Function Calling

### 입력 형식: 함수 정의

```json
{
  "model": "gpt-4",
  "messages": [
    {"role": "user", "content": "서울의 현재 날씨는?"}
  ],
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "get_weather",
        "description": "특정 도시의 현재 날씨 정보를 가져옵니다.",
        "parameters": {
          "type": "object",
          "properties": {
            "location": {
              "type": "string",
              "description": "도시 이름 (예: 서울, 부산)"
            },
            "unit": {
              "type": "string",
              "enum": ["celsius", "fahrenheit"],
              "description": "온도 단위"
            }
          },
          "required": ["location"]
        }
      }
    }
  ],
  "tool_choice": "auto"
}
```

### 함수 스키마 분석

#### 1. name (필수)
```json
"name": "get_weather"
```
- 함수 이름 (영문, 언더스코어만 사용)
- LLM이 이 이름으로 함수를 참조

#### 2. description (필수)
```json
"description": "특정 도시의 현재 날씨 정보를 가져옵니다."
```
- 함수의 목적 설명
- **매우 중요**: LLM이 이 설명을 보고 함수 사용 여부 결정
- 명확하고 구체적으로 작성

#### 3. parameters (필수)
```json
"parameters": {
  "type": "object",
  "properties": {
    "location": {
      "type": "string",
      "description": "도시 이름"
    }
  },
  "required": ["location"]
}
```
- **JSON Schema 형식** 사용
- `properties`: 각 파라미터 정의
- `required`: 필수 파라미터 목록
- `enum`: 허용 값 제한

### tool_choice 옵션

| 값 | 동작 |
|---|------|
| `"auto"` (기본) | LLM이 자동으로 함수 호출 여부 결정 |
| `"none"` | 함수를 절대 호출하지 않음 (일반 응답만) |
| `{"type": "function", "function": {"name": "get_weather"}}` | 특정 함수를 강제 호출 |

### 출력 형식: 함수 호출 요청

```json
{
  "id": "chatcmpl-abc123",
  "object": "chat.completion",
  "created": 1699564800,
  "model": "gpt-4-0613",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": null,
        "tool_calls": [
          {
            "id": "call_abc123",
            "type": "function",
            "function": {
              "name": "get_weather",
              "arguments": "{\"location\": \"서울\", \"unit\": \"celsius\"}"
            }
          }
        ]
      },
      "finish_reason": "tool_calls"
    }
  ]
}
```

**핵심 필드**:
- `content`: `null` (텍스트 응답 없음)
- `tool_calls`: 호출할 함수 목록
- `tool_calls[].id`: 호출 ID (응답 시 참조용)
- `tool_calls[].function.name`: 함수 이름
- `tool_calls[].function.arguments`: JSON 문자열 형식의 파라미터
- `finish_reason`: `"tool_calls"` (함수 호출 요청으로 종료)

## 전체 실행 흐름

### Step-by-Step 예제

```python
import openai
import json

client = openai.OpenAI()

# Step 1: 함수 정의
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "특정 도시의 현재 날씨 정보를 가져옵니다",
            "parameters": {
                "type": "object",
                "properties": {
                    "location": {
                        "type": "string",
                        "description": "도시 이름"
                    },
                    "unit": {
                        "type": "string",
                        "enum": ["celsius", "fahrenheit"]
                    }
                },
                "required": ["location"]
            }
        }
    }
]

# Step 2: 초기 요청
messages = [
    {"role": "user", "content": "서울과 부산의 날씨를 알려줘"}
]

response = client.chat.completions.create(
    model="gpt-4",
    messages=messages,
    tools=tools,
    tool_choice="auto"
)

# Step 3: 응답 확인
response_message = response.choices[0].message
print("Finish reason:", response.choices[0].finish_reason)

if response.choices[0].finish_reason == "tool_calls":
    # Step 4: 함수 호출 실행
    tool_calls = response_message.tool_calls

    # 메시지 히스토리에 추가
    messages.append(response_message)

    # 각 함수 호출 처리
    for tool_call in tool_calls:
        function_name = tool_call.function.name
        function_args = json.loads(tool_call.function.arguments)

        print(f"호출: {function_name}({function_args})")

        # 실제 함수 실행 (실제 API 호출)
        if function_name == "get_weather":
            # 여기서 실제 날씨 API를 호출해야 함
            function_response = get_weather_api(
                location=function_args["location"],
                unit=function_args.get("unit", "celsius")
            )
        else:
            function_response = {"error": "Unknown function"}

        # Step 5: 함수 결과를 메시지에 추가
        messages.append({
            "tool_call_id": tool_call.id,
            "role": "tool",
            "name": function_name,
            "content": json.dumps(function_response)
        })

    # Step 6: 최종 응답 생성
    final_response = client.chat.completions.create(
        model="gpt-4",
        messages=messages
    )

    print("최종 응답:", final_response.choices[0].message.content)

# 실제 날씨 API 함수 (예시)
def get_weather_api(location, unit="celsius"):
    # 실제로는 API 호출
    # 여기서는 더미 데이터 반환
    weather_data = {
        "서울": {"temp": 15, "condition": "맑음"},
        "부산": {"temp": 18, "condition": "흐림"}
    }
    return weather_data.get(location, {"temp": 0, "condition": "알 수 없음"})
```

**출력**:
```
Finish reason: tool_calls
호출: get_weather({'location': '서울', 'unit': 'celsius'})
호출: get_weather({'location': '부산', 'unit': 'celsius'})
최종 응답: 서울의 현재 날씨는 15도로 맑고, 부산은 18도로 흐립니다.
```

## 실전 예제

### 예제 1: 데이터베이스 쿼리

```python
tools = [
    {
        "type": "function",
        "function": {
            "name": "query_database",
            "description": "SQL 데이터베이스에 쿼리를 실행합니다",
            "parameters": {
                "type": "object",
                "properties": {
                    "query": {
                        "type": "string",
                        "description": "실행할 SQL 쿼리"
                    },
                    "database": {
                        "type": "string",
                        "enum": ["users", "products", "orders"],
                        "description": "대상 데이터베이스"
                    }
                },
                "required": ["query", "database"]
            }
        }
    }
]

messages = [
    {"role": "user", "content": "지난 달 가장 많이 팔린 상품 5개를 보여줘"}
]

# LLM이 SQL 쿼리 생성
response = client.chat.completions.create(
    model="gpt-4",
    messages=messages,
    tools=tools
)

# 출력: query_database 호출 요청
# arguments: {
#   "query": "SELECT product_name, SUM(quantity) as total_sold FROM orders WHERE order_date >= DATE_SUB(NOW(), INTERVAL 1 MONTH) GROUP BY product_name ORDER BY total_sold DESC LIMIT 5",
#   "database": "products"
# }
```

### 예제 2: 다중 도구 사용

```python
tools = [
    {
        "type": "function",
        "function": {
            "name": "search_web",
            "description": "웹에서 정보를 검색합니다",
            "parameters": {
                "type": "object",
                "properties": {
                    "query": {"type": "string", "description": "검색 쿼리"}
                },
                "required": ["query"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "calculate",
            "description": "수학 계산을 수행합니다",
            "parameters": {
                "type": "object",
                "properties": {
                    "expression": {"type": "string", "description": "계산식 (예: '2+2')"}
                },
                "required": ["expression"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "get_stock_price",
            "description": "주식 현재가를 조회합니다",
            "parameters": {
                "type": "object",
                "properties": {
                    "symbol": {"type": "string", "description": "주식 심볼 (예: AAPL)"}
                },
                "required": ["symbol"]
            }
        }
    }
]

messages = [
    {"role": "user", "content": "애플 주식 100주를 사려면 얼마가 필요한지 계산해줘"}
]

# LLM이 자동으로 적절한 도구 선택:
# 1. get_stock_price(symbol="AAPL")
# 2. calculate(expression="182.31 * 100")
```

### 예제 3: 순차적 함수 호출 (체이닝)

```python
# 사용자: "최신 AI 뉴스를 검색하고 요약해줘"

# Round 1: LLM → search_web("AI news") 호출 요청
# → App: 검색 결과 10개 반환

# Round 2: LLM → summarize_text(text="검색 결과...") 호출 요청
# → App: 요약 생성

# Round 3: LLM → 최종 응답 "다음은 최신 AI 뉴스 요약입니다..."
```

## Anthropic Tool Use

Anthropic Claude의 Tool Use는 OpenAI와 유사하지만 약간 다른 형식입니다.

### 입력 형식

```json
{
  "model": "claude-3-5-sonnet-20241022",
  "max_tokens": 1024,
  "tools": [
    {
      "name": "get_weather",
      "description": "특정 도시의 현재 날씨 정보를 가져옵니다.",
      "input_schema": {
        "type": "object",
        "properties": {
          "location": {
            "type": "string",
            "description": "도시 이름"
          },
          "unit": {
            "type": "string",
            "enum": ["celsius", "fahrenheit"]
          }
        },
        "required": ["location"]
      }
    }
  ],
  "messages": [
    {"role": "user", "content": "서울의 날씨는?"}
  ]
}
```

**OpenAI와의 차이**:
- `tools[].type` 없음 (모든 도구가 function 타입)
- `tools[].function` → 직접 최상위에 `name`, `description`, `input_schema`
- `parameters` → `input_schema`

### 출력 형식

```json
{
  "id": "msg_abc123",
  "type": "message",
  "role": "assistant",
  "content": [
    {
      "type": "tool_use",
      "id": "toolu_abc123",
      "name": "get_weather",
      "input": {
        "location": "서울",
        "unit": "celsius"
      }
    }
  ],
  "stop_reason": "tool_use"
}
```

**OpenAI와의 차이**:
- `tool_calls` → `content` 배열 내 `type: "tool_use"` 객체
- `arguments` (JSON 문자열) → `input` (객체)
- `finish_reason` → `stop_reason`

### 함수 결과 반환

```json
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_abc123",
      "content": "{\"temp\": 15, \"condition\": \"맑음\"}"
    }
  ]
}
```

**OpenAI와의 차이**:
- `role: "tool"` → `role: "user"` + `type: "tool_result"`
- `tool_call_id` → `tool_use_id`

## 고급 패턴

### 패턴 1: 함수 설명 최적화

**나쁜 예**:
```json
{
  "name": "search",
  "description": "검색합니다"
}
```

**좋은 예**:
```json
{
  "name": "search_product_database",
  "description": "제품 데이터베이스에서 이름, 카테고리, 가격 범위로 제품을 검색합니다. 재고 여부 및 할인 정보도 포함됩니다. 사용자가 '찾아줘', '검색해줘' 등의 요청을 하거나 특정 제품에 대해 질문할 때 사용하세요."
}
```

**팁**:
- 함수가 **무엇을 하는지** 명확하게
- **언제 사용해야 하는지** 힌트 제공
- 파라미터 설명도 상세하게

### 패턴 2: 에러 처리

```python
def execute_function_call(function_name, arguments):
    try:
        # 함수 실행
        if function_name == "get_weather":
            result = get_weather(**arguments)
            return {"success": True, "data": result}

    except KeyError as e:
        return {"success": False, "error": f"Missing parameter: {e}"}

    except Exception as e:
        return {"success": False, "error": str(e)}

# 에러를 LLM에게 전달
messages.append({
    "role": "tool",
    "tool_call_id": tool_call_id,
    "content": json.dumps({
        "success": False,
        "error": "API rate limit exceeded. Please try again later."
    })
})

# LLM이 에러 메시지를 사용자 친화적으로 변환
# → "죄송합니다. 날씨 서비스가 일시적으로 사용 불가능합니다. 잠시 후 다시 시도해주세요."
```

### 패턴 3: 병렬 함수 호출

```python
# 하나의 응답에 여러 함수 호출 요청
{
  "tool_calls": [
    {
      "id": "call_1",
      "function": {"name": "get_weather", "arguments": "{\"location\": \"서울\"}"}
    },
    {
      "id": "call_2",
      "function": {"name": "get_weather", "arguments": "{\"location\": \"부산\"}"}
    },
    {
      "id": "call_3",
      "function": {"name": "get_weather", "arguments": "{\"location\": \"제주\"}"}
    }
  ]
}

# 병렬 실행으로 성능 최적화
import asyncio

async def execute_all_tools(tool_calls):
    tasks = [execute_tool_async(tc) for tc in tool_calls]
    results = await asyncio.gather(*tasks)
    return results
```

## 보안 고려사항

### 1. 함수 실행 전 검증

```python
# ❌ 위험: 무조건 실행
def execute_tool(name, args):
    return eval(f"{name}(**{args})")  # 절대 이렇게 하지 마세요!

# ✅ 안전: 화이트리스트 검증
ALLOWED_FUNCTIONS = {
    "get_weather": get_weather,
    "search_products": search_products
}

def execute_tool(name, args):
    if name not in ALLOWED_FUNCTIONS:
        raise ValueError(f"Function {name} not allowed")

    func = ALLOWED_FUNCTIONS[name]
    return func(**args)
```

### 2. 파라미터 검증

```python
def execute_tool(name, args):
    # 파라미터 타입 및 범위 검증
    if name == "query_database":
        query = args.get("query", "")
        # SQL Injection 방지
        if any(keyword in query.upper() for keyword in ["DROP", "DELETE", "UPDATE", "INSERT"]):
            return {"error": "Destructive queries not allowed"}

        # 읽기 전용 확인
        if not query.upper().startswith("SELECT"):
            return {"error": "Only SELECT queries allowed"}

    # 실행
    return ALLOWED_FUNCTIONS[name](**args)
```

### 3. 사용자 확인 (중요 작업)

```python
# 금융 거래 등 중요 작업은 사용자 확인 필요
if function_name in ["transfer_money", "delete_account", "make_purchase"]:
    # LLM 응답에 확인 메시지 포함
    return {
        "status": "pending_confirmation",
        "message": "이 작업을 실행하려면 사용자 확인이 필요합니다",
        "details": arguments
    }
```

## 다음 단계

Function Calling을 마스터했습니다! 다음은 **멀티모달 인터페이스**에서 이미지, 음성 등 다양한 입력 형식을 다루는 방법을 배워봅시다.

➡️ [5장: 멀티모달 인터페이스](./05-multimodal.md)

---

## 핵심 요약

- ✅ Function Calling은 LLM을 외부 도구와 연결하는 핵심 기능
- ✅ LLM이 직접 실행하는 것이 아니라 **호출 요청**만 함
- ✅ JSON Schema로 함수 파라미터 정의
- ✅ `finish_reason: "tool_calls"` 시 함수 호출 처리
- ✅ 함수 결과를 `role: "tool"` 메시지로 반환
- ✅ 명확한 함수 설명이 성능의 핵심
- ✅ 보안 검증 필수 (화이트리스트, 파라미터 검증)
- ✅ OpenAI와 Anthropic은 형식이 약간 다름
