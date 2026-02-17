# 통합 경정청구 환급액 산출 시스템 - 엔티티 관계 정의서

> **문서 버전**: v1.0 (외부자료 v4.0 기반)
> **작성일**: 2026-02-17

---

## 1. 핵심 엔티티 관계도 (ERD)

### 1.1 최상위 엔티티 관계

```
RI_S_APPLICANT (신청인)         RI_S_TAXPAYER (납세자)
   1 ─────────── N                 1 ─────────── N
         │                                │
         │  FK: applicant_id              │  FK: taxpayer_id
         │                                │
         ▼                                ▼
   ┌──────────────────────────────────────────────┐
   │         RI_S_REQUEST (요청 마스터)              │
   │                req_id (PK)                    │
   │    1 Request = 1 경정청구 진단 트랜잭션          │
   └──────────────────┬───────────────────────────┘
                      │
          ┌───────────┼───────────┬──────────────┐
          │           │           │              │
     1:N  │      1:N  │      1:N  │         1:N  │
          ▼           ▼           ▼              ▼
   RI_S_RAW_DATA   SV_* 테이블   CO_* 테이블   AL_S_CALCULATION
   (원시 JSON)     (요약/검증)   (산출 결과)    (감사 로그)
                      │
                      │ 참조 (FK JOIN)
                      ▼
               RF_* 기준정보 (독립)
```

### 1.2 3계층 데이터 아키텍처

```
┌─────────────────────────────────────────────────────────┐
│                 RI_S_REQUEST (요청 마스터)                 │
│            req_id = 신청인 + 날짜 기반 유니크 키            │
└──────────┬──────────────────────────────┬───────────────┘
           │ FK: req_id                    │ FK: req_id
  ┌────────▼────────┐            ┌────────▼────────┐
  │   입력자료 계층    │            │   출력결과 계층    │
  │                  │            │                  │
  │ RI_S_RAW_DATA   │            │  CO_* 테이블     │
  │ (원시 JSON)      │            │  (산출 결과)      │
  │     │ 파싱       │            │     │ 직렬화     │
  │ SV_* 요약       │            │ CO_S_REPORT_JSON │
  │ (계산용)         │            │  (전달용 JSON)    │
  └──────────────────┘            └──────────────────┘
           │                               │
           └───────► 점검/산출 엔진 ◄────────┘
                     (M2~M5)
                     + RF_* 참조
```

---

## 2. 엔티티 간 상세 관계

### 2.1 요청 관리 관계 (Request Management)

| 부모 엔티티 | 자식 엔티티 | 관계 | FK | 설명 |
|------------|-----------|------|-----|------|
| RI_S_APPLICANT | RI_S_REQUEST | 1:N | applicant_id | 한 신청인이 여러 요청 가능 |
| RI_S_TAXPAYER | RI_S_REQUEST | 1:N | taxpayer_id | 한 납세자에 대해 복수 요청 가능 |
| RI_S_REQUEST | RI_S_RAW_DATA | 1:N | req_id | 1 요청 : N 카테고리별 원시 JSON |
| RI_S_REQUEST | 모든 SV_* | 1:1 또는 1:N | req_id | 요청별 요약/검증 데이터 |
| RI_S_REQUEST | 모든 CO_* | 1:1 또는 1:N | req_id | 요청별 산출 결과 |
| RI_S_REQUEST | AL_S_CALCULATION | 1:N | req_id | 요청별 감사 로그 (APPEND) |

### 2.2 입력-요약 관계 (Input to Summary)

```
RI_S_RAW_DATA (원시 JSON)
    │
    │ M1-03 파싱 모듈
    │
    ├──▶ SV_S_BASIC         (1:1)  기본정보 요약
    ├──▶ SV_S_EMPLOYEE      (1:2)  고용정보 요약 (PRIOR/CURRENT)
    ├──▶ SV_S_DEDUCTION     (1:N)  공제/감면 기초 요약
    └──▶ SV_S_FINANCIAL     (1:1)  재무/세무 수치 요약
```

