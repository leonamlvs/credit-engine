# Credit Engine Technical Plan

## Summary

The engine is implemented inside out, with rule values in validated JSON configuration and small
pure domain functions exposed to the application layer.

```text
versioned JSON rules
        ↓
Zod validation at startup
        ↓
typed, immutable RuleConfiguration
        ↓
generic condition evaluation
        ↓
cluster → job → income → penalty → limit
        ↓
pure classification use case
        ↓
request/response Zod contracts
        ↓
Express endpoint → OpenAPI → integration tests
```

The repository contains the complete classification behavior, HTTP API, OpenAPI contract, logging,
environment configuration, and verification tooling. The accepted decisions in `ASM-001`,
`ASM-002`, `ASM-007`, `ASM-008`, and `ASM-010` keep all tasks deterministic.

## Architecture and interfaces

- Organize customer contracts and use-case/HTTP code separately from pure credit-engine domain functions.
- Keep `server.ts` as the composition root: load and validate rules once, construct the classifier, inject it into `createApp`, then listen.
- Use `CustomerSchema` as the request source of truth and infer `Customer` with `z.infer`.
- Configure customer and nested location schemas to accept and preserve unknown properties. Enforce only the constraints approved in `ASM-009`; do not add age, string-content, state, uniqueness, debt-field consistency, or unknown-field rejection constraints.
- Preserve accepted additional customer properties through classification so the HTTP layer can return the accepted customer enriched with calculated data.
- Define `RuleConfigurationSchema` and infer `RuleConfiguration`; domain functions accept only validated configuration.
- Keep the internal `CoreClassification` separate from the public response mapper. It contains the selected cluster/category, calculated income/limit, and the approved decision: `true` for A–C and `false` for D under the accepted contract in `ASM-002`.
- Map the original accepted customer to a flat public response with exactly the six calculated fields approved in `ASM-002`. Monetary fields are JSON numbers in BRL units; do not expose internal calculation or evaluator metadata.
- Use the shared application-owned error envelope and the exact success, validation, and malformed-JSON behavior approved in `ASM-008`. Public errors must not expose raw Zod structures.
- Do not add dependencies, persistence, authentication, CORS, repositories, or a generalized rule-engine framework.

## Rule configuration

Use `config/rules/credit-engine.v1.json`, loaded relative to the application working directory and validated once before the server listens.

The top-level document contains:

- `schemaVersion`;
- ordered cluster rules;
- ordered job-category rules;
- the complete cluster/category income matrix;
- penalty rules.

Each cluster contains a code, display label, priority, base limit, cap, an all-of condition list, and approved decision metadata. Under the accepted response contract, A–C carry `approved: true`; `CLUSTER_D` uses an explicit `always` condition and carries `approved: false`.

Each job category contains a code, priority, multiplier, and one or more bounded keyword-match conditions, or an explicit `always` fallback for `OTHER`. Ordinary keywords use `containsAnySubstringCaseInsensitive`; only the executive acronym keywords covered by `ASM-010` use `containsStandaloneTermCaseInsensitive`. For that operator, a term is standalone when neither adjacent character is a Unicode or ASCII letter or digit; whitespace, punctuation, and underscores are delimiters. A non-fallback category matches when any of its configured keyword-match conditions succeeds, without introducing a generalized OR tree.

Conditions form a small Zod discriminated union limited to:

- numeric `greaterThanOrEqual`;
- numeric `inclusiveRange`;
- boolean `equals`;
- array `containsAny`;
- array `containsNone`;
- string `containsAnySubstringCaseInsensitive`;
- string `containsStandaloneTermCaseInsensitive`;
- `always`.

The schema validates structural safety: positive unique priorities, unique codes, compatible field/operator combinations, valid debt values, nonnegative monetary values, complete income-matrix references, and one lowest-priority fallback for clusters and job categories. Separate explicit tests verify every approved business value; the schema itself must not hard-code those values.

Cluster evaluation requires all configured cluster conditions to match. Job-category evaluation uses the bounded keyword-match behavior above. Both evaluators sort a copy by numeric priority and choose the first matching rule without mutating the configuration or customer.

## Dependency flow

```text
T1 → T2 → T3 → T4
             ├→ T5
             └→ T7 → T8 → T9
      └→ T6

T4 + T5 + T6 + T7 + T9 → T10
T1 + T10 → T11 → T12 → T13
T12 → T14
all completed work → T15
```

All tasks are complete. Their declared dependencies document the implementation sequence.

The completed dependency conditions are:

- Task 10 composes the domain capabilities completed in Tasks 4–9.
- Tasks 11–13 build the public HTTP and OpenAPI contracts on Task 10.
- Task 14 verifies repository-owned representative responses through the real HTTP stack under `ASM-001`.
- Task 15 completes documentation and the final audit.

## Decision status

No design decision or external artifact remains pending. `ASM-001`, `ASM-002`, `ASM-007`,
`ASM-008`, and `ASM-010` contain the accepted contract decisions used by the implementation and
tests.

The versioned JSON rule representation remains governed by ADR-002. The public response and HTTP
decisions are behavioral contract choices captured in the specification and do not require a new
architectural framework or dependency.
