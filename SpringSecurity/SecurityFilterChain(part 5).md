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

## Anatomy of a SecurityFilterChain Configuration

Let's break a typical configuration into its parts and see what each one does.

```java
@Configuration                                              // (1)
@EnableWebSecurity                                          // (2)
public class SecurityConfig {

    @Bean                                                   // (3)
    public SecurityFilterChain securityFilterChain(         // (4)
            HttpSecurity http) throws Exception {           // (5)

        http                                                // (6)
            .authorizeHttpRequests(auth -> auth             // (7)
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .formLogin(Customizer.withDefaults())           // (8)
            .httpBasic(Customizer.withDefaults())
            .csrf(csrf -> csrf.disable())                   // (9)
            .sessionManagement(sm -> sm                     // (10)
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            );

        return http.build();                                // (11)
    }
}
```

### The outer shell (1 to 5)

| # | Part | Meaning |
|---|---|---|
| 1 | `@Configuration` | Tells Spring this class declares beans |
| 2 | `@EnableWebSecurity` | Turns on Spring Security's web support and lets our config replace the default one |
| 3 | `@Bean` | Registers the returned object in the Spring container, so Spring Security picks it up |
| 4 | `SecurityFilterChain` (return type) | The object we are producing: an ordered list of filters |
| 5 | `HttpSecurity http` | A builder, injected by Spring, used to describe which filters we want. `throws Exception` is needed because the builder methods can throw |

### The builder chain (6 to 10)

`http` is a **builder**. Every method call adds or configures one or more filters:

```
http.authorizeHttpRequests(...)   → configures AuthorizationFilter
http.formLogin(...)               → adds UsernamePasswordAuthenticationFilter (+ login page filters)
http.httpBasic(...)               → adds BasicAuthenticationFilter
http.csrf(...)                    → adds / removes CsrfFilter
http.sessionManagement(...)       → configures SessionManagementFilter
http.logout(...)                  → configures LogoutFilter
http.cors(...)                    → adds CorsFilter
```

Each method takes a **lambda** (`Customizer`). The lambda gives you a configurer object for that feature:

```java
.csrf(csrf -> csrf.disable())
//     ↑ parameter name   ↑ configure that feature using the object
```

If you want the defaults for a feature, pass `Customizer.withDefaults()`:

```java
.formLogin(Customizer.withDefaults())   // default login page, default behaviour
```

### 7. Authorization rules in detail

```java
.authorizeHttpRequests(auth -> auth
    .requestMatchers("/public/**").permitAll()
    .requestMatchers("/admin/**").hasRole("ADMIN")
    .anyRequest().authenticated()
)
```

Each rule has two halves:

```
 requestMatchers("/admin/**")   .hasRole("ADMIN")
 └──── WHICH requests? ────┘    └─── WHO is allowed? ───┘
```

- `requestMatchers(...)` picks requests by URL pattern (and optionally HTTP method, e.g. `HttpMethod.GET`).
- The second half is the access rule: `permitAll()`, `authenticated()`, `hasRole()`, `hasAnyRole()`, `hasAuthority()`, `denyAll()`.
- `anyRequest()` is the catch-all and **must be last**.
- Rules are checked top to bottom and the **first match wins**.

Wrong order (the specific rule is never reached):

```java
.anyRequest().authenticated()
.requestMatchers("/public/**").permitAll()   // never applied, and Spring will actually reject this config
```

### 8. Authentication mechanisms

```java
.formLogin(Customizer.withDefaults())   // HTML form, session based
.httpBasic(Customizer.withDefaults())   // Authorization: Basic header
```

You can enable more than one. Spring Security tries each filter in order until one authenticates the request.

### 9. CSRF

```java
.csrf(csrf -> csrf.disable())
```

CSRF protection is **on by default**. Disable it only for stateless APIs that don't use cookies or sessions. For form based web apps, leave it on.

### 10. Session management

```java
.sessionManagement(sm -> sm
    .sessionCreationPolicy(SessionCreationPolicy.STATELESS))
```

| Policy | Behaviour |
|---|---|
| `ALWAYS` | Always create a session |
| `IF_REQUIRED` | Create one only when needed (default) |
| `NEVER` | Never create one, but use one if it exists |
| `STATELESS` | Never create or use a session (REST/JWT) |

### 11. `http.build()`

```java
return http.build();
```

This is the step that turns all the configuration into the final ordered `SecurityFilterChain`. **Forgetting it or not returning it means no chain is registered.**

### Putting it together: what happens to a request

```
GET /admin/users   (not logged in)
        ↓
SecurityContextHolderFilter     → no context found, empty one set
        ↓
CsrfFilter                      → GET, so no token check (or disabled)
        ↓
BasicAuthenticationFilter       → no Authorization header, skip
        ↓
AnonymousAuthenticationFilter   → sets anonymous user
        ↓
AuthorizationFilter             → rule: /admin/** needs ROLE_ADMIN → DENY
        ↓
ExceptionTranslationFilter      → not authenticated → entry point
        ↓
401 response (or redirect to /login)
```

```
GET /admin/users   (logged in as ADMIN)
        ↓
... filters load the authenticated user ...
        ↓
AuthorizationFilter             → ROLE_ADMIN present → ALLOW
        ↓
Controller → 200 OK
```

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
