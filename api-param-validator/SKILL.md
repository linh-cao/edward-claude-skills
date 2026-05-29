---
name: api-param-validator
description: 'Automates API parameter data validation testing for Garoon APIs using Postman + Newman. Use this skill whenever the user wants to: generate Postman test cases for API parameters, validate path/query/body parameters with invalid data, run automated API validation with Newman, create test collections for GET/POST/DELETE endpoints, or test data types like wrong type, special characters, out-of-range values, invalid formats. Trigger on phrases like "test API params", "validate parameters", "generate Postman collection", "auto test API", "run API validation", "test data validation for endpoint", or any mention of testing parameters with various invalid data types. Always use this skill when the user provides an API spec and asks to test it — even if they do not explicitly say "skill".'
---

# API Parameter Validator

Automates data validation testing for Garoon API parameters. Given a spec/testspec + endpoint info, generates a complete Postman collection covering standard invalid data scenarios, runs it via Newman, and produces a pass/fail report.

---
## Safety and Runtime Rules

- Never print, log, or save real tokens/session cookies in generated files or reports.
- Always use `{{token}}` or another environment variable for authentication.
- Do not hard-code credentials into `collection.json`.
- For POST and DELETE requests, assume the request may modify data.
- Warn the user before running POST/DELETE against a non-test environment.
- If DELETE requires an existing resource ID, ask the user for a disposable test resource ID.
- If POST creates data, prefer using clearly identifiable test data and include cleanup notes when applicable.

---

## How to Invoke This Skill

Prompt format:
```
Test API params for [METHOD] [endpoint] on [site].
Spec: [paste content here, or attach file/screenshot]
Testspec: [attach file if available — optional]
Token: [bearer token or cookie value]
```

Example:
```
Test API params for GET /api/v1/schedule/events on https://example.garoon.com
Token: Bearer eyJhbGci...
Spec:
event_id (path, integer, required): ID of the event
limit (query, integer, optional): max 100, min 1
start_datetime (query, datetime, optional): ISO 8601 format
```

---

## Step 0 — Check Required Inputs

Before proceeding, verify these are present. If anything is missing, ask the user before continuing:

| Input | Required | Notes |
|---|---|---|
| HTTP method | YES | GET / POST / DELETE |
| Endpoint path | YES | e.g. `/api/v1/schedule/events/{event_id}` |
| Garoon site URL | YES | e.g. `https://example.garoon.com` |
| API spec | YES | See "Providing Spec" below |
| Auth token | YES | Bearer token or session cookie |
| Request body schema | POST only | Required if method is POST |
| Testspec file | Optional | Contains test cases + expected results; if provided, will be combined with spec to determine accurate expected results |

### Providing Spec (when direct link sharing is restricted)

Ask the user to choose one of these methods:

1. Paste content directly — Copy text from sharedoc into the prompt (preferred)
2. Attach .md / .txt file — Export sharedoc to file and attach
3. Screenshot — Attach a screenshot; Claude will read it

If spec is unclear or ambiguous in places, see "Clarifying the Spec" section before generating.

---

## Step 1 — Parse the Spec

Extract and build a structured parameter map:

```json
{
  "path_params": [
    { "name": "event_id", "type": "integer", "required": true, "constraints": {} }
  ],
  "query_params": [
    { "name": "limit", "type": "integer", "required": false, "constraints": { "min": 1, "max": 100 } },
    { "name": "start_datetime", "type": "datetime", "required": false, "constraints": { "format": "ISO8601" } }
  ],
  "body_fields": []
}
```
For each parameter, note:
- Name and type
- Required / optional
- Constraints: min, max, enum, format, pattern
- Any special business logic mentioned in spec

### Nested Body Parsing (POST)

For POST request bodies, recursively flatten ALL nested fields into leaf paths,
regardless of field names. Use this notation:

- Object field      → <parent>.<child>
- Array of objects  → <array>[].<child>
- Array of values   → <array>[]

Apply this to whatever field names appear in the actual spec/testspec/body —
the names below are only examples of the notation, not fixed fields.

Example (body with fields "mentions", "attachmentIds"):
  mentions[].id        (string)
  mentions[].type      (enum)
  attachmentIds[]      (array of string)
  data                 (string)

Record each leaf field with its type, exactly like top-level params.
If the spec does not describe a nested field's type, infer it from the
example body or testspec; if still unclear, ask the user.

