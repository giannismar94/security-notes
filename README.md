# security-notes

My notes and writeups while learning web application security, mainly with Burp Suite
and the PortSwigger Web Security Academy.

## Notes

What I learn each day: theory, how things work, commands.

| # | Topic |
|---|---|
| 01 | [HTTP, status codes, URL encoding, Burp Suite](notes/01-http-burp.md) |
| 02 | [Authentication, sessions, cookie flags, CSRF](notes/02-auth-sessions-cookies.md) |
| 03 | [SQL injection UNION attacks](notes/03-sqli-union.md) |
| 04 | [SQL injection — examining the database](notes/04-sqli-examining-database.md) |

## Writeups

How I solved specific labs: what I noticed, what I tried, the payload, and the fix.

| # | Lab | Platform | Level | Topic |
|---|---|---|---|---|
| 01 | [SQL injection in WHERE clause — retrieving hidden data](writeups/portswigger/01-sqli-where-clause-hidden-data.md) | PortSwigger | Apprentice | SQLi |
| 02 | [SQL injection — login bypass](writeups/portswigger/02-sqli-login-bypass.md) | PortSwigger | Apprentice | SQLi |
| 03 | [SQL injection UNION attack — determining the number of columns](writeups/portswigger/03-sqli-union-number-of-columns.md) | PortSwigger | Practitioner | SQLi (UNION) |
| 04 | [SQL injection UNION attack — finding a column containing text](writeups/portswigger/04-sqli-union-text-column.md) | PortSwigger | Practitioner | SQLi (UNION) |
| 05 | [SQL injection UNION attack — retrieving data from other tables](writeups/portswigger/05-sqli-union-other-tables.md) | PortSwigger | Practitioner | SQLi (UNION) |
| 06 | [SQL injection UNION attack — retrieving multiple values in a single column](writeups/portswigger/06-sqli-union-multiple-values-single-column.md) | PortSwigger | Practitioner | SQLi (UNION) |
| 07 | [SQL injection — querying the database version (Oracle)](writeups/portswigger/07-sqli-examining-db-version-oracle.md) | PortSwigger | Practitioner | SQLi (examining DB) |
| 08 | [SQL injection — querying the database version (MySQL/Microsoft)](writeups/portswigger/08-sqli-examining-db-version-mysql-microsoft.md) | PortSwigger | Practitioner | SQLi (examining DB) |
| 09 | [SQL injection — listing database contents (non-Oracle)](writeups/portswigger/09-sqli-listing-contents-non-oracle.md) | PortSwigger | Practitioner | SQLi (examining DB) |
| 10 | [SQL injection — listing database contents (Oracle)](writeups/portswigger/10-sqli-listing-contents-oracle.md) | PortSwigger | Practitioner | SQLi (examining DB) |
