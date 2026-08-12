
# Phase 3-A: 전문 에이전트 구현 (90분)

## 학습 목표

3개의 전문 에이전트(Triage, Analysis, Recommendation)를 구현하여 종합건강검진 분석의 각 단계를 담당하게 합니다.

---

## 이론: 멀티 에이전트 패턴 (20분 브리핑)

### 왜 멀티 에이전트가 필요한가?

싱글 에이전트의 한계는 **도구 수 증가에 따른 성능 저하**로 나타납니다:

**문제 1: Tool Selection Accuracy 하락**

LLM이 선택해야 할 도구가 많아질수록 정확도가 떨어집니다. 연구에 따르면 도구가 10개를 넘어가면 선택 정확도가 급격히 하락합니다.

```
도구 3개:  정확도 ~95%   ← Phase 1 (search, urgency, lab)
도구 7개:  정확도 ~85%   ← Phase 2 (+ analyze, memory, audit)
도구 15개: 정확도 ~65%   ← 모든 기능을 한 에이전트에 넣으면
```

**문제 2: 시스템 프롬프트 비대화**

하나의 에이전트에 모든 역할을 부여하면:
- Triage 규칙 + Analysis 기준 + Recommendation 가이드라인 + 보안 규칙 = 수천 토큰
- 프롬프트가 길어질수록 LLM의 지시 준수율(instruction following) 저하
- 상충하는 규칙 간 충돌 가능 (예: "상세히 분석하라" vs "간결하게 답하라")

**문제 3: 테스트/디버깅 어려움**

모놀리식 에이전트에서는:
- "분석이 잘못된 건 Triage 로직 때문인지, Analysis 로직 때문인지" 분리 불가
- 하나를 수정하면 다른 기능에 영향 (regression)
- 도구별 독립적 단위 테스트 불가

**멀티 에이전트로 해결:**

| 싱글 에이전트 문제 | 멀티 에이전트 해결 |
|-------------------|------------------|
| 도구 15개 → 선택 정확도 65% | 에이전트당 2-3개 도구 → 95% 유지 |
| 시스템 프롬프트 3,000 토큰 | 에이전트당 500 토큰 (역할 집중) |
| 에러 원인 추적 불가 | 에이전트별 독립 로그 + 개별 테스트 |
| 하나의 변경 → 전체 영향 | 에이전트별 독립 배포/롤백 가능 |

### Agent-as-Tool 패턴

Strands Agents에서 멀티 에이전트를 구현하는 가장 간단한 패턴입니다:

```
Supervisor Agent
  ├── @tool triage_specialist()      → Triage Agent 호출
  ├── @tool analysis_specialist()    → Analysis Agent 호출
  └── @tool recommendation_specialist() → Recommendation Agent 호출
```

각 전문 에이전트를 `@tool` 함수로 래핑하면, Supervisor가 일반 도구처럼 호출할 수 있습니다.

### 멀티 에이전트 오케스트레이션 패턴 비교

멀티 에이전트를 조율하는 방식은 크게 3가지가 있습니다:

| 패턴 | 구조 | 특징 | 적합한 경우 |
|------|------|------|-----------|
| **Graph** | 노드(에이전트) + 엣지(데이터 흐름)를 DAG로 정의 | 실행 순서가 명시적, 조건부 분기 가능 | 워크플로우가 고정된 파이프라인 (예: ETL, 순차 분석) |
| **Swarm** | 에이전트들이 자율적으로 핸드오프 (다음 에이전트를 스스로 결정) | 중앙 조율자 없음, 에이전트 간 직접 위임 | 동적 라우팅, 고객 지원 (상담 → 기술 → 결제 전환) |
| **Supervisor (본 워크샵)** | 중앙 Supervisor가 전문 에이전트를 도구로 호출 | LLM이 호출 순서 결정, 단순 구현 | 전문가 협업, 분석 보고서 생성 |

```
[Graph 패턴]                    [Swarm 패턴]                [Supervisor 패턴]

A → B → C                      A ←→ B ←→ C              Supervisor
    ↘ D ↗                      (자율 핸드오프)              ├→ A
(DAG 고정 흐름)                 (탈중앙)                    ├→ B
                                                            └→ C
                                                          (중앙 조율)
```

**Graph**: LangGraph에서 주로 사용. 노드 간 엣지를 명시적으로 정의하여 실행 순서를 제어합니다. 조건부 분기(`if 긴급 → 응급경로`)가 가능하지만 유연성이 제한됩니다.

