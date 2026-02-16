# 경정청구 환급액 계산 시스템 도메인 모델 (Domain Model)

> **Project**: TaxServiceENTEC
> **Version**: 1.0
> **Date**: 2026-02-17
> **Phase**: Phase 1 - Schema/Terminology Definition

---

## 1. 핵심 엔티티 식별

### 1.1 Core Entities (핵심 엔티티)

```
┌─────────────┐   ┌──────────────┐   ┌──────────────┐
│  Applicant  │   │   Taxpayer   │   │   Request    │
│  (신청인)    │   │   (납세자)    │   │   (요청)      │
└──────┬──────┘   └──────┬───────┘   └──────┬───────┘
       │                 │                   │
       └────────┬────────┘                   │
                │  N:1                       │
                ▼                            ▼
┌──────────────────────────────────────────────────┐
│                  Amendment Claim                  │
│                 (경정청구 진단 요청)                │
│        req_id = 1 Transaction Unit               │
└──────────────────┬───────────────────────────────┘
                   │
       ┌───────────┼───────────┐
       ▼           ▼           ▼
┌────────┐   ┌──────────┐  ┌──────────┐
│ Input  │   │ Calc     │  │ Output   │
│ (입력)  │   │ Engine   │  │ (산출)    │
│ RI_*   │   │ M3~M5    │  │ CO_*     │
└────────┘   └──────────┘  └──────────┘
```

### 1.2 Entity Catalog

| Entity | 한글명 | 역할 | 주요 테이블 |
|--------|--------|------|-----------|
| **Applicant** | 신청인 | 경정청구를 신청하는 주체 | RI_S_APPLICANT |
| **Taxpayer** | 납세자 | 경정청구 대상 법인/개인 | RI_S_TAXPAYER |
| **Request** | 요청 | 1회 경정청구 진단 트랜잭션 | RI_S_REQUEST |
| **RawData** | 원시 데이터 | 불변 원본 JSON | RI_S_RAW_DATA |
| **TaxpayerSummary** | 납세자 요약 | 계산용 정규화 데이터 | SV_S_BASIC/EMPLOYEE/DEDUCTION/FINANCIAL |
| **Eligibility** | 자격 진단 | 사전점검 결과 | SV_S_ELIGIBILITY |
| **CreditItem** | 공제·감면 항목 | 14개 개별 산출 결과 | CO_S_CREDIT_DETAIL |
| **Combination** | 조합 | 그룹별 비교·최적 선택 | CO_S_COMBINATION |
| **Refund** | 환급액 | 최종 환급 산출 | CO_S_REFUND |
| **Report** | 보고서 | 3종 보고서 JSON | CO_S_REPORT_JSON |
| **ReferenceData** | 기준 정보 | 세율·법령 등 공통 기준 | RF_* (26개) |

---

## 2. 엔티티 관계도 (ER Diagram)

### 2.1 Request-Driven 모델 (핵심 관계)

```
  RI_S_APPLICANT                    RI_S_TAXPAYER
  ┌──────────────┐                  ┌───────────────────┐
  │ applicant_id │ PK               │ taxpayer_id       │ PK
  │ applicant_type│                  │ taxpayer_type     │ (CORP/INDV)
  │ agent_name   │                  │ biz_reg_no        │
  │ agent_reg_no │                  │ taxpayer_name     │
  └──────┬───────┘                  │ industry_code     │
         │ FK                       │ corp_size         │
         │                          └────────┬──────────┘
         │                                   │ FK
         ▼                                   ▼
  ┌──────────────────────────────────────────────────────┐
  │                    RI_S_REQUEST                       │
  │  req_id (PK)                                         │
  │  applicant_id (FK) ─────→ RI_S_APPLICANT             │
  │  taxpayer_id  (FK) ─────→ RI_S_TAXPAYER              │
  │  tax_type (CORP/INC)                                 │
  │  request_date, request_status                        │
  └────────┬─────────────────────────┬───────────────────┘
           │                         │
     ┌─────┴──────┐           ┌──────┴───────┐
     │ 법인(CORP)  │           │ 개인(INC)    │
     ▼            ▼           ▼              ▼
  RI_C_*       RI_S_*      RI_I_*        RI_S_*
  (17개)       (공동 12개)  (8개)         (공동 12개)
```

### 2.2 세목 분기 (TAX_TYPE) 패턴

