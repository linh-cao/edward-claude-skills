---
name: api-param-validator
description: 'Automates API parameter data validation testing for Garoon APIs using Postman + Newman. Use this skill whenever the user wants to: generate Postman test cases for API parameters, validate path/query/body parameters with invalid data, run automated API validation with Newman, create test collections for GET/POST/DELETE/PUT/PATCH endpoints, or test data types like wrong type, special characters, out-of-range values, invalid formats. Trigger on phrases like "test API params", "validate parameters", "generate Postman collection", "auto test API", "run API validation", "test data validation for endpoint", or any mention of testing parameters with various invalid data types. Always use this skill when the user provides an API spec and asks to test it — even if they do not explicitly say "skill".'
---

# API Parameter Validator

Automates data validation testing for Garoon API parameters. Given a spec/testspec + endpoint info, generates a complete Postman collection covering standard invalid data scenarios, runs it via Newman, and produces a pass/fail report.

---

## Safety and Runtime Rules
- Never print, log, or save real credentials (X-Cybozu-Authorization / base64 of username:password) in generated files or reports.
- Always use `{{cybozu_auth}}` (or another environment variable) for authentication.
- Do not hard-code credentials into `collection.json`.
- For POST/PUT/PATCH/DELETE requests, assume the request may modify data.
- If DELETE requires an existing resource ID, ask the user for a disposable test resource ID.
- If POST creates data, prefer using clearly identifiable test data.
- Only run against test environments. For Cloud Neco: the domain MUST contain `cybozu-dev`; if it does not, treat the site as non-test (production) and REFUSE to run; warn the user.
- Onpremise: the site is IP-based, assumed to be a test instance, so run normally without confirmation.
- Use pnpm exclusively for installing Node packages. NEVER invoke npm or yarn under any circumstances.
- Never add or override the package registry (no `--registry` flag); always use the registry already configured on the machine.
- If a pnpm install fails (e.g. 401 Unauthorized, missing global bin dir, network error), STOP and report the exact error to the user. Do NOT try an alternative package manager, an alternative registry, or any workaround. Let the user resolve it manually.

---

## How to Invoke This Skill
Prompt format:
```
Test API params for [METHOD] [endpoint] on [site].
Spec: [paste content here, or attach file/screenshot]
Testspec: [attach file if available — optional]
Auth: username:password (e.g. user1:1) — the skill base64-encodes it for the X-Cybozu-Authorization header
```

Example:
```
Test API params for GET /api/v1/schedule/events on https://example.garoon.com
Auth: user1:1
Spec:
event_id (path, integer, required): ID of the event
limit (query, integer, optional): max 100, min 1
start_datetime (query, datetime, optional): ISO 8601 format
```

---

## No Fabrication Rule (applies to the entire skill)
Never invent, guess, or assume information that is not grounded in the spec, the
testspec, or the sample_data files. Specifically:

- Do NOT invent expected status codes or errorCodes. Use only values documented in
  the testspec or spec. If neither documents one, fall back to the default rule —
  never guess a code.
- Do NOT add test cases, values, or parameters that are not in sample_data or the
  spec (e.g. do not add xss/sql_injection cases unless they exist in sample_data).
- Do NOT change an expected result to match what the API actually returned (see
  Expected Result Integrity in Step 4).
- Do NOT drop a parameter just because one source omits it. If a parameter appears
  in the spec or the testspec, include it in testing.
- If information is missing, ask the user or flag it for review — never fill the
  gap with an assumption.

### errorCode / status priority
Resolution order: testspec → spec → (look up the HTTP status for an already-documented
code via `references/garoon_error_codes.md`, if it exists) → default rule.
- `garoon_error_codes.md` is a LOOKUP for the STATUS of a code that the testspec/spec
  already states. It is NOT a source to choose WHICH code applies.
- Never assign an errorCode taken from `garoon_error_codes.md` (or anywhere) unless
  the testspec/spec documents that code for the case.

---

