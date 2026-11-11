# Day 1 — HTTP, Burp Suite & first SQL injection

## HTTP basics
- The client sends a request, the server replies with a response. Both are plain text.
- Request structure: request line (method + path + version), headers, blank line, body.
- GET: parameters go in the URL after `?`. POST: parameters go in the body.
- Golden rule: everything in the request (params, headers, cookies, body) is
  controlled by the user. The server must not trust any of it without validation.

## Status codes
- 200 OK, 302 redirect (check the Location header), 404 not found.
- 401 Unauthorized: the server doesn't know who you are (not logged in).
- 403 Forbidden: the server knows who you are, but you don't have permission.
- 500 Internal Server Error: the code crashed. In a pentest this is a signal that
  my input reaches the database/logic and broke something — worth investigating.

## URL encoding
- Special characters are encoded as %XX: ' → %27, space → + or %20, : → %3A, / → %2F.
- The server decodes the value before using it. Encoding is transport, NOT protection:
  %27 reaches the database as '.

## Key takeaway
SQL injection = making your data be read as code. The fix always keeps data and
code apart (parameterized queries / prepared statements).
