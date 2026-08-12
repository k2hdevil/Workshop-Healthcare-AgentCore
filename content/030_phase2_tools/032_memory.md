---
title: "Phase 2-B: 메모리 통합"
weight: 32
time: "60분"
---

# Phase 2-B: AgentCore Memory 통합 (60분)

## 학습 목표

AgentCore Memory를 활용하여 환자 상담 이력을 유지하고, 세션 간 개인화된 응답을 제공합니다.

---

## 이론: AgentCore Memory 개념 (10분 브리핑)

### Agentic AI에 메모리가 필요한 이유

일반적인 LLM은 **무상태(stateless)** 입니다. 같은 세션 안에서는 컨텍스트 윈도우 덕분에 대화가 이어지지만, 세션이 종료되면 이전 대화를 전혀 기억하지 못합니다. 이것은 건강 상담처럼 여러 날에 걸쳐 연속적인 맥락이 중요한 서비스에서 큰 문제가 됩니다.

**메모리 없는 에이전트의 한계:**

```
[1차 방문] 환자: 두통이 있어요 → AI: 두통 원인을 설명합니다...
[2차 방문] 환자: 지난번 두통이 다시 심해졌어요 → AI: "지난번"이 뭔지 모름 ❌
```

**메모리 있는 에이전트:**

```
[1차 방문] 환자: 두통이 있어요 → AI: 두통 원인을 설명합니다... (메모리에 저장)
[2차 방문] 환자: 지난번 두통이 다시 심해졌어요 → AI: (이력 조회) "지난번 긴장성 두통으로 
           상담하셨죠. 악화됐다면 신경과 진료를 권합니다." ✅
```

**의료 상담에서 메모리가 필수인 3가지 이유:**

| # | 이유 | 예시 |
|---|------|------|
| 1 | **연속성** — 환자의 증상 경과를 추적 | "지난달 혈당 115였는데 이번에 130으로 올랐네요" |
| 2 | **개인화** — 환자별 맞춤 응답 제공 | "고혈압 가족력이 있으시니 염분 섭취를 특히 주의하세요" |
| 3 | **안전성** — 과거 알레르기·약물 이력 참조 | "이전에 아스피린 부작용이 있으셨으니 대체 약물을 의사와 상의하세요" |

### LLM 컨텍스트 윈도우 vs 메모리

LLM에는 "컨텍스트 윈도우"라는 입력 제한이 있습니다. 그럼 컨텍스트 윈도우에 모든 대화를 넣으면 메모리가 필요 없는 걸까요?

| 비교 항목 | 컨텍스트 윈도우 | 외부 메모리 (AgentCore Memory) |
|-----------|---------------|-------------------------------|
| 용량 | 제한적 (예: 200K 토큰) | 사실상 무제한 |
| 지속성 | 한 세션 내에서만 유효. 세션 종료 시 소멸 | 영구 저장. 다음 날, 다음 달에도 조회 가능 |
| 비용 | 토큰 수에 비례하여 과금. 이전 대화를 모두 넣으면 비용 급증 | 필요한 정보만 검색하여 최소 토큰 사용 |
| 검색 방식 | 전체를 통째로 전달 (관련 없는 정보도 포함) | 의미 검색으로 관련 이력만 선별 조회 |
| 환자 수 확장 | 환자 100명의 이력을 모두 넣을 수 없음 | 환자별 네임스페이스로 독립 관리 가능 |

**핵심 차이:**

```
컨텍스트 윈도우 = "지금 이 대화에서 기억하는 것" (단기 작업 메모리)
외부 메모리     = "과거 모든 상담 기록을 보관하는 캐비닛" (장기 저장소)
```

실제 운영에서는 **외부 메모리에서 관련 이력을 검색 → 컨텍스트 윈도우에 주입 → LLM이 참고하여 응답**하는 방식으로 둘을 조합합니다. AgentCore Memory가 바로 이 "외부 메모리 + 검색" 역할을 해주는 서비스입니다.

### Short-Term Memory (STM) vs Long-Term Memory (LTM)

| 구분 | Short-Term Memory | Long-Term Memory |
|------|------------------|-----------------|
| 범위 | 현재 세션 | 세션 간 영구 보존 |
| 용도 | 대화 흐름 유지 | 사용자 선호도, 과거 요약 |
| 저장 방식 | 원본 이벤트 그대로 저장 | 추출 전략으로 핵심만 추출 |
| 조회 방식 | 세션 ID로 재로드 | 의미 검색 (Semantic Search) |
| 만료 | 설정 가능 (기본 24시간) | 삭제 전까지 영구 |

