# 통합 경정청구 환급액 산출 시스템 - 엔티티 및 데이터 스키마 정의서

> **문서 버전**: v1.0 (외부자료 v4.0 기반)
> **작성일**: 2026-02-17
> **총 테이블**: 83개 + 뷰 1개

---

## 1. 테이블 명명규칙

```
{주제영역}_{활용구분}_{테이블명}
```

| 구성 요소 | 코드 | 의미 |
|----------|------|------|
| 주제영역 (2자) | RI / SV / CO / AL / RF | 요청입력 / 요약검증 / 산출결과 / 감사로그 / 기준정보 |
| 활용구분 (1자) | C / I / S | 법인전용 / 개인전용 / 공동활용 |
| 테이블명 | UPPER_SNAKE_CASE | 업무 의미 반영 |

---

## 2. 주제영역 ① 요청 및 입력 상세 (RI_) - 37개

### 2.1 공동활용 테이블 (S) - 12개

#### RI_S_REQUEST - 요청 마스터
> 모든 비기준 테이블의 최상위 부모. 신청인별/날짜별 유니크한 요청 1건 = 1 row.

| 컬럼명 | 한글명 | 데이터타입 | 필수 | 설명 |
|--------|--------|-----------|:----:|------|
| **req_id** | 요청번호 | VARCHAR(30) | PK | C-1234567890-20260216-001 |
| **applicant_id** | 신청인ID | VARCHAR(20) | FK | &rarr; RI_S_APPLICANT |
| **taxpayer_id** | 납세자ID | VARCHAR(20) | FK | &rarr; RI_S_TAXPAYER |
| tax_type | 세목구분 | VARCHAR(4) | Y | CORP / INC |
| tax_year | 대상과세연도 | VARCHAR(4) | Y | 예: 2024 |
| request_date | 요청일자 | DATE | Y | req_id의 날짜 부분 |
| seq_no | 일련번호 | INT | Y | 당일 순번 |
| request_status | 요청상태 | VARCHAR(20) | Y | received/parsing/parsed/preparing/checking/calculating/optimizing/reporting/completed/error |
| prompt_version | 프롬프트버전 | VARCHAR(20) | N | 예: v1.3 |
| design_version | 설계서버전 | VARCHAR(10) | N | 예: v4.0 |
| created_at | 생성일시 | TIMESTAMP | Y | 자동 |
| completed_at | 완료일시 | TIMESTAMP | N | |
| request_source | 요청원천 | VARCHAR(30) | Y | external_api/portal/batch/prompt |
| requested_by | 요청자ID | VARCHAR(50) | Y | |
| client_ip | 요청자IP | VARCHAR(45) | N | IPv4/IPv6 |
| error_message | 오류메시지 | TEXT | N | |
| modified_by | 수정자ID | VARCHAR(50) | N | |
| modified_at | 수정일시 | TIMESTAMP | N | |
| version | 데이터버전 | INT | Y | DEFAULT 1 |

**req_id 생성규칙**: `{신청인구분(1)}-{사업자번호(10)}-{요청일자(8)}-{일련번호(3)}`

#### RI_S_APPLICANT - 신청인 정보

| 컬럼명 | 한글명 | 데이터타입 | 필수 | 설명 |
|--------|--------|-----------|:----:|------|
| **applicant_id** | 신청인ID | VARCHAR(20) | PK | APP-{seq} |
| applicant_type | 신청인유형 | VARCHAR(15) | Y | TAX_AGENT/SELF_CORP/SELF_INDV |
| agent_name | 신청인명 | VARCHAR(100) | Y | |
| agent_biz_reg_no | 사업자번호 | VARCHAR(15) | N | 세무대리인의 사업자번호 |
| agent_license_no | 세무사등록번호 | VARCHAR(20) | N | TAX_AGENT일 때 필수 |
| agent_phone | 연락처 | VARCHAR(20) | N | |
| agent_email | 이메일 | VARCHAR(100) | N | |
| is_active | 활성여부 | BOOLEAN | Y | DEFAULT TRUE |
| created_at | 등록일시 | TIMESTAMP | Y | |
| updated_at | 수정일시 | TIMESTAMP | N | |

#### RI_S_TAXPAYER - 납세자 정보