```
                    RI_S_REQUEST
                    tax_type = ?
                         │
            ┌────────────┼────────────┐
            │            │            │
         CORP           INC        SHARED
            │            │            │
     ┌──────┴──────┐  ┌──┴──────┐  ┌─┴───────────┐
     │ RI_C_* (17) │  │RI_I_*(8)│  │RI_S_* (12)  │
     │ RF_C_*  (7) │  │RF_I_*(4)│  │RF_S_* (15)  │
     │             │  │CO_I_*(3)│  │SV_S_*  (8)  │
     │             │  │         │  │CO_S_*  (8)  │
     └─────────────┘  └─────────┘  │AL_S_*  (1)  │
                                   └─────────────┘
```

### 2.3 데이터 흐름 관계

```
┌────────────────────────────────────────────────────────────────┐
│ Layer 1: INPUT (입력 계층)                                       │
│                                                                │
│  RI_S_RAW_DATA ──파싱──▶ SV_S_BASIC                            │
│  (원시 JSON)              SV_S_EMPLOYEE                         │
│  INSERT ONLY             SV_S_DEDUCTION                        │
│                          SV_S_FINANCIAL                         │
│                                                                │
│  RI_C_* / RI_I_* (세목별 입력 스키마 정의)                         │
└──────────────────────────────┬─────────────────────────────────┘
                               │ FK: req_id
                               ▼
┌────────────────────────────────────────────────────────────────┐
│ Layer 2: PROCESS (처리 계층)                                     │
│                                                                │
│  M3: SV_S_PREP_SUMMARY ── SV_S_ELIGIBILITY                     │
│      SV_S_INSPECTION_LOG ── SV_S_VALIDATION_LOG                 │
│                                                                │
│  M4: CO_S_EMPLOYEE_SUMMARY ── CO_S_CREDIT_DETAIL (14개 항목)     │
│      CO_I_INCOME_ALLOCATION ── CO_I_DEDUCTION_IMPACT (개인)      │
│                                                                │
│  M5: CO_S_EXCLUSION_VERIFY ── CO_S_COMBINATION                  │
│                      ▲                                          │
│                      │ 참조                                      │
│                 RF_* (26개 기준정보)                               │
└──────────────────────────────┬─────────────────────────────────┘
                               │ FK: req_id
                               ▼
┌────────────────────────────────────────────────────────────────┐
│ Layer 3: OUTPUT (산출 계층)                                       │
│                                                                │
│  M6: CO_S_REFUND ── CO_S_RISK ── CO_S_ADDITIONAL_CHECK          │
│      CO_I_LOCAL_TAX (개인)                                       │
│      CO_S_REPORT_JSON (7섹션 A~G)                               │
│      AL_S_CALCULATION (감사 로그)                                 │
└────────────────────────────────────────────────────────────────┘
```

---

## 3. 핵심 관계 정의

### 3.1 1:1 관계

| 부모 | 자식 | 관계키 | 설명 |
|------|------|--------|------|
| RI_S_REQUEST | RI_S_RAW_DATA | req_id | 요청당 1개 원시 JSON |
| RI_S_REQUEST | RI_C_CORP_BASIC | req_id | 법인 요청당 1개 기본정보 |
| RI_S_REQUEST | RI_I_BASIC | req_id | 개인 요청당 1개 기본정보 |
| RI_S_REQUEST | SV_S_BASIC | req_id | 요청당 1개 기본요약 |
| RI_S_REQUEST | SV_S_ELIGIBILITY | req_id | 요청당 1개 자격진단 |
| RI_S_REQUEST | CO_S_REFUND | req_id | 요청당 1개 환급 산출 |
| RI_S_REQUEST | CO_S_REPORT_JSON | req_id | 요청당 1개 보고서 JSON |

### 3.2 1:N 관계

| 부모 | 자식 | 관계키 | 카디널리티 | 설명 |
|------|------|--------|:---------:|------|
| RI_S_REQUEST | RI_S_EMPLOYEE_DETAIL | req_id | 1:N | 요청당 N명 근로자 |
| RI_S_REQUEST | RI_S_EMPLOYEE_MONTHLY | req_id | 1:N | 근로자×12개월 |
| RI_S_REQUEST | RI_C_BRANCH_LOCATION | req_id | 1:N | 법인 지점 N개 |
| RI_S_REQUEST | RI_C_INVESTMENT_ASSET | req_id | 1:N | 투자자산 N건 |
| RI_S_REQUEST | RI_I_BUSINESS | req_id | 1:N | 개인 사업장 N개 |
| RI_S_REQUEST | RI_I_FOREIGN_TAX | req_id | 1:N | 외국납부 N건 |
| RI_S_REQUEST | RI_I_RENTAL_REDUCTION | req_id | 1:N | 착한임대인 N건 |
| RI_S_REQUEST | RI_I_JOINT_BIZ | req_id | 1:N | 공동사업장 N개 |
| RI_S_REQUEST | CO_S_CREDIT_DETAIL | req_id | 1:N | 14개 공제·감면 항목 |
| RI_S_REQUEST | CO_S_COMBINATION | req_id | 1:N | 조합 N개 비교 |
| RI_S_REQUEST | CO_S_EXCLUSION_VERIFY | req_id | 1:N | 상호배제 쌍 검증 |
| RI_S_REQUEST | SV_S_INSPECTION_LOG | req_id | 1:N | 점검항목 N건 |
| RI_S_REQUEST | SV_S_VALIDATION_LOG | req_id | 1:N | 71개 검증 규칙 |
| RI_S_REQUEST | AL_S_CALCULATION | req_id | 1:N | 감사 로그 APPEND |
| RI_S_APPLICANT | RI_S_REQUEST | applicant_id | 1:N | 신청인당 N건 요청 |
| RI_S_TAXPAYER | RI_S_REQUEST | taxpayer_id | 1:N | 납세자당 N건 요청 |

