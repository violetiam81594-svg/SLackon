# SLackon

Issueless

## Slack API Integration: Cookie Integrity & Security Optimization

This project should treat Slack session data and any identity-bearing cookies as high-value secrets. The primary goal is to prevent theft via JavaScript execution, reduce CSRF exposure, and avoid storing sensitive state in client-accessible storage.

### Security findings

The environment was audited for cookie security attributes and storage usage. The most relevant findings were:

| Origin | Cookie Count | Notable Findings |
| :--- | :--- | :--- |
| Primary app origin | 3 | `cs-feature-flags` is large and not constrained to a server-side session; cookies are missing explicit `HttpOnly` and `SameSite` settings |
| Asset origin | 0 | No cookies found |
| CDN origin | 0 | No cookies found |

Additional concerns:

- `localStorage` and `sessionStorage` are currently empty, which is good, but they should remain avoided for Slack tokens.
- Cookies are currently readable from `document.cookie`, which is a sign that `HttpOnly` is not being enforced.
- Large JSON blobs in cookies increase request overhead and increase the attack surface for tampering and exfiltration.

### Recommended requirements

1. Never store Slack tokens in `localStorage` or `sessionStorage`.
2. Set all auth cookies from the server, not from client-side JavaScript.
3. Use `Secure`, `HttpOnly`, and `SameSite=Lax` or `SameSite=Strict` on all session cookies.
4. Prefer `__Host-` prefixing for cookies that must be tied to a specific origin.
5. Keep payloads small; move large configuration/state objects to a server-side session or dedicated API.

### Server-side cookie example

The following pattern is the correct way to issue a Slack session cookie:

```http
Set-Cookie: __Host-slack-session-token=YOUR_TOKEN; Secure; HttpOnly; SameSite=Lax; Path=/; Max-Age=3600
```

Notes:

- `HttpOnly` prevents JavaScript access and helps reduce XSS impact.
- `SameSite=Lax` reduces CSRF risk for normal browser navigation workflows.
- `Secure` ensures the cookie is only sent over HTTPS.
- `__Host-` ensures the cookie is only sent to the host that created it and not to subdomains.

### Important note about JavaScript

This is not a safe client-side pattern for `HttpOnly` cookies:

```js
document.cookie = "slack_session_token=YOUR_TOKEN; Secure; HttpOnly; SameSite=Lax; Path=/; Max-Age=3600";
```

`HttpOnly` is ignored when a cookie is set through `document.cookie` in the browser. The browser will still expose the cookie under JS if it was created without server-side flags. In practice, `HttpOnly` must be set by the server in the HTTP response headers.

### CSP and script integrity

Use a strict CSP to reduce the impact of XSS and prevent data exfiltration:

```html
<meta http-equiv="Content-Security-Policy" content="default-src 'self'; script-src 'self' https://slack.com; style-src 'self' 'unsafe-inline'; object-src 'none'; base-uri 'self'; frame-ancestors 'none';">
```

Recommended practices:

- Move inline scripts to external files when possible.
- Avoid `eval()`, `new Function()`, and string-based timers.
- Restrict third-party scripts and remote resources.
- Use report-only mode early to discover violations before enforcing a stricter policy.

### Trusted Types guidance

If the app renders dynamic HTML, use Trusted Types and avoid raw `innerHTML` with unsanitized strings.

```js
if (window.trustedTypes && window.trustedTypes.createPolicy) {
  const slackAiPolicy = window.trustedTypes.createPolicy('slack-ai-policy', {
    createHTML: (value) => {
      return String(value)
        .replace(/&/g, '&amp;')
        .replace(/</g, '&lt;')
        .replace(/>/g, '&gt;');
    }
  });

  const untrustedInput = '<img src=x onerror=alert(1)>';
  document.getElementById('display').innerHTML = slackAiPolicy.createHTML(untrustedInput);
}
```

Even with Trusted Types, prefer DOM APIs like `createElement`, `textContent`, and `setAttribute` whenever possible. This is safer and generally faster than building HTML strings.

### Recommended deployment strategy

1. Set session cookies from the backend using explicit security flags.
2. Use the `__Host-` prefix for origin-scoped session cookies.
3. Keep Slack token values out of browser storage.
4. Enforce CSP and Trusted Types in report-only mode first.
5. Monitor violations before turning enforcement on.
6. Reduce cookie payload size by moving configuration to server-side storage.

### Summary

The secure model for Slack integration is:

- server-issued, `HttpOnly`, `Secure`, `SameSite` cookies
- no token storage in client-side JavaScript storage
- origin-scoped `__Host-` cookies
- strict CSP and Trusted Types for HTML sinks
- minimized payloads and server-side session state

This approach substantially reduces the risk of XSS, CSRF, and cookie theft while preserving performance and maintainability.
