# Design Document Validation Report

> **Summary**: 환급액계산프로그램 Design 문서의 완전성, 일관성, 구현가능성 검증 결과
>
> **Validation Target**: `docs/02-design/features/환급액계산프로그램.design.md`
> **Validation Date**: 2026-02-17
> **Validator**: Design Validation Agent
> **Status**: Review Required

---

## Completeness Score: 78/100

| Category | Score | Weight | Weighted Score |
|----------|:-----:|:------:|:--------------:|
| Plan-Design FR Coverage | 75% | 25% | 18.75 |
| Schema-Design Consistency | 92% | 20% | 18.40 |
| Domain Model Consistency | 85% | 15% | 12.75 |
| Glossary Consistency | 95% | 10% | 9.50 |
| Internal Consistency | 80% | 15% | 12.00 |
| External Reference Alignment | 70% | 15% | 10.50 |
| **Total** | | **100%** | **81.90 -> 78** |

> **Note**: Weighted Score 81.90에서 누락 항목의 심각도를 반영하여 78점으로 조정.

---

## 1. Plan - Design FR Coverage (75%)

### 1.1 FR Coverage Matrix

Plan 문서(tax-refund-system.plan.md)의 55개 FR을 Design 문서에서 추적한 결과.

| FR Range | Category | Total | Covered | Partial | Missing | Rate |
|----------|----------|:-----:|:-------:|:-------:|:-------:|:----:|
| FR-01~FR-10 | 입력 관리 | 10 | 9 | 1 | 0 | 95% |
| FR-11~FR-20 | 사전 점검/검증 | 14* | 12 | 1 | 1 | 86% |
| FR-21~FR-35 | 개별 공제/감면 | 22* | 13 | 3 | 6 | 59% |
| FR-36~FR-45 | 최적 조합 | 10 | 10 | 0 | 0 | 100% |
| FR-46~FR-55 | 최종 산출/보고 | 12* | 10 | 1 | 1 | 83% |
| **Total** | | **68** | **54** | **6** | **8** | **79%** |

*sub-item(a/b/c) 포함 시 68건

### 1.2 Missing/Partial FR Details

#### Missing FRs (Design에서 확인 불가)

| FR ID | Requirement | Priority | Impact |
|-------|-------------|:--------:|--------|
| FR-17 | 개인 종합소득금액 산정(사업소득+근로소득 합산) | Must | IncomeAllocationService에서 암시적으로 처리되나, 종합소득금액 산정 자체의 명시적 설계 없음 |
| FR-25 | 사회보험료세액공제(SS30-4, ~2024) | Should | M4_25_SocialInsuranceCredit 구현체 존재하나 상세 로직 기술 없음 |
| FR-29 | 이월결손금 재검토(법인SS13) | Must | M4_29_LossCarryforward 구현체 존재하나 상세 설계 없음 |
| FR-30 | 수입배당금 익금불산입(법인SS18-2, 2023 개정) | Should | M4_09_DividendExclusion 구현체 존재하나 3단계 개정 반영 설계 없음 |
| FR-31/31a | 결손금 소급공제 + NPV 비교 | Should | M4_31_LossCarryback 구현체만 존재, NPV 비교 로직 미설계 |
| FR-32/32a | 세무조정 재검토 + 업무용 승용차 | Should | 구현체 자체 누락 |
| FR-33a | 성실사업자 의료비/교육비 최저한세 비대상 별도 적용 | Must | P4_04 구현체는 있으나 최저한세 비대상 처리 명시 없음 |
| FR-55 | 다수 과세연도 시뮬레이션(M7-01) | Could | M7 모듈 전체가 설계 미수록 (AmendmentClaimService.runSimulation() 메서드 시그니처만 존재) |

#### Partial FRs (일부 설계만 존재)

