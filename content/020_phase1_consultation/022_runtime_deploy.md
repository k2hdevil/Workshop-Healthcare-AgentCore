---
title: "Phase 1-B: AgentCore Runtime 배포"
weight: 22
time: "60분"
---

# Phase 1-B: AgentCore Runtime 배포 (60분)

## 학습 목표

구축한 에이전트를 AgentCore Runtime에 배포하고, 외부에서 API로 호출할 수 있는 상태를 만듭니다.

---

## 이론: AgentCore Runtime이란? (10분 브리핑)

AgentCore Runtime은 에이전트를 **관리형 인프라**에서 호스팅하는 서버리스 런타임 환경입니다. 스케일링, 세션 관리, 보안 격리, 인프라 운영을 모두 처리하여 개발자가 에이전트 로직에만 집중할 수 있게 합니다.

### 비관리형(Self-managed) 대비 AgentCore Runtime의 장점

| 항목 | 비관리형 (EC2/ECS/Lambda 직접 운영) | AgentCore Runtime |
|------|--------------------------------------|-------------------|
| **인프라 관리** | 서버 프로비저닝, 패치, 스케일링 직접 수행 | 완전 관리형, 인프라 작업 없음 |
| **세션 격리** | 애플리케이션 레벨에서 직접 구현 | **microVM 기반 세션 격리** — 세션마다 전용 CPU/메모리/파일시스템 |
| **확장성** | Auto Scaling 규칙 직접 설정 | 자동 스케일링 내장 |
| **콜드 스타트** | 컨테이너/Lambda 워밍업 관리 필요 | 실시간 상호작용에 최적화된 빠른 시작 |
| **보안** | 네트워크, IAM, 시크릿 관리 직접 구성 | IAM + OAuth 2.0 인증 내장, VPC 연동 지원 |
| **모니터링** | 커스텀 로깅/메트릭 코드 작성 | **Observability 자동 계측** (추가 코드 불필요) |
| **버전 관리** | 배포 파이프라인 직접 구축 | 불변 버전 + 엔드포인트 기반 롤백 |
| **API 엔드포인트** | API Gateway/ALB 직접 구성 | HTTPS API 자동 생성 |

### 핵심 아키텍처 개념

**microVM 기반 세션 격리:**

```
┌─── AgentCore Runtime ─────────────────────────────────────────┐
│                                                               │
│  ┌─── microVM (세션 A) ───┐    ┌─── microVM (세션 B) ───┐       │
│  │  CPU / Memory / FS     │   │  CPU / Memory / FS     │     │
│  │  ┌──────────────────┐  │   │  ┌──────────────────┐  │     │
│  │  │  Agent Process   │  │   │  │  Agent Process   │  │     │
│  │  │  (환자 김민수)      │  │   │  │  (환자 이영희)      │  │     │
│  │  └──────────────────┘  │   │  └──────────────────┘  │     │
│  └────────────────────────┘   └────────────────────────┘     │
│           ✕ 상호 접근 불가 ✕                                     │
└───────────────────────────────────────────────────────────────┘
```

- 각 세션은 전용 microVM에서 실행됩니다
- 다른 세션 간의 파일이나 메모리 접근이 원천적으로 차단됩니다 (의료 데이터 보호에 핵심)
- 세션 종료 시 microVM이 완전 삭제되고 메모리 상태 모두 정리됩니다

**세션 수명 — 동기식 vs 비동기식:**

| | 동기식 (Synchronous) | 비동기식 (Asynchronous) |
|--|---------------------|----------------------|
| **동작** | 요청 → 처리 → 응답 (한 번에 완료) | 요청 → 즉시 응답 + 백그라운드 처리 계속 |
| **유휴 타임아웃** | 15분 미활동 시 자동 종료 (기본값 900초) | `HealthyBusy` 상태 반환 시 유휴 타임아웃 무시 |
| **최대 수명** | 8시간 (지속 대화 시) | 8시간 |
| **적합한 경우** | 실시간 대화, 즉시 응답 필요 | 장시간 데이터 분석, 보고서 생성 등 |
| **본 워크샵** | Phase 1~4에서 사용 | Phase 5 부하 테스트에서 참고 |

