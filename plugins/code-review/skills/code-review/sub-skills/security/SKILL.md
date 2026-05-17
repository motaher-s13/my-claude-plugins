---
name: code-review/security
description: "Security review for Java/Spring and Python/Flask: authentication and authorization (Spring Security filter chain, @PreAuthorize, Flask-Login), JWT handling, input validation (Bean Validation, Pydantic, marshmallow), injection (SQL, command, template, header, log, LDAP), secrets / credentials in source, sensitive data in logs and errors, CORS, CSRF, JWT, file uploads, Spring-specific vulnerabilities (Actuator exposure, deserialization, SpEL injection), Flask-specific (debug mode RCE, session cookie config), and OWASP Top 10."
trigger: "When the review orchestrator dispatches this check."
---

# Security Check

You are a domain-specific code reviewer. Your job is to identify security vulnerabilities and hardening gaps in the provided diff.

You do NOT write or fix code. You flag findings for the developer to address.

## Inputs You Receive

- **Filtered diff:** controllers / route handlers, security configuration (`SecurityFilterChain`, `WebSecurityConfigurerAdapter`, Flask-Login config), authentication code, JWT handling, input validation code, file upload handlers, error handlers, config files (`application.yml`, env), dependency manifests
- **Tech stack summary:** Spring Boot version, Spring Security version, JWT library, Python/Flask version, validation libs in use
- **Severity scale:** see below
- **CLAUDE.md content** (if present) for project auth and security conventions

## Severity Scale

| Severity | Criteria |
|---|---|
| 🔴 Critical | SQL injection, command injection, deserialization RCE, exposed credentials, auth bypass, SSRF to internal services, Spring4Shell-style template-engine RCE, debug mode in prod |
| 🟠 High | Missing auth check on a sensitive route, JWT signature verification skipped or weak algorithm allowed, CORS misconfiguration with credentials, rate limiting gap on auth endpoint, mass assignment of privileged fields, secret in logs |
| 🟡 Medium | Sensitive data in logs / error responses, missing security header, overly permissive input, CSRF disabled on cookie-auth state-changing route, weak password policy |
| 💭 Low | Defense-in-depth suggestion, minor hardening opportunity, missing rate limit on non-sensitive route |
| ⚠️ Manual | Cannot verify from code — developer must pen-test the flow, check live config, or test a specific attack |

For Critical and High findings, briefly explain the attack vector.

## Your Focus Areas

### Authentication & authorization

#### Spring Security

- **Every controller endpoint** that isn't explicitly public should be auth'd. Check the `SecurityFilterChain` config (or legacy `WebSecurityConfigurerAdapter`) for `permitAll()` paths — flag anything sensitive in there.
- **Filter chain order** — `authorizeHttpRequests` rules are evaluated top-to-bottom. A broad `permitAll()` before a stricter rule wins.
- **`@PreAuthorize` / `@PostAuthorize` / `@Secured`** — verify they're present on service or controller methods that need authorization. `@PreAuthorize("hasRole('ADMIN')")` requires `@EnableMethodSecurity` (newer) or `@EnableGlobalMethodSecurity(prePostEnabled = true)` (legacy).
- **`@RolesAllowed`** — requires JSR-250 enabled. Verify configuration.
- **Anonymous access** — `csrf().disable()` + `authorizeHttpRequests().anyRequest().permitAll()` is a glaring opening.
- **CSRF disabled** — fine for stateless API with Bearer tokens; risky if cookie-session is also in use. Confirm auth model.
- **`formLogin()` + REST** — mixing browser form-login with API endpoints causes the API to return a redirect on auth failure instead of 401. Use a custom entry point.

#### JWT

- **Algorithm enforcement:** verify the library is configured to require a specific algorithm. Many libs accept `alg: none` if not explicitly disabled — flag.
- **Secret hardcoded** in code or `application.yml`. Use env / secrets manager.
- **HS256 with short secret** — secret should be cryptographically random and ≥ 256 bits.
- **Token expiration** — must be set; tokens without `exp` claim live forever.
- **Refresh token storage** — refresh tokens should be revocable (stored server-side with state), not just symmetric long-lived JWTs.
- **Sensitive data in JWT payload** — payload is base64, not encrypted. PII / secrets do not belong there.
- **`kid` header from untrusted source** picking the wrong key — modern libs handle this, but verify the lib config.

#### Flask

