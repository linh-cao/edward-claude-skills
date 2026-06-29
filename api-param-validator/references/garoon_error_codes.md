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
| GRN_REST_API_00203 | Invalid parameter. | The specified value of `{param2}` is more than or equal to the value of `{param1}`. | |
| GRN_REST_API_00205 | Invalid parameter. | `{param2}` is mandatory when `{param1}` is specified. | |
| GRN_REST_API_00206 | Invalid parameter. | The object specified in `{param}` does not exist. | |
| GRN_REST_API_00207 | Invalid parameter. | There is no permission to the object specified in `{param}`. | |
| GRN_REST_API_00213 | Invalid parameter. | Only `{type_options}` type(s) can be specified in `{param}`. | |
| GRN_REST_API_00214 | Invalid parameter. | The following value(s) are required for `{param}`: `[&key_options&]` | |
| GRN_REST_API_00216 | Invalid parameter. | One of the following values must be specified: `[{param1}, {param2}, {param3}]` | |
| GRN_REST_API_00221 | Invalid parameter. | The specified time zone of `{param}` is invalid. | |
| GRN_REST_API_00223 | Invalid parameter. | Both isStartOnly and isAllDay cannot be specified as 'true' at the same time. | |
| GRN_REST_API_00224 | Invalid parameter. | Either 'end' must be specified, or isStartOnly must be specified as 'true'. | |
| GRN_REST_API_00225 | Invalid input. | `{param}` contains invalid characters for a file name. | |
| GRN_REST_API_00226 | Invalid input. | The specified value of `{param}` is not allowed. | |
| GRN_REST_API_00228 | The specified parameter is not valid. | The following parameters cannot be specified at the same time: `[{param}]` | |
| GRN_REST_API_00229 | The specified parameter is not valid. | Specify one of the following parameters: `[{param}]` | |

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
| GRN_REST_API_00006 | Authentication failed. | The access token expired. | |

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
  1. Given an errorCode documented in the testspec/spec, look up its HTTP status and
     meaning (e.g. GRN_REST_API_00202 → 400). Useful when the testspec states a code
     but not a status.
  2. This file is a LOOKUP dictionary (code → meaning → status), NOT a source to pick
     codes from freely.
  3. For API BEHAVIOR patterns (which input causes which code, not-found vs 404,
     boolean/enum handling, how to treat an unknown code), see
     `garoon_api_conventions.md`. This file answers "what a code means"; the
     conventions file answers "how the API behaves".
