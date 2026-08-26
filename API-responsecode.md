# HTTP Response Status Codes

| Code | Meaning | Common Situation |
|---|---|---|
| **200 OK** | Request successful | Page/API worked |
| **201 Created** | Resource successfully created | User registration, creating a record |
| **204 No Content** | Successful, but no response body | Delete/update succeeded |
| **301 Moved Permanently** | Permanent redirect | HTTP → HTTPS |
| **302 Found** | Temporary redirect | Redirecting to another URL |
| **400 Bad Request** | Invalid request | Wrong/missing data |
| **401 Unauthorized** | Authentication required/failed | Invalid login/token |
| **403 Forbidden** | Server refuses access | Permission denied |
| **404 Not Found** | Resource/route doesn't exist | Wrong URL |
| **405 Method Not Allowed** | HTTP method isn't allowed | Sending `GET` to a `POST` route |
| **408 Request Timeout** | Client request timed out | Request took too long |
| **409 Conflict** | Request conflicts with existing data | Duplicate user/email |
| **429 Too Many Requests** | Too many requests | Rate limiting |
| **500 Internal Server Error** | Backend/server error | Node.js application error/crash |
| **501 Not Implemented** | Functionality not supported | Unsupported feature |
| **502 Bad Gateway** | Proxy received invalid response | NGINX → Node.js problem |
| **503 Service Unavailable** | Service temporarily unavailable | Backend stopped/unavailable |
| **504 Gateway Timeout** | Proxy didn't receive backend response | NGINX → backend timeout |
