# SLackon

Issueless

## Trusted Types Implementation and Performance Optimization

This project focuses on preventing DOM-based XSS by enforcing Trusted Types and improving the performance of the sanitization policies used in client-side rendering.

### Why this matters

DOM-based XSS can occur when untrusted input is passed into sinks such as `innerHTML`, `outerHTML`, or similar APIs without sanitization. Trusted Types helps by requiring that DOM sinks receive a `TrustedHTML`, `TrustedScript`, or `TrustedScriptURL` object instead of raw strings.

### Recommended pattern

```js
if (window.trustedTypes && window.trustedTypes.createPolicy) {
  const slackAiPolicy = window.trustedTypes.createPolicy('slack-ai-policy', {
    createHTML: (value) => {
      return value
        .replace(/&/g, '&amp;')
        .replace(/</g, '&lt;')
        .replace(/>/g, '&gt;');
    }
  });

  const untrustedInput = '<img src=x onerror=alert(1)>';
  document.getElementById('display').innerHTML = slackAiPolicy.createHTML(untrustedInput);
}
```

### Enforcement

To make the policy mandatory, add the following response header:

```http
Content-Security-Policy: require-trusted-types-for 'script';
```

This ensures that script execution is blocked unless the application passes trusted values through the proper API.

### Performance guidance

The main performance risks in Trusted Types implementations are:

- Synchronous sanitization logic in `createHTML` that blocks the main thread
- Recreating security policies inside loops or hot paths
- Redundant sanitization of static content that is already trusted

Best practices:

- Create Trusted Types policies once at the application entry point
- Sanitize only dynamic user-generated content
- Avoid expensive regex-heavy sanitization in critical render paths
- Use faster, battle-tested libraries like DOMPurify when needed
- Check DevTools Performance for tasks longer than 50 ms and optimize them

### CSP overview

Content Security Policy (CSP) is a browser security mechanism that reduces the impact of XSS by defining which sources of scripts, styles, images, and other resources are allowed.

A typical policy looks like this:

```http
Content-Security-Policy: default-src 'self'; script-src 'self' https://cdn.example.com; style-src 'self' 'unsafe-inline'; img-src 'self' data:;
```

Core ideas:

- Restrict trusted content to allowlisted origins
- Block inline script and event-handler execution when possible
- Avoid `eval()` and string-based script execution
- Use `report-uri` or report-only mode to monitor policy violations
- Prefer external scripts over inline code to improve performance and security

### Recommended deployment strategy

1. Start in report-only mode to identify violations.
2. Move inline scripts and styles into external assets.
3. Enforce Trusted Types and CSP gradually.
4. Monitor violation reports and refine the allowlist.
5. Keep the policy strict but practical for your app's required third-party resources.

### Summary

`SLackon` is designed around secure client-side rendering practices. The recommended approach is to combine Trusted Types with a strict CSP policy, restrict sanitization to dynamic content, and avoid expensive policy creation in hot paths.
