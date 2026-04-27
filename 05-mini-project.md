# 5장. 미니 프로젝트: 텍스트 요약 및 번역 파이프라인 구축

이제 기본기를 모두 배웠으니, 두 개 이상의 별개 체인을 결합하여 **"긴 영문 글을 읽고 핵심을 요약한 뒤 한국어로 번역하는"** 다중 파이프라인을 구축해 보겠습니다.

이 챕터의 핵심은 파이프라인 중간에서 **데이터를 원하는 형태로 변환하거나 라우팅하는 방법(Data Routing)**을 깊이 있게 이해하는 것입니다.

---

## 1. 다중 체인 조립의 병목: 타입 불일치 (Type Mismatch)

앞서 만든 체인(`prompt | llm | parser`)의 최종 출력은 **순수 문자열(`str`)**입니다.
그런데 이 출력을 다음 체인의 입력으로 곧바로 연결하면 어떻게 될까요?

```python
# ⛔ 에러 발생 코드 (개념 설명용)
summary_chain = summary_prompt | llm | parser   # 출력: str ("The core concept is...")
translate_chain = translate_prompt | llm | parser # 기대하는 입력: {"summary_text": "..."} 형태의 dict

# summary_chain의 출력(str)이 translate_chain의 입력(dict)과 맞지 않아 폭발합니다!
final_chain = summary_chain | translate_chain 
```

`translate_prompt`는 `{summary_text}`라는 변수를 채우기 위해 딕셔너리(`dict`) 입력을 기다리고 있는데, 앞 체인에서 냅다 문자열(`str`)을 던지니 파이프라인이 터지는 것입니다.

---

## 2. 해결책: 데이터 라우팅과 람다(`lambda`) 변환

이 문제를 해결하기 위해, 체인과 체인 사이에 문자열을 딕셔너리로 예쁘게 포장해 주는 데이터 변환기를 끼워 넣어야 합니다.

LangChain에서는 이를 위해 `RunnablePassthrough`나 `RunnableParallel` 같은 전용 도구를 제공하지만, 가장 직관적이고 널리 쓰이는 방법은 **파이썬의 익명 함수인 `lambda`를 사용하는 것**입니다.

```python
# 문자열(summary)을 받아서, {"summary_text": summary} 형태의 딕셔너리로 반환합니다.
lambda summary: {"summary_text": summary}
```

LCEL 파이프라인 내부에 딕셔너리 `{}` 나 람다 함수 `lambda` 가 등장하면, LangChain은 내부적으로 이를 눈치채고 데이터를 변환하는 `Runnable` 객체로 자동 캐스팅(Casting)해 줍니다. 이것도 파이프 연산자 오버로딩에 숨겨진 비밀 중 하나입니다.

---

## 3. 요약 → 번역 파이프라인 구축 실습 (코드 해설)

`04_mini_project.py` 파일을 생성하고 코드를 작성하며 흐름을 따라가 보세요.

### [Step 1] 요약 체인과 번역 체인 준비
```python
# 04_mini_project.py
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_community.chat_models import ChatOllama

llm = ChatOllama(model="glm-4.7-flash:q4_K_M", temperature=0)

# ── [첫 번째 체인] 텍스트 요약 ──
# 입력 기대값: {"text": "길고 장황한 원본 글..."}
# 출력 반환값: str (요약된 영문 문자열)
summary_prompt = ChatPromptTemplate.from_template(
    "다음 텍스트를 자세히 읽고, 가장 중요한 핵심 내용을 2문장 이내로 짧게 요약해:\n\n{text}"
)
summary_chain = summary_prompt | llm | StrOutputParser()

# ── [두 번째 체인] 한국어 번역 ──
# 입력 기대값: {"summary_text": "요약된 문자열"}
# 출력 반환값: str (최종 번역된 한국어 문자열)
translate_prompt = ChatPromptTemplate.from_template(
    "다음 문장을 아주 자연스럽고 매끄러운 한국어로 번역해:\n\n{summary_text}"
)
translate_chain = translate_prompt | llm | StrOutputParser()
```

### [Step 2] 마스터 체인 조립 (Type Flow 매핑의 정수)
```python
# 사용자는 최초에 순수 문자열("LangChain is...")을 단독으로 입력할 것입니다.
# 따라서 입력부터 중간중간 타입을 맞춰주는 징검다리 작업이 필요합니다.

final_chain = (
    # 1. 사용자 최초 입력(str)을 summary_prompt가 원하는 딕셔너리 형태로 포장합니다.
    # {"text": lambda x: x} 는 LangChain에서 RunnableParallel 문법의 축약형으로 동작합니다.
    {"text": lambda x: x}                          
    
    # 2. 요약 체인을 통과합니다. 결과는 영문 str 입니다.
    | summary_chain                                 
    
    # 3. 요약된 결과(str)를 translate_prompt가 원하는 딕셔너리 형태로 다시 포장합니다.
    | (lambda summary: {"summary_text": summary})   
    
    # 4. 번역 체인을 통과합니다. 결과는 한국어 str 입니다.
    | translate_chain                               
)
```

### [Step 3] 봇 가동!
```python
sample_text = """
LangChain is a framework designed to simplify the creation of applications using
large language models (LLMs). As a language model integration framework, LangChain's
use-cases largely overlap with those of language models in general, including document
analysis and summarization, chatbots, and code analysis. It allows developers to connect
LLMs with other sources of data to create complex workflows.
"""

print("── 미니 프로젝트: 요약 & 번역 시작 ──\n")
# 사용자는 딕셔너리 포장 없이 원본 문자열만 쿨하게 넘기면 알아서 파이프라인을 탑니다!
result = final_chain.invoke(sample_text)

print("\n── 최종 결과 (한국어 번역본) ──")
print(result)
```

터미널에서 실행해 봅니다: `uv run 04_mini_project.py`

---

## 4. 🏋️‍♂️ Your Turn! (도전 과제)

지금 만든 다중 체인 파이프라인의 내부 동작을 더 깊이 이해하기 위해 실험해 봅시다.

1. **징검다리 부숴보기**: 마스터 체인(`final_chain`) 조립 부분에서 세 번째 줄에 있는 중간 람다 변환기 `| (lambda summary: {"summary_text": summary})` 코드를 주석 처리하고 지워보세요. 그리고 코드를 실행했을 때 발생하는 길고 무서운 에러 메시지(Traceback)를 직접 확인해 보세요. 어떤 타입의 입력이 없어서 터졌는지 에러 로그 읽는 법을 훈련할 수 있습니다.
2. **새로운 체인 추가해보기**: `translate_chain` 아래에 하나 더 체인을 연결해 보세요. 번역된 한국어 텍스트를 받아서 "이 텍스트의 감정(긍정/부정)은 무엇인가요?"라고 묻는 세 번째 체인을 붙이고 파이프라인을 연장해 보는 것입니다.

[⬅️ 이전 단계: 4장. LCEL과 Runnable 인터페이스](04-lcel-basics.md) | [🏠 메인 메뉴](README.md) | [➡️ 다음 단계: 6장. 메인 프로젝트: 웹 크롤러 자동화](06-main-project.md)
