# Design Validator Agent Memory

## Project: TaxServiceENTEC

### Key Document Paths
- Plan: `docs/01-plan/features/tax-refund-system.plan.md` (55 FRs)
- Design: `docs/02-design/features/환급액계산프로그램.design.md`
- Schema: `docs/01-plan/schema.md` (83 tables)
- Domain Model: `docs/01-plan/domain-model.md`
- Glossary: `docs/01-plan/glossary.md`
- External Ref: `외부자료_v4_0.md` (v4.0 original spec)

### Known Issues (2026-02-17 Validation)
1. **req_id prefix conflict**: Plan docs use C/I, external ref uses C/P
2. **RF_* table count**: External ref internally inconsistent (22 vs 26). Schema.md (26) is authoritative
3. **M4 submodule gap**: Design lists 14, external ref has 17+ (M4-01~M4-15, M42, P4-01~P4-09)
4. **CANCELLED state**: Defined in domain-model.md but missing from Design's RequestStatus enum
5. **Report JSON sections**: Design says "7 sections A~G" but only defines A~F. section_g_meta in external ref

### Validation Score: 78/100
- 3 Critical, 9 Warning, 4 Info items
- Implementation approved after resolving Critical items
