# Spring Security Authentication (https://blog.devops-monk.com/tutorials/spring-security/authentication-manager-provider/)

Authentication answers one question: **"Who is this user, and can we trust them?"** Spring Security answers it by passing a login request through a fixed set of components, each with one small job.

**Running example used in all these notes:** a user types username `Meow` and password `hello123`. Spring Security must check three things:

- Does the user exist?
- Is the password correct?
- Is the account allowed to log in (not locked, not expired, enabled)?

**The whole flow in one view:**

```text
HTTP Request (Meow / hello123)
        |
Security Filter Chain            <- catches the request first
        |
Authentication object            <- wraps username + password (not yet trusted)
        |
AuthenticationManager            <- front door: "who can handle this?"
        |
ProviderManager                  <- the usual implementation; loops over providers
        |
AuthenticationProvider           <- e.g. DaoAuthenticationProvider, does the real work
        |
UserDetailsService               <- loads the user by username
        |
UserDetails                      <- user data in Spring Security's format
        |
PasswordEncoder                  <- compares hello123 with the stored hash
        |
Authenticated Authentication     <- now trusted
        |
SecurityContext                  <- stored for the rest of the request
        |
DispatcherServlet -> Controller
```

**Analogy (airport check-in):** the filter chain is the entrance gate, the AuthenticationManager is the reception desk, the AuthenticationProvider is the officer who checks your passport, the UserDetailsService is the database the officer looks you up in, the PasswordEncoder is the fingerprint scanner, and the SecurityContext is the wristband you wear afterwards.

**Key idea to remember:** each component does exactly one job, and most of them are interfaces. That is why you can swap parts (database users, LDAP users, OTP login) without rewriting the rest.

## The request hits the Security Filter Chain

Every HTTP request passes through a chain of security filters **before** it reaches your controller.

**How it gets wired up in Spring Boot:**

1. `SecurityFilterAutoConfiguration` automatically registers a filter called `DelegatingFilterProxy` (named `springSecurityFilterChain`).
2. When a request arrives, this filter hands it to the `FilterChainProxy` bean.
3. `FilterChainProxy` picks a `SecurityFilterChain` and runs its filters one after another.

**Some filters in the chain (in order):**

| Filter | What it does (simple words) |
| --- | --- |
| WebAsyncManagerIntegrationFilter | Makes security context work with async requests |
| SecurityContextPersistenceFilter | Loads and saves the SecurityContext for the request |
| HeaderWriterFilter | Adds security headers to the response |
| LogoutFilter | Handles logout URLs |
| UsernamePasswordAuthenticationFilter | Reads username and password from a login request and starts authentication |

**Example:** the client posts `Meow` / `hello123` to `/login`. `UsernamePasswordAuthenticationFilter` reads both values, builds an Authentication request, and passes it on to the AuthenticationManager.

**Note:** in your own REST login controller you can skip that filter's work and build the Authentication object yourself (shown in the code section later).

## The Authentication object

`Authentication` is the object that carries login information through the whole process. It is used twice: first as a **request** (not trusted), then as a **result** (trusted).

**What it holds:**

| Part | Meaning | Example |
| --- | --- | --- |
| Principal | Who the user is (username or user object) | `Meow` |
| Credentials | Proof of identity (password) | `hello123` |
| Authorities | What the user may do (roles) | `ROLE_USER` |
| Authenticated flag | Has it been verified? | `false` then `true` |

**Before and after:**

```text
BEFORE:  Principal = Meow, Credentials = hello123, Authorities = none, Authenticated = false
AFTER:   Principal = Meow (UserDetails), Credentials = erased, Authorities = ROLE_USER, Authenticated = true
```

**Code example (request object):**

```java
Authentication request =
    new UsernamePasswordAuthenticationToken("Meow", "hello123");
// request.isAuthenticated() == false at this point
```

**Remember:** an Authentication object with `Authenticated = false` is just a claim ("I say I am Meow"). Only the AuthenticationManager can turn it into a trusted one.

## AuthenticationManager and ProviderManager