### AgentCore Memory 동작 흐름

```
┌─────────────────┐        ┌─────────────────────┐        ┌─────────────────────┐
│                 │        │  Short-Term Memory  │        │   Long-Term Memory  │
│  메시지 (대화)     │───────▶│  (단기 메모리)        │───────▶│   (장기 메모리)        │
│                 │        │                     │        │                     │
│ USER/ASSISTANT  │  Hook  │ - 원본 이벤트 보관      │ 추출   │ - 핵심 정보만 추출      │
│ 턴 페이로드        │  저장   │ - 세션 ID 기준        │ 전략   │ - 의미 검색 가능        │
│                 │        │ - 만료 시간 설정     │        │ - 영구 보존             │
└─────────────────┘        └─────────────────────┘        └──────────┬──────────┘
                                                                     │
                                                                     │ 통합
                                                                     ▼
                                                          ┌─────────────────────┐
                                                          │  기존 장기 메모리와  │
                                                          │  병합 (Consolidation)│
                                                          │                     │
                                                          │ - 중복 제거         │
                                                          │ - 시간순 정리       │
                                                          │ - 요약 업데이트     │
                                                          └─────────────────────┘

                        ┌──────────────────────────────────────────────────┐
    새 세션 시작 시 ◀────.  │  Hook이 장기 메모리에서 관련 이력을 검색하여       │
                        │  시스템 프롬프트에 자동 주입                      │
                        └──────────────────────────────────────────────────┘
```

### 통합 방식: Strands Agent Hook

Strands Agent는 **Hook** 시스템을 통해 에이전트 라이프사이클의 각 단계에 콜백을 등록할 수 있습니다. 이를 활용하면 별도 도구(`@tool`) 없이도 메모리 저장/조회가 **자동으로** 동작합니다.

| Hook 이벤트 | 시점 | 메모리 동작 |
|------------|------|------------|
| `BeforeInvocationEvent` | 에이전트 호출 직전 | 장기 메모리에서 이력 조회 → 시스템 프롬프트에 주입 |
| `AfterInvocationEvent` | 에이전트 응답 완료 후 | 대화 턴을 단기 메모리에 저장 |

---

## 실습 시작

### Step 1: 의존성 설치 및 파일 생성

```bash
pip install bedrock-agentcore
cd ~/agentcore/src
touch memory_hook.py
```

### Step 2: 메모리 Hook 구현

`memory_hook.py`를 열고 아래 코드를 작성하세요:

```python
"""
AgentCore Memory Hook
- BeforeInvocationEvent: 장기 메모리에서 환자 이력 조회 → 컨텍스트 주입
- AfterInvocationEvent: 대화 내용을 단기 메모리에 저장
"""
import os
import boto3
import json
from datetime import datetime
from strands.hooks import HookProvider, HookRegistry, BeforeInvocationEvent, AfterInvocationEvent

# ──────────────────────────────────────────────
# AgentCore Memory Hook 클래스
# - HookProvider를 구현하여 에이전트 라이프사이클에 연결
# - 매 호출마다 자동으로 메모리 저장/조회 수행
# ──────────────────────────────────────────────
class AgentCoreMemoryHook(HookProvider):
    """AgentCore Memory를 Strands Agent Hook으로 통합합니다."""
    
    def __init__(self, memory_id: str, actor_id: str, region: str = "us-west-2"):
        self.memory_id = memory_id
        self.actor_id = actor_id
        self.session_id = f"session-{actor_id}-{datetime.utcnow().strftime('%Y%m%d')}"
        self.client = boto3.client("bedrock-agentcore", region_name=region)
    
    def register_hooks(self, registry: HookRegistry) -> None:
        """Hook 이벤트에 콜백을 등록합니다."""
        # TODO: 두 개의 콜백을 등록하세요
        # 힌트: BeforeInvocationEvent → 이력 조회, AfterInvocationEvent → 대화 저장
        registry.add_callback(_________, self.retrieve_memory)
        registry.add_callback(_________, self.save_memory)
    
    def retrieve_memory(self, event: BeforeInvocationEvent) -> None:
        """에이전트 호출 전, 장기 메모리에서 환자 이력을 조회합니다."""
        try:
            # TODO: 장기 메모리에서 기록을 검색하는 API를 호출하세요
            # 힌트: 위 "관련 API" 표에서 "이력 조회" 행을 참고하세요
            response = self.client._________(
                memoryId=self.memory_id,
                namespace=f"/preferences/{self.actor_id}/",
                searchCriteria={
                    "searchQuery": f"환자 {self.actor_id}의 상담 이력"
                }
            )
            
            records = response.get("memoryRecordSummaries", [])
            if records:
                # 조회된 이력을 시스템 프롬프트 컨텍스트에 추가
                memory_context = "\n## 이전 상담 이력 (메모리에서 조회됨)\n"
                for record in records:
                    content = record.get("content", {}).get("text", "")
                    memory_context += f"- {content}\n"
                
                # 에이전트의 시스템 프롬프트에 이력 주입
                if hasattr(event, 'agent') and event.agent.system_prompt:
                    event.agent.system_prompt += memory_context
        except Exception as e:
            print(f"[Memory Hook] 이력 조회 실패: {e}")
    
    def save_memory(self, event: AfterInvocationEvent) -> None:
        """에이전트 응답 후, 대화 내용을 단기 메모리에 저장합니다."""
        try:
            # TODO: 단기 메모리에 이벤트를 저장하는 API를 호출하세요
            # 힌트: 위 "관련 API" 표에서 "이벤트 저장" 행을 참고하세요
            self.client._________(
                memoryId=self.memory_id,
                actorId=self.actor_id,
                sessionId=self.session_id,
                eventTimestamp=datetime.utcnow(),
                payload=[{
                    "conversational": {
                        "content": {"text": str(event.result) if event.result else ""},
                        "role": "ASSISTANT"
                    }
                }]
            )
        except Exception as e:
            print(f"[Memory Hook] 저장 실패: {e}")
```

