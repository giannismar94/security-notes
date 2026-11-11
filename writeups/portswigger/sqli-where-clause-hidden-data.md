# SQL injection in WHERE clause — retrieving hidden data

- **Platform:** PortSwigger Web Security Academy
- **Lab:** [SQL injection vulnerability in WHERE clause allowing retrieval of hidden data](https://portswigger.net/web-security/sql-injection/lab-retrieve-hidden-data)
- **Level:** Apprentice
- **Date solved:** 2026-11-09
- **Tools:** Burp Suite (Proxy, Repeater)

## What it is
The server concatenates user input directly into the SQL query as a string, so the
user can break out of the string and inject their own SQL.

Vulnerable code:
    const sql = "SELECT * FROM products WHERE category = '" + category + "' AND released = 1";

## How I exploited it
- Entry point: the `category` parameter in GET /filter?category=...
- Detection: sent `category=Tech gifts'` → status 500 (the extra ' broke the syntax).
- Exploitation: `category=Tech+gifts'+OR+1=1--`
- Resulting query:
    SELECT * FROM products WHERE category = 'Tech gifts' OR 1=1--' AND released = 1
  - `'` closes the string
  - `OR 1=1` is always true, so every row matches
  - `--` comments out the `AND released = 1` check
- Result: status 200, the page returned ALL products, including unreleased ones
  (released = 0) that were meant to be hidden.

## How to fix it
Use parameterized queries so data and code stay separate — the database treats the
input as text, never as SQL:

    db.query(
      "SELECT * FROM products WHERE category = ? AND released = 1",
      [category]
    );
