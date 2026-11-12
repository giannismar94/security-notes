# SQL injection — querying the database version (Oracle)

- **Platform:** PortSwigger Web Security Academy
- **Lab:** [SQL injection attack, querying the database type and version on Oracle](https://portswigger.net/web-security/sql-injection/examining-the-database/lab-querying-database-version-oracle)
- **Level:** Practitioner
- **Date solved:** 2026-11-12
- **Tools:** Burp Suite (Proxy, Repeater)

## Goal
Make the database reveal its type and version string through the product category
filter.

## What it is
Same UNION injection as Day 3, but now the goal is not another table — it is the
database's own version. Knowing the exact version tells an attacker which known
vulnerabilities (CVEs) might apply to that build.

## What I tried
- Sent `GET /filter?category=Lifestyle` to Repeater.
- Started counting columns the usual way: `Lifestyle' UNION SELECT NULL-- -`.
- Got **500**. Added more NULLs, still 500. That was the clue that this was Oracle.
- On Oracle every SELECT needs a FROM, so a bare `SELECT NULL` is invalid. Added
  `FROM dual` (Oracle's built-in dummy table):

      ' UNION SELECT NULL, NULL FROM dual-- -   → 200

  So **2 columns**, and testing with `'a'` showed both accept text.

## How I exploited it
- Oracle does not have `@@version`. The version lives in the `v$version` view, in a
  column called `banner`.
- Payload:

      Lifestyle'+UNION+SELECT+BANNER,NULL+FROM+v$version-- -

- Resulting query:

      SELECT ... FROM products WHERE category = 'Lifestyle'
      UNION SELECT BANNER, NULL FROM v$version-- -

- The page listed several banner rows among the products, including
  `Oracle Database 11g Express Edition Release 11.2.0.2.0 - 64bit Production`.
  No `FROM dual` here because `v$version` is already a real table. Lab solved.

## How to fix it
1. Parameterized queries so no UNION can be injected in the first place.
2. Do not return raw database errors or injected rows to the user.
3. Keep the database patched — leaking the exact version is only useful to an attacker
   if that version has known holes.

## Key takeaway
The 500 errors were the tell: on Oracle every `SELECT` needs a `FROM`, so column
counting uses `FROM dual`, and the version comes from `v$version` (banner), not
`@@version`.
