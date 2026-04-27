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
   - 예: "너는 친절한 어시스턴트야. 다음 질문에 답해: {사용자 입력}"
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

이 세 가지 컴포넌트를 조립해 나가는 것이 LangChain 프로그래밍의 90%입니다.

---

## 2. LLM 호출 3가지 방식 실습 (코드)

우선 프롬프트나 파서 없이, 가장 핵심인 **ChatModel**을 쌩으로 호출하는 방법을 알아보겠습니다.
`01_llm_basics.py` 파일을 생성하고 아래 코드를 직접 타이핑해 보세요.

```python
# 01_llm_basics.py
from langchain_community.chat_models import ChatOllama

# 우리가 구동시켜둔 로컬 Ollama 모델과 연결합니다.
# temperature는 0에 가까울수록 진지하고 일관된 답변을, 1에 가까울수록 창의적이고 랜덤한 답변을 냅니다.
llm = ChatOllama(model="glm-4.7-flash:q4_K_M", temperature=0)

# ── [1] invoke (동기식 호출) ──────────────────────────────────────────
# 응답 전체가 완성될 때까지 멈춰서 기다렸다가 한 번에 반환합니다.
# 유즈케이스: 짧은 응답을 원할 때, 데이터 전처리를 위해 백그라운드에서 실행할 때
print("── [1] invoke 방식 ──")
response = llm.invoke("랭체인이 뭐야? 한 문장으로 대답해줘.")

# 💡 주의: response 자체는 텍스트가 아니라 AIMessage 객체입니다.
print("타입:", type(response))
# 실제 텍스트 내용만 보려면 .content 속성에 접근해야 합니다.
print("내용:", response.content)
print("\n")


# ── [2] stream (스트리밍 호출) ─────────────────────────────────────────
# 답변 토큰이 생성될 때마다 실시간으로 조각(chunk)을 반환합니다.
# 유즈케이스: ChatGPT처럼 사용자에게 글자가 타자 쳐지듯 보여줘야 하는 웹/앱 UI
print("── [2] stream 방식 ──")
for chunk in llm.stream("우주에 대해 3문장으로 설명해줘."):
    # flush=True를 주어야 버퍼에 쌓이지 않고 즉시 화면에 출력됩니다.
    print(chunk.content, end="", flush=True)
print("\n\n")


# ── [3] batch (병렬 일괄 호출) ─────────────────────────────────────────
# 여러 개의 프롬프트를 리스트로 전달하면, 순차적으로 처리하지 않고 병렬로 동시에 처리합니다.
# 유즈케이스: 엑셀 파일 수백 줄을 읽어서 한 번에 요약해야 할 때 시간 단축
print("── [3] batch 방식 ──")
prompts = ["사과는 무슨 색이야?", "바나나는 무슨 색이야?", "포도는 무슨 색이야?"]
responses = llm.batch(prompts)

# responses는 AIMessage들의 리스트가 됩니다.
for i, res in enumerate(responses):
    print(f"Q{i+1}: {prompts[i]}")
    print(f"A{i+1}: {res.content}\n")
```

### 💻 실행 및 결과 확인

터미널에서 아래 명령어로 스크립트를 실행해 보세요.
```bash
uv run 01_llm_basics.py
```

- **invoke**는 1~2초 정지해 있다가 답변이 툭 튀어나오고,
- **stream**은 타자기를 치듯 한 글자씩 출력되는 것을 눈으로 확연히 구분할 수 있습니다.
- **batch**는 내부 모델 엔진의 병렬 처리 능력에 따라 개별적으로 여러 번 invoke하는 것보다 훨씬 빠른 속도로 리스트 전체에 대한 답을 내놓습니다.

---
### 🎯 2장 요약
- LangChain은 `Prompt → LLM → Parser`의 3단계 조립 블록입니다.
- 모델 호출 시 상황에 맞게 기다리는 `invoke`, 타자치듯 보여주는 `stream`, 한 번에 쏟아붓는 `batch`를 선택해 사용할 수 있습니다.

[⬅️ 이전 단계: 1장. 환경 설정 및 로컬 LLM 준비](01-setup-and-local-llm.md) | [🏠 메인 메뉴](README.md) | [➡️ 다음 단계: 3장. 프롬프트 템플릿과 출력 파서](03-prompt-and-parsers.md)
