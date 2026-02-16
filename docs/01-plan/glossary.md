# 경정청구 환급액 계산 시스템 용어집 (Glossary)

> **Project**: TaxServiceENTEC
> **Version**: 1.0
> **Date**: 2026-02-17
> **Phase**: Phase 1 - Schema/Terminology Definition

---

## 1. 세법 핵심 용어 (Tax Law Core Terms)

| 한글 용어 | English | 코드명 | 정의 | 근거법령 |
|-----------|---------|--------|------|---------|
| 경정청구 | Tax Amendment Claim | `AMENDMENT_CLAIM` | 과세표준 신고서에 기재된 과세표준·세액이 세법에 의하여 신고해야 할 과세표준·세액을 초과할 때 그 차액의 경정을 청구하는 것 | 국기법 §45의2 |
| 후발적 경정청구 | Post-Event Amendment | `POST_EVENT_CLAIM` | 판결, 결정 등 후발적 사유 발생 시 그 사유를 안 날부터 3개월 이내 경정청구 | 국기법 §45의2② |
| 각 사업연도 소득 | Business Year Income | `biz_income` | 법인의 각 사업연도의 익금총액에서 손금총액을 공제한 금액 | 법인세법 §14 |
| 과세표준 | Tax Base | `taxable_income` / `taxable_base` | (법인) 각사업연도소득 - 이월결손금 - 비과세소득 / (개인) 종합소득금액 - 소득공제 | 법인세법 §13 / 소득세법 §14 |
| 산출세액 | Computed Tax | `computed_tax` | 과세표준 x 세율 | 법인세법 §55 / 소득세법 §55 |
| 결정세액 | Determined Tax | `determined_tax` | 산출세액 - 세액감면 - 세액공제 + 가산세 | 법인세법 §64 |
| 최저한세 | Alternative Minimum Tax (AMT) | `min_tax` | 각종 감면·공제를 적용한 후에도 납부해야 하는 최소한의 세금 | 조특법 §132 |
| 환급가산금 | Refund Interest | `refund_interest` | 국세환급금에 대해 일정 이율을 곱한 금액 | 국기법 §52 |
| 실질 환급액 | Net Refund Amount | `net_refund` | 공제·감면세액 합계 - 농어촌특별세 = 실제 수령 가능 환급액 | - |
| 농어촌특별세 | Rural Special Tax | `nongteuk_tax` | 조세감면액의 20%를 별도 부과하는 목적세 (제6·7·10조 비과세) | 농특세법 §4 |

---

## 2. 세액감면 용어 (Tax Exemption Terms)

| 한글 용어 | English | 코드 | 정의 | 근거법령 |
|-----------|---------|------|------|---------|
| 세액감면 | Tax Exemption | `EXEMPTION` | 산출세액에서 일정 비율을 면제. 적용순서 1순위 | 조특법 각조 |
| 세액공제 | Tax Credit | `CREDIT` | 산출세액에서 일정 금액을 차감. 이월불가/이월가능으로 구분 | 조특법 각조 |
| 창업중소기업 세액감면 | Startup SME Exemption | `STARTUP_EXEMPT` | 창업 후 5년간 50~100% 세액감면 | 조특법 §6 |
| 중소기업 특별세액감면 | SME Special Exemption | `SME_SPECIAL` | 중소기업 업종·소재지별 5~30% 세액감면 | 조특법 §7 |
| 통합투자세액공제 | Integrated Investment Credit | `INVEST_CREDIT` | 사업용 유형자산 투자 시 세액공제 (기본 + 추가) | 조특법 §24 |
| 통합고용세액공제 | Integrated Employment Credit | `EMPLOY_CREDIT` | 상시근로자 순증 시 세액공제 (기본 + 추가) | 조특법 §29의8 |
| 연구인력개발비 세액공제 | R&D Tax Credit | `RD_CREDIT` | R&D 지출에 대한 세액공제 (당기분/증가분) | 조특법 §10 |
| 사회보험료 세액공제 | Social Insurance Credit | `SOCIAL_INS_CREDIT` | 상시근로자 사회보험료 사업주 부담분 공제 (~2024) | 조특법 §30의4 |
| 외국납부세액공제 | Foreign Tax Credit | `FOREIGN_TAX_CREDIT` | 외국에서 납부한 세금을 국내 세액에서 공제 | 법인세법 §57 / 소득세법 §57 |
| 이월세액공제 | Carryforward Tax Credit | `CARRYFORWARD_CREDIT` | 최저한세 등으로 당해 사용 불가한 공제를 향후 10년 이월 | 조특법 §144 |
| 결손금 소급공제 | Loss Carryback | `LOSS_CARRYBACK` | 당기 결손금을 직전 사업연도 소득에서 공제하여 환급 | 법인세법 §72 / 소득세법 §85의2 |

