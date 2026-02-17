# Design Validator Agent Memory

## Project: TaxServiceENTEC

### Key Document Paths (old-doc structure)
- Plan: `docs/01-plan/features/tax-refund-system.plan.md` (55 FRs)
- Dev Design: `old-doc/04-development-design.md` (v1.0, 1211 lines, 12 sections)
- Glossary: `old-doc/01-glossary.md`
- Schema: `old-doc/02-entity-schema.md` (83 tables + 1 view)
- ERD: `old-doc/03-entity-relationship.md`
- Validation Report: `old-doc/05-design-validation-report.md`
- External Ref: `외부자료_v4_0.md` (v4.0 original spec, ~6000 lines)

### Validation Results (2026-02-17, 04-development-design.md)
- **Score: 82/100** (3 Critical, 9 Major, 4 Minor)
- FR coverage: 51/55 (93%) - FR-25, FR-29, FR-51, FR-52 gaps
- Schema: 83/83 (100%) - full match
- Modules: 42/52 (81%) - P4-09~14, M4-08/12/13/14, P1-*, M2-10~13 missing
- API: 8/8 (100%) - full match
- Formulas: 33/39 (85%) - F-INC-07~11, F-COM-01, F16, F19 missing
- Validation rules: 75/95+ (79%) - XP01~05, X14~21, VP04~06 etc missing

### Known Critical Issues
1. **req_id prefix conflict**: Design/glossary use C/I, external ref uses C/P
2. **P4-09~P4-14 missing**: 6 INC-only credit modules not designed (P4-09 critical for FR-31)
3. **LP04 rule missing**: 2025+ SS6+SS29의8 mutual exclusion Hard Fail not in validation engine

### Patterns Confirmed
- External ref v4.0 has more validation rules than design (84+ vs 71 common)
- M6 module numbering inconsistency: M6-03 vs M6-05 vs M6-06 overlap
- RF_* table count: 26 is authoritative (external ref has internal inconsistency 22 vs 26)