| FR ID | Requirement | Coverage | Gap |
|-------|-------------|----------|-----|
| FR-09 | 이월결손금/이월세액공제 입력 | 70% | 이월세액공제 입력 스키마 설계 누락 (RI_S_CREDIT_CARRYFORWARD DDL 없음) |
| FR-12b | 개인: 성실신고확인대상 자동판정 | 80% | RF_I_SINCERITY_THRESHOLD 참조 로직 미상세 |
| FR-21c | 벤처기업 3년 감면 검증 | 50% | StartupExemption에 벤처 관련 분기 미기술 |
| FR-23a | 제29조의7 경과규정 비교(2024년 이전) | 60% | 비교 로직 코드 없음 |
| FR-27/27a | 법인 외국납부세액공제 직접+간접+손금산입 비교 | 60% | M4_07 구현체 존재하나 비교 로직 미설계 |
| FR-34/34a | 성실신고확인비용 최저한세 적용 대상 구분 | 70% | P4_05 존재하나 최저한세 대상 여부 명시 약함 |

---

## 2. Schema - Design Consistency (92%)

### 2.1 Table Count Verification

| Category | Schema (83) | Design DDL | Design Entity | Match |
|----------|:-----------:|:----------:|:-------------:|:-----:|
| RI_* (입력) | 37 | 2 (핵심만) | 참조 | Partial |
| SV_* (요약/검증) | 8 | 0 | 참조 | Ref Only |
| CO_* (산출) | 11 | 2 (핵심만) | 4 | Partial |
| AL_* (감사) | 1 | 0 | 참조 | Ref Only |
| RF_* (기준정보) | 26 | 0 | 참조 | Ref Only |
| **Total** | **83** | **4** | | |

### 2.2 DDL Detail Verification (제공된 4개 테이블)

| Table | Schema Match | Issues |
|-------|:-----------:|--------|
| `ri_s_request` | PASS | Schema 컬럼 정의와 DDL 일치. request_status CHECK 제약 차이 -- Schema에는 6개 상태(RECEIVED/PROCESSING/COMPLETED/ERROR), Design에는 6개 상태(RECEIVED/VALIDATING/PROCESSING/COMPLETED/INVALID/ERROR) |
| `ri_s_raw_data` | PASS | INSERT ONLY 트리거 포함 |
| `co_s_credit_detail` | PASS | Schema 컬럼 정의와 DDL 일치 |
| `co_s_refund` | PASS | Schema 컬럼 정의와 DDL 일치 |

### 2.3 Issues Found

| # | Severity | Issue | Details |
|---|:--------:|-------|---------|
| S-01 | Warning | DDL 범위 제한 | 83개 테이블 중 4개만 DDL 제공. 나머지 79개 테이블의 DDL은 구현 시 schema.md 기반 작성 필요 |
| S-02 | Warning | request_status 값 불일치 | Schema(8.1절): `RECEIVED/PROCESSING/COMPLETED/ERROR` (4개). Design DDL: CHECK 제약 없음. Design Entity: 6개 상태(+VALIDATING/INVALID). domain-model.md: 7개 상태(+CANCELLED) |
| S-03 | Info | Index 정의 | Design에 2개 인덱스 정의(idx_request_tax_date, idx_request_taxpayer). Schema 11.1절에 7개 인덱스 정의. Design은 부분 반영 |
| S-04 | Warning | RI_S_CREDIT_CARRYFORWARD DDL 누락 | FR-09 관련 이월세액공제 테이블의 DDL/Entity 미제공 |

---

## 3. Domain Model - Design Consistency (85%)

### 3.1 Entity Mapping

| Domain Model Entity | Design Entity | Match | Notes |
|---------------------|--------------|:-----:|-------|
| Applicant | (미정의) | Partial | AmendmentRequest에 FK(applicantId)만 참조 |
| Taxpayer | (미정의) | Partial | AmendmentRequest에 FK(taxpayerId)만 참조 |
| Request | AmendmentRequest | PASS | |
| RawData | (미정의) | Ref Only | RI_S_RAW_DATA DDL은 있으나 Domain Entity 없음 |
| TaxpayerSummary | (미정의) | Ref Only | SV_* 참조만 |
| Eligibility | (미정의) | Ref Only | PreCheckService 결과로 사용 |
| CreditItem | CreditItem | PASS | |
| Combination | Combination | PASS | |
| Refund | RefundResult | PASS | |
| Report | (미정의) | Ref Only | CO_S_REPORT_JSON 참조만 |
| ReferenceData | (미정의) | Ref Only | RF_* 참조만 |

