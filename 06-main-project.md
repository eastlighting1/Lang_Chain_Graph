# 6장. 메인 프로젝트: 웹 크롤러 기반 E2E 자동화 파이프라인

이제 가장 실용적이고 강력한 E2E(End-to-End) 파이프라인을 구축해 보겠습니다.
목표는 **웹페이지 URL만 던져주면 👉 자동으로 접속해서 본문을 긁어오고 👉 핵심을 요약한 뒤 👉 한국어 뉴스로 번역해서 출력해주는** 자동화 봇을 만드는 것입니다.

이 챕터의 핵심은 LangChain 외부의 **일반 Python 함수를 LCEL 파이프라인 안에 편입시키는 방법**입니다.

---

## 1. 일반 함수를 Runnable로 변환하기 (`RunnableLambda`)

LangChain 외부의 라이브러리(예: `requests`, `BeautifulSoup`, `pandas` 등)를 사용하는 일반적인 파이썬 함수는 `|` 파이프 연산자로 바로 연결할 수 없습니다. 인터페이스가 달라서 LCEL이 이를 인식하지 못하기 때문입니다.

이때 **`RunnableLambda`** 라는 마법의 래퍼(Wrapper)를 사용합니다. 일반 함수에 옷을 입혀 LangChain의 `Runnable` 인터페이스 규격에 맞게 둔갑시켜 줍니다.

---

## 2. URL → 크롤링 → 요약 → 번역 파이프라인 실습 (코드)

1장에서 미리 설치했던 `requests`와 `beautifulsoup4` 라이브러리를 활용합니다.
`05_main_project.py` 파일을 생성하고 아래 코드를 작성합니다.

```python
# 05_main_project.py
import requests
from bs4 import BeautifulSoup
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnableLambda
from langchain_community.chat_models import ChatOllama

llm = ChatOllama(model="glm-4.7-flash:q4_K_M", temperature=0)

# ── [1] 외부 라이브러리를 사용한 일반 크롤러 함수 ──────────────────────────
# URL(str)을 입력받아, 웹페이지의 P 태그 텍스트들을 긁어모아 반환하는 순수 Python 함수입니다.
def crawl_article(url: str) -> str:
    print(f"📡 [진행 중] {url} 주소에서 크롤링을 시작합니다...")
    
    # 일부 웹사이트의 차단을 막기 위해 User-Agent를 사람처럼 속입니다.
    headers = {"User-Agent": "Mozilla/5.0"}
    response = requests.get(url, headers=headers, timeout=10)
    
    # HTML에서 의미 있는 <p> 태그의 텍스트만 추출합니다.
    soup = BeautifulSoup(response.text, "html.parser")
    paragraphs = soup.find_all("p")
    content = " ".join(p.get_text(strip=True) for p in paragraphs if p.get_text(strip=True))

    # 로컬 모델의 컨텍스트 한계(토큰 제한)와 속도를 고려해 최대 3000자까지만 자릅니다.
    return content[:3000]

# ── [2] 일반 함수를 LCEL에 끼울 수 있게 Runnable로 변장! ───────────────
crawler_runnable = RunnableLambda(crawl_article)


# ── [3] 첫 번째 체인: 기사 요약 ─────────────────────────────────────────
summary_prompt = ChatPromptTemplate.from_template(
    "다음 기사 본문을 자세히 읽고, 가장 중요한 핵심 내용을 3가지 불릿 포인트(- )로 요약해줘.\n\n"
    "기사 내용:\n{article_text}"
)
summary_chain = summary_prompt | llm | StrOutputParser()


# ── [4] 두 번째 체인: 한국어 뉴스 문체로 번역 ───────────────────────────
translate_prompt = ChatPromptTemplate.from_template(
    "다음 요약된 내용을 자연스럽고 전문적인 한국어 뉴스 앵커 문체로 번역해줘.\n"
    "반드시 높임말을 사용해야 해.\n\n"
    "요약 내용:\n{summarized_text}"
)
translate_chain = translate_prompt | llm | StrOutputParser()


# ── [5] 최종 파이프라인 조립 (마스터피스) ──────────────────────────────
# URL(str) → [크롤링] → 텍스트(str) → [dict 매핑] → [요약 체인] → 요약문(str) → [dict 매핑] → [번역 체인]
main_pipeline = (
    crawler_runnable
    | (lambda text: {"article_text": text})
    | summary_chain
    | (lambda summary: {"summarized_text": summary})
    | translate_chain
)


# ── [6] 봇 가동! ───────────────────────────────────────────────────────
if __name__ == "__main__":
    # 영문 위키피디아의 'Large Language Model' 문서 URL입니다.
    test_url = "https://en.wikipedia.org/wiki/Large_language_model"

    print("🚀 메인 프로젝트 파이프라인 시작 🚀\n")
    
    # 우리는 그저 파이프라인에 URL 하나만 던지면 됩니다.
    result = main_pipeline.invoke(test_url)

    print("\n" + "="*50)
    print("🎯 최종 한국어 뉴스 브리핑 결과")
    print("="*50)
    print(result)
```

터미널에서 웅장하게 실행해 봅니다: `uv run 05_main_project.py`

> 💡 **코드 리뷰**: `crawler_runnable = RunnableLambda(crawl_article)` 이 한 줄이 핵심입니다. 이 처리를 하지 않고 `main_pipeline`에 `crawl_article`을 그대로 집어넣으면 파이썬은 "지원하지 않는 피연산자 타입입니다(TypeError: unsupported operand type(s) for |)" 라며 에러를 뿜습니다. LCEL 파이프는 오직 `Runnable` 규격끼리만 연결될 수 있기 때문입니다.

---
### 🎯 6장 요약
- 외부 API, 데이터베이스 조회, 웹 크롤링 등의 일반 로직을 파이프라인에 넣으려면 `RunnableLambda`로 감싸주어야 합니다.
- 입력값 변환(람다 매핑)을 적절히 활용하면, 무한히 긴 복잡한 워크플로우도 하나의 단일 `chain` 객체처럼 단순하게 제어할 수 있습니다.

[⬅️ 이전 단계: 5장. 미니 파이프라인 프로젝트](05-mini-project.md) | [🏠 메인 메뉴](README.md) | [➡️ 다음 단계: 7장. 총복습 및 다음 단계](07-review-and-next-steps.md)
