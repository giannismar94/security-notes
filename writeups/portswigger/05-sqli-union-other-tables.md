# SQL injection UNION attack — retrieving data from other tables

- **Platform:** PortSwigger Web Security Academy
- **Lab:** [SQL injection UNION attack, retrieving data from other tables](https://portswigger.net/web-security/sql-injection/union-attacks/lab-retrieve-data-from-other-tables)
- **Level:** Practitioner
- **Date solved:** 2026-11-11
- **Tools:** Burp Suite (Proxy, Repeater)

## Goal
Retrieve all usernames and passwords from the `users` table and log in as
`administrator`.

## What it is
Once the column count and the text columns are known, the NULLs can be replaced with
real columns from another table. The page then shows rows from `users` as if they
were products.

The lab gives the table and column names (`users`, `username`, `password`). In a real
test I would have to find them first, for example through `information_schema`.

## What I tried
- New lab, so a fresh request from HTTP history: `GET /filter?category=Gifts`.
- Counted the columns again: 2 columns.
- Tested each one with `'a'`. Both columns accept text, so username and password can
  each go in their own column. There was no need to combine them.

## How I exploited it
- Payload: `Gifts'+UNION+SELECT+username,password+FROM+users--`
- Resulting query:

      SELECT <col1>, <col2> FROM products WHERE category = 'Gifts'
      UNION SELECT username, password FROM users--'

- The page listed every user with their password next to it, including
  `administrator`. The passwords were stored in **plaintext**.
- Logged in through My account as `administrator` with that password, and the lab
  was marked solved.

## Impact
This is much worse than the login bypass:

- **Data exfiltration:** it leaks the credentials of every user, and the same method
  reads any other table in the database (customers, emails, orders).
- **Persistent access:** with real passwords I can still log in after the SQLi is
  fixed, until every password is changed.
- **Beyond this site:** people reuse passwords, so leaked plaintext passwords can be
  used on other services (credential stuffing).
- **Hard to detect:** a login with a correct password looks legitimate in the logs.

Two weaknesses chained together (SQLi + plaintext passwords) gave admin access in a
few minutes.

## How to fix it
1. Parameterized queries, so the UNION can never be injected.
2. Store passwords only as hashes with a slow algorithm (bcrypt or Argon2) and a
   unique salt per user. Even if the table leaks, the passwords are not readable.
3. Give the application's database user the least privileges it needs, so the product
   page cannot read the `users` table at all.

## Key takeaway
Step 3 of a UNION attack: once the column count and text columns are known, any table
can be read. The real impact comes from what the data allows next.
