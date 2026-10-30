# Configuring SecurityFilterChain

## Introduction

`SecurityFilterChain` is the core component in **Spring Security 6+** that defines how security is applied to incoming HTTP requests. It replaced the old `WebSecurityConfigurerAdapter` approach (deprecated in 5.7, removed in 6).

It is a **bean** that contains an **ordered list of security filters**. Each filter handles one security concern (authentication, authorization, CSRF, sessions, etc.).

```
HTTP Request
     ↓
Filter 1 → Filter 2 → Filter 3 → ... → Filter N
     ↓
DispatcherServlet → Controller
```

Think of it as a pipeline where every request must pass through multiple security checks before it reaches your controller.

---

## Step 1: Add the Dependency

In `pom.xml`:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

> Just adding this dependency secures **every** endpoint with a default login form and a generated password (printed in the console at startup).

---

## Old Way vs New Way

### Old way (deprecated, do NOT use)

```java
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
            .anyRequest().authenticated()
            .and()
            .formLogin();
    }
}
```

### New way (bean based)

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/admin/**").hasRole("ADMIN")
            .anyRequest().authenticated()
        )
        .formLogin(Customizer.withDefaults())
        .httpBasic(Customizer.withDefaults());

    return http.build();
}
```

Why the new way is better: it is more flexible, easier to test, and lets you define **multiple** chains in a modular way.

---

## Step 2: Create a Security Configuration Class

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.Customizer;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.provisioning.InMemoryUserDetailsManager;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity   // tells Spring Boot we want to configure the SecurityFilterChain ourselves
public class WebSecurityConfig {

    @Bean
    SecurityFilterChain securityFilterChain(HttpSecurity httpSecurity) throws Exception {
        httpSecurity
                .authorizeHttpRequests(auth -> auth
                        .requestMatchers("/public/**").permitAll()   // open endpoints
                        .anyRequest().authenticated())               // everything else needs login

                .csrf(csrf -> csrf.disable())   // simplicity only, NOT recommended for production

                .sessionManagement(session -> session
                        .sessionCreationPolicy(SessionCreationPolicy.STATELESS))   // no JSESSIONID

                .formLogin(Customizer.withDefaults())   // default login page
                .logout(Customizer.withDefaults());     // default /logout

        return httpSecurity.build();   // build() throws Exception, so the method declares it
    }

    // Custom users (in-memory)
    @Bean
    UserDetailsService myInMemoryUserDetailsService() {

        UserDetails user = User
                .withUsername("meow")
                .password(passwordEncoder().encode("hello123"))
                .roles("USER")
                .build();

        UserDetails admin = User
                .withUsername("tom")
                .password(passwordEncoder().encode("admin123"))
                .roles("ADMIN")
                .build();

        return new InMemoryUserDetailsManager(user, admin);
    }

    // Password encoding (see the password hashing notes)
    @Bean
    PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

> **Note:** `STATELESS` combined with `formLogin()` is contradictory, because form login relies on a session to remember the user. Use `STATELESS` with JWT or Basic auth, and keep the default session policy for form login.

---

## Explanation of Key Components

| Component | What it does |
|---|---|
| `@EnableWebSecurity` | Switches on web security and lets us supply our own `SecurityFilterChain` |
| `HttpSecurity` | Builder object used to customize security settings |
| `authorizeHttpRequests` | Defines which URLs need which access |
| `csrf` | Configures CSRF protection |
| `sessionManagement` | Configures how sessions are created and used |
| `formLogin` | Enables form-based login |
| `logout` | Enables the logout endpoint (`/logout` by default) |
| `UserDetailsService` | Tells Spring Security where users come from (in-memory here, a database in real apps) |
| `PasswordEncoder` | Hashes and verifies passwords |

---

## Running the Application

With the configuration above:

```
GET /public/hello      → accessible without logging in
GET /orders            → redirected to /login (authentication required)
GET /login             → default login form
```

---

## Key Configuration Areas

### 1. Authorization Rules (who can access what)

| Method | Meaning |
|---|---|
| `permitAll()` | Anyone can access |
| `authenticated()` | Must be logged in |
| `hasRole("ADMIN")` | Requires a specific role |
| `hasAnyRole("USER", "ADMIN")` | Requires any one of several roles |
| `hasAuthority("READ_PRIVILEGE")` | Requires a specific permission |

Example:

```java
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/public/**").permitAll()
    .requestMatchers("/admin/**").hasRole("ADMIN")
    .requestMatchers(HttpMethod.GET, "/products/**").hasAnyRole("USER", "ADMIN")
    .anyRequest().authenticated()
);
```

> Rules are evaluated **top to bottom** and the first match wins, so put specific rules before `anyRequest()`.

### 2. Authentication Mechanisms (how users prove identity)

| Method | Meaning |
|---|---|
| `formLogin()` | Traditional username/password form |
| `httpBasic()` | `Authorization: Basic base64(user:pass)` header |
| `oauth2Login()` | OAuth 2.0 / OpenID Connect login |

```java
.formLogin(Customizer.withDefaults())
.httpBasic(Customizer.withDefaults())
```

### 3. Security Features

| Method | Meaning |
|---|---|
| `csrf()` | Cross-Site Request Forgery protection |
| `cors()` | Cross-Origin Resource Sharing |
| `sessionManagement()` | Session handling strategy |

---

## Common Patterns

### Pattern 1: Web app with a custom login page

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/public/**", "/login").permitAll()
            .requestMatchers("/admin/**").hasRole("ADMIN")
            .anyRequest().authenticated()
        )
        .formLogin(form -> form
            .loginPage("/login")
            .defaultSuccessUrl("/dashboard")
            .permitAll()
        )
        .logout(logout -> logout
            .logoutSuccessUrl("/login?logout")
            .permitAll()
        );

    return http.build();
}
```

### Pattern 2: Stateless REST API

```java
@Bean
public SecurityFilterChain apiSecurityFilterChain(HttpSecurity http) throws Exception {
    http
        .securityMatcher("/api/**")
        .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
        .sessionManagement(session -> session
            .sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .csrf(csrf -> csrf.disable())
        .httpBasic(Customizer.withDefaults());

    return http.build();
}
```

### Pattern 3: Role based REST endpoints

```java
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/api/public/**", "/api/auth/**").permitAll()
    .requestMatchers("/api/admin/**").hasRole("ADMIN")
    .requestMatchers("/api/user/**").hasAnyRole("USER", "ADMIN")
    .anyRequest().authenticated()
);
```

---

## The Filters Inside the Chain

A `SecurityFilterChain` is really 15 to 20 individual filters, each responsible for exactly one concern. Understanding them helps you:

- Know exactly where in the chain a request fails
- Add your own filter in the right position
- Remove filters you don't need
- Debug authentication and authorization problems

### Execution order

```
HTTP Request
     ↓
 1. DisableEncodeUrlFilter                 → stops session ID being added to URLs
 2. WebAsyncManagerIntegrationFilter       → propagates SecurityContext to async threads
 3. SecurityContextHolderFilter            → loads/saves SecurityContext
 4. HeaderWriterFilter                     → writes security headers (HSTS, nosniff...)
 5. CorsFilter                             → handles CORS preflight
 6. CsrfFilter                             → validates CSRF token (POST/PUT/PATCH/DELETE)
 7. LogoutFilter                           → handles /logout
 8. UsernamePasswordAuthenticationFilter   → handles POST /login
 9. DefaultLoginPageGeneratingFilter       → builds the default login page
10. DefaultLogoutPageGeneratingFilter      → builds the default logout page
11. BasicAuthenticationFilter              → handles Authorization: Basic header
12. RequestCacheAwareFilter                → restores the originally requested URL after login
13. SecurityContextHolderAwareRequestFilter→ makes request.isUserInRole() etc. work
14. AnonymousAuthenticationFilter          → sets anonymous user if nobody is logged in
15. SessionManagementFilter                → session fixation, concurrent sessions
16. ExceptionTranslationFilter             → converts security exceptions into 401/403
17. AuthorizationFilter                    → final allow/deny decision
     ↓
DispatcherServlet → Controller
```

> Not all filters are active by default. A filter is added only when you configure its feature, for example `BasicAuthenticationFilter` appears only if you call `httpBasic()`.

> In older Spring Security versions, step 3 was `SecurityContextPersistenceFilter`. In Spring Security 6 it was replaced by `SecurityContextHolderFilter`.

### The most important filters in plain words

**SecurityContextHolderFilter**

```
Before chain:  load SecurityContext from session/request → put in SecurityContextHolder
After chain:   save SecurityContext back → clear SecurityContextHolder
```

**UsernamePasswordAuthenticationFilter** (form login)

```
POST /login {username, password}
        ↓
attemptAuthentication()
        ↓
AuthenticationManager.authenticate()
        ↓
 success → redirect to / (or return 200)
 failure → redirect to /login?error
```

**BasicAuthenticationFilter**

```
Authorization: Basic YWxpY2U6c2VjcmV0
                     ↑ base64("alice:secret")
```

**AnonymousAuthenticationFilter**

If nobody is authenticated by this point, it sets an `AnonymousAuthenticationToken` with `ROLE_ANONYMOUS`, so `getAuthentication()` never returns `null`.

**ExceptionTranslationFilter**

```
AuthenticationException (not logged in)       → AuthenticationEntryPoint → 401 or redirect to login
AccessDeniedException   (logged in, no right) → AccessDeniedHandler      → 403
```

**AuthorizationFilter**

The last filter. It evaluates the rules from `authorizeHttpRequests()`. If access is denied it throws `AccessDeniedException`, which `ExceptionTranslationFilter` turns into a 403.

---

## Customizing Error Responses

```java
http.exceptionHandling(ex -> ex
    .authenticationEntryPoint((request, response, authException) -> {
        response.setStatus(HttpStatus.UNAUTHORIZED.value());
        response.setContentType("application/json");
        response.getWriter().write("{\"error\": \"Authentication required\"}");
    })
    .accessDeniedHandler((request, response, accessDeniedException) -> {
        response.setStatus(HttpStatus.FORBIDDEN.value());
        response.setContentType("application/json");
        response.getWriter().write("{\"error\": \"Access denied\"}");
    })
);
```

## Customizing Logout

```java
http.logout(logout -> logout
    .logoutUrl("/api/auth/logout")
    .logoutSuccessHandler((request, response, auth) ->
        response.setStatus(HttpStatus.OK.value()))
    .invalidateHttpSession(true)
    .clearAuthentication(true)
    .deleteCookies("JSESSIONID")
);
```

---

## Adding Custom Filters

When you add your own filter (for example a JWT filter), you must say **where** in the chain it belongs.

```java
http
    .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)
    // runs BEFORE form login. Use for: JWT, API keys, custom tokens

    .addFilterAfter(myAuditFilter, AuthorizationFilter.class)
    // runs AFTER the access decision. Use for: audit logging

    .addFilterAt(myCustomFilter, BasicAuthenticationFilter.class);
    // takes the SAME position as the given filter. Use for: replacing a built-in filter
```

### Example: custom JWT filter

```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtService jwtService;
    private final UserDetailsService userDetailsService;

    public JwtAuthenticationFilter(JwtService jwtService, UserDetailsService userDetailsService) {
        this.jwtService = jwtService;
        this.userDetailsService = userDetailsService;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {

        String authHeader = request.getHeader("Authorization");

        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);   // no JWT, pass through
            return;
        }

        String token = authHeader.substring(7);
        String username = jwtService.extractUsername(token);

        if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
            UserDetails userDetails = userDetailsService.loadUserByUsername(username);

            if (jwtService.isTokenValid(token, userDetails)) {
                UsernamePasswordAuthenticationToken authToken =
                        new UsernamePasswordAuthenticationToken(
                                userDetails, null, userDetails.getAuthorities());
                authToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                SecurityContextHolder.getContext().setAuthentication(authToken);
            }
        }

        filterChain.doFilter(request, response);
    }
}
```

Register it:

```java
http.addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class);
```

> `OncePerRequestFilter` guarantees the filter runs exactly once per request, even with async or error dispatches.

---

## Multiple SecurityFilterChains

Different URL patterns can have completely different security setups.

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    // Chain 1: REST API (stateless, JWT)
    @Bean
    @Order(1)
    public SecurityFilterChain apiSecurityFilterChain(HttpSecurity http) throws Exception {
        http
            .securityMatcher("/api/**")      // only applies to /api/**
            .csrf(csrf -> csrf.disable())
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .anyRequest().authenticated()
            )
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);
        return http.build();
    }

    // Chain 2: Web UI (stateful, form login)
    @Bean
    @Order(2)
    public SecurityFilterChain webSecurityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**", "/login").permitAll()
                .anyRequest().authenticated()
            )
            .formLogin(Customizer.withDefaults());
        return http.build();
    }
}
```

How requests are routed:

```
GET /api/products     → matches Chain 1 (JWT, stateless)
GET /admin/dashboard  → does not match /api/**, falls to Chain 2 (form login)
```

- `@Order` decides which chain is checked first (lower number = first).
- `securityMatcher()` restricts which requests a chain applies to.
- The chain without a `securityMatcher` should have the **highest** order number, since it matches everything.

---

## Debugging the Filter Chain

### Enable TRACE logging

```yaml
logging:
  level:
    org.springframework.security: TRACE
```

Sample output:

```
TRACE FilterChainProxy                - Securing GET /api/products
TRACE SecurityContextHolderFilter     - Set SecurityContextHolder to empty SecurityContext
TRACE AnonymousAuthenticationFilter   - Set SecurityContextHolder to anonymous SecurityContext
TRACE AuthorizationFilter             - Authorized filter invocation [GET /api/products]
```

### Print the active filters

```java
@Component
public class FilterChainDiagnostics implements ApplicationRunner {

    @Autowired
    private FilterChainProxy filterChainProxy;

    @Override
    public void run(ApplicationArguments args) {
        filterChainProxy.getFilterChains().forEach(chain -> {
            System.out.println("Chain: " + chain);
            chain.getFilters().forEach(f ->
                System.out.println("  → " + f.getClass().getSimpleName()));
        });
    }
}
```

Sample output:

```
  → DisableEncodeUrlFilter
  → WebAsyncManagerIntegrationFilter
  → SecurityContextHolderFilter
  → HeaderWriterFilter
  → CorsFilter
  → CsrfFilter
  → LogoutFilter
  → JwtAuthenticationFilter          ← your custom filter
  → UsernamePasswordAuthenticationFilter
  → AnonymousAuthenticationFilter
  → ExceptionTranslationFilter
  → AuthorizationFilter
```

---

## Customization Tips

- **Custom authentication logic:** create and register an `AuthenticationProvider` bean.
- **In-memory users:** fine for demos or simple APIs where you don't want a database. Use a database-backed `UserDetailsService` in real applications.
- **Password encoding:** always use a proper `PasswordEncoder` (for example `BCryptPasswordEncoder`).
- **CSRF:** disable it only for stateless APIs. For web apps with forms, keep it enabled.
- **CORS:** `.cors(cors -> cors.configurationSource(corsConfigurationSource()))`
- **OAuth2 / OIDC:** `.oauth2Login(oauth -> oauth.defaultSuccessUrl("/home"))`

---

## Quick Summary

- `SecurityFilterChain` replaces `WebSecurityConfigurerAdapter`.
- It is an **ordered list of servlet filters**, each handling one security concern.
- Configure it with a `@Bean` method that takes `HttpSecurity` and **always returns `http.build()`**.
- Main building blocks: `authorizeHttpRequests`, `formLogin` / `httpBasic`, `csrf`, `sessionManagement`, `logout`, `exceptionHandling`.
- Add custom filters with `addFilterBefore()`, `addFilterAfter()` or `addFilterAt()`, extending `OncePerRequestFilter`.
- Use multiple chains with `@Order` and `securityMatcher()` for different parts of the app (API vs web UI).
- Debug with TRACE logging or by inspecting `FilterChainProxy`.

```
Request → [Security Filters] → Authorized? → Controller
                  ↓ no
            401 / 403 response
```
