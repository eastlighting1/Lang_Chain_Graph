# 5장. 미니 프로젝트: 텍스트 요약 및 번역 파이프라인 구축

이제 기본기를 모두 배웠으니, 두 개 이상의 체인을 결합하여 **"긴 영문 글을 읽고 핵심을 요약한 뒤 한국어로 번역하는"** 파이프라인을 구축해 보겠습니다.

이 챕터의 핵심은 4장에서 강조했던 **입출력 타입(Type Flow)**을 어떻게 수동으로 맞추어 연결하는지를 배우는 것입니다.

---

## 1. 다중 체인 조립의 문제점: 타입 불일치 (Type Mismatch)

앞서 만든 체인(`prompt | llm | parser`)의 최종 출력은 **순수 문자열(`str`)**입니다.
그런데 이 출력을 다음 체인의 입력으로 곧바로 연결하면 어떻게 될까요?

```python
# ⛔ 에러 발생 코드 (개념 설명용)
summary_chain = summary_prompt | llm | parser   # 출력: str ("The core concept is...")
translate_chain = translate_prompt | llm | parser # 입력 대기: {"summary_text": "..."} 형태의 dict

# summary_chain의 출력(str)이 translate_chain의 입력(dict)과 맞지 않아 에러가 납니다!
final_chain = summary_chain | translate_chain 
```

`translate_prompt`는 `{summary_text}`라는 변수를 채우기 위해 딕셔너리(`dict`) 입력을 기다리고 있는데, 앞 체인에서 냅다 문자열(`str`)을 던지니 파이프라인이 터지는 것입니다.

## 2. 해결책: 파이썬 람다(`lambda`)를 이용한 중간 변환

이 문제를 해결하기 위해, 체인과 체인 사이에 문자열을 딕셔너리로 예쁘게 포장해 주는 아주 작은 변환기가 필요합니다. 파이썬의 익명 함수인 `lambda`를 사용하면 쉽습니다.

```python
# 문자열(summary)을 받아서, {"summary_text": summary} 형태의 딕셔너리로 반환합니다.
lambda summary: {"summary_text": summary}
```
이제 이 중간 다리를 넣어서 코드를 작성해 봅시다.

---

## 3. 요약 → 번역 파이프라인 구축 실습 (코드)

`04_mini_project.py` 파일을 생성하고 아래 코드를 꼼꼼히 주석과 함께 읽으며 작성해 보세요.

```python
# 04_mini_project.py
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_community.chat_models import ChatOllama

llm = ChatOllama(model="glm-4.7-flash:q4_K_M", temperature=0)

# ── [1] 첫 번째 체인: 텍스트 요약 ───────────────────────────────────────
# 입력: {"text": "길고 장황한 원본 글..."}
# 출력: str (요약된 문자열)
summary_prompt = ChatPromptTemplate.from_template(
    "다음 텍스트를 자세히 읽고, 가장 중요한 핵심 내용을 2문장 이내로 짧게 요약해줘:\n\n{text}"
)
summary_chain = summary_prompt | llm | StrOutputParser()

# ── [2] 두 번째 체인: 한국어 번역 ───────────────────────────────────────
# 입력: {"summary_text": "요약된 문자열"}
# 출력: str (최종 번역된 문자열)
translate_prompt = ChatPromptTemplate.from_template(
    "다음 문장을 아주 자연스럽고 매끄러운 한국어로 번역해줘:\n\n{summary_text}"
)
translate_chain = translate_prompt | llm | StrOutputParser()

# ── [3] 두 체인을 하나로 결합 (Type Flow 매핑!) ───────────────────────
# 사용자는 최초에 순수 문자열("LangChain is...")을 입력할 것입니다.
final_chain = (
    # 1. 사용자 입력(str)을 summary_prompt가 원하는 딕셔너리 형태로 포장합니다.
    {"text": lambda x: x}                          
    # 2. 요약 체인을 통과합니다. 결과는 str 입니다.
    | summary_chain                                 
    # 3. 요약된 결과(str)를 translate_prompt가 원하는 딕셔너리 형태로 다시 포장합니다.
    | (lambda summary: {"summary_text": summary})   
    # 4. 번역 체인을 통과합니다. 결과는 한국어 str 입니다.
    | translate_chain                               
)

# ── 4. 테스트 실행 ──────────────────────────────────────────────────────
sample_text = """
LangChain is a framework designed to simplify the creation of applications using
large language models (LLMs). As a language model integration framework, LangChain's
use-cases largely overlap with those of language models in general, including document
analysis and summarization, chatbots, and code analysis. It allows developers to connect
LLMs with other sources of data to create complex workflows.
"""

print("── 미니 프로젝트: 요약 & 번역 시작 ──\n")
# 사용자는 딕셔너리 포장 없이 원본 문자열만 쿨하게 넘기면 됩니다!
result = final_chain.invoke(sample_text)

print("\n── 최종 결과 (한국어 번역본) ──")
print(result)
```

터미널에서 실행해 봅니다: `uv run 04_mini_project.py`

> 💡 **코드 리뷰**: `{"text": lambda x: x}`는 LangChain의 특별한 문법인 `RunnableParallel`의 축약형입니다. 들어온 입력(`x`)을 그대로 사용하되, 딕셔너리의 `text` 키 값에 매핑하겠다는 의미입니다.

---
### 🎯 5장 요약
- 여러 개의 체인을 직렬로 연결할 때는 앞 체인의 **출력 타입**과 뒤 체인의 **입력 타입**을 완벽하게 맞춰야 합니다.
- 이 간극을 메우기 위해 파이썬의 `lambda` 함수를 파이프라인(`|`) 중간에 삽입하여 딕셔너리 구조로 맵핑(Mapping)합니다.
- 복잡한 로직을 작은 단위의 체인으로 쪼개고, 이를 하나로 조립하는 것이 유지보수에 유리합니다.

[⬅️ 이전 단계: 4장. LCEL과 Runnable 인터페이스](04-lcel-basics.md) | [🏠 메인 메뉴](README.md) | [➡️ 다음 단계: 6장. 메인 프로젝트: 웹 크롤러 자동화](06-main-project.md)
