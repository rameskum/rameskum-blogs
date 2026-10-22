---
title: 'JWT Auth in Spring Boot 4: The Security 7 Way'
date: 2026-10-20T12:00:00+05:30
draft: false
ShowToc: true
description: >-
  The 2019 JWT tutorial — custom OncePerRequestFilter, a third-party JWT
  library, five glue classes — is dead. Spring Security 7's built-in OAuth2
  Resource Server does it in one config: lambda DSL, stateless JWT validation,
  Argon2id password upgrades, refresh-token rotation, and MockMvc tests.
tags:
  - spring-boot
  - spring-security
  - java
  - backend
  - api
categories: article
keywords:
  - spring security 7 jwt
  - spring boot 4 jwt authentication
  - oauth2 resource server jwt
  - argon2id spring security
---

Google "Spring Boot JWT tutorial" and the top results still hand you the 2019 recipe: a custom `OncePerRequestFilter` that parses the `Authorization` header by hand, a third-party JWT library, a `UserDetailsService`, and five config classes of glue. It works — the way a hand-rolled ORM works. On Spring Boot 4, Spring Security 7 has first-class JWT support through the built-in OAuth2 Resource Server. No custom filter. No third-party JWT library. Here's the modern shape, including what Security 7 removed out from under the old tutorials.

## What Security 7 removed (why old tutorials don't compile)

