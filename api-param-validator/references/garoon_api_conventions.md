# Garoon API Conventions

## Purpose
This file describes how the Garoon internal REST API **behaves** — which input causes
which error, how status codes are assigned, and how to treat an unknown code. It is a
companion to `garoon_error_codes.md`:

- `garoon_error_codes.md` answers **"what a code means"** (code → meaning → status). It is a dictionary.
- `garoon_api_conventions.md` (this file) answers **"how the API behaves"** (input → result, patterns).

**Important — role boundary:** This file is for INTERPRETING results after a run. It is
NOT a source for setting expected values. The expected status/errorCode priority is
always: testspec → spec → default rule. Never assign a status or code from this file
unless the testspec/spec documents it for the case.

---

## 1. HTTP status policy (errorCode → status)

### Default: almost everything is 400
Garoon groups the large majority of client-side errors under **400 Bad Request** — this
includes most "not found" and most permission errors, not just validation errors.
404 / 403 / 401 are the minority.

**Rule:** when you need the HTTP status of an errorCode and neither the testspec nor the
spec states it:
- If the code is in the **exception list** below → use that status.
- Otherwise → assume **400 Bad Request**.
- For the exact meaning of any specific code, look it up in `garoon_error_codes.md`.

This "default 400" is only for LOOKING UP the status of a code that already appeared
(e.g. in a response or documented in testspec/spec). It is NOT used to set the expected
value of a test case — that still follows testspec → spec → sample_data validity.

### Exception list — codes that are NOT 400
Everything not listed here defaults to 400. These are the only common non-400 codes:

**401 Unauthorized (authentication)**
| Code | Meaning |
|------|---------|
| GRN_REST_API_00001 / 00002 / 00003 / 00005 / 00006 / 00007 | Authentication failed |
| FW00008 | Cannot log in (inactive account) |

**403 Forbidden (permission / not available)**
| Code | Meaning |
|------|---------|
| GRN_REST_API_00004 | License expired / request failed |
| GRN_CMMN_00003 / 00004 | Application not available |
| GRN_CMMN_00005 | Cannot perform this operation (lacks privilege) |
| GRN_SCHD_13002 / 13043 / 13044 / 13045 / 13215 / 13369 | Schedule: cannot view/add/change/delete/act on appointment |

**404 Not Found**
| Code | Meaning |
|------|---------|
| GRN_BLLT_16003 / 16005 / 16082 | Bulletin: cannot find topic / draft / survey |
| GRN_CMMN_00105 | Cannot find the specified user |
| GRN_SCHD_13001 / 13203 | Schedule: cannot find appointment / facility |
| GRN_SPACE_00021 | Space: cannot find the specified folder (folderId not found / 0; confirmed 404 on live API) |

**405 / 415 / 429**
| Code | Status | Meaning |
|------|--------|---------|
| GRN_REST_API_00102 | 405 Method Not Allowed | HTTP method not supported |
| GRN_REST_API_00103 | 415 Unsupported Media Type | Missing/invalid Content-Type |
| GRN_REST_API_00104 | 429 Too Many Requests | Concurrency limit exceeded |

Note: "Not found" is NOT reliably 404. Most not-found errors return 400 (e.g.
GRN_SPACE_00001, GRN_BLLT_16004, GRN_SCHD_13205, GRN_MSSG_15003 are all 400). Only the
codes in the 404 table above are confirmed 404. Never assume a not-found situation is
404 — look up the specific code, and default to 400 if unknown.

Note: Status values can change between Garoon versions. testspec/spec always takes priority over this file.

### Not-found by module (summary)
For almost every module, a not-found resource returns 400 with a module-specific code
(e.g. GRN_SPACE_00001, GRN_MSSG_15003, GRN_WRKF_*, GRN_RPRT_* — all 400). Only a few
codes in bulletin, common, and schedule return 404 (already listed in the exception
table above). So: not-found → assume 400 unless the specific code is in the 404 list.

---

## 2. Mapping: invalid input → likely errorCode (HINTS ONLY — testspec/spec takes priority)
These map a kind of invalid input to the code Garoon typically returns. They are HINTS,
not rules: always prefer the errorCode/status documented in the testspec or spec. Do NOT
assign a code from these hints unless the testspec/spec documents it for the case.
For the exact meaning/status of any code below, look it up in `garoon_error_codes.md`.

- Wrong type / malformed parameter value → GRN_REST_API_00202 (400)
- Missing required parameter             → GRN_REST_API_00201 (400)
- Value below minimum length             → GRN_REST_API_00208 (400)
- Value exceeds maximum length           → GRN_REST_API_00209 (400)
- Value below minimum value              → GRN_REST_API_00210 (400)
- Value exceeds maximum value            → GRN_REST_API_00211 (400)
- Invalid enum value                     → GRN_REST_API_00212 (400)
- Invalid date value                     → GRN_REST_API_00220 (400)
- Malformed request body                 → GRN_REST_API_00105 (400)
- Unsupported HTTP method                → GRN_REST_API_00102 (405)
- Nonexistent resource id                → resource-specific code (e.g. GRN_MSSG_15003); status VARIES, look up the code

---

## 3. Handling an unknown errorCode (not in testspec/spec)
When the API returns an errorCode that the testspec/spec does not document:
- If the actual **status matches** the expected status → the case PASSES on status alone.
  Do not try to interpret the unknown code.
- If the status does NOT match → report the unknown code as-is (e.g. "got
  GRN_SCHD_99999, status 400") and classify the case as [REVIEW EXPECTED]. Look up the
  code in `garoon_error_codes.md` if present; if absent, state it is undocumented and
  needs confirmation. NEVER invent a meaning or status for an unknown code.

---

## 4. Restricted-value parameters (enum / boolean / type-limited)
For a parameter that only accepts a fixed set of values, an out-of-range or wrong-type
value is rejected with 400. Typical codes:
- Wrong type for the parameter        → GRN_REST_API_00202 (400)
- Invalid enum value                  → GRN_REST_API_00212 (400)
- Only certain types allowed          → GRN_REST_API_00213 (400)
- Value not in the allowed set        → GRN_REST_API_00226 (400)

The exact code per field still follows testspec → spec. This is an interpretation aid,
not a source for expected values.
