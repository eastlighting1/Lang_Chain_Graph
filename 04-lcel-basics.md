# 4장. LCEL(LangChain Expression Language)과 Runnable 인터페이스

3장까지 우리는 템플릿 렌더링 -> 모델 호출 -> 파서 호출의 과정을 각 변수에 담아 절차적으로 코딩했습니다. 하지만 실제 AI 애플리케이션은 훨씬 길고 복잡한 워크플로우를 갖습니다.

**"이 모든 과정을 하나의 조립식 레고 블록처럼 이어붙일 수 있다면 어떨까?"**
이 철학에서 탄생한 것이 LangChain의 심장, **LCEL(LangChain Expression Language)** 입니다.

---

## 1. Runnable 인터페이스란? (이론 딥다이브)

LangChain을 구성하는 거의 모든 주요 클래스(Prompt, LLM, Parser, Memory 등)는 내부적으로 **`Runnable`** 이라는 추상 클래스를 상속받아 구현되어 있습니다.

파이썬의 관점에서 `Runnable` 규약을 따른다는 것은, 객체 내부 로직이 달라도 겉보기에 다음 메서드들을 반드시 구현하고 있다는 뜻입니다.
- `invoke()`: 단일 동기 실행
- `ainvoke()`: 비동기 실행 (Async/Await)
- `stream()` / `astream()`: 스트리밍 실행
- `batch()` / `abatch()`: 병렬 일괄 실행

> 💡 **LCEL의 진짜 존재 이유**: 단순히 코드를 짧게 줄여주기 위함이 아닙니다. 모든 컴포넌트가 `Runnable`을 상속받기 때문에, 개발자가 복잡한 스레드 처리나 비동기 코드를 짜지 않아도 체인 통째로 비동기 실행(`ainvoke`)이나 실시간 스트리밍(`stream`)을 무료로 누릴 수 있다는 것이 핵심입니다!

---

## 2. 파이프 연산자 `|` 의 비밀 (Under the Hood)

LCEL은 다음과 같이 매우 직관적인 문법을 자랑합니다.

```python
# LCEL 파이프라인 조립
chain = prompt | llm | parser
```

이 코드를 본 파이썬 숙련자라면 한 가지 의문이 들 것입니다. *"파이썬에서 `|` 기호는 비트 연산자(Bitwise OR)이거나 집합(Set)의 합집합 연산자인데, 객체끼리 이걸 어떻게 쓴 거지?"*

> 💡 **Under the Hood**: LangChain의 `Runnable` 클래스 내부에는 파이썬 매직 메서드인 **`__or__`**가 오버라이딩 되어 있습니다. 
`A | B` 라는 코드가 실행되면, 내부적으로는 `A.__or__(B)`가 호출되어 앞쪽 객체의 출력(Output)을 뒤쪽 객체의 입력(Input)으로 자동 연결하는 새로운 `RunnableSequence` 객체를 만들어 반환하는 것입니다. 객체 지향 프로그래밍의 극치를 보여주는 멋진 설계입니다.

---

## 3. LCEL 체인 조립 실습 (코드 분할 해설)

위의 마법을 코드로 직접 짜보며 체감해 보겠습니다.
`03_lcel_basics.py` 파일을 생성합니다.

### [Step 1] 부품(Component) 준비하기
```python
# 03_lcel_basics.py
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_community.chat_models import ChatOllama

llm = ChatOllama(model="glm-4.7-flash:q4_K_M", temperature=0)

# 프롬프트 준비 (입력으로 "topic" 이라는 딕셔너리 키를 기대합니다)
prompt = ChatPromptTemplate.from_messages([
    ("system", "너는 친절한 IT 전문가야. {topic}에 대해 초보자도 이해하기 쉽게 3줄로 요약 설명해줘."),
    ("human", "설명 시작!")
])

# 파서 준비
parser = StrOutputParser()
```

### [Step 2] 체인 조립 및 타입 흐름(Type Flow) 시각화
```python
# 부품들을 조립하여 하나의 거대한 덩어리(RunnableSequence)를 만듭니다.
chain = prompt | llm | parser

print("💡 조립된 체인 객체의 타입:", type(chain))
# 출력: <class 'langchain_core.runnables.base.RunnableSequence'>
```

조립할 때 머릿속에는 반드시 다음과 같은 **입출력 타입 흐름(Type Flow)**이 그려져야 합니다. LCEL 에러의 90%는 여기서 타입이 어긋나서 발생합니다.

1. **`{"topic": "..."}`** (dict 타입 입력) 
   → *`prompt`가 이 dict를 받아 구멍을 채움*
2. **`PromptValue`** (프롬프트 내부 포맷 객체 출력) 
   → *`llm`이 이를 읽어서 답변을 생성함*
3. **`AIMessage`** (메타데이터 포함 답변 객체 출력) 
   → *`parser`가 이를 읽어서 텍스트만 추출함*
4. **`"..."`** (최종 str 타입 출력)

### [Step 3] 체인 실행하기 (invoke / stream)
```python
# ── 체인 통째로 invoke 실행 ──
# 사용자는 복잡한 중간 단계를 신경 쓸 필요 없이, 최초 입력(dict)만 주면 
# 최종 출력(str)을 얻을 수 있습니다.
print("── [1] 체인 invoke ──")
result = chain.invoke({"topic": "클라우드 컴퓨팅"})
print(result)

# ── 체인 통째로 stream 실행 ──
# 내부 LLM이 뱉어내는 청크(chunk)들이 자동으로 파서를 거쳐 실시간으로 튀어나옵니다!
print("\n── [2] 체인 stream ──")
for chunk in chain.stream({"topic": "블록체인"}):
    print(chunk, end="", flush=True)
print("\n")
```

터미널에서 실행해 봅니다: `uv run 03_lcel_basics.py`

---

## 4. 🏋️‍♂️ Your Turn! (도전 과제)

지금까지 배운 LCEL 체인을 변형해 보세요.

1. **파서 제거해보기**: `chain = prompt | llm` 으로 파서를 빼고 조립해 보세요. 그리고 `chain.invoke({"topic": "AI"})`를 실행했을 때 최종 출력의 타입이 무엇으로 바뀌는지 확인해 보세요. (힌트: `AIMessage`가 튀어나와야 정상입니다.)
2. **Batch 모드 추가해보기**: 위 코드의 가장 아래쪽에 `chain.batch([{"topic": "양자역학"}, {"topic": "상대성이론"}])` 을 추가하고, 리스트 안의 결과가 어떻게 병렬로 튀어나오는지 프린트해 보세요.

[⬅️ 이전 단계: 3장. 프롬프트 템플릿과 출력 파서](03-prompt-and-parsers.md) | [🏠 메인 메뉴](README.md) | [➡️ 다음 단계: 5장. 미니 프로젝트 - 요약 및 번역](05-mini-project.md)