| 원시 JSON 카테고리 | 요약 테이블 | 변환 방식 |
|-------------------|-----------|----------|
| corp_basic / inc_basic | SV_S_BASIC | 핵심 필드 추출 |
| employee_detail + employee_monthly | SV_S_EMPLOYEE | 월별 집계, 연평균 산정 |
| investment + rd_expense + startup + sme_special + ... | SV_S_DEDUCTION | 항목별 기초 수치 추출 |
| financial + tax_adjustment + dividend + deduction | SV_S_FINANCIAL | 재무 수치 집계 |

### 2.3 요약-점검 관계 (Summary to Validation)

```
SV_S_BASIC + SV_S_EMPLOYEE + SV_S_DEDUCTION + SV_S_FINANCIAL
    │
    │ M3-PREP 사전 점검
    │
    ├──▶ SV_S_PREP_SUMMARY    (1:1)  사전 데이터 준비 요약
    │
    │ M3-00~M3-06 점검
    │
    ├──▶ SV_S_ELIGIBILITY     (1:1)  자격 진단 결과
    ├──▶ SV_S_INSPECTION_LOG  (1:N)  점검항목별 판정
    └──▶ SV_S_VALIDATION_LOG  (1:N)  검증 규칙 실행 결과
```

### 2.4 산출 결과 관계 (Calculation Output)

```
SV_* 요약 데이터 + RF_* 기준정보
    │
    │ M4 개별 산출
    │
    ├──▶ CO_S_EMPLOYEE_SUMMARY  (1:2)   상시근로자 확정 (PRIOR/CURRENT)
    └──▶ CO_S_CREDIT_DETAIL     (1:N)   개별 공제/감면 산출
              │
              │ M5 최적 조합
              │
              ├──▶ CO_S_COMBINATION       (1:N)   조합 비교/최적 선택
              ├──▶ CO_S_EXCLUSION_VERIFY   (1:N)   상호배제 검증
              │
              │ M6 최종 산출
              │
              ├──▶ CO_S_REFUND            (1:1)   최종 환급액
              │         │
              │         │ FK: optimal_combo_id
              │         └──▶ CO_S_COMBINATION (참조)
              │
              ├──▶ CO_S_RISK              (1:N)   사후관리/리스크
              ├──▶ CO_S_ADDITIONAL_CHECK   (1:N)   추가 확인 필요
              └──▶ CO_S_REPORT_JSON       (1:1)   보고서 JSON (7섹션)
```

### 2.5 개인 전용 산출 관계 (INC Only)

```
SV_S_BASIC (INC) + RI_I_BUSINESS + RI_I_OTHER_INCOME
    │
    │ P3-03, P3-04 종합소득/과세표준 산정
    │ P4-02 감면소득배분, P4-03 공동사업자 배분
    │
    ├──▶ CO_I_INCOME_ALLOCATION  (1:N)  다사업장 소득 배분
    ├──▶ CO_I_DEDUCTION_IMPACT   (1:1)  소득공제 세율구간 변동
    └──▶ CO_I_LOCAL_TAX          (1:1)  지방소득세 별도 산출
```

---

## 3. 기준정보 참조 관계 (RF_ Reference)

> RF_* 테이블은 req_id 없이 독립 관리. 계산 엔진이 참조만 함.

### 3.1 세율/세액 계산 참조

| 참조 대상 | 참조 테이블 | 참조 키 | 사용 모듈 |
|----------|-----------|---------|----------|
| 법인세 산출세액 | RF_S_TAX_RATE, RF_C_TAX_RATE_HISTORY | tax_year, bracket | M3, F03 |
| 소득세 산출세액 | RF_I_TAX_RATE | effective_from, bracket_no | P3-04, F-INC-03 |
| 최저한세 (법인) | RF_S_MIN_TAX_RATE | corp_size, bracket | M5-03, F04 |
| 최저한세 (개인) | RF_I_MIN_TAX | effective_from | P5-01, F-INC-06 |

