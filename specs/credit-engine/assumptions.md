# Credit Engine Assumptions and Contract Decisions

This file records explicit interpretations and contract decisions that materially affect Credit
Engine behavior. It must not be used to invent requirements absent from the project specification.

## ASM-001 — Repository-owned integration scenarios

The project owns its representative integration scenarios and expected responses. They are derived
from the accepted contract and versioned rule configuration.

### Status

Accepted and verified.

### Policy

- Exercise every cluster through the real HTTP stack.
- Keep expected values explicit and reviewable in the repository.
- Trace expected behavior to the project specification and validated rule configuration.

### Effect

- `REQ-036` and `AC-028` are satisfied by the repository-owned integration scenarios.
- The test suite has no dependency on an external fixture.

## ASM-002 — Public classification response contract

The endpoint returns the accepted customer enriched with the calculated classification fields
defined below.

### Status

Accepted and implemented.

### Accepted contract

- `approved = true` for `CLUSTER_A`, `CLUSTER_B`, and `CLUSTER_C`.
- `approved = false` for `CLUSTER_D`.
- Return the original accepted customer object, enriched at the top level with exactly these calculated fields:
  - `cluster_id`: a JSON string containing the selected cluster code;
  - `cluster_name`: a JSON string containing the selected cluster display name;
  - `job_category`: a JSON string containing the selected job-category code;
  - `monthly_income`: a JSON number expressed directly in BRL units;
  - `approved`: a JSON boolean;
  - `approved_limit`: a JSON number expressed directly in BRL units.
- Keep the response flat. Do not wrap it in a success envelope.
- Do not expose internal calculation or rule-engine metadata such as `base_limit`, `cluster_cap`, `job_multiplier`, `penalty_factor`, matched conditions, or internal matched rule IDs.
- Monetary values use BRL major units: `6500` represents R$ 6.500,00 and `6500.25` represents R$ 6.500,25. Do not use formatted currency strings or integer minor units, and do not require trailing decimal zeroes in JSON.

### Effect

- `REQ-026`, `REQ-034`, `AC-025`, and `AC-028` are verified against this contract.
- Runtime schemas and OpenAPI documentation use the same field definitions.

## ASM-003 — Inclusive cluster age ranges

The cluster table writes the age ranges as `25–60` and `18–65` without separate comparison operators.

### Approved interpretation

- `CLUSTER_A`: `25 <= age <= 60`.
- `CLUSTER_B`: `18 <= age <= 65`.

This interpretation is accepted.

## ASM-004 — Independent debt fields

The project contract does not define a consistency constraint between `has_market_debt` and `market_debt_types`.

### Approved interpretation

- Evaluate `has_market_debt` only where an explicit rule references it.
- Evaluate `market_debt_types` only where an explicit rule references it.
- Do not reject or normalize a customer merely because the two fields appear inconsistent.

## ASM-005 — Descriptive customer identifier

The customer schema describes `id` as a “Unique identifier”, while the API is explicitly stateless and persists nothing between calls.

### Approved interpretation

Treat `id` as the identifier accepted with the current record. Do not infer global or cross-request uniqueness enforcement.

## ASM-006 — Single default-debt penalty activation

The configured `DEFAULT_DEBT_PENALTY` triggers when `credit_default` **or** `loan_default` is present in `market_debt_types` and applies an effect of `×0.5`.

### Approved interpretation

The rule activates once when either or both default debt types are present. The two values are not independent penalty applications, and the factor is not stacked.

When the rule does not trigger, `penalty_factor` is the multiplicative identity `1.0` required to evaluate the configured formula.

## ASM-007 — Rounding midpoint semantics

The project contract requires `round_to_nearest_100` and defines exact midpoint behavior here.

### Approved interpretation

- For nonnegative monetary amounts, exact midpoint ties round upward to the next hundred.
- `1,750` rounds to `1,800`.
- `3,750` rounds to `3,800`.

This is the accepted midpoint interpretation.

## ASM-008 — HTTP success and error contract

The project contract specifies the route and defines success and error behavior here.

### Approved interpretation

- A successful `POST /customers/classify` returns HTTP `200 OK` with the enriched customer object directly as the response body. It does not return `201 Created`, a generic success envelope, or a creation message.
- Missing required fields, wrong declared types, invalid enum values, and invalid score ranges return HTTP `400 Bad Request`.
- Schema-validation failures use this application-owned public envelope:

  ```json
  {
    "error": {
      "code": "VALIDATION_ERROR",
      "message": "Request validation failed",
      "details": [
        {
          "path": "<field path>",
          "message": "<issue description>"
        }
      ]
    }
  }
  ```

- Validation details expose stable field paths and human-readable messages derived from validation failures without exposing unnecessary Zod-specific structure.
- Malformed JSON returns HTTP `400 Bad Request` using the same public envelope schema:

  ```json
  {
    "error": {
      "code": "MALFORMED_JSON",
      "message": "Request body contains invalid JSON",
      "details": []
    }
  }
  ```

- For malformed JSON, `details` remains present and is always an empty array. It must not expose Express, JSON-parser, or other parser-internal information.
- Malformed JSON does not introduce a separate error-response schema.

These are the accepted HTTP status and response details.

## ASM-009 — Customer-schema interpretation

The project contract defines one customer schema and requires integration tests for invalid or missing fields. Location properties use dotted names in the field table.

### Approved interpretation

- Treat every listed customer field as part of the request schema and a missing listed field as invalid for integration coverage.
- Treat `location.city`, `location.state`, and `location.region` as properties of a nested `location` object.
- Enforce only specified constraints: declared types, the score range, the region values, and the five valid market-debt types.
- Do not invent additional constraints such as a state enum, a general age range, non-empty-string rules, or cross-field consistency validation.

## ASM-010 — Acronym keyword matching

The project contract requires case-insensitive substring matching and lists `COO`
as an `EXECUTIVE` keyword while also listing `Coordinator` as a
`SENIOR_PROFESSIONAL` keyword.

Literal substring evaluation would cause `Coordinator` to match `COO`
at the higher executive priority, making the explicit `Coordinator`
senior keyword ineffective.

### Approved interpretation

Executive acronym keywords (`CEO`, `CFO`, `CTO`, `COO`, `CIO`, `CMO`,
and `VP`) match only as standalone terms.

A standalone term is not immediately adjacent on either side to a Unicode or
ASCII letter or digit. Whitespace, punctuation, and underscores delimit the
term.

Consequently, `COO`, `COO Brazil`, `coo`, `(COO)`, `ex-COO`, `COO/CTO`,
and `COO_Brazil` match the standalone executive acronym. `Coordinator`,
`myCOO`, `COO2`, and `COOOperations` do not match the standalone term `COO`.

Non-acronym keywords continue to use the specified
case-insensitive substring behavior.

Therefore `Coordinator` does not match `COO` and remains eligible for
`SENIOR_PROFESSIONAL`.

Category priority remains unchanged.
