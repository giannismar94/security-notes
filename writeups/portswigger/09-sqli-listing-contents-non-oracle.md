# SQL injection — listing database contents (non-Oracle)

- **Platform:** PortSwigger Web Security Academy
- **Lab:** [SQL injection attack, listing the database contents on non-Oracle databases](https://portswigger.net/web-security/sql-injection/examining-the-database/lab-listing-database-contents-non-oracle)
- **Level:** Practitioner
- **Date solved:** 2026-11-12
- **Tools:** Burp Suite (Proxy, Repeater)

## Goal
Find the users table, read its credentials, and log in as `administrator` — without
being told any table or column names.

## What it is
This is the Day 3 UNION attack done "for real". In Day 3 the lab handed me the names
`users`, `username`, `password`. Here they are randomly generated, so I have to
discover them myself through `information_schema`, the built-in catalog that lists
every table and column in the database.

## What I tried
- Counted columns: `' UNION SELECT NULL,NULL-- -` → 200. **2 columns**, both text.
- Listed every table:

      ' UNION SELECT table_name,NULL FROM information_schema.tables-- -

  The response is mostly system tables. I used the Repeater Response **Search** box
  with `users` to jump straight to the real one, a random name like `users_iqibvh`.
  (First attempt I grabbed `user_defined_types` by mistake — that is an
  information_schema system table, not the users table. Lesson: the target has a random
  suffix, the system ones do not.)
- Listed that table's columns:

      ' UNION SELECT column_name,NULL FROM information_schema.columns
        WHERE table_name='users_iqibvh'-- -

  Two columns with random suffixes: `username_betbfx` and `password_...`.

## How I exploited it
- Payload:

      ' UNION SELECT username_betbfx,password_xxxxxx FROM users_iqibvh-- -

- The page listed every username with its password next to it, including
  `administrator`. I copied the administrator password, went to My account, logged in,
  and the lab was solved.

## Impact
Starting from zero knowledge of the schema I read the whole user table and took over
the admin account. The same `information_schema` technique would reveal any other table
(orders, payment data, etc.). Combined with passwords stored in a readable form, this
is full account takeover.

## How to fix it
1. Parameterized queries, so the UNION can never be injected.
2. Store passwords only as salted hashes (bcrypt / Argon2), so a leaked table is not
   directly usable.
3. Least-privilege database account, so the product page cannot read the users table
   at all.

## Key takeaway
`information_schema` makes the attack self-service: list tables → list columns → read
data, with no prior knowledge of the schema. Use the Response search to find the real
table hiding among the system ones.
