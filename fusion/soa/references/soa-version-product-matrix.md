# Oracle SOA Version and Product Matrix

## Scope

Use this matrix to map Oracle SOA products from `11g` through the latest line.
As of `2026-05-20`, treat Oracle SOA Suite `14.1.2` as the latest SOA Suite documentation line.

## Release Mapping

| Label | Common version line(s) | Notes |
|---|---|---|
| 11g | 11.1.1.x | Legacy SOA generation; includes many product-specific docs now used for migration reference. |
| 12c | 12.1.3, 12.2.1.4 | Widely used modernization baseline in long-lived middleware estates. |
| 14c | 14.1.2 | Latest SOA Suite line in Oracle docs listed below. |

## Oracle SOA Product Coverage (Docs-Listed)

Legend:
- `Listed` = product appears in referenced documentation index for that release line.
- `Not listed in cited index` = not visible in the specific index used here; this is not a support verdict.

| Product | 11g | 12.2.1.4 | 14.1.2 | Notes |
|---|---|---|---|---|
| Oracle SOA Suite Core | Listed | Listed | Listed | Core SOA platform line. |
| Oracle SOA Adapters | Listed | Listed | Listed | Adapter documentation appears in all three lines. |
| Oracle BPEL Process Manager | Listed | Listed | Listed | Core process orchestration component. |
| Oracle Mediator | Listed | Listed | Listed | Core mediation component. |
| Oracle Human Workflow | Listed | Listed | Listed | Workflow/human task component. |
| Oracle Business Rules | Listed | Listed | Listed | Rules component in SOA stack. |
| Oracle B2B | Listed | Listed | Listed | B2B integration component. |
| Oracle Service Bus | Listed | Listed | Listed | Listed under related products for newer lines. |
| Oracle Business Process Management (BPM) | Not listed in cited 11g SOA index | Listed | Listed | Listed on SOA docs landing for newer lines. |
| Oracle Enterprise Scheduler (ESS) | Not listed in cited 11g SOA index | Listed | Listed | Listed as related product on SOA docs landing. |
| Oracle Managed File Transfer (MFT) | Not listed in cited 11g SOA index | Listed | Listed | Listed as related product on SOA docs landing. |
| Oracle SOA Suite for Healthcare Integration | Not listed in cited 11g SOA index | Listed | Not listed in cited 14.1.2 index | Verify exact availability via current product docs/certification. |
| Oracle Business Activity Monitoring (BAM) | Listed | Not listed in cited 12.2.1.4 index | Not listed in cited 14.1.2 index | Primarily visible in 11g SOA docs index. |
| Oracle Complex Event Processing | Listed | Not listed in cited 12.2.1.4 index | Not listed in cited 14.1.2 index | Legacy component listed in 11g index. |
| Oracle SOA Suite Spring | Listed | Not listed in cited 12.2.1.4 index | Not listed in cited 14.1.2 index | Legacy 11g documentation item. |
| Oracle Web Services (11g page item) | Listed | Not listed in cited 12.2.1.4 index | Not listed in cited 14.1.2 index | Use as legacy documentation reference only. |

## Usage Rules

- Use this matrix to answer `what is listed where` questions quickly.
- Perform separate certification checks before asserting deployment support.
- When the estate is on 11g, require explicit product-by-product migration analysis.

## Sources

- Oracle SOA documentation landing (14.1.2 / 12.2.1.4 product lists): https://docs.oracle.com/en/middleware/soa-suite/
- Oracle SOA Suite 11g documentation index: https://www.oracle.com/middleware/technologies/soa-suite-11gdocumentation.html
- Oracle SOA product page: https://www.oracle.com/middleware/technologies/soa-suite.html
- Oracle Fusion Middleware certification page: https://www.oracle.com/middleware/technologies/fusion-certification.html
