# Java Version Matrix (6 to Latest)

## Scope

Use this reference for Java version mapping and upgrade planning from `Java 6` through the latest release.
As of `2026-05-20`:
- Latest feature release: `Java 26` (GA 2026-03-17)
- Latest LTS release: `Java 25`

## Release Timeline Summary

| Java line | GA date | LTS | Operational posture |
|---|---:|:---:|---|
| 6 | 2006-12-12 | Yes | Legacy only; migrate off. |
| 7 | 2011-07-11 | Yes | Legacy only; migrate off. |
| 8 | 2014-03-18 | Yes | Common legacy baseline in enterprise estates. |
| 9 | 2017-09-21 | No | Feature line; do not target for long-term support. |
| 10 | 2018-03-20 | No | Feature line. |
| 11 | 2018-09-25 | Yes | Major long-lived migration target from Java 8. |
| 12 | 2019-03-19 | No | Feature line. |
| 13 | 2019-09-17 | No | Feature line. |
| 14 | 2020-03-17 | No | Feature line. |
| 15 | 2020-09-15 | No | Feature line. |
| 16 | 2021-03-16 | No | Feature line. |
| 17 | 2021-09-14 | Yes | Stable LTS modernization target. |
| 18 | 2022-03-22 | No | Feature line. |
| 19 | 2022-09-20 | No | Feature line. |
| 20 | 2023-03-21 | No | Feature line. |
| 21 | 2023-09-19 | Yes | Current broadly adopted LTS in many estates. |
| 22 | 2024-03-19 | No | Feature line. |
| 23 | 2024-09-17 | No | Feature line. |
| 24 | 2025-03-18 | No | Feature line. |
| 25 | 2025-09-16 | Yes | Latest LTS as of 2026-05-20. |
| 26 | 2026-03-17 | No | Latest feature release as of 2026-05-20. |

## Upgrade Guidance

- Prefer LTS-to-LTS planning: `8 -> 11 -> 17 -> 21 -> 25`.
- Treat Java 6/7 migrations as high-risk legacy modernization work.
- For Java 8 to 11 moves, explicitly validate removed Java EE/CORBA modules and old runtime assumptions.
- For Java 9+ moves, validate module/encapsulation impact (`jdeps`, internal API usage).

## Sources

- Java release timeline: https://www.java.com/en/releases/
- Oracle Java SE support roadmap: https://www.oracle.com/java/technologies/java-se-support-roadmap.html
- OpenJDK JDK project release train: https://openjdk.org/projects/jdk/
- JDK 11 migration guide: https://docs.oracle.com/en/java/javase/11/migrate/index.html
- JDK 26 migration guide: https://docs.oracle.com/en/java/javase/26/migrate/index.html
- JEP 320 (remove Java EE/CORBA modules): https://openjdk.org/jeps/320
- JEP 261 (module system): https://openjdk.org/jeps/261
- JEP 403 (strong encapsulation): https://openjdk.org/jeps/403
