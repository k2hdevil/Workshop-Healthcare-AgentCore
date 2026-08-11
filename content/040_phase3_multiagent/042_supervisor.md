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

### 세션 내 데이터 공유 방법

멀티 에이전트에서 **이전 에이전트의 결과를 다음 에이전트가 참조**하는 방법은 패턴에 따라 다릅니다:

| 방식 | 구현 | 장점 | 단점 |
|------|------|------|------|
| **프롬프트 체이닝** (본 워크샵) | Supervisor가 A 결과를 B 프롬프트에 포함 | 구현 간단, 추가 인프라 불필요 | 컨텍스트 윈도우 제한에 걸릴 수 있음 |
| **공유 딕셔너리** | Python dict를 전역으로 두고 에이전트 간 참조 | 빠름, 구조화된 데이터 | 단일 프로세스에서만 동작 |
| **Swarm SharedContext** | Strands Swarm 내장 공유 메모리 | 프레임워크 지원, 자동 관리 | Swarm 패턴 전용 |
| **Graph State** | Graph 노드 간 결과 자동 전파 | 의존성 기반 자동 흐름 | Graph 패턴 전용 |

**본 워크샵의 데이터 공유 흐름:**

```
Supervisor
  │
  ├─ triage_specialist("patient-001 분석해줘")
  │   └→ Triage Agent가 get_lab_results 호출 → 결과 반환
  │
  ├─ analysis_specialist("Triage 결과: WBC↑, 혈당↑, ... 심층 분석해줘")
  │                        ↑ Supervisor가 이전 결과를 프롬프트에 포함
  │   └→ Analysis Agent가 상관관계 분석 → 결과 반환
  │
  └─ recommendation_specialist("분석 결과: 대사증후군 위험... 권고해줘")
                               ↑ 이전 두 에이전트 결과를 모두 전달
      └→ Recommendation Agent가 건강 관리 계획 → 결과 반환
```

> **핵심**: Supervisor의 LLM이 이전 도구 호출 결과를 자동으로 컨텍스트에 유지합니다.
> 별도 코드 없이도 Supervisor가 A 결과를 참조하여 B에게 적절한 요청을 생성합니다.
> 이것이 Agent-as-Tool 패턴에서 데이터 공유가 자연스럽게 이루어지는 이유입니다.

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

````python
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
- 이름: 김민수
- 나이/성별: 45세 / 남성
- 검진일: 2026-07-15
- 검진기관: OO건강검진센터

## 2. 검진 결과 분류 (Triage)
- 종합 판정: B등급 (경계 수준)
- 비정상 항목: 5개
  | 항목 | 수치 | 정상범위 | 판정 |
  |------|------|---------|------|
  | WBC | 12.5 (10³/μL) | 4.0-10.0 | B (경계) |
  | 공복혈당 | 115 mg/dL | <100 | B (경계) |
  | 총콜레스테롤 | 235 mg/dL | <200 | B (경계) |
  | LDL | 145 mg/dL | <130 | B (경계) |
  | 중성지방 | 180 mg/dL | <150 | B (경계) |
- 우선순위: 중간 (3개월 내 생활습관 개선 후 재검)

## 3. 심층 분석 (Analysis)
- 상관관계: LDL↑ + 중성지방↑ → 대사증후군 위험 패턴
- 공복혈당 115 + 이상지질혈증 → 인슐린 저항성 가능성
- WBC 경미한 상승: 스트레스 또는 경미한 염증 가능
- 추가 검사 권고: HbA1c, 경동맥 초음파, CRP

## 4. 건강 관리 권고 (Recommendation)
### 식이
- 권장: 등푸른 생선(주 2-3회), 나물 반찬, 잡곡밥, 견과류
- 제한: 내장류, 가공육, 흰쌀밥 단독, 음주(주 2회 이하)
- 한국식 팁: 식사 순서(나물→단백질→밥), 국물 기름 제거

### 운동
- 빈도: 주 4-5회
- 종류: 빠르게 걷기 40분 + 근력운동 2회
- 강도: 약간 숨찬 수준 (RPE 5-6/10)
- 주의: 준비운동 10분 필수

### 추적 검사
- 3개월 후: 공복혈당, 지질 패널 재검
- 6개월 후: 종합 재평가

