# Day 2 — Authentication, Sessions & Cookies

## How login & sessions work
- HTTP is stateless: the server forgets who you are after each request.
- Session flow:
  1. Client sends POST /login with credentials.
  2. Server verifies them against the database.
  3. Server creates a random session ID and stores it on its side.
  4. Server returns it: Set-Cookie: session=...
  5. The browser automatically sends that cookie on every later request.
  6. Server looks up the session ID and knows who you are.
- The session ID is the key to your account. Anyone who has it is you,
  so it must be random, long, and must not leak.

## Session vs JWT
- Session cookie: the "truth" lives on the SERVER (DB/memory). The cookie is
  just a random key. Easy to log out (delete the session server-side).
- JWT: the "truth" lives INSIDE the token, signed by the server. The payload
  is readable by anyone (base64, not encrypted). Harder to revoke before expiry.
- Common JWT mistake: trusting role=admin in the payload. If the signature isn't
  verified properly, an attacker can change it. The signature is what matters.

## Cookie flags
- HttpOnly: JavaScript cannot read the cookie (document.cookie won't return it),
  but the browser still sends it with requests. Protects the session from XSS theft.
  Note: it does NOT stop XSS itself — the attacker's script can still make requests
  as you while the page is open; it just can't read the cookie value.
- Secure: the cookie is sent only over HTTPS, never plain HTTP. Protects against
  interception on an insecure connection (e.g. public Wi-Fi).
- SameSite: controls whether the cookie is sent on requests that start from
  another site. Main defence against CSRF.
    - Strict: never sent from another site.
    - Lax: sent only when following a link (GET), not on cross-site POST. Modern default.
    - None: always sent; requires Secure.

## CSRF (Cross-Site Request Forgery)
- A malicious site makes YOUR browser send a request to a site you're logged into
  (e.g. your bank). The browser auto-attaches your session cookie, so the request
  runs as you. The attacker never sees or steals the cookie — they abuse the fact
  that the browser sends it automatically.
- Limited by SameSite (and by CSRF tokens).

## Site vs Origin (don't confuse these)
- Origin = scheme + full host + port (login.example.com != example.com).
- Site = scheme + the registrable domain (login.example.com == example.com).
- SameSite looks at the SITE, not the origin.

## Key takeaway
Same root cause as yesterday: user input read as code. The fix keeps data and code
separate (parameterized queries), and sensitive checks like passwords never belong
in the query string at all.
