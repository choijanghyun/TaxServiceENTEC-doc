# 경정청구 환급액 계산 시스템 데이터 스키마 (Schema)

> **Project**: TaxServiceENTEC
> **Version**: 1.0
> **Date**: 2026-02-17
> **Phase**: Phase 1 - Schema/Terminology Definition
> **Reference**: 외부자료_v4_0.md (통합개발상세설계서 v4.0)

---

## 1. 테이블 명명규칙

### 1.1 명명 형식

```
{주제영역}_{활용구분}_{테이블명}
```

| 구성 요소 | 코드 | 의미 | 예시 |
|----------|------|------|------|
| **주제영역** (2자) | `RI` | Request & Input (요청 및 입력) | `RI_S_REQUEST` |
| | `SV` | Summary & Validation (요약 및 검증) | `SV_S_BASIC` |
| | `CO` | Calculation Output (산출 결과) | `CO_S_REFUND` |
| | `AL` | Audit Log (감사 로그) | `AL_S_CALCULATION` |
| | `RF` | Reference (기준 정보) | `RF_S_TAX_RATE` |
| **활용구분** (1자) | `C` | Corporate (법인 전용) | `RI_C_CORP_BASIC` |
| | `I` | Individual (개인 전용) | `RI_I_BASIC` |
| | `S` | Shared (공동 활용) | `RI_S_RAW_DATA` |
| **테이블명** | — | UPPER_SNAKE_CASE | `EMPLOYEE_DETAIL` |

### 1.2 뷰 명명규칙

```
VW_{활용구분}_{뷰명}
```

예: `VW_C_RD_5Y_BASE` (법인 전용 R&D 5개년 집계 뷰)

---

## 2. 주제영역별 테이블 총괄 (5영역, 83개 + 뷰 1개)

| 주제영역 | 접두어 | 설명 | 테이블 수 | C | I | S |
|----------|--------|------|:---------:|:-:|:-:|:-:|
| **① 요청 및 입력** | `RI_` | 요청 마스터, 원시 JSON, 입력 스키마 | 37 | 17 | 8 | 12 |
| **② 요약 및 검증** | `SV_` | 계산용 요약 + 점검/검증 결과 | 8 | - | - | 8 |
| **③ 산출 결과** | `CO_` | 공제감면, 조합 최적화, 환급액, 보고서 | 11 | - | 3 | 8 |
| **④ 감사 로그** | `AL_` | 전 단계 계산 이력 추적 | 1 | - | - | 1 |
| **⑤ 기준 정보** | `RF_` | 세율·공제율·법령 (req_id 없음) | 26 | 7 | 4 | 15 |
| **합계** | | | **83** | **24** | **15** | **44** |

---

## 3. 주제영역 ① 요청 및 입력 상세 (RI_) — 37개

### 3.1 공동 활용 (S) — 12개

| No | 테이블명 | 한글명 | PK | 역할 |
|----|---------|--------|-----|------|
| 1 | `RI_S_REQUEST` | 요청 마스터 | `req_id` | 전체 트랜잭션 기준점. 모든 비RF 테이블의 FK 원천 |
| 2 | `RI_S_APPLICANT` | 신청인 정보 | `applicant_id` | 세무대리인/직접신청 구분 |
| 3 | `RI_S_TAXPAYER` | 납세자 정보 | `taxpayer_id` | 법인/개인사업자 기본정보 |
| 4 | `RI_S_RAW_DATA` | 원시 JSON 보관 | `req_id` | **INSERT ONLY** 원본 불변 |
| 5 | `RI_S_EMPLOYEE_DETAIL` | 근로자 상세 | `req_id, emp_seq` | 개별 근로자 정보 |
| 6 | `RI_S_EMPLOYEE_MONTHLY` | 근로자 월별 | `req_id, emp_seq, month` | 월별 근무시간·급여 |
| 7 | `RI_S_STARTUP_INFO` | 창업 정보 | `req_id` | 창업감면(§6) 기초데이터 |
| 8 | `RI_S_SME_SPECIAL` | 중소기업 감면정보 | `req_id` | 중소감면(§7) 기초데이터 |
| 9 | `RI_S_FOREIGN_TAX` | 외국납부세액 | `req_id, seq` | 외국납부세액공제 기초 |
| 10 | `RI_S_LOSS_CARRYFORWARD` | 이월결손금 | `req_id, year` | 연도별 이월결손금 명세 |
| 11 | `RI_S_CREDIT_CARRYFORWARD` | 이월세액공제 | `req_id, provision, year` | 이월된 미공제 세액 명세 |
| 12 | `RI_S_EXISTING_DEDUCTION` | 기존 신고 공제 | `req_id` | 기신고 적용 공제·감면 내역 |

