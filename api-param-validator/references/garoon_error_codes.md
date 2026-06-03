# Specification of Common Error (Garoon)

## Overview
This document describes common errors when using the internal API.

---

## Error Codes by HTTP Status

---

## 400 - Bad Request

| Error Code | Error Message | Occur Condition | Note |
|------------|--------------|----------------|------|
| GRN_REST_API_00101 | Invalid input. | The specified URI path is not found. | |
| GRN_REST_API_00201 | Invalid parameter. | Missing required parameter | |
| GRN_REST_API_00202 | Invalid parameter. | `{param_name}` is not valid. | |
| GRN_REST_API_00105 | Invalid input. | The request body is invalid. | |
| GRN_REST_API_00208 | Invalid input. | Input less than the minimum number of characters. | |
| GRN_REST_API_00209 | Invalid input. | Input exceeds the maximum number of characters. | |
| GRN_REST_API_00204 | Invalid parameter | The specified value of `{param_name}` is more than `{param_name}` | |
| GRN_REST_API_00210 | Invalid input. | `{param_name}` is less than the minimum value. | |
| GRN_REST_API_00211 | Invalid input. | `{param_name}` exceeds the maximum value. | |
| GRN_REST_API_00212 | Invalid parameter. | Only allowed values for `{param_name}`: `[&&enum_options&&]` | |
| GRN_REST_API_00220 | Invalid parameter. | The specified date of `{param_name}` is invalid | |

Note: The "Occur Condition" descriptions above are typical/representative. In practice some codes (e.g. GRN_REST_API_00101) may be returned in broader cases than the single condition listed. Match by the error code itself, not by assuming the condition is exhaustive.

---

## 401 - Unauthorized

| Error Code | Error Message | Occur Condition | Note |
|------------|--------------|----------------|------|
| GRN_REST_API_00001 | Authentication failed. | Invalid `X-Cybozu-Authorization` header | |
| GRN_REST_API_00002 | Authentication failed. | Missing `X-Requested-With` header | |
| GRN_REST_API_00003 | Authentication failed. | No authentication information found | |
| GRN_REST_API_00005 | Authentication failed. | Invalid access token format | |
| SLASH_OA08 | The access token is not found | Access token not found | |
| FW00007 | Cannot log in. | Incorrect login name or password | Confirm credentials and retry |
| FW00008 | Cannot log in. | User account is inactive | Request not from Garoon |
| GRN_REST_API_00007 | Authentication failed. | Operation not allowed with this access token | |

---

## 403 - Forbidden

| Error Code | Error Message | Occur Condition | Note |
|------------|--------------|----------------|------|
| GRN_REST_API_00004 | The request failed. | License has expired | |
| GRN_CMMN_00003 | The application is not available. | Application is deactivated | |
| GRN_CMMN_00004 | The application is not available. | User is not allowed to use application | |
| GRN_CMMN_00022 | You cannot perform this operation | Requires admin privileges (system/application/basic) | |
| GRN_CMMN_00005 | You cannot perform this operation | User lacks required privileges | |

---

## 404 - Not Found

| Error Code | Error Message | Occur Condition | Note |
|------------|--------------|----------------|------|
| GRN_CMMN_00105 | Cannot find the specified user | User does not exist or incorrect user specified | |

---

## 405 - Method Not Allowed

| Error Code | Error Message | Occur Condition | Note |
|------------|--------------|----------------|------|
| GRN_REST_API_00102 | Invalid input. | HTTP method is not supported | Requested method is not allowed |

---

## 415 - Unsupported Media Type

| Error Code | Error Message | Occur Condition | Note |
|------------|--------------|----------------|------|
| GRN_REST_API_00103 | Invalid header. | Missing or invalid `Content-Type` | |

---

## 429 - Too Many Requests

| Error Code | Error Message | Occur Condition | Note |
|------------|--------------|----------------|------|
| GRN_REST_API_00104 | The request failed. | API request exceeds concurrency limit | |

---

## 502 - Bad Gateway

| Error Code | Error Message | Occur Condition | Note |
|------------|--------------|----------------|------|
| SLASH_LO02 | Invalid login name or password. | Inactive user sends API request | |

---

## 520 - Unknown Error

| Error Code | Error Message | Occur Condition | Note |
|------------|--------------|----------------|------|
| GRN_CMMN_00106 | Cannot find the specified organization | Organization does not exist or incorrect | |
| GRN_CMMN_00107 | Cannot find specified role | Role does not exist or incorrect | |
| GRN_MOBILE_00001 | Cannot access this URL | Mobile view is prohibited | |

Note: This table is for reference only. The actual occur conditions for 520 may be broader than listed here; treat these as representative examples, not an exhaustive or strict mapping.

---

## Mapping hints for test cases (HINTS ONLY — testspec/spec takes priority)
These are typical mappings to help look up a likely code. They are NOT rules:
always prefer the errorCode/status documented in the testspec or spec. Do NOT
assign a code from these hints unless the testspec/spec documents it for the case.

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
- Nonexistent resource id                → resource-specific code (e.g. GRN_MSSG_15003), often 404

## Notes for AI Usage
- Error structure includes:
  - Error Code
  - Error Message
  - Occur Condition
- `{param_name}` and `[&&enum_options&&]` are dynamic placeholders
- When matching an errorCode from the testspec/spec against this table, match by the
  CODE only (e.g. GRN_REST_API_00202). Ignore the dynamic parts of the message
  ({param_name}, [&&enum_options&&]) — the real response fills them with actual values.
- How the skill uses this file:
  1. Given an errorCode documented in the testspec/spec, look up its HTTP status
     (e.g. GRN_REST_API_00202 → 400) — useful when the testspec states a code but not a status.
  2. Use the mapping hints only as a fallback guide, never to override testspec/spec.
  3. This file is a LOOKUP reference, not a source to pick codes from freely.