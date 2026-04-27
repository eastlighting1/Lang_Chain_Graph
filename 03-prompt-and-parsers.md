# 3장. 프롬프트 템플릿 설계와 출력 파서(Output Parser)

2장에서는 문자열을 그대로 LLM에 던졌습니다. 하지만 실제 애플리케이션에서는 사용자 입력을 그대로 던지지 않고 **"너는 전문가야, 이런 형식으로 대답해"**라는 지시사항(System Prompt)과 함께 포장해서 보냅니다. 

이 챕터에서는 **프롬프트 템플릿(Prompt Template)**으로 질문을 예쁘게 포장하는 법과, 모델이 뱉어낸 `AIMessage`를 파이썬에서 쓰기 좋은 형태로 바꿔주는 **출력 파서(Output Parser)**를 다룹니다.

---

## 1. 프롬프트 템플릿 (ChatPromptTemplate)

채팅에 최적화된 LLM은 대화의 '역할(Role)'을 구분합니다.
- `System`: AI의 페르소나, 규칙, 제약조건을 설정 (절대적인 지시사항)
- `Human`: 실제 사용자가 입력하는 질문이나 텍스트
- `AI`: (참고용) AI가 이전에 대답했던 내용

템플릿을 사용하면 매번 `f-string`을 조작할 필요 없이, 구멍(변수)만 뚫어놓고 필요할 때 값을 쏙쏙 주입할 수 있습니다.

---

## 2. 순수 텍스트 추출: StrOutputParser 실습

2장에서 보았듯 LLM은 `response.content`처럼 `.content`를 꺼내줘야 하는 귀찮은 `AIMessage` 객체를 반환합니다. `StrOutputParser`는 이 껍데기를 벗기고 순수한 문자열(String)만 쏙 뽑아줍니다.

`02a_str_parser.py` 파일을 만들고 아래 코드를 작성해 보세요.

```python
# 02a_str_parser.py
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_community.chat_models import ChatOllama

llm = ChatOllama(model="glm-4.7-flash:q4_K_M", temperature=0)

# 1. 프롬프트 템플릿 정의 (역할 구분)
prompt = ChatPromptTemplate.from_messages([
    ("system", "너는 친절하고 전문적인 번역가야. 사용자의 입력을 {target_language}로 번역해줘. 번역 결과만 깔끔하게 출력해."),
    ("human", "{text}")
])

# 2. 템플릿에 변수 주입 (구멍 채우기)
# 타겟 언어를 '영어'로, 텍스트를 '만나서 반갑습니다.'로 채웁니다.
formatted = prompt.format_messages(target_language="영어", text="만나서 반갑습니다.")
print("── 렌더링된 프롬프트 ──")
print(formatted)
print()

# 3. 모델 호출
ai_message = llm.invoke(formatted)

print("── 파싱 전 (AIMessage 객체) ──")
print(type(ai_message))        
print(ai_message)  # 잡다한 메타데이터가 함께 출력됩니다.
print()

# 4. 파서를 통해 문자열 추출
parser = StrOutputParser()
result = parser.invoke(ai_message)

print("── 파싱 후 (순수 문자열) ──")
print(type(result))            
print(result)  # "Nice to meet you." 등 깔끔한 텍스트만 남습니다.
```

실행해 봅니다: `uv run 02a_str_parser.py`

---

## 3. 구조화된 데이터 추출: PydanticOutputParser 실습 (고급)

때로는 텍스트가 아니라, 데이터베이스에 저장하거나 후속 로직에 쓰기 위해 **Python 객체(Object)** 형태가 필요할 때가 있습니다. (예: `recipe.name`, `recipe.ingredients`)

`PydanticOutputParser`는 LLM에게 **"반드시 이 JSON 형식에 맞춰서 대답해!"**라는 가이드를 몰래 프롬프트에 끼워 넣고, 응답을 받아 실제 Python 인스턴스로 변환해 줍니다.

`02b_pydantic_parser.py` 파일을 만들어 봅시다.

```python
# 02b_pydantic_parser.py
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import PydanticOutputParser
from langchain_community.chat_models import ChatOllama
from pydantic import BaseModel, Field

llm = ChatOllama(model="glm-4.7-flash:q4_K_M", temperature=0)

# ── 1. 우리가 원하는 데이터 구조(스키마)를 정의합니다.
class Recipe(BaseModel):
    dish_name: str = Field(description="요리의 이름")
    ingredients: list[str] = Field(description="필요한 재료 목록 (리스트 형태)")
    time_minutes: int = Field(description="예상 조리 시간 (분 단위 숫자)")

# ── 2. 파서를 초기화하고, LLM에게 줄 포맷 지시사항을 가져옵니다.
parser = PydanticOutputParser(pydantic_object=Recipe)
format_instructions = parser.get_format_instructions()

print("── LLM에게 몰래 전달되는 포맷 지시문 ──")
print(format_instructions)    # JSON 스키마 구조가 문자열로 적혀 있습니다.
print("-" * 50 + "\n")

# ── 3. 프롬프트에 포맷 지시사항({format_instructions})을 주입합니다.
prompt = ChatPromptTemplate.from_messages([
    ("system", "너는 최고의 요리사야. 사용자가 요청한 요리의 레시피를 분석해.\n\n{format_instructions}"),
    ("human", "{dish} 레시피를 알려줘.")
])

formatted = prompt.format_messages(
    format_instructions=format_instructions,
    dish="김치볶음밥"
)

# ── 4. 모델 실행 및 결과 파싱
raw_response = llm.invoke(formatted)

print("── LLM의 원본 응답 (문자열 JSON) ──")
print(raw_response.content)
print("-" * 50 + "\n")

# 파서가 원본 응답(JSON 문자열)을 읽어서 Recipe 객체로 변환합니다.
recipe = parser.invoke(raw_response)

print("── 최종 Pydantic 파싱 결과 ──")
print(f"타입   : {type(recipe)}")        # <class '__main__.Recipe'>
print(f"요리명 : {recipe.dish_name}")
print(f"재료   : {recipe.ingredients}")
print(f"시간   : {recipe.time_minutes}분")
```

실행해 봅니다: `uv run 02b_pydantic_parser.py`

> 💡 **트러블슈팅 팁**: Pydantic 파싱 중 JSON Decode 에러가 난다면, LLM이 JSON 형식을 지키지 않고 쓸데없는 말("네 알겠습니다. 레시피입니다..." 등)을 덧붙였기 때문입니다. `temperature=0`으로 설정하고, 프롬프트 지시문을 엄격하게 제어하는 프롬프트 엔지니어링이 필요합니다.

---
### 🎯 3장 요약
- `ChatPromptTemplate`으로 System과 Human의 역할을 나누어 안전하게 변수를 주입합니다.
- `StrOutputParser`는 흔히 쓰이는 범용 텍스트 변환기입니다.
- `PydanticOutputParser`를 쓰면 마법처럼 비정형 텍스트 응답을 완벽히 구조화된 Python 객체로 뽑아낼 수 있습니다.

[⬅️ 이전 단계: 2장. LangChain 핵심 아키텍처 이해 및 LLM 호출 실습](02-langchain-llm-calls.md) | [🏠 메인 메뉴](README.md) | [➡️ 다음 단계: 4장. LCEL과 Runnable 인터페이스](04-lcel-basics.md)