### 3.2 공제/감면 기준 참조

| 참조 대상 | 참조 테이블 | 참조 키 |
|----------|-----------|---------|
| 통합고용 공제단가 | RF_S_EMPLOYMENT_CREDIT | corp_size, region, worker_type |
| 통합투자 공제율 | RF_C_INVESTMENT_CREDIT_RATE | invest_type, corp_size |
| R&D 공제율 | RF_C_RD_CREDIT_RATE | rd_type, method, corp_size |
| R&D 최저한세 배제율 | RF_C_RD_MIN_TAX_EXEMPT | rd_type, corp_size |
| 창업감면 감면율 | RF_S_STARTUP_DEDUCTION_RATE | founder_type, location_type |
| 중소특별 감면율 | RF_S_SME_DEDUCTION_RATE | corp_size_detail, industry_class, zone_type |
| 상호배제 규칙 | RF_S_MUTUAL_EXCLUSION | provision_a, provision_b, year |
| 농특세 적용 | RF_S_NONGTEUKSE | provision |

### 3.3 지역/업종 판단 참조

| 참조 대상 | 참조 테이블 | 참조 키 |
|----------|-----------|---------|
| 수도권 구분 | RF_S_CAPITAL_ZONE | sido, sigungu |
| 인구감소지역 | RF_S_DEPOPULATION_AREA | sido, sigungu |
| 업종 적격성 | RF_S_INDUSTRY_ELIGIBILITY | ksic_code |
| 업종 코드 검증 | RF_S_KSIC_CODE | ksic_code |

### 3.4 기타 참조

| 참조 대상 | 참조 테이블 | 참조 키 |
|----------|-----------|---------|
| 환급가산금 이율 | RF_S_REFUND_INTEREST_RATE | effective_from/to |
| 외국환 환율 | RF_S_EXCHANGE_RATE | rate_date, currency |
| 법령 버전 | RF_S_LAW_VERSION | law_name, provision, year |
| 시스템 파라미터 | RF_S_SYSTEM_PARAM | param_key |
| 접대비 한도 | RF_C_ENTERTAINMENT_LIMIT | corp_size, revenue_bracket |
| 인정이자율 | RF_C_DEEMED_INTEREST_RATE | year |
| 수입배당금 불산입률 | RF_C_DIVIDEND_EXCLUSION | year, share_ratio |
| 소득공제 한도 | RF_I_DEDUCTION_LIMIT | deduction_type, income_bracket |
| 성실신고 기준 | RF_I_SINCERITY_THRESHOLD | industry_group |

---

## 4. 데이터 처리 파이프라인 (Data Flow)

### 4.1 전체 처리 흐름

```
[Client] ──1. JSON 전송──▶ [API-01 POST /requests]
                              │
                              ▼
                    2. req_id 발급 + 원본 보관
                    RI_S_REQUEST + RI_S_RAW_DATA
                              │
               ╔══════════════╧══════════════╗
               ║  Step 1: 저장 & 파싱          ║
               ║  M1-03: RAW_DATA → SV_* 요약  ║
               ╚══════════════╤══════════════╝
                              │
               ╔══════════════╧══════════════╗
               ║  Step 2: 사전 점검 (M3)       ║
               ║  M3-PREP → M3-00~06 / P3-*   ║
               ║  산출: SV_S_PREP_SUMMARY      ║
               ║       SV_S_ELIGIBILITY       ║
               ║       SV_S_INSPECTION_LOG    ║
               ║       SV_S_VALIDATION_LOG    ║
               ╚══════════════╤══════════════╝
                              │
               ╔══════════════╧══════════════╗
               ║  Step 3: 개별 산출 (M4)       ║
               ║  14개+ 서브모듈 병렬 산출       ║
               ║  + RF_* 기준정보 참조          ║
               ║  산출: CO_S_CREDIT_DETAIL     ║
               ╚══════════════╤══════════════╝
                              │
               ╔══════════════╧══════════════╗
               ║  Step 4: 최적 조합 (M5)       ║
               ║  상호배제 → 최저한세 → 농특세   ║
               ║  → 적용순서                   ║
               ║  산출: CO_S_COMBINATION       ║
               ║       CO_S_EXCLUSION_VERIFY  ║
               ╚══════════════╤══════════════╝
                              │
               ╔══════════════╧══════════════╗
               ║  Step 5: 최종 산출 (M6)       ║
               ║  환급액 → 환급가산금 → 보고서    ║
               ║  산출: CO_S_REFUND           ║
               ║       CO_S_RISK             ║
               ║       CO_S_REPORT_JSON      ║
               ╚══════════════╤══════════════╝
                              │
                              ▼
                    7. API-04 응답 ──▶ [Client]
```

