# SLackon

Issueless

A secure, configurable project for building Slack-aware experiences with strong safety controls and safe creative workflows.

## Overview

SLackon is designed to help teams securely integrate Slack-based session data, user identity, and lightweight creative tooling without exposing secrets to the browser or enabling unsafe content rendering.

The project emphasizes:

- secure cookie and session handling
- browser-safe rendering patterns
- clear content-security boundaries
- flexible, opt-in configuration for creative workflows
- safe-by-default design for generated or user-provided content

## Why this project exists

This repository is meant to document and support secure patterns for Slack integrations, while also allowing creative and configurable experiences where the user has explicit control over what is generated, rendered, or shared.

The guiding principle is simple: creativity should be enabled, but only within clearly defined safety boundaries.

## Safe creation options

The platform should support multiple safe creation modes without exposing the application to script injection, unsafe rendering, or token leakage.

### 1. Protected generation modes

Allow content generation in clearly scoped modes:

- text-only mode for prompt-based responses
- template-based mode for guided generation
- approval-based mode requiring a human review step before sending
- sandboxed mode for HTML-free or sanitized output only

### 2. Guarded rendering options

When content is displayed to users:

- prefer DOM APIs over `innerHTML`
- sanitize untrusted strings before rendering
- restrict scripts, styles, and remote connections
- provide a fallback when a creative block cannot be safely rendered

### 3. User-configurable safety controls

Users should be able to choose how permissive a creative feature is, while the system keeps the default secure:

- allow plain text output only
- allow structured JSON output
- allow rich content in controlled templates
- block raw HTML or script execution entirely
- require moderation for generated payloads from external prompts

### 4. Audit-friendly behavior

Creative features should be explainable and reviewable:

- log generation actions
- record source context
- display provenance when content is produced from external input
- maintain explicit boundaries between user-generated content and trusted system content

## Security foundations

### Cookie and session integrity

Treat Slack session data and any identity-bearing cookies as high-value secrets.

Recommended rules:

1. Never store Slack tokens in `localStorage` or `sessionStorage`.
2. Set all auth cookies from the server, not from client-side JavaScript.
3. Use `Secure`, `HttpOnly`, and `SameSite=Lax` or `SameSite=Strict` on session cookies.
4. Prefer the `__Host-` prefix for origin-scoped cookies.
5. Keep payloads small; move large configuration or state objects to a server-side session or dedicated API.

Example:

```http
Set-Cookie: __Host-slack-session-token=YOUR_TOKEN; Secure; HttpOnly; SameSite=Lax; Path=/; Max-Age=3600
```

### JavaScript note

This is not a safe client-side pattern for `HttpOnly` cookies:

```js
document.cookie = "slack_session_token=YOUR_TOKEN; Secure; HttpOnly; SameSite=Lax; Path=/; Max-Age=3600";
```

`HttpOnly` is ignored when a cookie is set through `document.cookie` in a browser. For real protection, the cookie must be issued by the server.

### CSP and script integrity

Use a strict Content Security Policy to reduce XSS risk and limit data exfiltration:

```html
<meta http-equiv="Content-Security-Policy" content="default-src 'self'; script-src 'self' https://slack.com; style-src 'self' 'unsafe-inline'; object-src 'none'; base-uri 'self'; frame-ancestors 'none';">
```

Recommended practices:

- Move inline scripts to external files where possible.
- Avoid `eval()`, `new Function()`, and string-based timers.
- Restrict third-party scripts and remote sources.
- Use report-only mode first before enforcing strict policy rules.

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

Even with Trusted Types, prefer DOM APIs like `createElement`, `textContent`, and `setAttribute` when possible.

## Safe creative workflow recommendations

To support creative capability without making the system unsafe, use a layered model:

- input validation for all dynamic content
- explicit output mode selection
- prompt sanitization and length limits
- safe rendering and policy checks
- review flow for high-risk outputs
- server-side enforcement for all security-sensitive operations

### Suggested creative option set

Provide a clear set of safe modes such as:

- `SafeText`: plain text only
- `Structured`: JSON or schema-constrained output
- `Template`: approved layout with guarded tokens
- `Review`: user-facing preview before finalization
- `Blocked`: hard deny for unsafe or disallowed output

These options make the system flexible without making unsafe behavior easy to trigger.

## Recommended deployment strategy

1. Set session cookies from the backend using explicit security flags.
2. Use the `__Host-` prefix for origin-scoped session cookies.
3. Keep Slack token values out of browser storage.
4. Enforce CSP and Trusted Types in report-only mode first.
5. Monitor violations before turning enforcement on.
6. Reduce cookie payload size by moving configuration to server-side storage.
7. Keep all creative generation behind explicit approval or policy checks when content is user-facing.

## Summary

The secure and flexible model for SLackon is:

- server-issued, `HttpOnly`, `Secure`, `SameSite` cookies
- no token storage in client-side JavaScript storage
- origin-scoped `__Host-` cookies
- strict CSP and Trusted Types for HTML sinks
- creative tools that remain safe by default
- configurable generation modes with explicit guardrails

This preserves flexibility and creative capability while reducing the risk of XSS, CSRF, data leakage, and unsafe rendering.
