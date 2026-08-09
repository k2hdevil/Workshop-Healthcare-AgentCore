---
title: "소개 및 환경 설정"
weight: 10
time: "20분"
---

# Day 1 오프닝: 워크샵 소개 및 환경 설정

## 학습 목표

이 모듈을 완료하면 다음을 할 수 있습니다:
- 워크샵의 전체 흐름과 시나리오를 이해합니다
- 실습 환경(IDE, SDK, AWS 인증)이 정상 동작하는 것을 확인합니다

## 워크샵 시나리오

**환자 김민수** (45세, 남성)가 종합건강검진을 받았습니다.

일부 검사 항목에서 이상 소견이 발견되어, 우리는 AI 기반 분석 시스템을 구축하여:
1. 결과를 분류하고 (Triage)
2. 심층 분석하며 (Analysis)
3. 개인화된 건강 관리 계획을 제공합니다 (Recommendation)

### 비정상 소견 5개

| 검사 항목 | 수치 | 판정 |
|-----------|------|------|
| WBC | 12.5 (10³/μL) | 높음 (정상: 4.0-10.0) |
| 공복혈당 | 115 mg/dL | 높음 (정상: <100) |
| 총콜레스테롤 | 235 mg/dL | 높음 (정상: <200) |
| LDL | 145 mg/dL | 높음 (정상: <130) |
| 중성지방 | 180 mg/dL | 높음 (정상: <150) |

### 대한민국 의료 규정

본 시스템은 아래 법규를 준수합니다:

- **의료법 제27조**: 의료인이 아니면 누구든지 의료행위를 할 수 없습니다.
- **개인정보보호법**: 원칙적으로 병원과 의료기관은 환자의 동의 없이 환자 정보를 외부에 공개하거나 타인에게 제공할 수 없습니다. 다만 PHI/PII는 자동 마스킹됩니다.

---

## 환경 확인

VS Code Server 터미널을 열고 아래 명령어를 순서대로 실행하세요.

### 1단계: 작업 디렉토리 생성 및 이동

```bash
mkdir -p ~/agentcore
cd ~/agentcore
```

### 2단계: VS Code 작업 디렉토리 변경

VS Code에서 작업 폴더를 `~/agentcore`로 변경하세요:

1. 좌측 상단 메뉴 (≡) → **File** → **Open Folder**
2. 경로에 `~/agentcore` 입력 → **OK** 클릭
3. 페이지가 새로고침되면 터미널을 다시 열어주고 아래와 같은 명령어를 실행해주세요. (Terminal → New Terminal)

```bash
cd ~/agentcore
```
> 이후 모든 실습은 이 디렉토리에서 진행됩니다.

### 2단계: Git 사용자 정보 및 기본 브랜치 설정
다음 명령에서 `Your Name`과 `you@example.com`을 자신의 이름과 이메일 주소로 바꾸고 실행하세요.

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main
```

### 3단계: pyproject.toml 생성

아래 내용으로 `pyproject.toml` 파일을 생성하세요:

1. VS Code 좌측 Explorer에서 **agentcore** 폴더 우클릭 → **New File** → `pyproject.toml` 입력
2. 아래 내용을 복사하여 붙여넣기 후 저장 (Ctrl+S)

```toml
[project]
name = "healthcare-agentic-ai-workshop"
version = "1.0.0"
description = "Healthcare Agentic AI Workshop - AWS AgentCore 기반 의료 AI 에이전트 시스템"
requires-python = ">=3.11"
dependencies = [
    "strands-agents>=1.0.0",
    "strands-agents-tools>=0.8.0",
    "bedrock-agentcore>=1.0.0",
    "boto3>=1.35.0",
    "rich>=13.0.0",
    "httpx>=0.27.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0.0",
    "ruff>=0.5.0",
]

[dependency-groups]
dev = [
    "pytest>=8.0.0",
    "ruff>=0.5.0",
]
```

> 파일이 `~/agentcore/pyproject.toml` 경로에 저장되었는지 확인하세요.
### 4단계: 종속성 설치 (uv sync)

```bash
cd ~/agentcore
uv sync --all-extras
```

예상 출력:
```
Resolved X packages in Xs
Installed X packages in Xs
```

> `uv sync --all-extras`는 `pyproject.toml`에 정의된 기본 의존성과 optional 의존성(pytest, ruff 등)을 모두 설치합니다.

### 4.5단계: AgentCore CLI 설치

AgentCore Runtime에 배포할 때 사용하는 CLI 도구를 설치합니다. Node.js 20 이상이 필요하므로 순서대로 진행하세요.

**1. Node.js 20 설치 (nvm 사용):**

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
source ~/.bashrc
nvm install 20
```

설치 확인:

```bash
node --version   # v20.x.x 이상이어야 합니다
npm --version
```

**2. AgentCore CLI 설치:**

```bash
npm install -g @aws/agentcore
```

