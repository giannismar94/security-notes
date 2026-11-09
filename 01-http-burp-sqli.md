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

## SQL Injection — WHERE clause (PortSwigger lab, APPRENTICE)

### What it is
The server concatenates user input directly into the SQL query as a string, so the
user can break out of the string and inject their own SQL.

Vulnerable code:
    const sql = "SELECT * FROM products WHERE category = '" + category + "' AND released = 1";

### How I exploited it
- Entry point: the `category` parameter in GET /filter?category=...
- Detection: sent `category=Tech gifts'` → status 500 (the extra ' broke the syntax).
- Exploitation: `category=Tech+gifts'+OR+1=1--`
- Resulting query:
    SELECT * FROM products WHERE category = 'Tech gifts' OR 1=1--' AND released = 1
  - `'` closes the string
  - `OR 1=1` is always true, so every row matches
  - `--` comments out the `AND released = 1` check
- Result: status 200, the page returned ALL products, including unreleased ones
  (released = 0) that were meant to be hidden.

### How to fix it
Use parameterized queries so data and code stay separate — the database treats the
input as text, never as SQL:

    db.query(
      "SELECT * FROM products WHERE category = ? AND released = 1",
      [category]
    );

## Key takeaway
SQL injection = making your data be read as code. The fix always keeps data and
code apart (parameterized queries / prepared statements).