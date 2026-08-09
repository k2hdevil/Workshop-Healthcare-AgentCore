---
title: "Phase 2-C: Observability 설정"
weight: 33
time: "45분"
---

# Phase 2-C: AgentCore Observability 설정 (45분)

## 학습 목표

AgentCore Observability를 설정하여 에이전트 동작을 추적하고 의료 감사 로그를 생성합니다.

---

## 이론: AgentCore Observability 개념 (10분 브리핑)

### 왜 에이전트에 Observability가 필요한가?

전통적인 애플리케이션은 요청-응답이 단순합니다. 하지만 Agentic AI는:
- LLM을 여러 번 호출하고
- 도구를 선택적으로 사용하며
- 비결정적(non-deterministic) 경로로 실행됩니다

**같은 질문이라도 매번 다른 도구를 호출하고 다른 경로를 탈 수 있기 때문에**, "에이전트가 왜 그런 응답을 했는지"를 추적하려면 전용 Observability가 필요합니다.

의료 도메인에서는 특히:
- 환자 데이터에 **누가, 언제, 무엇을** 접근했는지 감사 추적 필수
- AI 판단 근거의 **설명 가능성** 요구 (인공지능 기본법)
- 이상 동작 감지 시 **즉시 중단** 가능해야 함

### Session → Trace → Span 3단계 계층

AgentCore Observability는 3단계 계층으로 데이터를 수집합니다:

```
┌─────────────────────────────────────────────────────────────┐
│  Session (세션)                                             │
│  = 환자와의 전체 상담 대화 (여러 턴 포함)                   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Trace (트레이스)                                    │   │
│  │  = 하나의 요청-응답 사이클 (1턴)                         │   │
│  │                                                    │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐           │   │
│  │  │  Span 1  │ │  Span 2  │ │  Span 3  │           │   │
│  │  │ LLM 호출  │ │ Tool 호출 │ │ LLM 호출 │           │   │
│  │  │ 1.2초    │ │ 0.3초    │ │ 0.8초    │           │   │
│  │  └──────────┘ └──────────┘ └──────────┘           │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Trace 2 (두 번째 턴)                               │   │
│  │  ...                                                │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

| 계층 | 의미 | 예시 |
|------|------|------|
| **Session** | 사용자와의 전체 대화 | patient-001(김민수)과의 상담 세션 전체 |
| **Trace** | 한 번의 요청-응답 | "건강검진 분석해주세요" → 분석 결과 응답 |
| **Span** | 개별 작업 단위 | LLM 호출 (1.2초), analyze_lab_values 실행 (0.3초) |

**실제 건강 상담 에이전트 동작 예시:**

환자 김민수가 "건강검진 결과를 분석해 주세요"라고 질문했을 때, 에이전트 내부에서 일어나는 일을 Observability로 추적하면:

```
Session: session-patient-001-20260810
│
├─ Trace 1: "건강검진 결과를 분석해 주세요"
│  │
│  ├─ Span: LLM 호출 #1 (0.9초)
│  │   → 모델이 "analyze_lab_values 도구를 호출해야겠다"고 판단
│  │
│  ├─ Span: analyze_lab_values 실행 (0.2초)
│  │   → patient-001 데이터 로드, 비정상 5개 식별, 등급 판정
│  │
│  ├─ Span: LLM 호출 #2 (1.5초)
│  │   → 도구 결과를 바탕으로 환자에게 설명할 응답 생성
│  │
│  └─ 총 소요: 2.6초, 입력 토큰: 1,200, 출력 토큰: 850
│
├─ Trace 2: "콜레스테롤 관리 방법을 알려주세요"
│  │
│  ├─ Span: LLM 호출 #1 (0.7초)
│  │   → "search_medical_knowledge 도구로 콜레스테롤 정보 검색"
│  │
│  ├─ Span: search_medical_knowledge 실행 (0.1초)
│  │   → 콜레스테롤 관련 의료 지식 반환
│  │
│  ├─ Span: LLM 호출 #2 (1.3초)
│  │   → 검사 결과(235mg/dL)에 맞춘 개인화된 생활습관 권고 생성
│  │
│  └─ 총 소요: 2.1초, 입력 토큰: 1,500, 출력 토큰: 600
│
└─ Trace 3: "가슴이 아프고 숨이 차요"
   │
   ├─ Span: LLM 호출 #1 (0.5초)
   │   → "assess_urgency 도구로 긴급도 평가 필요"
   │
   ├─ Span: assess_urgency 실행 (0.05초)
   │   → "흉통" 키워드 감지 → 🚨 긴급 판정
   │
   ├─ Span: LLM 호출 #2 (0.8초)
   │   → "즉시 119 또는 응급실 방문" 안내 생성
   │
   └─ 총 소요: 1.35초, 입력 토큰: 800, 출력 토큰: 300