동기식은 대부분의 대화형 에이전트에 적합하며, 비동기식은 수 분~수 시간 걸리는 작업(예: 대량 검사 결과 일괄 분석)에 사용합니다.

**버전과 엔드포인트:**
- Runtime 생성 시 Version 1이 자동 생성됩니다
- 설정 업데이트마다 새 버전이 생성되며, `DEFAULT` 엔드포인트가 최신 버전을 가리킵니다
- 커스텀 엔드포인트(dev, test, prod)를 별도 생성하여 환경별 관리가 가능합니다

### 배포 방식과 배포 도구

AgentCore Runtime은 코드를 받는 **배포 방식** 2가지와, 이를 실행하는 **배포 도구**로 구분됩니다.

**배포 방식 (Runtime이 코드를 받는 형태):**

| 배포 방식 | 설명 | 적합한 경우 | 콜드스타트 |
|-----------|------|-----------|-----------|
| **Direct Code Deploy (CodeZip)** | Python/Node.js 코드와 의존성을 zip으로 패키징하여 업로드. AWS가 런타임 환경(Python, 보안 패치)을 관리. | 빠른 반복 개발, 커스텀 런타임 불필요 시 | ~5초 |
| **Container (ECR)** | Docker 이미지를 ECR에 push하고 Runtime이 pull. 완전한 환경 제어. | GPU 필요, 커스텀 시스템 패키지, 대형 의존성. **본 워크샵에서 사용** | ~10-15초 |

**배포 도구 (배포 방식을 실행하는 인터페이스):**

| 배포 도구 | 설명 | 내부적으로 사용하는 배포 방식 |
|-----------|------|--------------------------|
| **AgentCore CLI** | `agentcore deploy` 한 줄로 패키징+업로드+Runtime 생성을 자동화 | Direct Code Deploy |
| **AWS SDK/API** | `create_agent_runtime` API를 직접 호출. 세밀한 제어 가능. **본 워크샵에서 사용** | Direct Code 또는 Container |

#### AWS SDK 배포 흐름 (본 워크샵)

```
개발자 PC                        AWS
─────────                        ───
1. docker build
   └→ 이미지 빌드
2. docker push                  → 3. ECR에 이미지 저장
                                 → 4. create_agent_runtime API 호출
                                 → 5. Runtime 생성 + DEFAULT 엔드포인트
                                 → 6. 상태: READY

7. invoke_agent_runtime
   └→ API 호출                  → 8. microVM 할당 (이미지에서 컨테이너 시작)
                                 → 9. @app.entrypoint 함수 실행
                                 → 10. 응답 반환
```

---

## 실습: AgentCore Runtime 배포 순서

AWS SDK(boto3)를 사용한 Container 배포 방식으로 배포합니다. 아래 5단계로 진행됩니다:

```
① 엔트리포인트 작성 → ② Docker 이미지 빌드 → ③ 로컬 테스트 → ④ ECR 업로드 + Runtime 생성 → ⑤ 호출 테스트
```

---

### ① 엔트리포인트 작성

AgentCore Runtime은 `@app.entrypoint`로 표시된 함수를 진입점으로 인식합니다. 이 함수가 API 호출 시 실행됩니다.

```bash
cd ~/agentcore/src
touch main.py
```

`main.py` 파일을 열고 아래 코드를 작성하세요. `TODO` 주석이 있는 빈칸(`____`)을 직접 채워서 엔트리포인트를 완성합니다.

> 정말 완성하기 어렵다면 하단의 **부록: 정답 코드**를 참고하세요.

