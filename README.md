# Healthcare Agentic AI Workshop

## AWS AgentCore 기반 의료 AI 에이전트 시스템 구축 (2.5일)

> 본 워크샵에서 사용하는 환자 데이터는 가상의 합성 데이터입니다.
> 실제 환자 정보가 아니며, 교육 목적으로만 사용됩니다.

---

## 과정 개요

| 항목 | 내용 |
|------|------|
| 과정명 | Healthcare Agentic AI Workshop |
| 기간 | 2.5일 (교육 16.5h + 휴식/점심 별도) |
| 레벨 | 300 (고급) |
| 대상 | AI/ML 엔지니어, 헬스케어 IT 개발자, 솔루션 아키텍트 |
| 핵심 기술 | Amazon Bedrock AgentCore, Strands Agents SDK, Claude Sonnet 4.5 |
| 실습 비중 | 70%+ |
| 리전 | us-west-2 (Oregon) |
| IDE | VS Code Server (EC2, 브라우저 접속) |
| 시나리오 | 환자 "김민수"의 종합건강검진 AI 분석 시스템 구축 |

---

## 커리큘럼 구성

```
Day 1                    Day 2                         Day 3 (반일)
━━━━━━━━━━━━━━━━━━━━    ━━━━━━━━━━━━━━━━━━━━━━━━━    ━━━━━━━━━━━━━━━━━
[싱글 에이전트 구축]      [멀티 에이전트 + 보안]          [풀스택 웹 앱]

Phase 1: 에이전트 구축    Phase 3: 멀티 에이전트          Phase 5: 풀스택 웹 앱
 └ 상담 에이전트            └ Triage/Analysis/Rec.        └ Claude Code
 └ Runtime 배포             └ Supervisor Agent            └ AgentCore 배포
                            └ 컨텍스트 엔지니어링          └ MCP PDF 서버
Phase 2: 도구 + 운영       └ 프롬프트 캐시               └ Streamlit 배포
 └ 혈액검사 도구
 └ Memory (STM/LTM)      Phase 4: 보안 + 평가         결과 공유 + 수료
 └ Observability            └ Policy + Guardrails
                            └ Evaluations
                            └ 침투 테스트
```

---

## 콘텐츠 구조

```
content/
├── 010_opening/
│   └── 011_environment_setting.md        # 환경 설정 (VS Code, AWS)
├── 020_phase1_consultation/
│   ├── 021_agent_build.md                # Phase 1-A: 상담 에이전트 구축
│   └── 022_runtime_deploy.md             # Phase 1-B: Runtime 배포
├── 030_phase2_tools/
│   ├── 031_lab_tools.md                  # Phase 2-A: 혈액 검사 도구
│   ├── 032_memory.md                     # Phase 2-B: 메모리 (STM/LTM)
│   └── 033_observability.md              # Phase 2-C: Observability
├── 040_phase3_multiagent/
│   ├── 041_specialist_agents.md          # Phase 3-A: 전문 에이전트 3개
│   ├── 042_supervisor.md                 # Phase 3-B: Supervisor Agent
│   └── 043_context_engineering.md        # Phase 3-C: 컨텍스트 엔지니어링 + 캐시
├── 050_phase4_security/
│   ├── 051_guardrails_policy.md          # Phase 4-A: Guardrails + Cedar
│   ├── 052_evaluations.md                # Phase 4-B: 평가 파이프라인
│   └── 053_penetration_test.md           # Phase 4-C: 보안 침투 테스트
└── 060_phase5_production/
    ├── 060_claude_code_setup.md           # Claude Code on Bedrock 설정
    └── 061_fullstack_app.md              # Phase 5: 풀스택 웹 앱
```

---

## Day 1: 싱글 에이전트 구축 및 도구 통합

| 시간 | 내용 | 유형 |
|------|------|------|
| 09:00-09:20 | 오프닝 + 환경 확인 | 이론 |
| 09:20-10:20 | **Phase 1-A**: 의료 상담 에이전트 구축 (Strands SDK) | 실습 |
| 10:30-10:40 | AgentCore Runtime 배포 개념 | 이론 |
| 10:40-11:30 | **Phase 1-B**: Runtime 배포 + API 호출 테스트 | 실습 |
| 11:30-11:40 | Phase 1 산출물 확인 | 리뷰 |
| 11:40-12:00 | 의료 도구 설계 원칙 + 환자 데이터 구조 | 이론 |
| 12:00-13:00 | **점심** | - |
| 13:00-14:15 | **Phase 2-A**: 혈액 검사 도구 구현 + 에이전트 통합 | 실습 |
| 14:25-14:35 | AgentCore Memory 개념 (STM/LTM) | 이론 |
| 14:35-15:35 | **Phase 2-B**: 메모리 통합 + 멀티턴 대화 | 실습 |
| 15:45-15:55 | AgentCore Observability 개념 | 이론 |
| 15:55-16:40 | **Phase 2-C**: Observability + 감사 로그 | 실습 |
| 16:40-17:00 | Day 1 종합 정리 | 리뷰 |