| 컬럼명 | 한글명 | 데이터타입 | 필수 | 설명 |
|--------|--------|-----------|:----:|------|
| **taxpayer_id** | 납세자ID | VARCHAR(20) | PK | TP-{seq} |
| taxpayer_type | 납세자유형 | VARCHAR(4) | Y | CORP/INDV |
| biz_reg_no | 사업자등록번호 | VARCHAR(15) | Y | 유니크 |
| corp_reg_no | 법인등록번호 | VARCHAR(15) | N | CORP 전용 |
| taxpayer_name | 납세자명 | VARCHAR(100) | Y | |
| representative_name | 대표자명 | VARCHAR(50) | N | CORP일 때 필수 |
| industry_code | 업종코드 | VARCHAR(10) | Y | KSIC |
| hq_location | 본점소재지 | VARCHAR(200) | Y | |
| capital_zone | 수도권구분 | VARCHAR(20) | N | 과밀억제/성장관리/자연보전/비수도권 |
| depopulation_area | 인구감소지역 | BOOLEAN | N | |
| corp_size | 기업규모 | VARCHAR(10) | Y | 소기업/중기업/중견/대기업 |
| founding_date | 설립/창업일 | DATE | N | |
| is_active | 활성여부 | BOOLEAN | Y | |
| created_at | 등록일시 | TIMESTAMP | Y | |
| updated_at | 수정일시 | TIMESTAMP | N | |

#### RI_S_RAW_DATA - 원시 입력자료 보관 (INSERT ONLY)

| 컬럼명 | 한글명 | 데이터타입 | 필수 | 설명 |
|--------|--------|-----------|:----:|------|
| raw_id | 원시데이터ID | BIGINT | PK | AUTO_INCREMENT |
| **req_id** | 요청번호 | VARCHAR(30) | FK | &rarr; RI_S_REQUEST |
| category | 데이터카테고리 | VARCHAR(30) | Y | 32개 카테고리 코드 |
| sub_category | 세부카테고리 | VARCHAR(30) | N | |
| **raw_json** | 원시JSON데이터 | JSON/TEXT | Y | 수신된 원본 그대로 보관 |
| json_schema_version | 스키마버전 | VARCHAR(10) | N | |
| record_count | 레코드건수 | INT | N | |
| byte_size | 데이터크기 | INT | N | |
| checksum | 체크섬 | VARCHAR(64) | N | SHA-256 |
| received_at | 수신일시 | TIMESTAMP | Y | |

**카테고리 코드 (32개)**:
- 공통: employee_detail, employee_monthly, investment, startup, sme_special, rd_expense, existing_deduction, foreign_tax
- CORP 전용: corp_basic, representative, loss_carryforward, credit_carryforward, branch_location, interim_tax, dividend_income, business_vehicle, tax_adjustment, entertainment, government_subsidy, shareholder_loan, non_business_asset, disaster_loss, consolidated_sub, depreciation_adjust
- INC 전용: inc_basic, inc_business, inc_other_income, inc_deduction, inc_sincerity, inc_foreign_tax, inc_rental_reduction, inc_joint_biz

#### 기타 공동활용 RI_S_* 테이블 (JSON 스키마 정의 역할)

| 테이블 | 설명 | PK |
|--------|------|----|
| RI_S_EMPLOYEE_DETAIL | 상시근로자 상세 명부 | (req_id, emp_id, year_type) |
| RI_S_EMPLOYEE_MONTHLY | 월별 상시근로자 요약 | (req_id, year_month) |
| RI_S_STARTUP_INFO | 창업기업 감면 정보 | (req_id) |
| RI_S_SME_SPECIAL | 중소기업 특별감면 | (req_id) |
| RI_S_FOREIGN_TAX | 외국납부세액 상세 | (req_id, country_seq) |
| RI_S_LOSS_CARRYFORWARD | 이월결손금 상세 | (req_id, origin_year, loss_seq) |
| RI_S_CREDIT_CARRYFORWARD | 이월세액공제 상세 | (req_id, origin_year, provision, cf_seq) |
| RI_S_EXISTING_DEDUCTION | 기존 신고 공제/감면 | (req_id, provision, tax_year, deduction_seq) |

### 2.2 법인 전용 테이블 (C) - 17개

