
# Phase 3-C: 컨텍스트 엔지니어링 + 프롬프트 캐시 (45분)

## 학습 목표

멀티 에이전트 시스템에서 컨텍스트를 최적화하고, 프롬프트 캐시를 적용하여 비용과 지연 시간을 줄입니다.

---

## 이론: 컨텍스트 엔지니어링 (10분 브리핑)

### 프롬프트 엔지니어링 vs 컨텍스트 엔지니어링

| | 프롬프트 엔지니어링 | 컨텍스트 엔지니어링 |
|--|-------------------|-------------------|
| 초점 | "어떻게 질문할 것인가" | "어떤 정보를 LLM에 넣을 것인가" |
| 범위 | 사용자 프롬프트 최적화 | 시스템 프롬프트 + 도구 결과 + 이력 전체 관리 |
| 핵심 문제 | 좋은 답변 유도 | 컨텍스트 윈도우 한계 내 최적 배치 |
| 비용 영향 | 낮음 | **높음** (토큰 수 = 비용) |

### 왜 멀티 에이전트에서 중요한가?

싱글 에이전트 대비 멀티 에이전트는 토큰 소비가 급증합니다:

```
싱글 에이전트 (Phase 1-2):
  시스템 프롬프트: ~500 토큰
  사용자 입력: ~100 토큰
  도구 결과: ~200 토큰
  총 입력: ~800 토큰 / 호출

멀티 에이전트 (Phase 3):
  Supervisor 시스템 프롬프트: ~800 토큰
  ├ Triage 호출: ~1,500 토큰 (프롬프트 + 응답)
  ├ Analysis 호출: ~2,000 토큰
  └ Recommendation 호출: ~2,000 토큰
  Supervisor 종합: 이전 결과 모두 컨텍스트에 포함
  총 입력: ~8,000+ 토큰 / 세션

→ 비용 10배 증가, 지연 시간 3-4배 증가
```

### 컨텍스트 엔지니어링 전략

| 전략 | 설명 | 효과 |
|------|------|------|
| **시스템 프롬프트 압축** | 불필요한 예시/설명 제거, 핵심 규칙만 유지 | 토큰 30-50% 절감 |
| **도구 결과 요약** | 전체 JSON이 아닌 핵심 요약만 전달 | 토큰 50-70% 절감 |
| **데이터 포맷 최적화** | JSON 대신 토큰 효율이 높은 포맷 사용 | 토큰 20-40% 절감 |
| **이력 윈도우** | 최근 N턴만 유지, 오래된 것은 요약으로 대체 | 무한 증가 방지 |
| **프롬프트 캐시** | 반복되는 컨텍스트를 캐싱 | 비용 90% 절감 + 지연 85% 감소 |

> **권장**: 도구 → 에이전트 결과 전달 시 JSON 대신 자연어 요약이나 마크다운 표를 사용하면
> 동일 정보를 절반의 토큰으로 전달할 수 있습니다.

### 프롬프트 캐시란?

Amazon Bedrock의 **프롬프트 캐시**는 동일한 프롬프트 접두사(prefix)를 캐싱하여:
- 캐시 히트 시 **입력 토큰 비용 90% 절감**
- **지연 시간 85% 감소** (TTFT — Time To First Token)
- TTL(Time To Live) 동안 유지 (최소 5분 ~ 최대 1시간)

```
[캐시 미적용]
매 호출마다: 시스템 프롬프트(800토큰) + 도구 정의(500토큰) + 사용자 입력(100토큰)
→ 전체 1,400 토큰을 매번 처리

[캐시 적용]
첫 호출: 시스템 프롬프트(800) + 도구 정의(500) = 1,300토큰 캐싱 (캐시 쓰기)
이후 호출: 캐시 히트(1,300토큰 무료) + 사용자 입력(100토큰)만 처리
→ 입력 비용 93% 절감
```

**캐시가 효과적인 조건:**
- 시스템 프롬프트가 길고 고정적 (변경 안 됨)
- 동일 에이전트에 반복 요청이 많음 (워크샵 시나리오에 해당)
- 캐시할 접두사가 모델별 최소 토큰 이상

### Bedrock 프롬프트 캐시 지원 모델 및 제한

| 모델 | 최소 토큰 (캐시 포인트당) | 최대 캐시 포인트 수 | TTL | 캐시 가능 필드 |
|------|:---:|:---:|:---:|------|
| **Claude Sonnet 4.5** (본 워크샵) | 4,096 | 4 | 5분, **1시간** | system, messages, tools |
| Claude Sonnet 4.6 | 1,024 | 4 | 5분 | system, messages, tools |
| Claude Opus 4.5 | 4,096 | 4 | 5분, 1시간 | system, messages, tools |
| Claude Opus 4.6 | 4,096 | 4 | 5분 | system, messages, tools |
| Claude Haiku 4.5 | 4,096 | 4 | 5분, 1시간 | system, messages, tools |
| Claude Opus 4 | 1,024 | 4 | 5분 | system, messages, tools |
| Claude 3.7 Sonnet | 1,024 | 4 | 5분 | system, messages, tools |
| GPT-5.6 (Sol/Terra/Luna) | 1,024 | 4 | 30분 | input_text, input_image, input_file |
| Amazon Nova (Lite/Pro) | 자동 | 자동 | 자동 | system, messages (자동 캐시) |

