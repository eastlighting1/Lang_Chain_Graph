# 3장. 프롬프트 템플릿 설계와 출력 파서(Output Parser)

2장에서 메시지 객체의 중요성을 배웠습니다. 하지만 매번 `SystemMessage(content="...")`를 수동으로 만들고 문자열을 조작(`f-string`)하는 것은 코드 유지보수에 최악입니다.

이 챕터에서는 **프롬프트 템플릿(Prompt Template)**으로 질문을 캡슐화하는 법과, 모델이 뱉어낸 `AIMessage`를 파이썬에서 쓰기 좋은 형태로 구조화하는 **출력 파서(Output Parser)**의 놀라운 내부 원리를 파헤칩니다.

---

## 1. 프롬프트 템플릿 (ChatPromptTemplate)

파이썬의 `f-string`만으로도 변수 주입이 가능한데, 왜 굳이 `ChatPromptTemplate`을 쓸까요?
- **메시지 역할(Role)의 명확한 분리**: System, Human, AI의 대화 구조를 리스트 형태로 우아하게 관리할 수 있습니다.
- **통합(Integration) 용이성**: LangChain의 다양한 출력 파서 및 모델과 완벽하게 호환되며, 프롬프트를 파일로 저장하거나 관리(Prompt Management)하기 쉬워집니다.

---

## 2. 순수 텍스트 추출: StrOutputParser 딥다이브

LLM은 `response.content`처럼 껍데기가 있는 `AIMessage` 객체를 반환합니다. `StrOutputParser`는 이 껍데기를 벗기고 순수한 문자열(String)만 쏙 뽑아줍니다.

`02a_str_parser.py` 파일을 만들어 코드를 단계별로 살펴보겠습니다.

```python
# 02a_str_parser.py
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_community.chat_models import ChatOllama

llm = ChatOllama(model="glm-4.7-flash:q4_K_M", temperature=0)

# [1] 템플릿 정의: 중괄호 {} 를 사용하여 동적 변수가 들어갈 자리를 만듭니다.
prompt = ChatPromptTemplate.from_messages([
    ("system", "너는 친절하고 전문적인 번역가야. 사용자의 입력을 {target_language}로 번역해."),
    ("human", "{text}")
])

# [2] 템플릿 렌더링 (변수 주입)
# .format_messages()를 호출하면, 변수가 채워진 BaseMessage 객체 리스트가 반환됩니다.
formatted = prompt.format_messages(target_language="영어", text="만나서 반갑습니다.")

# [3] 모델 호출
ai_message = llm.invoke(formatted)

# [4] 파서를 통해 순수 문자열 추출
parser = StrOutputParser()
result = parser.invoke(ai_message)

print(result) # "Nice to meet you."
```

> 💡 **동작 원리**: `StrOutputParser`의 내부는 단순히 입력받은 객체가 `AIMessage`인지 확인하고, 맞다면 `.content` 속성을 반환하도록 오버라이딩된 아주 작은 클래스입니다.

---

## 3. 구조화된 데이터 추출: PydanticOutputParser의 마법 (핵심)

때로는 데이터베이스에 저장하거나 후속 로직에 쓰기 위해 텍스트가 아닌 **Python 객체(Object)** 형태가 필요합니다. (예: `recipe.dish_name`, `recipe.ingredients`)

어떻게 자연어 모델이 파이썬 코드를 알아듣고 정확한 형태의 JSON을 내뱉을까요? `PydanticOutputParser`의 내부 로직을 뜯어보겠습니다.

`02b_pydantic_parser.py` 파일을 생성하세요.

### [Step 1] Pydantic으로 스키마 정의하기
```python
# 02b_pydantic_parser.py
from pydantic import BaseModel, Field

class Recipe(BaseModel):
    dish_name: str = Field(description="요리의 이름")
    ingredients: list[str] = Field(description="필요한 재료 목록 (리스트 형태)")
    time_minutes: int = Field(description="예상 조리 시간 (분 단위 숫자)")
```
- 파이썬의 타입 힌팅 기반 데이터 검증 라이브러리인 Pydantic을 사용해 우리가 원하는 완벽한 형태의 틀을 만듭니다. `Field`의 `description`은 나중에 LLM에게 가이드로 제공되므로 자세히 적을수록 좋습니다.

### [Step 2] 몰래 숨겨둔 지시문(Format Instructions) 확인하기
```python
from langchain_core.output_parsers import PydanticOutputParser

parser = PydanticOutputParser(pydantic_object=Recipe)

# 파서가 Pydantic 모델을 분석해 생성한 "가이드라인"을 뽑아봅니다.
format_instructions = parser.get_format_instructions()
print("── LLM에게 강제할 지시문 ──\n", format_instructions)
```
> 💡 **Under the Hood**: 이것이 핵심입니다! `PydanticOutputParser`는 우리가 만든 파이썬 클래스를 읽어서, **"반드시 이 JSON 스키마를 지켜서 출력해라. 다른 말은 절대 하지 마라."**라는 장문의 프롬프트 문자열을 동적으로 생성해 냅니다. 우리는 이 문자열을 시스템 프롬프트에 살짝 끼워 넣기만 하면 됩니다. (이를 프롬프트 엔지니어링 용어로 *Schema Enforcement* 라고 합니다.)

### [Step 3] 프롬프트에 지시문 주입 후 실행
```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_community.chat_models import ChatOllama

llm = ChatOllama(model="glm-4.7-flash:q4_K_M", temperature=0)

# 프롬프트 어딘가에 {format_instructions} 변수를 뚫어 놓습니다.
prompt = ChatPromptTemplate.from_messages([
    ("system", "너는 최고의 요리사야. 요청한 요리의 레시피를 분석해.\n\n{format_instructions}"),
    ("human", "{dish} 레시피를 알려줘.")
])

# 구멍 채우기
formatted = prompt.format_messages(
    format_instructions=format_instructions,
    dish="김치볶음밥"
)

# LLM 호출 -> 원본 문자열 JSON 반환
raw_response = llm.invoke(formatted)

# 파서를 거쳐 완벽한 파이썬 객체로 변환!
recipe = parser.invoke(raw_response)

print("\n── 최종 파이썬 객체 ──")
print(f"요리명: {recipe.dish_name}")
print(f"재료: {recipe.ingredients}")
```

터미널에서 실행해 봅니다: `uv run 02b_pydantic_parser.py`

---

## 4. 🏋️‍♂️ Your Turn! (도전 과제)

지금까지 배운 파서를 응용해 봅시다.

1. **새로운 데이터 구조 만들기**: `02b_pydantic_parser.py`를 수정하여, 요리 레시피가 아니라 **"영화 정보(MovieInfo)"**를 추출해 보세요. 필드는 `title(문자열)`, `release_year(정수)`, `genres(리스트)`, `is_recommended(불리언)`를 포함해야 합니다.
2. **에러 유도해 보기**: 모델의 `temperature`를 `1.0`으로 올리고 프롬프트에서 `{format_instructions}`를 지운 채로 실행해 보세요. 파서가 어떻게 에러(ValidationError)를 뱉어내는지 확인하는 것도 훌륭한 디버깅 공부가 됩니다.

[⬅️ 이전 단계: 2장. LangChain 핵심 아키텍처 이해](02-langchain-llm-calls.md) | [🏠 메인 메뉴](README.md) | [➡️ 다음 단계: 4장. LCEL과 Runnable 인터페이스](04-lcel-basics.md)