### 3.2 Relationship Verification

| Relationship | Domain Model | Design | Match |
|-------------|-------------|--------|:-----:|
| Applicant 1:N Request | Defined | FK only | PASS |
| Taxpayer 1:N Request | Defined | FK only | PASS |
| Request 1:1 RawData | Defined | DDL FK | PASS |
| Request 1:1 Eligibility | Defined | Service only | PASS |
| Request 1:14 CreditItem | Defined | Entity | PASS |
| Request 1:N Combination | Defined | Entity | PASS |
| Request 1:1 RefundResult | Defined | Entity | PASS |
| Request 1:71 ValidationLog | Defined | MX-03 ref | PASS |

### 3.3 State Transition Verification

| State | Domain Model | Design Entity (RequestStatus) | Match |
|-------|:-----------:|:----------------------------:|:-----:|
| RECEIVED | PASS | PASS | PASS |
| VALIDATING | PASS | PASS | PASS |
| PROCESSING | PASS | PASS | PASS |
| COMPLETED | PASS | PASS | PASS |
| INVALID | PASS | PASS | PASS |
| ERROR | PASS | PASS | PASS |
| **CANCELLED** | **Defined** | **Missing** | **FAIL** |

### 3.4 Issues Found

| # | Severity | Issue | Details |
|---|:--------:|-------|---------|
| D-01 | Warning | CANCELLED 상태 누락 | Domain Model에 정의된 CANCELLED 상태가 Design의 RequestStatus enum에 미반영. 취소 API도 미설계 |
| D-02 | Warning | 6개 Domain Entity 미정의 | Applicant, Taxpayer, RawData, TaxpayerSummary, Eligibility, Report에 대한 Java Entity 클래스 미정의 (Ref Only) |
| D-03 | Info | Aggregate 패턴 부분 반영 | Domain Model의 2개 Aggregate(Request/Calculation) 중 Request Aggregate만 Design에 암시적 반영 |

---

## 4. Glossary - Design Consistency (95%)

### 4.1 Term Usage Verification

| Term (Glossary) | Code (Glossary) | Design Usage | Match |
|-----------------|-----------------|-------------|:-----:|
| 경정청구 | AMENDMENT_CLAIM | AmendmentRequest, AmendmentClaimService | PASS |
| 과세표준 | taxable_income/taxable_base | taxBase (Java), taxable_income (DB) | PASS |
| 산출세액 | computed_tax | computedTax (Java), computed_tax (DB) | PASS |
| 결정세액 | determined_tax | determinedTax (Java), determined_tax (DB) | PASS |
| 최저한세 | min_tax | minTax (Java) | PASS |
| 환급가산금 | refund_interest | interestAmount (Java), interest_amount (DB) | PASS |
| 실질 환급액 | net_refund | netRefund (Java) | PASS |
| 농어촌특별세 | nongteuk_tax | nongteukAmount (Java), nongteuk_amount (DB) | PASS |
| 세액감면 | EXEMPTION | CreditType.EXEMPTION | PASS |
| 세액공제 | CREDIT | CreditType.CREDIT | PASS |
| 상시근로자 | regular_employee | EmployeeResult (Service) | PASS |

### 4.2 req_id Format Discrepancy

| Source | Format | Prefix Convention |
|--------|--------|:----------------:|
| Glossary | `{C/I}-{사업자번호}-{YYYYMMDD}-{순번}` | C=법인, I=개인 |
| Schema | `{C/I}-{사업자번호}-{YYYYMMDD}-{순번}` | C=법인, I=개인 |
| Design | `{C/I}-{bizRegNo}-{yyyyMMdd}-{seq}` | C=법인, I=개인 |
| **외부자료_v4_0** | `{C/P}-{사업자번호}-{YYYYMMDD}-{순번}` | **C=법인, P=개인** |

