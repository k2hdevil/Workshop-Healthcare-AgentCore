# Healthcare Agentic AI Workshop — 커리큘럼 소개

## AWS AgentCore 기반 의료 AI 에이전트 시스템 구축 (2.5일)

---

### 과정 개요

| 항목 | 내용 |
|------|------|
| 과정명 | Healthcare Agentic AI Workshop |
| 기간 | 2.5일 (교육 16.5h + 휴식/점심 별도) |
| 레벨 | 300 (고급) |
| 대상 | AI/ML 엔지니어, 헬스케어 IT 개발자, 솔루션 아키텍트 |
| 핵심 기술 | Amazon Bedrock AgentCore, Strands Agents SDK, Claude Sonnet 4 |
| 실습 비중 | 70%+ |
| 리전 | us-west-2 (Oregon) |
| IDE | VS Code Server (EC2, 브라우저 접속) |
| 시나리오 | 환자 "김민수"의 종합건강검진 AI 분석 시스템 구축 |

---

### 학습 목표

본 워크샵을 수료하면 다음을 할 수 있습니다:

1. Strands Agents SDK로 의료 상담 에이전트를 구축하고 AgentCore Runtime에 배포
2. Agent-as-Tool 패턴으로 멀티 에이전트 협업 시스템 설계 및 구현
3. AgentCore Memory, Observability, Policy, Evaluations를 활용한 프로덕션 운영
4. Bedrock Guardrails로 PHI 필터링 및 Cedar 정책으로 접근 제어 구현
5. 보안 침투 테스트 및 LLM-as-Judge 평가 파이프라인 구축
6. 대한민국 의료법/개인정보보호법을 고려한 AI 시스템 설계

---

### 커리큘럼 구성

```
Day 1                    Day 2                         Day 3 (반일)
━━━━━━━━━━━━━━━━━━━━    ━━━━━━━━━━━━━━━━━━━━━━━━━    ━━━━━━━━━━━━━━━━━
[싱글 에이전트 구축]      [멀티 에이전트 + 보안]          [프로덕션 + 발표]

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

### Day 1: 싱글 에이전트 구축 및 도구 통합

Strands Agents SDK로 의료 상담 에이전트를 처음부터 구축하고, AgentCore Runtime에 배포하여 API로 호출할 수 있는 상태까지 완성합니다. 이후 혈액 검사 도구, 대화 메모리, 모니터링을 차례로 통합하여 실제 사용 가능한 싱글 에이전트 시스템을 완성합니다.

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

### Day 2: 멀티 에이전트 시스템 + 보안 + 평가

Day 1에서 만든 싱글 에이전트를 3개 전문 에이전트(Triage, Analysis, Recommendation)로 확장하고, Supervisor Agent가 이를 조율하여 종합 건강검진 AI 분석 보고서를 생성합니다. 오후에는 Cedar 정책과 Guardrails로 보안을 강화하고, 평가 파이프라인과 침투 테스트로 시스템의 품질과 견고성을 검증합니다.

| 시간 | 내용 | 유형 |
|------|------|------|
| 09:00-09:10 | Day 1 리캡 + 체크포인트 코드 배포 | 리뷰 |
| 09:10-09:30 | 멀티 에이전트 패턴 소개 (Agent-as-Tool) | 이론 |
| 09:30-10:10 | **Phase 3-A**: 전문 에이전트 3개 구현 (스켈레톤 코드 제공) | 실습 |
| 10:10-10:20 | Supervisor 오케스트레이션 패턴 | 이론 |
| 10:20-11:00 | **Phase 3-B**: Supervisor Agent + 종합 건강 검진 보고서 생성 | 실습 |
| 11:10-11:20 | 컨텍스트 엔지니어링 + 프롬프트 캐시 개념 | 이론 |
| 11:20-12:00 | **Phase 3-C**: 컨텍스트 엔지니어링 + 프롬프트 캐시 적용 | 실습 |
| 12:00-13:00 | **점심** | - |
| 13:00-13:25 | AgentCore Policy + Bedrock Guardrails 개념 | 이론 |
| 13:25-14:25 | **Phase 4-A**: PHI 필터링 + Cedar 정책 + VPC 배포 | 실습 |
| 14:35-14:45 | AgentCore Evaluations 개념 (LLM-as-Judge) | 이론 |
| 14:45-15:45 | **Phase 4-B**: 평가 파이프라인 구축 | 실습 |
| 15:55-16:55 | **Phase 4-C**: 보안 침투 테스트 (8개 공격 시나리오) | 실습 |
| 16:55-17:10 | Day 2 종합 정리 | 리뷰 |

**Day 2 산출물**: 멀티 에이전트 시스템 + 컨텍스트 엔지니어링/프롬프트 캐시 + 보안 정책 + 평가 리포트 + 침투 테스트 결과

---

### Day 3 (0.5일): 풀스택 의료 AI 웹 애플리케이션

Claude Code on Bedrock를 활용하여, Day 1-2에서 학습한 모든 요소를 통합한 프로덕션급 풀스택 웹 애플리케이션을 구축합니다. AgentCore Runtime에 4개 에이전트를 배포하고, S3 데이터 연동, Guardrail, 프롬프트 캐시, 공유 메모리, MCP PDF 서버, Streamlit 프론트엔드까지 완성합니다.

| 시간 | 내용 | 유형 |
|------|------|------|
| 09:00-09:10 | Day 1-2 리캡 + Day 3 목표 소개 | 리뷰 |
| 09:10-09:30 | Claude Code on Bedrock 환경 설정 | 실습 |
| 09:30-12:00 | **Phase 5**: 풀스택 의료 AI 웹 앱 구축 | 실습 |
| | ① AgentCore Runtime 배포 (Supervisor + 3 전문 에이전트) | |
| | ② 4개 에이전트 상호작용 구현 (건강검진 보고서 생성) | |
| | ③ Supervisor에 Guardrail 설정 | |
| | ④ 3개 전문 에이전트에 프롬프트 캐시 적용 | |
| | ⑤ S3 건강검진 데이터 업로드 → 분석 파이프라인 | |
| | ⑥ 4개 에이전트 공유 메모리 설정 | |
| | ⑦ markdown2pdf MCP 서버 연결 (PDF 보고서 생성) | |
| | ⑧ Streamlit 프론트엔드 배포 (AWS) | |
| 12:00-12:20 | 앱 구축 프로젝트 결과 공유 및 피드백 | 리뷰 |
| 12:20-12:40 | 프로덕션 규모 앱 배포시 고려사항 설명 및 수료 | 클로징 |

**Day 3 산출물**: 프로덕션 Ready 풀스택 웹 애플리케이션 + PDF 보고서

---

### 사용 AWS 서비스

| 서비스 | 용도 |
|--------|------|
| Amazon Bedrock (Claude Sonnet 4) | LLM 추론 |
| Bedrock AgentCore Runtime | 에이전트 호스팅/배포 |
| Bedrock AgentCore Memory | 대화 이력 관리 (STM + LTM) |
| Bedrock AgentCore Observability | CloudWatch GenAI 모니터링 |
| Bedrock AgentCore Policy | Cedar 기반 접근 제어 |
| Bedrock AgentCore Evaluations | LLM-as-Judge 품질 평가 |
| Bedrock Guardrails | PHI/PII 필터링, 토픽 차단 |
| Amazon CloudWatch | 메트릭/로그/알람 |
| Amazon S3 | 환자 데이터 + 산출물 |

---

### 보안 계층 (4-Layer)

```
Layer 1: AgentCore Policy (Cedar 정책)
  └ 입력 파라미터 검증 + 시간 기반 접근 제어

