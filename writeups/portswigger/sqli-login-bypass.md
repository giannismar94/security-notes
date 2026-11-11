# SQL injection — login bypass

- **Platform:** PortSwigger Web Security Academy
- **Lab:** [SQL injection vulnerability allowing login bypass](https://portswigger.net/web-security/sql-injection/lab-login-bypass)
- **Level:** Apprentice
- **Date solved:** 2026-11-10
- **Tools:** Burp Suite (Proxy, Repeater)

## What it is
The login query concatenates the username directly into SQL, so I can break out
of the string and comment out the password check.

Vulnerable code:
    const sql = "SELECT * FROM users WHERE username = '" + user + "' AND password = '" + pass + "'";

## How I exploited it
- Username field: administrator'--
- Password field: anything (not checked)
- Resulting query:
    SELECT * FROM users WHERE username = 'administrator'--' AND password = '...'
  - ' closes the username string
  - -- comments out the whole AND password = ... check
- Result: logged in as administrator without knowing the password.
- Note: `OR 1=1` would log me in as the FIRST user in the table; `administrator'--`
  lets me pick exactly which user to be.

## How to fix it
Two layers, both needed:
1. Parameterized queries so input is never treated as SQL:
       db.query("SELECT * FROM users WHERE username = ?", [user]);
2. Never compare the password inside SQL. Store only a hash (bcrypt) and compare
   in code:
       const match = await bcrypt.compare(pass, row.password_hash);