```

> 이 트레이스 데이터가 CloudWatch GenAI Observability에 자동으로 기록되어, "어떤 도구가 가장 많이 호출되는지", "응답 시간이 느린 구간은 어디인지", "토큰 비용은 얼마인지" 를 한눈에 파악할 수 있습니다.

### 자동 계측 vs 수동 계측

| 구분 | 자동 계측 | 수동 계측 (ADOT) |
|------|----------|-----------------|
| 설정 | AgentCore Runtime 배포 시 자동 | 코드에 ADOT SDK 추가 필요 |
| 수집 항목 | 세션 수, 지연 시간, 토큰 수, 오류율 | 커스텀 Span, 속성, 메트릭 |
| 적합한 용도 | 기본 모니터링, 비용 추적 | 상세 디버깅, 의료 감사 로그 |

**자동 계측으로 수집되는 기본 메트릭:**

| 메트릭 | 설명 |
|--------|------|
| Session Count | 활성/완료 세션 수 |
| Latency (P50/P95) | 응답 시간 분포 |
| Input/Output Token Usage | 입출력 토큰 합산 |
| Error Rate | 실패한 세션 비율 |
| Tool Invocations | 도구별 호출 횟수 |

**수동 계측(ADOT)으로 추가 가능한 항목:**

| 항목 | 설명 |
|------|------|
| 커스텀 Span | 특정 함수 실행 시간 추적 |
| 커스텀 속성 | patient_id, tool_name 등 메타데이터 |
| 커스텀 메트릭 | "긴급 증상 감지 횟수" 등 비즈니스 지표 |
| 구조화된 로그 | 감사 로그, 데이터 접근 기록 |

### CloudWatch GenAI Observability

AgentCore는 Amazon CloudWatch의 **GenAI Observability** 페이지에 데이터를 자동 전송합니다. 이 페이지에서 다음을 확인할 수 있습니다:

- 에이전트별 세션 대시보드
- Trace 타임라인 (각 Span의 실행 순서와 시간)
- LLM 호출 비용 추적
- 오류 상세 (스택 트레이스 포함)

---

## 실습 시작

### Step 1: CloudWatch Transaction Search 활성화

AgentCore Runtime에 배포된 에이전트의 트레이스를 보려면 Transaction Search가 활성화되어 있어야 합니다:

```
1. AWS 콘솔 → CloudWatch
2. 좌측 메뉴: Application Signals → Transaction Search
3. Spans 화면이 보이면 이미 활성화된 상태 ✅
4. "Enable Transaction Search" 버튼이 보이면 클릭하여 활성화
```

> 이미 활성화된 계정에서는 버튼 없이 바로 Spans 쿼리 화면이 표시됩니다.

<details>
<summary>참고: ADOT를 이용한 자동 계측 (AgentCore Runtime 배포 시)</summary>

AgentCore Runtime에 배포된 에이전트는 ADOT(AWS Distro for OpenTelemetry)를 통해 자동 계측됩니다. Runtime이 OTLP Collector를 내장하고 있어 별도 설정 없이 Trace/Span이 CloudWatch에 기록됩니다.

로컬에서 테스트하려면 아래 환경변수를 설정하고 실행합니다:

```bash
pip install "aws-opentelemetry-distro>=0.10.0"

export AGENT_OBSERVABILITY_ENABLED=true
export OTEL_PYTHON_DISTRO=aws_distro
export OTEL_PYTHON_CONFIGURATOR=aws_configurator
export OTEL_RESOURCE_ATTRIBUTES="service.name=healthcare-consultation-agent"
export OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
export OTEL_TRACES_EXPORTER=otlp

opentelemetry-instrument python consultation_agent.py
```

> ⚠️ 로컬 환경에서는 OTLP Collector가 없어 `403/404` 에러가 발생합니다. 에이전트 자체는 정상 동작하며, **AgentCore Runtime에 배포하면 자동으로 해결**됩니다.

</details>

---

### Step 2: 의료 감사 로그 구현

의료법 제21조에 따라 의료기록은 **5년간 보존**해야 합니다. AI 에이전트가 환자 데이터에 접근한 기록도 감사 추적(Audit Trail)이 필요합니다.

이 실습에서는 boto3로 CloudWatch Logs에 직접 감사 로그를 기록합니다 (로컬에서 바로 동작):

```bash
cd ~/agentcore/src
touch audit_logger.py
```

`audit_logger.py`를 열고 아래 코드를 작성하세요:

```python
"""
의료 감사 로그 생성기
- 환자 데이터 접근 시 자동으로 감사 로그 기록
- 의료법 제21조에 따라 5년(1827일) 보존
"""
import json
import boto3
from datetime import datetime
from strands import tool

logs_client = boto3.client("logs", region_name="us-west-2")
LOG_GROUP = "/workshop/healthcare-agent/audit"


def _ensure_log_group():
    """로그 그룹이 없으면 생성합니다."""
    try:
        logs_client.create_log_group(logGroupName=LOG_GROUP)
        # 의료법 제21조: 의료기록 5년 보존
        logs_client.put_retention_policy(
            logGroupName=LOG_GROUP,
            retentionInDays=1827  # 약 5년
        )
    except logs_client.exceptions.ResourceAlreadyExistsException:
        pass