`AuthenticationManager` is the main entry point for authentication(means it recieve the authentication request), but it **does not check the password itself**. It passes the request to someone who can.

**Its one important method:**

```java
public interface AuthenticationManager {

    Authentication authenticate(Authentication authentication) throws AuthenticationException;

}

```

Input: an unauthenticated Authentication (e.g., UsernamePasswordAuthenticationToken with username and plaintext password). 
Output: an authenticated Authentication (same type, but with isAuthenticated() = true and populated authorities), or an AuthenticationException if authentication fails.

**ProviderManager** is the main implementation of this interface. The ProviderManager holds a list of AuthenticationProviders and loops through each one until authentication is successful or all have been tried.

1. Take the first provider in the list.
2. Ask it: `supports(thisType)?` (can you handle this kind of login?).
3. If yes, call its `authenticate()`.
4. If it succeeds, stop and return the result.
5. If not, try the next provider. If none succeed, throw an `AuthenticationException`.

**Example:**

```java
// ProviderManager selects providers by type

public boolean supports(Class<?> authentication) {

    return UsernamePasswordAuthenticationToken.class.isAssignableFrom(authentication);

}

```

```text
Request: UsernamePasswordAuthenticationToken(Meow, hello123)

ProviderManager providers = [ DaoAuthenticationProvider, OtpAuthenticationProvider ]

  DaoAuthenticationProvider.supports(UsernamePassword...) -> true  -> authenticate() -> success
  (OtpAuthenticationProvider is never asked)
```


## AuthenticationProvider and DaoAuthenticationProvider

An `AuthenticationProvider` does the **actual authentication work** for one type of login. It is an interface with two methods:

| Method | Purpose |
| --- | --- |
| `authenticate(Authentication)` | Verify the request; return an authenticated Authentication or throw |
| `supports(Class<?>)` | Tell the ProviderManager whether this provider handles this kind of request |

**Several providers can exist side by side:**
```java
public interface AuthenticationProvider {

    Authentication authenticate(Authentication authentication) throws AuthenticationException;

    boolean supports(Class<?> authentication); // which token types can this handle?

}

```
<img width="1122" height="383" alt="image" src="https://github.com/user-attachments/assets/b4a86c7a-277c-464d-bd2b-8fc699d6f71a" />


**Example:** Meow logs in with a password, so the Dao provider handles it. Tom logs in with an OTP code, so the OTP provider handles it. Same manager, different providers.

### DaoAuthenticationProvider

The most common provider for username/password login. "Dao" just means it reads user data from a data source through a `UserDetailsService`.

**What it does, step by step:**

1. Receives the request (`Meow` / `hello123`).
2. Gets the username and calls `UserDetailsService.loadUserByUsername("Meow")`.
3. Receives a `UserDetails` with the **stored password hash**.
4. Uses `PasswordEncoder.matches("hello123", storedHash)` to compare.
5. Checks account status: enabled, not locked, not expired, credentials not expired.
6. If everything passes, builds a **new authenticated** Authentication object and returns it.
7. If the password is wrong, throws an exception (for example `BadCredentialsException`) and access is denied.

**Remember:** the provider is the *coordinator*. It does not store users (UserDetailsService does) and does not hash passwords (PasswordEncoder does).

## UserDetailsService and UserDetails

### UserDetailsService: "find the user"

Its only job is to **load a user by username**. It does **not** check the password.

```java
UserDetails loadUserByUsername(String username) throws UsernameNotFoundException
```

The data can come from anywhere: a database, LDAP, an external service, or memory.

```text
Username = Meow -> UserDetailsService -> data source -> find Meow -> return UserDetails
```

If the user is not found, throw `UsernameNotFoundException`.

**Built-in implementation for tests and small apps:** `InMemoryUserDetailsManager` keeps users in memory.

```java
@Bean
UserDetailsService users(PasswordEncoder encoder) {
    UserDetails user = User.withUsername("Meow")
            .password(encoder.encode("hello123"))
            .roles("USER")
            .build();
    UserDetails admin = User.withUsername("Admin")
            .password(encoder.encode("admin123"))
            .roles("ADMIN")
            .build();
    return new InMemoryUserDetailsManager(user, admin);
}
```