### 4.3 Issues Found

| # | Severity | Issue | Details |
|---|:--------:|-------|---------|
| G-01 | Critical | req_id 접두어 불일치 (Plan 문서군 vs 외부자료) | Glossary/Schema/Design은 모두 `C/I` 사용. 외부자료_v4_0.md는 `C/P` 사용. Plan 문서군(glossary, schema, domain-model) 내부는 일관되나, 원본 설계서와 충돌. **어느 쪽이 정본인지 확정 필요** |
| G-02 | Info | Naming convention 일관성 | 변수명(camelCase), DB 컬럼(lower_snake_case), API(snake_case) 규칙이 Glossary SS16과 Design SS10.1 모두에 정의되어 일관 |

---

## 5. Design Internal Consistency (80%)

### 5.1 API - Controller Mapping

| API Endpoint | Controller (Design 11.1) | Mapped | Notes |
|-------------|-------------------------|:------:|-------|
| POST /api/v1/requests | RequestController | PASS | API-01, API-02 comment |
| GET /api/v1/requests/{req_id}/status | RequestController | PASS | |
| PUT /api/v1/requests/{req_id}/calculate | CalculationController | PASS | API-03 |
| GET /api/v1/requests/{req_id}/report | ReportController | PASS | API-04 |
| GET /api/v1/requests/{req_id}/report/{type} | ReportController | PASS | API-04 variant |
| GET /api/v1/requests/{req_id}/credits | (미매핑) | FAIL | Controller 미지정 |
| GET /api/v1/requests/{req_id}/combinations | (미매핑) | FAIL | Controller 미지정 |
| GET /api/v1/requests/{req_id}/audit-log | (미매핑) | FAIL | Controller 미지정 |
| GET /api/v1/references/law-versions | (미매핑) | FAIL | Controller 미지정 |

### 5.2 Module Dependency Verification

| Dependency | Design SS2.3 | Validation | Match |
|-----------|-------------|-----------|:-----:|
| M1 -> (없음) | PASS | 독립 입력 모듈 | PASS |
| M2 -> RF_* | PASS | 기준정보 참조 | PASS |
| M3 -> M1, M2, MX-03 | PASS | 입력+기준정보+검증엔진 | PASS |
| M4 -> M3, RF_* | PASS | 사전점검 결과 + 기준정보 | PASS |
| M5 -> M4, MX-01 | PASS | 개별산출 + 순환참조 | PASS |
| M6 -> M5 | PASS | 최적조합 기반 보고 | PASS |
| MX-01 -> M4, M5 | Warning | 순환 의존 설계 (의도적) | PASS* |
| MX-02 -> RF_S_LAW_VERSION | PASS | 법령 버전 참조 | PASS |
| MX-03 -> SV_* | PASS | 요약 데이터 검증 | PASS |

### 5.3 Error Code Completeness

| Code | HTTP | Category | Defined | Test Coverage |
|------|:----:|----------|:-------:|:------------:|
| REQ_001 | 400 | Input | PASS | Not specified |
| REQ_002 | 400 | Input | PASS | Not specified |
| REQ_003 | 404 | Input | PASS | Not specified |
| REQ_004 | 409 | Input | PASS | Not specified |
| DIS_001 | 422 | Disqualify | PASS | Not specified |
| DIS_002 | 422 | Disqualify | PASS | Not specified |
| DIS_003 | 422 | Disqualify | PASS | Not specified |
| VAL_001 | 422 | Validation | PASS | Not specified |
| VAL_002 | 422 | Validation | PASS | Not specified |
| CAL_001 | 500 | Calc | PASS | Not specified |
| CAL_002 | 500 | Calc | PASS | Not specified |
| SYS_001 | 500 | System | PASS | Not specified |
| SYS_002 | 500 | System | PASS | Not specified |
| **AUTH_xxx** | **401/403** | **Auth** | **Missing** | - |

### 5.4 Test Plan Coverage