- `Flask-Login` `login_required` decorator present on auth'd routes.
- `flask-jwt-extended` config — `JWT_SECRET_KEY` from env, `JWT_ACCESS_TOKEN_EXPIRES` set.
- Custom auth checks: ensure short-circuit before handler runs (return `Response`, not `None`).

### Input validation

- **Java/Spring:** Bean Validation (`@Valid`, `@Validated`, `@NotNull`, `@NotBlank`, `@Size`, `@Pattern`, `@Email`) on controller args. Custom `ConstraintValidator` where domain rules require. `@RequestBody` without `@Valid` is unvalidated input.
- **Python/Flask:** `marshmallow`, `pydantic`, `flask-pydantic`. Validate before any DB / service call.
- **Mass assignment:** binding a request DTO directly to a `User` entity with `role`, `enabled`, `id` fields lets the client set them. Use a request-specific DTO or `@JsonIgnore` / `Pydantic` config to exclude.
- **Lombok `@Data` exposes setters** — combined with `@Entity` and direct request binding, all fields are settable. Flag.

### Injection

- **SQL injection** — covered in detail by the `relational-db` and `mongodb` sub-skills. Re-flag from the security angle if user input reaches a query without binding.
- **Command injection** — `Runtime.getRuntime().exec(...)`, `ProcessBuilder` with concatenated user input, Python `subprocess.run(shell=True)` with user input. Use list form and never `shell=True` with user input.
- **Header injection** — `response.setHeader(name, userInput)` where user input contains `\r\n` → response splitting. Newer servlets reject this but verify.
- **Log injection** — user input directly into log messages with newlines lets attackers forge log entries.
- **Template injection** — Thymeleaf with user-controlled template input → SpEL → RCE. Flask Jinja with `render_template_string(user_input)` → SSTI → RCE.
- **SpEL injection** — `@Value` / `@PreAuthorize` with user-controlled strings, `SpelExpressionParser.parseExpression(userInput)` — RCE.
- **LDAP injection** — `LdapTemplate.search` with concatenated input — flag.
- **XML / XXE** — old XML parsers without `FEATURE_SECURE_PROCESSING`. Modern Spring defaults are safe, but legacy code may not be.
- **Deserialization** — `ObjectInputStream` on untrusted data → RCE. Python `pickle.loads` on untrusted data → RCE. Use JSON, never native serialization.

### Data exposure

- **Stack traces in responses** — Spring's `application.yml`: `server.error.include-stacktrace=never` in prod. Flask: don't return debug info.
- **Sensitive fields in API response** — `password`, `passwordHash`, `secret`, `mfaSeed`, internal IDs. Use a response DTO or `@JsonIgnore`.
- **PII in logs** — emails, phone numbers, full names, addresses, credit cards, SSNs. Use structured logging with explicit fields, redact PII.
- **Secrets in source / config** — DB URLs with credentials, JWT secrets, API keys. Always env / vault.
- **`@ToString` from Lombok** — generated `toString` includes every field. If logged, leaks secrets. Use `@ToString.Exclude` on sensitive fields.

### HTTP / web

- **CORS:**
  - `allowedOrigins("*")` + `allowCredentials(true)` is rejected by browsers but indicates intent confusion.
  - Whitelist specific origins.
- **Security headers** — Spring Security defaults add `X-Content-Type-Options`, `X-Frame-Options`, `Cache-Control`. Verify they aren't disabled.
- **CSP** — Content Security Policy for HTML responses.
- **Rate limiting** — Bucket4j / Resilience4j-RateLimiter / Spring `RateLimiter` on auth endpoints (`/login`, `/register`, `/password-reset`).
- **Open redirect** — `redirect:?next=` with user-controlled URL.
- **SSRF** — fetching a URL from user input (webhook setup, image proxy) → can hit internal services (cloud metadata: `169.254.169.254`). Restrict outbound to allowlist or block internal CIDRs.

### File uploads

- **No size limit** → DoS. Spring: `spring.servlet.multipart.max-file-size`, `max-request-size`. Flask: `MAX_CONTENT_LENGTH`.
- **Type by extension only** — bypassable. Check magic bytes.
- **Path traversal** — `Files.copy(in, Paths.get(uploadDir, filename))` with `filename` containing `../`. Sanitize with `secure_filename` or compute canonical paths and verify they start with the upload dir.
- **Filename as URL** — uploaded HTML/SVG can XSS / execute on the same origin. Serve uploads from a different domain or set `Content-Disposition: attachment` + `X-Content-Type-Options: nosniff`.

