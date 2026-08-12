
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

### Step 1: CloudWatch GenAI Observability의 세션, 트레이스 및 스팬 추적

AgentCore Runtime에서 자동 계측(ADOT)을 활성화하려면 Docker 이미지에 OpenTelemetry를 추가하고 재배포해야 합니다.

**1. `requirements.txt`에 ADOT 패키지 추가:**

```bash
cd ~/agentcore/src
echo "aws-opentelemetry-distro" >> requirements.txt
```

**2. Dockerfile CMD 변경:**

`Dockerfile`의 마지막 줄을 아래와 같이 수정하세요:

```dockerfile
# 기존: CMD ["python", "main.py"]
# 변경: opentelemetry-instrument로 자동 계측 활성화
CMD ["opentelemetry-instrument", "python", "main.py"]
```

**3. Docker 이미지 재빌드 + Runtime 재배포:**

```bash
cd ~/agentcore/src

# 이미지 재빌드
docker build --platform linux/arm64 -t healthcare-agent:latest .

# 기존 Runtime 삭제
uv run python -c "
import boto3
client = boto3.client('bedrock-agentcore-control', region_name='us-west-2')
for rt in client.list_agent_runtimes().get('agentRuntimes', []):
    if rt['agentRuntimeName'] == 'healthcare_consultation_agent':
        client.delete_agent_runtime(agentRuntimeId=rt['agentRuntimeId'])
        print('삭제 완료')
        break
"

# 재배포
sleep 30
uv run python deploy.py
```

> 재배포 후 Runtime 상태가 `READY`가 되면, 이후 호출부터 Trace/Span이 CloudWatch에 자동 기록됩니다.

**4. Transaction Search 활성화 확인:**

```
1. AWS 콘솔 → CloudWatch
2. 좌측 메뉴: Application Signals → Transaction Search
3. Spans 화면이 보이면 이미 활성화된 상태 ✅
4. "Enable Transaction Search" 버튼이 보이면 클릭하여 활성화
```

> 이미 활성화된 계정에서는 버튼 없이 바로 Spans 쿼리 화면이 표시됩니다.

AgentCore Runtime에 배포된 에이전트는 ADOT(AWS Distro for OpenTelemetry)를 통해 자동 계측됩니다. Runtime이 OTLP Collector를 내장하고 있어 별도 설정 없이 Trace/Span이 CloudWatch에 기록됩니다.

이제 `uv run python invoke_agent.py`를 통해서 에이전트를 호출하도록 합니다. 사용자 프롬프트를 아래와 같이 바꾸어가며 진행해주세요:

```bash

# 시나리오 1
{"prompt": "공복 혈당 수치가 50으로 내려갔어요"}

# 시나리오 2
{"prompt": "몸무게가 90kg인데 어떻게 다이어트를 해야 할까요"}

# 시나리오 3
{"prompt": "가슴이 아프고 숨이 차요"}
```

각 호출 후 AWS 콘솔에서 에이전트의 Observability 데이터를 확인합니다:

```
1. AWS 콘솔 → Amazon Bedrock → AgentCore
2. 좌측 메뉴: Runtimes → 실행 중인 에이전트 선택
3. Observability 탭의 Dashboard 클릭
4. Session, Trace, Span을 관찰하세요:
   - Session: 각 invoke 호출이 하나의 세션으로 기록됨
   - Trace: 세션 내의 요청-응답 사이클
   - Span: LLM 호출, 도구 호출 등 개별 작업 단위와 소요 시간
```