| FR Category | Test Type | Planned | FR Coverage |
|------------|-----------|:-------:|:-----------:|
| 계산 공식 (F01~F56) | Unit | 56+ | 100% of formulas |
| 14개 서브모듈 | Unit | 14x5=70 | 100% of 14 modules |
| 절사 정책 | Unit | 20+ | FR-42 covered |
| 71개 검증 규칙 | Integration | 71 | FR-16 covered |
| API 엔드포인트 | Integration | 20+ | 9 endpoints |
| 순환참조 | Integration | 10+ | FR-41 covered |
| 법인 E2E | E2E | 10 | CORP flow |
| 개인 E2E | E2E | 10 | INC flow |

### 5.5 Issues Found

| # | Severity | Issue | Details |
|---|:--------:|-------|---------|
| I-01 | Warning | API-Controller 매핑 4개 누락 | credits, combinations, audit-log, law-versions 엔드포인트의 Controller 미지정 |
| I-02 | Warning | 인증/인가 Error Code 미정의 | JWT 인증 실패(401), 권한 부족(403)에 대한 에러 코드가 Error Code System에 없음 |
| I-03 | Warning | API Response Format 불완전 | Plan Phase 4 표준 `{ data, meta? }` / `{ error: { code, message, details? } }` 중 Success format이 비표준. 현재 응답은 flat structure |
| I-04 | Info | Test 시나리오 건수 부족 | 상호배제 3건, 순환참조 3건, 세목분기 4건으로 edge case 부족. Should Have FR 관련 테스트 미계획 |
| I-05 | Warning | M7 모듈 설계 전무 | AmendmentClaimService.runSimulation() 시그니처만 있고 M7 상세 설계 없음. FR-55 (Could) 항목이나 Architecture Diagram에는 M7 포함 |

---

## 6. External Reference (외부자료_v4_0.md) Alignment (70%)

### 6.1 Module Coverage

| Module (외부자료) | Submodules | Design Coverage | Rate |
|-------------------|:---------:|:---------------:|:----:|
| M1 (입력관리) | M1-01~M1-12 | M1 전체 참조 (API-01) | 90% |
| M2 (기준정보) | M2-01~M2-13 | M2 참조, RF_* 26개 | 85% |
| M3 (사전점검) | M3-00~M3-06 | M3-00~M3-04 설계 | 80% |
| M4 법인 (개별산출) | M4-01~M4-15, M42 | 9개 구현체 | 53% |
| M4 개인 (개별산출) | P4-01~P4-09 | 3개 구현체 | 33% |
| M5 (최적조합) | M5-01~M5-05 | M5-01~M5-05 전체 | 100% |
| M6 (보고) | M6-01~M6-04 | M6-01~M6-04 전체 | 100% |
| M7 (시뮬레이션) | M7-01~M7-03 | 미설계 | 0% |
| MX (공통) | MX-01~MX-05 | MX-01~MX-03 설계 | 60% |

### 6.2 Missing Submodules (외부자료 대비)

#### Critical Missing (법인 M4)

| ID | Name | FR Mapping | Priority |
|----|------|-----------|:--------:|
| M4-08 | 이월결손금 재검토 | FR-29 | Must |
| M4-10 | 결손금 소급공제 검토 | FR-31 | Should |
| M4-11 | 세무조정 재검토 | FR-32 | Should |
| M4-12 | 토지등 양도세 추가과세 | - | Should |
| M4-13 | 기업구조조정 과세이연 | - | Should |
| M4-14 | 연결납세 특화 점검 | - | Should |
| M4-15 | 재해손실세액공제 | - | v4.0r1 |
| M42 | 감가상각 시부인 이월 추인 | - | v4.0r1 |

#### Critical Missing (개인 P4)