### 3.3 참조 관계 (기준정보 → 계산엔진)

| 기준 테이블 | 참조자 | 참조 시점 | 설명 |
|-----------|--------|---------|------|
| RF_S_TAX_RATE | M4 산출엔진 | 세액 계산 | 과세표준 → 세율 매칭 |
| RF_S_MIN_TAX_RATE | M5 최적화 | 최저한세 판정 | 기업규모 → 최저한세율 |
| RF_S_EMPLOYMENT_CREDIT | M4-02 통합고용 | 공제액 산출 | 기업규모·유형별 금액 |
| RF_S_MUTUAL_EXCLUSION | M5-01 상호배제 | 조합 검증 | 조문 쌍별 배제 여부 |
| RF_S_LAW_VERSION | MX-02 법령매칭 | 전체 | 사업연도 → 적용 법령 버전 |
| RF_I_TAX_RATE | M4 (개인) | 세액 계산 | 8단계 누진세율 |
| RF_I_SINCERITY_THRESHOLD | M3 (개인) | 사전점검 | 업종별 수입금액 기준 |

---

## 4. Aggregate 패턴

### 4.1 Request Aggregate (요청 집합체)

```
Request Aggregate
├── Root: RI_S_REQUEST (req_id)
├── Identity:
│   ├── RI_S_APPLICANT (신청인)
│   └── RI_S_TAXPAYER (납세자)
├── Raw Data:
│   └── RI_S_RAW_DATA (원시 JSON)
├── Corporate Input (CORP):
│   ├── RI_C_CORP_BASIC
│   ├── RI_C_REPRESENTATIVE
│   ├── RI_C_BRANCH_LOCATION[]
│   ├── RI_C_INVESTMENT_ASSET[]
│   ├── RI_C_RD_EXPENSE
│   └── ... (17개)
├── Individual Input (INC):
│   ├── RI_I_BASIC
│   ├── RI_I_BUSINESS[]
│   ├── RI_I_OTHER_INCOME
│   ├── RI_I_DEDUCTION
│   └── ... (8개)
└── Shared Input:
    ├── RI_S_EMPLOYEE_DETAIL[]
    ├── RI_S_EMPLOYEE_MONTHLY[]
    ├── RI_S_STARTUP_INFO
    └── ... (12개)
```

### 4.2 Calculation Aggregate (산출 집합체)

```
Calculation Aggregate
├── Root: CO_S_REFUND (req_id)
├── Pre-Check:
│   ├── SV_S_PREP_SUMMARY
│   ├── SV_S_ELIGIBILITY
│   └── SV_S_VALIDATION_LOG[]
├── Individual Credits:
│   ├── CO_S_EMPLOYEE_SUMMARY
│   ├── CO_S_CREDIT_DETAIL[] (14개)
│   ├── CO_I_INCOME_ALLOCATION (개인)
│   └── CO_I_DEDUCTION_IMPACT (개인)
├── Optimization:
│   ├── CO_S_EXCLUSION_VERIFY[]
│   └── CO_S_COMBINATION[]
├── Final:
│   ├── CO_S_REFUND (환급액)
│   ├── CO_I_LOCAL_TAX (개인 지방세)
│   ├── CO_S_RISK[] (사후관리)
│   └── CO_S_ADDITIONAL_CHECK[] (추가확인)
└── Report:
    ├── CO_S_REPORT_JSON (7섹션)
    └── AL_S_CALCULATION[] (감사 로그)
```

---

## 5. 상태 전이 (State Transitions)

### 5.1 Request Status