```python
from strands import Agent
from strands.models import BedrockModel
from strands.multiagent.graph import GraphBuilder

# 에이전트 정의
triage = Agent(model=BedrockModel(...), system_prompt="분류 전문가")
analysis = Agent(model=BedrockModel(...), system_prompt="분석 전문가")
recommendation = Agent(model=BedrockModel(...), system_prompt="권고 전문가")

# Graph 빌드 (DAG 구조 — 실행 순서를 엣지로 명시)
builder = GraphBuilder()
triage_node = builder.add_node(triage, node_id="triage")
analysis_node = builder.add_node(analysis, node_id="analysis")
rec_node = builder.add_node(recommendation, node_id="recommendation")

builder.add_edge(triage_node, analysis_node)     # triage → analysis
builder.add_edge(analysis_node, rec_node)        # analysis → recommendation
builder.set_entry_point("triage")
builder.set_max_node_executions(10)

graph = builder.build()
result = graph("patient-001의 건강검진 결과를 분석해 주세요")
```

**Swarm**: OpenAI Swarm, Strands Swarm에서 지원. 현재 에이전트가 "다음은 B가 처리해야 해"라고 판단하면 자동 전환됩니다. 중앙 조율자 없이 에이전트끼리 직접 대화합니다.

```python
from strands import Agent
from strands.models import BedrockModel
from strands.multiagent.swarm import Swarm

# 에이전트 — 시스템 프롬프트에서 핸드오프 대상을 지시
triage = Agent(model=BedrockModel(...),
    system_prompt="분류 전문가. 완료 후 'analysis'에게 핸드오프하세요.")
analysis = Agent(model=BedrockModel(...),
    system_prompt="분석 전문가. 완료 후 'recommendation'에게 핸드오프하세요.")
recommendation = Agent(model=BedrockModel(...),
    system_prompt="건강관리 전문가. 최종 권고를 생성하고 종료하세요.")

# Swarm — 에이전트들이 자율적으로 핸드오프
swarm = Swarm(
    nodes=[triage, analysis, recommendation],
    entry_point=triage,
    max_handoffs=10,
    execution_timeout=300.0
)

result = swarm("patient-001의 건강검진 결과를 종합 분석해 주세요")
```

**Supervisor (Agent-as-Tool)**: 본 워크샵에서 사용하는 패턴. 구현이 간단하고, Supervisor의 시스템 프롬프트로 호출 순서를 유연하게 제어할 수 있습니다.

> **본 워크샵에서 Supervisor 패턴을 선택한 이유:**
> - 구현 복잡도가 낮아 워크샵 시간 내 완성 가능
> - 종합 보고서 생성에 적합 (순차적 전문가 의견 수집)
> - Strands SDK의 `@tool` 데코레이터만으로 구현 가능

### 본 워크샵의 에이전트 구성

| 에이전트 | 역할 | 입력 | 출력 |
|----------|------|------|------|
| **Triage Agent** | 검진 결과 분류 + 우선순위 결정 | 환자 검진 데이터 | 판정 등급, 비정상 항목 목록, 우선순위 |
| **Analysis Agent** | 패널별 심층 분석 | 비정상 항목 | 원인 분석, 상관관계, 추가 검사 권고 |
| **Recommendation Agent** | 개인화 건강 관리 계획 | 분석 결과 + 환자 정보 | 식이/운동/생활습관 권고 |

### 워크플로우

```
환자 김민수 건강검진 데이터
        │
        ▼
┌─────────────────┐
│  Triage Agent   │  → "B등급, 비정상 5개 항목 (WBC↑, 혈당↑, 콜레스테롤↑, LDL↑, TG↑)"
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Analysis Agent  │  → "WBC↑: 감염/스트레스 가능, 대사증후군 위험 패턴 감지"
└────────┬────────┘
         │
         ▼
┌──────────────────────┐
│ Recommendation Agent │  → "식이: 저염식, 운동: 주 3회 유산소, 3개월 후 재검"
└──────────────────────┘
```

---

## 실습 시작

> **실습 방식**: 핵심 로직 부분만 빈칸으로 된 스켈레톤 코드를 제공합니다.
> `TODO` 주석이 있는 부분만 채워넣으면 동작합니다.

---

### Step 1: Triage Agent 구현

검진 결과를 한국 건강검진 기준으로 분류하고 우선순위를 결정합니다.

```bash
cd ~/agentcore/src
touch triage_agent.py
```