| 테이블 | 설명 | 핵심 컬럼 |
|--------|------|----------|
| RI_C_CORP_BASIC | 법인 기본 정보 (53컬럼) | corp_size, industry_code, revenue, taxable_income, computed_tax |
| RI_C_REPRESENTATIVE | 대표자/임원/가족 | role_type, birth_date, military_start/end |
| RI_C_BRANCH_LOCATION | 지점 소재지 | address, capital_zone, is_depopulation |
| RI_C_INVESTMENT_ASSET | 투자 자산 명세 | acquire_cost, invest_type, is_rental_asset, is_used_asset |
| RI_C_RD_EXPENSE | R&D 비용 상세 (5개년) | tax_year, rd_type, total_rd_cost, is_commissioned |
| RI_C_BUSINESS_VEHICLE | 업무용 승용차 | logbook_yn, dedicated_insurance_yn |
| RI_C_TAX_ADJUSTMENT | 세무조정계산서 | adjustment_type, settlement_or_filing |
| RI_C_ENTERTAINMENT | 접대비 | expense_amount |
| RI_C_SHAREHOLDER_LOAN | 가지급금 (주주차입금) | loan_balance, interest_rate |
| RI_C_NON_BUSINESS_ASSET | 업무무관자산 | asset_value, related_interest |
| RI_C_GOVERNMENT_SUBSIDY | 국고보조금 | subsidy_amount |
| RI_C_DIVIDEND_INCOME | 수입배당금 | share_ratio, dividend_amt, is_listed |
| RI_C_INTERIM_TAX | 중간예납 | payment_date, payment_amount |
| RI_C_DISASTER_LOSS | 재해손실 | loss_ratio |
| RI_C_CONSOLIDATED_SUB | 연결납세 자회사 | sub_corp_name, ownership_ratio |
| RI_C_DEPRECIATION_ADJUST | 감가상각 시부인 이월 | excess_amount, deferred_amount |

### 2.3 개인 전용 테이블 (I) - 8개

| 테이블 | 설명 | 핵심 컬럼 |
|--------|------|----------|
| RI_I_BASIC | 개인사업자 기본 정보 | bookkeeping_type, sincerity_target, joint_biz_yn |
| RI_I_BUSINESS | 사업장별 상세 정보 | biz_name, industry_code, location, revenue, biz_income |
| RI_I_OTHER_INCOME | 사업소득 외 합산소득 | wage_income, financial_income, pension_income |
| RI_I_DEDUCTION | 소득공제 항목 | personal_deduction, pension_deduction, yellow_umbrella |
| RI_I_SINCERITY | 성실신고확인 관련 | verification_cost, medical_expense, education_expense |
| RI_I_FOREIGN_TAX | 개인 외국납부세액 | country, tax_amount, credit_or_expense |
| RI_I_RENTAL_REDUCTION | 착한 임대인 세액공제 | reduction_amount, reduction_period |
| RI_I_JOINT_BIZ | 공동사업자 정보 | profit_share_ratio, joint_biz_income |

---

## 3. 주제영역 ② 요약 및 검증 (SV_) - 8개 (전체 공동활용)

### 3.1 계산용 요약 테이블 (4개)

#### SV_S_BASIC - 납세자 기본 정보 요약

| 컬럼명 | 한글명 | 데이터타입 | 필수 | 설명 |
|--------|--------|-----------|:----:|------|
| **req_id** | 요청번호 | VARCHAR(30) | PK/FK | |
| **request_date** | 입력자료요청일자 | DATE | Y | req_id의 날짜 부분 |
| tax_type | 세목구분 | VARCHAR(4) | Y | CORP/INC |
| taxpayer_name | 납세자명 | VARCHAR(100) | Y | |
| biz_reg_no | 사업자등록번호 | VARCHAR(15) | Y | |
| corp_size | 기업규모 | VARCHAR(10) | Y | |
| industry_code | 업종코드 | VARCHAR(10) | Y | |
| hq_location | 본점소재지 | VARCHAR(200) | Y | |
| capital_zone | 수도권구분 | VARCHAR(20) | Y | |
| depopulation_area | 인구감소지역 | BOOLEAN | N | |
| tax_year | 대상과세연도 | VARCHAR(4) | Y | |
| fiscal_start | 사업연도시작 | DATE | N | CORP 전용 |
| fiscal_end | 사업연도종료 | DATE | N | CORP 전용 |
| revenue | 매출액 | BIGINT | Y | |
| taxable_income | 과세표준 | BIGINT | Y | |
| computed_tax | 산출세액 | BIGINT | Y | |
| paid_tax | 기납부세액 | BIGINT | Y | |
| founding_date | 창업일자 | DATE | N | |
| venture_yn | 벤처기업확인 | BOOLEAN | N | |
| rd_dept_yn | 연구전담부서 | BOOLEAN | N | |
| claim_reason | 경정청구사유 | VARCHAR(200) | Y | |
| sincerity_target | 성실신고대상 | BOOLEAN | N | INC 전용 |
| bookkeeping_type | 기장유형 | VARCHAR(10) | N | INC 전용 |
| consolidated_tax | 연결납세여부 | BOOLEAN | N | CORP 전용 |