## Step 0 — Check Required Inputs
Before proceeding, verify these are present. If anything is missing, ask the user before continuing:
| Input | Required | Notes |
|---|---|---|
| HTTP method | YES | Common: GET / POST / DELETE / PUT / PATCH. Methods with a request body (POST/PUT/PATCH) follow the body-field rules in Step 4 |
| Endpoint path | YES | e.g. `/api/v1/schedule/events/{event_id}` |
| Garoon site URL | YES | e.g. `https://example.garoon.com` |
| API spec | YES | See "Providing Spec" below |
| Auth credentials | YES | Provide as `username:password` (e.g. user1:1). The skill base64-encodes the whole string for the `X-Cybozu-Authorization` header. Never log the raw credentials or the encoded value. Note: base64 is reversible, treat as a secret |
| Request body schema | POST/PUT/PATCH | Required if the method has a request body |
| Testspec file | Optional | Contains test cases + expected results; if provided, will be combined with spec to determine accurate expected results |

### Environments and URL Construction
Two supported environments:
- Cloud Neco: base_url is the app root, e.g. https://ed20961.cybozu-dev.com/g
- Onpremise:  base_url is the app root, e.g. http://10.224.152.xxx/grn

Normalization (REQUIRED): when reading the site URL, strip any trailing slash from
base_url before use, so ".../g/" and ".../g" behave the same. Also drop `index.csp?`
(that is the UI entry, not the API root). Then join the endpoint path with exactly
ONE leading slash.