### 4.2 모듈별 입출력 매핑

| 모듈 | 입력 | 출력 | 참조 |
|------|------|------|------|
| M1-03 | RI_S_RAW_DATA | SV_S_BASIC, SV_S_EMPLOYEE, SV_S_DEDUCTION, SV_S_FINANCIAL | - |
| M3-PREP | SV_* 4개 | SV_S_PREP_SUMMARY | RF_S_LAW_VERSION, RF_S_TAX_RATE, RF_I_TAX_RATE |
| M3-00~06 | SV_S_BASIC, SV_S_PREP_SUMMARY | SV_S_ELIGIBILITY, SV_S_INSPECTION_LOG, SV_S_VALIDATION_LOG | RF_S_CAPITAL_ZONE, RF_S_INDUSTRY_ELIGIBILITY |
| M4-01~15 | SV_S_DEDUCTION, SV_S_EMPLOYEE | CO_S_CREDIT_DETAIL | RF_C_INVESTMENT_CREDIT_RATE, RF_C_RD_CREDIT_RATE, RF_S_EMPLOYMENT_CREDIT 등 |
| M5-01~05 | CO_S_CREDIT_DETAIL | CO_S_COMBINATION, CO_S_EXCLUSION_VERIFY | RF_S_MUTUAL_EXCLUSION, RF_S_NONGTEUKSE |
| M6-01~04 | CO_S_COMBINATION, CO_S_REFUND | CO_S_REFUND, CO_S_RISK, CO_S_REPORT_JSON | RF_S_REFUND_INTEREST_RATE |

---

## 5. 세목 분기(TAX_TYPE) 관계

### 5.1 세목별 활성화 테이블

| TAX_TYPE | 입력 (RI_) | 기준정보 (RF_) | 산출 (CO_) |
|----------|-----------|--------------|-----------|
| **CORP** | RI_C_* 17개 + RI_S_* 12개 | RF_C_* 7개 + RF_S_* 15개 | CO_S_* 8개 |
| **INC** | RI_I_* 8개 + RI_S_* 12개 | RF_I_* 4개 + RF_S_* 15개 | CO_S_* 8개 + CO_I_* 3개 |

### 5.2 세목별 모듈 분기

| 모듈 | 공통 | CORP 전용 | INC 전용 |
|------|------|----------|---------|
| M1 입력 | M1-01,03~06,08,09 | M1-02,07,10~12 | P1-01~08 |
| M2 기준정보 | M2-01~09 | - | M2-10~13 |
| M3 사전점검 | M3-00~02,04~06 | M3-03(본점간주) | P3-01~04 |
| M4 개별산출 | M4-01~06 | M4-07~15, M42 | P4-01~14 |
| M5 최적조합 | M5-01,02,04,05 | M5-03(과세표준기준) | P5-01(산출세액기준) |
| M6 최종산출 | M6-01~04 | - | INC 보고서 확장 |

---

## 6. 검증 규칙 체계 (71건 + 개인 12건)

