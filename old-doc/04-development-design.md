# 통합 경정청구 환급액 산출 시스템 - 개발설계서

> **문서 버전**: v1.0
> **작성일**: 2026-02-17
> **기반 문서**: 외부자료_v4_0.md, tax-refund-system.plan.md, 01-glossary.md, 02-entity-schema.md, 03-entity-relationship.md
> **프로젝트 레벨**: Enterprise (Clean Architecture)

---

## 목차

1. [시스템 개요](#1-시스템-개요)
2. [아키텍처 설계](#2-아키텍처-설계)
3. [모듈 상세 설계](#3-모듈-상세-설계)
4. [REST API 설계](#4-rest-api-설계)
5. [데이터 처리 파이프라인](#5-데이터-처리-파이프라인)
6. [계산 엔진 설계](#6-계산-엔진-설계)
7. [검증 엔진 설계](#7-검증-엔진-설계)
8. [보고서 생성 설계](#8-보고서-생성-설계)
9. [트랜잭션 및 동시성 설계](#9-트랜잭션-및-동시성-설계)
10. [보안 및 데이터 보호](#10-보안-및-데이터-보호)
11. [구현 순서 및 의존성](#11-구현-순서-및-의존성)
12. [부록: 핵심 의사코드](#12-부록-핵심-의사코드)

---

## 1. 시스템 개요

### 1.1 목적

법인사업자(법인세, CORP)와 개인사업자(종합소득세, INC)의 경정청구 시 환급액을 극대화하는 AI 기반 통합 점검 시스템.

### 1.2 핵심 설계 원칙

| 원칙 | 설명 |
|------|------|
| **Request-Driven** | 1회 경정청구 진단 = 1 트랜잭션 단위 (req_id 관통) |
| **원본 불변성** | RI_S_RAW_DATA = INSERT ONLY, 수정 시 새 req_id 발급 |
| **요약 재생성** | SV_* = RI_S_RAW_DATA에서 언제든 재파싱 가능 (파생 데이터) |
| **결과 이중 보관** | CO_* 정규 테이블 + CO_S_REPORT_JSON (JSON 직렬화) |
| **기준정보 독립** | RF_* = req_id 없이 독립 관리 (법령/세율 공통 기준) |
| **세목 분기** | TAX_TYPE(CORP/INC)으로 모듈/테이블/기준정보 자동 활성화 |
| **절사 원칙** | 금액 10원 미만 TRUNCATE, 비율 소수점 3자리, 반올림 절대 금지 |
| **적용순서** | 감면 → 이월불가 공제 → 이월가능 공제 (법인세법 §59, 소득세법 §60) |

### 1.3 req_id 생성규칙

```
형식: {신청인구분(1)}-{사업자번호(10)}-{요청일자(8)}-{일련번호(3)}
예시: C-1234567890-20260216-001

신청인구분: C(법인), I(개인)
일련번호: 당일 동일 신청인 순번 (001~999)
```

### 1.4 데이터 규모

| 주제영역 | 접두어 | C (법인) | I (개인) | S (공동) | 합계 |
|----------|--------|:--------:|:--------:|:--------:|:----:|
| 요청 및 입력 상세 | RI_ | 17 | 8 | 12 | **37** |
| 요약 및 검증 | SV_ | - | - | 8 | **8** |
| 산출 결과 | CO_ | - | 3 | 8 | **11** |
| 감사 로그 | AL_ | - | - | 1 | **1** |
| 기준 정보 | RF_ | 7 | 4 | 15 | **26** |
| **합계** | | **24** | **15** | **44** | **83** + 뷰 1 |

---

## 2. 아키텍처 설계

### 2.1 기술 스택

| 항목 | 선정 기술 | 근거 |
|------|----------|------|
| Framework | Spring Boot 3.x | 엔터프라이즈급 DI, 트랜잭션 관리 |
| Language | Java 17 | LTS, Record/Sealed 활용 |
| Database | PostgreSQL 15 | JSON 컬럼 지원, 부분 유니크 인덱스 |
| ORM | JPA + MyBatis 병행 | 복잡 쿼리(MyBatis), 엔티티 관리(JPA) |
| API | REST (JSON) | 단순성, 국세청 표준 준수 |
| Validation | Bean Validation + MX-03 | 71개 검증 규칙 계층 분리 |
| Security | Spring Security + JWT | req_id 토큰화, RBAC |
| Testing | JUnit 5 + AssertJ | 56개 공식 단위 테스트 |
| Logging | Logback + SLF4J | AL_S_CALCULATION 연동 |

### 2.2 Clean Architecture 패키지 구조

```
src/main/java/com/taxservice/entec/
├── presentation/                    # API 계층
│   ├── controller/
│   │   ├── RequestController.java       # API-01: 요청접수
│   │   ├── AnalyzeController.java       # API-02: 점검실행
│   │   ├── StatusController.java        # API-03: 상태조회
│   │   ├── ReportController.java        # API-04~06: 보고서
│   │   └── ReferenceController.java     # API-07: 기준정보
│   ├── dto/
│   │   ├── request/                     # 요청 DTO
│   │   └── response/                    # 응답 DTO
│   └── advice/
│       └── GlobalExceptionHandler.java  # 통합 오류 응답
│
├── application/                     # 유스케이스 계층
│   ├── service/
│   │   ├── RequestService.java          # 요청 접수 유스케이스
│   │   ├── AnalyzeService.java          # 분석 오케스트레이션
│   │   ├── ReportService.java           # 보고서 유스케이스
│   │   └── StatusService.java           # 상태 조회
│   ├── port/
│   │   ├── in/                          # 인바운드 포트 (인터페이스)
│   │   └── out/                         # 아웃바운드 포트 (인터페이스)
│   └── mapper/
│       └── DtoMapper.java               # DTO ↔ Domain 매핑
│
├── domain/                          # 도메인 계층 (핵심 비즈니스 로직)
│   ├── model/
│   │   ├── ReqId.java                   # Value Object
│   │   ├── TaxType.java                 # Enum: CORP / INC
│   │   ├── RequestStatus.java           # Enum: 요청 상태
│   │   ├── CreditItem.java             # 공제/감면 항목 도메인 모델
│   │   ├── Combination.java            # 조합 도메인 모델
│   │   └── RefundResult.java           # 환급 결과 모델
│   ├── repository/                      # Repository 인터페이스
│   │   ├── RequestRepository.java
│   │   ├── RawDataRepository.java
│   │   ├── SummaryRepository.java
│   │   ├── CreditRepository.java
│   │   └── ReferenceRepository.java
│   └── service/
│       ├── module/                      # M 모듈 (공통)
│       │   ├── m1/                      # M1 입력 관리
│       │   │   ├── RequestCreator.java      # M1-01
│       │   │   ├── RawDataStore.java        # M1-02
│       │   │   └── SummaryGenerator.java    # M1-03
│       │   ├── m2/                      # M2 기준정보
│       │   │   └── LawVersionMatcher.java   # MX-02
│       │   ├── m3/                      # M3 사전 점검
│       │   │   ├── PrepDataValidator.java   # M3-PREP
│       │   │   ├── ClaimValidator.java      # M3-00 ~ M3-01
│       │   │   ├── SmeEvaluator.java        # M3-02
│       │   │   ├── LocationEvaluator.java   # M3-03
│       │   │   ├── EmployeeCalculator.java  # M3-04
│       │   │   └── SettlementChecker.java   # M3-06
│       │   ├── m4/                      # M4 개별 공제/감면 산출
│       │   │   ├── IntegratedInvestmentCredit.java    # M4-01 §24
│       │   │   ├── IntegratedEmploymentCredit.java    # M4-02 §29의8
│       │   │   ├── SocialInsuranceCredit.java         # M4-03 §30의4
│       │   │   ├── StartupExemption.java              # M4-04 §6
│       │   │   ├── SmeSpecialExemption.java           # M4-05 §7
│       │   │   ├── RdCredit.java                      # M4-06 §10
│       │   │   ├── ForeignTaxCredit.java              # M4-07 §57(법인)
│       │   │   ├── DividendExclusion.java             # M4-09 §18의2
│       │   │   ├── LossCarryback.java                 # M4-10 §72
│       │   │   ├── TaxAdjustmentReview.java           # M4-11
│       │   │   ├── DepreciationReinstatement.java     # M4-2 (감가상각 추인)
│       │   │   ├── DisasterLossCredit.java            # M4-15 §58
│       │   │   └── CreditCalculatorRegistry.java      # 서브모듈 레지스트리
│       │   ├── m5/                      # M5 최적 조합
│       │   │   ├── ExclusionVerifier.java       # M5-01 상호배제
│       │   │   ├── CombinationOptimizer.java    # M5-02 조합 탐색
│       │   │   ├── MinimumTaxCalculator.java    # M5-03 최저한세
│       │   │   ├── AgriTaxCalculator.java       # M5-04 농특세
│       │   │   └── ApplicationOrderer.java      # M5-05 적용순서
│       │   ├── m6/                      # M6 최종 산출
│       │   │   ├── RefundCalculator.java        # M6-01 환급액
│       │   │   ├── InterestCalculator.java      # M6-02 환급가산금
│       │   │   ├── LocalTaxGuidance.java        # M6-05 지방소득세
│       │   │   └── ReportJsonSerializer.java    # M6-05 JSON 직렬화
│       │   └── mx/                      # MX 횡단 기능
│       │       ├── CircularRefResolver.java     # MX-01 순환참조
│       │       ├── LawVersionResolver.java      # MX-02 법령 매칭
│       │       └── ValidationEngine.java        # MX-03 검증 엔진
│       ├── inc/                         # P 모듈 (개인 전용)
│       │   ├── IncLocationEvaluator.java    # P3-01
│       │   ├── SincerityEvaluator.java      # P3-02
│       │   ├── ComprehensiveIncome.java     # P3-03
│       │   ├── IncDeductionOptimizer.java   # P4-01
│       │   ├── ExemptionAllocator.java      # P4-02
│       │   ├── JointBizAllocator.java       # P4-03
│       │   ├── SincerityMedEduCredit.java   # P4-04
│       │   ├── SincerityFeeCredit.java      # P4-05
│       │   ├── BookkeepingCredit.java       # P4-06
│       │   ├── IncForeignTaxCredit.java     # P4-07
│       │   ├── GoodLandlordCredit.java      # P4-08
│       │   └── IncMinimumTax.java           # P5-01
│       └── common/
│           ├── TruncationPolicy.java        # 절사 정책
│           └── TaxCalculationUtils.java     # 세액 계산 유틸
│
└── infrastructure/                  # 인프라 계층
    ├── persistence/
    │   ├── jpa/
    │   │   ├── entity/                  # JPA Entity (83개)
    │   │   └── repository/              # JpaRepository 구현
    │   └── mybatis/
    │       ├── mapper/                  # MyBatis Mapper 인터페이스
    │       └── xml/                     # MyBatis XML
    ├── config/
    │   ├── DataSourceConfig.java
    │   ├── SecurityConfig.java
    │   ├── JpaConfig.java
    │   └── MyBatisConfig.java
    └── common/
        ├── constants/
        │   ├── TaxConstants.java        # 세목/상태 상수
        │   └── ErrorCodes.java          # 오류 코드
        └── exception/
            ├── HardFailException.java
            ├── SoftFailException.java
            └── TaxCalculationException.java
```

### 2.3 3계층 데이터 아키텍처

```
┌────────────────────────────────────────────────────────────┐
│                  RI_S_REQUEST (요청 마스터)                   │
│             req_id = 신청인 + 날짜 기반 유니크 키             │
└──────────┬──────────────────────────────────┬──────────────┘
           │ FK: req_id                       │ FK: req_id
  ┌────────▼────────┐               ┌────────▼────────┐
  │   입력자료 계층    │               │   출력결과 계층    │
  │                  │               │                  │
  │ RI_S_RAW_DATA   │               │  CO_* 테이블     │
  │ (원시 JSON)      │               │  (산출 결과)      │
  │     │ 파싱       │               │     │ 직렬화     │
  │ SV_* 요약       │               │ CO_S_REPORT_JSON │
  │ (계산용)         │               │  (전달용 JSON)    │
  └──────────────────┘               └──────────────────┘
           │                                  │
           └──────► 점검/산출 엔진 ◄───────────┘
                    (M2~M5)
                    + RF_* 참조
```

### 2.4 TAX_TYPE 분기 매트릭스

| TAX_TYPE | 입력 (RI_) | 기준정보 (RF_) | 산출 (CO_) | 모듈 |
|----------|-----------|--------------|-----------|------|
| **CORP** | RI_C_* 17개 + RI_S_* 12개 | RF_C_* 7개 + RF_S_* 15개 | CO_S_* 8개 | M3-03(본점간주), M4-07~15, M5-03(과세표준기준) |
| **INC** | RI_I_* 8개 + RI_S_* 12개 | RF_I_* 4개 + RF_S_* 15개 | CO_S_* 8개 + CO_I_* 3개 | P3-01~04, P4-01~14, P5-01(산출세액기준) |

---

## 3. 모듈 상세 설계

### 3.1 모듈 계층 구조

```
M0 (분기) ─────────────────────────────────────────────────────
  │
  ├── M1 (입력 관리) ── TX-1 ──────────────────────────────────
  │   ├── M1-01: req_id 발급 + RI_S_REQUEST 생성
  │   ├── M1-02: 원시 JSON 수신 + RI_S_RAW_DATA 보관
  │   └── M1-03: 요약 테이블 생성 (SV_S_BASIC/EMPLOYEE/DEDUCTION/FINANCIAL)
  │
  ├── M3-PREP (데이터 준비) ── TX-2 시작 ──────────────────────
  │   ├── PREP-1: SV_* 완전성 검증
  │   ├── PREP-2: 적용 법령 버전 결정
  │   ├── PREP-3: 기준정보 사전 조회 (세율/공제율/한도 스냅샷)
  │   ├── PREP-4: 기초 데이터 사전 산정 (규모/소재지/고용 잠정값)
  │   ├── PREP-5: 적용 가능 공제/감면 후보 조항 도출
  │   └── PREP-6: SV_S_PREP_SUMMARY INSERT
  │
  ├── M3 (사전 점검) ── STEP 0 ────────────────────────────────
  │   ├── M3-00: 경정청구 불가 사유 사전 차단 (Hard Fail)
  │   ├── M3-01: 경정청구 기한 점검 (5년/3개월)
  │   ├── M3-02: 중소기업 해당 여부 (매출/독립성/졸업유예)
  │   ├── M3-03: 소재지 구분 (본점간주 vs 실제소재지)
  │   ├── M3-04: 상시근로자 수 산정 (제외/청년등/월별평균)
  │   ├── M3-05: 업종 적격성 판정
  │   └── M3-06: 결산확정 원칙 검증 (CORP)
  │   ├── P3-01: 소재지 사업장별 개별 판단 (INC)
  │   ├── P3-02: 성실신고확인대상 자동판정 (INC)
  │   ├── P3-03: 종합소득금액 산정 (INC)
  │   └── P3-04: 소득공제/과세표준/산출세액 산정 (INC)
  │
  ├── M4 (개별 공제/감면) ── STEP 1~2 ─────────────────────────
  │   ├── M4-01: 통합투자세액공제 (§24)
  │   ├── M4-02: 통합고용세액공제 (§29의8) + 경과규정(§29의7) 비교
  │   ├── M4-03: 사회보험료세액공제 (§30의4)
  │   ├── M4-04: 창업중소기업 세액감면 (§6)
  │   ├── M4-05: 중소기업 특별세액감면 (§7)
  │   ├── M4-06: 연구인력개발비 세액공제 (§10)
  │   ├── M4-07: 외국납부세액공제 (§57, CORP)
  │   ├── M4-09: 수입배당금 익금불산입 (§18의2, CORP)
  │   ├── M4-10: 결손금 소급공제 (§72/§85의2)
  │   ├── M4-11: 세무조정 재검토 (CORP)
  │   ├── M42:   감가상각 시부인 이월추인 (CORP)
  │   ├── M4-15: 재해손실세액공제 (§58, CORP)
  │   ├── P4-01: 소득공제 누락/세율구간 최적화 (INC)
  │   ├── P4-02: 감면소득 배분 계산 (INC, 다사업장)
  │   ├── P4-03: 공동사업자 소득 배분 (INC)
  │   ├── P4-04: 성실사업자 의료비/교육비 공제 (INC, 최저한세 비대상)
  │   ├── P4-05: 성실신고확인비용 공제 (INC, 최저한세 대상)
  │   ├── P4-06: 기장세액공제 (INC, 중복배제 비대상)
  │   ├── P4-07: 외국납부세액공제 (INC, §57)
  │   └── P4-08: 착한 임대인 세액공제 (INC, §96의3)
  │
  ├── M5 (최적 조합) ── STEP 3 ────────────────────────────────
  │   ├── M5-01: 상호배제 검증 (RF_S_MUTUAL_EXCLUSION)
  │   ├── M5-02: 최적 조합 탐색 (Branch & Bound / Greedy)
  │   ├── M5-03: 최저한세 적용 (CORP: 과세표준기준, INC: 산출세액기준)
  │   ├── M5-04: 농특세 적용 (항목별 비과세/과세 구분)
  │   ├── M5-05: 세액 적용순서 처리
  │   └── P5-01: 개인 최저한세 산출 (INC, 35%/45%)
  │
  ├── M6 (최종 산출) ── STEP 4~5 ──────────────────────────────
  │   ├── M6-01: 환급액 비교표 (기존 vs 경정후)
  │   ├── M6-02: 환급가산금 (본세/중간예납 기산일 분리)
  │   ├── M6-04: 보고서 3종 생성
  │   ├── M6-05: JSON 직렬화 (CO_S_REPORT_JSON, 7섹션)
  │   └── M6-06: 지방소득세 경정청구 안내
  │
  ├── M7 (시뮬레이션) ── Phase 2 ──────────────────────────────
  │   ├── M7-01: 다수 과세연도 시뮬레이션
  │   └── M7-03: 사후관리 리스크 평가
  │
  └── MX (횡단 기능) ──────────────────────────────────────────
      ├── MX-01: 순환참조 해결 (과세표준↔최저한세, 최대 5회 수렴)
      ├── MX-02: 법령 버전 자동 매칭
      └── MX-03: 검증 엔진 (83개 규칙)
```

### 3.2 M1 입력 관리 모듈

#### M1-01: 요청 접수 및 req_id 발급

| 항목 | 내용 |
|------|------|
| **입력** | applicant_id, taxpayer_id, tax_type, tax_year |
| **출력** | req_id, RI_S_REQUEST 1행 |
| **동시성** | SELECT ... FOR UPDATE + SERIALIZABLE 트랜잭션 |
| **재시도** | 최대 3회, 지수 백오프 (100ms x 2^n) |

```
처리 절차:
1. BEGIN TRANSACTION (ISOLATION = SERIALIZABLE)
2. SELECT MAX(seq_no) FROM RI_S_REQUEST
   WHERE applicant_id = ? AND request_date = CURRENT_DATE FOR UPDATE
3. seq_no = COALESCE(MAX, 0) + 1
4. req_id = {type}-{biz_no}-{YYYYMMDD}-{seq:03d}
5. INSERT INTO RI_S_REQUEST (req_id, ..., status='received')
6. COMMIT
```

#### M1-02: 원시 JSON 수신 및 보관

| 항목 | 내용 |
|------|------|
| **입력** | req_id, datasets[] |
| **출력** | RI_S_RAW_DATA N행 |
| **검증** | datasets <= 40개, 카테고리별 <= 10MB, 전체 <= 50MB |
| **필수 카테고리** | CORP: corp_basic / INC: inc_basic |

**32개 카테고리 코드**:
- 공통(8): employee_detail, employee_monthly, investment, startup, sme_special, rd_expense, existing_deduction, foreign_tax
- CORP 전용(16): corp_basic, representative, loss_carryforward, credit_carryforward, branch_location, interim_tax, dividend_income, business_vehicle, tax_adjustment, entertainment, government_subsidy, shareholder_loan, non_business_asset, disaster_loss, consolidated_sub, depreciation_adjust
- INC 전용(8): inc_basic, inc_business, inc_other_income, inc_deduction, inc_sincerity, inc_foreign_tax, inc_rental_reduction, inc_joint_biz

#### M1-03: 요약 테이블 생성 (5 PHASE)

```
PHASE 0: 재처리 판단 (version > 1 → 기존 SV_* DELETE 후 재생성)
PHASE 1: SV_S_BASIC 생성 (세목/규모 기초 → 최우선)
PHASE 2: SV_S_EMPLOYEE 생성 (PRIOR/CURRENT 2행, 제외대상 필터링)
PHASE 3: SV_S_DEDUCTION 생성 (N행, 카테고리별 파싱)
PHASE 4: SV_S_FINANCIAL 생성 (재무/세무 수치 집계)
PHASE 5: 완료 확인 및 상태 전이 (parsed)
```

**PHASE 3 카테고리-항목 매핑**:

| 카테고리 | item_category | provision | 집계 단위 |
|---------|---------------|-----------|----------|
| investment | INVEST | §24 | 자산유형 x 소재지별 그루핑 |
| rd_expense | RD | §10 | 연도 x 유형 x 방식별 (5개년) |
| startup | STARTUP | §6 | 요청당 1행 |
| sme_special | SME_SPECIAL | §7 | 요청당 1행 |
| existing_deduction | (역참조) | (동적) | 기존행 UPDATE |
| credit_carryforward | (역참조/신규) | (동적) | 조항 x 연도별 |
| inc_sincerity (INC) | INC_SINCERITY | §126의6 | 요청당 1행 |
| inc_rental_reduction (INC) | INC_LANDLORD | §99의9 | 요청당 1행 |

### 3.3 M3-PREP 데이터 준비 모듈

| 단계 | 처리내용 | 입력 | 출력 |
|------|---------|------|------|
| PREP-1 | SV_* 완전성 검증 | SV_S_BASIC/EMPLOYEE/DEDUCTION/FINANCIAL | 존재/무결성 판정 |
| PREP-2 | 적용 법령 버전 결정 | tax_year, RF_S_LAW_VERSION | law_version_id |
| PREP-3 | 기준정보 스냅샷 | RF_S_TAX_RATE, RF_S_MIN_TAX_RATE 등 | 세율/최저한세율/이율 |
| PREP-4 | 기초 데이터 산정 | SV_S_BASIC, SV_S_EMPLOYEE | 규모/소재지/고용 잠정값 |
| PREP-5 | 후보 조항 도출 | SV_S_DEDUCTION, SV_S_FINANCIAL | candidate_provisions |
| PREP-6 | SV_S_PREP_SUMMARY 생성 | 위 결과 종합 | 1행 INSERT |

**PREP 이후 모듈 활용**:
- M3-00: prep_status FAIL → TX-2 ROLLBACK
- M3-02/03: 잠정값에서 출발 → SV_S_ELIGIBILITY 확정
- **M4 전체**: candidate_provisions 목록에 없는 조항은 SKIP
- M5-04: refund_interest_rate 스냅샷 사용

### 3.4 M3 사전 점검 모듈

#### M3-00: 경정청구 불가 사유 사전 차단

| 검증 항목 | Hard/Soft | 조건 | 법령 |
|----------|:---------:|------|------|
| 기한 경과 | Hard | 법정신고기한 + 5년 초과 | 국기법 §45의2 |
| 분식회계 | Hard | 사실과 다른 회계처리 해당 | 국기법 §45의2④ |
| 결산조정 항목 (CORP) | Hard | 감가상각비/퇴직급여/대손충당금 추가 계상 | 법인세법 §40 |
| 추계신고 (INC) | Hard | 추계 → 감면/공제 배제 | 조특법 §128 |
| 직권경정 중 감면 제한 | Soft | 과소신고분 감면 제한 확인 | 조특법 §128③ |

#### M3-03: 소재지 구분 판단 (TAX_TYPE 분기)

| 적용 조항 | 판단 기준 | CORP | INC |
|----------|----------|------|-----|
| §7 (중소특별) | 본점 간주 | 본점=수도권 → 전사업장 수도권 | 사업장별 개별 판단 |
| §24 (투자) | 실제 설치장소 | 자산 소재지 기준 | 자산 소재지 기준 |
| §6 (창업) | 법인/사업장 소재지 | 법인 소재지 | 사업장 소재지 |
| §29의8 (고용) | 합산 (소재지 무관) | 법인 전체 | 전사업장 합산 |

#### M3-04: 상시근로자 수 산정

```
제외 대상:
  [CORP] 임원, 최대주주 4촌이내 혈족/인척, 일용직, 주 15시간 미만 단시간
  [INC]  대표자 본인, 배우자, 직계존비속

연평균 산정:
  annual_avg = TRUNCATE(SUM(월별인원) / 해당월수, 2)  -- 소수점 3자리 절사

청년 판단 시점 연도별 분기:
  2024 이전: 과세연도 말일 기준 (만 15~34세 + 병역가산 최대 6년)
  2025 이후: 근로계약 체결일 기준
```

### 3.5 M4 개별 공제/감면 모듈

#### 서브모듈 레지스트리 패턴

```java
public interface CreditCalculator {
    String getProvision();              // 적용 조항 (§24, §10 등)
    Set<TaxType> getSupportedTaxTypes(); // CORP, INC, or BOTH
    CreditResult calculate(CalculationContext ctx);
}

@Component
public class CreditCalculatorRegistry {
    private final Map<String, CreditCalculator> calculators;

    public List<CreditResult> calculateAll(
        CalculationContext ctx,
        Set<String> candidateProvisions  // M3-PREP에서 도출
    ) {
        return candidateProvisions.stream()
            .filter(p -> calculators.containsKey(p))
            .filter(p -> calculators.get(p).getSupportedTaxTypes().contains(ctx.getTaxType()))
            .map(p -> calculators.get(p).calculate(ctx))
            .collect(toList());
    }
}
```

#### 주요 서브모듈 설계

| 서브모듈 | 입력 | 출력 | 핵심 로직 |
|---------|------|------|----------|
| M4-01 §24 | SV_S_DEDUCTION(INVEST) | CO_S_CREDIT_DETAIL | 기본공제(투자 x 공제율) + 추가공제(당해-3년평균), 임대/중고/무형 배제 |
| M4-02 §29의8 | SV_S_EMPLOYEE | CO_S_CREDIT_DETAIL | 기본(청년등단가 x 증가분) + 추가(경력단절/정규직/육아), 2023~24 §29의7 비교 |
| M4-04 §6 | SV_S_DEDUCTION(STARTUP) | CO_S_CREDIT_DETAIL | 산출세액 x (감면소득/총소득) x 감면율, 인구감소지역 100%, 벤처 3년 |
| M4-05 §7 | SV_S_DEDUCTION(SME_SPECIAL) | CO_S_CREDIT_DETAIL | 업종/규모/소재지별 5~30%, 한도 MIN(감면, MAX(0, 1억-감소인원x500만)) |
| M4-06 §10 | SV_S_DEDUCTION(RD) | CO_S_CREDIT_DETAIL | MAX(증가분, 당기분), 유형별 3단계 최저한세 배제 |
| M4-07 §57 | SV_S_FINANCIAL | CO_S_CREDIT_DETAIL | 직접/간접/손금산입 3방식 비교, 한도=산출세액x(국외소득/과세표준) |
| P4-02 배분 | RI_I_BUSINESS | CO_I_INCOME_ALLOCATION | 감면세액=산출세액x(감면소득/종합소득)x감면율, 다사업장 개별합산 |
| P5-01 INC최저한세 | computed_tax | minimum_tax | 3천만이하 35%, 초과 45%, 비대상 항목 별도 적용 |

### 3.6 M5 최적 조합 모듈

#### M5-02: 조합 탐색 알고리즘

```
항목 수 판단 (RF_S_SYSTEM_PARAM.greedy_fallback_threshold = 15):

IF 항목수 <= 15:
  Branch & Bound (정밀 탐색)
ELSE:
  Greedy 폴백:
    (1) net_amount 내림차순 정렬
    (2) 상위 15개에 B&B 수행
    (3) 나머지는 상호배제 미충돌 시 Greedy 추가

각 조합에 대해:
  1. 적용순서 (법인§59/소득세§60): 감면 → 이월불가공제 → 이월가능공제
  2. 최저한세: 산출세액 - (과세표준 x 최저한세율) = 공제가능한도
  3. R&D 최저한세 배제: 국가전략(전액) > 신성장/중소(전액) > 일반/중소(50%)
  4. 감면 초과분 = 소멸, 공제 초과분 = 이월(10년)
  5. 농특세: 항목별 비과세(§6,§7,§10) / 과세 20%(§24,§29의8,§30의4)
  6. 실질환급액 = 총공제감면 - 농특세
  7. 순환참조 해결 (MX-01): 과세표준↔최저한세 최대 5회, 1원 수렴
```

#### M5-01: 상호배제 규칙

| 조항 A | 조항 B | 허용 | 조건 |
|--------|--------|:----:|------|
| §6 (창업) | §7 (중소특별) | X | 택1 |
| §6 (창업) | §29의8 (고용) | 조건부 | 2024 이전: 허용 / 2025 이후: 택1 |
| §7 (중소특별) | §29의8 (고용) | O | 병행 가능 |
| §6 (창업) | §24 (투자) | O | 병행 가능 |
| §7 (중소특별) | §24 (투자) | O | 병행 가능 |
| §10 (R&D) | 모든 항목 | O | 병행 가능 (최저한세만 조정) |

#### 최적 조합 시나리오

| 시나리오 | 감면 | 공제 | R&D | 적용 조건 |
|---------|------|------|-----|----------|
| 그룹A-1 | §6(창업) | §29의8(고용) | §10 | 2024 이전만 중복 가능 |
| 그룹A-2 | §7(중소특별) | §29의8(고용) | §10 | 3중 병행, 핵심 전략 |
| 그룹B-1 | - | §24+§29의8 | §10 | 투자액 큰 경우 |
| 그룹C | §6(창업) | - | §10 | 2025 이후 감면+R&D |

### 3.7 MX 횡단 기능

#### MX-01: 순환참조 해결

```
FOR i = 1 TO 5:
  과세표준(i) = 소득 - 이월결손금(i-1) - 공제액(i-1)
  최저한세(i) = 과세표준(i) x 최저한세율
  공제액(i) = MIN(산출세액 - 최저한세(i), 공제요청액)
  IF ABS(과세표준(i) - 과세표준(i-1)) <= 1원:
    수렴 완료 → BREAK

수렴 실패 시: WARNING 로그 + 현재까지 최선 결과 반환
```

#### MX-03: 검증 엔진 (83개 규칙)

| 유형 | 코드 | 건수 | 설명 | 실패 시 |
|------|-----|:----:|------|--------|
| V (Validation) | V01~V19 | 19 | 입력 형식/범위 검증 | ERROR |
| C (Calculation) | C01~C13 | 13 | 계산 결과 논리 검증 | ERROR |
| B (Business) | B01~B11 | 11 | 업무 규칙 준수 | ERROR |
| L (Legal) | L01~L09 | 9 | 법령 적합성 | ERROR |
| X (Cross) | X01~X13 | 13 | 교차 검증 (정합성) | ERROR |
| W (Warning) | W01~W06 | 6 | 경고 (Hard Fail 아님) | WARNING |
| VP/CP/BP/LP/WP (INC) | VP01~WP01 | 12 | 개인 전용 검증 | 혼합 |

---

## 4. REST API 설계

### 4.1 기본 사항

| 항목 | 내용 |
|------|------|
| 프로토콜 | HTTPS (TLS 1.2+) |
| 인증 | API Key (헤더: `X-API-Key`) |
| 형식 | JSON (Content-Type: application/json; charset=utf-8) |
| 버전 관리 | `/api/v1/...` |

### 4.2 엔드포인트 목록

| No | Method | Path | 설명 | 트랜잭션 |
|----|--------|------|------|---------|
| API-01 | POST | `/api/v1/requests` | 요청 접수 + 원시 JSON 수신 | TX-1 |
| API-02 | POST | `/api/v1/requests/{req_id}/analyze` | 점검 실행 (M3-PREP ~ M6) | TX-2 |
| API-03 | GET | `/api/v1/requests/{req_id}/status` | 요청 상태 조회 | 읽기 |
| API-04 | GET | `/api/v1/requests/{req_id}/report` | 전체 보고서 JSON | 읽기 |
| API-05 | GET | `/api/v1/requests/{req_id}/report/sections/{A-F}` | 섹션별 JSON | 읽기 |
| API-06 | GET | `/api/v1/requests/{req_id}/raw-data` | 원시 입력 JSON 조회 | 읽기 |
| API-07 | GET | `/api/v1/reference/exclusion-matrix` | 상호배제 기준정보 | 읽기 |
| API-08 | GET | `/api/v1/requests/{req_id}/summary` | 경량 요약 조회 | 읽기 |

### 4.3 API-01 요청/응답 스키마

**요청**:
```json
{
  "applicant_type": "C",
  "applicant_id": "123-45-67890",
  "tax_type": "CORP",
  "tax_year": "2024",
  "datasets": [
    { "category": "corp_basic", "data": { ... } },
    { "category": "employee_detail", "data": [ ... ] },
    { "category": "investment", "data": [ ... ] }
  ]
}
```

**성공 응답 (201)**:
```json
{
  "req_id": "C-1234567890-20260216-001",
  "status": "received",
  "datasets_received": 3,
  "created_at": "2026-02-16T14:00:00+09:00"
}
```

**Idempotency-Key**: (Idempotency-Key, applicant_id, tax_year) 조합으로 중복 체크, TTL 24시간

### 4.4 표준 오류 응답

```json
{
  "error": {
    "code": "ERR_VALIDATION_FAILED",
    "message": "필수 카테고리 'corp_basic'이 누락되었습니다.",
    "details": [{ "field": "...", "issue": "...", "expected": "...", "received": null }],
    "req_id": "C-1234567890-20260216-001",
    "timestamp": "2026-02-16T14:00:05+09:00",
    "trace_id": "abc-123-def-456"
  }
}
```

| HTTP 상태 | 오류 코드 | 발생 시점 |
|-----------|----------|----------|
| 400 | ERR_INVALID_JSON | API-01 M1-02 |
| 400 | ERR_VALIDATION_FAILED | API-01 M1-02 |
| 404 | ERR_REQUEST_NOT_FOUND | API-02~06 |
| 409 | ERR_INVALID_STATUS | API-02 |
| 422 | ERR_HARD_FAIL | API-02 M3-00 |
| 422 | ERR_CALCULATION_FAILED | API-02 M4~M6 |
| 500 | ERR_INTERNAL | 전체 |
| 504 | ERR_TIMEOUT | API-02 M5 (combo_timeout 초과) |

---

## 5. 데이터 처리 파이프라인

### 5.1 전체 파이프라인

```
① API-01 요청접수 ═══ [TX-1: 요청 트랜잭션] ═══
   └→ M1-01: req_id 발급 → RI_S_REQUEST
   └→ M1-02: 원시 JSON 보관 → RI_S_RAW_DATA
   └→ M1-03: 요약 생성 → SV_S_BASIC / EMPLOYEE / DEDUCTION / FINANCIAL
   └→ COMMIT TX-1

② API-02 점검실행 ═══ [TX-2: 산출 트랜잭션] ═══
   └→ PREP: 데이터 준비 → SV_S_PREP_SUMMARY
   └→ STEP 0: 사전 점검 → SV_S_ELIGIBILITY, SV_S_INSPECTION_LOG
   └→ STEP 1~2: 개별 산출 → CO_S_CREDIT_DETAIL
   └→ STEP 3: 최적 조합 → CO_S_COMBINATION, CO_S_EXCLUSION_VERIFY
   └→ STEP 4: 환급액 산출 → CO_S_REFUND, CO_S_RISK
   └→ STEP 5: JSON 직렬화 → CO_S_REPORT_JSON
   └→ COMMIT TX-2

③ API-04 결과조회 ═══ [읽기 전용] ═══
   └→ CO_S_REPORT_JSON → JSON 응답 반환
```

### 5.2 상태 전이 (request_status)

```
received → parsing → parsed → preparing → checking → calculating → optimizing → reporting → completed
                                                                                              │
                                 Any state ───(TX ROLLBACK)──→ error ──────────────────────────┘
```

| 전이 시점 | 이전 | 이후 | 트리거 |
|-----------|------|------|--------|
| API-01 접수 | - | received | M1-01 INSERT |
| M1-02 시작 | received | parsing | TX-1 시작 |
| M1-03 완료 | parsing | parsed | TX-1 COMMIT |
| API-02 시작 | parsed | preparing | TX-2 시작, M3-PREP |
| M3-PREP 완료 | preparing | checking | M3-00 시작 |
| M3 완료 | checking | calculating | M4 시작 |
| M4 완료 | calculating | optimizing | M5 시작 |
| M5 완료 | optimizing | reporting | M6 시작 |
| M6 완료 | reporting | completed | TX-2 COMMIT |

### 5.3 모듈별 입출력 매핑

| 모듈 | 입력 테이블 | 출력 테이블 | 참조 테이블 |
|------|-----------|-----------|-----------|
| M1-03 | RI_S_RAW_DATA | SV_S_BASIC, SV_S_EMPLOYEE, SV_S_DEDUCTION, SV_S_FINANCIAL | - |
| M3-PREP | SV_* 4개 | SV_S_PREP_SUMMARY | RF_S_LAW_VERSION, RF_S_TAX_RATE |
| M3-00~06 | SV_S_BASIC, SV_S_PREP_SUMMARY | SV_S_ELIGIBILITY, SV_S_INSPECTION_LOG, SV_S_VALIDATION_LOG | RF_S_CAPITAL_ZONE, RF_S_INDUSTRY_ELIGIBILITY |
| M4-01~15 | SV_S_DEDUCTION, SV_S_EMPLOYEE | CO_S_CREDIT_DETAIL | RF_C_INVESTMENT_CREDIT_RATE, RF_C_RD_CREDIT_RATE 등 |
| M5-01~05 | CO_S_CREDIT_DETAIL | CO_S_COMBINATION, CO_S_EXCLUSION_VERIFY | RF_S_MUTUAL_EXCLUSION, RF_S_NONGTEUKSE |
| M6-01~05 | CO_S_COMBINATION | CO_S_REFUND, CO_S_RISK, CO_S_REPORT_JSON | RF_S_REFUND_INTEREST_RATE |

---

## 6. 계산 엔진 설계

### 6.1 법인세 계산 공식 (F01~F34)

#### 과세표준 및 산출세액

| 공식ID | 공식명 | 산출 공식 | 절사 | 근거법령 |
|--------|--------|----------|------|---------|
| F01 | 과세표준 | 각사업연도소득 - 이월결손금 - 비과세소득 | 10원절사 | 법인세법 §13 |
| F02 | 이월결손금 공제한도 | MIN(잔액, 소득 x 한도율) [중소 100%, 기타 80%] | - | 법인세법 §13 |
| F03 | 산출세액 | 과세표준 x 초과누진세율 | 10원절사 | 법인세법 §55 |
| F04 | 최저한세 | 과세표준 x 최저한세율 [중소 7%, 중견 10~17%] | 10원절사 | 조특법 §132 |

#### 법인세율 (2023~2025)

| 과세표준 구간 | 세율 | 누진공제 |
|-------------|------|---------|
| 2억 이하 | 9% | 0 |
| 2억~200억 | 19% | 2,000만 |
| 200억~3,000억 | 21% | 42,000만 |
| 3,000억 초과 | 24% | 942,000만 |

#### 공제/감면 세액

| 공식ID | 공식명 | 산출 공식 | 근거 |
|--------|--------|----------|------|
| F10 | 통합고용 기본 | (청년등증가 x 단가) + (일반증가 x 단가) | §29의8 |
| F12 | 통합투자 기본 | 투자금액 x 기본공제율 | §24 |
| F13 | 통합투자 추가 | MAX(0, 당해-직전3년평균) x 추가율 | §24 |
| F14 | 창업감면 | 산출세액 x (감면소득/총소득) x 감면율 | §6 |
| F15 | 중소특별감면 | 산출세액 x (감면소득/총소득) x 감면율 | §7 |
| F17 | R&D 증가분 | (해당R&D비 - 직전R&D비) x 공제율 | §10 |
| F18 | R&D 당기분 | 해당R&D비 x 공제율 | §10 |

#### 최종 환급액

| 공식ID | 공식명 | 산출 공식 | 절사 |
|--------|--------|----------|------|
| F30 | 법인세환급액 | 기존납부세액 - 경정후결정세액 | - |
| F31 | 환급가산금(본세) | 환급액 x 이율 x 일수 / 365 | 1원절사 |
| F32 | 환급가산금(중간예납) | 중간예납과납 x 이율 x 일수 / 365 | 1원절사 |
| F33 | 지방소득세환급 | 법인세환급액 x 10% | 10원절사 |
| F34 | 총수령예상액 | F30 + F31 + F32 + F33 | - |

#### 농특세

| 공식ID | 공식명 | 비과세 항목 | 과세 항목 |
|--------|--------|-----------|---------|
| F40 | 농특세 | §6(창업), §7(중소특별), §10(R&D) | §24(투자), §29의8(고용), §30의4(사보) |
| F41 | 농특세 변동 | 경정후 - 기존 | 조합 변경 시 연쇄 재계산 |

### 6.2 종합소득세 계산 공식 (F-INC-01~10)

| 공식ID | 공식명 | 산출 공식 | 근거 |
|--------|--------|----------|------|
| F-INC-01 | 종합소득금액 | 사업소득 + 근로소득 + 금융소득(2천만초과시) + 연금 + 기타 | 소득세법 §14 |
| F-INC-02 | 과세표준 | 종합소득금액 - 소득공제합계 | 소득세법 §14 |
| F-INC-03 | 산출세액 | 과세표준 x 초과누진세율 (6~45% 8단계) | 소득세법 §55 |
| F-INC-04 | 감면소득배분 | 산출세액 x (감면대상소득 / 종합소득금액) x 감면율 | 시행령 §117 |
| F-INC-05 | 공동사업자 배분 | 전체소득 x 손익분배비율 | 소득세법 §43 |
| F-INC-06 | 개인 최저한세 | 3천만이하: 산출세액 x 35% / 초과: 45% | 조특법 §132② |

#### 종합소득세율 (2023년~)

| 과세표준 구간 | 세율 | 누진공제 |
|-------------|------|---------|
| 1,400만 이하 | 6% | 0 |
| 1,400만~5,000만 | 15% | 126만 |
| 5,000만~8,800만 | 24% | 576만 |
| 8,800만~1.5억 | 35% | 1,544만 |
| 1.5억~3억 | 38% | 1,994만 |
| 3억~5억 | 40% | 2,594만 |
| 5억~10억 | 42% | 3,594만 |
| 10억 초과 | 45% | 6,594만 |

### 6.3 절사(Truncation) 정책

| 대상 | 절사 규칙 | 금지 사항 |
|------|----------|----------|
| 금액 | 10원 미만 TRUNCATE | 반올림 절대 금지 |
| 비율 | 소수점 3자리 TRUNCATE | |
| 환급가산금 | 1원 미만 TRUNCATE | |
| 근로자 수 | 소수점 2자리 (3자리 절사) | |

```java
public final class TruncationPolicy {
    public static long truncateAmount(long amount) {
        return (amount / 10) * 10;  // 10원 미만 절사
    }
    public static double truncateRate(double rate) {
        return Math.floor(rate * 1000) / 1000;  // 소수점 3자리
    }
    public static long truncateInterest(long interest) {
        return interest;  // 1원 미만 절사 (long이므로 자동)
    }
    public static double truncateEmployee(double count) {
        return Math.floor(count * 100) / 100;  // 소수점 2자리
    }
}
```

---

## 7. 검증 엔진 설계

### 7.1 검증 규칙 분류 (83개)

#### 공통 검증 (71개)

**V (Validation) - 19개**: 입력 데이터 형식/범위

| 코드 | 설명 | 실패 시 |
|------|------|--------|
| V01 | 경정청구 기한 검증 (5년 이내) | Hard Fail |
| V02 | req_id 형식 검증 | Hard Fail |
| V03 | tax_type 유효값 (CORP/INC) | Hard Fail |
| V04 | 필수 필드 NULL 검증 | Hard Fail |
| V-RD-03 | R&D method 유효성 (INCREMENTAL/CURRENT) | Hard Fail |

**C (Calculation) - 13개**: 계산 결과 논리 검증

| 코드 | 설명 |
|------|------|
| C01 | 결산조정 항목 차단 (CORP) |
| C02 | 산출세액 >= 0 확인 |
| C03 | 공제액 <= 산출세액 확인 |

**B (Business) - 11개**: 업무 규칙

| 코드 | 설명 |
|------|------|
| B01 | 상호배제 규칙 (2025~ §6+§29의8 중복배제) |
| B02 | 최저한세 한도 초과 → 이월공제 적용 |

**L (Legal) - 9개**: 법령 적합성

| 코드 | 설명 |
|------|------|
| L01 | 최저한세 한도 검증 |
| L02 | 적용순서 준수 (법인§59/소득세§60) |

**X (Cross) - 13개**: 교차 검증

| 코드 | 설명 |
|------|------|
| X01 | 종합소득금액 분모 검증 (금융소득 2천만 초과 시) |
| X02 | 공동사업자 지분율 합계 = 100% |
| X14~X17 | JSON 보고서 스키마 검증 |

**W (Warning) - 6개**: 경고

| 코드 | 설명 |
|------|------|
| W01 | 순환참조 미수렴 (5회 초과) |
| W02 | 재창업 리스크 (동일업종 5년 이내) |

#### 개인 전용 검증 (12개)

| 코드 | 유형 | 설명 |
|------|------|------|
| VP01 | Validation | TAX_TYPE='INC' 시 RI_I_BASIC 필수 |
| VP02 | Validation | 다사업장 시 RI_I_BUSINESS 최소 1건 |
| VP03 | Validation | 공동사업자 손익분배비율 합계 = 100% |
| CP01 | Calculation | 감면세액 <= 산출세액 x 배분비율 x 감면율 |
| CP02 | Calculation | 소득공제 후 과세표준 >= 0 |
| CP03 | Calculation | 개인 최저한세 산출 검증 |
| BP01 | Business | 기장세액공제 + 조특법 감면 중복 허용 확인 |
| BP02 | Business | 성실신고확인비용 = 최저한세 대상 |
| BP03 | Business | 성실사업자 의료비/교육비 = 최저한세 비대상 |
| LP01 | Legal | 개인 적용순서 = 소득세법 §60 |
| LP02 | Legal | 개인 경정청구 기산일 = 5.31 (성실: 6.30) |
| WP01 | Warning | 분리과세 소득 제외 확인 |

### 7.2 검증 엔진 아키텍처

```java
public interface ValidationRule {
    String getCode();                    // V01, C01 등
    String getType();                    // V/C/B/L/X/W
    Set<TaxType> getApplicableTaxTypes();
    ValidationResult validate(ValidationContext ctx);
}

public enum ValidationResult {
    PASS, FAIL, WARNING, SKIP
}

@Component
public class ValidationEngine {
    private final List<ValidationRule> rules;  // 83개 규칙

    public List<ValidationLog> executeAll(String reqId, ValidationContext ctx) {
        return rules.stream()
            .filter(r -> r.getApplicableTaxTypes().contains(ctx.getTaxType()))
            .map(r -> {
                ValidationResult result = r.validate(ctx);
                return new ValidationLog(reqId, r.getCode(), r.getType(), result);
            })
            .collect(toList());
    }
}
```

---

## 8. 보고서 생성 설계

### 8.1 JSON 보고서 7섹션 구조

| 섹션 | 컬럼 | 소스 테이블 | 설명 |
|------|------|-----------|------|
| A | section_a_json | SV_S_ELIGIBILITY + SV_S_BASIC | 해당가능성 진단 |
| B | section_b_json | CO_S_CREDIT_DETAIL | 후보 항목 리스트 |
| C | section_c_json | CO_S_COMBINATION + CO_S_EXCLUSION_VERIFY | 최적조합/배제 |
| D | section_d_json | CO_S_REFUND | 환급/추징 금액 |
| E | section_e_json | SV_S_INSPECTION_LOG | 산출 진행 로그 |
| F | section_f_json | CO_S_RISK + CO_S_ADDITIONAL_CHECK | 주의/사후관리 |
| G | section_g_meta | report_meta + taxpayer_info | 제출 데이터/메타 |

### 8.2 보고서 3종

| 보고서 유형 | 대상 | 포함 내용 |
|------------|------|----------|
| 경영진용 | 경영진 | 환급예상액, 최적조합, 핵심 변경, 리스크 요약 |
| 세무대리인용 | 세무담당자 | 전체 산출 과정, 계산식, 근거법령, 조합별 비교 |
| 국세청용 | 과세관청 | 경정청구서 양식, 첨부서류 목록, 근거서류 |

### 8.3 보고서 필수 포함 섹션

1. 환급액 요약표 (기존 vs 경정후)
2. 최적 조합 상세 (적용 조항, 공제/감면액)
3. 최저한세 영향 분석 (R&D 배제 상세)
4. 환급가산금 산출 내역 (본세/중간예납 분리)
5. 지방소득세 경정청구 안내
6. 농특세 변동 내역
7. 이월공제 잔액 및 향후 활용 계획
8. 사후관리 리스크 평가 (추징 예상액, 의무기간)
9. 필요 증빙서류 목록
10. 추가 확인 필요 항목

---

## 9. 트랜잭션 및 동시성 설계

### 9.1 트랜잭션 경계

| TX ID | 범위 | 격리수준 | 타임아웃 | 성공 시 | 실패 시 |
|-------|------|---------|---------|--------|--------|
| TX-1 | M1-01 ~ M1-03 | READ COMMITTED | 60초 | COMMIT → parsed | ROLLBACK → error |
| TX-2 | M3-PREP ~ M6-05 | READ COMMITTED | 300초 | COMMIT → completed | ROLLBACK → error |

**설계 원칙**:
1. TX-1과 TX-2 분리 (별도 HTTP 요청)
2. All-or-Nothing: TX-2 내 부분 성공 불허
3. 상태 기록은 TX 외부 (실패 시 error 기록은 별도 커넥션)
4. 재처리 지원: TX-2 실패 후 재실행 시 기존 데이터 DELETE 후 재생성

### 9.2 동시성 제어

| 대상 | 방식 | 설명 |
|------|------|------|
| req_id 발급 | SELECT FOR UPDATE | 동일 (applicant_id, request_date) 행 수준 락 |
| seq_no 충돌 | UNIQUE INDEX | idx_applicant_date (applicant_id, request_date, seq_no) |
| 데드락 | 재시도 | 최대 3회, 지수 백오프 100ms x 2^n |
| 배치 처리 | 단일 TX | request_source='batch' 시 다건 seq_no 일괄 발급 |

### 9.3 성능 제한 사양

| 항목 | 제한값 | 초과 시 |
|------|-------|--------|
| 카테고리별 JSON 크기 | 10 MB | ERR_VALIDATION_FAILED |
| 최대 카테고리 수 | 40개/요청 | ERR_VALIDATION_FAILED |
| 전체 페이로드 크기 | 50 MB | HTTP 413 |
| employee_detail 최대 행 수 | 10,000건 | 비동기 처리 전환 |
| M5 조합 탐색 타임아웃 | 120초 | ERR_TIMEOUT + 현재까지 최선 반환 |
| TX-1 타임아웃 | 60초 | ROLLBACK |
| TX-2 타임아웃 | 300초 | ROLLBACK |
| 보고서 JSON 최대 크기 | 20 MB | 섹션별 분리 반환 권고 |

---

## 10. 보안 및 데이터 보호

### 10.1 보안 요건

| 항목 | 방식 |
|------|------|
| API 인증 | API Key (X-API-Key 헤더) |
| 개인정보 암호화 | AES-256 (주민번호, 사업자번호) |
| req_id 보호 | JWT 토큰화 (외부 노출 금지) |
| 통신 보안 | HTTPS TLS 1.2+ |
| 데이터 보관 | 90일 자동 삭제 |
| 로그 마스킹 | 사업자번호/대표자 식별정보 마스킹 |

### 10.2 권장 사항 (Phase 2)

- mTLS 또는 OAuth2 Client Credentials 병행
- RBAC (Role-Based Access Control) 구현
- 감사 로그 장기 보관 (별도 스토리지)

---

## 11. 구현 순서 및 의존성

### 11.1 Phase 1 (MVP - 6주)

#### Week 1: 환경 설정 & 데이터 모델링

- [ ] Spring Boot 3.x 프로젝트 초기화
- [ ] PostgreSQL 15 설치 및 DDL 생성 (83개 테이블)
- [ ] JPA Entity 클래스 작성 (핵심 테이블 우선)
- [ ] RF_* 기준정보 시드 데이터 적재
- [ ] req_id 생성 로직 + 동시성 테스트

#### Week 2: 입력 관리 (M1) + API-01

- [ ] POST /api/v1/requests 구현
- [ ] M1-01: req_id 발급 (SELECT FOR UPDATE)
- [ ] M1-02: RI_S_RAW_DATA INSERT ONLY 제약
- [ ] M1-03: SV_* 파싱 로직 (5 PHASE)
- [ ] 법인/개인 스키마 분기 테스트

#### Week 3: 사전 점검 (M3) + API-02

- [ ] M3-PREP: SV_S_PREP_SUMMARY 생성
- [ ] M3-00: Hard Fail 필터 (결산조정/기한경과/추계)
- [ ] M3-01~M3-04: 기한/중소/소재지/상시근로자
- [ ] P3-01~P3-04: 개인 전용 (소재지/성실신고/종합소득)
- [ ] MX-03 검증 엔진 (50개 규칙 우선)

#### Week 4: 개별 공제/감면 (M4)

- [ ] M4-04: 창업감면 (§6)
- [ ] M4-05: 중소기업감면 (§7)
- [ ] M4-02: 통합고용 (§29의8) + 경과규정 비교
- [ ] M4-01: 통합투자 (§24)
- [ ] M4-06: R&D (§10) + 최저한세 배제 3단계
- [ ] P4-02: 감면소득배분, P4-04: 의료비/교육비, P4-06: 기장세액

#### Week 5: 최적 조합 (M5)

- [ ] M5-01: 상호배제 검증
- [ ] M5-02: B&B / Greedy 조합 탐색
- [ ] M5-03: 최저한세 (CORP: 과세표준기준, INC: 산출세액기준)
- [ ] M5-04: 농특세 (항목별 비과세/과세)
- [ ] MX-01: 순환참조 해결 (최대 5회)

#### Week 6: 최종 산출 & 통합 테스트

- [ ] M6-01: 환급액 비교표
- [ ] M6-02: 환급가산금 (본세/중간예납 분리)
- [ ] M6-05: JSON 직렬화 (7섹션)
- [ ] 보고서 3종 생성
- [ ] 통합 테스트 (법인 10건, 개인 10건)

### 11.2 Phase 2 (고도화 - 4주)

- M4-07: 외국납부세액 (직접/간접/손금산입 비교)
- M4-09: 수입배당금 익금불산입 (2023 개정법)
- M4-10: 결손금 소급공제 (NPV 비교)
- M4-11: 세무조정 재검토 (국고보조금/가지급금/업무용승용차)
- M4-15: 재해손실세액공제
- M7-01: 다수 과세연도 시뮬레이션
- Excel/PDF 파서

### 11.3 모듈 의존성 그래프

```
M1 (입력) → M3-PREP (준비) → M3 (점검) → M4 (개별산출) → M5 (최적조합) → M6 (최종산출)
     │                           │              │                │              │
     └── RF_* (기준정보) ──────────┘──────────────┘────────────────┘──────────────┘
                                  │
                              MX-01 (순환참조)
                              MX-02 (법령매칭)
                              MX-03 (검증엔진)
```

---

## 12. 부록: 핵심 의사코드

### 12.1 최적 조합 탐색 (M5-02)

```
FUNCTION find_optimal_combination(individual_credits):

  // 1. 상호배제 규칙 로드
  exclusion_rules = RF_S_MUTUAL_EXCLUSION.load()
  IF target_tax_year >= 2025:
    exclusion_rules.deny('§6', '§29의8')

  // 2. 알고리즘 선택 (항목수 기반)
  IF individual_credits.length <= 15:
    candidates = Branch_and_Bound(individual_credits, exclusion_rules)
  ELSE:
    candidates = Greedy_with_partial_BnB(individual_credits, exclusion_rules)

  // 3. 각 조합 평가
  FOR EACH combo IN candidates:
    // 3-1. 적용순서 (법인§59 / 소득세§60)
    step1 = apply_exemptions(combo.exemptions)      // 감면
    step2 = apply_non_carryover(combo.credits_nc)    // 이월불가공제
    step3 = apply_carryover(combo.credits_cf)        // 이월가능공제

    // 3-2. 최저한세
    min_tax = computed_tax * min_tax_rate(corp_size)
    max_deduction = computed_tax - min_tax

    // 3-3. R&D 최저한세 배제 3단계
    rd_exempt = 국가전략(전액) + 신성장중소(전액) + 일반중소(50%)

    // 3-4. 감면 초과 = 소멸, 공제 초과 = 이월(10년)
    // 3-5. 농특세 (항목별 차등)
    combo.nongteuk = calc_nongteuk_by_item(combo)

    // 3-6. 실질 환급액
    combo.net_refund = combo.total_applied - combo.nongteuk

  // 4. 순환참조 해결 (MX-01)
  FOR EACH combo IN top_candidates:
    combo = resolve_circular_reference(combo, max_iterations=5)

  RETURN candidates.sort_by(net_refund DESC)
```

### 12.2 환급가산금 산출 (M6-02)

```
FUNCTION calc_refund_interest(refund_amount, payment_date):

  // 본세 환급가산금
  main_start = payment_date + 1일        // 기산일: 납부일 다음날
  main_end = refund_decision_date         // 종료일: 지급결정일
  main_rate = RF_S_REFUND_INTEREST_RATE.get(main_start, main_end)
  main_interest = TRUNCATE(refund_amount * main_rate * days / 365, 0)

  // 중간예납 환급가산금 (별도 기산)
  FOR EACH interim IN RI_C_INTERIM_TAX:
    int_start = interim.payment_date + 1일
    int_rate = RF_S_REFUND_INTEREST_RATE.get(int_start, decision_date)
    interim_interest += TRUNCATE(interim.overpaid * int_rate * days / 365, 0)

  RETURN { main_interest, interim_interest, total: main + interim }
```

### 12.3 개인 감면 소득 배분 (P4-02)

```
FUNCTION calculate_inc_exemption_allocation(computed_tax, comprehensive_income):

  total_exemption = 0
  FOR EACH biz IN RI_I_BUSINESS WHERE exempt_eligible = TRUE:

    // 배분비율
    income_ratio = TRUNCATE(biz.biz_income / comprehensive_income, 4)

    // 감면세액 = 산출세액 x 배분비율 x 감면율
    exemption = TRUNCATE(computed_tax * income_ratio * rate, -1)

    // 공동사업자
    IF is_joint_biz:
      exemption = TRUNCATE(exemption * joint.ratio / 100, -1)

    total_exemption += exemption

  // 한도 적용 (§7: 1억원)
  IF provision = '§7':
    total_exemption = MIN(total_exemption, 100,000,000)

  RETURN total_exemption
```

### 12.4 개인 최저한세 (P5-01)

```
FUNCTION calculate_inc_minimum_tax(computed_tax):
  threshold = 30,000,000

  IF computed_tax <= threshold:
    min_tax = TRUNCATE(computed_tax * 0.35, -1)
  ELSE:
    below = TRUNCATE(threshold * 0.35, -1)
    above = TRUNCATE((computed_tax - threshold) * 0.45, -1)
    min_tax = below + above

  max_deductible = computed_tax - min_tax

  // 최저한세 비대상 (별도 적용):
  //   성실사업자 의료비/교육비 (§122의3)
  //   기장세액공제 (소득세법 §56의2)
  //   외국납부세액공제 (소득세법 §57)
  //   자녀세액공제, 연금저축세액공제

  // 최저한세 대상:
  //   §6, §7, §24, §29의8, §30의4 등 조특법 감면/공제
  //   성실신고확인비용 (§126의6)
  //   착한 임대인 (§96의3)

  RETURN { min_tax, max_deductible }
```

---

## 버전 이력

| 버전 | 일자 | 변경 내용 | 작성자 |
|------|------|----------|--------|
| v1.0 | 2026-02-17 | 초안 작성 - 전체 아키텍처, 모듈설계, API, 계산/검증 엔진, 구현순서 | Development Design Agent |

---

## 참조 문서

| 문서 | 위치 | 설명 |
|------|------|------|
| 요구사항 계획서 | `docs/01-plan/features/tax-refund-system.plan.md` | PM Agent 작성, 55개 FR, 6 Epic |
| 용어 정의서 | `old-doc/01-glossary.md` | 9개 섹션, 세법/시스템 용어 |
| 엔티티 스키마 | `old-doc/02-entity-schema.md` | 83개 테이블 + 뷰 1개 상세 컬럼 |
| 엔티티 관계 | `old-doc/03-entity-relationship.md` | ERD, 데이터 흐름, TAX_TYPE 분기 |
| 초안 설계서 | `외부자료_v4_0.md` | v4.0 원본 상세 설계 (373KB) |
| 법인사업자 프롬프트 | `법인사업자-프롬프트_v1.3.md` | 40개 점검항목 |
| 개인사업자 프롬프트 | `개인사업자-프롬프트_v1.3.md` | 37개 점검항목 |
