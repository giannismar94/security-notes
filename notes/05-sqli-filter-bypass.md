# Day 5 — SQL injection: bypassing filters (WAF)

Sometimes the injection point is clearly there, but a filter (a WAF) sits in front of
the application and blocks requests that contain obvious attack keywords like `UNION`
or `SELECT`. The injection still works — I just have to get the payload past the filter.

## The setup I hit
The injection was not in a URL parameter this time. It was in the **stock check**
feature, which sends an **XML** body in a POST request:

    POST /product/stock
    Content-Type: application/xml

    <?xml version="1.0" encoding="UTF-8"?>
    <stockCheck>
      <productId>2</productId>
      <storeId>1</storeId>
    </stockCheck>

The `storeId` is injectable, and the results come back in the response, so it is a
UNION attack. But any payload containing `UNION SELECT` returned:

    HTTP/2 403 Forbidden
    "Attack detected"

That is the WAF. It scans the raw request text for bad keywords and blocks them before
the request reaches the application.

## The idea: encode so the WAF can't read it, but the app can
A WAF and the application often parse the request differently. Here the body is XML, and
**the XML parser decodes character entities before the value is used**. So:

- XML entity `&#x55;` decodes to `U`, `&#x4e;` to `N`, and so on.
- If I write `UNION` as `&#x55;&#x4e;&#x49;&#x4f;&#x4e;`, the WAF sees that encoded blob
  and finds no `UNION` keyword, so it lets the request through.
- The XML parser then decodes it back to `UNION` before handing it to the SQL
  interpreter, so the injection still runs.

This is the general trick: find a decoding step that happens **after** the filter but
**before** the SQL engine, and hide the keywords in that encoding.

## Doing it with Hackvertor
Encoding every character by hand is tedious. The **Hackvertor** Burp extension
(Extensions → BApp Store → Hackvertor) does it automatically:

1. Select the SQL part of the payload in the request.
2. Right-click → Extensions → Hackvertor → Encode → **hex_entities**.
3. It wraps the selection in tags:

       <storeId>1 <@hex_entities>UNION SELECT ...<@/hex_entities></storeId>

4. On Send, Hackvertor replaces the tagged text with its `&#x..;` encoded form.

**Important:** every keyword the WAF blocks must stay *inside* the tags. If any part of
`UNION SELECT` is left outside, the WAF sees it and blocks again.

## Same UNION method as before
Once past the WAF, it is the normal UNION attack:
- `UNION SELECT NULL` → one column (no column-count error, so the count is 1).
- `UNION SELECT 'a'` → confirms the column accepts text.
- The stock query returns a single column, so both username and password go into it,
  concatenated: `username || '~' || password` (PostgreSQL / Oracle syntax).

## Key takeaway
A filter that blocks `UNION`/`SELECT` is not the end. If there is an encoding or
decoding layer between the filter and the SQL engine (XML entities here), hide the
keywords in it. The injection logic does not change — only the packaging does.