### 3.2 법인 전용 (C) — 17개

| No | 테이블명 | 한글명 | PK | 주요 컬럼 수 |
|----|---------|--------|-----|:----------:|
| 1 | `RI_C_CORP_BASIC` | 법인 기본정보 | `req_id` | 53 |
| 2 | `RI_C_REPRESENTATIVE` | 대표자 정보 | `req_id` | - |
| 3 | `RI_C_BRANCH_LOCATION` | 지점 소재지 | `req_id, branch_seq` | - |
| 4 | `RI_C_CONSOLIDATED_SUB` | 연결납세 자회사 | `req_id, sub_seq` | - |
| 5 | `RI_C_INVESTMENT_ASSET` | 투자자산 명세 | `req_id, asset_seq` | - |
| 6 | `RI_C_BUSINESS_VEHICLE` | 업무용 차량 | `req_id, vehicle_seq` | - |
| 7 | `RI_C_NON_BUSINESS_ASSET` | 비사업용 자산 | `req_id, asset_seq` | - |
| 8 | `RI_C_DEPRECIATION_ADJUST` | 감가상각 조정 | `req_id, asset_seq` | - |
| 9 | `RI_C_RD_EXPENSE` | R&D 비용 | `req_id` | - |
| 10 | `RI_C_TAX_ADJUSTMENT` | 세무조정 | `req_id, adj_seq` | - |
| 11 | `RI_C_ENTERTAINMENT` | 접대비 | `req_id` | - |
| 12 | `RI_C_SHAREHOLDER_LOAN` | 주주 차입금 | `req_id` | - |
| 13 | `RI_C_GOVERNMENT_SUBSIDY` | 정부보조금 | `req_id, sub_seq` | - |
| 14 | `RI_C_DIVIDEND_INCOME` | 수입배당금 | `req_id, div_seq` | - |
| 15 | `RI_C_INTERIM_TAX` | 중간예납 | `req_id` | - |
| 16 | `RI_C_DISASTER_LOSS` | 재해손실 | `req_id` | - |
| 17 | `RI_C_SOCIAL_INSURANCE` | 사회보험료 | `req_id` | - |

### 3.3 개인 전용 (I) — 8개

| No | 테이블명 | 한글명 | PK | 주요 컬럼 수 |
|----|---------|--------|-----|:----------:|
| 1 | `RI_I_BASIC` | 개인 기본정보 | `req_id` | 31 |
| 2 | `RI_I_BUSINESS` | 사업장별 상세 | `req_id, biz_id` | 23 |
| 3 | `RI_I_OTHER_INCOME` | 합산소득 | `req_id` | 10 |
| 4 | `RI_I_DEDUCTION` | 소득공제 항목 | `req_id` | 18 |
| 5 | `RI_I_SINCERITY` | 성실신고확인 | `req_id` | 7 |
| 6 | `RI_I_FOREIGN_TAX` | 외국납부세액 | `req_id, seq` | 7 |
| 7 | `RI_I_RENTAL_REDUCTION` | 착한 임대인 | `req_id, seq` | 9 |
| 8 | `RI_I_JOINT_BIZ` | 공동사업자 | `req_id, joint_seq` | 5 |

---

## 4. 주제영역 ② 요약 및 검증 (SV_) — 8개 (전체 S)

### 4.1 계산용 요약 (4개)