설치 확인:

```bash
agentcore --version
```

**3. AWS CDK 및 TypeScript 설치, 부트스트랩:**

AgentCore CLI는 내부적으로 AWS CDK(TypeScript)를 사용하여 인프라를 배포합니다. CDK CLI와 TypeScript 컴파일러를 설치하고 AWS 계정을 부트스트랩하세요:

```bash
npm install -g aws-cdk typescript@5
cdk --version
tsc --version
```

CDK 부트스트랩 (계정당 최초 1회만 실행):

```bash
cdk bootstrap aws://$(aws sts get-caller-identity --query Account --output text)/us-west-2
```

> CDK 부트스트랩은 CloudFormation 배포에 필요한 S3 버킷과 IAM 역할을 생성합니다. 이미 부트스트랩된 계정이라면 건너뛰어도 됩니다.

### 5단계: AWS 인증 구성

```bash
aws sts get-caller-identity
```

예상 출력:
```json
{
    "Account": "123456789012",
    "Arn": "arn:aws:sts::123456789012:assumed-role/workshop-ide-role/...",
    "UserId": "AROA..."
}
```

출력된 `Arn`에서 Role 이름을 확인하세요 (예: `workshop-ide-role`).
이 Role에 AdministratorAccess 정책을 추가합니다:

1. AWS 콘솔 → **IAM** → 좌측 메뉴 **Roles**
2. 위에서 확인한 Role 이름 검색 → 클릭
3. **Permissions** 탭 → **Add permissions** → **Attach policies**
4. `AdministratorAccess` 검색 → 체크 → **Add permissions** 클릭

또는 터미널에서 아래 명령어로 추가할 수 있습니다:

```bash
# Role 이름을 위 출력에서 확인한 값으로 교체하세요
ROLE_NAME="workshop-ide-role"

aws iam attach-role-policy \
  --role-name $ROLE_NAME \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
```

> 워크샵 종료 후 반드시 이 정책을 제거하세요.

### 6단계: Bedrock 모델 호출 테스트

아래 Python 스크립트를 실행하여 Bedrock 모델 호출이 정상 동작하는지 확인하세요:

```bash
uv run python -c "
import boto3, json

client = boto3.client('bedrock-runtime', region_name='us-west-2')
response = client.invoke_model(
    modelId='global.anthropic.claude-sonnet-4-5-20250929-v1:0',
    body=json.dumps({
        'anthropic_version': 'bedrock-2023-05-31',
        'max_tokens': 100,
        'messages': [{'role': 'user', 'content': '안녕하세요, 간단히 자기소개 해주세요.'}]
    })
)
result = json.loads(response['body'].read())
print(result['content'][0]['text'])
"
```

한국어 응답이 출력되면 Bedrock 모델 호출이 정상 동작하는 것입니다.

### 7단계: 환자 데이터 업로드

Local PC에 있는 `patient-001.json` 파일을 VS Code Server에 업로드합니다.

**1. 폴더 생성**

터미널에서 데이터 저장 폴더를 먼저 만드세요:

```bash
mkdir -p ~/agentcore/patient-data
```

**2. VS Code로 파일 업로드**

1. VS Code 좌측 Explorer에서 **patient-data** 폴더를 클릭하여 선택
2. 폴더를 **우클릭** → **Upload...** 선택
3. Local PC에서 `patient-001.json` 파일을 선택하여 업로드

> 또는 Local PC에서 파일을 드래그하여 Explorer의 `patient-data` 폴더에 드롭해도 됩니다.

**3. 업로드 확인**

```bash
cat ~/agentcore/patient-data/patient-001.json | uv run python -m json.tool | head -20
```

예상 출력:
```json
{
    "patient_id": "patient-001",
    "demographics": {
        "name": "김민수",
        "age": 45,
        "gender": "M",
        ...
    }
}
```
---

## 디렉토리 구조

워크샵 코드는 아래 구조로 작성합니다:

```bash
ls ~/agentcore/
```

```
agentcore/
├── patient-data/
│   └── patient-001.json    ← 환자 김민수 건강검진 데이터
├── src/                     ← 실습 코드 작성
├── outputs/                 ← 산출물 저장
└── pyproject.toml           ← 종속성 정의 (uv sync)
```

---

## 트러블슈팅

| 증상 | 해결 |
|------|------|
| `python: command not found` | `source ~/agentcore/.venv/bin/activate` 실행 |
| AWS 인증 실패 | EC2 Instance Profile 확인 — 강사에게 문의 |
| Bedrock 호출 에러 | 모델 액세스 활성화 여부 확인 — 강사에게 문의 |
| 환자 데이터 없음 | `echo $PATIENT_DATA_DIR` 로 경로 확인 |

---

환경 확인이 완료되었으면 다음 단계로 이동하세요.

[Phase 1: 의료 상담 에이전트 구축 →](../020_phase1_consultation/021_agent_build.md)
