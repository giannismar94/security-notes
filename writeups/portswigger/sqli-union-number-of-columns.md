# SQL injection UNION attack — determining the number of columns

- **Platform:** PortSwigger Web Security Academy
- **Lab:** [SQL injection UNION attack, determining the number of columns returned by the query](https://portswigger.net/web-security/sql-injection/union-attacks/lab-determine-number-of-columns)
- **Level:** Practitioner
- **Date solved:** 2026-11-11
- **Tools:** Burp Suite (Proxy, Repeater)

## Goal
Find how many columns the product query returns, and prove it with a UNION that
returns an extra row of NULL values.

## What it is
A UNION attack adds a second SELECT to the original query so its results show up on
the page. It only works if both SELECTs return the same number of columns, so the
first step is always to find that number.

## What I tried
- At first I opened the login page and started there, but the lab description says
  the injection is in the product category filter. Lesson: read the lab description
  before looking for the entry point.
- Went back to Home, clicked the Accessories category and sent
  `GET /filter?category=Accessories` from HTTP history to Repeater.
- Used `ORDER BY` and increased the number until it broke:

  | Payload | Status |
  |---|---|
  | `Accessories'+ORDER+BY+1--` | 200 |
  | `Accessories'+ORDER+BY+2--` | 200 |
  | `Accessories'+ORDER+BY+3--` | 200 |
  | `Accessories'+ORDER+BY+4--` | 500 |

  Ordering by a 4th column fails, so the query returns 3 columns.
- `ORDER BY` alone did not solve the lab, because the lab expects a UNION that
  returns an extra row.

## How I exploited it
- Confirmed the number with a UNION of NULLs:
  `Accessories'+UNION+SELECT+NULL,NULL,NULL--` → 200.
- Sent the same payload from the browser. The page showed an extra empty row at the
  bottom of the product list (NULL means no value, so nothing is printed), and the lab
  was marked solved.
- I used NULL because it fits any data type, so the only thing being tested is the
  number of columns.

## How to fix it
The root cause is the same as every SQLi: user input is concatenated into the query.
Use parameterized queries:

```js
db.query("SELECT name, description, price FROM products WHERE category = ?", [category]);
```

Also avoid returning a 500 with a different page for SQL errors. The difference between
200 and 500 is exactly what let me count the columns.

## Key takeaway
Errors are information. `ORDER BY n` and `UNION SELECT NULL,...` both use the error to
find the column count, which is step 1 of every UNION attack.