| No | 테이블명 | 한글명 | 생성 단계 | 역할 |
|----|---------|--------|----------|------|
| 1 | `SV_S_BASIC` | 납세자 기본정보 요약 | M1 파싱 | TAX_TYPE 분기, 핵심 판단값 |
| 2 | `SV_S_EMPLOYEE` | 고용 정보 요약 | M1 파싱 | 상시근로자 집계, 증감 산정 |
| 3 | `SV_S_DEDUCTION` | 공제·감면 기초 요약 | M1 파싱 | 기존 적용·미적용 현황 |
| 4 | `SV_S_FINANCIAL` | 재무·세무 수치 요약 | M1 파싱 | 과세표준·산출세액·납부세액 |

### 4.2 점검·검증 결과 (4개)

| No | 테이블명 | 한글명 | 생성 단계 | 역할 |
|----|---------|--------|----------|------|
| 5 | `SV_S_PREP_SUMMARY` | 사전 데이터 준비 요약 | M3 | 사전점검 통과 여부 |
| 6 | `SV_S_ELIGIBILITY` | 자격 진단 결과 | M3 | 중소기업·소재지·기한 판정 |
| 7 | `SV_S_INSPECTION_LOG` | 점검항목별 판정 | M3~M6 | 전체 점검항목 판정 이력 |
| 8 | `SV_S_VALIDATION_LOG` | 검증 규칙 실행 결과 | MX-03 | 71개 규칙 Pass/Fail 결과 |

---

## 5. 주제영역 ③ 산출 결과 (CO_) — 11개

### 5.1 공동 활용 (S) — 8개

| No | 테이블명 | 한글명 | 생성 모듈 | 역할 |
|----|---------|--------|----------|------|
| 1 | `CO_S_EMPLOYEE_SUMMARY` | 상시근로자 산정 | M4 | 산정 결과·제외자 명세 |
| 2 | `CO_S_CREDIT_DETAIL` | 개별 공제·감면 | M4 | 14개 항목별 산출 결과 |
| 3 | `CO_S_COMBINATION` | 조합 비교·최적 선택 | M5 | 그룹A/B 비교, 최적 선택 |
| 4 | `CO_S_EXCLUSION_VERIFY` | 상호배제 검증 | M5 | §127④ 위반 여부 |
| 5 | `CO_S_REFUND` | 최종 환급액 산출 | M6 | 기존 vs 경정 후 비교 |
| 6 | `CO_S_RISK` | 사후관리·리스크 | M6 | 고용유지·자산처분 리스크 |
| 7 | `CO_S_ADDITIONAL_CHECK` | 추가 확인 필요 | M6 | 인적 확인 필요 항목 |
| 8 | `CO_S_REPORT_JSON` | 보고서 JSON | M6 | 7섹션(A~G) 직렬화 |

### 5.2 개인 전용 (I) — 3개

| No | 테이블명 | 한글명 | 생성 모듈 | 역할 |
|----|---------|--------|----------|------|
| 9 | `CO_I_INCOME_ALLOCATION` | 소득구분별 배분 | M4 | 감면소득 배분 산출 |
| 10 | `CO_I_DEDUCTION_IMPACT` | 소득공제 영향 분석 | M4 | 소득공제별 세액 영향도 |
| 11 | `CO_I_LOCAL_TAX` | 지방소득세 | M6 | 지방소득세 10% 산출 |

---

## 6. 주제영역 ④ 감사 로그 (AL_) — 1개

| No | 테이블명 | 한글명 | PK | 역할 |
|----|---------|--------|-----|------|
| 1 | `AL_S_CALCULATION` | 감사추적 로그 | `req_id, log_seq` | 전 단계 계산 파라미터·결과·타임스탬프 기록 (APPEND ONLY) |

---

## 7. 주제영역 ⑤ 기준 정보 (RF_) — 26개

> **특징**: req_id 없이 독립 관리. 법령·세율 등 공통 기준.

### 7.1 공동 활용 (S) — 15개