> 도구 호출 순서, 각 Span의 지연 시간, 토큰 사용량을 직접 확인할 수 있습니다.
시간을 내셔서 Session, Trace(Turn), Span을 하나씩 천천히 보세요.
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
- 주의: @tool이 아닌 일반 함수로 구현 (LLM이 아닌 코드에서 직접 호출)
"""
import json
import boto3
from datetime import datetime

# CloudWatch Logs 클라이언트 초기화
logs_client = boto3.client("________", region_name="us-west-2")  # TODO ①: CloudWatch Logs 서비스 이름을 채우세요
# 감사 로그 전용 로그 그룹 이름
LOG_GROUP = "/workshop/healthcare-agent/audit"


def _ensure_log_group():
    """로그 그룹이 없으면 생성하고 보존 정책을 설정합니다."""
    try:
        logs_client.create_log_group(logGroupName=LOG_GROUP)
        # 의료법 제21조: 의료기록 5년 보존
        logs_client.put_retention_policy(
            logGroupName=LOG_GROUP,
            retentionInDays=________  # TODO ②: 5년에 해당하는 일수를 채우세요
        )
    except logs_client.exceptions.ResourceAlreadyExistsException:
        # 이미 존재하면 무시
        pass


def log_data_access(patient_id: str, action: str,
                    data_accessed: str, authorization: str = "GRANTED",
                    reason: str = ""):
    """환자 데이터 접근을 감사 로그에 기록합니다.
    
    이 함수는 @tool이 아닌 일반 함수입니다.
    LLM이 호출하는 것이 아니라, 도구 코드 내부에서 직접 호출합니다.
    """
    # 로그 그룹 존재 확인 (최초 1회만 실제 생성)
    _ensure_log_group()

    # 감사 이벤트 구조 — 규정 준수에 필요한 모든 필드 포함
    audit_event = {
        "timestamp": datetime.utcnow().isoformat() + "Z",  # UTC 표준 시간
        "event_type": "DATA_ACCESS",
        "agent_id": "________",                             # TODO ③: 에이전트 식별자를 채우세요
        "patient_id": patient_id,
        "action": action,
        "data_accessed": data_accessed.split(", ") if data_accessed else [],
        "authorization": authorization,                     # GRANTED 또는 DENIED
        "reason": reason,
        "compliance": {
            "pipa_consent": authorization == "________",    # TODO ④: 개인정보보호법 동의가 true가 되는 조건을 채우세요
            "medical_act_21": True
        }
    }

    # 로그 스트림: 날짜별로 구분 (하루 단위)
    stream_name = datetime.utcnow().strftime("________")   # TODO ⑤: 날짜 포맷을 채우세요 (예: 2026/08/10)
    try:
        logs_client.create_log_stream(
            logGroupName=LOG_GROUP,
            logStreamName=stream_name
        )
    except logs_client.exceptions.ResourceAlreadyExistsException:
        pass

    # CloudWatch Logs에 감사 이벤트 기록
    logs_client.________(                                   # TODO ⑥: 로그 이벤트를 기록하는 API 이름을 채우세요
        logGroupName=LOG_GROUP,
        logStreamName=stream_name,
        logEvents=[{
            "timestamp": int(datetime.utcnow().timestamp() * 1000),  # 밀리초 단위
            "message": json.dumps(audit_event, ensure_ascii=False)
        }]
    )
```

> **왜 @tool이 아닌 일반 함수인가?**
> 감사 로그를 `@tool`로 만들면 LLM이 `authorization` 값을 자의적으로 결정합니다 (예: 접근 거부 상황에서도 "GRANTED"로 기록). 감사 로그는 **코드 레벨에서 자동 기록**되어야 정확성이 보장됩니다.

### Step 3: lab_tools.py에서 감사 로그 직접 호출

`lab_tools.py`에서 데이터 접근 시 감사 로그를 **코드 레벨에서 자동 기록**하도록 라이브러리 import 진행해주시고, `get_lab_results` 함수의 내용을 아래와 같이 교체하세요:

```python
# lab_tools.py 상단에 import 추가
from audit_logger import log_data_access

@tool
def get_lab_results(patient_id: str, test_type: str = "all") -> str:
    """환자의 혈액 검사 결과를 조회합니다."""
    ALLOWED_PATIENTS = ["patient-001", "patient-002"]

    if patient_id not in ALLOWED_PATIENTS:
        # 접근 거부 → DENIED 감사 로그 자동 기록
        log_data_access(patient_id, "get_lab_results", "", "DENIED", "unauthorized_patient")
        return f"접근 거부: 환자 {patient_id}의 데이터에 대한 접근 권한이 없습니다."

    # 파일에서 환자 데이터 로드
    try:
        data = _load_patient_data(patient_id)
    except FileNotFoundError as e:
        return str(e)

    lab_results = data.get("lab_results", {})

    # 접근 허용 → GRANTED 감사 로그 자동 기록
    log_data_access(patient_id, "get_lab_results", test_type.upper(), "GRANTED")

    if test_type == "all":
        return json.dumps(lab_results, ensure_ascii=False, indent=2)
    elif test_type.upper() in lab_results:
        return json.dumps(lab_results[test_type.upper()], ensure_ascii=False, indent=2)
    else:
        available = ", ".join(lab_results.keys())
        return f"'{test_type}' 검사 유형을 찾을 수 없습니다. 사용 가능한 유형: {available}"
```

> **핵심**: `log_data_access`는 에이전트 도구(@tool)가 아니라 일반 함수이므로, LLM의 판단과 무관하게 데이터 접근 시점에 **반드시** 호출됩니다. GRANTED/DENIED 값도 코드 로직이 결정하므로 정확합니다.

### Step 4: 감사 로그 동작 테스트

에이전트를 실행하고 아래와 같이 환자 데이터를 조회하세요:

```bash
python consultation_agent.py
```

```
건강검진 결과를 분석해 주세요. patient-001입니다.
```
```
건강검진 결과를 분석해 주세요. patient-002입니다.
```
```
건강검진 결과를 분석해 주세요. patient-003입니다.
```

**확인:** 에이전트가 검사 결과 조회 시 `log_data_access`를 호출하여 감사 로그를 기록하는가?

### Step 5: CloudWatch Logs에서 감사 로그 확인

```bash
aws logs tail /workshop/healthcare-agent/audit \
  --region us-west-2 \
  --since 1h \
  --format short
```

**기대 출력:**

```json
{
    "timestamp": "2026-08-10T19:59:40.954062+09:00",
    "event_type": "DATA_ACCESS",
    "agent_id": "healthcare-consultation-agent",
    "patient_id": "patient-001",
    "action": "get_lab_results",
    "data_accessed": [
        "ALL"
    ],
    "authorization": "GRANTED",
    "reason": "",
    "compliance": {
        "pipa_consent": true,
        "medical_act_21": true
    }
}
```

접근 거부 시:

```json
{
    "timestamp": "2026-08-10T20:00:23.627025+09:00",
    "event_type": "DATA_ACCESS",
    "agent_id": "healthcare-consultation-agent",
    "patient_id": "patient-003",
    "action": "get_lab_results",
    "data_accessed": [],
    "authorization": "DENIED",
    "reason": "unauthorized_patient",
    "compliance": {
        "pipa_consent": false,
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
- ✅ `audit_logger.py`가 일반 함수로 구현됨 (LLM 도구가 아님)
- ✅ 환자 데이터 조회 시 GRANTED 감사 로그가 자동 기록됨
- ✅ 허용되지 않은 환자 접근 시 DENIED 감사 로그가 자동 기록됨
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

---

## 부록: 정답 코드

<details>
<summary>audit_logger.py 정답 코드 보기 (클릭하여 펼치기)</summary>

```python
"""
의료 감사 로그 생성기
- 환자 데이터 접근 시 자동으로 감사 로그 기록
- 의료법 제21조에 따라 5년(1827일) 보존
- 주의: @tool이 아닌 일반 함수로 구현 (LLM이 아닌 코드에서 직접 호출)
"""
import json
import boto3
from datetime import datetime

logs_client = boto3.client("logs", region_name="us-west-2")
LOG_GROUP = "/workshop/healthcare-agent/audit"


def _ensure_log_group():
    """로그 그룹이 없으면 생성하고 보존 정책을 설정합니다."""
    try:
        logs_client.create_log_group(logGroupName=LOG_GROUP)
        logs_client.put_retention_policy(
            logGroupName=LOG_GROUP,
            retentionInDays=1827
        )
    except logs_client.exceptions.ResourceAlreadyExistsException:
        pass


def log_data_access(patient_id: str, action: str,
                    data_accessed: str, authorization: str = "GRANTED",
                    reason: str = ""):
    """환자 데이터 접근을 감사 로그에 기록합니다."""
    _ensure_log_group()

    audit_event = {
        "timestamp": datetime.utcnow().isoformat() + "Z",
        "event_type": "DATA_ACCESS",
        "agent_id": "healthcare-consultation-agent",
        "patient_id": patient_id,
        "action": action,
        "data_accessed": data_accessed.split(", ") if data_accessed else [],
        "authorization": authorization,
        "reason": reason,
        "compliance": {
            "pipa_consent": authorization == "GRANTED",
            "medical_act_21": True
        }
    }

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
```

</details>