---

## 3. 법인세 전용 용어 (Corporate Tax Terms)

| 한글 용어 | English | 코드명 | 정의 | 근거법령 |
|-----------|---------|--------|------|---------|
| 이월결손금 | Loss Carryforward | `loss_carryforward` | 각 사업연도 소득의 80% 한도 내 15년간 이월공제 (중소 100%) | 법인세법 §13 |
| 익금 | Gross Income | `gross_income` | 법인의 자산 증가 또는 부채 감소로 인한 순자산 증가액 | 법인세법 §15 |
| 손금 | Deductible Expense | `deductible_expense` | 법인의 자산 감소 또는 부채 증가로 인한 순자산 감소액 | 법인세법 §19 |
| 결산조정 | Settlement Adjustment | `SETTLEMENT_ADJ` | 법인의 결산 확정 시 장부에 계상해야만 세무상 인정되는 항목. **경정청구 불가** | 법인세법 §40 |
| 신고조정 | Filing Adjustment | `FILING_ADJ` | 세무조정계산서에 의해 조정하는 항목. **경정청구 가능** | 법인세법 §60 |
| 본점 간주 규정 | HQ Deemed Rule | `HQ_DEEMED_RULE` | 법인 본점이 수도권이면 모든 사업장을 수도권으로 간주 (§7 한정, §24는 실제 소재지) | 조특법 §7①단서 |
| 수입배당금 익금불산입 | Dividend Exclusion | `DIVIDEND_EXCLUSION` | 법인이 수령한 배당금의 일부를 익금에서 제외 | 법인세법 §18의2 |
| 연결납세 | Consolidated Tax Filing | `CONSOLIDATED_TAX` | 모·자법인 소득을 합산하여 과세 | 법인세법 §76의8 |

---

## 4. 개인사업자 전용 용어 (Individual Business Terms)

| 한글 용어 | English | 코드명 | 정의 | 근거법령 |
|-----------|---------|--------|------|---------|
| 종합소득금액 | Aggregate Income | `aggregate_income` | 사업소득 + 근로소득 + 이자·배당 + 연금 + 기타소득 합산 | 소득세법 §14 |
| 감면 소득 배분 | Exemption Income Allocation | `income_allocation` | 감면세액 = 산출세액 x (감면대상소득 / 종합소득금액) x 감면율 | 조특법 §6·§7 |
| 소득공제 | Income Deduction | `income_deduction` | 과세표준 산출 전 차감 (인적·연금·건강보험·노란우산 등). 법인에는 없음 | 소득세법 §50~§54 |
| 성실신고확인대상 | Sincerity Reporting Target | `sincerity_target` | 업종별 수입금액 기준 초과 시 해당. 신고기한 6.30 / 의료비·교육비 공제 활성화 | 소득세법 §70의2 |
| 기장세액공제 | Bookkeeping Tax Credit | `BOOKKEEPING_CREDIT` | 소득세법상 공제로 조특법 §127 중복배제 비대상. 모든 감면·공제와 병행 가능 | 소득세법 §56의2 |
| 노란우산공제 | Yellow Umbrella | `YELLOW_UMBRELLA` | 소기업·소상공인 공제부금. 소득구간별 200~500만원 소득공제 | 중소기업협동조합법 §115 |
| 착한 임대인 세액공제 | Good Landlord Credit | `GOOD_LANDLORD` | 소상공인 임차인에게 임대료 인하 시 인하액의 70% 세액공제 | 조특법 §96의3 |
| 공동사업자 | Joint Business | `JOINT_BIZ` | 2인 이상이 공동사업. 손익분배비율에 따라 각자 소득 배분 후 종합소득에 합산 | 소득세법 §43 |

---

## 5. 납세자·사업자 분류 용어 (Taxpayer Classification)

| 한글 용어 | English | 코드값 | 정의 |
|-----------|---------|--------|------|
| 세목 | Tax Type | `CORP` / `INC` | 법인세(CORP) 또는 종합소득세(INC) |
| 법인사업자 | Corporate Taxpayer | `CORP` | 법인세법 적용 내국법인 |
| 개인사업자 | Individual Business | `INC` / `INDV` | 소득세법 적용 거주자 |
| 소기업 | Small Enterprise | `SMALL` | 중소기업기본법 기준 소기업 |
| 중기업 | Medium Enterprise | `MEDIUM` | 중소기업기본법 기준 중기업 |
| 중견기업 | Mid-Size Enterprise | `MID_LARGE` | 중소기업 졸업 후 중견기업 |
| 대기업 | Large Enterprise | `LARGE` | 중견기업 범위 초과 |
| 세무대리인 | Tax Agent | `TAX_AGENT` | 세무사/회계사 (납세자 대리 신청) |
| 신청인 | Applicant | `applicant` | 경정청구를 신청하는 주체 (세무대리인 또는 납세자 직접) |
| 납세자 | Taxpayer | `taxpayer` | 경정청구 대상이 되는 법인/개인 |