```
RECEIVED ──▶ VALIDATING ──▶ PROCESSING ──▶ COMPLETED
   │              │              │              │
   │              ▼              ▼              │
   │          INVALID       ERROR          COMPLETED
   │          (불가사유)     (시스템오류)
   │
   └──▶ CANCELLED (사용자 취소)
```

| 상태 | 설명 | 트리거 |
|------|------|--------|
| `RECEIVED` | 요청 수신, req_id 발급 | API-01 POST /requests |
| `VALIDATING` | M3 사전점검 진행 중 | M3-00 시작 |
| `INVALID` | 경정청구 불가 판정 | M3-00 Hard Fail |
| `PROCESSING` | M4~M6 계산 진행 중 | M3 통과 후 |
| `ERROR` | 시스템 오류 발생 | 예외 발생 |
| `COMPLETED` | 전체 산출 완료 | M6 보고서 생성 완료 |
| `CANCELLED` | 사용자 취소 | 사용자 요청 |

### 5.2 Credit Item Status

```
PENDING ──▶ CALCULATING ──▶ APPLICABLE
                 │             │
                 ▼             ▼
            NOT_APPLICABLE  NEEDS_REVIEW
```

| 상태 | 설명 |
|------|------|
| `applicable` | 적용 가능 (공제·감면액 산출 완료) |
| `not_applicable` | 적용 불가 (자격 미충족) |
| `needs_review` | 추가 확인 필요 (인적 판단 필요) |

---

## 6. 법인/개인 구조 비교

### 6.1 입력 스키마 차이

| 영역 | 법인 (CORP) | 개인 (INC) |
|------|------------|-----------|
| **기본정보** | RI_C_CORP_BASIC (53컬럼) | RI_I_BASIC (31컬럼) |
| **사업장** | RI_C_BRANCH_LOCATION (지점별) | RI_I_BUSINESS (사업장별) |
| **소재지 판단** | 본점 간주 규정 (§7 한정) | 사업장별 개별 판단 |
| **과세구조** | 법인 단위 합산 | 대표자 1인 종합과세 |
| **소득공제** | 없음 (손금·익금 조정) | RI_I_DEDUCTION (18컬럼) |
| **합산소득** | 없음 | RI_I_OTHER_INCOME (10컬럼) |
| **성실신고** | 해당 없음 | RI_I_SINCERITY (7컬럼) |
| **공동사업** | 해당 없음 | RI_I_JOINT_BIZ (5컬럼) |
| **세무조정** | RI_C_TAX_ADJUSTMENT | 해당 없음 |
| **배당금** | RI_C_DIVIDEND_INCOME | 해당 없음 |
| **연결납세** | RI_C_CONSOLIDATED_SUB | 해당 없음 |

### 6.2 산출 구조 차이

| 영역 | 법인 (CORP) | 개인 (INC) |
|------|------------|-----------|
| **세율** | 9~24% (4단계) | 6~45% (8단계) |
| **최저한세 기준** | 과세표준 × 7~17% | 산출세액 × 35%/45% |
| **적용순서** | 법인세법 §59 | 소득세법 §60 |
| **감면 소득배분** | 해당 없음 | 감면세액 = 산출세액 × (감면소득/종합소득금액) × 감면율 |
| **소득구분별 배분** | 해당 없음 | CO_I_INCOME_ALLOCATION |
| **소득공제 영향** | 해당 없음 | CO_I_DEDUCTION_IMPACT |
| **지방소득세** | CO_S_REFUND에 포함 | CO_I_LOCAL_TAX (별도) |

---

## 7. 데이터 처리 파이프라인

### 7.1 3계층 파이프라인