### Step 3: 메모리 리소스 생성

```bash
# AgentCore Memory 생성 (userPreference 전략으로 장기 메모리 활성화)
aws bedrock-agentcore-control create-memory \
  --name "patient_consultation_memory" \
  --description "환자 상담 이력 저장용 메모리" \
  --event-expiry-duration 90 \
  --memory-strategies '[{
    "userPreferenceMemoryStrategy": {
      "name": "PatientPreferenceLearner",
      "namespaceTemplates": ["/preferences/{actorId}/"]
    }
  }]' \
  --region us-west-2
```

출력에서 `memoryId` 값을 복사하여 아래 Step 4에서 `MEMORY_ID`에 입력하세요.

> **메모리 전략:** `userPreferenceMemoryStrategy`는 매 대화 턴마다 환자의 선호도, 증상 패턴, 건강 특성을 자동 추출하여 장기 메모리에 저장합니다. 전략을 설정하지 않으면 장기 메모리로의 추출이 일어나지 않습니다.

### Step 4: 에이전트에 Hook 연결

`consultation_agent.py`를 열고 Hook을 추가하세요:

```python
import os
from memory_hook import AgentCoreMemoryHook

# 메모리 Hook 생성
MEMORY_ID = os.environ.get("MEMORY_ID", "<YOUR_MEMORY_ID>")

memory_hook = AgentCoreMemoryHook(
    memory_id=MEMORY_ID,
    actor_id="patient-001"
)

# 에이전트에 Hook 연결
consultation_agent = Agent(
    model=BedrockModel(
        model_id="global.anthropic.claude-sonnet-4-5-20250929-v1:0",
        region_name="us-west-2"
    ),
    tools=[
        search_medical_knowledge,
        assess_urgency,
        get_lab_results,
        analyze_lab_values
    ],
    system_prompt=MEDICAL_SYSTEM_PROMPT,
    hooks=[memory_hook]    # Hook으로 메모리 자동 통합
)
```

> **`@tool` 방식과의 차이:** Hook 방식은 에이전트가 도구 호출 여부를 판단할 필요 없이, 매 호출마다 **자동으로** 저장/조회가 수행됩니다.

### Step 5: 멀티턴 대화 테스트

```bash
python consultation_agent.py
```

---

## 테스트: 멀티턴 대화 시나리오

### 사전 준비: 이전 상담 기록 생성

메모리 조회 테스트를 위해, 먼저 이전 상담 기록을 남겨야 합니다. 아래 대화를 먼저 진행하고 **세션을 종료**하세요:

```
안녕하세요. 환자 ID는 patient-001입니다. 3일 전부터 두통이 있어요. 오후에 특히 심해집니다.
```

```
그러면 두통에 있어서 특히 주의해야 할 사항은 무엇인가요?
```

