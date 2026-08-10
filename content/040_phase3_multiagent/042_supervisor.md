---
title: "Phase 3-B: Supervisor Agent 구현"
weight: 42
time: "50분"
---

# Phase 3-B: Supervisor Agent 구현 (50분)

## 학습 목표

Agent-as-Tool 패턴으로 Supervisor Agent가 3개 전문 에이전트를 순차적으로 조율하여 종합 건강검진 AI 분석 보고서를 생성합니다.

---

## 이론: Supervisor 오케스트레이션 (10분 브리핑)

### Supervisor 패턴이란?

Supervisor Agent는 **지휘자(Conductor)** 역할을 합니다:
- 어떤 전문 에이전트를 호출할지 결정
- 호출 순서와 데이터 흐름을 조율
- 각 에이전트의 결과를 종합하여 최종 보고서 생성

```
┌─────────────────────────────────────────────┐
│              Supervisor Agent                │
│      (Health Checkup Coordinator)           │
│                                             │
│  "환자 김민수의 검진 결과를 분석해 주세요"    │
│                                             │
│  1. triage_specialist 호출                  │
│     → 분류 결과 수신                         │
│  2. analysis_specialist 호출                │
│     → 심층 분석 결과 수신                    │
│  3. recommendation_specialist 호출          │
│     → 건강 관리 계획 수신                    │
│  4. 종합 보고서 작성                         │
└─────────────────────────────────────────────┘
```

### Agent-as-Tool 구현 방식

각 전문 에이전트를 `@tool` 함수로 래핑합니다:

```python
@tool
def triage_specialist(request: str) -> str:
    """Triage 전문가에게 분류를 요청합니다."""
    response = triage_agent(request)
    return response.message['content'][0]['text']
```

Supervisor 입장에서는 일반 도구와 동일하게 호출합니다. LLM이 상황에 맞는 전문가를 선택합니다.

### Supervisor vs 직접 호출

| 항목 | 하드코딩 순차 호출 | Supervisor 패턴 |
|------|-------------------|----------------|
| 유연성 | 순서 고정 | LLM이 상황에 맞게 결정 |
| 에러 처리 | 개발자가 분기 작성 | LLM이 실패 시 재시도/우회 |
| 확장성 | 새 에이전트 추가 시 코드 수정 | 도구만 추가하면 됨 |
| 적합한 경우 | 워크플로우가 고정된 경우 | 동적 판단이 필요한 경우 |

---

## 실습 시작

### Step 1: Supervisor Agent 코드 작성

```bash
cd ~/agentcore/src
touch supervisor_agent.py
```

`supervisor_agent.py`를 열고 아래 코드를 작성하세요:

```python
"""
Supervisor Agent — 종합 건강검진 분석 코디네이터
- 3개 전문 에이전트를 Agent-as-Tool 패턴으로 조율
- 최종 종합 보고서 생성
"""
from strands import Agent, tool
from strands.models import BedrockModel

# Phase 3-A에서 만든 전문 에이전트 import
from triage_agent import triage_agent
from analysis_agent import analysis_agent
from recommendation_agent import recommendation_agent


# ─────────────────────────────────────────────
# 전문 에이전트를 @tool로 래핑 (Agent-as-Tool)
# ─────────────────────────────────────────────

@tool
def triage_specialist(patient_data: str) -> str:
    """Triage 전문가에게 건강검진 결과 분류를 요청합니다.
    
    Args:
        patient_data: 분류를 요청할 환자 정보 또는 검진 데이터
    
    Returns:
        판정 등급, 비정상 항목 목록, 우선순위
    """
    # TODO ①: triage_agent를 호출하고 응답 텍스트를 반환하세요
    response = ________
    return ________


@tool
def analysis_specialist(analysis_request: str) -> str:
    """임상병리 분석 전문가에게 심층 분석을 요청합니다.
    
    Args:
        analysis_request: 분석을 요청할 비정상 항목 또는 상세 요청
    
    Returns:
        원인 분석, 상관관계, 추가 검사 권고
    """
    # TODO ②: analysis_agent를 호출하고 응답 텍스트를 반환하세요
    response = ________
    return ________


@tool
def recommendation_specialist(recommendation_request: str) -> str:
    """건강관리 전문가에게 종합 권고사항 생성을 요청합니다.
    
    Args:
        recommendation_request: 건강 관리 계획 수립에 필요한 정보
    
    Returns:
        식이/운동/생활습관 권고 + 추적 검사 일정
    """
    # TODO ③: recommendation_agent를 호출하고 응답 텍스트를 반환하세요
    response = ________
    return ________


# ─────────────────────────────────────────────
# Supervisor Agent 정의
# ─────────────────────────────────────────────

SUPERVISOR_SYSTEM_PROMPT = """
당신은 종합 건강검진 분석 코디네이터(Health Checkup Coordinator)입니다.

## 역할
환자의 건강검진 데이터를 받아 3명의 전문가에게 순차적으로 분석을 의뢰하고,
그 결과를 종합하여 최종 보고서를 작성합니다.

## 워크플로우
1. **triage_specialist**: 검진 결과 분류 및 우선순위 결정
2. **analysis_specialist**: 비정상 항목 심층 분석 (상관관계 + 추가 검사 권고)
3. **recommendation_specialist**: 개인화 건강 관리 계획 수립

## 보고서 형식
최종 보고서는 아래 구조로 작성하세요:

```
# 종합 건강검진 AI 분석 보고서