Add each flattened leaf field as an entry in the `body_fields` array of the
parameter map (same structure as path_params/query_params: name, type,
required, constraints). The leaf path is the field "name".

If `references/garoon_glossary.md` exists -> read it now to understand Garoon-specific terms in the spec.

---

## Step 2 — Parse Testspec (if provided)

If the user attaches a testspec file, read and extract the following for each test case:
- Parameter name
- Input value / data label
- Expected result (HTTP status code, error message if specified)

Then cross-reference with the parameter map from Step 1:

### Merge expected results
For each test case in the testspec:
- If it matches a parameter + data type/label already in sample_data -> use the expected result from testspec instead of the default 4xx
- If testspec contains a case not covered by sample_data -> add it as a new test case

Note: the final merge is applied in Step 4 when test cases are generated; this step only records the testspec expected results and detects conflicts.

### Conflict detection
Check for conflicts between spec and testspec and report them before proceeding. Common conflict types:
- Spec says param X is required, but testspec has a case omitting X with expected result 200
- Spec says min=1, but testspec expects value=0 to return 200 (meaning 0 is valid)
- Spec says format is ISO 8601, but testspec expects a different format to be valid
- Spec defines a constraint, but testspec expected result contradicts it

Report each conflict in this format:
```
  Conflict detected — [param_name]
  Spec     : [what the spec says]
  Testspec : [what the testspec says]
  Question : [specific question for the user to clarify]
```

Stop and wait for user confirmation before generating the collection if any conflict is found.
If no conflicts are found, proceed to Step 3 as normal.

---

## Step 3 — Load Sample Data

Read from `sample_data/` based on each parameter's type:

| Type | File |
|---|---|
| integer / number | `sample_data/integer.json` |
| string | `sample_data/string.json` |
| date / datetime | `sample_data/datetime.json` |
| email | `sample_data/email.json` |
| boolean | `sample_data/boolean.json` |
| id (numeric identifier) | `sample_data/id.json` |
| enum | `sample_data/enum.json` |

Each file contains labeled invalid test values. Additionally, for each parameter with `min`/`max` constraints, append out-of-range values dynamically.

---

## Step 4 — Generate Test Cases

### Build a Valid Baseline Request
Rules:
- Required path parameters must use existing valid values.
- Required query/body parameters must use valid values.
- Optional parameters should be omitted in the minimal happy path unless the spec requires them.
- If the spec provides examples, use them first.
- If no valid value can be inferred, ask the user.

Default valid value strategy:
- integer with min/max: use a value inside the range
- string: use `"test"` unless pattern/enum requires another value
- enum: use the first valid enum value
- datetime: use the exact format required by the spec
- email: use `test@example.com`
- boolean: use `true`
- id: ask the user for an existing valid ID unless the spec provides one

### Structure

For each invalid value per parameter, create one test case:
- All other parameters use their valid default values
- Only the target parameter has the invalid value
- Expected result priority order: (1) testspec if provided → (2) spec constraints → (3) default 4xx from sample_data

Also include:
- 1 happy path case: all params valid, all required params present → expect 2xx

### Nested Body Field Rules

These rules apply to any nested field, regardless of its name:

- Test each leaf field independently; keep all sibling fields valid.
- For an array of objects, apply the invalid value to ONE element only
  (e.g. the first element), keeping other elements valid.
- For an array of primitives, additionally test: empty array,
  wrong element type, and duplicate values.
- Naming for nested cases follows the same convention using the leaf path:
  POST /schedule/events | mentions[].type: invalid_enum
  POST /schedule/events | attachmentIds[]: wrong_element_type

### Enum Field Rules

For any field whose spec/testspec defines a fixed set of allowed values:

- Generate cases: value outside the allowed set, wrong case
  (e.g. lowercase when uppercase is required), empty string, null,
  and integer-instead-of-string.
- Read allowed values from the spec/testspec — do not assume a fixed list.
- Use sample_data/enum.json for the generic invalid values, and add the
  "value outside allowed set" dynamically based on the actual enum.

### Optional Parameter Rules

For optional parameters:
- Do not generate "missing required" cases.
- Include one happy path where optional parameters are omitted.
- Still generate invalid value cases when the optional parameter is explicitly included.
- If an optional parameter affects filtering or pagination, include it only when needed for that validation case.

### Naming Convention
```
[METHOD] [endpoint_short] | [param_name]: [data_label]
Examples:
GET /schedule/events | limit: negative_number
POST /schedule/events | title: special_characters
DELETE /schedule/events/{id} | event_id: string_value
```