| ID | Name | FR Mapping | Priority |
|----|------|-----------|:--------:|
| P4-01 | 소득공제 누락 점검/세율구간 최적화 | FR-10, FR-18 | Must |
| P4-02 | 감면 소득 배분 계산 | FR-21a, FR-22a | Must |
| P4-03 | 공동사업자 소득 배분 | FR-10 관련 | Should |
| P4-07 | 개인 외국납부세액공제 | FR-28 | Should |
| P4-08 | 착한 임대인 세액공제 | - | Should |
| P4-09 | 결손금 소급공제 [INC] | FR-31 | Should |

> **Note**: P4-02 (감면 소득 배분)는 Design의 `IncomeAllocationService`로 커버되나, 독립 모듈로 식별되지 않음. P4-07은 `M4_28_ForeignTaxCredit_Indv`로 커버됨.

#### Missing MX Modules

| ID | Name | Design Status |
|----|------|:------------:|
| MX-04 | 계산 이력/감사 추적 | AL_S_CALCULATION 참조만. 별도 서비스 미설계 |
| MX-05 | 절사 정책 | TruncateUtil 파일 구조에 존재하나 상세 미설계 |

### 6.3 RF_* Table Count Discrepancy

| Source | RF_* Count | Notes |
|--------|:---------:|-------|
| 외부자료 v4_0 (SS2.4 통계) | 22 | "기준정보 22" 표기 |
| 외부자료 v4_0 (SS3.3 총괄) | **26** | C:7, I:4, S:15 상세 열거 |
| 외부자료 v4_0 (SS5.1 인벤토리) | **26** | 상세 테이블명 나열 |
| Schema (schema.md) | **26** | 상세 컬럼 정의 포함 |
| Design | **26** | "RF_* (26개 기준정보)" 표기 |

> 외부자료 내부 불일치: SS2.4 통계표(22)와 SS3.3/SS5.1 상세 목록(26)이 상충. **schema.md(26개)가 정본**.

### 6.4 Key Design Principles Alignment

| Principle (외부자료) | Design Reflection | Match |
|---------------------|------------------|:-----:|
| Request-Driven (req_id 관통) | PASS - 전체 아키텍처 반영 | PASS |
| 원본 불변성 (INSERT ONLY) | PASS - DDL 트리거 포함 | PASS |
| 3계층 파이프라인 | PASS - Data Flow 명시 | PASS |
| 결과 이중보관 (RDB + JSON) | PASS - CO_* + CO_S_REPORT_JSON | PASS |
| 절사 원칙 (TRUNCATE) | PASS - MoneyAmount/TaxRate | PASS |
| 반올림 금지 | PASS - "ROUND 메서드 없음 (의도적 배제)" | PASS |
| 상호배제 (SS127-4) | PASS - M5-01 상세 설계 | PASS |
| 순환참조 5회 수렴 | PASS - MX-01 상세 구현 | PASS |
| 세목 분기 (TAX_TYPE) | PASS - Strategy Pattern | PASS |
| 보고서 3종 분리 | PASS - API-04 type param | PASS |

---

## 7. Checklist Results Summary

### 7.1 Required Section Check

| Section | Status | Notes |
|---------|:------:|-------|
| Overview (Purpose, Scope, References) | PASS | SS1 완비 |
| Architecture (Component Diagram, Data Flow) | PASS | SS2 Clean Architecture + 3계층 파이프라인 |
| Data Model (Entity, Relationship, DDL) | PASS | SS3 Entity + DDL (4개) |
| API Specification (Endpoints, Req/Res) | PASS | SS4 9개 엔드포인트 + 3개 상세 |
| Module Design (M3~M6) | PASS | SS5 Java 코드 수준 설계 |
| Error Handling (Codes, Format) | PASS | SS6 13개 코드 + JSON format |
| Security Considerations | PASS | SS7 9개 항목 |
| Test Plan (Scenarios, Criteria) | PASS | SS8 유형별 테스트 계획 |
| Clean Architecture (Layers, Rules) | PASS | SS9 4계층 + Import Rules |
| Coding Convention | PASS | SS10 Naming + 금액처리 + 분기패턴 |
| Implementation Guide | PASS | SS11 파일구조 + 6주 일정 |
| **M7 Simulation Design** | **FAIL** | **아키텍처에 포함되나 상세 설계 없음** |
| **Pagination/Response Standard** | **FAIL** | **Success 응답 `{ data, meta? }` 형식 미준수** |
| **Environment Variables** | **FAIL** | **DB/JWT/API 관련 환경변수 정의 없음** |