`triage_agent.py`를 열고 아래 코드를 작성하세요:

```python
"""
Triage Agent — 건강검진 결과 분류 및 우선순위 결정
- 한국 국민건강보험공단 건강검진 판정 기준 적용
"""
from strands import Agent, tool
from strands.models import BedrockModel
from lab_tools import get_lab_results, analyze_lab_values

# 한국 건강검진 판정 기준
HEALTH_GRADES = {
    "A": "정상",
    "B": "경계 수준 (생활습관 개선 필요)",
    "C": "질환 의심 (추가 검사 또는 치료 필요)",
    "D": "유질환자 (치료 및 관리 필요)",
    "R": "판정보류 (추가 검사 필요)"
}


@tool
def classify_health_risk(patient_id: str) -> str:
    """환자의 건강검진 결과를 위험도별로 분류합니다.
    
    Args:
        patient_id: 환자 ID
    
    Returns:
        위험도 분류 결과 (등급별 항목 목록)
    """
    # TODO ①: get_lab_results 도구를 호출하여 환자 데이터를 가져오세요
    raw_results = ________
    
    # TODO ②: analyze_lab_values 도구를 호출하여 분석 결과를 가져오세요
    analysis = ________
    
    return f"[분류 완료]\n{analysis}"


@tool
def prioritize_followup(analysis_result: str) -> str:
    """분석 결과를 바탕으로 후속 조치 우선순위를 결정합니다.
    
    Args:
        analysis_result: analyze_lab_values의 결과
    
    Returns:
        우선순위별 후속 조치 목록
    """
    priority_prompt = f"""
    아래 검진 분석 결과를 바탕으로 후속 조치 우선순위를 결정하세요.
    
    판정 기준:
    - 긴급 (D등급): 즉시 전문의 진료 필요
    - 높음 (C등급): 1개월 내 추가 검사 필요
    - 중간 (B등급): 3개월 내 생활습관 개선 후 재검
    - 낮음 (A등급): 정기 검진 유지
    
    분석 결과:
    {analysis_result}
    """
    return priority_prompt


# Triage Agent 시스템 프롬프트
TRIAGE_SYSTEM_PROMPT = """
당신은 건강검진 결과 분류 전문가(Triage Specialist)입니다.

## 역할
- 환자의 건강검진 결과를 한국 건강검진 판정 기준(A/B/C/D/R)으로 분류합니다.
- 비정상 항목을 식별하고 후속 조치의 우선순위를 결정합니다.

## 출력 형식
1. 종합 판정 등급
2. 비정상 항목 목록 (항목명, 수치, 등급)
3. 후속 조치 우선순위 (긴급/높음/중간/낮음)
"""

# TODO ③: Triage Agent를 생성하세요 (model, tools, system_prompt 지정)
triage_agent = Agent(
    model=BedrockModel(
        model_id="________",  # TODO: 글로벌 추론 프로파일 Model ID
        region_name="us-west-2"
    ),
    tools=[________, ________],  # TODO: 위에서 정의한 도구 2개
    system_prompt=________  # TODO: 시스템 프롬프트 변수명
)
```

---

### Step 2: Analysis Agent 구현

비정상 항목의 원인을 분석하고 항목 간 상관관계를 파악합니다.

```bash
touch ~/agentcore/src/analysis_agent.py
```

`analysis_agent.py`를 열고 아래 코드를 작성하세요:

```python
"""
Analysis Agent — 검진 결과 심층 분석
- 패널별 분석 (CBC, Lipid, Metabolic)
- 항목 간 상관관계 분석
"""
from strands import Agent, tool
from strands.models import BedrockModel


@tool
def analyze_correlation(abnormal_items: str) -> str:
    """비정상 항목 간의 상관관계를 분석합니다.
    
    Args:
        abnormal_items: 비정상 항목 목록 (쉼표 구분)
    
    Returns:
        상관관계 분석 결과
    """
    # 알려진 상관관계 패턴
    correlation_patterns = {
        ("WBC", "CRP"): "감염 또는 급성 염증 반응 가능성",
        ("LDL", "Triglyceride"): "대사증후군 위험 패턴 — 심혈관 질환 위험 상승",
        ("FastingGlucose", "TotalCholesterol"): "인슐린 저항성 가능성 — 당뇨/이상지질혈증 연관",
        ("FastingGlucose", "Triglyceride"): "대사증후군 진단 기준 2개 이상 해당",
    }
    
    items = [item.strip() for item in abnormal_items.split(",")]
    found_correlations = []
    
    for (item1, item2), description in correlation_patterns.items():
        if item1 in items and item2 in items:
            found_correlations.append(f"- {item1} + {item2}: {description}")
    
    # 단일 항목에 대한 항목이라도 대사증후군 패턴 체크
    metabolic_markers = ["FastingGlucose", "TotalCholesterol", "LDL", "Triglyceride"]
    metabolic_count = sum(1 for item in items if item in metabolic_markers)
    
    if metabolic_count >= 3:
        found_correlations.append(f"- 대사증후군 위험: 대사 관련 비정상 항목 {metabolic_count}개 감지")
    
    if not found_correlations:
        return "특별한 상관관계 패턴이 감지되지 않았습니다. 개별 항목 관리를 권장합니다."
    
    return "## 상관관계 분석 결과\n" + "\n".join(found_correlations)


@tool
def suggest_additional_tests(abnormal_items: str) -> str:
    """비정상 항목에 대해 추가로 필요한 검사를 제안합니다.
    
    Args:
        abnormal_items: 비정상 항목 목록
    
    Returns:
        추가 검사 권고 목록
    """
    additional_tests = {
        "WBC": ["CRP (C-반응성 단백)", "ESR (적혈구 침강속도)", "혈액 도말 검사"],
        "FastingGlucose": ["HbA1c (당화혈색소)", "경구당부하검사(OGTT)", "인슐린 농도"],
        "TotalCholesterol": ["Lp(a)", "ApoB", "심혈관 위험도 평가"],
        "LDL": ["small dense LDL", "관상동맥 CT", "경동맥 초음파"],
        "Triglyceride": ["간 초음파", "췌장 기능 검사", "지방간 평가"],
    }
    
    items = [item.strip() for item in abnormal_items.split(",")]
    suggestions = []
    
    for item in items:
        if item in additional_tests:
            tests = ", ".join(additional_tests[item])
            suggestions.append(f"- {item} 이상 → 추가 검사: {tests}")
    
    if not suggestions:
        return "현재 비정상 항목에 대한 추가 검사 권고 없음"
    
    return "## 추가 검사 권고\n" + "\n".join(suggestions)


ANALYSIS_SYSTEM_PROMPT = """
당신은 임상병리 분석 전문가(Clinical Pathology Analyst)입니다.

## 역할
- 비정상 검사 항목의 원인을 분석합니다.
- 항목 간 상관관계를 파악하여 종합적 임상 의미를 도출합니다.
- 추가 검사가 필요한 경우 구체적으로 제안합니다.

## 분석 원칙
- 단일 항목이 아닌 전체 패턴을 고려하세요
- 가능한 원인을 2-3가지 제시하되 확정 진단은 하지 마세요
- 한국 임상검사 기준을 적용하세요
"""

# TODO ④: Analysis Agent를 생성하세요
analysis_agent = Agent(
    model=BedrockModel(
        model_id="________",
        region_name="us-west-2"
    ),
    tools=[________, ________],  # TODO: 위에서 정의한 도구 2개
    system_prompt=________
)
```

---

### Step 3: Recommendation Agent 구현

분석 결과를 바탕으로 개인화된 건강 관리 계획을 수립합니다.

```bash
touch ~/agentcore/src/recommendation_agent.py
```

`recommendation_agent.py`를 열고 아래 코드를 작성하세요:

