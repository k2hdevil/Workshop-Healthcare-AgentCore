
# Phase 2-A: 혈액 검사 도구 통합 (75분)

## 학습 목표

환자 김민수의 건강검진 결과를 조회하고 한국 건강검진 기준으로 분석하는 도구를 구현합니다.

---

## 이론: 의료 도구 설계 원칙 (브리핑 20분에서 전달)

### 한국 규제 기반 생성형 AI 의료 도구 설계 원칙

대한민국은 생성형 AI 의료 도구에 대해 세계적으로 선도적인 규제 프레임워크를 수립했습니다. 아래 원칙들은 실습에서 구현하는 도구 설계에 직접 반영됩니다.

#### 인공지능 기본법의 6대 설계 원칙

**「인공지능 발전과 신뢰 기반 조성 등에 관한 기본법」** (2025.01.21 제정, 2026.01.22 시행)은 대한민국 최초의 포괄적 AI 법률로, 의료 AI를 **"고영향 AI (High-impact AI)"**로 분류합니다. 고영향 AI란 인간의 생명, 안전, 기본권에 중대한 영향을 미칠 수 있는 AI 시스템을 의미하며, 의료 서비스 제공·이용 체계에 사용되는 AI와 의료기기 개발·사용에 활용되는 AI가 이에 해당합니다.

고영향 AI를 제공하는 사업자는 위험 관리 계획 수립, 인간 감독 보장, AI 출력에 대한 설명 제공, 이용자 보호 조치 확립, 준수를 입증하는 문서 유지(5년 보존)를 이행해야 합니다. 위반 시 최대 3,000만원의 과태료가 부과됩니다.

| # | 원칙 | 설명 | 본 실습 적용 |
|---|------|------|-------------|
| 1 | **안전성 (Safety)** | 전 생애주기에 걸친 위험 식별·완화, 위험 관리 계획 수립 의무 | 입력 검증, 접근 제어, 에러 핸들링 구현 |
| 2 | **투명성 (Transparency)** | AI 사용 사실을 환자에게 고지하고, 생성형 AI 출력물은 AI 생성임을 사전 통보 의무 | 에이전트 응답에 "AI 분석 결과" 표시, 워터마크 개념 적용 |
| 3 | **설명 가능성 (Explainability)** | AI 의사결정의 핵심 기준과 원리를 기술적으로 실현 가능한 범위 내에서 설명 의무 | 판정 기준(A/B/C/D)과 근거 수치를 함께 출력 |
| 4 | **인간 중심 감독 (Human Oversight)** | 의미 있는 인간 감독 보장, 최종 임상 판단은 인간 전문가에게 유보 | "반드시 의료 전문가와 상담하세요" 고지 포함 |
| 5 | **정확성·품질 관리 (Accuracy & Quality)** | 임상 성능 평가 체계와 지속적 모니터링, 안전 문제 시 AI 사용 중단·시정 절차 | 한국 건강검진 기준 기반 자동 등급 판정 구현 |
| 6 | **데이터 편향·윤리 관리 (Bias & Ethics)** | 데이터 편향 방지, 접근 제어, 환자 기본권 보호, 문서 5년 보존 | patient-001만 접근 허용, 권한 없는 데이터 차단 |

> **간주 준수 (Deemed Compliance) 조항:** 디지털의료제품법에 따라 이미 품질관리 심사를 통과한 AI 의료기기는 인공지능 기본법의 위험 관리, 이용자 보호, 감독 관리자 지정 요건을 충족한 것으로 간주됩니다. 즉, 기존 의료기기 인허가와 중복 규제를 피할 수 있습니다.

