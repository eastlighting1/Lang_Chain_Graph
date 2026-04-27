# 1장. 환경 설정 및 로컬 LLM 준비

성공적인 LangChain 학습을 위해 가장 먼저 해야 할 일은 **깔끔한 Python 환경을 구축**하고, API 비용 걱정 없이 무제한으로 테스트할 수 있는 **로컬 LLM 런타임을 준비**하는 것입니다.

이 챕터에서는 빠르고 모던한 패키지 매니저인 `uv`를 사용해 프로젝트를 초기화하고, `Ollama`를 통해 로컬에서 LLM을 구동하는 방법을 배웁니다.

---

## 1. 빠르고 쾌적한 Python 환경 구성 (`uv`)

최근 Python 생태계에서는 Rust로 작성되어 기존 `pip`나 `poetry`보다 압도적으로 빠른 속도를 자랑하는 **`uv`**가 큰 인기를 끌고 있습니다. `uv`는 가상환경(Virtual Environment) 생성과 패키지 관리를 하나로 통합해 줍니다.

### 프로젝트 초기화 및 패키지 설치 실습

터미널을 열고 실습용 폴더를 만들 위치로 이동한 뒤, 아래 명령어를 순서대로 실행해 보세요.

```bash
# 1. 프로젝트 초기화 (pyproject.toml과 .venv 가상환경이 자동 생성됩니다)
uv init langchain-study
cd langchain-study

# 2. 튜토리얼 진행에 필요한 핵심 패키지들 설치
uv add langchain langchain-community langchain-core pydantic requests beautifulsoup4
```

#### 📦 설치된 패키지들의 역할
- `langchain`, `langchain-core`: LangChain의 핵심 인터페이스와 로직 체인 로직을 제공합니다.
- `langchain-community`: Ollama 등 외부 서드파티 통합을 위한 클래스들이 모여 있습니다.
- `pydantic`: 데이터 검증 및 LLM 응답을 파이썬 객체로 구조화할 때 사용합니다.
- `requests`, `beautifulsoup4`: 추후 6장에서 웹 문서를 크롤링할 때 사용할 라이브러리입니다.

### 환경 검증

설치가 잘 되었는지 확인해 보겠습니다. Windows와 macOS/Linux의 가상환경 활성화 방식이 다릅니다.

```bash
# 가상환경 활성화 (Windows 환경인 경우)
.venv\Scripts\activate

# 가상환경 활성화 (macOS / Linux 환경인 경우)
source .venv/bin/activate

# 설치 확인
uv pip list
```
화면에 `langchain`, `langchain-community`, `pydantic` 등이 정상적으로 리스트업된다면 성공입니다!

---

## 2. API 비용 0원! Ollama 로컬 LLM 준비

LangChain 실습을 위해 OpenAI(ChatGPT) API를 사용할 수도 있지만, 개발 및 테스트 단계에서는 비용이 발생하지 않고 인터넷 연결 없이도 동작하는 **로컬 LLM**을 활용하는 것이 최고의 선택입니다.

[Ollama](https://ollama.com/)는 로컬 머신에서 복잡한 설정 없이 오픈소스 LLM을 단일 명령어로 실행할 수 있게 해주는 혁신적인 도구입니다. Docker와 비슷한 사용자 경험을 제공합니다. (미리 Ollama가 설치되어 있다고 가정합니다.)

### 모델 다운로드 및 백그라운드 구동 실습

터미널에 아래 명령어를 입력합니다. 이번 튜토리얼에서는 가벼우면서도 한국어 성능이 준수한 `glm-4.7-flash:q4_K_M` 모델을 사용합니다.

```bash
ollama run glm-4.7-flash:q4_K_M
```

- **최초 실행 시**: 모델 파일(수 GB)을 다운로드하는 과정이 진행됩니다.
- **다운로드 완료 후**: `>>>` 라는 프롬프트가 나타나며, 여기서 직접 텍스트를 입력해 AI와 대화해볼 수 있습니다.
  - 예: `>>> 안녕? 넌 누구야?`
- **종료**: `Ctrl + D` 또는 `/bye`를 입력하면 대화창을 빠져나옵니다. 

> 💡 **중요**: 대화창을 종료하더라도 Ollama의 **백그라운드 API 서버(포트 11434)는 계속 실행 중인 상태**로 유지됩니다. 따라서 LangChain 코드에서 문제없이 모델을 호출할 수 있습니다.

### 동작 상태 점검

백그라운드에서 Ollama 서버가 정상적으로 돌고 있는지 확인하려면 웹 브라우저를 열고 다음 주소로 접속해 보세요.
👉 `http://localhost:11434`

화면에 덩그러니 **`Ollama is running`** 이라는 텍스트가 뜬다면 준비가 완벽하게 끝난 것입니다.

---
### 🎯 1장 요약
- `uv`를 통해 의존성 충돌 없는 독립적인 프로젝트 가상환경을 구축했습니다.
- `Ollama`를 이용해 무료로 무제한 호출이 가능한 로컬 AI 두뇌를 확보했습니다.
- 이제 본격적으로 코드를 작성해 LLM과 통신해 볼 차례입니다.

[➡️ 다음 단계: 2장. LangChain 핵심 아키텍처 이해 및 LLM 호출 실습](02-langchain-llm-calls.md)
