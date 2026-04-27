# 2장. LangChain 핵심 아키텍처 이해 및 LLM 호출 실습

환경 설정이 완료되었으니, 이제 본격적으로 코드를 작성해 볼 차례입니다. 그 전에 LangChain이 어떻게 생겼는지 아주 짧게 구조를 이해하고 넘어가겠습니다.

---

## 1. LangChain의 핵심 레이어 구조 (이론)

LangChain은 복잡한 AI 애플리케이션을 블록 조립하듯 쉽게 만들 수 있게 해주는 프레임워크입니다. 이를 위해 전체 과정을 3개의 큰 레이어로 나누어 관리합니다.

```text
[사용자 입력] "안녕! 넌 누구야?"
    │
    ▼
1. [Prompt Template]
   - 역할: 사용자의 입력을 받아서, LLM이 이해하기 좋은 완벽한 프롬프트(질문지)로 가공합니다.
    │
    ▼
2. [LLM / ChatModel] (예: ChatOllama)
   - 역할: 완성된 질문지를 받아 실제 AI 두뇌를 거쳐 답변을 생성합니다.
   - 반환값: 단순 문자열이 아니라, 메타데이터가 포함된 `AIMessage` 객체를 반환합니다.
    │
    ▼
3. [Output Parser]
   - 역할: 모델이 뱉어낸 `AIMessage` 객체에서 우리가 원하는 형태(순수 텍스트, 혹은 JSON 등)만 깔끔하게 추출합니다.
    │
    ▼
[최종 결과] "안녕하세요! 저는 AI 어시스턴트입니다."
```

---

## 2. LLM과 ChatModel의 차이, 그리고 Message 객체

과거의 언어 모델(LLM)은 단순히 텍스트를 이어붙이는 역할(Text Completion)만 했습니다. 하지만 요즘 우리가 쓰는 ChatGPT나 Claude 같은 모델들은 **대화(Chat)에 특화**되어 있습니다. 
따라서 LangChain에서는 이를 `ChatModel`이라고 부르며, 대화의 역할을 명확히 구분하기 위해 **Message 객체**를 사용합니다.

- `SystemMessage`: AI의 페르소나, 규칙, 제약조건을 설정 (절대적인 지시사항)
- `HumanMessage`: 실제 사용자가 입력하는 질문이나 텍스트
- `AIMessage`: AI가 생성한 응답 텍스트와 메타데이터(사용한 토큰 수 등)를 담는 객체

> 💡 **Under the Hood**: 단순 문자열("안녕")을 던져도, LangChain 내부에서는 이를 자동으로 `HumanMessage(content="안녕")` 객체로 변환하여 모델에 전달합니다. 규격화된 객체를 사용해야만 나중에 메모리(대화 기록)를 관리하거나 파서를 거칠 때 일관된 처리가 가능하기 때문입니다.

---

## 3. LLM 호출 3가지 방식 실습 (코드 딥다이브)

가장 핵심인 **ChatModel**을 직접 호출하는 코드를 단계별로 해설합니다.
`01_llm_basics.py` 파일을 생성하고 아래 코드를 하나씩 뜯어보세요.

### [Step 1] 모델 초기화와 파라미터 제어
```python
# 01_llm_basics.py
from langchain_community.chat_models import ChatOllama

# ChatOllama 객체를 생성하며 모델을 연결합니다.
llm = ChatOllama(model="glm-4.7-flash:q4_K_M", temperature=0)
```
- **`temperature`의 수학적 원리**: LLM은 다음에 올 단어의 확률 분포를 계산하여 답변을 만듭니다. `temperature`는 이 확률 분포를 조작하는 변수입니다.
  - `0`에 가까울수록: 가장 확률이 높은 단어만 무조건 선택합니다. (진지하고 일관된 답변, 정보 추출에 적합)
  - `1`에 가까울수록: 확률 분포를 평탄하게 만들어 가끔 엉뚱한 단어도 선택하게 합니다. (창의적인 소설 작성 등에 적합)

### [Step 2] 단일 호출 (invoke)
```python
print("── [1] invoke 방식 ──")
response = llm.invoke("랭체인이 뭐야? 한 문장으로 대답해줘.")

# response 자체는 텍스트가 아니라 AIMessage 객체입니다.
print("객체 타입:", type(response))
print("실제 내용:", response.content)
```
- `invoke()`는 모델이 답변을 **끝까지 다 만들 때까지 프로그램이 멈춰서 기다리는(Blocking)** 동기식 메서드입니다. 데이터 전처리 등 결과를 한 번에 받아야 할 때 주로 사용합니다.

### [Step 3] 스트리밍 호출 (stream)
```python
print("\n── [2] stream 방식 ──")
# 답변 토큰이 생성될 때마다 실시간으로 조각(chunk)을 반환합니다.
for chunk in llm.stream("우주에 대해 3문장으로 설명해줘."):
    # flush=True를 주어야 버퍼에 쌓이지 않고 콘솔 화면에 즉시 출력됩니다.
    print(chunk.content, end="", flush=True)
print("\n")
```
- ChatGPT 웹사이트를 보면 글자가 타자 쳐지듯 나옵니다. 기다리는 시간을 줄여 체감 응답 속도를 높이기 위해 `stream()`을 사용합니다. 생성되는 즉시 작은 덩어리(chunk)들이 계속해서 튀어나옵니다.

### [Step 4] 병렬 일괄 호출 (batch)
```python
print("\n── [3] batch 방식 ──")
prompts = ["사과는 무슨 색?", "바나나는 무슨 색?", "포도는 무슨 색?"]

# for문으로 여러 번 invoke() 하는 것보다 압도적으로 빠릅니다.
responses = llm.batch(prompts)

for i, res in enumerate(responses):
    print(f"Q: {prompts[i]} -> A: {res.content}")
```
- **Under the Hood**: `batch()`를 호출하면 내부적으로 스레드 풀(Thread Pool)이나 비동기 작업 큐를 생성하여 여러 프롬프트를 **동시에(Parallel)** 처리합니다. 엑셀 파일 수천 줄을 번역할 때 필수적인 기능입니다.

---

## 4. 🏋️‍♂️ Your Turn! (도전 과제)

지금까지 배운 내용을 바탕으로 `01_llm_basics.py` 코드를 직접 수정해 보세요.

1. **온도 조절 실험**: `temperature` 값을 `0`과 `0.9`로 각각 설정한 뒤 동일한 프롬프트("외계인이 지구에 온다면 첫 마디가 뭘까?")를 `invoke` 해보고 답변의 창의성 차이를 비교해 보세요.
2. **Message 객체 직접 다뤄보기**: 문자열을 던지는 대신, 아래처럼 명시적으로 메시지 리스트를 만들어서 `invoke`에 넣어보세요. 동작이 같나요?
   ```python
   from langchain_core.messages import HumanMessage, SystemMessage
   
   messages = [
       SystemMessage(content="너는 사투리를 쓰는 친근한 할아버지야."),
       HumanMessage(content="요즘 날씨가 너무 덥네요.")
   ]
   response = llm.invoke(messages)
   print(response.content)
   ```

[⬅️ 이전 단계: 1장. 환경 설정 및 로컬 LLM 준비](01-setup-and-local-llm.md) | [🏠 메인 메뉴](README.md) | [➡️ 다음 단계: 3장. 프롬프트 템플릿과 출력 파서](03-prompt-and-parsers.md)