**Real apps** write their own implementation that reads from a repository:

```java
@Service
@RequiredArgsConstructor
public class CustomUserDetailsService implements UserDetailsService {
    private final UserRepo userRepo;

    @Override
    public UserDetails loadUserByUsername(String email) throws UsernameNotFoundException {
        return userRepo.findByEmail(email)
            .orElseThrow(() -> new UsernameNotFoundException("User not found: " + email));
    }
}
```

### UserDetails: 

`UserDetails` is the interface that represents the security information about a user. Spring Security only needs these:

| Method | Gives |
| --- | --- |
| `getUsername()` | Login name |
| `getPassword()` | Stored password **hash** |
| `getAuthorities()` | Roles and permissions |
| `isEnabled()` | Is the account active? |
| `isAccountNonExpired()` | Not expired? |
| `isAccountNonLocked()` | Not locked? |
| `isCredentialsNonExpired()` | Password not expired? |

<img width="642" height="472" alt="image" src="https://github.com/user-attachments/assets/dcca1973-853f-4940-8460-c9aa8f886db3" />

**Why do we need it?** Your own `User` entity may have many fields (id, email, phone, address, createdAt). Spring Security needs only the security ones. `UserDetails` acts as an **adapter** between your data and Spring Security, and keeps your domain model separate from the security model.

```text
Your User entity (id, username, password, email, phone, address)
        |  map only what is needed
UserDetails (username, password, authorities, account status)
        |
Spring Security
```

**Two ways to create UserDetails:**

1. Use the built-in builder: `User.withUsername("Meow").password(hash).roles("USER").build()`. Note that `.roles("USER")` is stored internally as the authority `ROLE_USER`.
2. Make your entity implement `UserDetails` directly (shown in the code section below).

## PasswordEncoder, hashing and salt (https://medium.com/@azurebandage00/spring-security-part-4-9991c5c3dc36)

We now have the user's data. The last question: **is the password the user typed correct?** That is the job of `PasswordEncoder`.

Passwords must **never** be stored as plain text. The database stores a hash, not `hello123`:

```text
User types:       hello123
Database stores:  $2a$10$N9qo8uLOickgx2ZMRZoMye...   (bcrypt hash)
```

**The two methods of the interface:**

| Method | When used | Example |
| --- | --- | --- |
| `encode(raw)` | Sign-up: turn the password into a secure hash to store | `encoder.encode("hello123")` |
| `matches(raw, hash)` | Login: check the typed password against the stored hash | `encoder.matches("hello123", storedHash)` returns `true` |

**Common implementations:** `BCryptPasswordEncoder` (the usual choice), `Pbkdf2PasswordEncoder`, `SCryptPasswordEncoder`.

```java
@Bean
PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}
```

```text
Entered password -> PasswordEncoder.matches() -> stored hash -> match? -> YES: success / NO: failure
```

**Important:** a hash cannot be reversed. The encoder never "decrypts" the database value. It hashes the typed password again and compares the results.

### Why do we need salt?

A **salt** is a random value mixed into the password before hashing. Suppose two users share a password:

```text
Meow -> hello123
Tom  -> hello123
```

With a plain, unsalted hash both rows would store the **same** hash, so an attacker who cracks one cracks both, and precomputed tables (rainbow tables) work.

With bcrypt, each password gets a unique random salt:

```text
hello123 + Salt A -> bcrypt -> Hash A
hello123 + Salt B -> bcrypt -> Hash B   (different!)
```

Same password, different salt, different hash. This makes rainbow tables and "find everyone with the same password" attacks much less useful. bcrypt stores the salt inside the hash string itself, so `matches()` can still verify it.

## SecurityContext and the controller

After a successful check, the authenticated `Authentication` object goes back up the chain: provider, then ProviderManager, then AuthenticationManager.

It is then placed in the **SecurityContext**, which lives inside the **SecurityContextHolder**. From now on, any code in the application can ask "who is the current user?".