**캐시 가능 데이터:**
- `system` — 시스템 프롬프트 (역할, 규칙, 도메인 지식)
- `tools` — 도구 정의 (이름, 설명, 파라미터 스키마)
- `messages` — 대화 이력 (이전 턴의 user/assistant 메시지)

**TTL 동작:**
- 기본 5분: 캐시 히트가 발생하면 TTL이 리셋됨 (5분 내 재호출 시 유지)
- 1시간 옵션: `ttl: "1h"`를 명시적으로 설정해야 적용 (Claude Sonnet 4.5, Opus 4.5, Haiku 4.5만 지원)
- TTL 내에 캐시 히트가 없으면 캐시 만료

**본 워크샵 모델(Claude Sonnet 4.5)의 제한:**
- 최소 **4,096 토큰**이 있어야 캐시 포인트가 동작
- Supervisor 시스템 프롬프트(~800) + 도구 정의(~500) = ~1,300토큰 → **단독으로는 부족**
- 해결: messages(대화 이력)까지 포함하면 4,096 초과 → 두 번째 호출부터 캐시 히트
- 또는 시스템 프롬프트를 의도적으로 길게 작성 (상세 가이드라인 포함)

---

## 실습 시작

### Step 1: 캐시 미적용 상태에서 토큰 사용량 측정

먼저 현재 Supervisor Agent의 토큰 소비를 측정합니다:

```bash
cd ~/agentcore/src
touch measure_tokens.py
```

`measure_tokens.py`를 열고 아래 코드를 작성하세요:

```python
"""
토큰 사용량 측정 — 프롬프트 캐시 적용 전후 비교
"""
import time
from supervisor_agent import supervisor_agent


def measure_invocation(prompt: str, label: str):
    """에이전트를 호출하고 토큰 사용량과 응답 시간을 측정합니다."""
    start = time.time()
    response = supervisor_agent(prompt)
    elapsed = time.time() - start
    
    # Strands Agent의 metrics에서 토큰 사용량 추출
    metrics = response.metrics
    usage = metrics.accumulated_usage
    input_tokens = usage.get("inputTokens", 0)
    output_tokens = usage.get("outputTokens", 0)
    latency_ms = metrics.accumulated_metrics.get("latencyMs", 0)
    
    print(f"\n[{label}]")
    print(f"  응답 시간: {elapsed:.1f}초 (latency: {latency_ms}ms)")
    print(f"  입력 토큰: {input_tokens:,}")
    print(f"  출력 토큰: {output_tokens:,}")
    print(f"  총 토큰: {input_tokens + output_tokens:,}")
    
    return {
        "elapsed": elapsed,
        "input_tokens": input_tokens,
        "output_tokens": output_tokens
    }


if __name__ == "__main__":
    print("=" * 60)
    print("  토큰 사용량 측정")
    print("=" * 60)
    
    # 동일 프롬프트를 3회 호출하여 일관성 확인
    results = []
    for i in range(3):
        result = measure_invocation(
            "patient-001(김민수, 45세 남성)의 종합 건강검진 결과를 보고 증상 분류까지만 해주세요.",
            f"호출 #{i+1}"
        )
        results.append(result)
    
    # 평균 계산
    avg_input = sum(r["input_tokens"] for r in results) / len(results)
    avg_elapsed = sum(r["elapsed"] for r in results) / len(results)
    
    print(f"\n{'═'*60}")
    print(f"  평균 입력 토큰: {avg_input:,.0f}")
    print(f"  평균 응답 시간: {avg_elapsed:.1f}초")
    print(f"  3회 호출 총 입력 토큰: {sum(r['input_tokens'] for r in results):,}")
    print(f"{'═'*60}")
```

실행:

```bash
uv run python measure_tokens.py
```

> 3회 호출의 평균 입력 토큰과 응답 시간을 기록해 두세요.

---

### Step 2: 프롬프트 캐시 적용

Supervisor Agent에 프롬프트 캐시를 적용합니다. `supervisor_agent.py`의 모델 설정을 수정하세요:

```python
# supervisor_agent.py — 프롬프트 캐시 적용
from strands import Agent
from strands.models import BedrockModel

# 프롬프트 캐시에 필요한 import
from strands.models.bedrock import CacheConfig, CacheToolsConfig

supervisor_agent = Agent(
    model=BedrockModel(
        model_id="global.anthropic.claude-sonnet-4-5-20250929-v1:0",
        region_name="us-west-2",
        # 프롬프트 캐시 설정
        cache_config=________,  # TODO ①: 자동 캐시 전략을 설정하세요 (CacheConfig 사용)
        cache_tools=________,   # TODO ②: 도구 정의 캐시를 1시간 TTL로 설정하세요
    ),
    tools=[triage_specialist, analysis_specialist, recommendation_specialist],
    system_prompt=SUPERVISOR_SYSTEM_PROMPT
)
```

