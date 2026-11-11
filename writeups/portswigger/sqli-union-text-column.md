# SQL injection UNION attack — finding a column containing text

- **Platform:** PortSwigger Web Security Academy
- **Lab:** [SQL injection UNION attack, finding a column containing text](https://portswigger.net/web-security/sql-injection/union-attacks/lab-find-column-containing-text)
- **Level:** Practitioner
- **Date solved:** 2026-11-11
- **Tools:** Burp Suite (Proxy, Repeater)

## Goal
Make the database return a specific random string given by the lab (different for
each lab instance), which proves which column can hold text.

## What it is
To pull usernames or passwords with UNION, they must go into a column whose data type
is text. Putting a string into a column that holds a number (an id or a price) makes
the query fail.

## What I tried
- My first request in Repeater got no response ("Stream failed to close correctly").
  It was the request from the previous lab, whose instance had already expired.
  Lab instances are temporary, so I opened the new lab and sent a fresh request from
  HTTP history: `GET /filter?category=Pets`.
- It is a new lab, so I counted the columns again instead of assuming. It returned
  3 columns, the same as the previous lab.
- Replaced one NULL at a time with a string, keeping the rest as NULL so I would know
  which column caused an error:

  | Payload | Status |
  |---|---|
  | `Pets'+UNION+SELECT+'a',NULL,NULL--` | 500 |
  | `Pets'+UNION+SELECT+NULL,'a',NULL--` | 200 |
  | `Pets'+UNION+SELECT+NULL,NULL,'a'--` | 500 |

  Only column 2 accepts text. Columns 1 and 3 are probably numeric.

## How I exploited it
- Put the lab's string in column 2: `Pets'+UNION+SELECT+NULL,'<lab string>',NULL--`
- Sent it from the browser. The string appeared at the bottom of the product list,
  in the position of a product name, and the lab was marked solved.
- Since it shows up where the product name is, column 2 is most likely the name.

## How to fix it
Parameterized queries, the same as the previous labs. Knowing which column is text
only matters because the input reaches the SQL in the first place.

## Key takeaway
Change one thing at a time. Testing each column separately tells me exactly which
one accepts text. This is step 2 of a UNION attack, and it is what decides where the
stolen data can go.
