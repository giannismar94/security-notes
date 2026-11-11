# SQL injection in WHERE clause — retrieving hidden data

- **Platform:** PortSwigger Web Security Academy
- **Lab:** [SQL injection vulnerability in WHERE clause allowing retrieval of hidden data](https://portswigger.net/web-security/sql-injection/lab-retrieve-hidden-data)
- **Level:** Apprentice
- **Date solved:** 2026-11-09
- **Tools:** Burp Suite (Proxy, Repeater)

## What it is
The server concatenates user input directly into the SQL query as a string, so the
user can break out of the string and inject their own SQL.

## What I tried
- Opened the lab through Burp's browser and clicked a product category. In HTTP history
  I saw the filter is a GET request: `/filter?category=Tech+gifts`.
- Sent it to Repeater and changed the category to another value. The products changed,
  so the parameter is used to query the database.
- Added a single quote: `category=Tech gifts'`. The server returned 500, which told me
  my input reaches the SQL query unescaped.

## How I exploited it
- Entry point: the `category` parameter in `GET /filter?category=...`
- Payload: `category=Tech+gifts'+OR+1=1--`
- Resulting query:

      SELECT * FROM products WHERE category = 'Tech gifts' OR 1=1--' AND released = 1

  - `'` closes the string
  - `OR 1=1` is always true, so every row matches
  - `--` comments out the `AND released = 1` check
- Result: status 200 and the page returned all products, including unreleased ones
  (`released = 0`) that were meant to be hidden.

Likely vulnerable code (my guess from the behaviour, I can't see the server code):

    const sql = "SELECT * FROM products WHERE category = '" + category + "' AND released = 1";

## How to fix it
Use parameterized queries so data and code stay separate. The database treats the
input as text, never as SQL:

```js
db.query(
  "SELECT * FROM products WHERE category = ? AND released = 1",
  [category]
);
```

## Key takeaway
SQL injection means user data gets read as code. The fix always keeps data and code apart.