## 5. 종합 소견
45세 남성으로 주요 대사 지표 5개 항목에서 경계(B등급) 소견이
확인되었습니다. 현재 질환 단계는 아니나 대사증후군으로
진행될 위험이 있으므로, 식이 조절과 규칙적 운동을 통한
적극적 생활습관 개선이 필요합니다.

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
    tools=[________, ________, ________],  # TODO: 3개 전문가 도구
    system_prompt=________
)


# ─────────────────────────────────────────────
# 대화형 테스트 (도구 호출 실시간 추적)
# ─────────────────────────────────────────────
if __name__ == "__main__":
    print("=" * 60)
    print("  종합 건강검진 AI 분석 시스템 (Supervisor Agent)")
    print("  끝내시려면 quit, exit 또는 종료")
    print("=" * 60)
    
    def trace_callback(**kwargs):
        """도구 호출을 실시간으로 콘솔에 출력합니다."""
        if "current_tool_use" in kwargs:
            tool_use = kwargs["current_tool_use"]
            tool_name = tool_use.get("name", "unknown")
            print(f"\n  🔧 [도구 호출] {tool_name}")
        if "data" in kwargs:
            # 스트리밍 텍스트 출력
            print(kwargs["data"], end="", flush=True)
    
    while True:
        user_input = input("\n요청: ")
        if user_input.lower() in ["quit", "exit", "종료"]:
            break
        
        import time
        start = time.time()
        
        print("\n" + "-" * 40)
        print("  실행 중... (도구 호출 추적)")
        print("-" * 40)
        
        response = supervisor_agent(
            user_input,
            callback_handler=trace_callback
        )
        elapsed = time.time() - start
        
        print(f"\n\n{'='*60}")
        print(f"  [최종 보고서]")
        print(f"{'='*60}")
        print(response.message['content'][0]['text'])
        print(f"\n[소요 시간: {elapsed:.1f}초]")
````

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

**기대 콘솔 출력:**

```
요청: patient-001의 종합 건강검진 결과를 분석해 주세요.

----------------------------------------
  실행 중... (도구 호출 추적)
----------------------------------------

  🔧 [도구 호출] triage_specialist
  🔧 [도구 호출] analysis_specialist
  🔧 [도구 호출] recommendation_specialist

============================================================
  [최종 보고서]
============================================================
# 종합 건강검진 AI 분석 보고서
...

[소요 시간: 12.3초]
```

> 3개 도구 호출(`triage_specialist`, `analysis_specialist`, `recommendation_specialist`)이 순차적으로 출력되면 정상 동작입니다.

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

### Step 4: 데이터 공유 흐름 확인

Supervisor가 이전 에이전트의 결과를 다음 에이전트에 전달하는지 확인합니다.

터미널 출력에서 다음을 관찰하세요:

1. `triage_specialist` 호출 → Triage 결과 (비정상 5개 항목)
2. `analysis_specialist` 호출 시 **Triage 결과가 프롬프트에 포함**됨
3. `recommendation_specialist` 호출 시 **Triage + Analysis 결과가 모두 포함**됨

> **확인 방법**: Supervisor의 응답 생성 과정에서 "이전 분류 결과를 바탕으로..." 또는
> "위의 분석에 따르면..." 같은 표현이 있으면 데이터 공유가 정상 동작하는 것입니다.

**명시적 데이터 공유가 필요한 경우** (공유 딕셔너리 패턴):

```python
# 공유 컨텍스트를 딕셔너리로 관리하는 예시
shared_context = {}

@tool
def triage_specialist(patient_data: str) -> str:
    """Triage 전문가에게 분류를 요청합니다."""
    response = triage_agent(patient_data)
    result = response.message['content'][0]['text']
    # 공유 컨텍스트에 결과 저장
    shared_context["triage_result"] = result
    return result

@tool
def analysis_specialist(analysis_request: str) -> str:
    """임상병리 분석 전문가에게 심층 분석을 요청합니다."""
    # 공유 컨텍스트에서 Triage 결과 참조
    triage_result = shared_context.get("triage_result", "")
    full_request = f"Triage 결과:\n{triage_result}\n\n추가 요청:\n{analysis_request}"
    response = analysis_agent(full_request)
    result = response.message['content'][0]['text']
    shared_context["analysis_result"] = result
    return result
```

> 본 워크샵에서는 Supervisor LLM이 자동으로 컨텍스트를 전달하므로 위 코드는 참고용입니다.
> 하지만 LLM이 중요한 정보를 누락할 경우, 이처럼 명시적 공유 딕셔너리를 사용하면 확실합니다.

---

### Step 5: 마크다운 보고서 생성