```python
"""
Recommendation Agent — 개인화 건강 관리 계획 수립
- 한국인 영양섭취기준(KDRIs) 기반
- 단계적 운동 계획
"""
from strands import Agent, tool
from strands.models import BedrockModel


@tool
def generate_diet_plan(conditions: str, patient_info: str) -> str:
    """환자 상태에 맞는 식이 계획을 생성합니다.
    
    Args:
        conditions: 진단된 상태 (예: "고콜레스테롤, 공복혈당장애")
        patient_info: 환자 기본 정보 (나이, 성별 등)
    
    Returns:
        개인화된 식이 권고
    """
    diet_guidelines = {
        "고콜레스테롤": {
            "권장": "귀리, 콩류, 등푸른 생선(고등어, 꽁치), 견과류, 올리브오일",
            "제한": "내장류, 새우·오징어(주 1회 이하), 가공육, 버터",
            "한국식 팁": "나물 반찬 ↑, 국물 요리 시 기름 제거, 들기름 활용"
        },
        "공복혈당장애": {
            "권장": "잡곡밥, 채소 먼저 섭취, 단백질 위주 간식",
            "제한": "흰쌀밥 단독 섭취, 과일주스, 떡·빵류 과다",
            "한국식 팁": "식사 순서: 나물 → 단백질 → 밥, 야식 자제"
        },
        "중성지방상승": {
            "권장": "오메가-3 (들기름, 생선), 식이섬유, 통곡물",
            "제한": "음주(주 2회 이하), 단순당, 정제 탄수화물",
            "한국식 팁": "회식 시 음주량 조절, 안주는 생선·두부 선택"
        }
    }
    
    result = "## 식이 권고\n\n"
    for condition in conditions.split(","):
        condition = condition.strip()
        if condition in diet_guidelines:
            info = diet_guidelines[condition]
            result += f"### {condition}\n"
            result += f"- 권장: {info['권장']}\n"
            result += f"- 제한: {info['제한']}\n"
            result += f"- 한국식 팁: {info['한국식 팁']}\n\n"
    
    return result if len(result) > 20 else "일반적인 균형 식단을 권장합니다."


@tool
def generate_exercise_plan(risk_level: str, age: int) -> str:
    """위험도와 연령에 맞는 운동 계획을 생성합니다.
    
    Args:
        risk_level: 위험도 (low, moderate, high)
        age: 환자 나이
    
    Returns:
        단계적 운동 계획
    """
    base_plans = {
        "low": {
            "주 빈도": "3-4회",
            "종류": "빠르게 걷기 30분 + 가벼운 근력운동",
            "강도": "대화 가능한 수준 (RPE 3-4/10)"
        },
        "moderate": {
            "주 빈도": "4-5회",
            "종류": "유산소 40분 (걷기/자전거) + 근력운동 2회",
            "강도": "약간 숨찬 수준 (RPE 5-6/10)"
        },
        "high": {
            "주 빈도": "5회 이상",
            "종류": "유산소 50분 + 근력운동 3회 + 유연성 운동",
            "강도": "중강도 (RPE 6-7/10), 전문가 지도 권장"
        }
    }
    
    plan = base_plans.get(risk_level, base_plans["moderate"])
    
    # 연령 보정
    age_note = ""
    if age >= 50:
        age_note = "\n- 주의: 관절 부담 최소화, 준비운동 10분 필수, 수영/자전거 권장"
    elif age >= 40:
        age_note = "\n- 참고: 근력 감소 예방을 위해 저항운동 포함 권장"
    
    return f"## 운동 계획 ({risk_level} 위험도, {age}세)\n" \
           f"- 빈도: {plan['주 빈도']}\n" \
           f"- 종류: {plan['종류']}\n" \
           f"- 강도: {plan['강도']}{age_note}"


@tool
def generate_followup_schedule(abnormal_count: int, max_grade: str) -> str:
    """추적 검사 일정을 생성합니다.
    
    Args:
        abnormal_count: 비정상 항목 수
        max_grade: 가장 높은 판정 등급 (A/B/C/D)
    
    Returns:
        추적 검사 일정 권고
    """
    schedules = {
        "D": "1개월 후 해당 항목 재검 + 전문의 진료",
        "C": "2개월 후 재검 + 필요 시 전문의 의뢰",
        "B": "3개월 후 생활습관 개선 효과 확인 재검",
        "A": "1년 후 정기 건강검진"
    }
    
    schedule = schedules.get(max_grade, schedules["B"])
    
    result = f"## 추적 검사 일정\n"
    result += f"- 비정상 항목: {abnormal_count}개\n"
    result += f"- 최고 판정 등급: {max_grade}\n"
    result += f"- 권고: {schedule}\n"
    
    if abnormal_count >= 3:
        result += f"- 추가 권고: 종합내과 방문하여 대사증후군 종합 평가 권장"
    
    return result


RECOMMENDATION_SYSTEM_PROMPT = """
당신은 예방의학 및 건강관리 전문가(Preventive Health Advisor)입니다.

## 역할
- 검진 분석 결과를 바탕으로 개인화된 건강 관리 계획을 수립합니다.
- 식이, 운동, 생활습관 개선 방안을 구체적으로 제시합니다.
- 추적 검사 일정을 제안합니다.

## 원칙
- 한국인 영양섭취기준(KDRIs) 및 한국 식문화를 반영하세요
- 실행 가능한 구체적 권고를 제시하세요 (예: "운동하세요" ❌ → "주 3회 30분 걷기" ✅)
- 단계적 접근: 갑작스러운 변화보다 점진적 개선 권장
- 약물 처방은 하지 마세요. 필요 시 의료기관 방문을 안내하세요.
"""

# TODO ⑤: Recommendation Agent를 생성하세요
recommendation_agent = Agent(
    model=BedrockModel(
        model_id="________",
        region_name="us-west-2"
    ),
    tools=[________, ________, ________],  # TODO: 위에서 정의한 도구 3개
    system_prompt=________
)
```

