# Blind SQL injection with conditional responses

- **Platform:** PortSwigger Web Security Academy
- **Lab:** [Blind SQL injection with conditional responses](https://portswigger.net/web-security/sql-injection/blind/lab-conditional-responses)
- **Level:** Practitioner
- **Date solved:** 2026-11-13
- **Tools:** Burp Suite (Proxy, Repeater, Intruder)

## Goal
Retrieve the administrator's password and log in, with no query output or error messages
shown — only a "Welcome back" message that appears or not.

## What it is
The `TrackingId` cookie is used in a SQL query. If the query returns a row, the page
shows "Welcome back"; if not, it does not. Nothing from the query is ever printed, so
this is blind SQLi — but that true/false difference is an oracle I can use to extract
data one character at a time.

## What I noticed
The injection point is the **cookie**, not a URL parameter. In a cookie header `;`
separates cookies, so the condition goes right after the TrackingId value with a `'` to
break the string:

    Cookie: TrackingId=xxxx' AND '1'='1 ; session=yyyy

## What I tried
- Confirmed the oracle in Repeater:
  - `' AND '1'='1` -> "Welcome back" present (true)
  - `' AND '1'='2` -> "Welcome back" gone (false)
  - (First attempt I used `;` instead of `'` after the value, which just ended the
    cookie and the injection never ran — the quote is what breaks into the SQL string.)
- Confirmed the data exists:
  - `' AND (SELECT 'a' FROM users LIMIT 1)='a` -> true, so the `users` table exists.
  - `' AND (SELECT username FROM users WHERE username='administrator')='administrator`
    -> true, so the admin user exists.
- Found the password length with `LENGTH()`:
  - `' AND (SELECT username FROM users WHERE username='administrator' AND LENGTH(password)>N)='administrator`
  - Increased N until "Welcome back" disappeared -> length is **20**.

## How I exploited it
- Extracted the password character by character with `SUBSTRING`:

      ' AND (SELECT SUBSTRING(password,<pos>,1) FROM users
             WHERE username='administrator')='<char>

- Ran this in Burp Intruder as a **cluster bomb** attack:
  - payload 1 = position (Numbers 1-20)
  - payload 2 = character (Brute forcer `a-z0-9`, length 1)
  - Grep - Match on `Welcome back` to flag the true requests.
- Sorted the results by the Welcome-back column and read off the character for each
  position, giving the full 20-character password. (In Community Edition Intruder is
  time-throttled, so this takes a while.)
- Logged in through My account as `administrator` with that password. Lab solved.

## Impact
"No output, no error messages" is not a defence. A single observable difference in the
page (one message) let me read the entire admin password one character at a time and
take over the account. The same oracle reads any value in the database.

## How to fix it
1. Parameterized queries, so the cookie can never alter the SQL.
2. Store passwords as salted hashes (bcrypt / Argon2), so a leaked value is not usable.
3. Do not let user-controlled values (like a tracking cookie) reach a SQL query
   unsanitised.

## Key takeaway
Any true/false difference in the response is enough to extract data blindly. Method:
confirm the oracle, confirm what exists, find the length, then extract character by
character. In practice this step is scripted rather than done by hand.
