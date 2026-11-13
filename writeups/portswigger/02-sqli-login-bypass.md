# SQL injection — login bypass

- **Platform:** PortSwigger Web Security Academy
- **Lab:** [SQL injection vulnerability allowing login bypass](https://portswigger.net/web-security/sql-injection/lab-login-bypass)
- **Level:** Apprentice
- **Date solved:** 2026-11-10
- **Tools:** Burp Suite (Proxy, Repeater)

## What it is
The login query concatenates the username directly into SQL, so I can break out
of the string and comment out the password check.

## What I tried
- Logged in with a random username and password through Burp's browser and found
  the login request in HTTP history: `POST /login` with `username` and `password`
  in the body.
- Thought about what the server probably runs: a query that checks the username
  and the password together. If I can comment out the password part, only the
  username is checked.
- Considered `' OR 1=1--`. It would make the condition always true, but it logs you
  in as the first user in the table, not as a user I choose. The goal was to log in
  as `administrator`, so I used the username directly instead.

## How I exploited it
- Username field: `administrator'--`
- Password field: anything (not checked)
- Resulting query:

      SELECT * FROM users WHERE username = 'administrator'--' AND password = '...'

  - `'` closes the username string
  - `--` comments out the whole `AND password = ...` check
- Result: logged in as administrator without knowing the password.

Likely vulnerable code (my guess from the behaviour, I can't see the server code):

    const sql = "SELECT * FROM users WHERE username = '" + user + "' AND password = '" + pass + "'";

## How to fix it
Two layers, both needed:

1. Parameterized queries, so input is never treated as SQL:

   ```js
   db.query("SELECT * FROM users WHERE username = ?", [user]);
   ```

2. Never compare the password inside SQL. Store only a hash (bcrypt) and compare
   it in code:

   ```js
   const match = await bcrypt.compare(pass, row.password_hash);
   ```

## Key takeaway
Same root cause as the first lab: user input read as code. The fix keeps data and
code separate, and sensitive checks like passwords never belong in the query string.
