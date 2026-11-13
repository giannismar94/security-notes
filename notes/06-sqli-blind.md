# Day 6 — Blind SQL injection (conditional responses)

Up to now the page showed me the result of my injection (products, version strings,
usernames). In **blind** SQL injection the application runs my query but returns
neither the result nor any database error. It looks like nothing happens — but the
query still executes.

## The idea: an oracle
Even with no visible output, the injection is exploitable if the page **behaves
differently** depending on whether a condition is true or false. That difference is an
**oracle**: I ask the database yes/no questions and read the answer from the page's
behaviour. With a yes/no channel I can extract any data one character at a time.

In this lab the oracle is the **"Welcome back"** message:
- The app takes the `TrackingId` cookie and uses it in a SQL query.
- If the query returns a row, "Welcome back" is shown.
- If it returns nothing, "Welcome back" is gone.

The injection point here is the **cookie**, not a URL parameter.

## Step 1 — confirm the oracle
Inject a condition after the TrackingId value (note: `'` to break the string, no `;`,
that separates cookies):

    TrackingId=xxxx' AND '1'='1    -> Welcome back present (true)
    TrackingId=xxxx' AND '1'='2    -> Welcome back gone    (false)

Two different behaviours for true vs false = I have an oracle.

## Step 2 — confirm the table and user exist
Use a subquery that is only true if the thing exists:

    ' AND (SELECT 'a' FROM users LIMIT 1)='a                                  -- users table exists
    ' AND (SELECT username FROM users WHERE username='administrator')='administrator  -- admin exists

## Step 3 — find the password length
Ask "is the length greater than N?" and narrow it down (binary search is quickest):

    ' AND (SELECT username FROM users WHERE username='administrator'
           AND LENGTH(password)>1)='administrator

Increase the number until "Welcome back" disappears. The last true value + 1 is the
length. (PortSwigger passwords are 20 characters.)

## Step 4 — extract the password character by character
`SUBSTRING(password, position, 1)` returns one character. Ask "is character N equal to
'x'?" for every position and every candidate character:

    ' AND (SELECT SUBSTRING(password,1,1) FROM users
           WHERE username='administrator')='a

When "Welcome back" appears, that character is correct for that position.

**With Burp Intruder:** Cluster bomb attack, two payload positions:
- position 1 = the character index -> Numbers 1 to 20
- position 2 = the guessed character -> Brute forcer `a-z0-9`, length 1

Add a **Grep - Match** for `Welcome back` so the results table flags the true requests.
Run it, sort by the Welcome-back column, and read the correct character for each
position in order.

> Community Edition note: Intruder is heavily time-throttled, so 720 requests take a
> long time. The technique is identical in Pro; in the real world this kind of blind
> extraction is scripted (a short Python loop) or done with `sqlmap`, not clicked by
> hand.

## Syntax that changes per database
- `SUBSTRING()` is `SUBSTR()` on some databases (e.g. Oracle).
- `LENGTH()` vs `LEN()` (Microsoft).
- As always, identify the database first, then use the right functions.

## Why blind SQLi matters
It proves that "no visible output and no error messages" is **not** a defence. Any
difference in behaviour — a message, a status code, even response time (time-based
blind) — is enough to read the whole database one bit at a time.

## Key takeaway
Blind SQLi turns any true/false difference in the page into a data-exfiltration
channel. Method: confirm the oracle, confirm what exists, find the length, then extract
character by character.