---

## 6. 상시근로자 관련 용어 (Employee Terms)

| 한글 용어 | English | 코드명 | 정의 | 근거법령 |
|-----------|---------|--------|------|---------|
| 상시근로자 | Regular Employee | `regular_employee` | 근로계약 1년 이상, 월 60시간 이상. 임원·최대주주 친족 등 제외 | 조특법 시행령 §23⑩ |
| 청년 (통합고용) | Youth (Employment) | `YOUTH` | 만 15~34세 (병역 최대 6년 가산). 2025년부터 근로계약 체결 시점 기준 | 조특법 시행령 §26의8 |
| 청년 (사보공제) | Youth (Social Ins.) | `YOUTH_SI` | 만 15~29세 | 조특법 §30의4 |
| 경력단절여성 | Career-Break Woman | `CAREER_BREAK` | 결혼·임신·출산·육아 사유 퇴직 후 2~15년 재고용 여성 | 조특법 §29의8③ |
| 장애인 | Disabled Person | `DISABLED` | 장애인복지법 기준 | 조특법 §29의8 |
| 60세 이상 | Senior (60+) | `SENIOR_60` | 만 60세 이상 근로자 | 조특법 §29의8 |

---

## 7. 수도권·지역 분류 (Capital Zone Classification)

| 한글 용어 | English | 코드값 | 정의 |
|-----------|---------|--------|------|
| 과밀억제권역 | Overcrowding Control | `OVERCROWDED` | 서울 전역, 인천·경기 일부 (감면율 최저) |
| 성장관리권역 | Growth Management | `GROWTH_MGMT` | 수도권 중 과밀억제 외 일부 |
| 자연보전권역 | Natural Conservation | `NATURAL_CONS` | 수도권 중 자연보전 구역 |
| 비수도권 | Non-Capital Region | `NON_CAPITAL` | 수도권 외 전국 (감면율 최고) |
| 인구감소지역 | Depopulation Area | `DEPOPULATION` | 행정안전부 고시 지역 (2025~ 창업 100% 감면) |

---

## 8. 시스템 구조 용어 (System Architecture Terms)

| 한글 용어 | English | 코드명 | 정의 |
|-----------|---------|--------|------|
| 요청번호 | Request ID | `req_id` | 1회 경정청구 진단 = 1 트랜잭션. 형식: `{C/I}-{사업자번호}-{YYYYMMDD}-{순번}` |
| 원시 JSON | Raw JSON | `RI_S_RAW_DATA` | INSERT ONLY 원본 데이터. 수정 시 새 req_id 발급 필수 |
| 요약 테이블 | Summary Table | `SV_*` | 원시 JSON에서 파싱한 계산용 정규화 데이터 |
| 산출 결과 | Calculation Output | `CO_*` | 공제·감면 계산, 조합 최적화, 환급액 산출 결과 |
| 기준 정보 | Reference Data | `RF_*` | 세율·공제율·법령 등 공통 기준 (req_id 없이 독립 관리) |
| 감사 로그 | Audit Log | `AL_*` | 전 단계 계산 이력 추적 |
| 보고서 JSON | Report JSON | `CO_S_REPORT_JSON` | 최종 보고서 7섹션(A~G) JSON 직렬화 |

---

## 9. 주제영역 접두어 (Table Prefix Convention)

| 접두어 | 영문 Full Name | 한글명 | 테이블 수 |
|--------|---------------|--------|:---------:|
| `RI_` | Request & Input | 요청 및 입력 상세 데이터 | 37 |
| `SV_` | Summary & Validation | 입력자료 요약 및 검증 결과 | 8 |
| `CO_` | Calculation Output | 산출 결과 | 11 |
| `AL_` | Audit Log | 감사 로그 | 1 |
| `RF_` | Reference | 기준 정보 | 26 |
| **합계** | | | **83** |

### 활용구분 코드 (Usage Classifier)

| 코드 | 영문 | 한글 | 설명 |
|------|------|------|------|
| `C` | Corporate | 법인 전용 | 법인세 경정청구에만 사용 |
| `I` | Individual | 개인 전용 | 종합소득세 경정청구에만 사용 |
| `S` | Shared | 공동 활용 | 법인/개인 공통 사용 |

---

## 10. 모듈 식별자 (Module Identifiers)

