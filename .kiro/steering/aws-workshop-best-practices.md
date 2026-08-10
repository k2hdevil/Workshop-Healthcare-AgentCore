---
inclusion: auto
---

# AWS 공식 워크샵 모범 사례

이 문서는 Healthcare Agentic AI Workshop 개발 시 AWS 공식 워크샵 표준을 따르기 위한 가이드라인입니다.

## 콘텐츠 구조

### 디렉토리 구조 표준

본 워크샵은 아래 구조를 따릅니다:

```
content/
├── README.md                          # 워크샵 홈
├── day1/
│   ├── README.md                      # Day 1 개요
│   ├── 00_introduction/               # 오프닝 + 환경 설정
│   │   ├── README.md
│   │   └── 01_theory_agentcore.md     # 이론: Agentic AI + AgentCore
│   ├── 01_phase1_agent/               # Phase 1: 에이전트 구축 + 배포
│   │   ├── README.md
│   │   ├── 01_agent_build.md          # Phase 1-A: 에이전트 구축
│   │   └── 02_runtime_deploy.md       # Phase 1-B: Runtime 배포
│   └── 02_phase2_tools/               # Phase 2: 도구 + 메모리 + Observability
│       ├── README.md
│       ├── 01_lab_tools.md            # Phase 2-A: 혈액 검사 도구
│       ├── 02_memory.md               # Phase 2-B: 메모리 통합
│       └── 03_observability.md        # Phase 2-C: Observability
├── day2/
│   ├── README.md                      # Day 2 개요
│   ├── 03_phase3_multi_agent/         # Phase 3: 멀티 에이전트
│   │   ├── README.md
│   │   ├── 01_specialist_agents.md    # Phase 3-A: 전문 에이전트 3개
│   │   └── 02_supervisor.md           # Phase 3-B: Supervisor Agent
│   └── 04_phase4_security/            # Phase 4: 보안 + 평가
│       ├── README.md
│       ├── 01_policy_guardrails.md    # Phase 4-A: Policy + Guardrails + VPC
│       ├── 02_evaluations.md          # Phase 4-B: Evaluations
│       └── 03_penetration_test.md     # Phase 4-C: 침투 테스트
├── day3/
│   ├── README.md                      # Day 3 개요
│   └── 05_phase5_production/          # Phase 5: 프로덕션 배포
│       ├── README.md
│       ├── 01_vpc_deploy.md           # VPC 배포 + 부하 테스트
│       └── 02_presentation.md         # 팀별 발표
└── 99_cleanup/
    └── README.md                      # 리소스 정리
```

### 모듈 번호 규칙
- Day별 디렉토리: `day1/`, `day2/`, `day3/`
- Phase별 폴더: `00_`, `01_`, `02_`... (Day 내 순서)
- 서브 페이지: `01_`, `02_`, `03_`... (Phase 내 순서)
- Cleanup은 항상 `99_cleanup/`

## 실습 페이지 작성 규칙

### 각 실습 페이지 필수 요소

1. **학습 목표** (페이지 상단): "이 모듈을 완료하면 ~할 수 있습니다"
2. **예상 소요 시간**: 페이지 메타데이터에 명시
3. **단계별 지시사항**: 번호 매긴 순차적 단계
4. **코드 블록**: 복사 가능한 코드 블록 (언어 명시)
5. **검증 단계**: 각 주요 단계 후 "정상 동작 확인" 방법
6. **스크린샷/다이어그램**: 콘솔 조작 시 시각 자료 포함

### 코드 블록 작성 규칙

```python
# 좋은 예: 복사-붙여넣기로 바로 동작
from strands import Agent, tool

agent = Agent(model=BedrockModel(model_id="us.anthropic.claude-sonnet-4-20250514-v1:0"))
```

- 코드 블록에 언어 태그 필수 (python, bash, json, cedar 등)
- 주석으로 각 줄의 의도 설명
- 환경 변수는 `$VARIABLE` 형식으로 통일
- 하드코딩된 ARN, Account ID는 placeholder 사용 (`<YOUR_ACCOUNT_ID>`)

