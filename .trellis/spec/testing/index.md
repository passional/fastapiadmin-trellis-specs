# Testing Specifications

These specs describe the tests and commands that actually exist. A recommended
test is labeled as a coverage target; it is never presented as current
coverage.

| Spec | Covers |
|---|---|
| [Backend Testing](./backend-testing.md) | pytest configuration, TestClient/async patterns, SQLite and Redis fixtures, isolation, the `test_migrations.py` drift guard, and behavior assertions |
| [Frontend Testing](./frontend-testing.md) | Admin Web Vitest/jsdom patterns, `route-invariants.spec.ts` as the routing contract, the App's current no-runner state, and built-artifact (dist) verification |
| [Contract and Deployment Testing](./contract-and-deployment-testing.md) | REST/auth/realtime contracts, generator regressions, Docker/Nginx/config smoke checks |

Select tests by changed boundary, then run the package's existing commands
from its own directory. Markdown/spec-only changes need spec validation, not
unrelated product builds.