@tool
def log_data_access(patient_id: str, action: str,
                    data_accessed: str) -> str:
    """환자 데이터 접근을 감사 로그에 기록합니다.

    Args:
        patient_id: 접근한 환자 ID
        action: 수행한 작업 (예: lab_results_query, analyze_values)
        data_accessed: 접근한 데이터 종류 (예: CBC, LIPID)

    Returns:
        로그 기록 확인 메시지
    """
    _ensure_log_group()

    # 감사 이벤트 구조
    audit_event = {
        "timestamp": datetime.utcnow().isoformat() + "Z",
        "event_type": "DATA_ACCESS",
        "agent_id": "healthcare-consultation-agent",
        "patient_id": patient_id,
        "action": action,
        "data_accessed": data_accessed.split(", "),
        "authorization": "GRANTED",
        "compliance": {
            "pipa_consent": True,       # 개인정보보호법 동의
            "medical_act_21": True      # 의료법 제21조 준수
        }
    }

    # CloudWatch Logs에 기록
    stream_name = datetime.utcnow().strftime("%Y/%m/%d")
    try:
        logs_client.create_log_stream(
            logGroupName=LOG_GROUP,
            logStreamName=stream_name
        )
    except logs_client.exceptions.ResourceAlreadyExistsException:
        pass

    logs_client.put_log_events(
        logGroupName=LOG_GROUP,
        logStreamName=stream_name,
        logEvents=[{
            "timestamp": int(datetime.utcnow().timestamp() * 1000),
            "message": json.dumps(audit_event, ensure_ascii=False)
        }]
    )

    return f"감사 로그 기록 완료: {action} on {patient_id}"
```

### Step 3: 에이전트에 감사 로그 도구 추가

`consultation_agent.py`를 열고 감사 로그 도구를 추가하세요:

```python
# 상단 import 추가
from audit_logger import log_data_access

# 에이전트 tools 목록에 추가
consultation_agent = Agent(
    ...
    tools=[
        search_medical_knowledge,
        assess_urgency,
        get_lab_results,
        analyze_lab_values,
        log_data_access        # 감사 로그 추가
    ],
    ...
)
```

### Step 4: 감사 로그 동작 테스트

에이전트를 실행하고 환자 데이터를 조회하세요:

```bash
python consultation_agent.py
```

```
건강검진 결과를 분석해 주세요. patient-001입니다.
```

**확인:** 에이전트가 검사 결과 조회 시 `log_data_access`를 호출하여 감사 로그를 기록하는가?

### Step 5: CloudWatch Logs에서 감사 로그 확인

```bash
aws logs filter-log-events \
  --log-group-name /workshop/healthcare-agent/audit \
  --region us-west-2 \
  --limit 5 | python3 -m json.tool
```

**기대 출력:**

```json
{
  "timestamp": "2026-08-10T08:30:00Z",
  "event_type": "DATA_ACCESS",
  "agent_id": "healthcare-consultation-agent",
  "patient_id": "patient-001",
  "action": "analyze_values",
  "data_accessed": ["CBC", "LIPID", "METABOLIC", "LIVER"],
  "authorization": "GRANTED",
  "compliance": {
    "pipa_consent": true,
    "medical_act_21": true
  }
}
```

---

## Observability 모범 사례

| # | 사례 | 설명 |
|---|------|------|
| 1 | **일관된 Session ID 사용** | 관련 요청에 같은 세션 ID를 재사용하여 대화 맥락 추적 |
| 2 | **분산 추적 활성화** | X-Amzn-Trace-Id 헤더로 서비스 간 end-to-end 추적 |
| 3 | **커스텀 속성 추가** | patient_id, urgency_level 등 도메인 메타데이터 태깅 |
| 4 | **알람 설정** | 오류율 5% 초과, P95 지연 10초 초과 시 SNS 알림 |
| 5 | **리소스 사용량 모니터링** | 토큰 사용량으로 비용 예측 및 이상 감지 |

---

## 검증

- ✅ CloudWatch Transaction Search 활성화 확인
- ✅ `audit_logger.py`가 에이전트 도구로 정상 등록됨
- ✅ 환자 데이터 조회 시 `log_data_access`가 호출됨
- ✅ CloudWatch Logs에 감사 로그가 기록됨 (5년 보존 정책)

---

## 🏆 Challenge Task

1. CloudWatch 알람을 생성하여 오류율 5% 초과 시 SNS 알림을 받으세요.
2. 커스텀 메트릭 "긴급 증상 감지 횟수"를 CloudWatch Metrics에 기록하는 코드를 추가하세요.

---

## Day 1 완료

Day 1의 모든 Phase를 완료했습니다.

### Day 1 최종 산출물 확인

- ✅ 배포된 싱글 에이전트 (API 호출 가능)
- ✅ 검사 결과 조회/분석 기능 (5개 비정상 항목 식별)
- ✅ 멀티턴 대화 + Hook 기반 메모리 통합
- ✅ CloudWatch Observability + 의료 감사 로그

Day 2에서는 이 에이전트를 **3개 전문 에이전트**로 확장하고,
**보안/평가/침투 테스트**를 추가합니다.
