# SQL injection UNION attack — retrieving multiple values in a single column

- **Platform:** PortSwigger Web Security Academy
- **Lab:** [SQL injection UNION attack, retrieving multiple values in a single column](https://portswigger.net/web-security/sql-injection/union-attacks/lab-retrieve-multiple-values-in-single-column)
- **Level:** Practitioner
- **Date solved:** 2026-11-11
- **Tools:** Burp Suite (Proxy, Repeater)

## Goal
Retrieve the usernames and passwords from the `users` table and log in as
`administrator`, when only one column can hold text.

## What it is
Same UNION attack as before, but here only one column accepts text. Username and
password cannot each get their own column, so the two values have to be combined into
a single string.

## What I tried
- Counted the columns and found 2.
- Tested each column with `'a'` and found that only the second column accepts text.
  The first column is not text, so I cannot put username in one and password in the
  other.
- The database is PostgreSQL, which concatenates strings with `||`. I joined the two
  values into the one text column with a `~` separator so I could read them apart.

## How I exploited it
- Payload: `Lifestyle'+UNION+SELECT+NULL,username+||+'~'+||+password+FROM+users--`
- Resulting query:

      SELECT <col1>, <col2> FROM products WHERE category = 'Lifestyle'
      UNION SELECT NULL, username || '~' || password FROM users--'

- Each user appeared as one line in the format `username~password`, for example
  `administrator~<password>`.
- Took the part after the `~` and logged in through My account as `administrator`,
  which solved the lab.

## How to fix it
Parameterized queries, as in every SQLi. Storing passwords as bcrypt/Argon2 hashes
instead of plaintext would mean the leaked values are not usable passwords.

## Key takeaway
When only one column is text, concatenation (`||` in PostgreSQL/Oracle,
`CONCAT()` in MySQL) packs several values into it with a separator. The right syntax
depends on which database is running, so identifying the database comes first.
