# SQL injection — filter bypass via XML encoding

- **Platform:** PortSwigger Web Security Academy
- **Lab:** [SQL injection with filter bypass via XML encoding](https://portswigger.net/web-security/sql-injection/lab-sql-injection-with-filter-bypass-via-xml-encoding)
- **Level:** Practitioner
- **Date solved:** 2026-11-13
- **Tools:** Burp Suite (Proxy, Repeater), Hackvertor extension

## Goal
Retrieve the administrator's credentials and log in, past a WAF that blocks SQL
injection keywords.

## What it is
A UNION injection in the stock check feature, but a WAF blocks any request containing
`UNION`/`SELECT`. The stock check sends an XML body, and XML parsers decode character
entities before the value is used — so encoding the keywords as XML entities slips them
past the WAF while the SQL engine still sees valid SQL.

## What I noticed
Clicking **Check stock** sends a POST to `/product/stock` with an XML body:

    <?xml version="1.0" encoding="UTF-8"?>
    <stockCheck><productId>2</productId><storeId>1</storeId></stockCheck>

The result (e.g. `510 units`) is reflected in the response, so `storeId` looked like a
UNION injection point.

## What I tried
- Injected `1 UNION SELECT NULL` into `storeId`. Response:

      HTTP/2 403 Forbidden
      "Attack detected"

  So a WAF is filtering the keywords, not the database rejecting the query.
- Installed **Hackvertor** (Extensions → BApp Store), selected the `UNION SELECT NULL`
  text, and wrapped it with **Encode → hex_entities**:

      <storeId>1 <@hex_entities>UNION SELECT NULL<@/hex_entities></storeId>

- On Send, Hackvertor encoded the keywords as `&#x55;&#x4e;...`. Response was now
  **200 OK**, `510 units` and a `null` row — WAF bypassed, and 1 column confirmed (no
  column-count error).
- Confirmed the column accepts text with `UNION SELECT 'a'` (the `a` came back in the
  response).

## How I exploited it
- Kept the whole SQL inside the Hackvertor tags so the WAF never saw the keywords:

      <storeId>1 <@hex_entities>UNION SELECT username || '~' || password FROM users<@/hex_entities></storeId>

- One column, so username and password were concatenated with `||` and a `~` separator
  (PostgreSQL syntax).
- The response listed every user as `username~password`, including `administrator`.
  Logged in through My account with the admin credentials, and the lab was solved.

## Impact
The WAF gave a false sense of security. A single encoding trick (XML entities) defeated
it completely, and the underlying UNION injection then leaked every user's credentials.
A filter that blocks keywords is not a fix — the vulnerable query is still there.

## How to fix it
1. Parameterized queries, so the injection cannot happen regardless of the input's
   encoding. This is the real fix.
2. Do not rely on a keyword-blocking WAF as the primary defence — it only raises the bar
   and is routinely bypassed with encoding.
3. Least-privilege database account, so the stock feature cannot read the users table.

## Key takeaway
A WAF inspects the request before the app decodes it. If a decoding step (XML entities)
sits between the filter and the SQL engine, the keywords can be hidden in it. Encoding
bypasses the filter; it never fixes the injection.