## 1. 환자 정보
## 2. 검진 결과 분류 (Triage)
## 3. 심층 분석 (Analysis)
## 4. 건강 관리 권고 (Recommendation)
## 5. 종합 소견

※ 본 보고서는 AI가 생성한 참고 자료이며,
  정확한 진단과 치료는 의료 전문가와 상담하시기 바랍니다.
```

## 규칙
- 반드시 3명의 전문가를 모두 호출하세요
- 확정적 진단을 내리지 마세요
- 보고서 마지막에 면책 조항을 포함하세요
"""

# TODO ④: Supervisor Agent를 생성하세요
supervisor_agent = Agent(
    model=BedrockModel(
        model_id="________",
        region_name="us-west-2"
    ),
    tools=[________],  # TODO: 3개 전문가 도구
    system_prompt=________
)


# ─────────────────────────────────────────────
# 대화형 테스트
# ─────────────────────────────────────────────
if __name__ == "__main__":
    print("=" * 60)
    print("  종합 건강검진 AI 분석 시스템 (Supervisor Agent)")
    print("  끝내시려면 quit, exit 또는 종료")
    print("=" * 60)
    
    while True:
        user_input = input("\n요청: ")
        if user_input.lower() in ["quit", "exit", "종료"]:
            break
        
        import time
        start = time.time()
        response = supervisor_agent(user_input)
        elapsed = time.time() - start
        
        print(f"\n{'='*60}")
        print(response.message['content'][0]['text'])
        print(f"\n[소요 시간: {elapsed:.1f}초]")
```

---

### Step 2: Supervisor Agent 테스트

```bash
cd ~/agentcore/src
uv run python supervisor_agent.py
```

아래 프롬프트를 입력하세요:

```
patient-001(김민수, 45세 남성)의 종합 건강검진 결과를 분석해 주세요.
```

**기대 동작:**
1. Supervisor가 `triage_specialist` 호출 → 분류 결과 수신
2. Supervisor가 `analysis_specialist` 호출 → 심층 분석 수신
3. Supervisor가 `recommendation_specialist` 호출 → 권고사항 수신
4. 최종 종합 보고서 출력

---

### Step 3: 보고서 품질 확인

생성된 보고서에서 다음 항목을 확인하세요:

| # | 확인 항목 | 기대 결과 |
|---|----------|----------|
| 1 | 비정상 항목 수 | 5개 (WBC, 혈당, 콜레스테롤, LDL, 중성지방) |
| 2 | 종합 판정 등급 | B등급 |
| 3 | 상관관계 분석 | 대사증후군 위험 패턴 언급 |
| 4 | 식이 권고 | 한국 식문화 반영 (나물, 등푸른 생선 등) |
| 5 | 운동 권고 | 구체적 빈도/시간 (주 3-4회, 30-40분) |
| 6 | 추적 일정 | 3개월 후 재검 권고 |
| 7 | 면책 조항 | 보고서 마지막에 포함 |

---

## 검증

- [ ] Supervisor가 3개 전문 에이전트를 모두 호출함
- [ ] 종합 보고서 형식에 맞게 출력됨
- [ ] 한국 건강검진 기준에 맞는 분석 결과
- [ ] 면책 조항 포함

---

## 🏆 Challenge Task

1. Supervisor에 "요약 모드" 옵션을 추가하여, 간략한 1페이지 요약 보고서를 생성하도록 하세요
2. 여러 환자(patient-001, patient-002)를 순차 분석하는 배치 모드를 구현하세요

---

완료 후 [Phase 4-A: 보안 강화](../050_phase4_security/051_guardrails_policy.md)로 이동하세요.