| No | 테이블명 | 한글명 | 핵심 컬럼 |
|----|---------|--------|----------|
| 1 | `RF_S_TAX_RATE` | 통합 세율 | year_from, bracket_min/max, tax_rate |
| 2 | `RF_S_MIN_TAX_RATE` | 최저한세율 | corp_size, bracket_min/max, min_rate |
| 3 | `RF_S_EMPLOYMENT_CREDIT` | 통합고용 공제 기준 | corp_size, worker_type, amount |
| 4 | `RF_S_SME_DEDUCTION_RATE` | 중소기업 감면율 | industry, capital_zone, rate |
| 5 | `RF_S_STARTUP_DEDUCTION_RATE` | 창업감면 감면율 | capital_zone, depopulation, rate |
| 6 | `RF_S_MUTUAL_EXCLUSION` | 상호배제 매트릭스 | provision_a, provision_b, is_exclusive |
| 7 | `RF_S_NONGTEUKSE` | 농특세 적용 기준 | provision, is_exempt, rate |
| 8 | `RF_S_CAPITAL_ZONE` | 수도권 지역 구분 | region_code, zone_type |
| 9 | `RF_S_DEPOPULATION_AREA` | 인구감소지역 목록 | region_code, effective_from |
| 10 | `RF_S_INDUSTRY_ELIGIBILITY` | 업종별 감면 적격성 | industry_code, provision, eligible |
| 11 | `RF_S_KSIC_CODE` | 한국표준산업분류 | ksic_code, name, category |
| 12 | `RF_S_REFUND_INTEREST_RATE` | 환급가산금 이율 | effective_from, annual_rate |
| 13 | `RF_S_EXCHANGE_RATE` | 외국환 매매기준율 | currency, date, rate |
| 14 | `RF_S_SYSTEM_PARAM` | 시스템 파라미터 | param_key, param_value |
| 15 | `RF_S_LAW_VERSION` | 법령 버전 관리 | law_code, effective_from, version |

### 7.2 법인 전용 (C) — 7개

| No | 테이블명 | 한글명 | 핵심 컬럼 |
|----|---------|--------|----------|
| 1 | `RF_C_TAX_RATE_HISTORY` | 법인세율 이력 | year, bracket_min/max, rate |
| 2 | `RF_C_INVESTMENT_CREDIT_RATE` | 투자세액공제율 | asset_type, corp_size, rate |
| 3 | `RF_C_RD_CREDIT_RATE` | R&D 공제율 | rd_type, corp_size, rate |
| 4 | `RF_C_RD_MIN_TAX_EXEMPT` | R&D 최저한세 배제율 | rd_category, corp_size, exempt_rate |
| 5 | `RF_C_ENTERTAINMENT_LIMIT` | 접대비 한도 | revenue_bracket, limit_amount |
| 6 | `RF_C_DEEMED_INTEREST_RATE` | 인정이자율 | year, rate |
| 7 | `RF_C_DIVIDEND_EXCLUSION` | 배당익금불산입률 | ownership_ratio, exclusion_rate |

### 7.3 개인 전용 (I) — 4개

| No | 테이블명 | 한글명 | 핵심 컬럼 |
|----|---------|--------|----------|
| 1 | `RF_I_TAX_RATE` | 종합소득세율 | effective_from, bracket_no, rate |
| 2 | `RF_I_MIN_TAX` | 개인 최저한세율 | threshold, rate_below, rate_above |
| 3 | `RF_I_DEDUCTION_LIMIT` | 소득공제 한도 | deduction_type, income_bracket, limit |
| 4 | `RF_I_SINCERITY_THRESHOLD` | 성실신고 기준금액 | industry_group, revenue_threshold |

---

## 8. 핵심 테이블 컬럼 상세

### 8.1 RI_S_REQUEST (요청 마스터)

| 컬럼명 | 한글명 | 타입 | 필수 | 설명 |
|--------|--------|------|:----:|------|
| `req_id` | 요청번호 | VARCHAR(30) | **PK** | `{C/I}-{사업자번호}-{YYYYMMDD}-{순번}` |
| `applicant_id` | 신청인ID | VARCHAR(20) | FK | → RI_S_APPLICANT |
| `taxpayer_id` | 납세자ID | VARCHAR(20) | FK | → RI_S_TAXPAYER |
| `tax_type` | 세목구분 | VARCHAR(4) | Y | `CORP` / `INC` |
| `request_date` | 요청일자 | TIMESTAMP | Y | 입력자료 요청 시각 |
| `request_status` | 요청상태 | VARCHAR(20) | Y | `RECEIVED`/`PROCESSING`/`COMPLETED`/`ERROR` |
| `request_source` | 요청원천 | VARCHAR(20) | Y | `WEB`/`API`/`BATCH` |
| `requested_by` | 요청자 | VARCHAR(50) | Y | 사용자 식별 |
| `client_ip` | 요청자IP | VARCHAR(45) | N | 감사 대응 |
| `created_at` | 생성일시 | TIMESTAMP | Y | 자동 |
| `updated_at` | 수정일시 | TIMESTAMP | Y | 자동 |