---

### Step 3: 캐시 적용 후 토큰 사용량 재측정

캐시를 적용한 상태에서 동일하게 3회 호출합니다:

```bash
uv run python measure_tokens.py
```

**기대 결과 비교:**

| 지표 | 캐시 미적용 | 캐시 적용 | 절감 |
|------|-----------|----------|------|
| 입력 토큰 (호출당) | ~8,000 | ~1,000 (캐시 히트) | ~87% |
| 응답 시간 (TTFT) | ~3초 | ~0.5초 | ~83% |
| 3회 호출 총 비용 | 약 $0.024 | 약 $0.004 | ~83% |

> **참고**: 첫 번째 호출은 캐시 쓰기(cache write)로 약간 더 느릴 수 있습니다.
> 두 번째 호출부터 캐시 히트(cache read)로 극적인 절감 효과가 나타납니다.

---

### Step 4: 캐시 효과 분석

Bedrock의 응답에서 캐시 관련 메트릭을 확인할 수 있습니다:

```python
# response.metrics에서 캐시 정보 확인
usage = response.metrics.accumulated_usage
print(f"캐시 읽기 토큰: {usage.get('cacheReadInputTokens', 0)}")
print(f"캐시 쓰기 토큰: {usage.get('cacheWriteInputTokens', 0)}")
print(f"일반 입력 토큰: {usage.get('inputTokens', 0)}")
print(f"총 토큰: {usage.get('totalTokens', 0)}")
```

| 메트릭 | 의미 |
|--------|------|
| `cacheWriteInputTokens` | 첫 호출 시 캐시에 쓴 토큰 수 |
| `cacheReadInputTokens` | 이후 호출 시 캐시에서 읽은 토큰 수 (비용 90% 절감) |
| `inputTokens` | 캐시되지 않은 입력 토큰 (사용자 메시지 등) |

---

## 컨텍스트 엔지니어링 모범 사례

| # | 전략 | 적용 포인트 |
|---|------|-----------|
| 1 | **시스템 프롬프트를 프롬프트 앞에 배치** | 캐시 접두사로 활용 (변하지 않는 부분을 앞에) |
| 2 | **도구 정의를 고정** | 도구 목록이 변하지 않으면 캐시 히트율 상승 |
| 3 | **환자 기본 정보를 시스템 프롬프트에 포함** | 동일 환자 반복 상담 시 캐시 효과 극대화 |
| 4 | **대화 이력은 뒤에 배치** | 변하는 부분을 접미사로 → 접두사 캐시 유지 |
| 5 | **불필요한 예시 제거** | 토큰 수 자체를 줄이면 캐시 미적용이라도 비용 절감 |

---

## 검증

- [ ] 캐시 미적용 상태에서 3회 호출 토큰 측정 완료
- [ ] 프롬프트 캐시 설정 적용
- [ ] 캐시 적용 후 토큰 절감 확인 (입력 토큰 80%+ 감소)
- [ ] 응답 시간 단축 확인

---

## 🏆 Challenge Task

1. 각 전문 에이전트(Triage, Analysis, Recommendation)에도 프롬프트 캐시를 적용하고, Supervisor 전체 파이프라인의 총 비용 절감률을 계산하세요
2. TTL을 1시간으로 설정하고, 1시간 동안 반복 호출 시 캐시 히트율을 모니터링하세요
3. 시스템 프롬프트를 A/B 테스트하여 "짧은 프롬프트(300토큰) vs 긴 프롬프트(800토큰)"의 품질/비용 트레이드오프를 비교하세요

---

완료 후 [Phase 4-A: 보안 강화](../050_phase4_security/051_guardrails_policy.md)로 이동하세요.

---

## 부록: 정답 코드

<details>
<summary>TODO ①~② 정답: 프롬프트 캐시 설정 (클릭하여 펼치기)</summary>

```python
from strands.models.bedrock import CacheConfig, CacheToolsConfig

supervisor_agent = Agent(
    model=BedrockModel(
        model_id="global.anthropic.claude-sonnet-4-5-20250929-v1:0",
        region_name="us-west-2",
        cache_config=CacheConfig(strategy="auto"),                    # TODO ① 정답
        cache_tools=CacheToolsConfig(type="default", ttl="1h"),      # TODO ② 정답
    ),
    tools=[triage_specialist, analysis_specialist, recommendation_specialist],
    system_prompt=SUPERVISOR_SYSTEM_PROMPT
)
```

| # | 정답 | 설명 |
|---|------|------|
| ① | `CacheConfig(strategy="auto")` | 시스템 프롬프트에 자동으로 캐시 포인트 삽입 |
| ② | `CacheToolsConfig(type="default", ttl="1h")` | 도구 정의를 1시간 TTL로 캐시 |

</details>