**확인:** 에이전트 응답 완료 후 `[Memory Hook]` 관련 에러 없이 정상 종료되는가?

> 세션을 종료(`quit`)한 뒤, 다시 `python consultation_agent.py`로 **새 세션**을 시작하세요.

---

### 새 세션 시작 — 아래 3개 턴을 순서대로 입력하세요:

### Turn 1

```
안녕하세요, 지난번에 두통으로 상담받았었는데요.
```

**확인:** Hook이 `retrieve_memory`를 호출하고, 사전 준비에서 저장한 "두통, 오후에 심해짐" 이력을 참조하여 응답하는가?

### Turn 2

```
두통은 나아졌는데, 건강검진 결과가 궁금합니다. 분석해 주세요.
```

**확인:** 세션 컨텍스트가 유지되고, `analyze_lab_values`가 호출되는가?

### Turn 3

```
콜레스테롤 관리 방법을 알려주세요.
```

**확인:** Turn 2의 분석 결과를 참조하여 콜레스테롤 235에 맞는 맞춤 권고를 제공하는가?

---

## 검증

- ✅ `BeforeInvocationEvent` Hook으로 과거 이력 자동 조회 동작 확인
- ✅ `AfterInvocationEvent` Hook으로 상담 내용 자동 저장 동작 확인
- ✅ 3턴 연속 대화에서 컨텍스트가 유지됨
- ✅ 에이전트가 이전 턴의 검사 결과를 참조하여 개인화된 응답 제공

---

완료 후 [Phase 2-C: Observability 설정](./033_observability.md)로 이동하세요.

---

## 부록: 정답 코드 — memory_hook.py

<details>
<summary>정답 코드 펼치기 (클릭)</summary>

```python
"""
AgentCore Memory Hook
- BeforeInvocationEvent: 장기 메모리에서 환자 이력 조회 → 컨텍스트 주입
- AfterInvocationEvent: 대화 내용을 단기 메모리에 저장
"""
import os
import boto3
import json
from datetime import datetime
from strands.hooks import HookProvider, HookRegistry, BeforeInvocationEvent, AfterInvocationEvent


class AgentCoreMemoryHook(HookProvider):
    """AgentCore Memory를 Strands Agent Hook으로 통합합니다."""
    
    def __init__(self, memory_id: str, actor_id: str, region: str = "us-west-2"):
        self.memory_id = memory_id
        self.actor_id = actor_id
        self.session_id = f"session-{actor_id}-{datetime.utcnow().strftime('%Y%m%d')}"
        self.client = boto3.client("bedrock-agentcore", region_name=region)
    
    def register_hooks(self, registry: HookRegistry) -> None:
        """Hook 이벤트에 콜백을 등록합니다."""
        # 정답: BeforeInvocationEvent, AfterInvocationEvent
        registry.add_callback(BeforeInvocationEvent, self.retrieve_memory)
        registry.add_callback(AfterInvocationEvent, self.save_memory)
    
    def retrieve_memory(self, event: BeforeInvocationEvent) -> None:
        """에이전트 호출 전, 장기 메모리에서 환자 이력을 조회합니다."""
        try:
            response = self.client.retrieve_memory_records(
                memoryId=self.memory_id,
                namespace=f"/preferences/{self.actor_id}/",
                searchCriteria={
                    "searchQuery": f"환자 {self.actor_id}의 상담 이력"
                }
            )
            
            records = response.get("memoryRecordSummaries", [])
            if records:
                memory_context = "\n## 이전 상담 이력 (메모리에서 조회됨)\n"
                for record in records:
                    content = record.get("content", {}).get("text", "")
                    memory_context += f"- {content}\n"
                
                if hasattr(event, 'agent') and event.agent.system_prompt:
                    event.agent.system_prompt += memory_context
        except Exception as e:
            print(f"[Memory Hook] 이력 조회 실패: {e}")
    
    def save_memory(self, event: AfterInvocationEvent) -> None:
        """에이전트 응답 후, 대화 내용을 단기 메모리에 저장합니다."""
        try:
            self.client.create_event(
                memoryId=self.memory_id,
                actorId=self.actor_id,
                sessionId=self.session_id,
                eventTimestamp=datetime.utcnow(),
                payload=[{
                    "conversational": {
                        "content": {"text": str(event.result) if event.result else ""},
                        "role": "ASSISTANT"
                    }
                }]
            )
        except Exception as e:
            print(f"[Memory Hook] 저장 실패: {e}")
```

</details>
