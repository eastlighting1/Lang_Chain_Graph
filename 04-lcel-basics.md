# 4장. LCEL(LangChain Expression Language)과 Runnable 인터페이스

3장까지는 프롬프트를 만들고 -> 모델을 부르고 -> 파서를 부르는 과정을 변수에 일일이 담아 수동으로 연결했습니다. 코드가 길어지고 귀찮습니다. 

LangChain의 진정한 강력함은 파이프라인을 레고 블록 조립하듯 한 줄로 이어버리는 **LCEL**에서 나옵니다. 이번 챕터에서는 LCEL의 마법을 파헤쳐 봅니다.

---

## 1. Runnable 인터페이스란? (이론)

LangChain을 구성하는 거의 모든 주요 부품(Prompt, LLM, Parser, Memory 등)은 내부적으로 **`Runnable`** 이라는 공통 인터페이스(클래스 구조)를 상속받아 구현하고 있습니다.

공통 규격을 맞췄기 때문에, 내부 로직이 달라도 겉보기에는 모두 동일한 기능(메서드)들을 지원합니다.
- `invoke()`: 단일 입력에 대한 동기적 실행
- `stream()`: 단일 입력에 대한 스트리밍 실행
- `batch()`: 여러 입력 리스트에 대한 병렬 실행
- **`pipe()` 또는 `|` (파이프 연산자)**: 현재 부품의 출력을 다음 부품의 입력으로 연결

---

## 2. 마법의 파이프 연산자 `|`

리눅스 쉘(Shell)을 써보셨다면 `ls -al | grep txt` 처럼 앞 명령어의 결과를 뒤 명령어의 입력으로 넘기는 파이프(`|`) 기호에 익숙하실 겁니다. LCEL도 완전히 똑같은 철학을 파이썬에 가져왔습니다.

```python
# ⛔ [3장 방식] 수동으로 한 땀 한 땀 이어붙이기
formatted_prompt = prompt.format_messages(topic="인공지능")
ai_message = llm.invoke(formatted_prompt)
result = parser.invoke(ai_message)

# ✅ [4장 방식] LCEL 파이프라인으로 한 줄 조립
chain = prompt | llm | parser
result = chain.invoke({"topic": "인공지능"})
```

이렇게 `|`로 연결되어 탄생한 `chain` 객체 전체도 결국 거대한 하나의 `Runnable` 덩어리가 됩니다. 따라서 체인 통째로 `invoke`, `stream`, `batch`를 실행할 수 있습니다.

---

## 3. LCEL 체인 조립 실습 (코드)

위의 마법을 코드로 직접 짜보며 체감해 보겠습니다.
`03_lcel_basics.py` 파일을 생성합니다.

```python
# 03_lcel_basics.py
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_community.chat_models import ChatOllama

llm = ChatOllama(model="glm-4.7-flash:q4_K_M", temperature=0)

# ── 1. 체인 컴포넌트 준비 ───────────────────────────────────────────────
prompt = ChatPromptTemplate.from_messages([
    ("system", "너는 친절한 IT 전문가야. {topic}에 대해 초보자도 이해하기 쉽게 3줄로 요약 설명해줘."),
    ("human", "설명 시작!")
])

parser = StrOutputParser()

# ── 2. 체인 조립 (LCEL) ─────────────────────────────────────────────────
# 딕셔너리 -> 프롬프트 -> AIMessage -> 문자열(str) 로 흐르는 완벽한 파이프라인입니다.
chain = prompt | llm | parser

print("💡 조립된 체인 객체의 타입:", type(chain))
print("-" * 50 + "\n")

# ── 3. 체인 통째로 invoke 실행 ──────────────────────────────────────────
print("── [1] 체인 통째로 invoke ──")
result = chain.invoke({"topic": "클라우드 컴퓨팅"})
print(result)
print("\n")

# ── 4. 체인 통째로 stream 실행 ──────────────────────────────────────────
# 조립된 체인에 stream을 호출하면, 내부의 LLM이 내뱉는 chunk가 파서까지 거쳐서 실시간으로 튀어나옵니다.
print("── [2] 체인 통째로 stream ──")
for chunk in chain.stream({"topic": "블록체인"}):
    print(chunk, end="", flush=True)
print("\n\n")

# ── 5. 체인 통째로 batch 실행 ───────────────────────────────────────────
print("── [3] 체인 통째로 batch ──")
topics = [{"topic": "양자 컴퓨팅"}, {"topic": "메타버스"}]
results = chain.batch(topics)

for i, res in enumerate(results):
    print(f"[{topics[i]['topic']}]\n{res}\n")
```

터미널에서 실행해 봅니다: `uv run 03_lcel_basics.py`

### 💡 입출력 타입 흐름(Type Flow) 머릿속에 그리기
LCEL을 쓸 때 가장 많이 겪는 에러는 **"앞 블록의 출력 타입과 뒤 블록의 입력 타입이 안 맞을 때"** 발생합니다. 조립할 때 항상 이 흐름을 검증해야 합니다.

1. **입력 `{"topic": "..."}`** (dict 타입) 
   → *`prompt`가 이 dict를 받아 구멍을 채움*
2. **출력/입력 `PromptValue`** (완성된 텍스트 묶음 객체) 
   → *`llm`이 이를 읽어서 답변을 생성함*
3. **출력/입력 `AIMessage`** (메타데이터 포함 답변 객체) 
   → *`parser`가 이를 읽어서 텍스트만 추출함*
4. **최종 출력 `"..."`** (str 타입)

앞뒤가 딱딱 맞는 퍼즐 조각처럼 연결된 것을 볼 수 있습니다.

---
### 🎯 4장 요약
- `|` 연산자 하나로 Prompt, LLM, Parser를 이어붙여 직관적이고 강력한 `chain`을 생성할 수 있습니다.
- 만들어진 `chain`은 그 자체로 거대한 `Runnable`이 되어 `invoke/stream/batch` 호출이 가능해집니다.
- 퍼즐 조각이 맞물리듯 **데이터 타입이 파이프라인을 타고 어떻게 변환되는지 (Type Flow)** 추적하는 것이 에러 없는 LCEL 작성의 핵심입니다.

[⬅️ 이전 단계: 3장. 프롬프트 템플릿과 출력 파서](03-prompt-and-parsers.md) | [🏠 메인 메뉴](README.md) | [➡️ 다음 단계: 5장. 미니 파이프라인 프로젝트](05-mini-project.md)