### 콘솔 명령어 규칙

```bash
# 좋은 예: 결과 확인까지 포함
aws sts get-caller-identity
# 예상 출력:
# {
#   "Account": "123456789012",
#   "Arn": "arn:aws:sts::123456789012:assumed-role/...",
#   "UserId": "AROA..."
# }
```

## 실습 설계 원칙

### Just-in-Time 학습
- 이론은 실습 직전에 최소한으로 전달 (최대 15-20분)
- "왜"보다 "어떻게"에 집중
- 배경 지식은 접을 수 있는 섹션(expandable)으로 제공

### 실패 안전 설계
- 각 모듈은 독립적으로 시작 가능해야 함
- 이전 모듈 실패 시 체크포인트에서 재시작 가능
- git tag 또는 S3에 완성 코드 제공

### 시간 여유 확보
- 공식 예상 시간의 1.3배를 실제 배분에 반영
- 빠른 참가자를 위한 Challenge Task 별도 제공
- 느린 참가자를 위한 "빠른 경로" (완성 코드 제공) 안내

## 환경 설정 모범 사례

### 사전 프로비저닝
- CloudFormation/CDK로 모든 인프라 자동 생성
- UserData 또는 SSM으로 소프트웨어 자동 설치
- 참가자가 직접 설치해야 하는 것 최소화

### 환경 변수 표준
```bash
export AWS_REGION=us-west-2
export WORKSHOP_BUCKET=<auto-provisioned>
export PATIENT_DATA_DIR=~/workshop/data
```

### IDE 설정
- VS Code Server (code-server): Cloud9 대체 표준
- Python venv 자동 활성화
- 필요한 확장(Extension) 사전 설치

## 보안 및 정리

### 실습 계정 보안
- IAM 권한은 워크샵에 필요한 최소 범위로 제한
- AdministratorAccess 사용 시 명시적 경고 포함
- 프로덕션 리소스 삭제 방지 (termination protection 등)

### 정리(Cleanup) 모듈 필수
- 모든 워크샵은 마지막에 Cleanup 섹션 포함
- 생성된 리소스 목록 + 삭제 명령어 제공
- 비용 발생 리소스 강조 표시
- CloudFormation 사용 시: `aws cloudformation delete-stack` 안내

## 한국어 워크샵 특화 규칙

### 용어 일관성
- AWS 서비스명은 영문 유지 (Amazon Bedrock, AgentCore)
- 기술 용어는 첫 등장 시 한영 병기: "에이전트(Agent)"
- 이후에는 한글 또는 영문 중 하나로 통일

### 한국 의료 도메인
- 건강검진 판정 기준: 국민건강보험공단 A/B/C/D/R 등급 사용
- 법률 인용: 의료법 제X조, 개인정보보호법 등 조항 번호 명시
- 합성 데이터 사용 시: "본 데이터는 가상의 합성 데이터입니다" 면책 포함

### 문서 작성 톤
- 존댓말 사용 (~합니다, ~하세요)
- 명령형은 부드러운 표현으로: "클릭합니다" → "클릭하세요"
- 기술 설명은 간결하게, 불필요한 수식어 제거

## 품질 체크리스트

코드 작성 전 확인:
- [ ] 모든 코드 블록이 복사-붙여넣기로 동작하는가?
- [ ] 환경 변수가 정확히 설정되어 있는가?
- [ ] 이전 단계 실패 시 복구 경로가 있는가?
- [ ] 예상 소요 시간이 현실적인가?
- [ ] 스크린샷/다이어그램이 최신 콘솔 UI를 반영하는가?
- [ ] Cleanup 절차가 모든 생성 리소스를 커버하는가?
- [ ] 개인 계정 사용자를 위한 비용 안내가 있는가?

## 참고 자료

- AWS Modernization Workshop Template: https://github.com/aws-samples/aws-modernization-workshop-base
- AWS Workshop 구조 예시: https://awsworkshop.io
- Hugo Theme Learn (워크샵 사이트 빌더): https://learn.netlify.app/en/