#### SV_S_EMPLOYEE - 고용 정보 요약

| 컬럼명 | 데이터타입 | 설명 |
|--------|-----------|------|
| **req_id** | VARCHAR(30) | FK |
| **year_type** | VARCHAR(7) | PRIOR/CURRENT |
| total_regular | DECIMAL(10,2) | 전체상시근로자 (연평균) |
| youth_count | DECIMAL(10,2) | 청년등근로자 |
| disabled_count | DECIMAL(10,2) | 장애인 |
| aged_count | DECIMAL(10,2) | 60세이상 |
| career_break_count | DECIMAL(10,2) | 경력단절여성 |
| north_defector_count | DECIMAL(10,2) | 북한이탈주민 |
| general_count | DECIMAL(10,2) | 일반근로자 |
| excluded_count | INT | 제외대상인원 |
| total_salary | BIGINT | 임금총액 |
| social_insurance_paid | BIGINT | 사회보험료부담액 |

**PK**: (req_id, year_type)

#### SV_S_DEDUCTION - 공제/감면 기초 요약

| 컬럼명 | 데이터타입 | 설명 |
|--------|-----------|------|
| **req_id** | VARCHAR(30) | FK |
| **item_category** | VARCHAR(20) | INVEST/RD/STARTUP/SME_SPECIAL/EMPLOYMENT/... |
| **provision** | VARCHAR(20) | &sect;24, &sect;10, &sect;6, &sect;7 등 |
| **tax_year** | VARCHAR(4) | 과세연도 (R&D 5개년 등) |
| **item_seq** | INT | 건순번 |
| base_amount | BIGINT | 기초금액 |
| zone_type | VARCHAR(20) | 수도권/비수도권 |
| asset_type | VARCHAR(50) | 투자 자산유형 |
| rd_type | VARCHAR(30) | 국가전략/신성장/일반 |
| method | VARCHAR(10) | 증가분(INCREMENTAL)/당기분(CURRENT) |
| sub_detail | JSON/TEXT | 비목별 상세 |
| existing_applied | BOOLEAN | 기존적용여부 |
| existing_amount | BIGINT | 기존적용금액 |
| carryforward_balance | BIGINT | 이월잔액 |

**PK**: (req_id, item_category, provision, tax_year, item_seq)

#### SV_S_FINANCIAL - 재무/세무 수치 요약

| 컬럼명 | 데이터타입 | 설명 | 세목 |
|--------|-----------|------|------|
| **req_id** | VARCHAR(30) | PK/FK | 공통 |
| biz_income | BIGINT | 각사업연도소득 | CORP |
| non_taxable_income | BIGINT | 비과세소득 | CORP |
| loss_carryforward_total | BIGINT | 이월결손금총잔액 | 공통 |
| loss_carryforward_detail | TEXT | JSON: [{year, balance}] | 공통 |
| interim_prepaid_tax | BIGINT | 중간예납세액 | CORP |
| withholding_tax | BIGINT | 원천납부세액 | 공통 |
| determined_tax | BIGINT | 기존결정세액 | 공통 |
| dividend_income_total | BIGINT | 수입배당금합계 | CORP |
| foreign_tax_total | BIGINT | 외국납부세액합계 | 공통 |
| tax_adjustment_detail | TEXT | 세무조정요약 JSON | CORP |
| inc_deduction_total | BIGINT | 소득공제합계 | INC |
| inc_comprehensive_income | BIGINT | 종합소득금액 | INC |
| current_year_loss | BIGINT | 당기결손금 | 공통 |
| prior_year_tax_paid | BIGINT | 직전연도납부세액 | 공통 |

### 3.2 점검/검증 결과 테이블 (4개)

#### SV_S_PREP_SUMMARY - 사전 데이터 준비 요약

> M3-PREP 단계에서 SV_* 완전성 검증 + 기초 데이터 사전 산정 + 적용 후보 조항 도출

핵심 컬럼: inp_basic_yn, inp_employee_yn, law_version_id, applicable_tax_rates, candidate_provisions, prep_status

**PK**: req_id

#### SV_S_ELIGIBILITY - 자격 진단 결과

핵심 컬럼: company_size, capital_zone, filing_deadline, claim_deadline, deadline_eligible, sme_eligible, settlement_check_result, overall_status