The full request URL = base_url (no trailing slash) + "/" + endpoint path.
This guarantees no double slash (produce .../g/api/..., never .../g//api/...).

Example:
- Pasted site : https://ed20961.cybozu-dev.com/g/index.csp?pid=1
- Normalized  : https://ed20961.cybozu-dev.com/g   (drop index.csp? and trailing slash)
- Endpoint : api/internal/message/messages/{messageId}/comments/{commentId}
- Full URL Cloud Neco   : https://ed20961.cybozu-dev.com/g/api/internal/message/messages/{messageId}/comments/{commentId}
- Full URL Onpremise: http://10.224.152.xxx/grn/api/internal/message/messages/{messageId}/comments/{commentId}

See `references/garoon_api_conventions.md` for full URL/convention details if it exists.

### Providing Spec (when direct link sharing is restricted)
Ask the user to choose one of these methods:
1. Paste content directly — Copy text from sharedoc into the prompt (preferred)
2. Attach a file — .md, .txt, or .pdf
3. Screenshot — Attach a screenshot; Claude will read it

The prompt format shown above is a guideline, not a strict template. The spec may be
typed inline, pasted, attached as a file (.md / .txt / .pdf), or given as a screenshot
— accept whichever form the user provides. The spec may also be incomplete; if so, see
"Clarifying the Spec" before generating.

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
- Expected error status and/or errorCode for invalid input, IF the spec documents them (e.g. invalid event_id -> 404 / GRN_SCHD_13001)
- Distinguish `date` (date only, e.g. 2025-05-28) from `datetime` (date + time, e.g. 2025-05-28T10:30:00Z) — they have opposite valid/invalid expectations (see Step 4)

### Nested Body Parsing (POST/PUT/PATCH)
For POST/PUT/PATCH request bodies, recursively flatten ALL nested fields into leaf paths,
regardless of field names. Use this notation:
```
- Object field      → <parent>.<child>
- Array of objects  → <array>[].<child>
- Array of values   → <array>[]
```
Apply this to whatever field names appear in the actual spec/testspec/body —
the names below are only examples of the notation, not fixed fields.

Example (body with fields "mentions", "attachmentIds"):
```
  mentions[].id        (string)
  mentions[].type      (enum)
  attachmentIds[]      (array of string)
  data                 (string)
```
Record each leaf field with its type, exactly like top-level params.
If the spec does not describe a nested field's type, infer it from the
example body or testspec; if still unclear, ask the user.

Add each flattened leaf field as an entry in the `body_fields` array of the
parameter map (same structure as path_params/query_params: name, type,
required, constraints). The leaf path is the field "name".

### Detecting enum fields
Specs rarely label a field `type: enum`. Infer enum from prose that lists a fixed
set of allowed values, for example:
- "The value is USER / GROUP / STATIC_ROLE"
- "Must be one of: ACTIVE, INACTIVE"
- "Allowed values: ..." or "... otherwise an error is returned"

When such a list is detected, treat the field as enum: record its type as `enum`,
extract the allowed values, and apply the Enum Field Rules in Step 4.
See `references/garoon_api_conventions.md` for Garoon-specific enum phrasings if it exists.

If `references/garoon_glossary.md` exists -> read it now to understand Garoon-specific terms in the spec.

---

## Step 2 — Parse Testspec (if provided)
If the user attaches a testspec file, read and extract the following for each test case:
- Parameter name
- Input value / data label
- Expected HTTP status code
- Expected errorCode (ONLY if the testspec explicitly provides it, e.g. GRN_SCHD_13001)

Do NOT extract or assert on the error `message` text — it is human-readable and may
be localized, so it is unreliable for assertions.

Then cross-reference with the parameter map from Step 1:

### Record expected results (merged later in Step 4)
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

### Looking up status from errorCode
If `references/garoon_error_codes.md` exists, use it to map an errorCode found in the
testspec/spec to its HTTP status — useful when the testspec states an errorCode but
not the status (e.g. "Error GRN_REST_API_00202 is returned" -> status 400).
Match by the code only (ignore the dynamic {param_name} placeholder in the message).
This is a LOOKUP for codes already documented in the testspec/spec — NOT a license to
pick or guess a code that the testspec/spec does not state.

---

## Step 3 — Load Sample Data
Read from `sample_data/` based on each parameter's type:
| Type | File |
|---|---|
| integer / number | `sample_data/integer.json` |
| string | `sample_data/string.json` |
| datetime (date + time) | `sample_data/datetime.json` |
| date (date only, no time) | `sample_data/datetime.json` — same file; date-only fields treat time/timezone-bearing values as invalid and bare-date values as valid |
| boolean | `sample_data/boolean.json` |
| id (identifier) | Numeric id -> `sample_data/integer.json`; string id -> `sample_data/string.json`. See "ID Field Rules" in Step 4 for id-specific cases |
| enum | `sample_data/string.json` (enum is a string with a fixed value set). Generic invalid values come from string.json; enum-specific cases (outside set, wrong case, partial match) are generated dynamically per Step 4 Enum Field Rules |
| array | Use the element type's file (e.g. array of string → string.json, array of integer → integer.json) for element values. Array-structural cases (empty array, duplicate, wrong element type) come from Step 4 Nested Body Field Rules |
| body (raw payload) | `sample_data/body.json` — whole-body cases for POST/PUT/PATCH. Sent VERBATIM as the request body, NOT parsed or re-serialized |

Each file contains labeled invalid test values. Additionally, for each parameter with `min`/`max` constraints, append out-of-range values dynamically.

---

## Step 4 — Generate Test Cases
### Build a Valid Baseline Request
Rules:
- Required path parameters must use existing valid values.
- Required query/body parameters must use valid values.
- Optional parameters should be omitted in the minimal happy path unless the spec requires them.
- For valid baseline values, source priority is testspec → spec. Use a testspec happy-path value first if available, otherwise a spec example.
- If no valid value can be inferred, ask the user.

Default valid value strategy:
- integer with min/max: use a value inside the range
- string: use `"test"` unless pattern/enum requires another value
- enum: use the first valid enum value
- datetime: use a full ISO 8601 value (e.g. 2025-05-28T10:30:00Z)
- date: use a bare date in the required format (e.g. 2025-05-28)
- boolean: use `true`
- id: ask the user for an existing valid ID unless the spec provides one (no dedicated id file; uses integer/string)

### Structure
For each invalid value per parameter, create one test case:
- All other parameters use their valid default values
- Only the target parameter has the invalid value
- Expected result priority order: (1) testspec if provided → (2) spec constraints → (3) default 4xx from sample_data

Also include:
- 1 happy path case: all params valid, all required params present → expect 2xx

If the total generated cases exceed ~300, inform the user of the count and ask whether
to proceed or narrow scope. If the user chooses to narrow, drop cases in this priority
order (drop lowest first), keeping the highest-signal cases:
1. KEEP: type-mismatch and required/missing-field cases (highest signal)
2. KEEP: boundary cases (min/max, empty, zero, out-of-range)
3. KEEP: format cases (wrong format, invalid timezone, special characters)
4. DROP FIRST: redundant variants of the same kind on one parameter
   (e.g. keep ~1 representative "wrong string" case instead of several)
Always keep the happy-path case.

### Expected Result Integrity (do not fit to actual responses)
NEVER change a case's expected result just to make it pass against what the API
actually returned. The whole point is to detect mismatches.

- If the API returns 2xx for an invalid input (i.e. it does NOT validate), KEEP the
  expected error. Let the case FAIL, and report it as a possible missing-validation bug.
- Only adjust an expected result when the new value reflects CORRECT, documented
  behavior — e.g. 405 because an empty path segment routes to a different endpoint.
  Never adjust merely because "that is what the API returned".
- When unsure whether the actual behavior is correct, keep the original expected
  (FAIL) and raise a Smart Suggestion to confirm with the spec author. Do not silently
  convert it to a pass.

### Nested Body Field Rules
These rules apply to any nested field, regardless of its name:
- Test each leaf field independently; keep all sibling fields valid.
- For an array of objects, apply the invalid value to ONE element only
  (e.g. the first element), keeping other elements valid.
- For an array of primitives, additionally test: empty array,
  wrong element type, and duplicate values.
- Naming for nested cases follows the same convention using the leaf path:
  POST /schedule/events | mentions[].id: string_value
  POST /schedule/events | mentions[].type: invalid_enum
  POST /schedule/events | attachmentIds[]: wrong_element_type

### Whole-Body Validation Rules (POST/PUT/PATCH only)
In addition to field-level cases, generate body-level cases that send an invalid
request body as a whole (raw payload), independent of individual fields.
Load these from `sample_data/body.json`.

IMPORTANT: each body.json value is the COMPLETE raw request body. Send it verbatim
as the request body — do NOT parse it or re-serialize it (some values are
intentionally invalid JSON; parsing would either error out or "fix" them and
defeat the test).

Covered cases include: malformed JSON, special-characters-only body, wrong root
type (array/string/null instead of object), empty body, and empty object `{}`.

Most should return 4xx (malformed / invalid schema). Expected result follows the
usual priority: testspec → spec → default 4xx.
Naming: POST /schedule/comments | body: malformed_json

For GET/DELETE (no request body), these whole-body cases do not apply and are not
generated — do not list them as skipped.

### Enum Field Rules
For any field whose spec/testspec defines a fixed set of allowed values:
- Generate cases: value outside the allowed set, wrong case
  (e.g. lowercase when uppercase is required), empty string, null,
  and integer-instead-of-string.
- Read allowed values from the spec/testspec — do not assume a fixed list.
- Use sample_data/string.json for the generic invalid values, and add the
  "value outside allowed set" (and wrong-case / partial-match) dynamically based on the actual enum.

### Date vs Datetime Rules
`date` and `datetime` share sample_data/datetime.json but have OPPOSITE expectations:

- datetime field (date + time): a full ISO 8601 value is valid. Bare-date values
  like `2025-05-28` (label: date_only) are INVALID (missing time).
- date field (date only): a bare date like `2025-05-28` is valid. Values carrying
  time or timezone (e.g. ...T10:30:00Z, no_timezone, invalid_tz_offset) are INVALID.

For date-only fields, prefer the date-form range probes (far_future_date,
far_past_date) over the datetime ones.
Expected result still follows the usual priority: testspec → spec → default 4xx.

### ID Field Rules
IDs do not have their own sample_data file. Reuse integer.json (numeric id) or
string.json (string id) for type/format cases. Additionally, for id fields:

- nonexistent_id: a well-formed id that does not exist (e.g. a very large number
  like 999999999) -> expect 404 Not Found, not a validation 4xx.
- zero and negative values are INVALID for id fields -> expect an error.
  Treat them as invalid regardless of the generic integer note ("depends on spec").

Expected result follows the usual priority: testspec → spec → default
(4xx for malformed/zero/negative, 404 for a well-formed but nonexistent id).

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
- Request body fields (POST/PUT/PATCH)
- Whole request body (malformed / wrong-type raw payloads) for POST/PUT/PATCH

Out of scope — do NOT generate:
- Auth / login flows
- Multi-user or permission scenarios
- Complex system setup cases
- Business logic beyond data validation

---

## Step 5 — Generate Postman Collection
Output file: `./output/collection.json` (Postman Collection v2.1 format) — written relative to the current working
directory (where the skill was invoked), NOT inside the skill install directory
(.claude/skills/...). Create the ./output/ folder in the CWD if it does not exist.

Each request must include:
1. These headers:
- X-Cybozu-Authorization: {{cybozu_auth}}   (base64 of username:password, from env var)
- Content-Type: application/json

2. URL format (Postman Collection v2.1) — REQUIRED structure:
Build each request URL as a structured object. A bare {"raw": "..."} alone makes
Newman report "request url is empty" (it cannot resolve {{base_url}}). Use this exact shape:
```json
"url": {
  "raw": "{{base_url}}<path>[?<query>]",
  "host": ["{{base_url}}"],
  "path": ["segment1", "segment2", "..."],
  "query": [ { "key": "relationId", "value": "abc" } ] // example only — replace with the actual param key/value
}
```
Rules:
- `host` MUST be exactly ["{{base_url}}"] (base_url already contains scheme + domain
  + the /g/ or /grn/ prefix; keep it as a single host element — this is what Newman resolves).
- `path` MUST list each path segment separately, with the test value substituted into
  the path parameter segment (e.g. ".../messages/5", path ends with "5").
- Include `query` ONLY when the case targets a query parameter. Each query param is
  an object {"key": "...", "value": "..."}.
- `raw` MUST stay consistent with host/path/query (same value the structured parts encode).
- Omit `query` entirely for path-parameter or happy-path cases that have no query string.

3. Pre-request script — inject the expected values determined in Step 4:
```javascript
// Set expected status for THIS request
pm.variables.set("expected_status", "400");

// Optional errorCode (priority: testspec → spec). MUST reset to avoid leaking
// a value set by a previous request, since pm.variables persist across requests.
if (/* this case has an expected errorCode */ false) {
    pm.variables.set("expected_error_code", "GRN_SCHD_13001");
} else {
    pm.variables.unset("expected_error_code");
}
```

IMPORTANT: pm.variables persist across requests in a Newman run. Any optional
variable (e.g. expected_error_code) MUST be explicitly unset when not used for the
current case, otherwise it leaks from a previous request and causes false assertions.

- `expected_status` follows the priority order: testspec → spec constraint → default 4xx.
- `expected_error_code` is optional. Source priority: testspec → spec. Set it only when either provides one for this case; otherwise omit it entirely.

4. Test script (assertions):
```javascript
// 1. Always assert HTTP status (baseline)
const expected = parseInt(pm.variables.get("expected_status"));
pm.test("Status is " + expected, function () {
    pm.expect(pm.response.code).to.equal(expected);
});

// 2. Optionally assert errorCode — ONLY when expected_error_code is set.
//    If not set, this check is skipped and status alone decides pass/fail.
const expectedErrorCode = pm.variables.get("expected_error_code");
if (expectedErrorCode) {
    pm.test("errorCode is " + expectedErrorCode, function () {
        let body;
        try { body = pm.response.json(); } catch (e) { body = null; }
        pm.expect(body && body.error && body.error.errorCode).to.equal(expectedErrorCode);
    });
}
// Note: the error `message` text is intentionally NOT asserted.
```

5. For whole-body cases (values from body.json): set the RAW request body to the
   value verbatim — do not parse or re-serialize it. Keep header
   Content-Type: application/json (sending an invalid body with a JSON content-type
   is intentional; the API should reject it). This applies only to POST/PUT/PATCH.

Use environment variables throughout:
- {{base_url}} — Garoon site URL
- {{cybozu_auth}} — base64 of username:password (used in X-Cybozu-Authorization header)

### Special Value Handling
Some sample_data files may include sentinel values that are markers, NOT literal
values to send. They apply to ANY data type (boolean, integer, string, etc.) —
handle them the same way everywhere. Never send the literal sentinel string as the value.

`__NO_VALUE__` — field is present but carries no value:
- Query/path parameter -> render as `field=` (key present, nothing after `=`)
- Request body field    -> omit the field from the JSON body

`__OMIT_FIELD__` — field key is entirely absent (tests a missing parameter):
- Query/path parameter -> do not include the key at all (no `field`, no `field=`)
- Request body field    -> omit the field from the JSON body
- For a missing REQUIRED parameter, expect an error (4xx); for a missing OPTIONAL
  parameter, this matches the happy-path behavior in Step 4.

`__DUPLICATE__` — send the parameter twice, both using its valid value (from the
Default valid value strategy in Step 4):
- Query/path parameter -> render as `field=<valid>&field=<valid>`
- Request body field    -> JSON objects cannot have duplicate keys; skip this case for body
- Expected result is API-dependent (some accept and pick one value, some reject).
  Do not assume 4xx — use testspec/spec if available, otherwise flag the result for review.

These rules are shared across all sample_data files; do not special-case per type.

Rendering sentinels into the structured v2.1 URL (query parameters):
- __NO_VALUE__   -> include the key with empty value: {"key": "relationId", "value": ""}
                    (and reflect as `relationId=` in raw)
- __OMIT_FIELD__ -> remove that key from the url.query array entirely
                    (and remove it from raw)
- __DUPLICATE__  -> add the key twice in url.query, both with the valid value:
                    [{"key":"relationId","value":"<valid>"},{"key":"relationId","value":"<valid>"}]
Keep `raw` consistent with the resulting query array in every case.

---

## Step 6 — Run Newman
### Prepare Auth
Accept the credentials in whatever form the user provides, for example:
- `user1:1`
- `user1/1`
- `login name: user1, pw: 1`
- `username: user1, password: 1`

Identify the username and password from any of these forms, then normalize to the
string `username:password` (e.g. `user1:1`) and base64-encode it verbatim.
The result becomes the `cybozu_auth` value used for the X-Cybozu-Authorization header.
Passwords are typically simple (e.g. `1`, `cybozu`, `cybozu123`).
Do NOT print the raw credentials or the encoded value in any output.

Example: login name user1 / pw 1 -> "user1:1" -> base64 -> dTEwOjE... (cybozu_auth)

Check if Newman is installed by running: 
```bash
newman --version
```

- If it returns a version -> skip install, go straight to "Run command".
- If it returns "command not found" -> install using pnpm ONLY:
```bash
# Install (one-time) — pnpm only, no npm/yarn, no registry override
pnpm add -g newman newman-reporter-htmlextra
```
If this pnpm command fails for any reason (401 Unauthorized, ERR_PNPM_NO_GLOBAL_BIN_DIR,
network/registry error, etc.), STOP immediately and report the exact error to the user.
Do NOT fall back to npm, yarn, or a different registry. The user will resolve the
environment/registry issue manually and re-run the skill.

Run command:
Use the normalized base_url (no trailing slash, per Step 0) in --env-var "base_url=...".
This keeps the joined URL clean (no double slash).
Run newman from the current working directory. Use RELATIVE paths (./output/...) for the
collection and the report so they land in the CWD, NOT absolute paths into the skill folder.
```bash
newman run ./output/collection.json \
  --env-var "base_url=SITE_URL" \
  --env-var "cybozu_auth=BASE64_VALUE" \
  --reporters cli,htmlextra \
  --reporter-htmlextra-export ./output/report.html
```

### Run Failure Handling
Before reporting results, check for infrastructure failures (not validation bugs):
- If all/most cases return 401/403 -> auth is likely wrong. Stop and ask the user to re-check cybozu_auth. Do NOT report these as validation failures.
- If requests cannot connect (timeout, DNS, connection refused) -> report a connection error and ask the user to check base_url / network / VPN. Do NOT report as test results.
- If all/most cases return 5xx -> the server may be down or misconfigured. This is an
  infrastructure problem, not a per-case validation bug. Report it as a server/environment
  issue and do NOT list each case as a separate bug. (Note: a 5xx on only a FEW specific
  invalid inputs is different — that is a per-case [POTENTIAL BUG: got 5xx], handled in Step 7.)
- If Newman exits with an error before completing -> surface the actual error instead of a partial pass/fail summary.
- If every case "errored" with the same structural error (e.g. "request url is empty", malformed collection) -> this is a collection-generation bug, NOT API behavior.
  Fix the collection (see the URL object format in Step 5) and re-run. Do NOT report these as API/validation failures.

Note: Use `references/garoon_error_codes.md` (if it exists) to confirm an infrastructure/auth
error by its errorCode, rather than guessing from the status alone — e.g.
GRN_REST_API_00001 / 00003 (401, authentication) or GRN_CMMN_00003 / 00004 (403, app
unavailable) indicate an environment/auth problem, not a parameter-validation result.

---

## Step 7 — Report Summary
After Newman finishes, output a structured summary to the chat:
```
=== API Validation Test Summary ===
Endpoint : GET /api/v1/schedule/events
Run time : 2025-05-28 10:30
Total cases : 40
Passed  : 33
Failed  : 5
Skipped : 2
Skipped cases:
- isActive: duplicate_param  -> skipped (duplicate not applicable in JSON body)

Failed cases (source: testspec / spec constraint / sample_data default):

- limit: string_value      -> Expected 400, got 200  [source: sample_data default]
- event_id: float_number   -> Expected 400, got 500  [source: sample_data default] [POTENTIAL BUG: got 500]
- start_datetime: spaces   -> Expected 400, got 200  [source: testspec]
- offset: long_number      -> Expected 400, got 200  [source: spec constraint]
- event_id: string_value   -> status OK (400) but errorCode mismatch: expected GRN_SCHD_13001, got GRN_CMMN_00001  [source: testspec]

Full HTML report: ./output/report.html
Collection      : ./output/collection.json
```
Output formatting rules:
- List each failed case EXACTLY ONCE. Do not repeat the failed-case list or any case
  within it. Build the list by iterating the set of distinct failed cases a single time.
- Print the summary one time only — do not re-emit the whole report.
- Each failed case is one self-contained line; never concatenate two cases onto the
  same line or split one case across overlapping fragments.
- Report output file paths relative to the current working directory (./output/...),
  each path on its own line and listed exactly once. Do not print absolute paths into
  the skill folder, and do not repeat any path line.

Count integrity:
- The "Failed : N" number MUST equal the number of distinct failed cases listed below it.
- A case whose HTTP status is correct but whose errorCode differs IS counted as a failed
  case (see the errorCode rule below) — count it the same way in both the number and the list.
- Before printing, verify: Total = Passed + Failed + Skipped, and Failed = number of lines
  in the failed-case list.

Do not report a case as passed by having lowered its expected result to match the
response. A 2xx on invalid input must appear as a FAILED case (or an explicit
"missing validation" finding), never as a silent pass.

When an expected_error_code was set and the actual errorCode differs (even if the
HTTP status matched), report it as a failed case with both expected and actual
errorCode. A correct status but wrong errorCode means the API rejected the input
for a different reason than expected.

When the actual status is 5xx (especially 500), flag it separately with
`[POTENTIAL BUG: got 5xx]` in the report, not just as a normal mismatch.
A 5xx on invalid input usually means the API did not handle validation
gracefully (unhandled exception) — a higher-severity signal worth highlighting.

### Classifying failed cases
Group failed cases into two categories so the reader can separate real bugs from
expectations that may be too strict. Classify only — never change the expected to make
a case pass.

[POTENTIAL BUG] — the API behavior contradicts the spec/testspec:
- 5xx on invalid input (unhandled exception)
- Missing validation: API returns 2xx for an input the spec/testspec says is invalid
- API rejects an input the spec/testspec says is valid

[REVIEW EXPECTED] — the case failed but the API behavior may be acceptable; the expected
value (often a sample_data default, not from testspec/spec) may be too strict:
- Expected came from "sample_data default" and the API's actual behavior looks reasonable
- errorCode differs but the status is correct and the alternative code is plausible

Format each failed case with its category tag, e.g.:
- attachmentIds: object_value ({}) -> Expected 400, got 201  [POTENTIAL BUG: missing validation] [source: testspec row 194]
- attachmentIds[]: nonexistent_id  -> Expected 400, got 201  [REVIEW EXPECTED: sample_data default may be too strict]
Keep the [source: ...] tag on every line regardless of category.

---

## Clarifying the Spec/TestSpec
This section enforces the No Fabrication Rule: when required information is ambiguous
or missing, ASK before generating instead of guessing. If a required value cannot be
determined from spec + testspec + references, STOP and ask before generating the
collection — do not proceed with an assumed value. (Conflict detection in Step 2 handles
spec-vs-testspec contradictions; this section handles missing/ambiguous information.)

Common things to clarify:
- What HTTP status code is expected for invalid data? (400? 422?)
- Is this parameter required or optional?
- Is this optional parameter still validated for invalid values, or ignored if malformed?
- What is the valid range for this numeric field?
- Is this field a date (date only) or datetime (date + time)?
- What datetime format is accepted? (ISO 8601? Unix timestamp?)
- For an integer field, is a numeric string (e.g. "1") accepted, or only a real integer?

In an automated/batch run where asking is not possible, use the documented default and
FLAG the assumption in the report for review — never assume silently.

Check `references/garoon_glossary.md` for Garoon-specific terms if the file exists.
Check `references/garoon_api_conventions.md` for common API conventions, status code policy, datetime format, ID behavior.
Check `references/garoon_error_codes.md` to look up the HTTP status of a documented errorCode.

---

## Smart Suggestions
Raise a suggestion only when it adds real value. Two kinds:

### A. Pre-run (suggest additional tests based on the spec)
- High-risk parameter: ID field, datetime/date range, permission-related field
- Missing constraint in spec (no min/max, no format, no maxLength on free-text string)
- Bug-prone field type: datetime with no documented timezone rule; date vs datetime
  ambiguity (unclear if time component is required)
- Enum detected from prose but no documented behavior for invalid values
- Bug history in `references/garoon_bugs.md` matches this field type
- A non-obvious data variant likely to expose a real issue

### B. Post-run (flag based on actual API behavior — higher signal)
- [POTENTIAL BUG] API returned 2xx for an input the spec/testspec says is invalid
  (missing validation — e.g. object accepted where array expected)
- [POTENTIAL BUG] API returned 5xx on invalid input (unhandled exception)
- [POTENTIAL BUG] API rejected an input the spec/testspec says is valid
- errorCode mismatch: status correct but errorCode differs from testspec/spec
- A parameter the spec documents that the API appears not to validate at all
  (all invalid values returned 2xx)

Do NOT suggest for routine/low-risk cases. Keep each suggestion brief.

Format:
Warning: Suggestion — [param_name]
Reason  : [why this is worth flagging]
Action  : [specific test to add, OR what to confirm with the spec author]

---

## Folder Structure
The skill install directory holds ONLY static files (never write output here):
```
api-param-validator/
├── SKILL.md
├── sample_data/
│   ├── body.json
│   ├── boolean.json
│   ├── datetime.json
│   ├── integer.json
│   └── string.json
└── references/
    ├── garoon_api_conventions.md             (optional)
    ├── garoon_bugs.md                        (optional)
    ├── garoon_error_codes.md                 (optional)
    ├── garoon_glossary.md                    (optional)
    └── garoon_manual_guideline.md            (optional)
```
Runtime output is created in the CURRENT WORKING DIRECTORY (the folder where the user
invoked the skill), NOT inside the skill install directory:
```
<current-working-directory>/output/
    ├── collection.json
    └── report.html
```

---

## Optional: Garoon System Knowledge
If `references/garoon_glossary.md` exists -> read it during Step 1 to understand field semantics.
If `references/garoon_bugs.md` exists -> read it during Step 4 to enhance smart suggestions.
If `references/garoon_api_conventions.md` exists -> read it during Step 1 to Step 4 to understand common API conventions, status code policy, datetime format, ID behavior.
If `references/garoon_error_codes.md` exists -> use it during Step 2 and Step 4 to look up the HTTP status of an errorCode already documented in the testspec/spec. It is a lookup reference only, not a source to pick codes from.
If `references/garoon_manual_guideline.md` exists -> read it during Step 1 to Step 4 to understand about Garoon manual guideline/specification.

These files are optional. The skill works without them, but suggestions will be more targeted with them.