```
 Step 1: 저장 & 파싱                 Step 2: 계산 엔진              Step 3: 결과 생성
┌────────────────────┐     ┌─────────────────────────┐     ┌──────────────────────┐
│                    │     │                         │     │                      │
│  Client            │     │  M3: 사전점검             │     │  M6: 최종산출          │
│    │               │     │   ├ M3-00: 불가사유       │     │   ├ 환급액비교표        │
│    ▼               │     │   ├ M3-01: 기한점검       │     │   ├ 환급가산금          │
│  API-01            │     │   ├ M3-02: 중소기업       │     │   ├ 지방소득세          │
│    │               │     │   ├ M3-03: 소재지        │     │   └ 보고서 3종         │
│    ├─▶ req_id 발급  │     │   └ M3-04: 상시근로자     │     │                      │
│    ├─▶ RAW_DATA    │     │                         │     │  CO_S_REFUND          │
│    │   (JSON 보관)  │     │  M4: 개별산출 (14개)      │     │  CO_S_REPORT_JSON     │
│    └─▶ SV_* 파싱    │     │   ├ M4-04: 창업감면      │     │  AL_S_CALCULATION     │
│        (4개 테이블)  │     │   ├ M4-05: 중소감면      │     │                      │
│                    │     │   ├ M4-02: 통합고용       │     │  ┌─────────────────┐  │
│  RI_S_RAW_DATA     │     │   ├ M4-01: 통합투자       │     │  │ 보고서 3종        │  │
│  SV_S_BASIC        │     │   ├ M4-06: R&D          │     │  │  경영진용         │  │
│  SV_S_EMPLOYEE     │     │   └ ...                 │     │  │  세무대리인용      │  │
│  SV_S_DEDUCTION    │     │                         │     │  │  국세청용         │  │
│  SV_S_FINANCIAL    │     │  M5: 최적조합             │     │  └─────────────────┘  │
│                    │     │   ├ M5-01: 상호배제       │     │                      │
│                    │     │   ├ M5-02: 그룹비교       │     │                      │
│                    │     │   ├ M5-03: 최저한세       │     │                      │
│                    │     │   └ MX-01: 순환참조 해결   │     │                      │
│                    │     │                         │     │                      │
│                    │     │  RF_* (26개 기준정보 참조)  │     │                      │
└────────────────────┘     └─────────────────────────┘     └──────────────────────┘
```

### 7.2 모듈 간 의존성

```
M1 (입력) ──▶ M2 (기준정보) ──▶ M3 (사전점검) ──▶ M4 (개별산출) ──▶ M5 (최적조합) ──▶ M6 (보고)
                  ▲                                                    ▲
                  │                                                    │
              RF_* 참조                                          MX-01 순환참조
                                                                MX-02 법령매칭
                                                                MX-03 검증엔진
```

---

## 8. 불변성 규칙 (Immutability Rules)

| 규칙 | 대상 | 설명 |
|------|------|------|
| **INSERT ONLY** | RI_S_RAW_DATA | 원시 JSON 수정/삭제 금지. 오류 시 새 req_id |
| **APPEND ONLY** | AL_S_CALCULATION | 감사 로그 수정/삭제 금지 |
| **재생성 가능** | SV_* | 원시 JSON에서 언제든 재파싱 가능 (파생 데이터) |
| **이중 보관** | CO_* + CO_S_REPORT_JSON | RDB 저장 + JSON 직렬화 동시 |
| **독립 관리** | RF_* | req_id 없이 법령 변경 시에만 갱신 |

---

## 9. 순환참조 해결 모델 (MX-01)

```
Iteration Loop (최대 5회):

  과세표준(i) = 소득 - 이월결손금(i-1) - 공제액(i-1)
       │
       ▼
  산출세액(i) = 과세표준(i) × 세율
       │
       ▼
  최저한세(i) = 과세표준(i) × 최저한세율     ← 법인
               산출세액(i) × 35%/45%         ← 개인
       │
       ▼
  공제한도(i) = 산출세액(i) - 최저한세(i)
       │
       ▼
  공제액(i) = MIN(공제요청액, 공제한도(i))
       │
       ▼
  수렴검증: |과세표준(i) - 과세표준(i-1)| <= 1원?
       │
       ├── YES → 수렴 완료, 결과 확정
       └── NO  → i+1 반복 (i < 5)
```

---

## 10. 상호배제 모델 (조특법 §127④)

### 10.1 상호배제 매트릭스 (핵심)

```
         §6    §7    §10   §24   §29의8  §30의4
§6       —     X     O     O     COND    O
§7       X     —     O     O     O       O
§10      O     O     —     O     O       O
§24      O     O     O     —     O       O
§29의8   COND  O     O     O     —       X
§30의4   O     O     O     O     X       —
```

| 기호 | 의미 |
|------|------|
| `O` | 동시 적용 가능 |
| `X` | 상호배제 (택1) |
| `COND` | 조건부 (2025년부터 §6+§29의8 중복배제) |
| `—` | 자기 자신 |

### 10.2 그룹별 조합 패턴

```
그룹 A (감면 중심):
  §6 창업감면 (또는) §7 중소감면
  + §10 R&D (선택)
  + §24 투자 (선택)
  + §30의4 사보 (선택)
  → 농특세: §6·§7 비과세

그룹 B (공제 중심):
  §24 투자공제
  + §29의8 통합고용
  + §10 R&D (선택)
  → 농특세: §24·§29의8 과세(20%)

비교: 실질환급액 = 공제감면합계 - 농특세
```

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-02-17 | 초기 작성 - 11 엔티티, ER 관계, 데이터 흐름, 상태전이, 순환참조/상호배제 모델 |