**Day 1 산출물**: 배포된 싱글 에이전트 (검사 결과 조회 + 멀티턴 대화 + 모니터링)

---

## Day 2: 멀티 에이전트 시스템 + 보안 + 평가

| 시간 | 내용 | 유형 |
|------|------|------|
| 09:00-09:10 | Day 1 리캡 + 체크포인트 코드 배포 | 리뷰 |
| 09:10-09:30 | 멀티 에이전트 패턴 소개 (Agent-as-Tool) | 이론 |
| 09:30-10:10 | **Phase 3-A**: 전문 에이전트 3개 구현 (스켈레톤 코드 제공) | 실습 |
| 10:10-10:20 | Supervisor 오케스트레이션 패턴 | 이론 |
| 10:20-11:00 | **Phase 3-B**: Supervisor Agent + 종합 보고서 생성 | 실습 |
| 11:10-11:20 | 컨텍스트 엔지니어링 + 프롬프트 캐시 개념 | 이론 |
| 11:20-12:00 | **Phase 3-C**: 컨텍스트 엔지니어링 + 프롬프트 캐시 적용 | 실습 |
| 12:00-13:00 | **점심** | - |
| 13:00-13:25 | AgentCore Policy + Bedrock Guardrails 개념 | 이론 |
| 13:25-14:25 | **Phase 4-A**: PHI 필터링 + Cedar 정책 | 실습 |
| 14:35-14:45 | AgentCore Evaluations 개념 (LLM-as-Judge) | 이론 |
| 14:45-15:45 | **Phase 4-B**: 평가 파이프라인 구축 | 실습 |
| 15:55-16:55 | **Phase 4-C**: 보안 침투 테스트 (8개 공격 시나리오) | 실습 |
| 16:55-17:10 | Day 2 종합 정리 | 리뷰 |

**Day 2 산출물**: 멀티 에이전트 시스템 + 컨텍스트 엔지니어링/프롬프트 캐시 + 보안 정책 + 평가 리포트 + 침투 테스트 결과

---

## Day 3 (0.5일): 풀스택 의료 AI 웹 애플리케이션

Claude Code on Bedrock를 활용하여 Day 1-2에서 학습한 모든 요소를 통합한 프로덕션급 풀스택 웹 앱을 구축합니다.

| 시간 | 내용 | 유형 |
|------|------|------|
| 09:00-09:10 | Day 1-2 리캡 + Day 3 목표 소개 | 리뷰 |
| 09:10-09:30 | Claude Code on Bedrock 환경 설정 | 실습 |
| 09:30-12:00 | **Phase 5**: 풀스택 의료 AI 웹 앱 구축 | 실습 |
| | AgentCore Runtime 배포 (Supervisor + 3 전문 에이전트) | |
| | 4개 에이전트 상호작용 통합 (건강검진 보고서 생성) | |
| | Supervisor에 Guardrail 설정 | |
| | 3개 전문 에이전트에 프롬프트 캐시 적용 | |
| | S3 건강검진 데이터 업로드 → 분석 파이프라인 | |
| | 4개 에이전트 공유 메모리 설정 | |
| | @mcp-z/mcp-pdf MCP 서버 연결 (PDF 보고서 생성) | |
| | Streamlit 프론트엔드 배포 (AWS) | |
| 12:00-12:20 | 앱 구축 프로젝트 결과 공유 및 피드백 | 리뷰 |
| 12:20-12:40 | 프로덕션 규모 앱 배포시 고려사항 설명 및 수료 | 클로징 |

**Day 3 산출물**: 프로덕션 Ready 풀스택 웹 애플리케이션 + PDF 보고서

---

## 시작하기

[Day 1 오프닝: 환경 설정 →](./content/010_opening/011_environment_setting.md)

---

## 제작 정보

본 워크샵은 <img src="https://kiro.dev/favicon.ico" alt="Kiro" width="32" height="32"> [Kiro](https://kiro.dev)로 생성하였으며, HITL(Human-in-the-Loop)을 통해 컨텐츠의 정확성을 검수했습니다.
