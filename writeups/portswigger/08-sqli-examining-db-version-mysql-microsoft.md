# SQL injection — querying the database version (MySQL / Microsoft)

- **Platform:** PortSwigger Web Security Academy
- **Lab:** [SQL injection attack, querying the database type and version on MySQL and Microsoft](https://portswigger.net/web-security/sql-injection/examining-the-database/lab-querying-database-version-mysql-microsoft)
- **Level:** Practitioner
- **Date solved:** 2026-11-12
- **Tools:** Burp Suite (Proxy, Repeater)

## Goal
Make the database reveal its type and version string.

## What it is
The MySQL/Microsoft version of the previous lab. Both of these databases expose the
version through `@@version`. The twist here is the MySQL comment syntax.

## What I tried
- Sent the category filter request to Repeater.
- Tried `' UNION SELECT NULL,NULL,NULL,NULL-- -` and got **500**. On a fresh lab the
  column count is not guaranteed, so I adjusted the number of NULLs until it returned
  200. **2 columns**, both accepting text.
- Early on my `--` comment did nothing and the query kept breaking. In MySQL a comment
  needs a space (or another character) after the dashes, so a bare `--` at the end of
  the URL is ignored. Switching to `-- -` fixed it. `#` (`%23`) also works.

## How I exploited it
- Payload:

      Gifts'+UNION+SELECT+@@version,NULL-- -

- The page returned `8.0.42-0ubuntu0.20.04.1` among the products. The `ubuntu` in the
  string already gives it away as MySQL on Linux — Microsoft SQL Server does not run on
  Ubuntu. Lab solved.

## How to fix it
1. Parameterized queries.
2. Do not echo injected rows or database errors back to the user.
3. Patch the database so a leaked version does not map to known exploits.

## Key takeaway
`@@version` works on both MySQL and Microsoft. The real lesson was the MySQL comment:
use `-- -` (dash, space, dash) or `#`, never a bare `--`.
