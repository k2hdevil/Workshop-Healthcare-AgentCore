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
| **Direct Code Deploy (CodeZip)** | Python/Node.js 코드와 의존성을 zip으로 패키징하여 업로드. AWS가 런타임 환경(Python, 보안 패치)을 관리. | 빠른 반복 개발, 커스텀 런타임 불필요 시. **본 워크샵에서 사용** | ~5초 |
| **Container (ECR)** | Docker 이미지를 ECR에 push하고 Runtime이 pull. 완전한 환경 제어. | GPU 필요, 커스텀 시스템 패키지, 대형 의존성 | ~10-15초 |

**배포 도구 (배포 방식을 실행하는 인터페이스):**

| 배포 도구 | 설명 | 내부적으로 사용하는 배포 방식 |
|-----------|------|--------------------------|
| **AgentCore CLI** | `agentcore deploy` 한 줄로 패키징+업로드+Runtime 생성을 자동화 | Direct Code Deploy |
| **AWS SDK/API** | `create_agent_runtime` API를 직접 호출. 세밀한 제어 가능. **본 워크샵에서 사용** | Direct Code 또는 Container |

#### AWS SDK 배포 흐름 (본 워크샵)

```
개발자 PC                        AWS
─────────                        ───
1. uv pip install --target
   └→ 의존성 다운로드
2. zip 패키징                   → 3. S3 업로드
                                 → 4. create_agent_runtime API 호출
                                 → 5. Runtime 생성 + DEFAULT 엔드포인트
                                 → 6. 상태: ACTIVE

7. invoke_agent_runtime
   └→ API 호출                  → 8. microVM 할당
                                 → 9. @app.entrypoint 함수 실행
                                 → 10. 응답 반환
```

---

## 실습: AgentCore Runtime 배포 순서

AWS SDK(boto3)를 사용한 Direct Code Deploy 방식으로 배포합니다. 아래 5단계로 진행됩니다:

```
① 엔트리포인트 작성 → ② 배포 패키지 생성 → ③ 로컬 테스트 → ④ S3 업로드 + Runtime 생성 → ⑤ 호출 테스트
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

### ② 배포 패키지 생성 (zip)

Runtime에 업로드할 zip 파일을 생성합니다. **의존성을 미리 포함**시켜야 콜드스타트가 빨라집니다.

```bash
cd ~/agentcore/src

# 1. 의존성을 deployment_package/ 디렉토리에 설치 (ARM64 호환)
uv pip install \
  --python-platform aarch64-manylinux2014 \
  --python-version 3.12 \
  --target=deployment_package \
  --only-binary=:all: \
  strands-agents bedrock-agentcore boto3

# 2. 의존성 폴더를 zip으로 압축
cd deployment_package
zip -r ../deployment_package.zip .
cd ..

# 3. 에이전트 코드 파일을 zip 루트에 추가
zip deployment_package.zip main.py consultation_agent.py
```

> **참고**: `--only-binary=:all:`은 사전 빌드된 wheel만 설치합니다. 일부 패키지에서 오류가 나면 `--only-binary=:all:`을 제거하고 다시 시도하세요.

생성된 zip 파일 크기 확인:

```bash
ls -lh deployment_package.zip
# 250MB 이하여야 합니다
```

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

### ④ S3 업로드 + Runtime 생성

배포 스크립트 `deploy.py`를 작성하여 S3 업로드와 Runtime 생성을 수행합니다.

```bash
cd ~/agentcore/src
touch deploy.py
```

`deploy.py`를 열고 아래 코드를 작성하세요:

```python
"""
AgentCore Runtime 배포 스크립트
- IAM Role 생성, zip 파일을 S3에 업로드하고 Runtime을 생성합니다
"""
import boto3
import json
import time

# === 설정 ===
REGION = "us-west-2"
AGENT_NAME = "healthcare_consultation_agent"

# AWS 계정 ID 자동 획득
sts = boto3.client("sts")
ACCOUNT_ID = sts.get_caller_identity()["Account"]

# S3 버킷 이름 (AgentCore 규칙: bedrock-agentcore-code-{계정ID}-{리전})
BUCKET_NAME = f"bedrock-agentcore-code-{ACCOUNT_ID}-{REGION}"
S3_KEY = f"{AGENT_NAME}/deployment_package.zip"
ROLE_NAME = f"AmazonBedrockAgentCoreSDKRuntime-{REGION}"
ROLE_ARN = f"arn:aws:iam::{ACCOUNT_ID}:role/{ROLE_NAME}"

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
    # Role 전파 대기
    print("→ IAM Role 전파 대기 (10초)...")
    time.sleep(10)
