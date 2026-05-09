---
name: regression-scope-assessment
description: "Regression scope decision guide — use before building a QA test plan whenever you are verifying a bug fix or new feature. Determines which areas of the codebase are at risk of regression based on the change type and blast radius, maps changed modules to their dependants, and produces a scoped, prioritised regression checklist with 'must test' and 'nice to test' tiers. Invoke this skill after reading the issue and the PR diff but before calling qa-acceptance-testing, so the regression scope feeds directly into the test plan."
---

# Regression Scope Assessment

This skill answers the question: **beyond the acceptance criteria, what else could this change have broken, and is it worth testing?**

Call this skill **before** building your QA test plan. Its output — a scoped regression checklist — feeds into `qa-acceptance-testing` Step 3 (Build the test plan).

---

## When to invoke this skill

Invoke this skill when any of the following are true:

- You are verifying a bug fix (the fix could have introduced regressions in adjacent logic)
- You are verifying a feature that touches shared modules (auth, routing, form submission, DB migrations, API contracts)
- The PR diff touches more than one file or more than one layer (e.g. backend + frontend)
- The issue description or PR description mentions "could affect", "be careful of", or "also updated"

Skip this skill (proceed directly to `qa-acceptance-testing`) only when:

- The change is a pure documentation update or comment change
- The change is scoped to a brand-new file or module with zero dependants (confirmed by grep)

---

## Step 1 — Classify the change type

Read the PR diff or issue description and identify the **primary change type**. A single PR may have more than one type; apply all that match.

| Change type | Indicators |
|---|---|
| **Auth / security** | Changes to middleware, JWT handling, session management, RBAC checks, route guards, token validation |
| **API contract** | New, removed, or renamed request/response fields; changed HTTP method or status codes; versioning updates |
| **Service / business logic** | Core domain functions, calculation logic, state machine transitions, background jobs |
| **UI / component** | React/Vue/template components, form controls, layout, CSS that affects functionality |
| **Database / migration** | Schema changes, migration files, model changes, ORM queries |
| **Configuration / environment** | env var handling, feature flags, build config, deployment scripts |
| **Cross-cutting utility** | Shared helpers, date/time utils, error handling, logging infrastructure |

Record the change types before continuing. This determines the risk zones in Step 2.

---

## Step 2 — Map change type to risk zones

For each identified change type, the following modules and integration points are in the **primary risk zone** (must test) and **secondary risk zone** (nice to test).

### Auth / security changes

| Risk tier | What to check |
|---|---|
| **Must test** | All routes that the changed middleware applies to — verify a request can still succeed AND that an unauthorized request is still rejected |
| **Must test** | Any token or session lifecycle: creation, refresh, expiry, logout |
| **Must test** | Permission boundaries: confirm a lower-privilege role cannot access a higher-privilege resource |
| **Nice to test** | Adjacent middleware (logging, rate limiting) that wraps the changed auth layer |
| **Nice to test** | Public (unauthenticated) routes — confirm they were not accidentally gated |

### API contract changes

| Risk tier | What to check |
|---|---|
| **Must test** | All callers of the changed endpoint (search the codebase for the route path or function name) |
| **Must test** | The changed field in both success and error responses |
| **Must test** | Any downstream consumers (background jobs, webhooks, client-side parsers) that rely on the old contract |
| **Nice to test** | Related endpoints in the same resource group (e.g. if `/users/:id` changed, also spot-check `/users`) |
| **Nice to test** | SDK or generated client code if an OpenAPI/GraphQL schema was regenerated |

### Service / business logic changes

| Risk tier | What to check |
|---|---|
| **Must test** | The primary use case the fix addresses (the exact scenario from the bug report) |
| **Must test** | The inverse case — what happens when the fixed condition is NOT present |
| **Must test** | Any function that calls the changed function (one level up the call stack) |
| **Nice to test** | Edge cases adjacent to the fix: null input, empty collection, boundary values |
| **Nice to test** | Other code paths through the same module that the change did not intend to alter |

### UI / component changes

| Risk tier | What to check |
|---|---|
| **Must test** | The component's primary interaction (submit, click, navigate) in the happy path |
| **Must test** | Form validation: valid input succeeds, invalid input fails with correct error messages |
| **Must test** | Adjacent steps in the same flow (e.g. if wizard Step 2 changed, verify Step 1 → Step 2 → Step 3 still works end-to-end) |
| **Nice to test** | Responsive layout at the affected breakpoints |
| **Nice to test** | Other pages that import or compose the changed component |

### Database / migration changes