**PK**: req_id

#### SV_S_INSPECTION_LOG - 점검항목별 판정

핵심 컬럼: inspection_code, inspection_name, legal_basis, judgment, calculated_amount

**PK**: (req_id, inspection_code)

#### SV_S_VALIDATION_LOG - 검증 규칙 실행 결과

핵심 컬럼: rule_code, rule_type (V/C/B/L/W/X), result (PASS/FAIL/WARNING/SKIP)

**PK**: (req_id, rule_code)

---

## 4. 주제영역 ③ 산출 결과 (CO_) - 11개

### 4.1 공동활용 (S) - 8개

| 테이블 | 설명 | PK |
|--------|------|----|
| CO_S_EMPLOYEE_SUMMARY | 상시근로자 산정 결과 | (req_id, year_type) |
| CO_S_CREDIT_DETAIL | 개별 공제/감면 산출 (25컬럼) | (req_id, item_id) |
| CO_S_COMBINATION | 조합 비교/최적 선택 | (req_id, combo_id) |
| CO_S_EXCLUSION_VERIFY | 상호배제 검증 | (req_id, verify_id) |
| CO_S_REFUND | 최종 환급액 산출 | req_id |
| CO_S_RISK | 사후관리/리스크 평가 | (req_id, risk_id) |
| CO_S_ADDITIONAL_CHECK | 추가 확인 필요 | (req_id, check_id) |
| CO_S_REPORT_JSON | 보고서 JSON 직렬화 (7섹션 A~G) | req_id |

#### CO_S_CREDIT_DETAIL 주요 컬럼

| 컬럼명 | 설명 |
|--------|------|
| item_id | 항목ID (ITEM-07 등) |
| provision | 적용조항 |
| credit_type | 감면/공제 |
| item_status | applicable/not_applicable/needs_review |
| gross_amount | 총공제감면액 (농특세 차감 전) |
| nongteuk_amount | 농특세액 |
| net_amount | 실질환급액 (농특세 차감 후) |
| min_tax_subject | 최저한세대상 여부 |
| is_carryforward | 이월가능여부 |
| exclusion_reasons | 배제사유코드 JSON |

#### CO_S_REFUND 주요 컬럼

| 컬럼명 | 설명 |
|--------|------|
| existing_computed_tax | 기존산출세액 |
| new_determined_tax | 경정후결정세액 |
| refund_amount | 환급액 (기존납부 - 경정후결정) |
| refund_interest_amount | 환급가산금 |
| local_tax_refund | 지방소득세환급 (국세 &times; 10%) |
| total_expected | 총수령예상액 |
| optimal_combo_id | 최적조합ID (FK &rarr; CO_S_COMBINATION) |

### 4.2 개인 전용 (I) - 3개

| 테이블 | 설명 | PK |
|--------|------|----|
| CO_I_INCOME_ALLOCATION | 다사업장 소득 배분 산출 | (req_id, biz_seq) |
| CO_I_DEDUCTION_IMPACT | 소득공제 반영 전후 세율구간 변동 분석 | req_id |
| CO_I_LOCAL_TAX | 지방소득세 환급액 별도 산출 | req_id |

---

## 5. 주제영역 ④ 감사 로그 (AL_) - 1개

#### AL_S_CALCULATION - 감사추적 로그

| 컬럼명 | 데이터타입 | 설명 |
|--------|-----------|------|
| log_id | BIGINT | PK (AUTO_INCREMENT) |
| **req_id** | VARCHAR(30) | FK |
| calc_step | VARCHAR(50) | STEP0/STEP1/.../M1-03 등 |
| function_name | VARCHAR(100) | |
| input_data | TEXT | JSON |
| output_data | TEXT | JSON |
| legal_basis | VARCHAR(100) | |
| executed_at | TIMESTAMP | |
| log_level | VARCHAR(10) | INFO/WARN/ERROR/DEBUG |
| executed_by | VARCHAR(50) | 시스템ID/사용자ID |
| duration_ms | INT | 소요시간 |
| trace_id | VARCHAR(50) | API 요청 단위 고유 ID |
| prev_data_hash | VARCHAR(64) | 재처리 시 이전 데이터 SHA-256 |

---

## 6. 주제영역 ⑤ 기준 정보 (RF_) - 26개 (req_id 없이 독립 관리)

### 6.1 공동활용 (S) - 15개

