---
title: "Day 3 사전 설정: Claude Code on Bedrock 인증 구성"
weight: 60
time: "20분"
---

# Claude Code on Bedrock 인증 구성 (20분)

## 개요

Day 3에서는 Claude Code on Bedrock를 사용하여 풀스택 웹 앱을 구축합니다.
이 문서에서는 Claude Code가 Amazon Bedrock에 인증하여 Claude 모델을 호출할 수 있도록 환경을 구성합니다.

> [!NOTE]
> **리전 변경 안내 (Oregon 리전에서 실습하는 경우)**
>
> 이 워크샵은 **us-east-1 (N. Virginia)** 기준으로 작성되어 있습니다. **us-west-2 (Oregon)** 리전에서 실습하려면 아래 항목을 모두 `us-west-2`로 변경하세요:
>
> - 섹션 2.1: `export AWS_REGION="us-west-2"`
> - 섹션 3.1: settings.json의 `"AWS_REGION": "us-west-2"`
> - 섹션 6.8: terraform.tfvars의 `aws_region = "us-west-2"`, `availability_zones = ["us-west-2a", "us-west-2b"]`
> - 섹션 7.2: ECR 명령어의 `--region us-west-2` 및 ECR URI의 `.ecr.us-west-2.amazonaws.com`
> - 섹션 7.7: app terraform.tfvars의 `aws_region = "us-west-2"`
>
> 모델 ID(`global.anthropic.claude-*`)는 크로스 리전 추론을 사용하므로 변경 불필요합니다.

---

## 목차