```python
"""
AgentCore Runtime Entrypoint
- Phase 1-A에서 만든 에이전트를 Runtime에 배포
"""

from bedrock_agentcore.runtime import BedrockAgentCoreApp
# 같은 폴더(~/agentcore/src/)의 consultation_agent.py에서 에이전트 인스턴스를 가져옴
from consultation_agent import ________  # TODO: Phase 1-A에서 만든 에이전트 인스턴스를 import하세요

# AgentCore 앱 인스턴스 생성
app = BedrockAgentCoreApp()


@app.________  # TODO: Runtime 진입점 데코레이터를 채우세요
def healthcare_consultation(payload):
    """AgentCore Runtime 엔트리포인트
    
    Runtime이 API 요청을 받으면 이 함수가 호출됩니다.
    
    Args:
        payload: API 호출 시 전달되는 JSON 본문
            - prompt (str): 사용자 입력
            - patient_id (str, optional): 환자 ID
    
    Returns:
        str: 에이전트 응답 텍스트
    """
    user_input = payload.get("prompt", "")
    
    if not user_input:
        return "질문을 입력해 주세요."
    
    # Phase 1-A에서 만든 에이전트 호출
    response = consultation_agent(________)  # TODO: 에이전트에 전달할 입력을 채우세요
    return response.message['content'][0]['text']


# 로컬 실행 시 HTTP 서버로 동작 (배포 후에는 Runtime이 자동 처리)
if __name__ == "__main__":
    app.run(port=8888)
```

**핵심 포인트:**
- `BedrockAgentCoreApp()` — Runtime과 통신하는 앱 인스턴스
- `@app.entrypoint` — 이 함수가 API 요청의 진입점
- `payload` — 클라이언트가 보낸 JSON 요청 본문이 Python dict로 파싱되어 전달됨 (예: `{"prompt": "두통이 3일째"}` → `payload["prompt"]`로 접근)
- `app.run()` — 로컬에서는 HTTP 서버, 배포 후에는 Runtime이 관리

---

### ② Docker 이미지 빌드

Runtime에 배포할 Docker 이미지를 생성합니다. 의존성이 이미지에 포함되므로 콜드스타트가 안정적입니다.

먼저 `Dockerfile`을 생성하세요:

```bash
cd ~/agentcore/src
touch Dockerfile
```

`Dockerfile`을 열고 아래 내용을 작성하세요:

```dockerfile
FROM --platform=linux/arm64 python:3.12-slim

WORKDIR /app

# 의존성 설치
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 에이전트 코드 복사
COPY *.py ./

# AgentCore Runtime이 사용하는 포트
EXPOSE 8080

# 엔트리포인트
CMD ["python", "main.py"]
```

`requirements.txt`를 생성하세요:

```bash
cat > ~/agentcore/src/requirements.txt << 'EOF'
strands-agents>=1.0.0
bedrock-agentcore>=1.0.0
boto3>=1.35.0
EOF
```

Docker 이미지를 빌드합니다 (ARM64 대상):

```bash
cd ~/agentcore/src
docker build --no-cache --platform linux/arm64 -t healthcare-agent:latest .
```

> **참고**: AgentCore Runtime은 ARM64(Graviton) 아키텍처에서 실행됩니다. `--platform linux/arm64` 플래그를 반드시 포함하세요.

---

### ③ 로컬 테스트 (배포 전 확인)

배포 전에 로컬에서 동작을 확인합니다.

> AgentCore Runtime의 기본 포트는 8080이지만, 본 워크샵 환경에서는 VS Code Server가 8080을 사용하므로 `port=8888`로 변경하여 실행합니다.

**터미널 1 — 서버 실행:**

```bash
cd ~/agentcore/src
uv run python main.py
```

**터미널 2 — 호출 테스트:**