| 테이블 | 설명 | 핵심 컬럼 |
|--------|------|----------|
| RF_S_TAX_RATE | 통합 세율 기준 | year_from/to, bracket_min/max, tax_rate |
| RF_S_MIN_TAX_RATE | 최저한세율 기준 | corp_size, min_rate |
| RF_S_EMPLOYMENT_CREDIT | 통합고용 공제 기준 | corp_size, region, worker_type, credit_per_person |
| RF_S_SME_DEDUCTION_RATE | 중소기업 특별감면율 | corp_size_detail, industry_class, zone_type, deduction_rate |
| RF_S_STARTUP_DEDUCTION_RATE | 창업감면 감면율 | founder_type, location_type, deduction_rate |
| RF_S_MUTUAL_EXCLUSION | 상호배제 매트릭스 | provision_a/b, is_allowed, condition_note |
| RF_S_NONGTEUKSE | 농특세 적용 기준 | provision, is_exempt, tax_rate |
| RF_S_CAPITAL_ZONE | 수도권 지역 구분 | sido, sigungu, zone_type, is_capital |
| RF_S_DEPOPULATION_AREA | 인구감소지역 목록 | sido, sigungu, designation_date |
| RF_S_INDUSTRY_ELIGIBILITY | 업종별 감면 적격성 | ksic_code, startup_eligible, sme_special_eligible |
| RF_S_KSIC_CODE | 한국표준산업분류 | ksic_code, section, industry_name |
| RF_S_REFUND_INTEREST_RATE | 환급가산금 이율 | effective_from/to, annual_rate |
| RF_S_EXCHANGE_RATE | 외국환 매매기준율 | rate_date, currency, standard_rate |
| RF_S_SYSTEM_PARAM | 시스템 파라미터 | param_key, param_value |
| RF_S_LAW_VERSION | 법령 버전 관리 | law_name, provision, year_from/to |

### 6.2 법인 전용 (C) - 7개

| 테이블 | 설명 |
|--------|------|
| RF_C_TAX_RATE_HISTORY | 법인세율 이력 (2022년 이전 포함) |
| RF_C_INVESTMENT_CREDIT_RATE | 통합투자세액공제율 |
| RF_C_RD_CREDIT_RATE | R&D 세액공제율 (유형/방식/기업규모별) |
| RF_C_RD_MIN_TAX_EXEMPT | R&D 최저한세 배제율 |
| RF_C_ENTERTAINMENT_LIMIT | 접대비 한도 기준 |
| RF_C_DEEMED_INTEREST_RATE | 인정이자율 기준 |
| RF_C_DIVIDEND_EXCLUSION | 수입배당금 익금불산입률 |

### 6.3 개인 전용 (I) - 4개

| 테이블 | 설명 | 핵심 컬럼 |
|--------|------|----------|
| RF_I_TAX_RATE | 소득세율 (6~45% 8단계) | effective_from, bracket_no, tax_rate, progressive_deduction |
| RF_I_MIN_TAX | 개인 최저한세율 | threshold(3천만), rate_below(35%), rate_above(45%) |
| RF_I_DEDUCTION_LIMIT | 소득공제 한도 | deduction_type, income_bracket, annual_limit |
| RF_I_SINCERITY_THRESHOLD | 성실신고확인 기준 | industry_group, revenue_threshold |

---

## 7. 뷰 (View) - 1개

#### VW_C_RD_5Y_BASE - R&D 5개년 집계 뷰

> M4-06 모듈에서 증가분/당기분 산출 시 참조

```sql
SELECT d.req_id, d.provision, d.tax_year, d.rd_type, d.method,
       d.base_amount, r.credit_rate, r.min_tax_exempt
FROM SV_S_DEDUCTION d
LEFT JOIN RF_C_RD_CREDIT_RATE r ON d.rd_type = r.rd_type AND d.method = r.method
WHERE d.item_category = 'RD'
```

---

## 8. 테이블 수 총괄

| 주제영역 | 접두어 | C (법인) | I (개인) | S (공동) | 합계 |
|----------|--------|:--------:|:--------:|:--------:|:----:|
| 요청 및 입력 상세 | RI_ | 17 | 8 | 12 | **37** |
| 요약 및 검증 | SV_ | - | - | 8 | **8** |
| 산출 결과 | CO_ | - | 3 | 8 | **11** |
| 감사 로그 | AL_ | - | - | 1 | **1** |
| 기준 정보 | RF_ | 7 | 4 | 15 | **26** |
| **합계** | | **24** | **15** | **44** | **83** + 뷰 1 |