1. [Claude Code 설치](#1-claude-code-설치)
2. [AWS 인증 구성](#2-aws-인증-구성)
3. [Bedrock LLM 사용 설정](#3-bedrock-llm-사용-설정)
4. [SKILL.md 생성 — AWS + Terraform + Container](#4-skillmd-생성--aws--terraform--container)
5. [MCP 서버 연결 구성](#5-mcp-서버-연결-구성)
6. [인프라 Terraform 템플릿 생성 및 배포](#6-인프라-terraform-템플릿-생성-및-배포) — VPC, ECS Cluster
7. [Container 기반 애플리케이션 배포를 위한 Terraform 템플릿 생성 및 배포](#7-container-기반-애플리케이션-배포를-위한-terraform-템플릿-생성-및-배포) — Cats & Dogs 멀티 서비스
8. [비용 최적화 및 아키텍처 개선](#8-비용-최적화-및-아키텍처-개선)

---

## 1. Claude Code 설치

### 1.1 Claude Code란?

Claude Code는 Anthropic이 개발한 **에이전틱(agentic) 코딩 도구**입니다. 일반적인 코드 자동완성 도구나 채팅 사이드바와 달리, Claude Code는 터미널에서 직접 동작하며 파일 시스템, 셸 명령, Git에 접근하여 복잡한 멀티 스텝 작업을 자율적으로 수행합니다.

> 공식 문서: https://docs.anthropic.com/en/docs/claude-code/overview

**기존 코딩 도우미와의 차이점:**

| 구분 | 코드 자동완성 (Copilot 등) | Claude Code |
|------|---------------------------|-------------|
| 동작 방식 | 에디터 내 인라인 제안 | 터미널에서 에이전트 루프 실행 |
| 작업 범위 | 현재 파일, 현재 줄 | 전체 코드베이스, 다중 파일 |
| 실행 능력 | 코드 제안만 | 파일 편집 + 명령 실행 + 결과 검증 |
| 상호작용 | 탭으로 수락/거부 | 자연어로 대화하며 작업 위임 |

**핵심 기능:**

- **코드베이스 이해** — 프로젝트 구조를 파악하고, 파일 간 의존관계를 추적
- **파일 읽기/편집/생성** — 다중 파일을 동시에 수정하고 새 파일 생성
- **셸 명령 실행** — 빌드, 테스트, 린트 등 터미널 명령을 직접 실행하고 결과 확인
- **Git 워크플로우** — 커밋, 브랜치 생성, PR 작성 등을 자연어로 처리
- **MCP(Model Context Protocol)** — 외부 도구/API와 연결하여 기능 확장 (Terraform Registry, AWS 문서 등)
- **자율적 반복** — 오류 발생 시 스스로 원인을 분석하고 수정을 반복

**IaC 작업에서의 장점:**

이 워크샵에서 Claude Code를 사용하는 이유는 다음과 같습니다.

1. **Terraform 코드 생성** — 모범 사례에 맞는 모듈 구조를 자연어 설명만으로 생성
2. **즉시 검증** — 생성한 코드를 `terraform validate`, `terraform plan`으로 바로 검증
3. **문서 참조** — MCP를 통해 Terraform Registry의 최신 프로바이더 문서를 실시간 조회
4. **반복 수정** — plan 오류를 분석하고 코드를 자동으로 수정하는 피드백 루프
5. **인프라 분석** — 배포된 리소스 상태를 확인하고 비용/보안 개선점을 제안

### 1.2 사전 요구 사항

| 항목 | 요구 사항 |
|------|-----------|
| Node.js | 18 이상 |
| 운영체제 | macOS, Linux, 또는 Windows(WSL2) |
| 터미널 | bash 또는 zsh |

> **Windows 사용자:** WSL2가 설치되어 있어야 합니다. 모든 명령은 WSL2 터미널 내에서 실행합니다.  
> 설치 가이드: https://learn.microsoft.com/ko-kr/windows/wsl/install

### 1.3 설치

```bash
# npm을 통한 전역 설치
npm install -g @anthropic-ai/claude-code
```

### 1.4 설치 확인

```bash
claude --version
```

정상적으로 버전 정보가 출력되면 설치가 완료된 것입니다.

### 1.5 기본 사용법

```bash
# 대화형 모드 시작
claude
```

> **팁:** Claude Code는 대화형 모드에서 `/help` 명령으로 사용 가능한 슬래시 명령어 목록을 확인할 수 있습니다.

---

## 2. AWS 인증 구성

이 워크샵에서는 사전에 제공된 AWS 자격 증명을 사용합니다.

### 2.1 제공된 자격 증명 설정

워크샵에서 제공하는 AWS 자격 증명을 환경 변수로 설정합니다.

```bash
export AWS_ACCESS_KEY_ID="<워크샵에서 제공된 값>"
export AWS_SECRET_ACCESS_KEY="<워크샵에서 제공된 값>"
export AWS_SESSION_TOKEN="<워크샵에서 제공된 값>"
export AWS_REGION="us-east-1"
```

### 2.2 인증 확인

```bash
aws sts get-caller-identity
```

정상 출력 예시:

```json
{
    "UserId": "AROAEXAMPLEID:workshop-user",
    "Account": "123456789012",
    "Arn": "arn:aws:sts::123456789012:assumed-role/workshop-role/workshop-user"
}
```

### 2.3 Terraform용 AWS 인증

Terraform은 환경 변수에 설정된 AWS 자격 증명을 자동으로 인식합니다. 별도의 추가 설정 없이 `terraform plan` 및 `terraform apply` 명령이 동작합니다.

---

## 3. Bedrock LLM 사용 설정

이 워크샵에서는 워크샵 환경에서 제공하는 Amazon Bedrock 기반의 Claude 모델을 사용합니다.

> [!IMPORTANT]
> **Bedrock Claude 모델 접근을 위한 사전 작업 (자체 AWS 계정을 사용하는 경우에만)**
>
> AWS Workshop Studio를 통해 제공된 계정을 사용하는 경우, 모델 접근 권한이 이미 설정되어 있으므로 이 단계를 건너뛰세요.
>
> 자체 AWS 계정으로 실습하는 경우, Bedrock 관리 콘솔에서 **Submit use case details for Anthropic**을 진행해야 합니다.
>
> 1. AWS 관리 콘솔에 접속합니다.
> 2. **Bedrock** 서비스로 가서 **Tune → Marketplace model deployments**를 클릭합니다.
> 3. 오른쪽에 **View marketplace models**를 클릭합니다.
> 4. 위에 *"Anthropic requires first-time customers to submit use case details"*로 시작하는 안내가 있습니다. **Submit use case detail**을 클릭합니다.
> 5. 아래와 같이 입력합니다:
>    - **Company name**: `AWS`
>    - **Company website URL**: `https://aws.amazon.com`
>    - **What industry do you operate in?**: `Software as a Service`
>    - **External users** 체크
>    - **Describe**: `AWS training for customer` 입력
> 6. 최종 **Submit use case details**를 클릭하여 제출합니다.

### 3.1 환경 변수 설정

Claude Code가 Amazon Bedrock을 통해 모델을 호출하도록 설정합니다. Claude Code의 설정 파일(`~/.claude/settings.json`)을 아래 내용으로 교체합니다.

```json
{
  "model": "opusplan",
  "env": {
    "CLAUDE_CODE_USE_BEDROCK": "1",
    "AWS_REGION": "us-east-1",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "global.anthropic.claude-sonnet-4-6",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "global.anthropic.claude-opus-4-6-v1",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "global.anthropic.claude-haiku-4-5-20251001-v1:0"
  }
}
```

**설정 항목 설명:**

| 항목 | 설명 |
|------|------|
| `model` | 기본 실행 모드. `opusplan`은 계획은 Opus, 실행은 Sonnet으로 분리하여 비용 효율적으로 동작 |
| `CLAUDE_CODE_USE_BEDROCK` | `1`로 설정하면 Anthropic API 대신 Bedrock을 통해 모델 호출 |
| `AWS_REGION` | Bedrock 모델을 호출할 AWS 리전 |
| `ANTHROPIC_DEFAULT_SONNET_MODEL` | 코드 작성/실행에 사용할 Sonnet 모델 ID |
| `ANTHROPIC_DEFAULT_OPUS_MODEL` | 고수준 계획 수립에 사용할 Opus 모델 ID |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL` | 간단한 작업에 사용할 경량 Haiku 모델 ID |

> **참고:** `global.` 접두사가 붙은 모델 ID는 Amazon Bedrock의 크로스 리전 추론을 사용합니다. 특정 리전에 용량이 부족할 경우 다른 리전으로 자동 라우팅되어 가용성이 높아집니다. 단일 리전에서만 호출하려면 `us.anthropic.claude-sonnet-4-6`처럼 리전 접두사를 사용합니다.

### 3.2 Claude Code 최초 실행 — 로그인 우회

Claude Code를 처음 실행하면 기본적으로 Anthropic 계정 로그인 위저드가 나타납니다. 하지만 **위 설정 파일에 `CLAUDE_CODE_USE_BEDROCK`이 설정되어 있으면 로그인 위저드가 건너뛰어지고**, AWS 자격 증명을 통해 바로 Bedrock에 연결됩니다.

```bash
# settings.json 설정 완료 후 claude 실행
claude
```

> **중요:** 설정 파일이 적용되지 않거나 로그인 화면이 뜨는 경우, 터미널에서 `export CLAUDE_CODE_USE_BEDROCK=1`을 직접 실행한 후 `claude`를 다시 시작하면 됩니다.

정상적으로 연결되면 프롬프트가 나타나고 바로 대화를 시작할 수 있습니다.

### 3.3 연결 확인

Claude Code 대화형 모드에서 `/status` 명령어를 실행하여 Bedrock 연결 상태를 확인합니다.

```
> /status
```

다음 항목이 올바르게 표시되는지 확인합니다:

| 확인 항목 | 기대 값 |
|-----------|---------|
| API Provider | `Amazon Bedrock` |
| AWS Region | `us-east-1` (설정과 일치) |
| Model | `opusplan` 또는 설정한 모델명 |

출력 예시:

```
API Provider: Amazon Bedrock
Region: us-east-1
Model: opusplan
...
```

> **문제 발생 시:** API Provider가 `Anthropic`으로 표시되거나 리전이 다르면, `Ctrl+C`로 종료한 후 `~/.claude/settings.json` 설정과 AWS 자격 증명 환경 변수를 재확인하세요.

연결이 확인되면 간단한 메시지로 동작을 테스트합니다:

```
> 안녕하세요, Bedrock을 통해 정상적으로 연결되었는지 확인합니다.
```

정상적으로 응답이 오면 Bedrock 연동이 완료된 것입니다.

### 3.4 모델 동작 방식

```
[사용자 터미널] → [Claude Code CLI] → [Amazon Bedrock API] → [Claude 모델]
```

`opusplan` 모드에서는 작업이 두 단계로 나뉩니다:

1. **계획 수립 (Opus)** — 복잡한 작업의 전체 전략을 Opus 모델이 설계
2. **실행 (Sonnet)** — 계획에 따라 Sonnet 모델이 실제 코드 작성/편집 수행

이 방식은 Opus의 고수준 추론 능력과 Sonnet의 빠른 실행 속도를 결합하여, 품질과 비용 효율 사이의 균형을 잡습니다. 간단한 작업(파일 읽기, 짧은 질문 등)에는 경량 Haiku 모델이 자동으로 선택됩니다.

Claude Code는 `CLAUDE_CODE_USE_BEDROCK=1` 설정에 의해 Anthropic API 대신 AWS 자격 증명으로 Amazon Bedrock에 요청을 보내며, 모든 추론이 AWS 보안 경계 내에서 처리됩니다.

---

설정이 완료되면 [Phase 5: 풀스택 의료 AI 웹 애플리케이션](./061_fullstack_app.md)으로 이동하세요.