종합 보고서를 마크다운 파일로 저장합니다.

```bash
cd ~/agentcore/src
touch generate_report.py
```

`generate_report.py`를 열고 아래 코드를 작성하세요:

```python
"""
종합 건강검진 AI 분석 보고서 생성기
- Supervisor Agent를 호출하고 결과를 마크다운 파일로 저장
"""
import os
from datetime import datetime
from supervisor_agent import supervisor_agent

# 산출물 저장 폴더
OUTPUT_DIR = os.path.expanduser("~/agentcore/outputs")
os.makedirs(OUTPUT_DIR, exist_ok=True)


def generate_report(patient_id: str = "patient-001"):
    """에이전트를 호출하고 결과를 마크다운 파일로 저장합니다."""
    
    print(f"→ {patient_id} 분석 중...")
    response = supervisor_agent(
        f"{patient_id}의 종합 건강검진 결과를 분석해 주세요. "
        f"먼저 triage_specialist를 호출하여 {patient_id}의 검진 결과를 분류하고, "
        f"그 결과를 바탕으로 analysis_specialist와 recommendation_specialist를 "
        f"순차적으로 호출하여 종합 보고서를 작성해 주세요."
    )
    report_text = response.message['content'][0]['text']
    
    # 마크다운 파일로 저장
    timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
    md_path = os.path.join(OUTPUT_DIR, f"health_report_{patient_id}_{timestamp}.md")
    
    with open(md_path, "w", encoding="utf-8") as f:
        f.write(report_text)
    
    print(f"✓ 보고서 저장 완료: {md_path}")
    return md_path


if __name__ == "__main__":
    generate_report("patient-001")
```

실행:

```bash
cd ~/agentcore/src
uv run python generate_report.py
```

생성된 파일 확인:

```bash
cat ~/agentcore/outputs/health_report_patient-001_*.md
```

> **산출물 경로**: `~/agentcore/outputs/health_report_patient-001_YYYYMMDD_HHMMSS.md`
>
> **PDF 변환이 필요한 경우**: VS Code에서 마크다운 파일을 열고
> "Markdown PDF" 확장(`yzane.markdown-pdf`)을 설치하여
> `Ctrl+Shift+P` → "Markdown PDF: Export (pdf)"로 변환할 수 있습니다.

---

## 검증

- [ ] Supervisor가 3개 전문 에이전트를 모두 호출함
- [ ] 종합 보고서 형식에 맞게 출력됨
- [ ] 한국 건강검진 기준에 맞는 분석 결과
- [ ] 면책 조항 포함
- [ ] 마크다운 보고서 파일이 `~/agentcore/outputs/`에 정상 생성됨

---

## 🏆 Challenge Task

1. Supervisor에 "요약 모드" 옵션을 추가하여, 간략한 1페이지 요약 보고서를 생성하도록 하세요
2. 여러 환자(patient-001, patient-002)를 순차 분석하는 배치 모드를 구현하세요

---

완료 후 [Phase 4-A: 보안 강화](../050_phase4_security/051_guardrails_policy.md)로 이동하세요.

---

## 부록: 정답 코드

<details>
<summary>TODO ①~③ 정답: 전문 에이전트 @tool 래핑 (클릭하여 펼치기)</summary>

```python
@tool
def triage_specialist(patient_data: str) -> str:
    """Triage 전문가에게 건강검진 결과 분류를 요청합니다."""
    # TODO ①
    response = triage_agent(patient_data)
    return response.message['content'][0]['text']


@tool
def analysis_specialist(analysis_request: str) -> str:
    """임상병리 분석 전문가에게 심층 분석을 요청합니다."""
    # TODO ②
    response = analysis_agent(analysis_request)
    return response.message['content'][0]['text']


@tool
def recommendation_specialist(recommendation_request: str) -> str:
    """건강관리 전문가에게 종합 권고사항 생성을 요청합니다."""
    # TODO ③
    response = recommendation_agent(recommendation_request)
    return response.message['content'][0]['text']
```

</details>

<details>
<summary>TODO ④ 정답: Supervisor Agent 생성 (클릭하여 펼치기)</summary>

```python
supervisor_agent = Agent(
    model=BedrockModel(
        model_id="global.anthropic.claude-sonnet-4-5-20250929-v1:0",
        region_name="us-west-2"
    ),
    tools=[triage_specialist, analysis_specialist, recommendation_specialist],
    system_prompt=SUPERVISOR_SYSTEM_PROMPT
)
```

</details>