VS Code에서 새 터미널을 열어주세요 (Terminal → New Terminal 또는 `Ctrl+Shift+`\`). 서버가 실행 중인 터미널 1은 그대로 두고, 새 터미널에서 아래 명령을 실행합니다:

```bash
curl -s -X POST http://localhost:8888/invocations \
  -H "Content-Type: application/json" \
  -d '{"prompt": "두통이 3일째 지속됩니다"}'
```

응답이 정상적으로 반환되면 로컬 테스트 완료입니다. 터미널 1에서 `Ctrl+C`로 서버를 중단하세요.

---

### ④ ECR 업로드 + Runtime 생성

배포 스크립트 `deploy.py`를 작성하여 ECR에 이미지를 push하고 Runtime을 생성합니다.

```bash
cd ~/agentcore/src
touch deploy.py
```

`deploy.py`를 열고 아래 코드를 작성하세요:

```python
"""
AgentCore Runtime 배포 스크립트 (Container 방식)
- IAM Role 생성, Docker 이미지를 ECR에 push하고 Runtime을 생성합니다
"""
import boto3
import json
import time
import subprocess

# === 설정 ===
REGION = "us-west-2"
AGENT_NAME = "healthcare_consultation_agent"
ECR_REPO_NAME = "healthcare-consultation-agent"
IMAGE_TAG = "latest"

# AWS 계정 ID 자동 획득
sts = boto3.client("sts")
ACCOUNT_ID = sts.get_caller_identity()["Account"]

ROLE_NAME = f"AmazonBedrockAgentCoreSDKRuntime-{REGION}"
ROLE_ARN = f"arn:aws:iam::{ACCOUNT_ID}:role/{ROLE_NAME}"
ECR_URI = f"{ACCOUNT_ID}.dkr.ecr.{REGION}.amazonaws.com/{ECR_REPO_NAME}:{IMAGE_TAG}"

# === 1단계: IAM Role 생성 (없는 경우) ===
iam = boto3.client("iam")

trust_policy = {
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Allow",
        "Principal": {"Service": "________"},  # TODO ①: AgentCore 서비스가 이 Role을 assume할 수 있도록 서비스 프린시펄을 채우세요
        "Action": "sts:AssumeRole"
    }]
}

try:
    iam.create_role(
        RoleName=ROLE_NAME,
        AssumeRolePolicyDocument=json.dumps(trust_policy),
        Description="AgentCore Runtime execution role"
    )
    print(f"✓ IAM Role 생성: {ROLE_NAME}")
    print("→ IAM Role 전파 대기 (10초)...")
    time.sleep(10)
except iam.exceptions.EntityAlreadyExistsException:
    print(f"✓ IAM Role 이미 존재: {ROLE_NAME}")

# AWS 관리형 정책 부여
iam.attach_role_policy(
    RoleName=ROLE_NAME,
    PolicyArn="arn:aws:iam::aws:policy/________"  # TODO ②: Runtime이 Bedrock 모델을 호출하고 로그를 쓸 수 있는 관리형 정책 이름을 채우세요
)
print(f"✓ 관리형 정책 부여 완료")

# === 2단계: ECR 리포지토리 생성 (없는 경우) ===
ecr = boto3.client("ecr", region_name=REGION)
try:
    ecr.describe_repositories(repositoryNames=[ECR_REPO_NAME])
    print(f"✓ ECR 리포지토리 존재: {ECR_REPO_NAME}")
except ecr.exceptions.RepositoryNotFoundException:
    ecr.________(repositoryName=ECR_REPO_NAME)  # TODO ③: ECR 리포지토리를 생성하는 API를 채우세요
    print(f"✓ ECR 리포지토리 생성: {ECR_REPO_NAME}")

# === 3단계: Docker 이미지 태그 + ECR 푸시 ===
print(f"→ ECR 로그인...")
login_cmd = f"aws ecr get-login-password --region {REGION} | docker login --username AWS --password-stdin {ACCOUNT_ID}.dkr.ecr.{REGION}.amazonaws.com"
subprocess.run(login_cmd, shell=True, check=True)