---

### Step 4: 개별 에이전트 테스트

각 에이전트를 독립적으로 테스트합니다:

```bash
cd ~/agentcore/src

# Triage Agent 테스트
uv run python -c "
from triage_agent import triage_agent
response = triage_agent('patient-001의 건강검진 결과를 분류해 주세요')
print(response.message['content'][0]['text'])
"

# Analysis Agent 테스트
uv run python -c "
from analysis_agent import analysis_agent
response = analysis_agent('WBC, FastingGlucose, TotalCholesterol, LDL, Triglyceride 항목이 비정상입니다. 상관관계를 분석해 주세요.')
print(response.message['content'][0]['text'])
"

# Recommendation Agent 테스트
uv run python -c "
from recommendation_agent import recommendation_agent
response = recommendation_agent('45세 남성, B등급, 고콜레스테롤+공복혈당장애+중성지방상승. 건강관리 계획을 수립해 주세요.')
print(response.message['content'][0]['text'])
"
```

---

## 검증

- [ ] Triage Agent: 환자 김민수의 비정상 5개 항목을 정확히 식별
- [ ] Analysis Agent: 상관관계 분석에서 "대사증후군 위험 패턴" 감지
- [ ] Recommendation Agent: 한국 식문화를 반영한 구체적 식이/운동 권고 생성
- [ ] 각 에이전트가 독립적으로 동작 확인

---

## 🏆 Challenge Task

1. Analysis Agent에 `analyze_cbc_panel`, `analyze_lipid_panel` 등 패널별 전용 분석 도구를 추가하세요
2. Recommendation Agent에 환자의 이전 상담 이력(Memory)을 참조하여 "이전에 권고한 운동을 실천했는지" 확인하는 로직을 추가하세요

---

완료 후 [Phase 3-B: Supervisor Agent 구현](./042_supervisor.md)로 이동하세요.

---

## 부록: 정답 코드

<details>
<summary>TODO ①~② 정답: Triage Agent 도구 호출 (클릭하여 펼치기)</summary>

```python
@tool
def classify_health_risk(patient_id: str) -> str:
    # TODO ①: get_lab_results 도구를 호출하여 환자 데이터를 가져오세요
    raw_results = get_lab_results(patient_id=patient_id, test_type="all")
    
    # TODO ②: analyze_lab_values 도구를 호출하여 분석 결과를 가져오세요
    analysis = analyze_lab_values(patient_id=patient_id)
    
    return f"[분류 완료]\n{analysis}"
```

</details>

<details>
<summary>TODO ③ 정답: Triage Agent 생성 (클릭하여 펼치기)</summary>

```python
triage_agent = Agent(
    model=BedrockModel(
        model_id="global.anthropic.claude-sonnet-4-5-20250929-v1:0",
        region_name="us-west-2"
    ),
    tools=[classify_health_risk, prioritize_followup],
    system_prompt=TRIAGE_SYSTEM_PROMPT
)
```

</details>

<details>
<summary>TODO ④ 정답: Analysis Agent 생성 (클릭하여 펼치기)</summary>

```python
analysis_agent = Agent(
    model=BedrockModel(
        model_id="global.anthropic.claude-sonnet-4-5-20250929-v1:0",
        region_name="us-west-2"
    ),
    tools=[analyze_correlation, suggest_additional_tests],
    system_prompt=ANALYSIS_SYSTEM_PROMPT
)
```

</details>

<details>
<summary>TODO ⑤ 정답: Recommendation Agent 생성 (클릭하여 펼치기)</summary>

```python
recommendation_agent = Agent(
    model=BedrockModel(
        model_id="global.anthropic.claude-sonnet-4-5-20250929-v1:0",
        region_name="us-west-2"
    ),
    tools=[generate_diet_plan, generate_exercise_plan, generate_followup_schedule],
    system_prompt=RECOMMENDATION_SYSTEM_PROMPT
)
```

</details>
