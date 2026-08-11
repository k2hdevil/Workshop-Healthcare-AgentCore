---
title: "Day 3 사전 설정: Claude Code on Bedrock 인증 구성"
weight: 60
time: "20분"
---

# Claude Code on Bedrock 인증 구성 (20분)

## 개요

Day 3에서는 Claude Code on Bedrock를 사용하여 풀스택 웹 앱을 구축합니다.
이 문서에서는 Claude Code가 Amazon Bedrock에 인증하여 Claude 모델을 호출할 수 있도록 환경을 구성합니다.

> **참고**: 공식 문서 — [Claude Code on Amazon Bedrock](https://docs.anthropic.com/en/docs/claude-code/amazon-bedrock)

---

## 사전 요구사항

- AWS 계정에 Amazon Bedrock 접근 활성화
- Claude Sonnet 4.5 모델 접근 승인 완료
- Node.js 20+ 설치 (Claude Code는 npm 패키지)
- AWS CLI 설치 및 구성 완료

---

## Step 1: Claude Code 설치

```bash
# npm으로 Claude Code 설치
npm install -g @anthropic-ai/claude-code

# 설치 확인
claude --version
```

---

## Step 2: Bedrock 모델 접근 승인

처음 사용하는 경우, Bedrock 콘솔에서 모델 접근을 신청해야 합니다:

1. [Amazon Bedrock 콘솔](https://console.aws.amazon.com/bedrock/) 접속
2. Model catalog에서 Anthropic 모델 선택
3. Use case form 작성 → 즉시 승인됨

---

## Step 3: AWS 자격 증명 구성

Workshop Studio 환경에서는 이미 EC2 인스턴스에 IAM Role이 연결되어 있으므로,
별도 자격 증명 설정 없이 바로 사용할 수 있습니다.

### 방법 A: Workshop Studio 환경 (EC2 Instance Role) — 권장

EC2에 연결된 IAM Role이 자동으로 자격 증명을 제공합니다.
추가 설정 불필요.

```bash
# 현재 자격 증명 확인
aws sts get-caller-identity
```

### 방법 B: 환경 변수 설정 (로컬 환경)

로컬에서 실행하는 경우 환경 변수로 설정합니다:

```bash
# AWS 자격 증명
export AWS_ACCESS_KEY_ID="your-access-key"
export AWS_SECRET_ACCESS_KEY="your-secret-key"
export AWS_SESSION_TOKEN="your-session-token"  # 임시 자격 증명일 경우
export AWS_REGION="us-west-2"
```

### 방법 C: AWS CLI 프로파일

```bash
# AWS CLI로 프로파일 설정
aws configure --profile workshop
export AWS_PROFILE=workshop
```

---

## Step 4: Claude Code Bedrock 연결 구성

### 방법 1: 대화형 마법사 (권장)

```bash
# Claude Code 실행 후 Bedrock 설정 마법사 시작
claude

# Claude Code 내에서:
/setup-bedrock
```

마법사가 자격 증명, 리전, 모델 핀을 순서대로 안내합니다.

### 방법 2: 환경 변수 직접 설정

```bash
# Bedrock 사용 활성화
export CLAUDE_CODE_USE_BEDROCK=1
export AWS_REGION=us-west-2

# (선택) 모델 고정 — Workshop에서는 Sonnet 4.5 사용
export ANTHROPIC_DEFAULT_SONNET_MODEL="us.anthropic.claude-sonnet-4-5-20250929-v1:0"
```

---

## Step 5: 연결 확인

```bash
# Claude Code 시작
claude

# 내부에서 상태 확인
/status
```

**정상 출력 예시:**

```
Provider: Amazon Bedrock
Region: us-west-2
Model: us.anthropic.claude-sonnet-4-5-20250929-v1:0
```

---

## Step 6: IAM 권한 확인

Claude Code가 Bedrock을 호출하려면 아래 IAM 권한이 필요합니다:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "bedrock:InvokeModel",
        "bedrock:InvokeModelWithResponseStream",
        "bedrock:ListInferenceProfiles",
        "bedrock:GetInferenceProfile"
      ],
      "Resource": "*"
    }
  ]
}
```

> Workshop Studio 환경에서는 이미 이 권한이 포함되어 있습니다.

---

## 트러블슈팅

| 문제 | 원인 | 해결 |
|------|------|------|
| `Unable to locate credentials` | 자격 증명 미설정 | `aws sts get-caller-identity`로 확인 후 Step 3 수행 |
| `on-demand throughput isn't supported` | inference profile 필요 | 모델 ID에 `us.` prefix 추가 (예: `us.anthropic.claude-sonnet-4-5-20250929-v1:0`) |
| `/status`에서 Provider가 Anthropic API | Bedrock 활성화 안 됨 | `export CLAUDE_CODE_USE_BEDROCK=1` 확인 |
| `AccessDeniedException` | IAM 권한 부족 | Step 6의 IAM 정책 확인 |
| `Session token not found or invalid` | SSO 토큰 만료 | `aws sso login` 재실행 |

---

## 빠른 시작 (전체 명령 요약)

Workshop Studio 환경에서 한 번에 실행:

```bash
# 1. Claude Code 설치
npm install -g @anthropic-ai/claude-code

# 2. 환경 변수 설정
export CLAUDE_CODE_USE_BEDROCK=1
export AWS_REGION=us-west-2

# 3. Claude Code 실행
claude

# 4. (Claude Code 내에서) 상태 확인
/status
```

---

설정이 완료되면 [Phase 5: 풀스택 의료 AI 웹 애플리케이션](./061_fullstack_app.md)으로 이동하세요.