Layer 2: AgentCore Runtime VPC 배포
  └ "VPC and more" 마법사로 Private Subnet + NAT 생성

Layer 3: Bedrock Guardrails
  └ PHI 마스킹 + 진단/처방 차단 + 프롬프트 인젝션 방어

Layer 4: 감사 로그 (Audit Trail)
  └ 모든 데이터 접근 기록 + 개인정보보호법 증적
```

---

### 대한민국 의료 규정 준수

| 규정 | 워크샵 내 적용 |
|------|--------------|
| 의료법 제27조 | AI 면책 조항 필수 포함 (의료행위 불가 명시) |
| 의료법 제21조 | 감사 로그 5년 보존 설계 |
| 개인정보보호법 (PIPA) | Guardrails PHI 자동 마스킹 |
| 식약처 SaMD 가이드라인 | "참고 정보 제공"으로 한정 (의료기기 비해당) |

---

### 전제 조건

**필수 역량**:
- Python 프로그래밍 (중급 이상)
- AWS 기본 서비스 이해 (IAM, S3)
- REST API / JSON 처리 경험
- 생성형 AI 기본 개념

**권장 사전 학습** (AWS Skill Builder):
- Generative AI Fundamentals
- Building Production-Ready AI Agents with Amazon Bedrock AgentCore

---

### 교육 특징

| 특징 | 설명 |
|------|------|
| 시나리오 기반 | 2.5일 전체가 하나의 환자 시나리오로 연결 |
| 코드 컴플리션 | 단순히 완성된 코드를 실행하는 것이 아닌 빈칸 채우기 방식 제공 |
| 중간 체크포인트 | Day별 git tag로 뒤처진 참가자 구제 |
| Challenge Task | 빠른 학습자를 위한 확장 과제 |
| 보안 레드팀 | 공격자 관점 침투 테스트 (8개 시나리오) |
| 결과 공유 + 수료 | Day 3 프로젝트 결과 공유 및 피드백 |

---

*© 2026 Amazon Web Services, Inc. 또는 자회사. All rights reserved.*