### 8.2 RI_C_CORP_BASIC (법인 기본정보) — 주요 53개 컬럼

| 컬럼명 | 한글명 | 타입 | 필수 | 설명 |
|--------|--------|------|:----:|------|
| `req_id` | 요청번호 | VARCHAR(30) | PK/FK | → RI_S_REQUEST |
| `corp_size` | 기업규모 | VARCHAR(10) | Y | 소기업/중기업/중견/대기업 |
| `industry_code` | 업종코드 | VARCHAR(10) | Y | KSIC 코드 |
| `hq_location` | 본점소재지 | VARCHAR(200) | Y | |
| `capital_zone` | 수도권구분 | VARCHAR(20) | Y | 과밀억제/성장관리/자연보전/비수도권 |
| `depopulation_area` | 인구감소지역 | BOOLEAN | N | |
| `target_tax_year` | 대상과세연도 | VARCHAR(4) | Y | |
| `fiscal_start` | 사업연도시작일 | DATE | Y | |
| `fiscal_end` | 사업연도종료일 | DATE | Y | |
| `revenue` | 매출액 | BIGINT | Y | 원 단위 |
| `biz_income` | 각사업연도소득 | BIGINT | Y | 익금총액 - 손금총액 |
| `taxable_income` | 과세표준 | BIGINT | Y | 기신고 |
| `computed_tax` | 산출세액 | BIGINT | Y | |
| `paid_tax` | 납부세액 | BIGINT | Y | |
| `founding_date` | 창업일자 | DATE | N | 법인설립등기일 |
| `loss_carryforward` | 이월결손금잔액 | BIGINT | N | |
| `graduation_year` | 중소기업졸업연도 | VARCHAR(4) | N | 졸업유예 판단 |
| `claim_reason` | 경정청구사유 | VARCHAR(200) | Y | |

### 8.3 RI_I_BASIC (개인 기본정보) — 주요 31개 컬럼

| 컬럼명 | 한글명 | 타입 | 필수 | 설명 |
|--------|--------|------|:----:|------|
| `req_id` | 요청번호 | VARCHAR(30) | PK/FK | → RI_S_REQUEST |
| `taxpayer_name` | 사업자명 | VARCHAR(50) | Y | 대표자 성명 |
| `birth_date` | 생년월일 | DATE | Y | 청년 판단 |
| `gender` | 성별 | VARCHAR(2) | Y | 병역가산 |
| `military_months` | 복무개월수 | INT | N | 최대 72개월(6년) |
| `biz_scale` | 기업규모 | VARCHAR(10) | Y | SMALL/MEDIUM |
| `biz_count` | 사업장수 | INT | Y | 1 이상 |
| `is_joint_biz` | 공동사업자 | BOOLEAN | Y | |
| `bookkeeping_type` | 기장방식 | VARCHAR(20) | Y | DOUBLE/SIMPLE/ESTIMATED |
| `is_sincerity_target` | 성실신고대상 | BOOLEAN | Y | |
| `claim_tax_year` | 대상과세연도 | VARCHAR(4) | Y | |
| `computed_tax` | 산출세액 | BIGINT | Y | |
| `taxable_base` | 과세표준 | BIGINT | Y | |
| `paid_tax_total` | 총납부세액 | BIGINT | Y | |
| `next_year_loss_yn` | 차기결손여부 | BOOLEAN | N | 소급공제 검토 |

### 8.4 RI_I_BUSINESS (사업장별 상세) — 23개 컬럼