```java
Authentication auth = SecurityContextHolder.getContext().getAuthentication();
String username = auth.getName();   // "Meow"
```

Finally the request moves on to the `DispatcherServlet`, which calls your controller. By now Spring Security has confirmed who the user is.

**Example inside a controller:**

```java
@GetMapping("/profile")
public String profile(Authentication authentication) {
    return "Hello " + authentication.getName();   // Hello Meow
}
```

**What if authentication fails?** An `AuthenticationException` is thrown (for example `BadCredentialsException` or `UsernameNotFoundException`), nothing is stored in the SecurityContext, and the client gets a `401 Unauthorized`.

## Complete code example (database login)

This puts every piece together. Here the **email is the username**, and users come from a database. Written for Spring Boot 3 / Spring Security 6.

**1. Entity that implements UserDetails**

```java
@Entity
public class UserEntity implements UserDetails {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true)
    private String email;      // used as the login name
    private String password;   // stores the BCrypt hash, never plain text
    private String name;

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return List.of(new SimpleGrantedAuthority("ROLE_USER"));
    }
    @Override public String getPassword() { return password; }
    @Override public String getUsername() { return email; }   // must match what loadUserByUsername searches

    // getters and setters for id, email, name, password ...
}
```

The other four methods (`isEnabled`, `isAccountNonLocked`, and so on) have default implementations returning `true`, so you only override them when you need them.

**2. Repository**

```java
@Repository
public interface UserRepo extends JpaRepository<UserEntity, Long> {
    Optional<UserEntity> findByEmail(String email);
}
```

**3. UserDetailsService**

```java
@Service
@RequiredArgsConstructor
public class UserServiceImpl implements UserDetailsService {
    private final UserRepo userRepo;

    @Override
    public UserDetails loadUserByUsername(String email) throws UsernameNotFoundException {
        return userRepo.findByEmail(email)
            .orElseThrow(() -> new UsernameNotFoundException("User with email " + email + " not found"));
    }
}
```

**4. Security configuration**

```java
@Configuration
@EnableWebSecurity
public class WebSecurityConfig {

    @Bean
    PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    // Exposes the AuthenticationManager so a controller can use it.
    // Spring Boot auto-builds a DaoAuthenticationProvider from your
    // UserDetailsService and PasswordEncoder beans.
    @Bean
    AuthenticationManager authenticationManager(AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }

    @Bean
    SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.csrf(csrf -> csrf.disable())          // fine for a simple stateless API demo
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/auth/**").permitAll()
                .anyRequest().authenticated());
        return http.build();
    }
}
```

**5. Login controller (uses the AuthenticationManager directly)**

```java
@RestController
@RequestMapping("/auth")
@RequiredArgsConstructor
public class AuthController {
    private final AuthenticationManager authenticationManager;

    @PostMapping("/login")
    public String login(@RequestBody LoginRequest request) {
        Authentication token = new UsernamePasswordAuthenticationToken(
                request.email(), request.password());          // not authenticated yet

        Authentication result = authenticationManager.authenticate(token);  // throws if wrong

        SecurityContextHolder.getContext().setAuthentication(result);       // now trusted
        return "Login successful for " + result.getName();
    }
}

public record LoginRequest(String email, String password) {}
```

**6. Sign-up (this is where `encode()` is used)**

```java
user.setPassword(passwordEncoder.encode(rawPassword));   // store the hash
userRepo.save(user);
```

**Trace of a successful login with `Meow@mail.com` / `hello123`:**

1. Controller builds an unauthenticated token.
2. `authenticationManager.authenticate(token)` runs.
3. ProviderManager asks `DaoAuthenticationProvider.supports(...)`, which says yes.
4. The provider calls `loadUserByUsername("Meow@mail.com")` and gets a `UserEntity`.
5. `passwordEncoder.matches("hello123", user.getPassword())` returns `true`.
6. An authenticated Authentication is returned and stored in the SecurityContext.

**Trace of a failure (wrong password):** step 5 returns `false`, a `BadCredentialsException` is thrown, and the login is rejected.