print(f"→ 이미지 태그: healthcare-agent:latest → {ECR_URI}")
subprocess.run(f"docker tag healthcare-agent:latest {ECR_URI}", shell=True, check=True)

print(f"→ ECR 푸시 중...")
subprocess.run(f"docker push {ECR_URI}", shell=True, check=True)
print(f"✓ ECR 푸시 완료")

# === 4단계: AgentCore Runtime 생성 ===
agentcore = boto3.client("________", region_name=REGION)  # TODO ④: AgentCore 컨트롤 플레인 서비스 이름을 채우세요

try:
    response = agentcore.________(  # TODO ⑤: Runtime을 생성하는 API 이름을 채우세요
        agentRuntimeName=AGENT_NAME,
        agentRuntimeArtifact={
            "________": {  # TODO ⑥: Container 배포 시 사용하는 설정 키를 채우세요 (codeConfiguration 아님)
                "containerUri": ECR_URI
            }
        },
        networkConfiguration={"networkMode": "________"},  # TODO ⑦: 퍼블릭 네트워크 모드 값을 채우세요
        roleArn=ROLE_ARN,
        lifecycleConfiguration={
            "idleRuntimeSessionTimeout": 900,
            "maxLifetime": 28800
        },
    )
    print(f"✓ Runtime 생성 완료!")
    print(f"  ARN: {response['agentRuntimeArn']}")
    print(f"  Status: {response['status']}")
except agentcore.exceptions.ConflictException:
    print("⚠️ 동일 이름의 Runtime이 이미 존재합니다. 삭제 후 재생성하세요.")