### Scope

In scope:
- Path parameters
- Query string parameters
- Request body fields (POST)

Out of scope — do NOT generate:
- Auth / login flows
- Multi-user or permission scenarios
- Complex system setup cases
- Business logic beyond data validation

---

## Step 5 — Generate Postman Collection

Output file: `output/collection.json` (Postman Collection v2.1 format)

Each request must include:
1. Pre-request script — inject expected_status using the value determined in Step 4:

  pm.variables.set("expected_status", "400");

The value follows the priority order: testspec → spec constraint → default 4xx.

2. Test script (assertions):
```javascript
// Each request sets its own expected_status variable based on
// priority: testspec → spec constraint → default 4xx
const expected = parseInt(pm.variables.get("expected_status"));

pm.test("Status is " + expected, function () {
    pm.expect(pm.response.code).to.equal(expected);
});
```

Use environment variables throughout:
- {{base_url}} — Garoon site URL
- {{token}} — Auth token

---

## Step 6 — Run Newman

Check if Newman is installed. If not, show setup instructions:

```bash
# Install (one-time)
npm install -g newman newman-reporter-htmlextra
```

Run command:
```bash
newman run output/collection.json \
  --env-var "base_url=SITE_URL" \
  --env-var "token=TOKEN_VALUE" \
  --reporters cli,htmlextra \
  --reporter-htmlextra-export output/report.html
```

---

## Step 7 — Report Summary

After Newman finishes, output a structured summary to the chat:
```
=== API Validation Test Summary ===
Endpoint : GET /api/v1/schedule/events
Run time : 2025-05-28 10:30
Total cases : 38
Passed      : 34
Failed      : 4
Failed cases (source: testspec / spec constraint / sample_data default):

- limit: string_value      -> Expected 400, got 200  [source: sample_data default]
- event_id: float_number   -> Expected 400, got 500  [source: sample_data default]
- start_datetime: spaces   -> Expected 400, got 200  [source: testspec]
- offset: long_number      -> Expected 400, got 200  [source: spec constraint]

Full HTML report: output/report.html
```

---

## Clarifying the Spec/TestSpec

If the spec or testspec is ambiguous, ask before generating:
- What HTTP status code is expected for invalid data? (400? 422?)
- Is this parameter required or optional?
- What is the valid range for this numeric field?
- What datetime format is accepted? (ISO 8601? Unix timestamp?)
- Is 0 a valid value for this ID field?

Check `references/garoon_glossary.md` for Garoon-specific terms if the file exists.
Check `references/garoon_api_conventions.md` for common API conventions, status code policy, datetime format, ID behavior.

---

## Smart Suggestions

Only raise a suggestion if it meets at least one of these criteria:
- Parameter is high-risk: ID field, datetime range, permission-related field
- Spec is missing constraint info (no min/max, no format specified)
- Field type is known to cause bugs (e.g., timezone edge cases for datetime)
- Bug history in `references/garoon_bugs.md` matches this field type
- There is a non-obvious data variant likely to expose a real issue

Do NOT suggest for routine cases or low-risk fields. Keep it brief.

Format:
```
Warning: Suggestion — [param_name]
Reason  : [why this is worth flagging]
Add test : [specific data or case to add]
```

---

## Folder Structure
```
api-param-validator/
├── SKILL.md
├── sample_data/
│   ├── integer.json
│   ├── string.json
│   ├── datetime.json
│   ├── email.json
│   ├── boolean.json
│   ├── enum.json
│   └── id.json
├── references/
│   ├── garoon_glossary.md   (optional)
│   ├── garoon_bugs.md       (optional)
│   ├── garoon_api_conventions.md       (optional)
│   └── garoon_manual_guideline.md       (optional)
└── output/                  (generated at runtime)
    ├── collection.json
    └── report.html
```
---

## Optional: Garoon System Knowledge

If `references/garoon_glossary.md` exists -> read it during Step 1 to understand field semantics.
If `references/garoon_bugs.md` exists -> read it during Step 4 to enhance smart suggestions.
If `references/garoon_api_conventions.md` exists -> read it during Step 1 to Step 4 to understand common API conventions, status code policy, datetime format, ID behavior
If `references/garoon_manual_guideline.md` exists -> read it during Step 1 to Step 4 to understand about Garoon manual guideline/specification

These files are optional. The skill works without them, but suggestions will be more targeted with them.