| 컬럼명 | 한글명 | 타입 | 필수 | 설명 |
|--------|--------|------|:----:|------|
| `req_id` | 요청번호 | VARCHAR(30) | PK/FK | |
| `biz_id` | 사업장ID | VARCHAR(20) | PK | 복합키 |
| `industry_code` | 업종코드 | VARCHAR(10) | Y | KSIC |
| `address` | 소재지 | VARCHAR(200) | Y | |
| `capital_zone` | 수도권구분 | VARCHAR(20) | Y | |
| `is_depopulation` | 인구감소지역 | BOOLEAN | N | |
| `revenue` | 수입금액 | BIGINT | Y | |
| `biz_income` | 사업소득금액 | BIGINT | Y | |
| `emp_prior_total` | 직전상시근로자 | DECIMAL(10,2) | N | |
| `emp_current_total` | 해당상시근로자 | DECIMAL(10,2) | N | |
| `founding_date` | 창업일 | DATE | N | |
| `is_exempt_eligible` | 감면대상업종 | BOOLEAN | N | |
| `exemption_rate_7` | §7 감면율 | DECIMAL(5,2) | N | |

### 8.5 CO_S_CREDIT_DETAIL (개별 공제·감면 산출)

| 컬럼명 | 한글명 | 타입 | 필수 | 설명 |
|--------|--------|------|:----:|------|
| `req_id` | 요청번호 | VARCHAR(30) | PK/FK | |
| `item_id` | 항목ID | VARCHAR(20) | PK | `ITEM-06`, `ITEM-29-8` 등 |
| `item_name` | 항목명 | VARCHAR(100) | Y | |
| `provision` | 근거조문 | VARCHAR(20) | Y | `§6`, `§7`, `§24` 등 |
| `credit_type` | 유형 | VARCHAR(10) | Y | `EXEMPTION`/`CREDIT` |
| `item_status` | 적용상태 | VARCHAR(20) | Y | `applicable`/`not_applicable`/`needs_review` |
| `gross_amount` | 공제·감면액 | BIGINT | N | 농특세 차감 전 |
| `nongteuk_exempt` | 농특세비과세 | BOOLEAN | Y | |
| `nongteuk_amount` | 농특세액 | BIGINT | N | 과세 시 공제액×20% |
| `net_amount` | 실질공제액 | BIGINT | N | = gross - nongteuk |
| `min_tax_subject` | 최저한세대상 | BOOLEAN | Y | |
| `is_carryforward` | 이월가능여부 | BOOLEAN | Y | |
| `legal_basis` | 법령근거 | VARCHAR(100) | Y | |

### 8.6 CO_S_REFUND (최종 환급액 산출)

| 컬럼명 | 한글명 | 타입 | 필수 | 설명 |
|--------|--------|------|:----:|------|
| `req_id` | 요청번호 | VARCHAR(30) | PK/FK | |
| `original_computed_tax` | 기존산출세액 | BIGINT | Y | |
| `original_determined_tax` | 기존결정세액 | BIGINT | Y | |
| `original_paid_tax` | 기존납부세액 | BIGINT | Y | |
| `amended_computed_tax` | 경정산출세액 | BIGINT | Y | |
| `amended_deductions` | 경정공제감면합계 | BIGINT | Y | |
| `min_tax_adj` | 최저한세조정 | BIGINT | N | |
| `amended_determined_tax` | 경정결정세액 | BIGINT | Y | |
| `refund_amount` | 환급세액 | BIGINT | Y | = 기존 - 경정 |
| `interest_amount` | 환급가산금 | BIGINT | N | |
| `local_tax_refund` | 지방소득세환급 | BIGINT | N | 10% |
| `total_expected` | 총예상환급 | BIGINT | Y | 본세+가산금+지방세 |
| `combo_id` | 적용조합ID | INT | FK | → CO_S_COMBINATION |

---

## 9. RF_* 기준정보 상세

### 9.1 RF_S_TAX_RATE (법인세율)

| 컬럼명 | 타입 | 설명 |
|--------|------|------|
| `rate_id` | INT | PK |
| `year_from` | VARCHAR(4) | 적용시작연도 |
| `year_to` | VARCHAR(4) | 적용종료연도 |
| `bracket_min` | BIGINT | 과세표준 하한 (원) |
| `bracket_max` | BIGINT | 과세표준 상한 (NULL=무한) |
| `tax_rate` | DECIMAL(5,2) | 세율(%) |
| `progressive_deduction` | BIGINT | 누진공제 (원) |