except iam.exceptions.EntityAlreadyExistsException:
    print(f"✓ IAM Role 이미 존재: {ROLE_NAME}")

# AWS 관리형 정책 부여 (Bedrock AgentCore + CloudWatch Logs 권한)
iam.attach_role_policy(
    RoleName=ROLE_NAME,
    PolicyArn="arn:aws:iam::aws:policy/AdministratorAccess"
)
print(f"✓ 관리형 정책 부여 완료")

# === 2단계: S3 버킷 생성 (없는 경우) ===
s3 = boto3.client("s3", region_name=REGION)
try:
    s3.head_bucket(Bucket=BUCKET_NAME)
    print(f"✓ S3 버킷 존재: {BUCKET_NAME}")
except Exception:
    print(f"→ S3 버킷 생성: {BUCKET_NAME}")
    s3.create_bucket(
        Bucket=BUCKET_NAME,
        CreateBucketConfiguration={"LocationConstraint": REGION}
    )
    print(f"✓ S3 버킷 생성 완료")

# === 3단계: zip 파일 업로드 ===
print(f"→ 업로드 중: deployment_package.zip → s3://{BUCKET_NAME}/{S3_KEY}")
s3.upload_file(
    "deployment_package.zip",
    BUCKET_NAME,
    S3_KEY,
    ExtraArgs={"ExpectedBucketOwner": ACCOUNT_ID}
)
print(f"✓ 업로드 완료")

# === 4단계: AgentCore Runtime 생성 ===
agentcore = boto3.client("bedrock-agentcore-control", region_name=REGION)

try:
    response = agentcore.create_agent_runtime(
        agentRuntimeName=AGENT_NAME,
        agentRuntimeArtifact={
            "codeConfiguration": {
                "code": {
                    "s3": {
                        "bucket": BUCKET_NAME,
                        "prefix": S3_KEY
                    }
                },
                "runtime": "PYTHON_3_12",
                "entryPoint": ["main.py"]
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
    response = agentcore.update_agent_runtime(
        agentRuntimeName=AGENT_NAME,
        agentRuntimeArtifact={
            "codeConfiguration": {
                "code": {
                    "s3": {
                        "bucket": BUCKET_NAME,
                        "prefix": S3_KEY
                    }
                },
                "runtime": "PYTHON_3_12",
                "entryPoint": ["main.py"]
            }
        },
    )
    print(f"✓ Runtime 업데이트 완료!")
    print(f"  ARN: {response['agentRuntimeArn']}")
    print(f"  Status: {response['status']}")
```

배포 실행:

```bash
cd ~/agentcore/src
uv run python deploy.py
```

> **참고**: Runtime 생성 후 상태가 `ACTIVE`가 될 때까지 3~5분 소요될 수 있습니다.

**상태 확인:**

```bash
uv run python -c "
import boto3, json
client = boto3.client('bedrock-agentcore-control', region_name='us-west-2')
response = client.list_agent_runtimes()
for rt in response.get('agentRuntimes', []):
    print(f\"Name: {rt['agentRuntimeName']} | Status: {rt['status']} | ID: {rt['agentRuntimeId']}\")
"
```

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
| `AccessDeniedException` | IAM Role 권한 누락 | Runtime Role에 `bedrock:InvokeModel` 권한 확인 |
| `ModelNotAccessibleException` | Bedrock 모델 액세스 미활성화 | Bedrock 콘솔에서 Claude Sonnet 활성화 확인 |
| `ResourceNotFoundException` | 에이전트 이름 오타 | `list_agent_runtimes` API로 정확한 이름 확인 |
| `ConflictException` | 동일 이름의 Runtime 이미 존재 | `deploy.py`가 자동으로 update로 전환됨 |
| `ValidationException` | zip 파일 구조 오류 | `main.py`가 zip 루트에 있는지 확인: `unzip -l deployment_package.zip \| grep main.py` |
| zip 용량 250MB 초과 | 의존성이 너무 큼 | 불필요한 패키지 제거 또는 Container 방식 사용 |
| `Runtime initialization time exceeded` | 의존성 로딩 시간 초과 | zip에 의존성이 포함되었는지 확인 |
| `NoSuchBucket` | S3 버킷이 없음 | `deploy.py`가 자동 생성하므로 재실행 |
| Role not found | IAM Role 미생성 | 강사에게 문의 — `AmazonBedrockAgentCoreSDKRuntime-us-west-2` Role 필요 |

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
