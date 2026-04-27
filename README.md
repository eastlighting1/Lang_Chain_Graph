# LangChain 기초 쿡북: LLM과 대화하고 파이프라인 구축하기

> **학습 목표**: 로컬 환경에서 오픈소스 LLM을 구동하는 것부터 시작해, LangChain의 핵심 컴포넌트를 이해하고 LCEL(LangChain Expression Language)을 기반으로 한 자동화 파이프라인을 구축하는 과정까지 단계별로 직접 손으로 짜며 익힙니다.
> **사용 환경**: Python 3.11 이상, `uv` 패키지 매니저, Ollama (`glm-4.7-flash:q4_K_M` 모델)

---

## 📚 튜토리얼 목차

이 쿡북은 개념 이해와 실제 코드를 작성해보는 실습이 완결성 있게 하나의 챕터로 구성되어 있습니다. 각 챕터의 가이드를 따라 코드를 직접 타이핑하며 학습하는 것을 권장합니다.

| 챕터 | 내용 | 문서 링크 |
|------|------|-----------|
| **1장** | 프로젝트 환경 설정 및 Ollama를 활용한 로컬 LLM 구동 | [01-setup-and-local-llm.md](01-setup-and-local-llm.md) |
| **2장** | LangChain 핵심 아키텍처 이해 및 LLM 호출 실습 | [02-langchain-llm-calls.md](02-langchain-llm-calls.md) |
| **3장** | 프롬프트 템플릿 설계와 출력 파서(Output Parser) | [03-prompt-and-parsers.md](03-prompt-and-parsers.md) |
| **4장** | LCEL(LangChain Expression Language)과 Runnable 인터페이스 | [04-lcel-basics.md](04-lcel-basics.md) |
| **5장** | 미니 프로젝트: 텍스트 요약 및 번역 파이프라인 구축 | [05-mini-project.md](05-mini-project.md) |
| **6장** | 메인 프로젝트: 웹 크롤러 기반 E2E 자동화 파이프라인 | [06-main-project.md](06-main-project.md) |
| **7장** | 핵심 개념 총복습 및 심화 학습을 위한 다음 단계 | [07-review-and-next-steps.md](07-review-and-next-steps.md) |

---
> 💡 **Tip:** 모든 예제 코드는 복사-붙여넣기보다는 가급적 직접 타이핑해 보시길 권합니다. 코드의 흐름(Type Flow)을 이해하는 것이 LangChain 학습의 핵심입니다.