```

배포 실행:

```bash
cd ~/agentcore/src
uv run python deploy.py
```

> **참고**: Runtime 생성 후 상태가 `READY`가 될 때까지 3~5분 소요될 수 있습니다.

**상태 확인:**

```bash
uv run python -c "
import boto3, json
client = boto3.client('bedrock-agentcore-control', region_name='us-west-2')
response = client.list_agent_runtimes()
for rt in response.get('agentRuntimes', []):
    print(f\"Name: {rt['agentRuntimeName']} | Status: {rt['status']} | ID: {rt['agentRuntimeId']}\")
"

> **참고**: Runtime 생성 후 상태가 `ACTIVE`가 될 때까지 3~5분 소요될 수 있습니다.

---

### ⑤ 배포된 에이전트 호출 테스트

Runtime 상태가 `READY`가 되면 API로 호출합니다.

```bash
cd ~/agentcore/src
touch invoke_agent.py
```

`invoke_agent.py`를 열고 아래 코드를 작성하세요:

```python
"""배포된 AgentCore Runtime 에이전트 호출"""
import boto3
import json
import uuid

REGION = "us-west-2"
AGENT_NAME = "healthcare_consultation_agent"

# Runtime ARN 조회
control = boto3.client("bedrock-agentcore-control", region_name=REGION)
runtimes = control.list_agent_runtimes()
agent_arn = next(
    rt["agentRuntimeArn"] for rt in runtimes["agentRuntimes"]
    if rt["agentRuntimeName"] == AGENT_NAME
)

# 에이전트 호출
client = boto3.client("bedrock-agentcore", region_name=REGION)
payload = json.dumps({"prompt": "두통이 3일째 지속됩니다"}).encode()

response = client.invoke_agent_runtime(
    agentRuntimeArn=agent_arn,
    runtimeSessionId=str(uuid.uuid4()),
    payload=payload,
    qualifier="DEFAULT"
)

# 응답 파싱
content = []
for chunk in response.get("response", []):
    content.append(chunk.decode("utf-8"))
print("".join(content))
```

실행:

```bash
uv run python invoke_agent.py
```

---

## 검증

- ✅ `list_agent_runtimes`에서 상태가 `READY`로 표시됨
- ✅ `invoke_agent_runtime`으로 호출 시 한국어 응답이 반환됨
- ✅ 응답에 면책 조항이 포함됨
- ✅ 로컬 실행과 동일한 품질의 응답

---

## 트러블슈팅

| 오류 | 원인 | 해결 |
|------|------|------|
| `AccessDeniedException` | IAM Role 권한 누락 | Runtime Role에 `BedrockAgentCoreFullAccess` + `CloudWatchLogsFullAccess` 확인 |
| `ModelNotAccessibleException` | Bedrock 모델 액세스 미활성화 | Bedrock 콘솔에서 Claude Sonnet 활성화 확인 |
| `ResourceNotFoundException` | 에이전트 이름 오타 | `list_agent_runtimes` API로 정확한 이름 확인 |
| `ConflictException` | 동일 이름의 Runtime 이미 존재 | 기존 Runtime 삭제 후 재생성 |
| `docker build` 실패 | Docker 미설치 또는 권한 | `docker --version` 확인, 필요 시 `sudo` 사용 |
| ECR push 실패 | ECR 로그인 만료 | `aws ecr get-login-password` 재실행 |
| `Runtime initialization time exceeded` | 컨테이너 시작 시간 초과 | `main.py`에서 불필요한 import 제거, 이미지 크기 최소화 |
| Role not found | IAM Role 미생성 | `deploy.py` 재실행 또는 강사에게 문의 |

---

## 🏆 Challenge Task (시간 여유 시)

1. **AgentCore CLI 배포**: `npm install -g @aws/agentcore`로 CLI를 설치하고, `agentcore create` + `agentcore deploy`로 동일한 에이전트를 CLI 방식으로도 배포해 보세요
2. **다국어 지원**: 시스템 프롬프트에 "입력 언어를 감지하여 동일 언어로 응답" 규칙 추가

---

완료 후 **Phase 1 리뷰**에서 팀원과 산출물을 공유하세요.
그 다음 **030 Phase 2: 도구 통합 + 메모리 + Observability**로 이동합니다.

---

## 부록: 정답 코드

<details>
<summary>main.py 전체 정답 코드 보기 (클릭하여 펼치기)</summary>

```python
"""
AgentCore Runtime Entrypoint
- Phase 1-A에서 만든 에이전트를 Runtime에 배포
"""

from bedrock_agentcore.runtime import BedrockAgentCoreApp
# 같은 폴더(~/agentcore/src/)의 consultation_agent.py에서 에이전트 인스턴스를 가져옴
from consultation_agent import consultation_agent  # TODO: Phase 1-A에서 만든 에이전트 인스턴스를 import하세요

# AgentCore 앱 인스턴스 생성
app = BedrockAgentCoreApp()


@app.entrypoint  # TODO: Runtime 진입점 데코레이터를 채우세요
def healthcare_consultation(payload):
    """AgentCore Runtime 엔트리포인트
    
    Runtime이 API 요청을 받으면 이 함수가 호출됩니다.
    
    Args:
        payload: API 호출 시 전달되는 JSON 본문
            - prompt (str): 사용자 입력
            - patient_id (str, optional): 환자 ID
    
    Returns:
        str: 에이전트 응답 텍스트
    """
    user_input = payload.get("prompt", "")
    
    if not user_input:
        return "질문을 입력해 주세요."
    
    # Phase 1-A에서 만든 에이전트 호출
    response = consultation_agent(user_input)  # TODO: 에이전트에 전달할 입력을 채우세요
    return response.message['content'][0]['text']


# 로컬 실행 시 HTTP 서버로 동작 (배포 후에는 Runtime이 자동 처리)
if __name__ == "__main__":
    app.run(port=8888)
```

</details>

<details>
<summary>deploy.py 정답 코드 보기 (클릭하여 펼치기)</summary>

```python
"""
AgentCore Runtime 배포 스크립트 (Container 방식)
- IAM Role 생성, Docker 이미지를 ECR에 push하고 Runtime을 생성합니다
"""
import boto3
import json
import time
import subprocess

# === 설정 ===
REGION = "us-west-2"
AGENT_NAME = "healthcare_consultation_agent"
ECR_REPO_NAME = "healthcare-consultation-agent"
IMAGE_TAG = "latest"

# AWS 계정 ID 자동 획득
sts = boto3.client("sts")
ACCOUNT_ID = sts.get_caller_identity()["Account"]

ROLE_NAME = f"AmazonBedrockAgentCoreSDKRuntime-{REGION}"
ROLE_ARN = f"arn:aws:iam::{ACCOUNT_ID}:role/{ROLE_NAME}"
ECR_URI = f"{ACCOUNT_ID}.dkr.ecr.{REGION}.amazonaws.com/{ECR_REPO_NAME}:{IMAGE_TAG}"

# === 1단계: IAM Role 생성 (없는 경우) ===
iam = boto3.client("iam")

trust_policy = {
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Allow",
        "Principal": {"Service": "bedrock-agentcore.amazonaws.com"},
        "Action": "sts:AssumeRole"
    }]
}

try:
    iam.create_role(
        RoleName=ROLE_NAME,
        AssumeRolePolicyDocument=json.dumps(trust_policy),
        Description="AgentCore Runtime execution role"
    )
    print(f"✓ IAM Role 생성: {ROLE_NAME}")
    print("→ IAM Role 전파 대기 (10초)...")
    time.sleep(10)
except iam.exceptions.EntityAlreadyExistsException:
    print(f"✓ IAM Role 이미 존재: {ROLE_NAME}")

# AWS 관리형 정책 부여
iam.attach_role_policy(
    RoleName=ROLE_NAME,
    PolicyArn="arn:aws:iam::aws:policy/AdministratorAccess"
)
print(f"✓ 관리형 정책 부여 완료")

# === 2단계: ECR 리포지토리 생성 (없는 경우) ===
ecr = boto3.client("ecr", region_name=REGION)
try:
    ecr.describe_repositories(repositoryNames=[ECR_REPO_NAME])
    print(f"✓ ECR 리포지토리 존재: {ECR_REPO_NAME}")
except ecr.exceptions.RepositoryNotFoundException:
    ecr.create_repository(repositoryName=ECR_REPO_NAME)
    print(f"✓ ECR 리포지토리 생성: {ECR_REPO_NAME}")

# === 3단계: Docker 이미지 태그 + ECR 푸시 ===
print(f"→ ECR 로그인...")
login_cmd = f"aws ecr get-login-password --region {REGION} | docker login --username AWS --password-stdin {ACCOUNT_ID}.dkr.ecr.{REGION}.amazonaws.com"
subprocess.run(login_cmd, shell=True, check=True)

print(f"→ 이미지 태그: healthcare-agent:latest → {ECR_URI}")
subprocess.run(f"docker tag healthcare-agent:latest {ECR_URI}", shell=True, check=True)

print(f"→ ECR 푸시 중...")
subprocess.run(f"docker push {ECR_URI}", shell=True, check=True)
print(f"✓ ECR 푸시 완료")

# === 4단계: AgentCore Runtime 생성 ===
agentcore = boto3.client("bedrock-agentcore-control", region_name=REGION)

try:
    response = agentcore.create_agent_runtime(
        agentRuntimeName=AGENT_NAME,
        agentRuntimeArtifact={
            "containerConfiguration": {
                "containerUri": ECR_URI
            }
        },
        networkConfiguration={"networkMode": "PUBLIC"},
        roleArn=ROLE_ARN,
        lifecycleConfiguration={
            "idleRuntimeSessionTimeout": 900,
            "maxLifetime": 28800
        },
    )
    print(f"✓ Runtime 생성 완료!")
    print(f"  ARN: {response['agentRuntimeArn']}")
    print(f"  Status: {response['status']}")
except agentcore.exceptions.ConflictException:
    print("⚠️ 동일 이름의 Runtime이 이미 존재합니다. 삭제 후 재생성하세요.")
```

</details>