### 7.2 Consistency Check

| Check | Status | Notes |
|-------|:------:|-------|
| Term consistency (Glossary-based) | PASS | 핵심 30+ 용어 일관 사용 |
| Data type consistency | PASS | BIGINT(금액), VARCHAR(코드), TIMESTAMP(일시) 일관 |
| Naming convention consistency | PASS | camelCase(Java)/lower_snake(DB)/kebab(API) 일관 |
| RESTful compliance | Warning | PUT /calculate는 side-effect 있는 action - POST가 더 적합할 수 있음 |
| Response format standard | FAIL | `{ data, meta? }` / `{ error: { code, message, details? } }` 표준 미적용. Error만 표준 준수 |
| Clean Architecture layers | PASS | 4계층 정의 + Import Rules 명시 |
| Dependency direction | PASS | Presentation -> Application -> Domain <- Infrastructure |

---

## 8. Issues Summary

### 8.1 Critical Issues (Implementation Blocking)

| # | Issue | Impact | Recommended Action |
|---|-------|--------|-------------------|
| C-01 | req_id 접두어 불일치 (C/I vs C/P) | 원본 설계서와 Plan 문서군 간 충돌. 구현 시 어느 규칙을 따를지 불명확 | 프로젝트 이해관계자와 확정. Glossary/Schema를 기준으로 C/I 채택 시 외부자료 정정 필요 |
| C-02 | P4-01 (소득공제 누락 점검) 설계 전무 | 개인사업자 필수 기능(Must). 과세표준 재산정 -> 세율구간 변동 확인 -> 전체 감면/공제 전략 재수립의 핵심 로직 누락 | P4-01~P4-03 상세 설계 추가 필요 |
| C-03 | CANCELLED 상태 + 취소 API 미설계 | Domain Model에 정의된 상태. 사용자 취소 시나리오 처리 불가 | RequestStatus에 CANCELLED 추가 + DELETE/PATCH API 설계 |

### 8.2 Warning Issues (Improvement Needed)

| # | Issue | Impact | Recommended Action |
|---|-------|--------|-------------------|
| W-01 | M4 서브모듈 14개 중 상세 설계 부족 | 구현체 클래스명만 존재, 비즈니스 로직 미기술 (M4-08, M4-09, M4-25, M4-29, M4-31) | 최소 Must Priority 항목의 입력/산출/공식 상세 추가 |
| W-02 | M7 시뮬레이션 모듈 미설계 | Architecture Diagram에 포함되었으나 상세 없음 (Could Priority) | Phase 2 범위 명시 또는 아키텍처에서 제거 |
| W-03 | API Response Format 비표준 | Success 응답이 `{ data, meta? }` 래핑 없이 flat structure | 표준 응답 래퍼 적용 권고 |
| W-04 | Auth Error Code 미정의 | JWT 인증 실패, 권한 부족 에러 코드 없음 | AUTH_001(401), AUTH_002(403) 추가 |
| W-05 | 환경변수 정의 없음 | DB URL, JWT Secret, API Key 등 환경변수 미정의 | Phase 2 Convention 또는 별도 환경변수 명세 추가 |
| W-06 | DDL 4개만 제공 | 83개 중 4개. 나머지는 schema.md 참조해야 함 | 핵심 테이블(SV_S_BASIC, CO_S_COMBINATION 등) DDL 추가 |
| W-07 | 외부자료 M4 확장 모듈 미반영 | M4-12~M4-15, M42, P4-08 등 v4.0r1 신규 항목 미설계 | Phase 2 범위 정리 또는 Design에 명시적 제외 기록 |
| W-08 | Report JSON section_g 누락 | 외부자료에 section_g_meta 정의 있으나 Design에서 A~F 6섹션만 언급 (7섹션 표기와 불일치) | section_g(meta) 추가 또는 표기 수정 |
| W-09 | Controller 매핑 4개 미완 | credits, combinations, audit-log, law-versions API의 담당 Controller 미지정 | Controller 할당 명시 |