> 출처: [South Korea's AI Framework Act: A Guide for Healthcare Professionals (Korean Journal of Radiology, 2026)](https://pmc.ncbi.nlm.nih.gov/articles/PMC13056445/)

#### 식약처 가이드라인 핵심 설계 요구사항 (7대 규제 요건)

식약처(MFDS)가 2025년 1월 24일 발표한 「생성형 AI 의료기기 허가심사 가이드라인」은 LLM/LMM 기반 의료기기에 대해 다음 7가지 핵심 설계 요구사항을 명시합니다:

| # | 요구사항 | 상세 내용 |
|---|----------|-----------|
| 1 | **사용 목적 명시 (Intended Use Specification)** | 의약품 설명서처럼 의도된 사용 목적, 적응증, 대상 환자군을 명확히 기재하고 off-label(허가 외) 사용에 대한 경고를 이용자에게 제공해야 함 |
| 2 | **전문 의료인 사용 제한 (Qualified User Restriction)** | LLM/LMM 기반 의료기기는 해당 분야 자격을 갖춘 임상의만 사용하도록 제한하여, 오정보 및 자동화 편향(automation bias) 위험을 간접적으로 관리 |
| 3 | **AI 생성 콘텐츠 디지털 워터마크 (Digital Watermarking)** | AI가 생성한 콘텐츠에 디지털 워터마크를 의무 적용하여 AI 생성물임을 명확히 구분하고, 의료 전문가의 검토·확인을 거치도록 인간 감독 체계 구현 |
| 4 | **임상 성능 평가 (Clinical Performance Evaluation)** | 해당 분야 복수의 전문 임상의가 직접 임상적 맥락에서 평가해야 하며, BLEU·ROUGE·BERTScore 등 자동화 지표는 보조적으로만 사용. 주관적 평가 시 명확한 품질 등급 기준, 복수 평가자, 사전 방법론 명세 필요 |
| 5 | **확률적 비일관성 대응 (Stochasticity Management)** | 생성 결과의 비결정성(stochasticity)으로 인한 출력 비일관성 문제를 인지하고, 고위험 의료 응용에서는 결정론적 운영을 위한 모델 조정이 필요할 수 있음 |
| 6 | **사전 변경 관리 계획 (Predetermined Change Control Plan)** | 모델 업데이트에 따른 성능·행동 변화에 대비하여 사전 변경 관리 원칙을 수립하고, 이용 설명서에 성능 드리프트 가능성 경고 및 주기적 성능 평가 권고 포함 |
| 7 | **학습 데이터·모델 투명성 (Training Data & Model Transparency)** | 학습에 사용된 데이터의 특성, 모델 구조, 파인튜닝 방법 등을 문서화하여 제출해야 하며, 데이터 편향·윤리적 문제에 대한 식별 및 완화 조치 기술 |

> **주요 특징:** 식약처 가이드라인은 LLM/LMM이 기존 AI 의료기기 대비 가지는 고유한 위험(다목적성, 프롬프트 민감도, 확률적 출력, 환각/오정보)을 명시적으로 인정하면서도, 현재 대부분의 조치가 사용자 교육과 사용 제한을 통한 간접 관리에 의존한다는 한계를 가집니다. 향후 AI 의료기기에 추론 설명 기능과 신뢰도 지표 제공을 의무화하는 방향으로 발전할 것으로 전망됩니다.

#### 식약처 임상 출력 등급 체계 (LLM/LMM 의료기기)

AI가 생성한 의료 텍스트(예: 진단 보고서 초안)의 품질을 다음 5등급으로 판정합니다:

| 등급 | 정의 | 판정 |
|------|------|------|
| A | 오류 없음 | ✅ 수용 |
| B | 비임상적 오류만 존재 | ✅ 수용 |
| C | 임상 항목 오류 존재, 임상적 중요성 낮음 | ⚠️ 맥락에 따라 판단 |
| D | 임상 항목 오류 존재, 임상적 중요성 높음 | ❌ 불수용 |
| E | 환자에 심각한 해를 끼칠 수 있는 오류 | ❌ 불수용 |

> **참고 출처:**
> - [식약처 가이드라인 원문 (한국어)](https://www.mfds.go.kr/brd/m_1060/view.do?seq=15628)
> - [Overview of South Korean Guidelines for LLMs/LMMs as Medical Devices (Korean Journal of Radiology, 2025)](https://pmc.ncbi.nlm.nih.gov/articles/PMC12123075/)
> - [South Korea's AI Framework Act: A Guide for Healthcare Professionals (Korean Journal of Radiology, 2026)](https://pmc.ncbi.nlm.nih.gov/articles/PMC13056445/)

---

### 한국 의료정보 저장 형식

한국 병원에서 환자 데이터를 저장하는 방식은 다음과 같은 흐름으로 변화하고 있습니다:

| 구분 | 형식 | 설명 |
|------|------|------|
| **현재 주력** | 각 병원 자체 EMR (전자의무기록) | 병원마다 다른 DB 구조 사용. 서로 호환이 안 되는 것이 가장 큰 문제 |
| **병원 간 교류** | HL7 CDA (문서 형식) | 진료정보교류 시 XML 문서 형태로 주고받음. 2010년대부터 사용 |
| **새 표준 (전환 중)** | KR Core FHIR (2023~) | 한국형 국제 표준. "나의 건강기록" 앱에 적용. JSON/REST API 기반으로 더 쉽게 데이터 연동 가능 |

> 💡 본 워크숍에서는 실습 편의를 위해 JSON 파일로 환자 데이터를 저장합니다. 실제 병원 시스템에서는 위 표준들을 통해 데이터를 교환하며, AI 도구는 이러한 다양한 형식의 데이터를 처리할 수 있어야 합니다.

#### 형식 비교: 같은 검사 결과를 다르게 표현하는 법

아래는 "혈소판 수 152 (정상범위 150~400)"라는 동일한 검사 결과를 두 형식으로 표현한 예시입니다.

**HL7 CDA (XML) — 병원 간 진료정보교류에 사용:**

```xml
<observation classCode="OBS" moodCode="EVN">
  <code code="26515-7" displayName="혈소판수"
        codeSystem="2.16.840.1.113883.6.1" codeSystemName="LOINC"/>
  <statusCode code="completed"/>
  <value xsi:type="PQ" value="152" unit="10*3/uL"/>
  <interpretationCode code="N" displayName="Normal"/>
  <referenceRange>
    <observationRange>
      <value xsi:type="IVL_PQ">
        <low value="150" unit="10*3/uL"/>
        <high value="400" unit="10*3/uL"/>
      </value>
    </observationRange>
  </referenceRange>
</observation>
```

**KR Core FHIR (JSON) — "나의 건강기록" 앱에서 사용:**

```json
{
  "resourceType": "Observation",
  "status": "final",
  "category": [{"coding": [{"code": "laboratory"}]}],
  "code": {
    "coding": [{
      "system": "https://hira.or.kr/CodeSystem/edi-code",
      "code": "D2212001",
      "display": "혈소판수"
    }]
  },
  "subject": {"reference": "Patient/patient-001"},
  "valueQuantity": {
    "value": 152,
    "unit": "10*3/uL"
  },
  "interpretation": [{"coding": [{"code": "N", "display": "Normal"}]}],
  "referenceRange": [{
    "low": {"value": 150, "unit": "10*3/uL"},
    "high": {"value": 400, "unit": "10*3/uL"}
  }]
}
```

**차이점 요약:**

| 항목 | HL7 CDA | KR Core FHIR |
|------|---------|-------------|
| 형식 | XML (길고 복잡) | JSON (간결하고 읽기 쉬움) |
| 데이터 교환 | 문서 전체를 한 번에 전송 | REST API로 항목별 조회 가능 |
| 코드 체계 | LOINC (국제 코드) | EDI (한국 수가코드) |
| AI 연동 난이도 | 어려움 (XML 파싱 필요) | 쉬움 (JSON 바로 사용 가능) |

> 출처: [HL7 C-CDA Examples (GitHub)](https://github.com/HL7/C-CDA-Examples), [Development of a FHIR-based Korean IPS Data Pipeline (Nature, 2025)](https://www.nature.com/articles/s41598-025-33390-z)

### 실습용 환자 데이터 구조

환자 데이터는 `data/patient-001.json`에 아래 구조로 저장되어 있습니다:

```json
{
  "patient_id": "patient-001",
  "demographics": { "name": "김민수", "age": 45, ... },
  "lab_results": {
    "CBC": { "WBC": {"value": 12.5, "unit": "10^3/uL", "flag": "HIGH"}, ... },
    "LIPID": { "TotalCholesterol": {"value": 235, ...}, ... },
    "METABOLIC": { "FastingGlucose": {"value": 115, ...}, ... },
    "LIVER": { "AST": {"value": 28, ...}, ... }
  }
}
```

### 한국 의학 가이드라인 기반 참고 기준

| 검사 항목 | 정상 (A) | 경계 (B) | 질환의심 (C) | 유질환 (D) |
|-----------|---------|----------|-------------|-----------|
| 공복혈당 (mg/dL) | <100 | 100-125 | 126-199 | >=200 |
| 총콜레스테롤 (mg/dL) | <200 | 200-239 | 240-299 | >=300 |
| LDL (mg/dL) | <130 | 130-159 | 160-189 | >=190 |
| WBC 백혈구 수 (10^3/uL)* | 4.0-10.0 | 10.1-15.0 | 15.1-30.0 | >30 또는 <2 |

> *WBC(White Blood Cell, 백혈구) 판정 구간은 워크숍 실습 용도로 생성한 데이터입니다. 실제 국가건강검진 공식 판정 기준과 다를 수 있습니다.

---

## 실습 시작

### Step 1: 도구 파일 생성

```bash
cd ~/agentcore/src
touch lab_tools.py
```

### Step 2: 검사 결과 조회 도구 구현

`lab_tools.py`를 열고 아래 코드를 작성하세요:

```python
"""
혈액 검사 결과 조회/분석 도구
- 로컬 JSON 파일에서 환자 데이터 로드
- 한국 건강검진 기준으로 판정
"""
import json
import os
from pathlib import Path
from strands import tool

# 환자 데이터가 저장된 디렉토리 경로 (환경변수로 변경 가능)
DATA_DIR = Path(os.environ.get("PATIENT_DATA_DIR", os.path.expanduser("~/agentcore/patient-data")))


def _load_patient_data(patient_id: str) -> dict:
    """환자 데이터 JSON 파일을 로드합니다."""
    filepath = DATA_DIR / f"{patient_id}.json"
    if not filepath.exists():
        raise FileNotFoundError(f"환자 {patient_id}의 데이터를 찾을 수 없습니다.")
    return json.loads(filepath.read_text(encoding="utf-8"))


# ──────────────────────────────────────────────
# 도구 1: 혈액 검사 결과 조회
# - @tool 데코레이터로 Strands Agent가 호출 가능한 도구로 등록
# - 에이전트가 환자 ID와 검사 유형을 전달하면 해당 결과를 반환
# ──────────────────────────────────────────────
@tool
def get_lab_results(patient_id: str, test_type: str = "all") -> str:
    """환자의 혈액 검사 결과를 조회합니다.
    
    Args:
        patient_id: 환자 ID (예: "patient-001")
        test_type: 검사 유형 (all, CBC, LIPID, METABOLIC, LIVER)
    
    Returns:
        검사 결과 텍스트 (JSON 형식)
    """
    # [안전성 원칙] 접근 제어: 허가된 환자 ID만 조회 가능
    ALLOWED_PATIENTS = ["patient-001", "patient-002"]
    
    if patient_id not in ALLOWED_PATIENTS:
        return f"접근 거부: 환자 {patient_id}의 데이터에 대한 접근 권한이 없습니다."
#    if patient_id != "patient-001":
#        return f"접근 거부: 환자 {patient_id}의 데이터에 대한 접근 권한이 없습니다."
    
    # 파일에서 환자 데이터 로드 시도
    try:
        data = _load_patient_data(patient_id)
    except FileNotFoundError as e:
        return str(e)
    
    lab_results = data.get("lab_results", {})
    
    # 전체 조회 또는 특정 패널(CBC, LIPID 등) 조회 분기
    if test_type == "all":
        return json.dumps(lab_results, ensure_ascii=False, indent=2)
    elif test_type.upper() in lab_results:
        return json.dumps(lab_results[test_type.upper()], ensure_ascii=False, indent=2)
    else:
        available = ", ".join(lab_results.keys())
        return f"'{test_type}' 검사 유형을 찾을 수 없습니다. 사용 가능한 유형: {available}"


# ──────────────────────────────────────────────
# 한국 의학 가이드라인 기반 판정 기준 딕셔너리
# - A: 정상, B: 경계, C: 질환의심, D: 유질환
# - 각 항목별 수치 범위를 튜플(최소, 최대)로 정의
# ──────────────────────────────────────────────
HEALTH_CRITERIA = {
    "WBC": {"unit": "10^3/uL", "A": (4.0, 10.0), "B": (10.1, 15.0), "C": (15.1, 30.0)},
    "FastingGlucose": {"unit": "mg/dL", "A": (0, 99), "B": (100, 125), "C": (126, 199), "D_min": 200},
    "TotalCholesterol": {"unit": "mg/dL", "A": (0, 199), "B": (200, 239), "C": (240, 299), "D_min": 300},
    "LDL": {"unit": "mg/dL", "A": (0, 129), "B": (130, 159), "C": (160, 189), "D_min": 190},
    "Triglyceride": {"unit": "mg/dL", "A": (0, 149), "B": (150, 199), "C": (200, 499), "D_min": 500},
}


def _grade_value(test_name: str, value: float) -> str:
    """검사 수치를 한국 건강검진 기준으로 등급 판정합니다.
    
    [설명 가능성 원칙] 판정 근거를 등급과 함께 반환하여
    AI가 왜 그런 판단을 했는지 사용자가 이해할 수 있도록 합니다.
    """
    criteria = HEALTH_CRITERIA.get(test_name)
    if not criteria:
        return "판정 기준 없음"
    
    # D등급(유질환) 판정: 최소 기준값 이상이면 즉시 D 반환
    if "D_min" in criteria and value >= criteria["D_min"]:
        return "D (유질환)"
    
    # A~C 등급을 순서대로 확인하여 해당 범위에 속하면 반환
    for grade in ["A", "B", "C"]:
        if grade in criteria:
            low, high = criteria[grade]
            if low <= value <= high:
                grade_labels = {"A": "정상", "B": "경계", "C": "질환의심"}
                return f"{grade} ({grade_labels[grade]})"
    
    return "판정 불가"


# ──────────────────────────────────────────────
# 도구 2: 검사 결과 종합 분석
# - 모든 패널의 검사 항목을 순회하며 등급 판정
# - 비정상 항목을 별도로 수집하여 요약 제공
# ──────────────────────────────────────────────
@tool
def analyze_lab_values(patient_id: str) -> str:
    """검사 결과를 한국 건강검진 기준으로 종합 분석합니다.
    
    Args:
        patient_id: 환자 ID
    
    Returns:
        종합 분석 결과 (등급별 분류 + 비정상 항목 요약)
    """
    # [안전성 원칙] 접근 제어
    if patient_id not in ALLOWED_PATIENTS:
        return f"접근 거부: 환자 {patient_id}의 데이터에 대한 접근 권한이 없습니다."
#    if patient_id != "patient-001":
#        return f"접근 거부: 환자 {patient_id}의 데이터에 대한 접근 권한이 없습니다."
    
    try:
        data = _load_patient_data(patient_id)
    except FileNotFoundError as e:
        return str(e)
    
    lab_results = data.get("lab_results", {})
    
    analysis = []       # 전체 분석 결과 저장
    abnormal_items = [] # 비정상(경계 이상) 항목만 별도 수집
    
    # 각 검사 패널(CBC, LIPID 등)을 순회
    for panel_name, panel_data in lab_results.items():
        analysis.append(f"\n## {panel_name} 패널")
        for test_name, test_info in panel_data.items():
            value = test_info["value"]
            unit = test_info["unit"]
            flag = test_info["flag"]
            
            # [설명 가능성 원칙] 각 항목의 판정 등급과 수치를 함께 표시
            grade = _grade_value(test_name, value)
            
            status = "✅" if flag == "NORMAL" else "⚠️"
            analysis.append(f"  {status} {test_name}: {value} {unit} → {grade}")
            
            if flag != "NORMAL":
                abnormal_items.append(f"{test_name}: {value} {unit} ({grade})")
    
    # 종합 요약 생성
    summary = data.get("checkup_summary", {})
    overall_grade = summary.get("overall_grade", "N/A")
    
    # [투명성 원칙] AI 분석임을 명시하고, 판정 근거를 모두 포함
    result = f"# 종합 건강검진 분석 결과\n\n"
    result += f"환자: {data['demographics']['name']} ({data['demographics']['age']}세, {data['demographics']['gender']})\n"
    result += f"검진일: {data['demographics']['checkup_date']}\n"
    result += f"종합 판정: {overall_grade}등급\n\n"
    result += f"## 비정상 항목 ({len(abnormal_items)}개)\n"
    for item in abnormal_items:
        result += f"  - {item}\n"
    result += "\n" + "\n".join(analysis)
    
    return result
```

### Step 3: 에이전트에 도구 통합

`consultation_agent.py`를 열고 도구 import를 추가하세요:

```python
# consultation_agent.py 상단에 추가
from lab_tools import ________, ________  #TODO : `lab_tools.py`에서 `@tool` 데코레이터가 붙은 함수(도구) 이름 2개

# 에이전트 생성 부분 수정
consultation_agent = Agent(
    model=BedrockModel(
        model_id="global.anthropic.claude-sonnet-4-5-20250929-v1:0",
        region_name="________" #TODO : Amazon Bedrock Claude 모델을 사용할 AWS 리전
    ),
    tools=[
        search_medical_knowledge, 
        assess_urgency,
        ________,       # TODO : 혈액 검사 결과 조회 도구 추가
        ________        # TODO : 검사 결과 종합 분석 도구 추가
    ],
    system_prompt=MEDICAL_SYSTEM_PROMPT
)
```

### Step 4: 통합 테스트

```bash
cd ~/agentcore/src
python consultation_agent.py
```

---

## 테스트 시나리오

### 시나리오 1: 전체 결과 분석 요청

```
제 건강검진 결과를 분석해 주세요. 환자 ID는 patient-001입니다.
```

**확인 포인트:**
- [ ] `analyze_lab_values` 도구가 호출됨
- [ ] 비정상 항목 5개가 정확히 식별됨 (WBC, 공복혈당, 총콜레스테롤, LDL, 중성지방)
- [ ] 각 항목의 판정 등급이 B로 표시됨

### 시나리오 2: 특정 항목 문의

```
콜레스테롤 수치가 걱정됩니다. 자세히 설명해 주세요.
```

**확인 포인트:**
- [ ] `get_lab_results`에서 LIPID 패널을 조회함
- [ ] 총콜레스테롤 235, LDL 145, 중성지방 180의 의미를 설명함
- [ ] 생활습관 개선 권고를 포함함

### 시나리오 3: 접근 제어 테스트

```
patient-002의 검사 결과를 보여주세요.
```

**확인 포인트:**
- [ ] "접근 거부" 메시지가 반환됨
- [ ] 다른 환자의 데이터가 노출되지 않음

---

## 검증

- ✅ 환자 김민수의 비정상 항목 5개가 정확히 식별됨
- ✅ 한국 건강검진 기준(A/B/C/D)으로 등급이 산출됨
- ✅ 다른 환자 데이터 요청 시 접근 거부됨
- ✅ 에이전트가 검사 결과 + 의학 지식을 결합하여 응답함

---

## 🏆 Challenge Task: patient-002 이서연 분석

환자 ID `patient-002` (이서연, 58세 여성)의 건강검진 결과를 에이전트로 분석해보세요.

```
이서연의 건강검진 결과를 분석해 주세요. 환자 ID는 patient-002입니다.
```

### 문제

현재 코드 그대로 실행하면 **"접근 거부"** 메시지가 반환됩니다. `lab_tools.py`의 도구에서 접근 제어 로직을 수정하여 patient-002도 조회 가능하도록 변경하세요.

### 기대 결과

- [ ] patient-002의 비정상 항목 **11개**가 식별됨
- [ ] 종합 판정 **C등급 (질환의심)**으로 표시됨
- [ ] 빈혈, 당뇨 조절 불량, 신장 기능 저하, 간 기능 이상이 모두 감지됨
- [ ] 5개 진료과 추가 검사 권고가 포함됨

---

완료 후 [Phase 2-B: 메모리 통합](./032_memory.md)로 이동하세요.

---

## 부록: 정답 코드 — consultation_agent.py (Phase 2-A 통합 완료 버전)

<details>
<summary>정답 코드 펼치기 (클릭)</summary>

아래는 Phase 1에서 만든 `consultation_agent.py`에 혈액 검사 도구를 통합한 전체 코드입니다.

```python
"""
AI 건강 상담 에이전트 (Phase 2-A: 혈액 검사 도구 통합 완료)
- Strands Agents SDK + Amazon Bedrock Claude Sonnet 4.5
- 의료 지식 검색, 긴급도 평가, 혈액 검사 조회/분석 도구 포함
"""
import time
from strands import Agent, tool
from strands.models import BedrockModel

# Phase 2-A에서 추가된 도구 import
from lab_tools import get_lab_results, analyze_lab_values


# ──────────────────────────────────────────────
# 시스템 프롬프트: 에이전트의 역할과 행동 규칙 정의
# [인공지능 기본법] 투명성·인간감독·설명가능성 원칙 적용
# ──────────────────────────────────────────────
MEDICAL_SYSTEM_PROMPT = """당신은 대한민국의 AI 건강 상담 보조원입니다.

## 역할
환자의 건강 검진 결과를 분석하고, 증상에 대한 일반적인 의학 정보를 제공합니다.

## 반드시 지켜야 할 규칙

1. **확정적 진단 금지**: "~일 수 있습니다", "~가능성이 있습니다" 등 가능성 표현을 사용하세요.
2. **약물 처방 금지**: 구체적인 약물명이나 용량을 제시하지 마세요. 의료기관 방문을 안내하세요.
3. **면책 조항 필수**: 모든 응답 끝에 "본 정보는 AI가 제공하는 참고 정보이며, 정확한 진단과 치료를 위해 반드시 의료 전문가와 상담하세요."를 포함하세요.
4. **응급 증상 감지**: 흉통, 호흡곤란, 의식 소실, 심한 출혈 등 응급 증상이 감지되면 즉시 119 또는 가까운 응급실 방문을 안내하세요.

## 한국 건강검진 판정 기준
- A (정상): 건강한 상태
- B (경계): 생활습관 개선 권고
- C (질환의심): 추가 검사 또는 진료 권고
- D (유질환): 치료가 필요한 상태
- R (판정보류): 추가 검사 필요

## 응답 형식
- 한국어로 응답하세요.
- 검사 결과 설명 시 정상 범위와 비교하여 설명하세요.
- 생활습관 개선 방안을 포함하세요.
"""


# ──────────────────────────────────────────────
# 도구 1: 의료 지식 검색
# - 에이전트가 증상/검사 항목에 대한 기본 의학 정보를 조회
# - 실제 운영 시에는 RAG(검색증강생성) 시스템으로 대체
# ──────────────────────────────────────────────
@tool
def search_medical_knowledge(query: str, category: str = "general") -> str:
    """의료 지식 데이터베이스에서 증상이나 검사 항목 정보를 검색합니다.
    
    Args:
        query: 검색할 의료 키워드 (예: "두통", "콜레스테롤", "혈당")
        category: 카테고리 (general, symptom, lab_test, emergency)
    
    Returns:
        관련 의료 정보 텍스트
    """
    # 간단한 키워드 매칭 기반 의료 정보 (워크숍용 시뮬레이션)
    knowledge_base = {
        "두통": {
            "description": "두통은 다양한 원인에 의해 발생할 수 있는 증상입니다.",
            "common_causes": ["긴장성 두통", "편두통", "군발성 두통", "부비동염", "고혈압"],
            "warning_signs": ["갑자기 발생한 극심한 두통", "발열 동반", "시력 변화", "의식 변화"],
            "recommendation": "2주 이상 지속되거나 경고 증상이 동반되면 신경과 진료를 권합니다."
        },
        "WBC": {
            "description": "WBC(백혈구)는 면역 체계의 핵심 세포입니다.",
            "normal_range": "4,000-10,000/uL",
            "high_causes": ["감염", "염증", "스트레스", "흡연", "약물 반응", "백혈병(드물게)"],
            "grade_criteria": "정상(4-10), 경계(10.1-15), 질환의심(15.1-30), 유질환(>30 또는 <2)",
            "recommendation": "경미한 상승은 일시적 감염이나 스트레스로 인한 경우가 많습니다."
        },
        "콜레스테롤": {
            "description": "콜레스테롤은 세포막 구성과 호르몬 합성에 필요하지만, 과다 시 동맥경화 위험이 증가합니다.",
            "criteria": "총콜레스테롤 정상(<200), 경계(200-239), 질환의심(240-299), 유질환(>=300)",
            "ldl_criteria": "LDL 정상(<130), 경계(130-159), 질환의심(160-189), 유질환(>=190)",
            "recommendation": "식이요법(포화지방 줄이기), 규칙적 유산소 운동, 필요시 약물 치료"
        },
        "혈당": {
            "description": "공복혈당은 당뇨병 선별 검사의 기본 항목입니다.",
            "criteria": "정상(<100), 경계/공복혈당장애(100-125), 당뇨의심(126-199), 당뇨(>=200)",
            "hba1c": "HbA1c 정상(<5.7%), 전당뇨(5.7-6.4%), 당뇨(>=6.5%)",
            "recommendation": "경계 수준이면 생활습관 개선으로 정상화 가능합니다. 식이조절과 운동을 권합니다."
        },
        "흉통": {
            "description": "흉통은 심장, 폐, 소화기, 근골격계 등 다양한 원인이 있습니다.",
            "emergency_signs": ["운동 시 악화", "왼팔/턱으로 방사", "호흡곤란 동반", "식은땀"],
            "recommendation": "심장 관련 의심 증상이 있으면 즉시 응급실을 방문하세요."
        }
    }
    
    # 키워드 매칭으로 관련 정보 검색
    results = []
    query_lower = query.lower()
    for key, info in knowledge_base.items():
        if key.lower() in query_lower or query_lower in key.lower():
            results.append(f"[{key}]\n" + "\n".join(f"  {k}: {v}" for k, v in info.items()))
    
    if results:
        return "\n\n".join(results)
    else:
        return f"'{query}'에 대한 정보를 찾을 수 없습니다. 일반적인 의학 지식을 바탕으로 답변하겠습니다."


# ──────────────────────────────────────────────
# 도구 2: 증상 긴급도 평가
# - 응급/주의/일반으로 3단계 분류
# - [안전성 원칙] 응급 증상 감지 시 즉시 119 안내
# ──────────────────────────────────────────────
@tool
def assess_urgency(symptoms: str, duration_days: int = 0) -> str:
    """증상의 긴급도를 평가합니다.
    
    Args:
        symptoms: 환자가 호소하는 증상 설명
        duration_days: 증상 지속 기간 (일)
    
    Returns:
        긴급도 평가 결과 (긴급/주의/일반)
    """
    # 긴급 키워드 (즉시 응급실 방문 필요)
    emergency_keywords = ["흉통", "가슴 통증", "호흡곤란", "숨이 안 쉬어",
                          "의식 소실", "기절", "마비", "심한 출혈", "경련"]
    
    # 주의 키워드 (빠른 진료 필요)
    caution_keywords = ["고열", "38도 이상", "혈뇨", "혈변", 
                        "극심한 통증", "시력 저하", "갑작스런"]
    
    symptoms_lower = symptoms.lower()
    
    # 긴급 판정
    for keyword in emergency_keywords:
        if keyword in symptoms_lower:
            return f"🚨 [긴급] '{keyword}' 증상이 감지되었습니다. 즉시 119에 전화하거나 가까운 응급실을 방문하세요."
    
    # 주의 판정
    for keyword in caution_keywords:
        if keyword in symptoms_lower:
            return f"⚠️ [주의] '{keyword}' 관련 증상입니다. 가능한 빨리 (24-48시간 이내) 의료기관을 방문하시기 바랍니다."
    
    # 지속 기간 확인
    if duration_days > 14:
        return f"ℹ️ [일반-장기지속] 증상이 {duration_days}일간 지속되고 있습니다. 정확한 원인 파악을 위해 의료기관 방문을 권합니다."
    
    return "ℹ️ [일반] 현재 응급 상황은 아닌 것으로 판단됩니다. 증상이 악화되면 의료기관을 방문하세요."


# ──────────────────────────────────────────────
# 에이전트 생성
# - Phase 2-A: get_lab_results, analyze_lab_values 도구 추가
# - 총 4개 도구로 구성
# ──────────────────────────────────────────────
consultation_agent = Agent(
    model=BedrockModel(
        model_id="global.anthropic.claude-sonnet-4-5-20250929-v1:0",
        region_name="us-west-2"
    ),
    tools=[
        search_medical_knowledge,   # 의료 지식 검색
        assess_urgency,             # 긴급도 평가
        get_lab_results,            # [Phase 2-A] 혈액 검사 결과 조회
        analyze_lab_values          # [Phase 2-A] 검사 결과 종합 분석
    ],
    system_prompt=MEDICAL_SYSTEM_PROMPT
)


# ──────────────────────────────────────────────
# 대화형 테스트 루프
# - 터미널에서 직접 실행하여 에이전트와 대화
# - 'quit' 입력 시 종료
# ──────────────────────────────────────────────
if __name__ == "__main__":
    print("=" * 60)
    print("🏥 AI 건강 상담 에이전트 (Phase 2-A: 검사 도구 통합)")
    print("=" * 60)
    print("환자의 증상이나 건강검진 결과에 대해 질문하세요.")
    print("종료하려면 'quit'을 입력하세요.\n")
    
    while True:
        user_input = input("👤 환자: ").strip()
        if user_input.lower() in ["quit", "exit", "종료"]:
            print("상담을 종료합니다. 건강하세요! 👋")
            break
        if not user_input:
            continue
        
        print("\n🤖 상담원: ", end="")
        start_time = time.time()
        
        response = consultation_agent(user_input)
        
        elapsed = time.time() - start_time
        print(f"\n\n⏱️  응답 시간: {elapsed:.1f}초")
        
        # 메트릭 로깅 (도구 호출 정보 등)
        if hasattr(response, 'metrics'):
            print(f"📊 메트릭: {response.metrics}")
        print("-" * 60)
```

</details>
