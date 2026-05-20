# WebLogic Version Matrix (11g to 15c)

## Scope

Use this matrix for requests that require mapping Oracle WebLogic Server versions from `11g` through the latest `15c` line.
As of `2026-05-20`, treat `15c (15.1.1.0.0)` as the latest major WebLogic line.

## Major Release Mapping

| Family label | Canonical version line | Typical version string(s) | Notes |
|---|---|---|---|
| 11g | 10.3.6 | 10.3.6, 11.1.1.9 | WebLogic 11g release 1 PS6 line. |
| 12c (early) | 12.1.3 | 12.1.3.0.0 | Common 12c baseline in legacy estates. |
| 12c (current in many estates) | 12.2.1.4 | 12.2.1.4.0 | Last 12c family patch-set line widely used for long-lived middleware stacks. |
| 14c | 14.1.1 / 14.1.2 | 14.1.1.0.0, 14.1.2.0.0 | Newer standalone lines before 15c. |
| 15c (latest) | 15.1.1 | 15.1.1.0.0 | Latest major WebLogic line as of May 2026. |

## Normalization Rules

- Map `11g` to `10.3.6`.
- Map `12c` without minor context to either `12.1.3` or `12.2.1.4`; ask which one when migration precision matters.
- Map `14c` to `14.1.1.0.0` or `14.1.2.0.0` based on exact environment inventory.
- Map `15` or `15c` to `15.1.1.0.0` unless the user specifies a different patch train.

## Upgrade Planning Heuristics

- Collect source exact version, JDK level, and platform before proposing a target.
- Prefer phased hops for very old estates (`11g` and early `12c`) when supportability is unclear.
- Treat PSU/BP details as mandatory for production cutover planning.
- Provide both a preferred path and a risk note when Oracle support details are partially unknown.

## Sources

- 11g (10.3.6): https://docs.oracle.com/middleware/11119/wls/index.html
- 12.1.3: https://docs.oracle.com/middleware/1213/wls/index.html
- 12.2.1.4.0: https://docs.oracle.com/en/middleware/fusion-middleware/weblogic-server/12.2.1.4/index.html
- 14.1.1.0.0: https://docs.oracle.com/en/middleware/standalone/weblogic-server/14.1.1.0/index.html
- 14.1.2.0.0: https://docs.oracle.com/en/middleware/fusion-middleware/weblogic-server/14.1.2/index.html
- 15.1.1.0.0 documentation set example: https://docs.oracle.com/en/middleware/standalone/weblogic-server/15.1.1/wlsig/index.html
- WebLogic technical info page referencing 15.1.1.0 docs: https://www.oracle.com/middleware/technologies/weblogic.html