### 8.3 Info Issues (Reference)

| # | Issue | Notes |
|---|-------|-------|
| N-01 | PUT /calculate vs POST /calculate | RESTful 원칙상 side-effect가 있는 계산 실행은 POST가 더 적합. 다만 멱등성(동일 결과) 관점에서 PUT도 가능 |
| N-02 | 외부자료 내부 RF_* 카운트 불일치 (22 vs 26) | 외부자료 자체 오류. Schema.md 26개가 정본 |
| N-03 | M4 서브모듈 넘버링 불연속 | M4_25, M4_28, M4_29, M4_31 등 번호가 외부자료 FR 번호와 혼용. M4-03(사보)이 M4_25로 매핑되는 등 혼란 가능 |
| N-04 | 개인사업자 전용 모듈 접두어 | 외부자료는 P4-xx, Design은 P4_04처럼 혼용. 일관된 네이밍 필요 |

---

## 9. Recommendations

### 9.1 Urgent (Design 승인 전 필수)

1. **req_id 접두어 확정**: C/I(현 Plan 기준) vs C/P(외부자료 기준) 중 확정하고 전 문서 일치시킬 것
2. **P4-01~P4-03 설계 추가**: 개인사업자 핵심 모듈(소득공제 누락 점검, 감면소득배분, 공동사업자)의 상세 설계 필수
3. **CANCELLED 상태 및 취소 API 설계**: Domain Model과 일치시키기 위해 RequestStatus에 CANCELLED 추가

### 9.2 High Priority (구현 착수 전 권장)

4. **M4 Must Priority 서브모듈 상세 설계**: FR-29(이월결손금), FR-33a(최저한세 비대상) 등 Must 항목의 비즈니스 로직 상세화
5. **API Response Format 표준화**: Success 응답에 `{ data: {...}, meta: {...} }` 래퍼 적용
6. **Auth Error Code 추가**: AUTH_001(인증 실패), AUTH_002(권한 부족) 코드 정의
7. **Controller 매핑 완성**: 4개 미매핑 엔드포인트에 Controller 할당

### 9.3 Medium Priority (구현 중 보완 가능)

8. **M7 모듈 Phase 구분 명시**: M7을 Phase 2로 명확히 분류하거나, 아키텍처 다이어그램에 Phase 표기 추가
9. **Report JSON section_g 정리**: 6섹션 vs 7섹션 표기 통일
10. **핵심 DDL 추가**: SV_S_BASIC, SV_S_ELIGIBILITY, CO_S_COMBINATION 등 10개 테이블 DDL 보완
11. **환경변수 명세 작성**: SPRING_DATASOURCE_URL, JWT_SECRET 등 Phase 9 배포 대비

### 9.4 Low Priority (Backlog)

12. M4 서브모듈 넘버링 정리 (외부자료 ID vs 내부 클래스명 매핑표)
13. 외부자료 v4_0.md 내부 RF_* 카운트 오류(22 vs 26) 정정
14. 개인사업자 전용 모듈 접두어 통일 (P4-xx vs P4_xx)

---

## 10. Validation Verdict

```
Completeness Score: 78/100

Validation Score >= 70 && < 90:
  -> Implementation possible after improving Warning items.
  -> Critical items (C-01, C-02, C-03) must be resolved before implementation.
```

### Action Required

| Priority | Action | Estimated Effort |
|----------|--------|:----------------:|
| **Before Implementation** | C-01~C-03 해결 | 1~2일 |
| **Before Sprint 1** | W-01~W-04 개선 | 2~3일 |
| **During Implementation** | W-05~W-09 보완 | 지속적 |
| **Phase 2 Planning** | N-01~N-04 정리 | Backlog |

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-02-17 | 초기 검증 - 7개 카테고리, 78점, Critical 3건, Warning 9건, Info 4건 |
