# 14 - Security Standards (Microservices)

> **Author:** Emerson Lima — [github.com/Emersondll](https://github.com/Emersondll)
>
> Mandatory security baseline for every Java 26 / Spring Boot service in this
> workspace. Apply these rules on every refactoring, even when the task does not
> mention security. Violating any **MUST** item is a build-blocking defect.

---

## 0. Threat Model & Topology

- The **BFF / API gateway is the only public entry point**. Downstream services
  (`user`, `payment`, `contract`, `scheduling`, and any new microservice) are
  internal and MUST NOT be reachable directly from the internet.
- Service-to-service eventing uses the **message broker (RabbitMQ)**, not HTTP.
- Identity flows from the auth-service → BFF (JWT) → downstream (forwarded
  identity headers behind a shared secret). Downstream services **never** trust
  identity headers on their own.

---

## 1. Trust Boundary — Gateway Authentication (MUST)

Every downstream service MUST:

1. Depend on `spring-boot-starter-security` (and `spring-security-test` in test scope).
2. Provide a `security/GatewayAuthFilter` (a `OncePerRequestFilter`) that only
   authenticates a request when it carries a valid shared `X-Internal-Token`.
   Only then are the forwarded identity headers (`X-User-Id`, `X-User-Role`)
   honoured. The secret comparison MUST be constant-time (`MessageDigest.isEqual`).
3. Provide a `config/SecurityConfig` that is **stateless**, has **CSRF disabled**
   (no cookies/sessions), permits only health/info/docs, and requires
   authentication for everything else. **`permitAll()` on `anyRequest()` is FORBIDDEN.**

> **Never trust `X-User-Id` / `X-User-Role` without a valid `X-Internal-Token`.**
> This is the rule that prevents header-spoofing IDOR if a service is reached directly.

### Reference filter

```java
@Slf4j
@Component
public class GatewayAuthFilter extends OncePerRequestFilter {

    public static final String TOKEN_HEADER = "X-Internal-Token";
    public static final String USER_ID_HEADER = "X-User-Id";
    public static final String USER_ROLE_HEADER = "X-User-Role";

    private final byte[] expectedToken;

    public GatewayAuthFilter(@Value("${security.gateway.token}") final String gatewayToken) {
        this.expectedToken = gatewayToken.getBytes(StandardCharsets.UTF_8);
    }

    @Override
    protected void doFilterInternal(@NonNull HttpServletRequest request,
                                    @NonNull HttpServletResponse response,
                                    @NonNull FilterChain chain) throws ServletException, IOException {
        final String token = request.getHeader(TOKEN_HEADER);
        if (token != null && MessageDigest.isEqual(expectedToken, token.getBytes(StandardCharsets.UTF_8))) {
            final String userId = request.getHeader(USER_ID_HEADER);
            final String role = request.getHeader(USER_ROLE_HEADER);
            final String principal = (userId != null && !userId.isBlank()) ? userId : "gateway-service";
            final String authority = "ROLE_" + ((role != null && !role.isBlank()) ? role : "SERVICE");
            SecurityContextHolder.getContext().setAuthentication(
                new UsernamePasswordAuthenticationToken(principal, null,
                    List.of(new SimpleGrantedAuthority(authority))));
        } else {
            SecurityContextHolder.clearContext();
        }
        chain.doFilter(request, response);
    }
}
```

### Reference config

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    private final GatewayAuthFilter gatewayAuthFilter;

    public SecurityConfig(final GatewayAuthFilter gatewayAuthFilter) {
        this.gatewayAuthFilter = gatewayAuthFilter;
    }

    @Bean
    public SecurityFilterChain filterChain(final HttpSecurity http) throws Exception {
        http
            .csrf(AbstractHttpConfigurer::disable)
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health", "/actuator/health/**", "/actuator/info",
                                 "/openapi.yaml", "/swagger-ui/**", "/swagger-ui.html", "/v3/api-docs/**")
                    .permitAll()
                .anyRequest().authenticated())
            .addFilterBefore(gatewayAuthFilter, UsernamePasswordAuthenticationFilter.class);
        return http.build();
    }
}
```

### Required property (env-driven, no insecure default in prod)

```yaml
security:
  gateway:
    token: ${GATEWAY_TOKEN:local-dev-gateway-token}   # MUST be overridden in production
```

---

## 2. BFF Edge Security (MUST)

The BFF / gateway MUST:

- Validate the JWT on every non-public request (stateless, `anyRequest().authenticated()`,
  public only for `/auth/**`, docs, `/actuator/health`).
- Propagate the trust boundary to downstream via a Feign `RequestInterceptor`:
  attach `X-Internal-Token` on **every** call, plus `X-User-Id` and `X-User-Role`
  derived from the authenticated principal. The token value MUST match the
  downstream `security.gateway.token`.
- Enforce a **strict CORS allowlist** (`security.cors.allowed-origins`). Never use
  `*` together with `allowCredentials(true)`.
- Emit **security response headers**: HSTS (`includeSubDomains`, 1 year),
  `X-Frame-Options: DENY`, `X-Content-Type-Options: nosniff`,
  `Referrer-Policy: no-referrer`, and a restrictive `Content-Security-Policy`.
- Apply a **brute-force rate limiter** on the authentication endpoints
  (`/**/auth/**`). A single-instance in-memory fixed window is acceptable as a
  baseline; back it with a shared store (Redis) for multi-instance deployments.

---

## 3. Actuator Exposure (MUST)

- Expose only `health,info,metrics,prometheus`. **Never** expose `env`, `heapdump`,
  `loggers`, `threaddump`, `beans`, `configprops`.
- `metrics` and `prometheus` MUST require authentication (covered by
  `anyRequest().authenticated()`); only `health`/`info` are public.

---

## 4. Input Validation & Injection (MUST)

- Validate every request body/param with Jakarta Validation (`@Valid`,
  `@NotBlank`, `@Email`, `@Size`, ...). Validation failures → `400`.
- Use Spring Data **derived queries** or parameterized queries. **Never** build
  queries by string concatenation, and never pass unsanitized user input into a
  regex/`$where`/dynamic query.
- Treat all forwarded/opaque bodies as untrusted; bind to typed records so unknown
  fields are ignored.

---

## 5. Sensitive Data & Logging (MUST)

- **Never log** passwords, tokens, full card data, CPF, or raw secrets.
- Mask PII (e-mail, phone) before logging — use a `util/LogSanitizer`
  (e.g. `john@test.com` → `j***@test.com`).
- The fallback `500` handler MUST return a **generic** message
  (`"An unexpected error occurred"`); stack traces and internal messages are logged
  server-side only, never returned to the client.

---

## 6. Authorization (MUST)

- **Object-level / ownership (anti-IDOR):** a caller may only access or mutate
  resources it owns. Compare the path/resource owner against the authenticated
  principal (`X-User-Id`) and return `403` on mismatch (unless an admin role applies).
- **Role-based:** restrict role-specific routes (`@PreAuthorize` / request matchers).
  Authentication alone (`authenticated()`) is **not** authorization.
- **Server-controlled fields (no mass assignment):** never accept privilege fields
  from the request body. Examples that MUST be server-controlled and ignored on
  input: user `role`/`type`, verification/approval status, ratings, balances,
  payment amounts/status. Changing a user's role through a profile update is a
  privilege-escalation bug.

---

## 7. Secrets Management (MUST)

- All secrets come from environment variables / a secrets manager — never hardcoded.
- Local defaults are for development only and MUST be overridden in production
  (`GATEWAY_TOKEN`, broker/DB credentials, JWT keys). Prefer fail-fast when a
  required secret is missing in production.
- Keep `.env` files git-ignored.

---

## 8. Transport & Network (SHOULD — infrastructure)

- Terminate TLS at the edge; enforce TLS to the broker/DB; prefer **mTLS** for
  service-to-service traffic.
- Keep downstream services on a **private network**; the gateway/BFF is the only
  ingress.

---

## 9. Dependencies (MUST)

- Run OWASP Dependency-Check (or equivalent) in CI; keep Spring Boot on the latest
  patch of the supported minor line.
- Do not add a dependency without a clear justification (prefer the platform's
  native capabilities).

---

## Security Checklist (per service)

- [ ] `spring-boot-starter-security` present; `GatewayAuthFilter` + `SecurityConfig` in place
- [ ] Stateless chain, CSRF disabled, **no `permitAll()` on `anyRequest()`**
- [ ] Identity headers honoured only with a valid `X-Internal-Token` (constant-time compare)
- [ ] Actuator limited to `health,info,metrics,prometheus`; metrics/prometheus authenticated
- [ ] `@Valid` on all controller inputs; no string-concatenated queries
- [ ] PII masked in logs; no secrets logged; generic `500` body
- [ ] Ownership (IDOR) and role checks enforced where applicable
- [ ] Privilege fields (role/status/rating/amount) server-controlled, ignored on input
- [ ] Secrets env-driven; production overrides documented; `.env` git-ignored
- [ ] `GatewayAuthFilter` covered by a unit test (filter logic is not coverage-excluded)
- [ ] `mvn verify` green; JaCoCo gate met

### BFF additionally

- [ ] Feign interceptor sends `X-Internal-Token` + `X-User-Id`/`X-User-Role` on every call
- [ ] Strict CORS allowlist (no `*` with credentials)
- [ ] Security headers (HSTS, frame-deny, content-type, referrer, CSP)
- [ ] Rate limiter on `/**/auth/**`

---

## Coverage Note

`*Config` classes are JaCoCo-excluded, so `SecurityConfig` does not need coverage.
The `security` package filters (`GatewayAuthFilter`, rate limiter) are **not**
excluded on downstream services and MUST have unit tests. On the BFF the whole
`security` package is coverage-excluded, but the rate limiter SHOULD still be
unit-tested for correctness.

---

*Maintained by [Emerson Lima](https://github.com/Emersondll)*
