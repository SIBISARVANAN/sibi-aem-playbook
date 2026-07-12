# XSS Protection — XSSAPI

## Core rule

Never trust any value that came from a JCR property or a request parameter when writing it into HTML, JS, or a URL — always pass it through the correct `XSSAPI` method for the context it's being written into.

| Context | Method |
|---|---|
| HTML body text | `xssAPI.encodeForHTML(value)` |
| HTML attribute | `xssAPI.encodeForHTMLAttr(value)` |
| Inside a `<script>` block | `xssAPI.encodeForJSString(value)` |
| URL / href | `xssAPI.getValidHref(value)` |

## Why HTML-escaping is wrong inside a `<script>` block

`encodeForHTML()` escapes `<` and `>` but does nothing to neutralize a `"` that breaks out of a JS string literal — a value like `x"; alert(1); //` still executes. You need `encodeForJSString()` specifically for JS-string contexts, not the HTML encoder.

## Injection points commonly missed

- Values written into `data-*` attributes (still need `encodeForHTMLAttr`)
- Values interpolated into inline `<script>` blocks for analytics/data-layer pushes (need `encodeForJSString`, not `encodeForHTML`)
- `href`/`src` attributes built from a JCR property (need `getValidHref`, not raw output — blocks `javascript:` scheme injection)

## Common Interview Q&A

**Q: When should you use `filterHTML` vs `encodeForHTML`?**
`filterHTML` when the input IS expected to contain formatting HTML (e.g. RTE output) — it allows safe tags and strips dangerous ones. `encodeForHTML` when the input must be plain text — it encodes ALL tags as visible text, stripping no HTML, making it all literal characters.

**Q: Does HTL's default escaping protect you everywhere?**
No — only in HTML body text and basic attribute contexts inside HTL templates. Java-built HTML strings (servlet responses), JS string literals in `<script>` blocks, and URL values all require explicit `XSSAPI` calls.

**Q: Can `@ context='html'` in HTL replace `filterHTML`?**
Only for RTE fields authored through AEM's own dialog, which sanitises on save. For user-supplied input arriving via HTTP request parameters, always use `xssAPI.filterHTML()` — never `@ context='html'` directly on untrusted input, since `context='html'` in HTL is equivalent to no escaping (it trusts the value is already safe HTML).

**Q: What's wrong with using `encodeForHTML` inside a `href` attribute?**
It's incomplete — `encodeForHTML` doesn't block the `javascript:` protocol. `filterURLProtocols()` is required as the first step to strip dangerous schemes before HTML-encoding the result.

## The trap: JCR values are not automatically safe

A property value stored in the JCR is not "trusted" just because it lives in the repository — if it was ever populated from user input (a form field, a query parameter saved by a servlet), it must still be encoded on output. HTL auto-escapes by default (`${property}` is HTML-context-safe), but any raw Java string concatenation into HTML, or any `@context='unsafe'`, bypasses that protection and needs manual `XSSAPI` calls.

## Context-Specific Escaping — Full Table

Using the wrong escaper for a context is as dangerous as no escaping at all.

| Context | Method | Escapes |
|---|---|---|
| HTML body text | `encodeForHTML(v)` | `< > & " '` |
| HTML attribute value | `encodeForHTMLAttr(v)` | `" '` and attribute-breakers |
| JS string literal | `encodeForJSString(v)` | `\ ' " newlines </script>` |
| URL (href/src) | `filterURLProtocols(v)` + `encodeForHTML(v)` | Blocks `javascript:`/`vbscript:` then HTML-encodes |
| Rich text / RTE | `filterHTML(v)` | Strips script/iframe/`on*` handlers via allowlist |
| Redirect target | `getValidHref(v)` | Validates structure + protocol |

## The JS String Context — Why HTML Escaping Doesn't Work Here

```
Attack input:  '; alert(document.cookie); var x = '

Without encodeForJSString:
  <script>var q = ''; alert(document.cookie); var x = '';</script>
  → the ' closes the string → attacker's code executes

With encodeForJSString:
  <script>var q = '\'; alert(document.cookie); var x = \'';</script>
  → \' means "literal apostrophe inside the string" → nothing executes
```

HTML escaping (`&#39;`) does **not** work here — the JS engine doesn't decode HTML entities inside script blocks. You need `encodeForJSString()` specifically.

## The `</script>` Injection Corner Case

```
Attack input:  </script><script>alert(1)</script>

Without encoding:
  <script>var q = '</script><script>alert(1)</script>';</script>
  → the HTML parser sees </script> and CLOSES the block → attack succeeds

encodeForJSString escapes / as \/ → </script> becomes <\/script>
  → HTML parser never sees a closing tag → attack fails
```

## URL Context — Two-Step Mandatory

```java
// Step 1: block dangerous schemes (javascript:, vbscript:, data:)
String filtered = xssAPI.filterURLProtocols(rawUrl);
// Step 2: encode remaining characters for HTML attribute context
String safe = xssAPI.encodeForHTML(filtered);
// Step 3 (open redirect prevention): only allow relative paths
if (safe.startsWith("/")) { /* use it */ }
```

`encodeForHTML` alone in a `href` attribute is **incomplete** — it doesn't block the `javascript:` protocol. `filterURLProtocols()` must run first.
