---
title: "Phase 1-A: 의료 상담 에이전트 구축"
weight: 21
time: "60분"
---

# Phase 1-A: 의료 상담 에이전트 구축 (60분)

## 학습 목표

Strands Agents SDK를 사용하여 환자 증상 기반 초기 상담 에이전트를 구축합니다.

---

## 이론: Agentic AI 패턴과 Amazon Bedrock AgentCore

> **참고**: 이 섹션은 AWS 공식 과정 "Building Agentic AI with Amazon Bedrock AgentCore"의
> Module 1 (Foundations of Agentic AI Patterns)과 Module 2 (AgentCore Runtime) 내용을 기반으로 합니다.

### Agentic AI란?

**기존 AI 시스템**은 하나의 입력에 하나의 출력을 반환합니다 (Input → Model → Output).

**Agentic AI**는 목표를 달성하기 위해 **스스로 판단하고, 도구를 사용하고, 반복적으로 행동**합니다.

```
기존 AI:     사용자 → [LLM] → 응답 (1회)

Agentic AI:  사용자 → [Agent] → 도구 호출 → 결과 확인 → 추가 도구 호출 → ... → 최종 응답
                        ↑                                              │
                        └──────────────── 반복 (Agent Loop) ────────────┘
```

### 왜 Agentic AI 프레임워크가 필요한가?

LLM만으로는 아래와 같은 작업을 수행할 수 없습니다:

| 한계 | 프레임워크가 해결하는 방법 |
|------|-------------------------|
| LLM은 외부 데이터에 접근 불가 | **Tool 통합**: 검사 결과 DB 조회, API 호출 |
| 학습 이후 정보를 모름 | **Memory**: 이전 상담 이력 조회/저장 |
| 한 번의 호출로 복잡한 작업 불가 | **Agent Loop**: 여러 단계를 자동 반복 수행 |
| 응답 품질 일관성 보장 어려움 | **Guardrails + Evaluations**: 규칙 적용 + 자동 평가 |
| 여러 전문 역할이 필요한 작업 | **멀티 에이전트 오케스트레이션**: 역할별 에이전트 협업 |

프레임워크 없이 이 모든 것을 직접 구현하려면 Tool 호출 루프, 에러 핸들링, 메모리 관리, 모니터링 코드를 모두 처음부터 작성해야 합니다. Agentic AI 프레임워크는 이러한 공통 패턴을 추상화하여 개발자가 **비즈니스 로직(도구, 프롬프트)에만 집중**할 수 있게 합니다.

### 헬스케어 분야의 Agentic AI 프레임워크 활용

헬스케어는 Agentic AI가 가장 활발히 연구·적용되고 있는 분야 중 하나입니다:

| 프레임워크 | 프레임워크 적용 예시 |
|-----------|-----------------|
| **Strands Agents (AWS)** | AgentCore Runtime/Memory/Observability와 네이티브 통합. 다양한 사례에 적용 가능. 본 워크샵에서 사용. ([AWS Blog](https://aws.amazon.com/ko/blogs/industries/prior-authorization-for-medical-claims-using-strands-agents/)) |
| **LangGraph** | Doctolib(유럽 의료 예약 플랫폼)의 고객 지원 AI 에이전트 "Alfred" 구축. 상태 기반 워크플로우와 human-in-the-loop에 강점. ([zenml.io](https://www.zenml.io/llmops-database/building-an-agentic-ai-system-for-healthcare-support-using-langgraph)) |

> 헬스케어 분야는 규제, 감사 추적, 환자 데이터 격리가 중요하므로
> 프레임워크 선택 시 **프로덕션 운영 인프라(배포, 모니터링, 접근 제어)**가 함께 제공되는지가 핵심 기준입니다.
> 본 워크샵에서는 이 요구를 AgentCore 서비스로 충족합니다.

학술 연구에서도 Agentic AI가 임상 의사결정 지원(CDSS)의 한계를 극복하는 접근법으로 주목받고 있습니다. 자율 에이전트가 구조화된 메모리와 EHR 상호운용성을 결합하여 실시간 진단, 트리아지, 치료 계획을 지원하는 프레임워크가 제안되고 있습니다. ([Frontiers in Medicine, 2025](https://www.frontiersin.org/journals/medicine/articles/10.3389/fmed.2025.1753443/full))

### Agent의 핵심 구성 요소 (Building Blocks)

| 구성 요소 | 역할 | 본 워크샵 예시 |
|-----------|------|-------------|
| **Model** | 추론 + 판단 (어떤 도구를 호출할지 결정) | Claude Sonnet 4.5 |
| **Tools** | 에이전트가 실행할 수 있는 기능 | 검사 결과 조회, 긴급도 평가 |
| **System Prompt** | 에이전트의 역할/규칙/제약 정의 | 의료법 준수 규칙, 면책 조항 |
| **Memory** | 대화 이력 및 학습된 지식 유지 | 이전 상담 기록, 환자 선호도 |

### Amazon Bedrock AgentCore 소개

AgentCore는 에이전트를 **구축, 배포, 운영**하기 위한 통합 플랫폼입니다.

핵심 특징:
- **프레임워크 자유**: Strands Agents, LangGraph, CrewAI 등 어떤 SDK든 사용 가능
- **모델 자유**: Claude, Nova, Llama 등 어떤 모델이든 사용 가능
- **모듈형 서비스**: 필요한 서비스만 선택하여 조합

```
┌─────────────────────────────────────────────────────┐
│            Amazon Bedrock AgentCore                 │
│                                                     │
│  Day 1:                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐   │
│  │ Runtime  │  │  Memory  │  │  Observability   │   │
│  │(배포/호스팅)│  │(대화 이력) │  │(모니터링/추적)      │   │
│  └──────────┘  └──────────┘  └──────────────────┘   │
│                                                     │
│  Day 2:                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐   │
│  │  Policy  │  │ Gateway  │  │  Evaluations     │   │
│  │(접근 제어) │  │(도구 연결) │  │(품질 평가)          │   │
│  └──────────┘  └──────────┘  └──────────────────┘   │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### AgentCore Runtime

에이전트를 **서버리스 환경**에서 호스팅합니다.

| 직접 EC2/ECS 운영 | AgentCore Runtime |
|-------------------|-------------------|
| 인프라 관리 필요 | 관리형 서버리스 |
| 스케일링 직접 설정 | 자동 스케일링 |
| 세션 격리 직접 구현 | microVM 기반 세션 격리 |
| 모니터링 코드 작성 | Observability 자동 계측 |

### Strands Agents SDK

본 워크샵에서 에이전트 코드를 작성하는 데 사용하는 프레임워크입니다.

**왜 Strands인가?**
- 최소한의 코드로 에이전트 구축 가능
- `@tool` 데코레이터로 Python 함수를 바로 도구로 변환
- AgentCore Runtime/Memory/Observability와 코드 한 줄로 통합
- 의료 규정 준수 로직을 시스템 프롬프트 + Tool에 선언적으로 정의 가능
- 멀티 에이전트(Supervisor → Triage/Analysis/Recommendation) 패턴을 Agent-as-Tool로 간결하게 구현
- 오픈소스 ([GitHub: strands-agents/sdk-python](https://github.com/strands-agents/sdk-python))

**코드 예시: 가장 간단한 에이전트**

```python
from strands import Agent, tool
from strands.models import BedrockModel

@tool
def get_weather(city: str) -> str:
    """도시의 현재 날씨를 조회합니다."""
    return f"{city}: 맑음, 28도"

agent = Agent(
    model=BedrockModel(model_id="global.anthropic.claude-sonnet-4-5-20250929-v1:0"),
    tools=[get_weather],
    system_prompt="당신은 날씨 안내 도우미입니다."
)

response = agent("서울 날씨 알려줘")
print(response.message['content'][0]['text'])
```

### Agent Loop 동작 방식

```
사용자: "서울 날씨 알려줘"
    │
    ▼
Model 판단: "get_weather 도구를 호출해야겠다" (stop_reason: tool_use)
    │
    ▼
SDK가 get_weather("서울") 실행 → "서울: 맑음, 28도"
    │
    ▼
결과를 Model에 전달 → Model이 최종 응답 생성 (stop_reason: end_turn)
    │
    ▼
"서울의 현재 날씨는 맑음, 28도입니다."
```

**개발자가 "어떤 도구를 호출할지" if-else 로직을 작성할 필요 없음** — 모델이 판단합니다.

---

## 실습 시작

### Step 1: 프로젝트 파일 생성

```bash
mkdir -p ~/agentcore/src
cd ~/agentcore/src
touch consultation_agent.py
```

### Step 2: 전체 코드 작성

`consultation_agent.py` 파일을 열고 아래 코드를 복사하여 붙여넣으세요. `TODO` 주석이 있는 빈칸(`____`)을 직접 채워서 에이전트를 완성합니다.

> 최대한 직접 채우는 걸로 시도하고, 정말 어렵게 느껴진다면 하단의 **부록: 정답 코드**를 참고하세요.

```python
"""
Healthcare Consultation Agent
- 환자 증상 기반 초기 상담
- 대한민국 의료법 제27조 준수
"""

# Strands Agents SDK에서 Agent 클래스와 @tool 데코레이터를 임포트
from strands import Agent, tool
# Amazon Bedrock 모델을 사용하기 위한 모델 클래스 임포트
from strands.models import BedrockModel


# === 시스템 프롬프트 ===
# 에이전트의 역할, 규칙, 제약사항을 정의하는 텍스트
# LLM이 이 프롬프트를 기반으로 응답의 톤과 범위를 결정합니다
MEDICAL_SYSTEM_PROMPT = """
당신은 대한민국의 AI 건강 상담 보조원입니다.

## 역할
- 환자의 증상을 듣고 일반적인 건강 정보를 제공합니다.
- 건강검진 결과를 설명하고 후속 조치를 안내합니다.

## 반드시 지켜야 할 규칙
1. 확정적 진단을 내리지 마세요. ("~일 수 있습니다", "~가능성이 있습니다" 표현 사용)
2. 약물을 처방하지 마세요. 의료기관 방문을 안내하세요.
3. 모든 응답 마지막에 아래 면책 조항을 포함하세요:
   "※ 본 정보는 참고용이며, 정확한 진단과 치료는 의료 전문가와 상담하시기 바랍니다."
4. 응급 증상(흉통, 호흡곤란, 의식저하 등) 감지 시 즉시 119 또는 응급실 방문을 안내하세요.

## 한국 건강검진 판정 기준
- A: 정상
- B: 경계 수준 (생활습관 개선 필요)
- C: 질환 의심 (추가 검사 또는 치료 필요)
- D: 유질환자 (치료 및 관리 필요)
- R: 판정보류 (추가 검사 필요)
"""


# === 도구(Tool) 구현 ===
# @tool 데코레이터: Python 함수를 에이전트가 호출할 수 있는 도구로 변환
# LLM은 이 함수의 docstring을 읽고, 언제 호출할지 스스로 판단합니다
@tool
def search_medical_knowledge(query: str, category: str = "general") -> str:
    """의료 지식 데이터베이스에서 관련 정보를 검색합니다.
    
    Args:
        query: 검색할 의료 키워드 또는 증상
        category: 검색 카테고리 (general, lab_values, symptoms, emergency)
    
    Returns:
        관련 의료 정보 텍스트
    """
    # 한국 건강검진 기준 기반 지식 베이스 (실제 서비스에서는 DB/API 호출로 대체)
    knowledge_base = {
        "두통": {
            "common_causes": "긴장성 두통, 편두통, 수면 부족, 스트레스, 고혈압",
            "warning_signs": "갑작스러운 극심한 두통, 발열 동반, 의식 변화, 시력 장애",
            "recommendation": "2주 이상 지속 시 신경과 진료 권장"
        },
        "WBC": {
            "normal_range": "4.0-10.0 (10³/μL)",
            "high_causes": "감염, 염증, 스트레스, 흡연, 백혈병(드문 경우)",
            "grade_B": "10.1-15.0: 경계 수준, 추적 검사 권장",
            "grade_C": "15.1-30.0: 추가 검사 필요 (CRP, 감별진단)"
        },
        "콜레스테롤": {
            "total_normal": "<200 mg/dL",
            "LDL_normal": "<130 mg/dL",
            "HDL_normal": "≥40 mg/dL (남성)",
            "management": "식이 조절, 운동, 6개월 후 재검사. 필요 시 내과 진료"
        },
        "혈당": {
            "fasting_normal": "<100 mg/dL",
            "prediabetes": "100-125 mg/dL (공복혈당장애)",
            "diabetes_suspect": "≥126 mg/dL (당뇨 의심, 재검사 필요)",
            "management": "식이 조절 + 운동, HbA1c 추가 검사 권장"
        },
        "흉통": {
            "emergency": True,
            "action": "즉시 119에 연락하거나 가까운 응급실을 방문하세요.",
            "warning": "심근경색, 협심증 가능성 배제 필요"
        }
    }
    
    # 간단한 키워드 매칭으로 관련 정보 검색
    results = []
    for key, info in knowledge_base.items():
        if key in query or query in key:
            results.append(f"[{key}] {info}")
    
    if not results:
        return f"'{query}'에 대한 정보를 찾을 수 없습니다. 일반적인 건강 정보를 제공합니다."
    
    return "\n".join(str(r) for r in results)


@tool
def assess_urgency(symptoms: str, duration_days: int = 0) -> str:
    """증상의 긴급도를 평가합니다.
    
    Args:
        symptoms: 환자가 보고한 증상 (쉼표로 구분)
        duration_days: 증상 지속 일수
    
    Returns:
        긴급도 평가 결과 (긴급/주의/일반)
    """
    # 응급 증상 키워드 — 이 증상이 포함되면 즉시 응급 안내
    emergency_keywords = ["흉통", "가슴 통증", "호흡곤란", "숨이 차", "의식 소실",
                          "마비", "극심한 두통", "대량 출혈", "경련"]
    # 주의 증상 키워드 — 빠른 시일 내 의료기관 방문 권장
    caution_keywords = ["고열", "38도 이상", "지속되는 통증", "체중 감소",
                        "혈뇨", "혈변", "지속 구토"]
    
    symptoms_lower = symptoms.lower()
    
    # 1단계: 응급 증상 판별
    for keyword in emergency_keywords:
        if keyword in symptoms_lower:
            return (
                "🚨 [긴급] 응급 증상이 감지되었습니다.\n"
                f"감지된 증상: {keyword}\n"
                "즉시 119에 연락하거나 가까운 응급실을 방문하세요.\n"
                "- 서울: 서울대학교병원 응급센터 (02-2072-2222)\n"
                "- 전국 응급의료정보센터: 1339"
            )
    
    # 2단계: 주의 증상 판별
    for keyword in caution_keywords:
        if keyword in symptoms_lower:
            return (
                "⚠️ [주의] 의료기관 방문을 권장합니다.\n"
                f"감지된 증상: {keyword}\n"
                "가까운 시일 내 관련 진료과 방문을 권장합니다."
            )
    
    # 3단계: 장기 지속 여부 확인
    if duration_days > 14:
        return (
            "ℹ️ [일반 - 장기 지속] 증상이 2주 이상 지속되고 있습니다.\n"
            "자연 호전되지 않는 경우 의료기관 방문을 권장합니다."
        )
    
    # 4단계: 일반 (긴급하지 않음)
    return (
        "ℹ️ [일반] 긴급한 상황은 아닌 것으로 판단됩니다.\n"
        "증상이 악화되거나 새로운 증상이 나타나면 의료기관을 방문하세요."
    )


# === 에이전트 생성 ===
# TODO: 아래 빈칸을 채워서 에이전트를 완성하세요
# - tools: 위에서 정의한 도구 함수들을 리스트로 전달
# - system_prompt: 위에서 정의한 시스템 프롬프트 변수를 전달
consultation_agent = Agent(
    model=BedrockModel(
        model_id="________",  # TODO: Claude Sonnet 4.5 글로벌 추론 프로파일의 Model ID를 채우세요
        region_name="us-west-2"  # API 호출 시작 리전 (글로벌 프로파일이 최적 리전으로 자동 라우팅)
    ),
    tools=[________, ________],          # TODO: 위에서 정의한 도구 함수 2개를 채우세요
    system_prompt=________         # TODO: 위에서 정의한 시스템 프롬프트 변수명을 채우세요
)


# === 대화형 테스트 ===
# 이 파일을 직접 실행할 때만 대화 루프가 시작됩니다
if __name__ == "__main__":
    print("=" * 60)
    print("  Healthcare Consultation Agent (끝내시려면 quit, exit 또는 종료)")
    print("=" * 60)
    
    while True:
        user_input = input("\n환자: ")
        if user_input.lower() in ["quit", "exit", "종료"]:
            print("상담을 종료합니다.")
            break
        
        # 에이전트 호출: 내부적으로 Agent Loop가 실행됨
        # 1. LLM이 입력을 분석하고 필요한 도구를 선택
        # 2. 도구 실행 결과를 LLM에 전달
        # 3. LLM이 최종 응답 생성 (또는 추가 도구 호출)
        response = consultation_agent(user_input)
        print(f"\nAI 상담원: {________.message['content'][0]['text']}\n")  # TODO: 응답 변수를 채우세요
```

### Step 3: 에이전트 실행 및 테스트

```bash
cd ~/agentcore/src
uv run python consultation_agent.py
```

---

## 테스트 시나리오

아래 3개 시나리오를 순서대로 입력하여 에이전트 동작을 확인하세요.

### 시나리오 1: 일반 증상 상담

```
3일 전부터 두통이 있고 피로감을 느끼고 있습니다
```

**확인 포인트:**
- [ ] `search_medical_knowledge` 도구가 호출되었는가?
- [ ] 두통의 가능한 원인이 안내되었는가?
- [ ] 면책 조항이 포함되었는가?

### 시나리오 2: 긴급 증상 감지

```
갑자기 왼쪽 가슴이 아프고 숨이 차요
```

**확인 포인트:**
- [ ] `assess_urgency` 도구가 호출되었는가?
- [ ] "긴급" 판정이 나왔는가?
- [ ] 119 또는 응급실 안내가 포함되었는가?

### 시나리오 3: 검사 결과 문의

```
WBC가 12,500으로 나왔는데 정상인가요?
```

**확인 포인트:**
- [ ] `search_medical_knowledge` 에서 WBC 정보가 반환되었는가?
- [ ] "경계 수준(B등급)"으로 안내되었는가?
- [ ] 확정적 진단 없이 추적 검사를 권장하는가?

---

## 검증

모든 시나리오에서 아래 조건을 만족하면 Phase 1-A가 완료된 것입니다:

- ✅ 에이전트가 도구를 적절히 호출함
- ✅ 응급 상황을 정확히 감지함
- ✅ 확정적 진단을 하지 않음
- ✅ 모든 응답에 면책 조항이 포함됨

---

## 트러블슈팅

| 증상 | 해결 |
|------|------|
| `ModuleNotFoundError: No module named 'strands'` | `source ~/agentcore/.venv/bin/activate` 실행 |
| `AccessDeniedException` on Bedrock | IAM Role 확인 — 강사에게 문의 |
| 도구가 호출되지 않음 | `@tool` 데코레이터의 docstring 확인 (LLM이 설명을 읽고 판단) |
| 응답이 영어로 나옴 | 시스템 프롬프트에 "한국어로 응답" 추가 |

---

## 🏆 Challenge Task (시간 여유 시)

1. **에이전트 동작 로깅 추가**: 클라이언트 요청과 에이전트 응답 사이에 어떤 도구가 호출되었는지, 소요 시간은 얼마인지를 로그로 출력하는 코드를 추가하세요.

   **로깅 코드:**
   ```python
   import time

   start = time.time()
   response = consultation_agent(user_input)
   elapsed = time.time() - start

   # 에이전트 실행 메트릭 출력
   print(f"[LOG] Duration: {elapsed:.2f}s | Metrics: {response.metrics}")
   ```

   **힌트**: 로깅 코드는 대화형 테스트 루프(`while True`) 안에서 `response = consultation_agent(user_input)` 윗 부분을 위 코드로 교체하세요.

2. **시스템 프롬프트 추가**: 환자의 연령대에 맞는 어투 조절 규칙을 추가하세요.

---

완료 후 [Phase 1-B: AgentCore Runtime 배포](./022_runtime_deploy.md)로 이동하세요.

---

## 부록: 정답 코드

<details>
<summary>consultation_agent.py 정답 코드 확인 (클릭하여 펼치기)</summary>

```python
"""
Healthcare Consultation Agent
- 환자 증상 기반 초기 상담
- 대한민국 의료법 제27조 준수
"""

# Strands Agents SDK에서 Agent 클래스와 @tool 데코레이터를 임포트
from strands import Agent, tool
# Amazon Bedrock 모델을 사용하기 위한 모델 클래스 임포트
from strands.models import BedrockModel


# === 시스템 프롬프트 ===
MEDICAL_SYSTEM_PROMPT = """
당신은 대한민국의 AI 건강 상담 보조원입니다.

## 역할
- 환자의 증상을 듣고 일반적인 건강 정보를 제공합니다.
- 건강검진 결과를 설명하고 후속 조치를 안내합니다.

## 반드시 지켜야 할 규칙
1. 확정적 진단을 내리지 마세요. ("~일 수 있습니다", "~가능성이 있습니다" 표현 사용)
2. 약물을 처방하지 마세요. 의료기관 방문을 안내하세요.
3. 모든 응답 마지막에 아래 면책 조항을 포함하세요:
   "※ 본 정보는 참고용이며, 정확한 진단과 치료는 의료 전문가와 상담하시기 바랍니다."
4. 응급 증상(흉통, 호흡곤란, 의식저하 등) 감지 시 즉시 119 또는 응급실 방문을 안내하세요.

## 한국 건강검진 판정 기준
- A: 정상
- B: 경계 수준 (생활습관 개선 필요)
- C: 질환 의심 (추가 검사 또는 치료 필요)
- D: 유질환자 (치료 및 관리 필요)
- R: 판정보류 (추가 검사 필요)
"""


# === 도구(Tool) 구현 ===
@tool
def search_medical_knowledge(query: str, category: str = "general") -> str:
    """의료 지식 데이터베이스에서 관련 정보를 검색합니다.
    
    Args:
        query: 검색할 의료 키워드 또는 증상
        category: 검색 카테고리 (general, lab_values, symptoms, emergency)
    
    Returns:
        관련 의료 정보 텍스트
    """
    knowledge_base = {
        "두통": {
            "common_causes": "긴장성 두통, 편두통, 수면 부족, 스트레스, 고혈압",
            "warning_signs": "갑작스러운 극심한 두통, 발열 동반, 의식 변화, 시력 장애",
            "recommendation": "2주 이상 지속 시 신경과 진료 권장"
        },
        "WBC": {
            "normal_range": "4.0-10.0 (10³/μL)",
            "high_causes": "감염, 염증, 스트레스, 흡연, 백혈병(드문 경우)",
            "grade_B": "10.1-15.0: 경계 수준, 추적 검사 권장",
            "grade_C": "15.1-30.0: 추가 검사 필요 (CRP, 감별진단)"
        },
        "콜레스테롤": {
            "total_normal": "<200 mg/dL",
            "LDL_normal": "<130 mg/dL",
            "HDL_normal": "≥40 mg/dL (남성)",
            "management": "식이 조절, 운동, 6개월 후 재검사. 필요 시 내과 진료"
        },
        "혈당": {
            "fasting_normal": "<100 mg/dL",
            "prediabetes": "100-125 mg/dL (공복혈당장애)",
            "diabetes_suspect": "≥126 mg/dL (당뇨 의심, 재검사 필요)",
            "management": "식이 조절 + 운동, HbA1c 추가 검사 권장"
        },
        "흉통": {
            "emergency": True,
            "action": "즉시 119에 연락하거나 가까운 응급실을 방문하세요.",
            "warning": "심근경색, 협심증 가능성 배제 필요"
        }
    }
    
    results = []
    for key, info in knowledge_base.items():
        if key in query or query in key:
            results.append(f"[{key}] {info}")
    
    if not results:
        return f"'{query}'에 대한 정보를 찾을 수 없습니다. 일반적인 건강 정보를 제공합니다."
    
    return "\n".join(str(r) for r in results)


@tool
def assess_urgency(symptoms: str, duration_days: int = 0) -> str:
    """증상의 긴급도를 평가합니다.
    
    Args:
        symptoms: 환자가 보고한 증상 (쉼표로 구분)
        duration_days: 증상 지속 일수
    
    Returns:
        긴급도 평가 결과 (긴급/주의/일반)
    """
    emergency_keywords = ["흉통", "가슴 통증", "호흡곤란", "숨이 차", "의식 소실",
                          "마비", "극심한 두통", "대량 출혈", "경련"]
    caution_keywords = ["고열", "38도 이상", "지속되는 통증", "체중 감소",
                        "혈뇨", "혈변", "지속 구토"]
    
    symptoms_lower = symptoms.lower()
    
    for keyword in emergency_keywords:
        if keyword in symptoms_lower:
            return (
                "🚨 [긴급] 응급 증상이 감지되었습니다.\n"
                f"감지된 증상: {keyword}\n"
                "즉시 119에 연락하거나 가까운 응급실을 방문하세요.\n"
                "- 서울: 서울대학교병원 응급센터 (02-2072-2222)\n"
                "- 전국 응급의료정보센터: 1339"
            )
    
    for keyword in caution_keywords:
        if keyword in symptoms_lower:
            return (
                "⚠️ [주의] 의료기관 방문을 권장합니다.\n"
                f"감지된 증상: {keyword}\n"
                "가까운 시일 내 관련 진료과 방문을 권장합니다."
            )
    
    if duration_days > 14:
        return (
            "ℹ️ [일반 - 장기 지속] 증상이 2주 이상 지속되고 있습니다.\n"
            "자연 호전되지 않는 경우 의료기관 방문을 권장합니다."
        )
    
    return (
        "ℹ️ [일반] 긴급한 상황은 아닌 것으로 판단됩니다.\n"
        "증상이 악화되거나 새로운 증상이 나타나면 의료기관을 방문하세요."
    )


# === 에이전트 생성 ===
consultation_agent = Agent(
    model=BedrockModel(
        model_id="global.anthropic.claude-sonnet-4-5-20250929-v1:0",
        region_name="us-west-2"
    ),
    tools=[search_medical_knowledge, assess_urgency],
    system_prompt=MEDICAL_SYSTEM_PROMPT
)


# === 대화형 테스트 ===
if __name__ == "__main__":
    print("=" * 60)
    print("  Healthcare Consultation Agent (끝내시려면 quit, exit 또는 종료)")
    print("=" * 60)
    
    while True:
        user_input = input("\n환자: ")
        if user_input.lower() in ["quit", "exit", "종료"]:
            print("상담을 종료합니다.")
            break
        
        import time
        start = time.time()
        response = consultation_agent(user_input)
        elapsed = time.time() - start

        print(f"\nAI 상담원: {response.message['content'][0]['text']}\n")
        # 에이전트 실행 메트릭 출력
        print(f"[LOG] Duration: {elapsed:.2f}s | Metrics: {response.metrics}")
```

</details>