### 6.1 검증 유형 분류

| 유형 | 코드 접두어 | 건수 | 설명 |
|------|-----------|:----:|------|
| Validation | V | 19 | 입력 데이터 형식/범위 검증 |
| Calculation | C | 13 | 계산 결과 논리 검증 |
| Business | B | 11 | 업무 규칙 준수 검증 |
| Legal | L | 9 | 법령 적합성 검증 |
| Cross | X | 13 | 교차 검증 (항목 간 정합성) |
| Warning | W | 6 | 경고 (Hard Fail 아님) |
| **개인 전용** | VP/CP/BP/LP/WP | 12 | 종합소득세 전용 검증 |
| **합계** | | **83** | |

### 6.2 개인사업자 전용 검증 규칙 (12건)

| 코드 | 유형 | 설명 |
|------|------|------|
| VP01 | Validation | TAX_TYPE='INC' 시 RI_I_BASIC 필수 |
| VP02 | Validation | 다사업장 시 RI_I_BUSINESS 최소 1건 |
| VP03 | Validation | 공동사업자 손익분배비율 합계 = 100% |
| CP01 | Calculation | 감면세액 &le; 산출세액 &times; (감면대상소득&divide;종합소득금액) &times; 감면율 |
| CP02 | Calculation | 소득공제 후 과세표준 &ge; 0 |
| CP03 | Calculation | 개인 최저한세 = 산출세액 &times; 35%(3천만이하) + 45%(초과분) |
| BP01 | Business | 기장세액공제 + 조특법 감면 중복 허용 확인 |
| BP02 | Business | 성실신고확인비용 = 최저한세 대상 |
| BP03 | Business | 성실사업자 의료비/교육비 = 최저한세 비대상 |
| LP01 | Legal | 개인 적용순서 = 소득세법 &sect;60 |
| LP02 | Legal | 개인 경정청구 기산일 = 5.31 (성실: 6.30) |
| WP01 | Warning | 분리과세 소득이 종합소득금액에서 제외되었는지 확인 |

---

## 7. JSON 전달 스키마 (7섹션)

### 7.1 보고서 JSON 구조 (CO_S_REPORT_JSON)

| 섹션 | 컬럼 | 소스 테이블 | 설명 |
|------|------|-----------|------|
| A | section_a_json | SV_S_ELIGIBILITY + SV_S_BASIC | 해당가능성 진단 |
| B | section_b_json | CO_S_CREDIT_DETAIL | 후보 항목 리스트 |
| C | section_c_json | CO_S_COMBINATION + CO_S_EXCLUSION_VERIFY | 최적조합/배제 |
| D | section_d_json | CO_S_REFUND | 환급/추징 금액 |
| E | section_e_json | SV_S_INSPECTION_LOG | 산출 진행 로그 |
| F | section_f_json | CO_S_RISK + CO_S_ADDITIONAL_CHECK | 주의/사후관리 |
| G | section_g_meta | report_meta + taxpayer_info | 제출 데이터/메타 |

---

## 8. 핵심 설계 원칙 요약

| 원칙 | 설명 |
|------|------|
| **Request-Driven** | 1회 경정청구 진단 = 1 트랜잭션 단위 (req_id 관통) |
| **원본 불변성** | RI_S_RAW_DATA = INSERT ONLY, 수정 시 새 req_id 발급 |
| **요약 재생성** | SV_* = RI_S_RAW_DATA에서 언제든 재파싱 가능 (파생 데이터) |
| **결과 이중 보관** | CO_* 정규 테이블 + CO_S_REPORT_JSON (JSON 직렬화) |
| **기준정보 독립** | RF_* = req_id 없이 독립 관리 (법령/세율 공통 기준) |
| **세목 분기** | TAX_TYPE(CORP/INC)으로 모듈/테이블/기준정보 자동 활성화 |
| **절사 원칙** | 금액 10원 미만 TRUNCATE, 반올림 절대 금지 |