| Risk tier | What to check |
|---|---|
| **Must test** | The migration runs without error on a clean schema AND on an existing populated DB (if testable) |
| **Must test** | Any ORM query that references the altered table or column |
| **Must test** | Foreign key or constraint logic: verify referential integrity still holds after migration |
| **Nice to test** | Rollback path (if the migration has a `down` function) |
| **Nice to test** | Performance of queries on the altered table if row counts are significant |

### Configuration / environment changes

| Risk tier | What to check |
|---|---|
| **Must test** | The feature or code path that depends on the changed config, with the new config value active |
| **Must test** | That missing or malformed config fails gracefully (error message, not a crash) |
| **Nice to test** | Other env-dependent paths that use adjacent config keys |

### Cross-cutting utility changes

| Risk tier | What to check |
|---|---|
| **Must test** | All direct call sites of the changed utility function — grep the codebase and confirm the top 3–5 callers still work |
| **Must test** | The specific bug scenario the utility fix addressed |
| **Nice to test** | Any caller that applies the utility to unusual input (empty string, nil, very large values) |

---

## Step 3 — Assess blast radius

Before finalising the regression list, answer these questions. They determine how wide to cast the net.

| Question | Low blast radius | High blast radius |
|---|---|---|
| How many files were changed? | 1–2 files | 5+ files |
| Is the changed code on a hot path? | Called in edge cases only | Called on every request / page load |
| Does the change touch shared infrastructure? | Local to one feature | Auth, DB connection, global state, error handler |
| Are there existing automated tests? | Test suite covers the area | No tests; change is untested territory |
| Has this area regressed before? | No recent incidents | Recent bug history in this area |

**Scoring:**
- 0–1 high-blast answers → **Narrow scope**: run the must-test items only
- 2–3 high-blast answers → **Standard scope**: must-test + select nice-to-test items
- 4–5 high-blast answers → **Wide scope**: must-test + all nice-to-test items + flag for human review

Record the blast radius score before producing the final checklist.

---

## Step 4 — Produce the regression checklist

Combine the risk zone items from Step 2 with the blast radius from Step 3. Format the checklist as follows:

```markdown
## Regression Scope — [Issue/PR identifier]

**Change types:** [auth | API contract | service logic | UI | migration | config | utility]
**Blast radius:** [Narrow | Standard | Wide] ([N]/5 high-blast indicators)

### Must Test

- [ ] [Specific scenario — e.g. "POST /api/login with valid credentials returns 200 and a JWT"]
- [ ] [Specific scenario]
- [ ] ...

### Nice to Test (include if time permits / Wide scope)

- [ ] [Specific scenario]
- [ ] ...

### Excluded (out of scope for this change)

- [Area or module] — [one-line reason why it is not at risk]
```

Add this checklist to the QA issue as a comment or document (key: `regression-scope`). Then pass the "Must Test" items into `qa-acceptance-testing` Step 3 (Build the test plan) as additional test cases alongside the acceptance criteria.

---

## Step 5 — Flag high-risk scenarios

Before handing off to `qa-acceptance-testing`, call out any items that warrant human review:

- **Security-adjacent regressions** — if any must-test item involves an auth boundary or data access control, flag it explicitly and note that a false-pass here has security consequences.
- **Migration regressions** — if the must-test list includes a DB migration, note that production rollback may not be possible if regression is found post-deploy.
- **No existing tests** — if must-test items have no automated coverage, note this in the regression scope comment so the developer can be prompted to add tests.

Format:

```markdown
### Flags

⚠️ **[Auth/Security]** — [specific item] has security consequences if it regresses silently. Manual verification required; do not defer to a passing CI run alone.

⚠️ **[No test coverage]** — [specific item] has no existing automated tests. Post-verdict, create a follow-up issue to add regression coverage.
```

---

## Quick reference: change type → primary risk zone

| Change type | Primary risk zone (must test) |
|---|---|
| Auth / security | All guarded routes, token lifecycle, permission boundaries |
| API contract | All callers, changed response fields, downstream consumers |
| Service logic | The fixed scenario, its inverse, one level up the call stack |
| UI / component | Primary interaction, form validation, adjacent steps in same flow |
| DB / migration | Migration run, ORM queries on altered table, referential integrity |
| Config / env | The dependent code path, graceful failure on missing config |
| Cross-cutting utility | Top 3–5 direct call sites, the specific bug scenario |

---

## Related skills

| Skill | When to use it from this skill |
|---|---|
| `qa-acceptance-testing` | After producing the regression checklist — feeds the regression items into the test plan |
| `webapp-testing` | Executing automated regression tests against a running dev server |
| `sdlc-process` | Checking SDLC gates when regression scope is wide or change is security-adjacent |
| `completion-evidence-gate` | Confirming evidence requirements are met before issuing a QA verdict |

---

*TEC Custom Skill — maintained by the Deltek Technical Services Engineering team.*