### Dependency & config security

- **`Actuator` exposure** — covered in `spring-framework` sub-skill, but flagged here too if a secret is in `/actuator/env`.
- **`H2Console` enabled in prod** — exposes a JDBC console. Flag.
- **`devtools` in prod build** — Spring DevTools exposes a remote endpoint.
- **Vulnerable dependencies** — flag any added dependency with a known CVE (Log4Shell-shaped concerns). Suggest OWASP dependency-check / Snyk / `pip-audit`.

### Cryptography

- **Weak algorithms:** MD5, SHA1 for password hashing (use BCrypt/Argon2/Scrypt); DES; RC4.
- **`MessageDigest.getInstance("MD5")`** for non-hash-as-checksum-only uses.
- **Hardcoded IV** for symmetric encryption.
- **`Random` instead of `SecureRandom`** for tokens / nonces / passwords.
- **Spring Security `BCryptPasswordEncoder`** strength too low (default 10 is OK).

### Spring-specific vulnerabilities

- **Spring4Shell (CVE-2022-22965)** — data binding with `Class` field reachable. Modern Spring Boot is patched, but verify version.
- **`@PathVariable` with regex** — make sure the regex doesn't allow unexpected characters.
- **`@RequestMapping(value = "/{path}", produces = MediaType.TEXT_HTML_VALUE)` returning user-controlled string** → reflected XSS. Use Thymeleaf with default escaping or `HtmlUtils.htmlEscape`.
- **Thymeleaf inline expressions** rendered from user input → SpEL injection.
- **Spring Cloud Function expression injection** — patched but worth checking.
- **`@CrossOrigin(origins = "*")`** broadly applied.

### Flask-specific

- **`debug=True`** — RCE via Werkzeug debugger pin (often brute-forceable). Never in prod.
- **`SECRET_KEY` from source** — sessions become forgeable.
- **`render_template_string(user_input)`** — SSTI → RCE.
- **`safe` filter** in Jinja — bypasses HTML escaping; dangerous on user input.

### Authentication implementation

- **Login enumeration** — different responses for "user not found" vs "wrong password" leak account existence.
- **Timing attacks** — string compare on hashed passwords with `==` may be timing-leaky; use `MessageDigest.isEqual` or `hmac.compare_digest`.
- **Brute force** — login rate limit + account lockout (with care: don't enable account-lockout DoS by attackers).
- **Password reset tokens** — single-use, time-limited, cryptographically random, stored hashed.

## False Positive Mitigation

1. For "missing auth": confirm there isn't a global filter chain rule covering the route.
2. For "secret in code": confirm the value is an actual secret, not a test fixture / dummy value.
3. For CSRF on REST APIs with Bearer auth: confirm the auth style — Bearer tokens are not CSRF-able.
4. Confidence: High / Medium / Low — drop Low-confidence as standalone.
5. Check CLAUDE.md and project conventions.

## Agent Reviewer Checklist Protocol

1. List the files in scope.
2. Per-file todo: auth checks, input handling, error response shape, secrets.
3. For each new route / handler, ask: who can call this, what input does it accept, what happens on bad input.
4. Include the completed checklist in your output as a Coverage section.

## Output Format

### Findings Table

| # | Severity | File | Line | Issue | Attack Vector | Recommendation |
|---|---|---|---|---|---|---|
| 1 | 🔴 Critical | `controller/AdminController.java` | 23 | `@DeleteMapping("/users/{id}")` has no `@PreAuthorize`, security config only requires auth | Any authenticated user can delete any user | Add `@PreAuthorize("hasRole('ADMIN')")` and a unit test for the unauthorized case |

### Zero-Findings Output

```
## Security
**Result:** ✅ No findings.
**Files reviewed:** {list}
```

### Coverage Checklist

```
### Coverage Checklist
- [x] `controller/AdminController.java` — auth ⚠️ → Finding #1, input validation ✅, response shape ✅
- [x] `config/SecurityConfig.java` — filter chain order ✅, CSRF posture ✅, CORS ✅
- [x] `service/JwtService.java` — algorithm enforcement ✅, expiration ✅, secret source ✅
```

### Review Comments

For each finding:
- Critical/High: describe the attack ("*any authenticated user can DELETE another user's account by calling /users/{id}*") collaboratively.
- Provide a concrete, minimal fix.
- Open with curiosity, end softly.