**초기 데이터 (2023~2025)**:

| 구간 | 하한 | 상한 | 세율 |
|------|------|------|:----:|
| 1 | 0 | 2억 | 9% |
| 2 | 2억 | 200억 | 19% |
| 3 | 200억 | 3,000억 | 21% |
| 4 | 3,000억 | - | 24% |

### 9.2 RF_I_TAX_RATE (종합소득세율)

**초기 데이터 (2023~)**:

| 구간 | 하한 | 상한 | 세율 | 누진공제 |
|:----:|------|------|:----:|---------|
| 1 | 0 | 1,400만 | 6% | 0 |
| 2 | 1,400만 | 5,000만 | 15% | 126만 |
| 3 | 5,000만 | 8,800만 | 24% | 576만 |
| 4 | 8,800만 | 1.5억 | 35% | 1,544만 |
| 5 | 1.5억 | 3억 | 38% | 1,994만 |
| 6 | 3억 | 5억 | 40% | 2,594만 |
| 7 | 5억 | 10억 | 42% | 3,594만 |
| 8 | 10억 | - | 45% | 6,594만 |

### 9.3 RF_S_MIN_TAX_RATE (최저한세율)

| 구분 | 과세표준 | 법인 | 개인 |
|------|---------|:----:|:----:|
| 중소기업 | 전 구간 | 7% | 산출세액×35% (3천만 초과분 45%) |
| 중견기업 | ~100억 | 10% | - |
| 중견기업 | 100~1,000억 | 12% | - |
| 대기업 | 1,000억~ | 17% | - |

---

## 10. JSON 전달 스키마 (7섹션)

### 10.1 최상위 구조

```json
{
  "report_meta": { "req_id", "tax_type", "tax_year", "analysis_status" },
  "section_a": { "taxpayer_info", "eligibility" },
  "section_b": { "candidate_items[]" },
  "section_c": { "optimal_combination", "group_comparison", "exclusion_matrix" },
  "section_d": { "original", "amended", "refund" },
  "section_e": { "inspection_log[]" },
  "section_f": { "post_management[]", "additional_checks[]" }
}
```

### 10.2 JSON ↔ 테이블 매핑

| JSON 섹션 | 소스 테이블 |
|----------|------------|
| `section_a.taxpayer_info` | SV_S_BASIC |
| `section_a.eligibility` | SV_S_ELIGIBILITY |
| `section_b.candidate_items[]` | CO_S_CREDIT_DETAIL |
| `section_c.optimal_combination` | CO_S_COMBINATION (rank=1) |
| `section_c.exclusion_matrix` | CO_S_EXCLUSION_VERIFY |
| `section_d.refund` | CO_S_REFUND |
| `section_e.inspection_log[]` | SV_S_INSPECTION_LOG |
| `section_f.post_management[]` | CO_S_RISK |
| `section_f.additional_checks[]` | CO_S_ADDITIONAL_CHECK |

---

## 11. 인덱스 전략

### 11.1 필수 인덱스

| 테이블 | 인덱스 | 유형 | 용도 |
|--------|--------|------|------|
| RI_S_REQUEST | `req_id` | PK(Unique) | 전체 트랜잭션 기준 |
| RI_S_REQUEST | `tax_type, request_date` | Composite | 세목별 일자 조회 |
| RI_S_REQUEST | `taxpayer_id, request_date` | Composite | 납세자별 이력 |
| CO_S_CREDIT_DETAIL | `req_id, item_status` | Partial | 적용 가능 항목 필터 |
| CO_S_COMBINATION | `req_id, combo_rank` | Composite | 최적 조합 조회 |
| RF_S_LAW_VERSION | `law_code, effective_from` | Composite | 법령 버전 매칭 |
| SV_S_VALIDATION_LOG | `req_id, rule_result` | Partial | Fail 규칙 필터 |

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-02-17 | 초기 작성 - 83개 테이블 스키마, 핵심 컬럼 상세, RF_* 기준정보, JSON 스키마 |
