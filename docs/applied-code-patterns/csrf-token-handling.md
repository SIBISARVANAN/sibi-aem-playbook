# CSRF Token Handling

## What CSRF is

A third-party site forges a POST request to AEM on behalf of a logged-in user — the browser automatically attaches the session cookie, so AEM cannot distinguish it from a real request.

## How the token fixes it

AEM generates a random token tied to the user's session and puts it in a response body (not a cookie). Cross-origin scripts cannot read response bodies (browser Same-Origin Policy). Without the token, the forged POST is rejected with HTTP 403 by `CSRFFilter` before your servlet ever runs.

## Correct pattern — use the OOTB endpoint, no custom GET needed

```javascript
// Fetch once on page load from AEM's built-in endpoint
fetch("/libs/granite/csrf/token.json", { credentials: "same-origin" })
    .then(res => res.json())
    .then(data => { csrfToken = data.token; });

// Attach to every POST
fetch("/bin/myservlet", {
    method: "POST",
    credentials: "same-origin",
    headers: { "CSRF-Token": csrfToken }
});
```

```html
<!--/* Plain HTML form — render token server-side */-->
<sly data-sly-use.csrf="/libs/granite/csrf/token.json"/>
<form method="POST">
    <input type="hidden" name=":cq_csrf_token" value="${csrf.token}"/>
</form>
```

**Do NOT** write a custom GET endpoint to issue tokens — `/libs/granite/csrf/token.json` already exists.
**Do NOT** call `csrfTokenManager.isValidToken()` manually in your servlet — `CSRFFilter` already did it before `doPost()` was reached.

## Three-pillar security model

- HttpOnly session cookie — JS on evil.com can't steal it
- Same-Origin Policy — evil.com can't read response bodies from your domain
- Token tied to session — a stolen token string is useless without the matching session cookie

## Common interview Q&A

**Q: What HTTP status does `CSRFFilter` return on a bad token?**
HTTP 403 — before your servlet's `doPost()` is ever invoked.

**Q: Does CSRF apply to server-to-server calls?**
No — CSRF is a browser attack. Server-to-server calls don't carry session cookies automatically; exclude those paths from the filter or use service-user sessions, which bypass the HTTP filter stack.

**Q: Why can't evil.com just call `/libs/granite/csrf/token.json` and read the token itself?**
It can *send* the request but cannot *read* the response — Same-Origin Policy blocks cross-origin response reads. The token lives in the response body, not in a cookie, so SOP locks it away from the attacker's script.