| 모듈 ID | 한글명 | 영문명 | 역할 |
|---------|--------|--------|------|
| M1 | 입력 관리 | Input Management | req_id 발급, JSON 수신, 요약 생성 |
| M2 | 기준정보 관리 | Reference Management | RF_* 로드, 법령 버전 매칭 |
| M3 | 사전 점검 | Pre-Check | 불가사유 차단, 기한·자격·소재지·근로자 판단 |
| M4 | 개별 공제·감면 | Individual Credits | 14개 서브모듈 병렬 산출 |
| M5 | 최적 조합 | Optimal Combination | 상호배제, 최저한세, 농특세, 적용순서 |
| M6 | 최종 산출·보고 | Final Calc & Report | 환급액 비교, 가산금, 보고서 3종 |
| M7 | 시뮬레이션 | Simulation | 다수연도, 이월최적화, 사후관리 |
| MX-01 | 순환참조 해결 | Circular Ref Resolver | 과세표준↔최저한세 최대 5회 수렴 |
| MX-02 | 법령 버전 매칭 | Law Version Matcher | 사업연도 → 세율·공제율·적용기한 자동 로드 |
| MX-03 | 검증 엔진 | Validation Engine | 71개 검증 규칙 실행 |

---

## 11. 검증 규칙 코드 체계 (Validation Rule Codes)

| 접두어 | 카테고리 | 건수 | 설명 |
|--------|----------|:----:|------|
| `V` | Validation | 19 | 입력값 유효성 검증 |
| `C` | Constraint | 13 | 비즈니스 제약 조건 |
| `B` | Business Rule | 11 | 업무 규칙 검증 |
| `L` | Limit | 9 | 한도 초과 검증 |
| `X` | Cross-Check | 13 | 항목 간 교차 검증 |
| `W` | Warning | 6 | 경고 (진행 가능, 주의 필요) |
| **합계** | | **71** | |

---

## 12. 계산 공식 식별자 (Formula Identifiers)

| 범위 | 카테고리 | 설명 |
|------|----------|------|
| F01~F10 | 감면 공식 | 창업감면, 중소기업감면 등 |
| F11~F20 | 공제 공식 | 고용공제, 투자공제, R&D 등 |
| F21~F30 | 최적화 공식 | 상호배제, 최저한세, 농특세 |
| F31~F40 | 산출 공식 | 환급액, 가산금, 지방소득세 |
| F41~F56 | 보조 공식 | 소득배분, 순환참조, 이월 등 |

---

## 13. 적용순서 (Application Order)

### 법인세 (법인세법 §59)

```
① 세액감면 (§6 창업감면, §7 중소감면 등)
② 이월불가 세액공제 (§29의8 통합고용 등)
③ 이월가능 세액공제 (§24 투자, §10 R&D 등)
```

### 종합소득세 (소득세법 §60)

```
① 세액감면 (§6 창업감면, §7 중소감면 등)
② 이월불가 세액공제 (§29의8 통합고용, 기장세액공제 등)
③ 이월가능 세액공제 (§24 투자, §10 R&D 등)
※ 기장세액공제(§56의2)는 조특법 중복배제 비대상 → 모든 감면과 병행
```

---

## 14. 그룹 비교 분류 (Group Comparison)

| 그룹 | 전략 | 포함 항목 | 농특세 |
|------|------|-----------|--------|
| **그룹 A** | 감면 중심 | §6 창업감면 또는 §7 중소감면 | §6·§7 비과세 |
| **그룹 B** | 공제 중심 | §24 투자 + §29의8 고용 + §30의4 사보 | §24·§29의8 과세(20%) |

---

## 15. 절사(Truncate) 정책

| 대상 | 기준 | 예시 |
|------|------|------|
| 금액 | 10원 미만 절사 | 1,234,567원 → 1,234,560원 |
| 비율 | 소수점 3자리 절사 | 0.62578 → 0.625 |
| 환급가산금 | 1원 미만 절사 | 2,191,800.7원 → 2,191,800원 |
| **반올림** | **절대 금지** | 모든 단계 TRUNCATE만 허용 |

---

## 16. 코드에서의 용어 사용 규칙

1. **변수명**: 영문 camelCase (`taxableIncome`, `computedTax`, `netRefund`)
2. **DB 컬럼**: UPPER_SNAKE_CASE (`TAXABLE_INCOME`, `COMPUTED_TAX`)
3. **API 응답**: snake_case (`taxable_income`, `computed_tax`)
4. **UI/문서**: 한국어 우선 (과세표준, 산출세액, 실질 환급액)
5. **법령 인용**: 약칭 사용 (국기법, 법인세법, 소득세법, 조특법, 농특세법)

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-02-17 | 초기 작성 - 16개 카테고리, 100+ 용어 정의 |
