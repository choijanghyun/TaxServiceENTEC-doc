# Gap Detector Agent Memory

## Project: TaxServiceENTEC

### Key Architecture Facts
- Java/Spring Boot, PostgreSQL 15, Clean Architecture (4 layers)
- Tax types: CORP (법인세) and INC (종합소득세), branched via TaxType enum
- M1-M6 module pipeline + MX-01/02/03 cross-cutting modules
- M4 submodules: 22 (expanded from 14 in v2.0)
- Strategy Pattern for CreditCalculator interface
- Truncation-only policy: MoneyAmount.truncate10(), TaxRate.truncate3()

### Gap Analysis History
- v1.0 -> v2.0: 68% match rate, 22 Critical / 16 Warning / 8 Info
- Key gaps: Individual-only deductions (P4-01~P4-10), corporate tax adjustment (M4-11), loss carryback NPV, progressive tax rate logic, prepaid tax detail, joint business allocation
- Design v2.0 addressed all 22 Critical and most Warning issues

### Critical Design Patterns
- P4-04 MedicalEducation: minTaxSubject=FALSE (frequently confused with P4-05 which IS subject)
- P4-06 BookkeepingCredit: NOT subject to Article 127 overlap exclusion (소득세법, not 조특법)
- Individual income allocation: exemption = computedTax x (exemptIncome / aggregateIncome) x rate
- Joint business: ownership ratio applies to income allocation, NOT to employee count
- Loss carryforward: CORP 80% limit (SME 100%), 15 years; INC 10 years

### File Paths
- Design: docs/02-design/features/환급액계산프로그램.design.md
- Dev Design v1.0: old-doc/04-development-design.md (1,212 lines, 12 sections)
- Dev Design v2.0: old-doc/07-development-design-v2.md (~1,520 lines, 14 sections)
- Gap Analysis (prompt coverage): old-doc/06-gap-analysis-prompt-coverage.md (34 gaps)
- Analysis: docs/03-analysis/features/환급액계산프로그램.analysis.md
- Individual prompt: 개인사업자-프롬프트_v1.3.md (~1,420 lines)
- Corporate prompt: 법인사업자-프롬프트_v1.3.md (~2,000+ lines)
- Previous validation: docs/02-design/design-validation-report.md (78/100)
- Schema: old-doc/02-entity-schema.md (83 tables)
- Entity Relations: old-doc/03-entity-relationship.md
- Glossary: old-doc/01-glossary.md (9 sections)