Each item links the official migration material that documents it — mostly the [Migrating to 7.0 guide](https://docs.spring.io/spring-security/reference/migration/index.html):

- **The `.and()` DSL is gone.** The old chain-style configuration was deprecated in 6.1 and removed in 7.0; the **lambda DSL is now mandatory**. Every config snippet from the Security 5 era needs rewriting, and OpenRewrite can automate much of it.
- **`authorizeRequests` is gone** — deprecated in 6.0, removed in 7.0; use `authorizeHttpRequests`.
- **`AntPathRequestMatcher` and `MvcRequestMatcher` are gone** — the migration guide's [web migrations](https://docs.spring.io/spring-security/reference/6.5/migration-7/web.html) says it directly: "In Spring Security 7, `AntPathRequestMatcher` and `MvcRequestMatcher` are no longer supported," and Security 7 uses `PathPatternRequestMatcher` (in `org.springframework.security.web.servlet.util.matcher`) by default. The 7.x API docs bear that out: the old classes no longer have pages, `PathPatternRequestMatcher` does.
- **The OAuth2 password grant is removed** (OAuth 2.1 calls it legacy) — the 6.5 reference already announced it "will be removed in Spring Security 7." If something in your system still uses it, that's a migration project of its own.
- **The legacy Access API moved out** — as of Security 7, `AccessDecisionManager`, `AccessDecisionVoter`, and friends live in a legacy module, `spring-security-access`, per the [authorization reference](https://docs.spring.io/spring-security/reference/servlet/authorization/index.html). Expect import churn, not a rewrite.

The practical migration path: move to the latest 6.x first, fix every deprecation warning there, then upgrade. The deprecations *are* the migration guide.

## The dependency: the renamed starter

In Boot 4 the resource-server starter was renamed — the old `spring-boot-starter-oauth2-resource-server` is deprecated in favor of:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security-oauth2-resource-server</artifactId>
</dependency>
```

(The version is managed by Boot's BOM, as usual.) That one starter brings in the resource-server and JOSE (Nimbus) support everything below uses.

## The resource server: one config class

Your API doesn't issue tokens in this shape — it *validates* them. That's the resource server's job, and the whole configuration is this:

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())  // bearer-token chain only: no cookies are read here — the refresh endpoint below is a separate story
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers(HttpMethod.POST, "/api/orders")
                    .hasAuthority("SCOPE_orders:write")
                .anyRequest().authenticated())
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(Customizer.withDefaults()));
        return http.build();
    }
}
```

Point it at your issuer and the framework does the rest — the JWK set is fetched from the issuer and cached (not re-fetched on every request), signatures are verified against it, and expiry is checked per request:

```properties
spring.security.oauth2.resourceserver.jwt.issuer-uri=https://auth.example.com
```

That's the filter, the parser, and the validation — all framework code, all maintained. The custom-filter tutorials made you the maintainer of the most security-sensitive code in the app.

One validation gap ships by default, though: with only `issuer-uri`, Boot validates the `iss` claim and the timestamps — but **not the audience (`aud`)**. If your issuer signs tokens for several APIs, a token minted for a different service validates here too. Add an audience validator by declaring the decoder yourself:

```java
@Bean
JwtDecoder jwtDecoder() {
    NimbusJwtDecoder decoder = NimbusJwtDecoder
        .withIssuerLocation("https://auth.example.com")
        .build();
    OAuth2TokenValidator<Jwt> issuer =
        JwtValidators.createDefaultWithIssuer("https://auth.example.com");
    OAuth2TokenValidator<Jwt> audience = new JwtClaimValidator<List<String>>(
        JwtClaimNames.AUD, aud -> aud != null && aud.contains("orders-api"));
    decoder.setValidator(new DelegatingOAuth2TokenValidator<>(issuer, audience));
    return decoder;
}
```

Now a token has to be issued by your issuer, unexpired, *and* addressed to your API.

## And who issues the tokens?

Scope note: everything above is the *validating* side. The login, password, and refresh sections below are the *issuing* side — in a larger system that's a separate authorization server (Spring Authorization Server, Keycloak, your IdP), and this API never sees a password. If the same service does issue its own tokens, it signs them with a `JwtEncoder` — Security ships `NimbusJwtEncoder`, backed by an RSA key:

```java
@Bean
JwtEncoder jwtEncoder(RSAKey rsaKey) {
    JWKSource<SecurityContext> jwkSource =
        new ImmutableJWKSet<>(new JWKSet(rsaKey));
    return new NimbusJwtEncoder(jwkSource);
}

// issuing is then a claims set plus one encode call
JwtClaimsSet claims = JwtClaimsSet.builder()
    .issuer("https://auth.example.com")
    .subject(user.getUsername())
    .expiresAt(Instant.now().plus(10, ChronoUnit.MINUTES))
    .claim("scope", "orders:write")
    .build();
String token = jwtEncoder.encode(JwtEncoderParameters.from(claims)).getTokenValue();
```

Either way, the resource-server config doesn't change — it validates against the issuer's published JWK set, whoever signed the token.

## Claims to authorities: the `SCOPE_` trap

Out of the box, the JWT's `scope`/`scp` claims become authorities prefixed with `SCOPE_` — `orders:read` in the token is `SCOPE_orders:read` in your rules. The failure mode everyone hits once: roles in a custom claim checked with `hasRole("ADMIN")` while the converter produced something else entirely, and admins get 403s. If your tokens carry roles in a custom claim, supply a `JwtAuthenticationConverter` that maps them explicitly — but **compose, don't replace**: setting a custom granted-authorities converter *replaces* the default one, so a converter that only reads your `roles` claim silently deletes every `SCOPE_` authority, and a rule like `hasAuthority("SCOPE_orders:write")` then 403s every request. Delegate to the default converter for scopes, then add your roles on top:

```java
@Bean
JwtAuthenticationConverter jwtAuthenticationConverter() {
    var scopes = new JwtGrantedAuthoritiesConverter();  // the default: scope/scp → SCOPE_*

    var converter = new JwtAuthenticationConverter();
    converter.setJwtGrantedAuthoritiesConverter(jwt -> {
        Collection<GrantedAuthority> authorities = new ArrayList<>(scopes.convert(jwt));
        List<String> roles = jwt.getClaimAsStringList("roles");
        if (roles != null) {
            roles.forEach(role ->
                authorities.add(new SimpleGrantedAuthority("ROLE_" + role)));
        }
        return authorities;
    });
    return converter;
}
```

Rule of thumb: whatever the converter produces is what your matchers must check. Print the authorities in a test before debugging production 403s.

## Passwords: bcrypt → Argon2id without a flag day

If your service does check passwords (a login endpoint issuing tokens), the hashing story matters as much as the token story. Spring's answer is the `DelegatingPasswordEncoder`: every stored hash carries an algorithm prefix (`{bcrypt}…`, `{argon2}…`), verification delegates by prefix, and **new** hashes use whichever encoder you name as the default:

```java
@Bean
PasswordEncoder passwordEncoder() {
    Map<String, PasswordEncoder> encoders = new HashMap<>();
    encoders.put("argon2", Argon2PasswordEncoder.defaultsForSpringSecurity_v5_8());
    encoders.put("bcrypt", new BCryptPasswordEncoder());
    return new DelegatingPasswordEncoder("argon2", encoders);
}
```

One dependency gotcha: `Argon2PasswordEncoder` is implemented on top of BouncyCastle, and it isn't pulled in for you — without it the encoder fails at runtime with `NoClassDefFoundError`. Add:

```xml
<dependency>
    <groupId>org.bouncycastle</groupId>
    <artifactId>bcprov-jdk18on</artifactId>
</dependency>
```

The migration is lazy and needs no password-reset email blast — and you don't hand-roll it. `DaoAuthenticationProvider` already calls `upgradeEncoding()` on the stored hash after a successful authentication, and when your `UserDetailsService` also implements `UserDetailsPasswordService`, the provider re-encodes the just-verified password with the current default encoder and calls `updatePassword(...)` to persist it:

```java
@Service
public class AppUserDetailsService implements UserDetailsService, UserDetailsPasswordService {
    // loadUserByUsername(...) as usual

    @Override
    public UserDetails updatePassword(UserDetails user, String newPassword) {
        // newPassword arrives already encoded with the current default encoder
        userRepository.updatePasswordHash(user.getUsername(), newPassword);
        return user;
    }
}
```

Active users migrate to Argon2id transparently, one login at a time; the stragglers still on `{bcrypt}` after a year are accounts that never log in — which is its own useful signal. Argon2id is the target because it's memory-hard and GPU-resistant in a way bcrypt isn't; bcrypt stays in the map only to verify the old hashes.

## Refresh tokens: short access, rotating refresh

The token lifetime pattern that survives contact with production:

- **Access token:** a JWT, short-lived (5–15 minutes), sent as a bearer token. Stateless validation is the whole point — don't look it up in a database, or you've rebuilt sessions with extra steps.
- **Refresh token:** long-lived, opaque, stored server-side, delivered in an `HttpOnly`, `Secure`, `SameSite=Strict` cookie scoped with `Path=/auth` — never in `localStorage`, where any XSS reads it.
- **Rotation:** every refresh issues a *new* refresh token and invalidates the old one. If an old refresh token is ever presented again, that's reuse — treat it as theft and invalidate the whole token family, forcing re-login. Rotation without reuse detection is just longer sessions.

This part is a pattern, not a framework feature — Spring validates the access token; the refresh lifecycle is yours to implement. Keep it boring: a token table with family id, expiry, and a used flag is enough.

**The CSRF consequence of that cookie.** A cookie is ambient authentication — the browser attaches it automatically, which is exactly what CSRF exploits. So the "CSRF disabled, no cookies" config above needs scoping, and now it has it: CSRF stays off on the bearer-token API chain, where no cookie is ever read and there's nothing to forge. The refresh endpoint is different — it *is* cookie-authenticated, so it keeps CSRF protection, and the cookie itself is hardened (`SameSite=Strict`, `Path=/auth`) so cross-site pages can't even get it sent. Skip both, and any malicious page can ride a logged-in browser into minting itself fresh access tokens.

## Tests: MockMvc's `jwt()` post-processor

Security config without tests is a rumor. `spring-security-test` gives you a `jwt()` request post-processor that builds the authenticated state without real tokens:

```java
import org.springframework.boot.webmvc.test.autoconfigure.WebMvcTest;
import org.springframework.security.oauth2.jwt.JwtDecoder;
import org.springframework.test.context.bean.override.mockito.MockitoBean;

import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.jwt;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@WebMvcTest(OrderController.class)
@Import(SecurityConfig.class)
class OrderSecurityTest {

    // The slice has no issuer-uri property, so no JwtDecoder is auto-configured —
    // and oauth2ResourceServer().jwt() won't start without one. The jwt()
    // post-processor below bypasses decoding, but the bean must exist.
    @MockitoBean
    JwtDecoder jwtDecoder;

    @Autowired MockMvc mvc;

    @Test
    void anonymousGets401() throws Exception {
        mvc.perform(post("/api/orders"))
           .andExpect(status().isUnauthorized());
    }

    @Test
    void wrongScopeGets403() throws Exception {
        mvc.perform(post("/api/orders")
               .with(jwt().authorities(new SimpleGrantedAuthority("SCOPE_orders:read"))))
           .andExpect(status().isForbidden());
    }

    @Test
    void rightScopePasses() throws Exception {
        mvc.perform(post("/api/orders")
               .contentType(MediaType.APPLICATION_JSON).content("{}")
               .with(jwt().authorities(new SimpleGrantedAuthority("SCOPE_orders:write"))))
           .andExpect(status().isAccepted());
    }
}
```

The 401/403 split is the test that matters: anonymous is *unauthenticated* (401), wrong-scope is *authenticated but not allowed* (403). If your tests can't tell them apart, neither can your debugging at 2 a.m.

## The 2-minute interview answer

> "On Spring Boot 4, JWT auth is the built-in OAuth2 Resource Server — no custom filters, no third-party JWT library. One SecurityFilterChain in the lambda DSL (mandatory in Security 7; the `.and()` style and `authorizeRequests` are removed), stateless sessions, and `oauth2ResourceServer().jwt()` validating against the issuer's JWK set. Scopes become `SCOPE_`-prefixed authorities, and custom claims need an explicit `JwtAuthenticationConverter` — that's the classic 403 trap. Passwords go through a `DelegatingPasswordEncoder` so I can move bcrypt to Argon2id lazily — `DaoAuthenticationProvider` re-encodes on each successful login when the `UserDetailsService` implements `UserDetailsPasswordService`, no reset flag day. Access tokens are short-lived JWTs; refresh tokens are opaque, HttpOnly-cookie, rotated on every use with reuse detection invalidating the family. And I test it with MockMvc's `jwt()` post-processor — anonymous gets 401, wrong scope gets 403."

## Cheat sheet

| Piece | The Security 7 way |
|---|---|
| JWT validation | Built-in OAuth2 Resource Server — no custom filter |
| Config style | Lambda DSL only (`.and()` removed) |
| Authorization | `authorizeHttpRequests` + `PathPatternRequestMatcher` |
| Issuer config | `spring.security.oauth2.resourceserver.jwt.issuer-uri` |
| Scopes | Become `SCOPE_`-prefixed authorities |
| Custom claims | Explicit `JwtAuthenticationConverter` |
| Password hashing | `DelegatingPasswordEncoder`, Argon2id default, bcrypt for legacy verify |
| Argon2 dependency | `org.bouncycastle:bcprov-jdk18on` — not transitive; missing it = `NoClassDefFoundError` |
| Hash upgrades | Built-in: `DaoAuthenticationProvider` + `UserDetailsPasswordService` re-encode on login |
| Audience | Not validated from `issuer-uri` alone — add a validator on a `JwtDecoder` bean |
| Access token | Short-lived JWT, stateless, bearer |
| Refresh token | Opaque, HttpOnly cookie, rotate on use, reuse = kill the family |
| Tests | MockMvc `jwt()` post-processor; assert 401 vs 403 separately |
| Removed | Password grant, `authorizeRequests`, Ant/Mvc matchers |

Delete the custom filter. The framework grew up.
