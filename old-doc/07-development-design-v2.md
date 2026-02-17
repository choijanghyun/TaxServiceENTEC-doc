# 통합 경정청구 환급액 산출 시스템 - 개발설계서 v2.0

> **문서 버전**: v2.0
> **작성일**: 2026-02-17
> **기반 문서**:
>   - 요구사항 계획서: `docs/01-plan/features/tax-refund-system.plan.md` (v4.0, PM Agent)
>   - 상세설계서 초안: `docs/02-design/features/환급액계산프로그램.design.md` (v2.0, Design Agent)
>   - 외부자료 원본: `외부자료_v4_0.md` (DBMS/API/모듈 구조)
>   - 엔티티 스키마: `old-doc/02-entity-schema.md` (83개 테이블)
>   - 엔티티 관계: `old-doc/03-entity-relationship.md` (ERD/데이터흐름)
>   - 용어 정의: `old-doc/01-glossary.md` (9개 섹션)
>   - Gap 분석: `old-doc/06-gap-analysis-prompt-coverage.md` (34개 Gap 식별)
>   - 법인사업자 프롬프트: `법인사업자-프롬프트_v1.3.md` (40개 점검항목)
>   - 개인사업자 프롬프트: `개인사업자-프롬프트_v1.3.md` (37개 점검항목)
> **프로젝트 레벨**: Enterprise (Clean Architecture)
> **변경 이력**: 하단 [버전 이력](#14-버전-이력) 참조

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
12. [테스트 계획](#12-테스트-계획)
13. [부록: 핵심 의사코드](#13-부록-핵심-의사코드)
14. [버전 이력](#14-버전-이력)

---

## 1. 시스템 개요

### 1.1 목적

법인사업자(법인세, CORP)와 개인사업자(종합소득세, INC)의 경정청구 시 환급액을 극대화하는 AI 기반 통합 점검 시스템.

- 법인사업자 40개 점검항목 + 개인사업자 37개 점검항목 = **총 77개 점검항목** 완전 커버
- 4단계 점검 절차 자동화: 사전확인 -> 전제조건 -> 공제감면 산출 -> 최종산출
- 그룹별 비교분석(감면 중심 vs 공제 중심) 기반 최적 환급 전략 제시

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
| **적용순서** | 감면 -> 이월불가 공제 -> 이월가능 공제 (법인세법 SS59, 소득세법 SS60) |

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

### 1.5 v1.0 -> v2.0 주요 변경사항

> Gap 분석(06-gap-analysis-prompt-coverage.md)에서 식별된 34개 Gap(8 Critical, 14 Major, 12 Minor) 전수 반영.

| 변경 영역 | v1.0 | v2.0 | Gap 대응 |
|----------|------|------|---------|
| M4 서브모듈 수 | 14개 | **22개** | C-01, C-02 (개인 전용 6개 모듈 추가) |
| 계산 공식 수 | F01~F41 + F-INC-01~06 | F01~F41 + **F-INC-01~11** | C-03 (5개 공식 추가) |
| 검증 규칙 수 | 71개 공통 + 12개 INC | **83개 공통 + 24개 INC** = **107개** | C-04, C-08, M-08, M-09 |
| 세율 테이블 | 2023~2025만 | **2018~2025** (경정청구 5년 범위) | C-06, C-07 |
| 상호배제 매트릭스 | 6규칙 | **10규칙** (SS30의4 관련 4건 추가) | M-11 |
| 이월결손금 모듈 | M4-10만 | M4-10 + **M4-08**(이월결손금 재검토) | C-05 |
| 입력 스키마 | 필드 미명시 | **JSON 스키마 상세** (개인/법인) | m-01 |
| 보고서 | 7섹션 | **8섹션** (section_h 감사로그 추가) | m-02, m-03 |

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
| Validation | Bean Validation + MX-03 | 107개 검증 규칙 계층 분리 |
| Security | Spring Security + JWT | req_id 토큰화, RBAC |
| Testing | JUnit 5 + AssertJ | 56개 공식 단위 테스트 |
| Logging | Logback + SLF4J | AL_S_CALCULATION 연동 |

### 2.2 Clean Architecture (4계층)

```
+---------------------------------------------------------------------------+
|                         Presentation Layer                                  |
|  +---------------+  +---------------+  +------------------------------+    |
|  | REST API      |  | DTO           |  | Exception Handler            |    |
|  | Controllers   |  | Request/Resp  |  | (GlobalExceptionHandler)     |    |
|  +-------+-------+  +-------+-------+  +------------------------------+    |
|          |                  |                                               |
+----------+------------------+-----------------------------------------------+
|          v                  v         Application Layer                      |
|  +--------------------------------------------------------------------+    |
|  |  AmendmentClaimService (유스케이스 오케스트레이터)                       |
|  |  +- createRequest()      -> M1                                      |
|  |  +- executePreCheck()    -> M3                                      |
|  |  +- calculateCredits()   -> M4                                      |
|  |  +- optimizeCombination()-> M5                                      |
|  |  +- generateReport()     -> M6                                      |
|  |  +- runSimulation()      -> M7                                      |
|  +--------------------------------------------------------------------+    |
|                                                                             |
+-----------------------------------------------------------------------------+
|                         Domain Layer                                         |
|  +------------+  +----------------+  +-------------------------------+      |
|  | Entities   |  | Value Objects  |  | Domain Services               |      |
|  | +- Request |  | +- ReqId      |  | +- M3 PreCheckService         |      |
|  | +- Taxpayer|  | +- TaxType    |  | +- M4 CreditCalcService       |      |
|  | +- Credit  |  | +- MoneyAmount|  | |  +- 22개 서브모듈            |      |
|  | +- Combo   |  | +- TaxRate    |  | +- M5 OptimizationService     |      |
|  | +- Refund  |  | +- CapitalZone|  | +- MX01 CircularResolver      |      |
|  +------------+  +----------------+  | +- MX02 LawVersionMatcher     |      |
|  +----------------------------+      | +- MX03 ValidationEngine      |      |
|  | Repository Interfaces     |      +-------------------------------+      |
|  | +- RequestRepository      |                                              |
|  | +- CreditRepository       |                                              |
|  | +- ReferenceRepository    |                                              |
|  | +- AuditLogRepository     |                                              |
|  +----------------------------+                                              |
|                                                                             |
+-----------------------------------------------------------------------------+
|                     Infrastructure Layer                                     |
|  +--------------+  +--------------+  +----------------------------+         |
|  | JPA Entities |  | MyBatis      |  | Config                     |         |
|  | (83 tables)  |  | Mappers      |  | +- SecurityConfig          |         |
|  |              |  | (복잡쿼리용) |  | +- JpaConfig               |         |
|  +---------+----+  +------+-------+  | +- MyBatisConfig           |         |
|  | Repository   |  | JSON Parser  |  +----------------------------+         |
|  | Impl         |  | (RawData ->  |  | Common                     |         |
|  |              |  |  SV_* 파싱)  |  | +- TruncateUtil            |         |
|  +--------------+  +--------------+  | +- MoneyCalculator          |         |
|                                      | +- ExceptionClasses         |         |
|                                      +----------------------------+         |
+---------------------------------------------------------------------------+
        |                                        |
        v                                        v
  +----------+                           +--------------+
  |PostgreSQL|                           | RF_* 기준정보 |
  |   15     |                           | (26개 테이블)  |
  +----------+                           +--------------+
```

### 2.3 패키지 구조

```
src/main/java/com/taxservice/entec/
+-- presentation/                    # API 계층
|   +-- controller/
|   |   +-- RequestController.java       # API-01: 요청접수
|   |   +-- AnalyzeController.java       # API-02: 점검실행
|   |   +-- StatusController.java        # API-03: 상태조회
|   |   +-- ReportController.java        # API-04~06: 보고서
|   |   +-- ReferenceController.java     # API-07: 기준정보
|   +-- dto/
|   |   +-- request/                     # 요청 DTO
|   |   +-- response/                    # 응답 DTO
|   +-- advice/
|       +-- GlobalExceptionHandler.java  # 통합 오류 응답
|
+-- application/                     # 유스케이스 계층
|   +-- service/
|   |   +-- AmendmentClaimService.java   # 오케스트레이터
|   |   +-- RequestService.java          # 요청 접수
|   |   +-- AnalyzeService.java          # 분석 오케스트레이션
|   |   +-- ReportService.java           # 보고서
|   +-- port/
|   |   +-- in/                          # 인바운드 포트
|   |   +-- out/                         # 아웃바운드 포트
|   +-- mapper/
|       +-- DtoMapper.java               # DTO <-> Domain 매핑
|
+-- domain/                          # 도메인 계층
|   +-- model/
|   |   +-- ReqId.java                   # Value Object
|   |   +-- TaxType.java                 # Enum: CORP / INC
|   |   +-- CorpSize.java               # Enum: SMALL/MEDIUM/MID_LARGE/LARGE
|   |   +-- CapitalZone.java            # Enum: 4구분
|   |   +-- RequestStatus.java          # Enum: 요청 상태
|   |   +-- MoneyAmount.java            # 절사 정책 내장
|   |   +-- TaxRate.java                # 소수점 3자리 절사
|   |   +-- CreditItem.java             # 공제/감면 항목
|   |   +-- Combination.java            # 조합 도메인 모델
|   |   +-- RefundResult.java           # 환급 결과 모델
|   |   +-- PrepaidTaxDetail.java       # v2.0: 기납부세액 상세
|   +-- repository/                      # Repository 인터페이스
|   +-- service/
|       +-- module/                      # M 모듈 (공통)
|       |   +-- m1/                      # M1 입력 관리
|       |   +-- m2/                      # M2 기준정보
|       |   +-- m3/                      # M3 사전 점검
|       |   +-- m4/                      # M4 개별 공제/감면 (22개 서브모듈)
|       |   +-- m5/                      # M5 최적 조합
|       |   +-- m6/                      # M6 최종 산출
|       |   +-- mx/                      # MX 횡단 기능
|       +-- inc/                         # P 모듈 (개인 전용)
|       +-- common/
|           +-- TruncationPolicy.java
|           +-- TaxCalculationUtils.java
|
+-- infrastructure/                  # 인프라 계층
    +-- persistence/
    |   +-- jpa/entity/              # JPA Entity (83개)
    |   +-- jpa/repository/          # JpaRepository 구현
    |   +-- mybatis/mapper/          # MyBatis Mapper
    |   +-- mybatis/xml/             # MyBatis XML
    +-- config/
    +-- common/
        +-- constants/
        +-- exception/
        +-- util/
```

### 2.4 3계층 데이터 아키텍처

```
+------------------------------------------------------------+
|                  RI_S_REQUEST (요청 마스터)                   |
|             req_id = 신청인 + 날짜 기반 유니크 키             |
+----------+-------------------------------+-----------------+
           | FK: req_id                     | FK: req_id
  +--------v--------+            +----------v--------+
  |   입력자료 계층    |            |   출력결과 계층    |
  |                  |            |                   |
  | RI_S_RAW_DATA   |            |  CO_* 테이블      |
  | (원시 JSON)      |            |  (산출 결과)      |
  |     | 파싱       |            |     | 직렬화      |
  | SV_* 요약       |            | CO_S_REPORT_JSON  |
  | (계산용)         |            |  (전달용 JSON)    |
  +------------------+            +-------------------+
           |                               |
           +------> 점검/산출 엔진 <--------+
                    (M2~M5)
                    + RF_* 참조
```

### 2.5 TAX_TYPE 분기 매트릭스

| TAX_TYPE | 입력 (RI_) | 기준정보 (RF_) | 산출 (CO_) | 전용 모듈 |
|----------|-----------|--------------|-----------|---------|
| **CORP** | RI_C_* 17개 + RI_S_* 12개 | RF_C_* 7개 + RF_S_* 15개 | CO_S_* 8개 | M3-03(본점간주), M4-07/08/09/11/12/13/14/15/42, M5-03(과세표준기준) |
| **INC** | RI_I_* 8개 + RI_S_* 12개 | RF_I_* 4개 + RF_S_* 15개 | CO_S_* 8개 + CO_I_* 3개 | P3-01~04, P4-01~14, P5-01(산출세액기준) |

---

## 3. 모듈 상세 설계

### 3.1 모듈 계층 구조 (v2.0 확장)

```
M0 (분기) ---------------------------------------------------------------
  |
  +-- M1 (입력 관리) -- TX-1 -------------------------------------------
  |   +-- M1-01: req_id 발급 + RI_S_REQUEST 생성
  |   +-- M1-02: 원시 JSON 수신 + RI_S_RAW_DATA 보관
  |   +-- M1-03: 요약 테이블 생성 (SV_S_BASIC/EMPLOYEE/DEDUCTION/FINANCIAL)
  |
  +-- M3-PREP (데이터 준비) -- TX-2 시작 --------------------------------
  |   +-- PREP-1: SV_* 완전성 검증
  |   +-- PREP-2: 적용 법령 버전 결정
  |   +-- PREP-3: 기준정보 사전 조회 (세율/공제율/한도 스냅샷)
  |   +-- PREP-4: 기초 데이터 사전 산정 (규모/소재지/고용 잠정값)
  |   +-- PREP-5: 적용 가능 공제/감면 후보 조항 도출
  |   +-- PREP-6: SV_S_PREP_SUMMARY INSERT
  |
  +-- M3 (사전 점검) -- STEP 0 -----------------------------------------
  |   +-- M3-00: 경정청구 불가 사유 사전 차단 (Hard Fail)
  |   +-- M3-01: 경정청구 기한 점검 (5년/3개월)
  |   +-- M3-02: 중소기업 해당 여부 (매출/독립성/졸업유예)
  |   +-- M3-03: 소재지 구분 (본점간주 vs 실제소재지)
  |   +-- M3-04: 상시근로자 수 산정 (제외/청년등/월별평균)
  |   +-- M3-05: 업종 적격성 판정
  |   +-- M3-06: 결산확정 원칙 검증 (CORP)
  |   +-- P3-01: 소재지 사업장별 개별 판단 (INC)
  |   +-- P3-02: 성실신고확인대상 자동판정 (INC)
  |   +-- P3-03: 종합소득금액 산정 (INC)
  |   +-- P3-04: 소득공제/과세표준/산출세액 산정 (INC)
  |
  +-- M4 (개별 공제/감면) -- STEP 1~2 -- [22개 서브모듈] -----------------
  |   |
  |   +-- [공통 공제/감면 8개]
  |   |   +-- M4-01: 통합투자세액공제 (SS24)
  |   |   +-- M4-02: 통합고용세액공제 (SS29의8) + 경과규정(SS29의7) 비교
  |   |   +-- M4-03: 사회보험료세액공제 (SS30의4)
  |   |   +-- M4-04: 창업중소기업 세액감면 (SS6)
  |   |   +-- M4-05: 중소기업 특별세액감면 (SS7)
  |   |   +-- M4-06: 연구인력개발비 세액공제 (SS10)
  |   |   +-- M4-08: 이월결손금 재검토 (법인세법 SS13) [v2.0 신규 - C-05]
  |   |   +-- M4-10: 결손금 소급공제 (SS72/SS85의2)
  |   |
  |   +-- [법인 전용 6개]
  |   |   +-- M4-07: 외국납부세액공제 (법인 SS57)
  |   |   +-- M4-09: 수입배당금 익금불산입 (SS18의2)
  |   |   +-- M4-11: 세무조정 재검토 (접대비/가지급금/업무용승용차)
  |   |   +-- M4-12: 토지등 양도세 추가과세 (법인세법 SS55의2) [Phase 2]
  |   |   +-- M4-13: 기업구조조정 과세이연 (법인세법 SS44~47) [Phase 2]
  |   |   +-- M4-14: 연결납세 특화 점검 (법인세법 SS76의8) [Phase 2]
  |   |   +-- M4-15: 재해손실세액공제 (법인 SS58)
  |   |   +-- M4-42: 감가상각 시부인 이월추인
  |   |
  |   +-- [개인 전용 10개]
  |       +-- P4-01: 소득공제 누락/세율구간 최적화
  |       +-- P4-02: 감면소득 배분 계산 (다사업장)
  |       +-- P4-03: 공동사업자 소득 배분
  |       +-- P4-04: 성실사업자 의료비/교육비 (최저한세 비대상)
  |       +-- P4-05: 성실신고확인비용 (최저한세 대상)
  |       +-- P4-06: 기장세액공제 (중복배제 비대상)
  |       +-- P4-07: 외국납부세액공제 (소득세 SS57)
  |       +-- P4-08: 착한 임대인 세액공제 (SS96의3)
  |       +-- P4-09: 결손금 소급공제 (소득세 SS85의2) [v2.0 신규 - C-01]
  |       +-- P4-10: 노란우산공제 소득공제 (SS51의3) [v2.0 신규 - C-02]
  |       +-- P4-11: 연금저축세액공제 (SS59의3) [v2.0 신규 - C-02]
  |       +-- P4-12: 자녀세액공제 (SS59의2) [v2.0 신규 - C-02]
  |       +-- P4-13: 재해손실세액공제 (소득세 SS58) [Phase 2]
  |       +-- P4-14: 벤처투자조합 출자 소득공제 (SS16) [Phase 2]
  |
  +-- M5 (최적 조합) -- STEP 3 -----------------------------------------
  |   +-- M5-01: 상호배제 검증 (RF_S_MUTUAL_EXCLUSION) [v2.0: 10규칙]
  |   +-- M5-02: 최적 조합 탐색 (Branch & Bound / Greedy)
  |   +-- M5-03: 최저한세 적용 (CORP: 과세표준기준, INC: 산출세액기준)
  |   +-- M5-04: 농특세 적용 (항목별 비과세/과세 구분)
  |   +-- M5-05: 세액 적용순서 처리
  |   +-- P5-01: 개인 최저한세 산출 (INC, 35%/45%)
  |
  +-- M6 (최종 산출) -- STEP 4~5 ----------------------------------------
  |   +-- M6-01: 환급액 비교표 (기존 vs 경정후)
  |   +-- M6-02: 환급가산금 (본세/중간예납 기산일 분리)
  |   +-- M6-03: 지방소득세 경정청구 안내 [v2.0: 번호 정리]
  |   +-- M6-04: 보고서 3종 생성
  |   +-- M6-05: JSON 직렬화 (CO_S_REPORT_JSON, 8섹션) [v2.0: 7->8]
  |
  +-- M7 (시뮬레이션) -- Phase 2 ----------------------------------------
  |   +-- M7-01: 다수 과세연도 시뮬레이션
  |   +-- M7-03: 사후관리 리스크 평가
  |
  +-- MX (횡단 기능) ----------------------------------------------------
      +-- MX-01: 순환참조 해결 (과세표준<->최저한세, 최대 5회 수렴)
      +-- MX-02: 법령 버전 자동 매칭 [v2.0: 2018~2025 지원]
      +-- MX-03: 검증 엔진 (107개 규칙) [v2.0: 83+24]
```

### 3.2 M1 입력 관리 모듈

#### M1-01: 요청 접수 및 req_id 발급

| 항목 | 내용 |
|------|------|
| **입력** | applicant_id, taxpayer_id, tax_type, tax_year |
| **출력** | req_id, RI_S_REQUEST 1행 |
| **동시성** | SELECT ... FOR UPDATE + SERIALIZABLE 트랜잭션 |
| **재시도** | 최대 3회, 지수 백오프 (100ms x 2^n) |

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
PHASE 0: 재처리 판단 (version > 1 -> 기존 SV_* DELETE 후 재생성)
PHASE 1: SV_S_BASIC 생성 (세목/규모 기초 -> 최우선)
PHASE 2: SV_S_EMPLOYEE 생성 (PRIOR/CURRENT 2행, 제외대상 필터링)
PHASE 3: SV_S_DEDUCTION 생성 (N행, 카테고리별 파싱)
PHASE 4: SV_S_FINANCIAL 생성 (재무/세무 수치 집계)
PHASE 5: 완료 확인 및 상태 전이 (parsed)
```

### 3.3 M3 사전 점검 모듈

#### M3-00: 경정청구 불가 사유 사전 차단

| 검증 항목 | Hard/Soft | 조건 | 법령 |
|----------|:---------:|------|------|
| 기한 경과 | Hard | 법정신고기한 + 5년 초과 | 국기법 SS45의2 |
| 분식회계 | Hard | 사실과 다른 회계처리 해당 | 국기법 SS45의2(4) |
| 결산조정 항목 (CORP) | Hard | 감가상각비/퇴직급여/대손충당금 추가 계상 | 법인세법 SS40 |
| 추계신고 (INC) | Hard | 추계 -> 감면/공제 배제 | 조특법 SS128 |
| 직권경정 중 감면 제한 | Soft | 과소신고분 감면 제한 확인 | 조특법 SS128(3) |

#### M3-03: 소재지 구분 판단 (TAX_TYPE 분기)

| 적용 조항 | 판단 기준 | CORP | INC |
|----------|----------|------|-----|
| SS7 (중소특별) | 본점 간주 | 본점=수도권 -> 전사업장 수도권 | 사업장별 개별 판단 |
| SS24 (투자) | 실제 설치장소 | 자산 소재지 기준 | 자산 소재지 기준 |
| SS6 (창업) | 법인/사업장 소재지 | 법인 소재지 | 사업장 소재지 |
| SS29의8 (고용) | 합산 (소재지 무관) | 법인 전체 | 전사업장 합산 |

#### M3-04: 상시근로자 수 산정

```
제외 대상:
  [CORP] 임원, 최대주주 4촌이내 혈족/인척, 일용직, 주 15시간 미만 단시간
  [INC]  대표자 본인, 배우자, 직계존비속

연평균 산정:
  annual_avg = TRUNCATE(SUM(월별인원) / 해당월수, 2)  -- 소수점 3자리 절사

청년 판단 시점 연도별 분기:
  2024 이전: 과세연도 말일 기준 (만 15~34세 + 병역가산 최대 6년)
  2025 이후: 근로계약 체결일 기준 [v2.0: m-10 반영]
```

### 3.4 M4 개별 공제/감면 모듈 (22개 서브모듈)

#### CreditCalculator Strategy Pattern

```java
public interface CreditCalculator {
    String getProvision();              // 적용 조항 (SS24, SS10 등)
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
            .filter(p -> calculators.get(p).getSupportedTaxTypes()
                .contains(ctx.getTaxType()))
            .map(p -> calculators.get(p).calculate(ctx))
            .collect(toList());
    }
}
```

#### 서브모듈 상세표 (v2.0 완전판)

| 서브모듈 | 조항 | 세목 | Phase | 입력 | 핵심 로직 | 농특세 | 최저한세 | 이월 |
|---------|------|------|:-----:|------|----------|:------:|:-------:|:----:|
| M4-01 | SS24 | 공통 | 1 | SV_S_DEDUCTION(INVEST) | 기본(투자x공제율)+추가(당해-3년평균) | 과세20% | 대상 | O(10년) |
| M4-02 | SS29의8 | 공통 | 1 | SV_S_EMPLOYEE | 기본(단가x증가)+추가(경력단절/정규직/육아) | 과세20% | 대상 | O(10년) |
| M4-03 | SS30의4 | 공통 | 1 | SV_S_EMPLOYEE | 사업주부담분x공제율(청년100%/일반50%) | 과세20% | 대상 | X |
| M4-04 | SS6 | 공통 | 1 | SV_S_DEDUCTION(STARTUP) | 산출세액x(감면소득/총소득)x감면율 | 비과세 | 대상 | X |
| M4-05 | SS7 | 공통 | 1 | SV_S_DEDUCTION(SME_SPECIAL) | 업종/규모/소재지별 5~30%, 한도1억 | 비과세 | 대상 | X |
| M4-06 | SS10 | 공통 | 1 | SV_S_DEDUCTION(RD) | MAX(증가분,당기분), 3단계 최저한세 배제 | 비과세 | 조건부 | O(10년) |
| M4-07 | 법인SS57 | CORP | 2 | SV_S_FINANCIAL | 직접/간접/손금산입 3방식 비교 | N/A | 비대상 | X |
| **M4-08** | 법인SS13 | CORP | **1** | credit_carryforward | **이월결손금 재검토: 기간/한도 재산정, 연쇄영향** | N/A | N/A | N/A |
| M4-09 | SS18의2 | CORP | 2 | dividend_income | 지분율별 차등 익금불산입 | N/A | N/A | N/A |
| M4-10 | SS72/SS85의2 | 공통 | 2 | SV_S_FINANCIAL | 소급공제 환급세액 산출 + NPV 비교 | N/A | N/A | N/A |
| M4-11 | 법인세법 | CORP | 2 | tax_adjustment | 접대비/가지급금/업무용승용차 재검토 | N/A | N/A | N/A |
| M4-12 | SS55의2 | CORP | Phase2 | - | 토지등 양도세 추가과세 | - | - | - |
| M4-13 | SS44~47 | CORP | Phase2 | - | 기업구조조정 과세이연 | - | - | - |
| M4-14 | SS76의8 | CORP | Phase2 | - | 연결납세 특화 점검 | - | - | - |
| M4-15 | 법인SS58 | CORP | 2 | disaster_loss | 자산 20%이상 상실 시 세액공제 | N/A | 비대상 | X |
| M4-42 | 법인세법 | CORP | 2 | depreciation_adjust | 감가상각 시부인 이월추인 | N/A | N/A | N/A |
| P4-01 | 소득세법 | INC | 1 | inc_deduction | **소득공제 누락점검, 세율구간 최적화** | N/A | N/A | N/A |
| P4-02 | 시행령SS117 | INC | 1 | inc_business | 감면소득배분=산출세액x(감면소득/종합소득)x감면율 | - | - | - |
| P4-03 | SS43 | INC | 1 | inc_joint_biz | 소득x지분율, 상시근로자=전체기준 | - | - | - |
| P4-04 | SS122의3 | INC | 1 | inc_sincerity | 의료비15%+교육비15% | N/A | **비대상** | X |
| P4-05 | SS126의6 | INC | 1 | inc_sincerity | 60%, 120만원 한도 | N/A | **대상** | X |
| P4-06 | 소득세SS56의2 | INC | 1 | - | 20%, 100만원 한도, **조특법 중복배제 비대상** | N/A | 비대상 | X |
| P4-07 | 소득세SS57 | INC | 1 | inc_foreign_tax | 세액공제 vs 필요경비 비교 | N/A | 비대상 | X |
| P4-08 | SS96의3 | INC | 1 | inc_rental_reduction | 인하분x70% | 과세20% | 대상 | X |
| **P4-09** | SS85의2 | INC | **1** | SV_S_FINANCIAL | **소급공제: 직전연도 산출세액x(소급결손금/과세표준)** | N/A | N/A | N/A |
| **P4-10** | SS51의3 | INC | **1** | inc_deduction | **노란우산: 소득구간별 한도(200/300/500만원)** | N/A | N/A | N/A |
| **P4-11** | SS59의3 | INC | **1** | inc_deduction | **연금저축: 600만원 한도, 15%/12% 차등** | N/A | 비대상 | X |
| **P4-12** | SS59의2 | INC | **1** | inc_deduction | **자녀세액: 7세이상 15만원/인, 출산추가** | N/A | 비대상 | X |
| P4-13 | 소득세SS58 | INC | Phase2 | - | 재해손실세액공제 (개인) | N/A | 비대상 | X |
| P4-14 | SS16 | INC | Phase2 | - | 벤처투자조합 출자 소득공제 | N/A | N/A | N/A |

> **v2.0 신규 서브모듈**: M4-08(이월결손금 재검토), P4-09(개인 소급공제), P4-10(노란우산), P4-11(연금저축), P4-12(자녀세액), M4-12/13/14(Phase 2)

### 3.5 M4-08 이월결손금 재검토 모듈 [v2.0 신규 - C-05]

```
목적: 법인세법 제13조에 따른 이월결손금의 완전성 재검토

입력: SV_S_FINANCIAL, RI_S_LOSS_CARRYFORWARD (credit_carryforward 카테고리)
출력: CO_S_CREDIT_DETAIL (결손금 변동분)

핵심 로직:
  1. 사업연도별 이월결손금 잔액 확인 (FIFO 순서)
  2. 이월 기한 도과 여부 확인
     - 2020.1.1 이후 발생: 15년
     - 2020.1.1 이전 발생: 10년
  3. 공제 한도 재산정
     - 중소기업: 소득의 100%
     - 비중소기업: 소득의 80%
  4. 경정청구로 소득금액 변동 시 결손금 공제액 연쇄 재산정
     -> MX-01 순환참조 해결과 연동

연쇄 영향:
  결손금 변동 -> 과세표준 변동 -> 산출세액 변동 -> 최저한세 변동
  -> 공제한도 변동 -> 이월공제액 변동 -> 과세표준 재변동
```

### 3.6 P4-09 개인 결손금 소급공제 모듈 [v2.0 신규 - C-01]

```
목적: 소득세법 제85조의2에 따른 개인 결손금 소급공제

입력: SV_S_FINANCIAL(직전연도), SV_S_BASIC(중소기업 여부)
출력: CO_I_LOSS_CARRYBACK

핵심 공식 (F-INC-11):
  환급세액 = 직전연도 산출세액 x (소급결손금 / 직전연도 과세표준)

전제 조건:
  - 중소기업 해당 (소득세법 시행령 SS9-2)
  - 직전연도에 과세표준 및 납부세액 존재
  - 당해연도 결손 발생

NPV 비교 (M4-10 연동):
  소급공제 -> 즉시 현금 환급 (NPV 높음)
  이월공제 -> 향후 소득에서 차감 (10년, NPV 낮지만 총액 유리할 수 있음)
  -> 할인율 적용 비교 후 권고안 생성
```

### 3.7 M5 최적 조합 모듈

#### M5-01: 상호배제 규칙 (v2.0: 10규칙)

| No | 조항 A | 조항 B | 허용 | 조건 | v2.0 |
|----|--------|--------|:----:|------|:----:|
| 1 | SS6 (창업) | SS7 (중소특별) | X | 택1 | - |
| 2 | SS6 (창업) | SS29의8 (고용) | 조건부 | 2024 이전: 허용 / 2025 이후: 택1 | - |
| 3 | SS7 (중소특별) | SS29의8 (고용) | O | 병행 가능 | - |
| 4 | SS6 (창업) | SS24 (투자) | O | 병행 가능 | - |
| 5 | SS7 (중소특별) | SS24 (투자) | O | 병행 가능 | - |
| 6 | SS10 (R&D) | 모든 항목 | O | 병행 가능 (동일자산 SS24 제외) | - |
| **7** | **SS7 (중소특별)** | **SS30의4 (사보)** | **O** | **병행 가능 (제127조 제4항 괄호)** | **신규** |
| **8** | **SS6 (창업)** | **SS30의4 (사보)** | **X** | **중복 불가** | **신규** |
| **9** | **SS6 제7항(수입8천만이하)** | **SS29의8 (고용)** | **X** | **중복 불가 (제127조)** | **신규** |
| **10** | **SS29의8 (고용)** | **SS30의4 (사보)** | **X** | **중복 불가 (택1)** | **신규** |

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
  1. 적용순서 (법인SS59/소득세SS60): 감면 -> 이월불가공제 -> 이월가능공제
  2. 최저한세: 산출세액 - (과세표준 x 최저한세율) = 공제가능한도
  3. R&D 최저한세 배제: 국가전략(전액) > 신성장/중소(전액) > 일반/중소(50%)
  4. 최저한세 비대상 항목 별도 적용 [v2.0]:
     - P4-04 의료비/교육비(SS122의3): 최저한세 비대상
     - P4-06 기장세액공제(SS56의2): 조특법 외 -> 중복배제 비대상
     - P4-11 연금저축(SS59의3): 소득세법 공제
     - P4-12 자녀세액(SS59의2): 소득세법 공제
  5. 감면 초과분 = 소멸, 공제 초과분 = 이월(10년)
  6. 농특세: 항목별 비과세(SS6,SS7,SS10) / 과세 20%(SS24,SS29의8,SS30의4)
  7. 실질환급액 = 총공제감면 - 농특세
  8. 순환참조 해결 (MX-01): 과세표준<->최저한세 최대 5회, 1원 수렴
```

### 3.8 MX 횡단 기능

#### MX-01: 순환참조 해결

```
FOR i = 1 TO 5:
  과세표준(i) = 소득 - 이월결손금(i-1) - 공제액(i-1)
  최저한세(i) = 과세표준(i) x 최저한세율
  공제액(i) = MIN(산출세액 - 최저한세(i), 공제요청액)
  IF ABS(과세표준(i) - 과세표준(i-1)) <= 1원:
    수렴 완료 -> BREAK

수렴 실패 시: WARNING 로그 + 현재까지 최선 결과 반환
```

#### MX-02: 법령 버전 자동 매칭 [v2.0 확장]

```
경정청구 5년 범위 지원:
  2026년 기준 -> 2021~2025 과세연도 대상
  2025년 기준 -> 2020~2024 과세연도 대상

RF_S_TAX_RATE_BRACKET 테이블:
  법인세: 2018~2025 세율 보관 (경정청구 범위 확보)
  소득세: 2018~2025 세율 보관

v1.0에서 누락되었던 구세율 추가 [C-06, C-07]:
  법인세 ~2022: 10/20/22/25% 4단계
  소득세 ~2022: 1,200만 이하 6% 등 7~8단계
```

#### MX-03: 검증 엔진 (107개 규칙) [v2.0 확장]

| 유형 | 코드 | 건수 | 설명 | 실패 시 |
|------|-----|:----:|------|--------|
| V (Validation) | V01~V19 | 19 | 입력 형식/범위 검증 | ERROR |
| C (Calculation) | C01~C13 | 13 | 계산 결과 논리 검증 | ERROR |
| B (Business) | B01~B11 | 11 | 업무 규칙 준수 | ERROR |
| L (Legal) | L01~L09 | 9 | 법령 적합성 | ERROR |
| X (Cross) | X01~X21 | **21** | 교차 검증 (정합성) [v2.0: +8] | ERROR |
| W (Warning) | W01~W06 | 6 | 경고 (Hard Fail 아님) | WARNING |
| VP (INC Validation) | VP01~VP06 | **6** | 개인 입력 검증 [v2.0: +3] | ERROR |
| CP (INC Calculation) | CP01~CP05 | **5** | 개인 계산 검증 [v2.0: +2] | ERROR |
| BP (INC Business) | BP01~BP05 | **5** | 개인 업무 규칙 [v2.0: +2] | ERROR |
| LP (INC Legal) | LP01~LP04 | **4** | 개인 법령 적합 [v2.0: +2] | ERROR |
| WP (INC Warning) | WP01~WP02 | **2** | 개인 경고 [v2.0: +1] | WARNING |
| **XP** (INC Cross) | **XP01~XP05** | **5** | **개인 교차검증 [v2.0 신규]** | ERROR |

**v2.0 신규 검증 규칙 상세:**

```
LP03: 개인 결손금 소급공제 중소기업 한정 확인
LP04: 2025년 이후 SS6+SS29의8 중복 Hard Fail [C-04]

VP04: 성실신고확인대상 수입금액 기준 교차검증
VP05: 다사업장 소득 합계 일치 검증
VP06: 성실사업자 자격 교차검증

CP04: 환급액 <= 실제납부세액 상한 검증
CP05: 소득공제 후 과세표준 세율구간 변동 확인

BP04: 노란우산공제 한도(소득구간별 200/300/500만원) 검증
BP05: 간편장부 복식기장 시 기장세액공제 적용 가능 확인

WP02: 분리과세 소득 제외 확인 (부동산임대 등)

XP01: 종합소득금액 분모에서 분리과세 소득 제외 확인
XP02: 다사업장 감면소득 합계 <= 종합소득금액 확인
XP03: 공동사업자 지분율 배분 후 소득합계 일치 확인
XP04: 개인 결손금 소급공제 한도 <= 직전연도 납부세액 확인
XP05: 소득공제 합계 <= 종합소득금액 확인

X14: JSON 보고서 section_a 스키마 검증
X15: JSON 보고서 section_b 스키마 검증
X16: JSON 보고서 section_c 스키마 검증
X17: JSON 보고서 section_d 스키마 검증
X18: JSON 보고서 환급액 합산 일치 검증
X19: JSON 보고서 농특세 합산 일치 검증
X20: JSON 보고서 이월분석 정합 검증
X21: JSON 보고서 기납부세액 4분류 합산 검증
```

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
| API-05 | GET | `/api/v1/requests/{req_id}/report/sections/{A-H}` | 섹션별 JSON | 읽기 |
| API-06 | GET | `/api/v1/requests/{req_id}/raw-data` | 원시 입력 JSON 조회 | 읽기 |
| API-07 | GET | `/api/v1/reference/exclusion-matrix` | 상호배제 기준정보 | 읽기 |
| API-08 | GET | `/api/v1/requests/{req_id}/summary` | 경량 요약 조회 | 읽기 |
| API-09 | GET | `/api/v1/requests/{req_id}/credits` | 개별 공제/감면 조회 | 읽기 |
| API-10 | GET | `/api/v1/requests/{req_id}/combinations` | 조합 비교 조회 | 읽기 |
| API-11 | GET | `/api/v1/requests/{req_id}/audit-log` | 감사 로그 조회 | 읽기 |

### 4.3 API-01 요청/응답 스키마

**요청:**
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

**성공 응답 (201):**
```json
{
  "req_id": "C-1234567890-20260216-001",
  "status": "received",
  "datasets_received": 3,
  "created_at": "2026-02-16T14:00:00+09:00"
}
```

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
(1) API-01 요청접수 === [TX-1: 요청 트랜잭션] ===
   +-> M1-01: req_id 발급 -> RI_S_REQUEST
   +-> M1-02: 원시 JSON 보관 -> RI_S_RAW_DATA
   +-> M1-03: 요약 생성 -> SV_S_BASIC / EMPLOYEE / DEDUCTION / FINANCIAL
   +-> COMMIT TX-1

(2) API-02 점검실행 === [TX-2: 산출 트랜잭션] ===
   +-> PREP: 데이터 준비 -> SV_S_PREP_SUMMARY
   +-> STEP 0: 사전 점검 -> SV_S_ELIGIBILITY, SV_S_INSPECTION_LOG
   +-> STEP 1~2: 개별 산출 -> CO_S_CREDIT_DETAIL (22개 서브모듈)
   +-> STEP 3: 최적 조합 -> CO_S_COMBINATION, CO_S_EXCLUSION_VERIFY
   +-> STEP 4: 환급액 산출 -> CO_S_REFUND, CO_S_RISK
   +-> STEP 5: JSON 직렬화 -> CO_S_REPORT_JSON (8섹션)
   +-> COMMIT TX-2

(3) API-04 결과조회 === [읽기 전용] ===
   +-> CO_S_REPORT_JSON -> JSON 응답 반환
```

### 5.2 상태 전이 (request_status)

```
received -> parsing -> parsed -> preparing -> checking -> calculating
  -> optimizing -> reporting -> completed
                                    |
        Any state --(TX ROLLBACK)--> error
```

### 5.3 모듈별 입출력 매핑

| 모듈 | 입력 테이블 | 출력 테이블 | 참조 테이블 |
|------|-----------|-----------|-----------|
| M1-03 | RI_S_RAW_DATA | SV_S_BASIC, SV_S_EMPLOYEE, SV_S_DEDUCTION, SV_S_FINANCIAL | - |
| M3-PREP | SV_* 4개 | SV_S_PREP_SUMMARY | RF_S_LAW_VERSION, RF_S_TAX_RATE_BRACKET |
| M3-00~06 | SV_S_BASIC, SV_S_PREP | SV_S_ELIGIBILITY, SV_S_VALIDATION_LOG | RF_S_CAPITAL_ZONE, RF_S_INDUSTRY_ELIGIBILITY |
| M4-01~22 | SV_S_DEDUCTION, SV_S_EMPLOYEE | CO_S_CREDIT_DETAIL | RF_C_INVESTMENT_CREDIT_RATE 등 |
| M5-01~05 | CO_S_CREDIT_DETAIL | CO_S_COMBINATION, CO_S_EXCLUSION_VERIFY | RF_S_MUTUAL_EXCLUSION, RF_S_NONGTEUKSE |
| M6-01~05 | CO_S_COMBINATION | CO_S_REFUND, CO_S_RISK, CO_S_REPORT_JSON | RF_S_REFUND_INTEREST_RATE |

---

## 6. 계산 엔진 설계

### 6.1 법인세 계산 공식 (F01~F41)

#### 과세표준 및 산출세액

| 공식ID | 공식명 | 산출 공식 | 절사 | 근거법령 |
|--------|--------|----------|------|---------|
| F01 | 과세표준 | 각사업연도소득 - 이월결손금 - 비과세소득 | 10원절사 | 법인세법 SS13 |
| F02 | 이월결손금 공제한도 | MIN(잔액, 소득 x 한도율) [중소 100%, 기타 80%] | - | 법인세법 SS13 |
| F03 | 산출세액 | 과세표준 x 초과누진세율 | 10원절사 | 법인세법 SS55 |
| F04 | 최저한세 | 과세표준 x 최저한세율 [중소 7%, 중견 10~17%] | 10원절사 | 조특법 SS132 |

#### 법인세율 (과세연도별 -- v2.0 확장)

**2023~2025년:**

| 과세표준 구간 | 세율 | 누진공제 |
|-------------|------|---------|
| 2억 이하 | 9% | 0 |
| 2억~200억 | 19% | 2,000만 |
| 200억~3,000억 | 21% | 42,000만 |
| 3,000억 초과 | 24% | 942,000만 |

**~2022년 (v2.0 추가 - C-06):**

| 과세표준 구간 | 세율 | 누진공제 |
|-------------|------|---------|
| 2억 이하 | 10% | 0 |
| 2억~200억 | 20% | 2,000만 |
| 200억~3,000억 | 22% | 42,000만 |
| 3,000억 초과 | 25% | 942,000만 |

#### 공제/감면 세액

| 공식ID | 공식명 | 산출 공식 | 근거 |
|--------|--------|----------|------|
| F10 | 통합고용 기본 | (청년등증가 x 단가) + (일반증가 x 단가) | SS29의8 |
| F12 | 통합투자 기본 | 투자금액 x 기본공제율 | SS24 |
| F13 | 통합투자 추가 | MAX(0, 당해-직전3년평균) x 추가율 | SS24 |
| F14 | 창업감면 | 산출세액 x (감면소득/총소득) x 감면율 | SS6 |
| F15 | 중소특별감면 | 산출세액 x (감면소득/총소득) x 감면율 | SS7 |
| **F16** | **중소특별 감면한도** | **MIN(감면액, MAX(0, 1억-감소인원x500만))** | **SS7 [v2.0: M-05]** |
| F17 | R&D 증가분 | (해당R&D비 - 직전R&D비) x 공제율 | SS10 |
| F18 | R&D 당기분 | 해당R&D비 x 공제율 | SS10 |
| **F19** | **사회보험료 공제** | **SUM(사업주부담분 x 공제율, 근로자별)** | **SS30의4 [v2.0: M-06]** |

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
| F40 | 농특세 | SS6(창업), SS7(중소특별), SS10(R&D) | SS24(투자), SS29의8(고용), SS30의4(사보) |
| F41 | 농특세 변동 | 경정후 - 기존 | 조합 변경 시 연쇄 재계산 |

### 6.2 종합소득세 계산 공식 (F-INC-01~11) [v2.0 확장]

| 공식ID | 공식명 | 산출 공식 | 근거 |
|--------|--------|----------|------|
| F-INC-01 | 종합소득금액 | 사업소득 + 근로소득 + 금융소득(2천만초과시) + 연금 + 기타 | 소득세법 SS14 |
| F-INC-02 | 과세표준 | 종합소득금액 - 소득공제합계 | 소득세법 SS14 |
| F-INC-03 | 산출세액 | 과세표준 x 초과누진세율 (6~45% 8단계) | 소득세법 SS55 |
| F-INC-04 | 감면소득배분 | 산출세액 x (감면대상소득 / 종합소득금액) x 감면율 | 시행령 SS117 |
| F-INC-05 | 공동사업자 배분 | 전체소득 x 손익분배비율 | 소득세법 SS43 |
| F-INC-06 | 개인 최저한세 | 3천만이하: 산출세액 x 35% / 초과: 45% | 조특법 SS132(2) |
| **F-INC-07** | **공제가능한도** | **산출세액 - 최저한세액** | **[v2.0 신규: C-03]** |
| **F-INC-08** | **개인 환급액** | **실제납부세액 - 경정후 결정세액** | **[v2.0 신규: C-03]** |
| **F-INC-09** | **개인 환급가산금** | **환급액 x 이율 x 일수/365 (기산일: 5.31 또는 6.30)** | **[v2.0 신규: C-03]** |
| **F-INC-10** | **개인 총수령예상액** | **F-INC-08 + F-INC-09 + 지방소득세환급** | **[v2.0 신규: C-03]** |
| **F-INC-11** | **소급공제 환급세액** | **직전연도 산출세액 x (소급결손금/직전연도 과세표준)** | **[v2.0 신규: C-03]** |

#### 종합소득세율 (과세연도별 -- v2.0 확장)

**2023년 이후 (8단계):**

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

**2021~2022년 (8단계, v2.0 추가 - C-07):**

| 과세표준 구간 | 세율 | 누진공제 |
|-------------|------|---------|
| 1,200만 이하 | 6% | 0 |
| 1,200만~4,600만 | 15% | 108만 |
| 4,600만~8,800만 | 24% | 522만 |
| 8,800만~1.5억 | 35% | 1,490만 |
| 1.5억~3억 | 38% | 1,940만 |
| 3억~5억 | 40% | 2,540만 |
| 5억~10억 | 42% | 3,540만 |
| 10억 초과 | 45% | 6,540만 |

**~2020년 (7단계, v2.0 추가 - C-07):**

| 과세표준 구간 | 세율 | 누진공제 |
|-------------|------|---------|
| 1,200만 이하 | 6% | 0 |
| 1,200만~4,600만 | 15% | 108만 |
| 4,600만~8,800만 | 24% | 522만 |
| 8,800만~1.5억 | 35% | 1,490만 |
| 1.5억~3억 | 38% | 1,940만 |
| 3억~5억 | 40% | 2,540만 |
| 5억 초과 | 42% | 3,540만 |

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
        return (amount / 10) * 10;
    }
    public static double truncateRate(double rate) {
        return Math.floor(rate * 1000) / 1000;
    }
    public static long truncateInterest(long interest) {
        return interest;  // 1원 미만 절사 (long이므로 자동)
    }
    public static double truncateEmployee(double count) {
        return Math.floor(count * 100) / 100;
    }
}
```

---

## 7. 검증 엔진 설계

### 7.1 검증 규칙 분류 (107개)

> v1.0(83개) -> v2.0(107개): 24개 규칙 추가 (C-04, C-08, M-08, M-09 대응)

#### 공통 검증 (83개)

**V (Validation) - 19개**: 입력 데이터 형식/범위

| 코드 | 설명 | 실패 시 |
|------|------|--------|
| V01 | 경정청구 기한 검증 (5년 이내) | Hard Fail |
| V02 | req_id 형식 검증 | Hard Fail |
| V03 | tax_type 유효값 (CORP/INC) | Hard Fail |
| V04 | 필수 필드 NULL 검증 | Hard Fail |

**C (Calculation) - 13개**: 계산 결과 논리 검증

| 코드 | 설명 |
|------|------|
| C01 | 결산조정 항목 차단 (CORP) |
| C02 | 산출세액 >= 0 확인 |
| C03 | 공제액 <= 산출세액 확인 |

**B (Business) - 11개**: 업무 규칙

| 코드 | 설명 |
|------|------|
| B01 | 상호배제 규칙 (2025~ SS6+SS29의8 중복배제) |
| B02 | 최저한세 한도 초과 -> 이월공제 적용 |

**L (Legal) - 9개**: 법령 적합성

| 코드 | 설명 |
|------|------|
| L01 | 최저한세 한도 검증 |
| L02 | 적용순서 준수 (법인SS59/소득세SS60) |

**X (Cross) - 21개**: 교차 검증 [v2.0: 13->21개]

| 코드 | 설명 | v2.0 |
|------|------|:----:|
| X01 | 종합소득금액 분모 검증 (금융소득 2천만 초과 시) | - |
| X02 | 공동사업자 지분율 합계 = 100% | - |
| X14~X21 | JSON 보고서 스키마/합산 교차검증 (8건) | **신규** |

**W (Warning) - 6개**: 경고

| 코드 | 설명 |
|------|------|
| W01 | 순환참조 미수렴 (5회 초과) |
| W02 | 재창업 리스크 (동일업종 5년 이내) |

#### 개인 전용 검증 (24개) [v2.0: 12->24개]

| 코드 | 유형 | 설명 | v2.0 |
|------|------|------|:----:|
| VP01 | Validation | TAX_TYPE='INC' 시 RI_I_BASIC 필수 | - |
| VP02 | Validation | 다사업장 시 RI_I_BUSINESS 최소 1건 | - |
| VP03 | Validation | 공동사업자 손익분배비율 합계 = 100% | - |
| VP04 | Validation | 성실신고확인대상 수입금액 기준 교차검증 | **신규** |
| VP05 | Validation | 다사업장 소득 합계 일치 검증 | **신규** |
| VP06 | Validation | 성실사업자 자격 교차검증 | **신규** |
| CP01 | Calculation | 감면세액 <= 산출세액 x 배분비율 x 감면율 | - |
| CP02 | Calculation | 소득공제 후 과세표준 >= 0 | - |
| CP03 | Calculation | 개인 최저한세 산출 검증 | - |
| CP04 | Calculation | 환급액 <= 실제납부세액 상한 검증 | **신규** |
| CP05 | Calculation | 소득공제 후 세율구간 변동 확인 | **신규** |
| BP01 | Business | 기장세액공제 + 조특법 감면 중복 허용 확인 | - |
| BP02 | Business | 성실신고확인비용 = 최저한세 대상 | - |
| BP03 | Business | 성실사업자 의료비/교육비 = 최저한세 비대상 | - |
| BP04 | Business | 노란우산공제 한도 검증 | **신규** |
| BP05 | Business | 간편장부 기장세액공제 가능 확인 | **신규** |
| LP01 | Legal | 개인 적용순서 = 소득세법 SS60 | - |
| LP02 | Legal | 개인 경정청구 기산일 = 5.31 (성실: 6.30) | - |
| LP03 | Legal | 결손금 소급공제 중소기업 한정 | **신규** |
| LP04 | Legal | 2025년 이후 SS6+SS29의8 Hard Fail | **신규** |
| WP01 | Warning | 분리과세 소득 제외 확인 | - |
| WP02 | Warning | 건강보험료 가입유형별 처리 확인 | **신규** |
| XP01~05 | Cross | 개인 전용 교차검증 5건 | **신규** |

### 7.2 검증 엔진 아키텍처

```java
public interface ValidationRule {
    String getCode();                    // V01, C01 등
    String getType();                    // V/C/B/L/X/W
    Set<TaxType> getApplicableTaxTypes();
    ValidationResult validate(ValidationContext ctx);
}

@Component
public class ValidationEngine {
    private final List<ValidationRule> rules;  // 107개 규칙

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

### 8.1 JSON 보고서 8섹션 구조 [v2.0: 7->8섹션]

| 섹션 | 컬럼 | 소스 테이블 | 설명 |
|------|------|-----------|------|
| A | section_a_json | SV_S_ELIGIBILITY + SV_S_BASIC | 해당가능성 진단 |
| B | section_b_json | CO_S_CREDIT_DETAIL | 후보 항목 리스트 |
| C | section_c_json | CO_S_COMBINATION + CO_S_EXCLUSION_VERIFY | 최적조합/배제 |
| D | section_d_json | CO_S_REFUND | 환급/추징 금액 + 기납부세액 상세 |
| E | section_e_json | CO_S_RISK + CO_S_ADDITIONAL_CHECK | 사후관리/증빙목록 |
| F | section_f_json | co_s_carryforward_analysis | 이월분석/결손금소급공제NPV |
| G | section_g_meta | report_meta + taxpayer_info | 제출 데이터/메타 |
| **H** | **section_h_json** | **SV_S_INSPECTION_LOG + SV_S_VALIDATION_LOG** | **산출진행로그/검증결과 [v2.0 신규]** |

### 8.2 보고서 3종

| 보고서 유형 | 대상 | 포함 내용 |
|------------|------|----------|
| 경영진용 | 경영진 | 환급예상액, 최적조합, 핵심 변경, 리스크 요약 |
| 세무대리인용 | 세무담당자 | 전체 산출 과정, 계산식, 근거법령, 조합별 비교 |
| 국세청용 | 과세관청 | 경정청구서 양식, 첨부서류 목록, 근거서류 |

### 8.3 보고서 필수 포함 섹션 [v2.0 확장]

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
11. **세무조정 변동 사항 (법인 전용) [v2.0 신규: m-02]**
12. **환급가산금 익금불산입 안내 (법인세법 SS18 제4호) [v2.0 신규: m-03]**
13. **소득공제 세율구간 변동 분석 (개인 전용) [v2.0 신규]**

---

## 9. 트랜잭션 및 동시성 설계

### 9.1 트랜잭션 경계

| TX ID | 범위 | 격리수준 | 타임아웃 | 성공 시 | 실패 시 |
|-------|------|---------|---------|--------|--------|
| TX-1 | M1-01 ~ M1-03 | READ COMMITTED | 60초 | COMMIT -> parsed | ROLLBACK -> error |
| TX-2 | M3-PREP ~ M6-05 | READ COMMITTED | 300초 | COMMIT -> completed | ROLLBACK -> error |

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
| INSERT ONLY | RI_S_RAW_DATA DB 트리거로 UPDATE/DELETE 차단 |
| 감사 추적 | AL_S_CALCULATION 전 계산 이력 APPEND ONLY |

---

## 11. 구현 순서 및 의존성

### 11.1 Phase 1 (MVP - 7주) [v2.0: 6주->7주]

#### Week 1: 환경 설정 & 데이터 모델링

- [ ] Spring Boot 3.x 프로젝트 초기화 (Java 17)
- [ ] PostgreSQL 15 설치 및 DDL 생성 (83개 테이블)
- [ ] Domain 계층: Value Objects (ReqId, MoneyAmount, TaxRate, TaxType)
- [ ] Infrastructure 계층: JPA Entity 매핑 (핵심 10개 테이블)
- [ ] RF_* 기준정보 시드 데이터 적재 (2018~2025 세율 포함)
- [ ] req_id 생성 로직 + 동시성 테스트

#### Week 2: 입력 관리 (M1) + API-01

- [ ] POST /api/v1/requests 구현
- [ ] M1-01: req_id 발급 (SELECT FOR UPDATE)
- [ ] M1-02: RI_S_RAW_DATA INSERT ONLY 제약 (트리거)
- [ ] M1-03: SV_* 파싱 로직 (5 PHASE)
- [ ] 법인/개인 스키마 분기 테스트

#### Week 3: 사전 점검 (M3) + API-02

- [ ] M3-PREP: SV_S_PREP_SUMMARY 생성
- [ ] M3-00: Hard Fail 필터 (결산조정/기한경과/추계)
- [ ] M3-01~M3-06: 기한/중소/소재지/상시근로자/업종/결산확정
- [ ] P3-01~P3-04: 개인 전용 (소재지/성실신고/종합소득/과세표준)
- [ ] MX-03 검증 엔진 (핵심 60개 규칙 우선)
- [ ] MX-02 법령 버전 매칭 (2018~2025 세율 지원)

#### Week 4: 개별 공제/감면 - 공통 (M4)

- [ ] CreditCalculator 인터페이스 + CreditCalculatorRegistry
- [ ] M4-04: 창업감면 (SS6) -- 상세 산출공식 포함
- [ ] M4-05: 중소기업감면 (SS7) -- F16 감면한도 포함
- [ ] M4-02: 통합고용 (SS29의8) -- 상세 산출공식 포함
- [ ] M4-01: 통합투자 (SS24)
- [ ] M4-06: R&D (SS10) -- 증가분/당기분/비율분 3방식
- [ ] M4-03: 사회보험료 (SS30의4) -- F19 공식 포함
- [ ] M4-08: 이월결손금 재검토 (법인SS13) [v2.0 신규]

#### Week 5: 개별 공제/감면 - 세목별 확장 (M4)

- [ ] 개인 전용: P4-01 소득공제(IncomeDeductionService), P4-10 노란우산
- [ ] 개인 전용: P4-11 연금저축, P4-12 자녀세액
- [ ] 개인 전용: P4-04 의료비교육비(최저한세 비대상), P4-05 성실확인비용
- [ ] 개인 전용: P4-06 기장세액, P4-02 감면소득배분, P4-03 공동사업자
- [ ] 개인 전용: P4-07 외국납부세액, P4-08 착한임대인
- [ ] 개인 전용: P4-09 결손금 소급공제 [v2.0 신규]
- [ ] 법인 전용: M4-11 세무조정재검토(업무용승용차 포함)

#### Week 6: 최적 조합 (M5) + 기납부세액

- [ ] M5-01: 상호배제 검증 (10규칙 전체)
- [ ] M5-02: B&B / Greedy 조합 탐색
- [ ] M5-03: 최저한세 (CORP: 과세표준기준, INC: 산출세액기준)
- [ ] M5-03: 최저한세 비대상 항목 별도 적용 (P4-04, P4-06 등)
- [ ] M5-04: 농특세 (항목별 비과세/과세)
- [ ] M5-05: 적용순서 (감면->이월불가->이월가능)
- [ ] MX-01: 순환참조 해결 (최대 5회)
- [ ] 기납부세액 상세(PrepaidTaxDetail): 중간예납/원천징수/확정/분납
- [ ] M4-10: 결손금 소급공제 NPV 비교 (LossCarrybackService)

#### Week 7: 최종 산출 & 통합 테스트

- [ ] M6-01: 환급액 비교표
- [ ] M6-02: 환급가산금 (본세/중간예납 분리)
- [ ] M6-03: 지방소득세 경정청구 안내
- [ ] M6-04: 보고서 3종 생성
- [ ] M6-05: JSON 직렬화 (8섹션 A~H)
- [ ] 환급가산금 익금불산입 안내 (m-03 대응)
- [ ] 통합 테스트 (법인 10건, 개인 10건, 공동사업자 3건)
- [ ] 검증 규칙 107개 전수 테스트

### 11.2 Phase 2 (고도화 - 4주)

- M4-07: 외국납부세액 (직접/간접/손금산입 비교)
- M4-09: 수입배당금 익금불산입 (2023 개정법)
- M4-12: 토지등 양도세 추가과세
- M4-13: 기업구조조정 과세이연
- M4-14: 연결납세 특화 점검
- M4-15: 재해손실세액공제
- P4-13: 개인 재해손실세액공제
- P4-14: 벤처투자조합 출자 소득공제
- P4-15: 전자신고세액공제 (SS104의8)
- M4-42: 감가상각 시부인 이월추인
- M7-01: 다수 과세연도 시뮬레이션
- M7-03: 사후관리 리스크 평가
- Excel/PDF 파서

### 11.3 모듈 의존성 그래프

```
M1 (입력) -> M3-PREP (준비) -> M3 (점검) -> M4 (개별산출) -> M5 (최적조합) -> M6 (최종산출)
     |                            |              |                |              |
     +-- RF_* (기준정보) ----------+------+-------+---------+-----+---------+----+
                                  |             |
                              MX-01 (순환참조)
                              MX-02 (법령매칭)
                              MX-03 (검증엔진)
```

---

## 12. 테스트 계획

### 12.1 테스트 범위

| Type | Target | Tool | 건수 |
|------|--------|------|:----:|
| **Unit Test** | 56개 계산 공식 (F01~F56 + F-INC-01~11) | JUnit 5 + AssertJ | 67+ |
| **Unit Test** | 22개 서브모듈 개별 | JUnit 5 | 22x5 |
| **Unit Test** | MoneyAmount/TaxRate 절사 | JUnit 5 | 20+ |
| **Integration Test** | 107개 검증 규칙 | JUnit 5 + TestContainers | 107 |
| **Integration Test** | API 엔드포인트 | MockMvc | 20+ |
| **Integration Test** | 순환참조 수렴 | JUnit 5 | 10+ |
| **E2E Test** | 법인 전체 플로우 | RestAssured | 10 |
| **E2E Test** | 개인 전체 플로우 | RestAssured | 10 |
| **E2E Test** | 공동사업자 플로우 | RestAssured | 3 |

### 12.2 핵심 테스트 케이스

**계산 정확성:**
- [ ] F01~F04: 법인세 과세표준/산출세액/최저한세
- [ ] F-INC-01~F-INC-06: 개인 종합소득/과세표준/산출세액/감면배분/공동사업자/최저한세
- [ ] F-INC-07~F-INC-11: 개인 공제한도/환급액/가산금/총수령/소급공제 [v2.0]
- [ ] F14: 창업감면 = 산출세액 x (감면소득/종합소득) x 감면율
- [ ] F10: 통합고용 = (기본공제 + 청년추가) x TRUNCATE(증가인원, 0.00)
- [ ] F40: 농특세 비과세(SS6,SS7,SS10) / 과세20%(SS24,SS29의8,SS30의4)

**절사 정책:**
- [ ] 금액 10원 미만 절사: 1,234,567 -> 1,234,560
- [ ] 비율 소수점 3자리 절사: 0.62578 -> 0.625
- [ ] 반올림 시도 시 컴파일 오류 확인 (ROUND 메서드 미제공)

**상호배제 (10규칙):**
- [ ] SS6 + SS7: 항상 배제
- [ ] SS6 + SS29의8: 2025년부터 배제
- [ ] SS29의8 + SS30의4: 항상 배제
- [ ] SS7 + SS30의4: 병행 가능 [v2.0]
- [ ] SS6 + SS30의4: 중복 불가 [v2.0]

**순환참조:**
- [ ] 3회 이내 수렴 케이스
- [ ] 5회 도달 수렴 케이스
- [ ] 5회 미수렴 시 에러 처리

**세율 정확성 (v2.0):**
- [ ] 법인세 2023~2025: 9/19/21/24%
- [ ] 법인세 ~2022: 10/20/22/25%
- [ ] 소득세 2023~: 8단계 6~45%
- [ ] 소득세 2021~2022: 8단계 (1,200만 구간)
- [ ] 소득세 ~2020: 7단계

**개인 전용 (v2.0):**
- [ ] P4-01: 소득공제 반영 전/후 세율구간 변동 (8,800만원 경계)
- [ ] P4-04: 성실사업자 의료비/교육비 = 최저한세 비대상
- [ ] P4-05: 성실신고확인비용 = 최저한세 대상 (P4-04와 구분!)
- [ ] P4-06: 기장세액공제 = 조특법 중복배제 비대상
- [ ] P4-09: 개인 결손금 소급공제 = F-INC-11 공식
- [ ] P4-10: 노란우산공제 소득구간별 한도 (500/300/200만)
- [ ] P4-11: 연금저축 공제율 소득구간별 차등 (15%/12%)
- [ ] P4-12: 자녀세액공제 다자녀 추가 계산
- [ ] P4-03: 공동사업자 상시근로자 = 전체 기준 (공제세액만 지분배분)

---

## 13. 부록: 핵심 의사코드

### 13.1 최적 조합 탐색 (M5-02)

```
FUNCTION find_optimal_combination(individual_credits):

  // 1. 상호배제 규칙 로드 (v2.0: 10규칙)
  exclusion_rules = RF_S_MUTUAL_EXCLUSION.load()
  IF target_tax_year >= 2025:
    exclusion_rules.deny('SS6', 'SS29의8')

  // 2. 알고리즘 선택
  IF individual_credits.length <= 15:
    candidates = Branch_and_Bound(individual_credits, exclusion_rules)
  ELSE:
    candidates = Greedy_with_partial_BnB(individual_credits, exclusion_rules)

  // 3. 각 조합 평가
  FOR EACH combo IN candidates:
    // 3-1. 적용순서 (법인SS59 / 소득세SS60)
    step1 = apply_exemptions(combo.exemptions)      // 감면
    step2 = apply_non_carryover(combo.credits_nc)    // 이월불가공제
    step3 = apply_carryover(combo.credits_cf)        // 이월가능공제

    // 3-2. 최저한세
    min_tax = computed_tax * min_tax_rate(corp_size)
    max_deduction = computed_tax - min_tax

    // 3-3. 최저한세 비대상 항목 별도 적용 [v2.0]
    non_min_tax_items = filter(combo, item.minTaxSubject == FALSE)
    min_tax_items = filter(combo, item.minTaxSubject == TRUE)
    // min_tax_items에 max_deduction 적용 후, non_min_tax_items 별도 가산

    // 3-4. R&D 최저한세 배제 3단계
    rd_exempt = 국가전략(전액) + 신성장중소(전액) + 일반중소(50%)

    // 3-5. 감면 초과 = 소멸, 공제 초과 = 이월(10년)
    // 3-6. 농특세 (항목별 차등)
    combo.nongteuk = calc_nongteuk_by_item(combo)

    // 3-7. 실질 환급액
    combo.net_refund = combo.total_applied - combo.nongteuk

  // 4. 순환참조 해결 (MX-01)
  FOR EACH combo IN top_candidates:
    combo = resolve_circular_reference(combo, max_iterations=5)

  RETURN candidates.sort_by(net_refund DESC)
```

### 13.2 환급가산금 산출 (M6-02)

```
FUNCTION calc_refund_interest(refund_amount, payment_date):

  // 본세 환급가산금
  main_start = payment_date + 1일
  main_end = refund_decision_date
  main_rate = RF_S_REFUND_INTEREST_RATE.get(main_start, main_end)
  main_interest = TRUNCATE(refund_amount * main_rate * days / 365, 0)

  // 중간예납 환급가산금 (별도 기산)
  FOR EACH interim IN RI_C_INTERIM_TAX:
    int_start = interim.payment_date + 1일
    int_rate = RF_S_REFUND_INTEREST_RATE.get(int_start, decision_date)
    interim_interest += TRUNCATE(interim.overpaid * int_rate * days / 365, 0)

  RETURN { main_interest, interim_interest, total: main + interim }

  // v2.0 추가: 환급가산금 익금불산입 안내 (법인세법 SS18 제4호)
  IF tax_type == CORP:
    APPEND_NOTE: "환급가산금은 익금에 산입하지 아니함 (법인세법 SS18 제4호)"
```

### 13.3 개인 감면 소득 배분 (P4-02)

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
      // v2.0: 상시근로자는 전체 기준 산정 (지분율 배분 아님)
      //        공제세액만 손익분배비율에 따라 배분

    total_exemption += exemption

  // 한도 적용 (SS7: 1억원)
  IF provision = 'SS7':
    total_exemption = MIN(total_exemption, 100,000,000)

  RETURN total_exemption
```

### 13.4 개인 최저한세 (P5-01)

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
  //   성실사업자 의료비/교육비 (SS122의3)
  //   기장세액공제 (소득세법 SS56의2)
  //   외국납부세액공제 (소득세법 SS57)
  //   자녀세액공제 (SS59의2), 연금저축세액공제 (SS59의3)

  // 최저한세 대상:
  //   SS6, SS7, SS24, SS29의8, SS30의4 등 조특법 감면/공제
  //   성실신고확인비용 (SS126의6)
  //   착한 임대인 (SS96의3)

  RETURN { min_tax, max_deductible }
```

### 13.5 결손금 소급공제 NPV 비교 (M4-10) [v2.0 신규]

```
FUNCTION compare_loss_treatment(tax_type, loss_amount, prev_data, discount_rate):

  // 소급공제 환급세액
  loss_ratio = TRUNCATE(loss_amount / prev_data.tax_base, 3)
  carryback_refund = TRUNCATE(prev_data.computed_tax * loss_ratio, -1)

  // 이월공제 NPV (향후 5년 예상 기준)
  IF tax_type == CORP:
    carry_limit_rate = IF sme THEN 1.00 ELSE 0.80
    carry_period = 15  // 2020이후: 15년, 이전: 10년
  ELSE:  // INC
    carry_limit_rate = 1.00
    carry_period = 10

  carryforward_npv = 0
  FOR year = 1 TO 5:
    estimated_income = estimate_future_income(year)
    deductible = MIN(loss_remaining, estimated_income * carry_limit_rate)
    tax_savings = deductible * estimated_marginal_rate(year)
    carryforward_npv += TRUNCATE(tax_savings / (1 + discount_rate)^year, -1)
    loss_remaining -= deductible

  RETURN {
    carryback_refund,
    carryforward_npv,
    recommendation: IF carryback_refund > carryforward_npv THEN "CARRYBACK" ELSE "CARRYFORWARD"
  }
```

---

## 14. 버전 이력

| 버전 | 일자 | 변경 내용 | 작성자 |
|------|------|----------|--------|
| v1.0 | 2026-02-17 | 초안 작성 - 전체 아키텍처, 모듈설계, API, 계산/검증 엔진, 구현순서 | Development Design Agent |
| **v2.0** | **2026-02-17** | **Gap Analysis(74%) 전수 대응. 주요 변경: (1) M4 서브모듈 14->22개 확장 (M4-08 이월결손금 재검토, P4-09~P4-12 개인 전용 4개 신규), (2) 계산 공식 F-INC-07~F-INC-11 5개 추가, (3) 검증 규칙 83->107개 확장 (LP04 2025년 상호배제 Hard Fail, XP01~XP05 개인 교차검증 5건, X14~X21 보고서 검증 8건 등), (4) 세율 테이블 2018~2025 확장 (법인 ~2022 구세율, 소득세 ~2020 7단계), (5) 상호배제 매트릭스 6->10규칙 (SS30의4 관련 4건 추가), (6) 보고서 7->8섹션 (section_h 감사로그), (7) 기납부세액 4분류(중간예납/원천징수/확정/분납), (8) M6 번호 정리 (M6-03 지방소득세), (9) 환급가산금 익금불산입 안내 추가, (10) 구현일정 6주->7주 확장** | **Gap Detector Agent** |

---

## 참조 문서

| 문서 | 위치 | 설명 |
|------|------|------|
| 요구사항 계획서 | `docs/01-plan/features/tax-refund-system.plan.md` | PM Agent 작성, 55개 FR, 6 Epic |
| 상세설계서 초안 | `docs/02-design/features/환급액계산프로그램.design.md` | Design Agent v2.0, 2,288줄 |
| 용어 정의서 | `old-doc/01-glossary.md` | 9개 섹션, 세법/시스템 용어 |
| 엔티티 스키마 | `old-doc/02-entity-schema.md` | 83개 테이블 + 뷰 1개 상세 컬럼 |
| 엔티티 관계 | `old-doc/03-entity-relationship.md` | ERD, 데이터 흐름, TAX_TYPE 분기 |
| 이전 설계서 v1.0 | `old-doc/04-development-design.md` | 1,211줄, 초기 아키텍처 |
| 설계 검증 보고서 | `old-doc/05-design-validation-report.md` | 82/100점, 16개 이슈 |
| Gap 분석 보고서 | `old-doc/06-gap-analysis-prompt-coverage.md` | 74%, 34개 Gap |
| 법인사업자 프롬프트 | `법인사업자-프롬프트_v1.3.md` | 40개 점검항목 |
| 개인사업자 프롬프트 | `개인사업자-프롬프트_v1.3.md` | 37개 점검항목 |
| 외부자료 원본 | `외부자료_v4_0.md` | v4.0 원본 상세 설계 |
