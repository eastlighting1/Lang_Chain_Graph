# 6장. 메인 프로젝트: 웹 크롤러 기반 E2E 자동화 파이프라인

이제 가장 실용적이고 강력한 E2E(End-to-End) 파이프라인을 구축할 차례입니다.
목표는 **웹페이지 URL만 던져주면 👉 파이썬이 자동으로 접속해서 본문을 긁어오고 👉 핵심을 요약한 뒤 👉 한국어 뉴스로 번역해서 출력해주는** 자동화 봇입니다.

이 챕터의 핵심은 LangChain 생태계 외부의 **일반 Python 코드를 LCEL의 톱니바퀴 안에 편입시키는 방법**을 이해하는 것입니다.

---

## 1. 일반 함수를 체인에 엮기 (`RunnableLambda` 딥다이브)

우리는 웹을 크롤링하기 위해 파이썬의 표준적인 서드파티 라이브러리인 `requests`와 `BeautifulSoup`을 사용할 것입니다. 그런데 사용자가 작성한 일반 파이썬 함수 `def crawl_article(url):` 은 `Runnable` 객체가 아니므로, 아까 배운 `|` 파이프 연산자로 체인에 연결하려고 하면 파이썬이 문법 에러(`TypeError`)를 뿜어냅니다.

이때 구원투수로 등장하는 것이 **`RunnableLambda`** 입니다.

> 💡 **Under the Hood**: `RunnableLambda`는 데코레이터나 래퍼(Wrapper) 패턴과 같은 역할을 합니다. 여러분의 평범한 파이썬 함수를 감싸서(Wrap), 겉보기에는 `Runnable` 클래스처럼 보이도록 위장시켜 줍니다. 덕분에 `invoke`, `stream` 메서드를 지원하게 되고, `|` 연산자도 무리 없이 사용할 수 있게 됩니다.

---

## 2. URL → 크롤링 → 요약 → 번역 실습 (코드 해설)

`05_main_project.py` 파일을 생성하고 아래 코드를 부분별로 이해하며 작성해 보세요.

### [Step 1] 순수 파이썬 크롤링 함수 작성
```python
# 05_main_project.py
import requests
from bs4 import BeautifulSoup
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnableLambda
from langchain_community.chat_models import ChatOllama

llm = ChatOllama(model="glm-4.7-flash:q4_K_M", temperature=0)

# URL(str)을 입력받아, 웹페이지의 본문 텍스트를 추출하는 순수 파이썬 함수입니다.
# 이 함수 안에는 LangChain 관련 코드가 단 한 줄도 없습니다!
def crawl_article(url: str) -> str:
    print(f"📡 [크롤링 진행 중] {url} 접속 중...")
    
    # 일부 웹사이트 차단을 피하기 위해 봇이 아닌 일반 브라우저처럼 속입니다.
    headers = {"User-Agent": "Mozilla/5.0"}
    response = requests.get(url, headers=headers, timeout=10)
    
    # HTML 파싱 후 <p> 태그(문단)의 텍스트만 긁어모읍니다.
    soup = BeautifulSoup(response.text, "html.parser")
    paragraphs = soup.find_all("p")
    content = " ".join(p.get_text(strip=True) for p in paragraphs if p.get_text(strip=True))

    # 로컬 AI 모델의 문맥 창(Context Window) 한계를 고려해 앞부분 3000자만 리턴합니다.
    return content[:3000]
```

### [Step 2] 함수를 Runnable로 둔갑시키고 체인들 정의하기
```python
# 여기서 일반 함수가 LangChain 생태계의 시민권을 얻게 됩니다!
crawler_runnable = RunnableLambda(crawl_article)

# ── 첫 번째 체인: 기사 요약 ──
summary_prompt = ChatPromptTemplate.from_template(
    "다음 기사 본문을 자세히 읽고, 가장 중요한 핵심 내용을 3가지 불릿 포인트(- )로 요약해.\n\n기사 내용:\n{article_text}"
)
summary_chain = summary_prompt | llm | StrOutputParser()

# ── 두 번째 체인: 한국어 뉴스 앵커 톤 번역 ──
translate_prompt = ChatPromptTemplate.from_template(
    "다음 요약된 내용을 자연스럽고 전문적인 한국어 뉴스 앵커 문체로 번역해. 반드시 높임말을 써.\n\n요약 내용:\n{summarized_text}"
)
translate_chain = translate_prompt | llm | StrOutputParser()
```

### [Step 3] 거대 E2E 파이프라인 조립 및 실행
```python
# [크롤링] → (딕셔너리 포장) → [요약] → (딕셔너리 포장) → [번역]
main_pipeline = (
    crawler_runnable
    | (lambda text: {"article_text": text})
    | summary_chain
    | (lambda summary: {"summarized_text": summary})
    | translate_chain
)

if __name__ == "__main__":
    # 영문 위키피디아의 'Large Language Model' 문서 URL
    test_url = "https://en.wikipedia.org/wiki/Large_language_model"

    print("🚀 메인 프로젝트 파이프라인 시작 🚀\n")
    
    # 우리는 그저 파이프라인 입구에 URL 하나만 툭 던지면 끝입니다.
    result = main_pipeline.invoke(test_url)

    print("\n" + "="*50)
    print("🎯 최종 한국어 뉴스 브리핑 결과")
    print("="*50)
    print(result)
```

터미널에서 웅장하게 실행해 봅니다: `uv run 05_main_project.py`

> 💡 **비동기 확장성에 대한 고찰 (Advanced)**: 크롤링(`requests.get`)처럼 네트워크 응답을 기다려야 하는 I/O 바운드 작업은 파이프라인을 멈추게(Block) 합니다. 
만약 100개의 URL을 처리해야 한다면? LCEL에 내장된 `.abatch()` 메서드와 비동기 라이브러리(`aiohttp`)를 결합하면 코드를 크게 바꾸지 않고도 동시에 100개의 크롤링을 처리하도록 확장할 수 있습니다. 이것이 프레임워크를 쓰는 이유입니다.

---

## 3. 🏋️‍♂️ Your Turn! (도전 과제)

마지막 프로젝트 코드를 여러분의 입맛에 맞게 개조해 봅시다.

1. **관심 있는 URL로 테스트**: 위키피디아가 아닌 해외 유명 IT 뉴스 사이트(TechCrunch, The Verge 등)나 블로그의 영문 기사 URL을 `test_url`에 넣고 실행해 보세요. 요약 퀄리티가 만족스러운가요?
2. **에러 핸들링 추가해보기 (어려움)**: 만약 사용자가 잘못된 URL을 입력해서 `requests.get`이 에러(404 등)를 뿜으면 파이프라인 전체가 멈춥니다. `crawl_article` 함수 내부에 `try-except` 문을 넣고, 에러 발생 시 "URL에 접속할 수 없습니다"라는 텍스트를 대신 리턴하도록 수정하여 파이프라인의 맷집(Robustness)을 길러보세요.

[⬅️ 이전 단계: 5장. 미니 프로젝트 - 요약 및 번역](05-mini-project.md) | [🏠 메인 메뉴](README.md) | [➡️ 다음 단계: 7장. 총복습 및 다음 단계](07-review-and-next-steps.md)
