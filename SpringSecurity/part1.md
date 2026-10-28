# Spring Security: Simple Notes

## What it is
- A framework that adds a security layer in front of a Spring app.
- Works as a chain of filters handling: authentication, authorization, password security, session management, JWT support, and protection against common attacks (e.g. CSRF).

## Authentication vs Authorization
| | Authentication | Authorization |
|---|---|---|
| Question | Who are you? - Authentication is the process of verifying the identity of a user. | What are you allowed to do? |
| Checks | Username/password, OTP, token | Roles/permissions (USER, MANAGER, ADMIN) |
| Happens | First | After authentication |
| Failure | 401 Unauthorized | 403 Forbidden |

- Example roles: USER = view orders; MANAGER = view + update; ADMIN = view + update + delete.
- Being logged in does NOT mean you can access everything.

## 401 vs 403
- **401**: no valid authentication (no login, wrong password, invalid/expired token). "Who are you?"
- **403**: authenticated but not permitted (e.g. ROLE_USER trying DELETE that needs ADMIN). "I know you, but you can't do this."

## Vulnerability
- A weakness an attacker can exploit (access/modify data, bypass checks, steal info).
- Example: `GET /users/{id}` only checks that you're logged in, not that user `id` is you. Anyone could read other users' data.

## Add Spring Security
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

If you run your application, in your run console you can see this application generates a default password and the username by default is ‘user’.
<img width="1247" height="119" alt="image" src="https://github.com/user-attachments/assets/7dcc6dee-ac8e-4711-ae62-9e30ce744608" />

<img width="496" height="317" alt="image" src="https://github.com/user-attachments/assets/87259901-69a0-4e36-9e13-482096b89957" />

If you put in the wrong username or password you will get a “Bad Credentials” error in your browser.



## Internal working (request flow)
1. **SecurityFilterAutoConfiguration**: auto-registers the `DelegatingFilterProxy` as `springSecurityFilterChain`.
2. **DelegatingFilterProxy**: intercepts incoming HTTP requests, hands them to FilterChainProxy.
3. **FilterChainProxy**: decides which security filters apply to the current request.
4. **SecurityFilterChain**: the ordered list of filters (authentication, authorization, etc.) that run.

Mall analogy: DelegatingFilterProxy = main gate, filters = checkpoints, VIP lounge = restricted resources.

<img width="400" height="400" alt="image" src="https://github.com/user-attachments/assets/0b7d709b-d479-41f3-af5a-5c81a48cb27d" />


## Default behavior
- **Authentication**: form login with a basic login page; HTTP Basic for REST APIs.
- **User**: in-memory user `user` with a generated password (shown in console on startup).
- **Authorization**: all requests secured; only login, logout, error pages open.
- **CSRF**: enabled for state-changing requests (hidden `_csrf` token in login form).
- **Sessions**: session fixation protection; cookie `JSESSIONID` until logout.
- **Passwords**: encoded with BCrypt.
- **Logout**: `/logout` invalidates session and redirects to login.
- **Access denied**: 403 error or access-denied page.

## Trying the defaults
1. Create a Maven/Spring Boot app with `spring-boot-starter-security`.
2. Run it and copy the generated password from the console (username: `user`).
3. Open any GET endpoint in the browser; a sign-in form appears.
4. Wrong credentials give "Bad credentials".
5. After login, a `JSESSIONID` cookie is set (DevTools > Application > Cookies).
6. Visit `/logout` and confirm to end the session.

## Custom username/password (application.properties)
```properties
spring.security.user.name=<name>
spring.security.user.password=<password>
```
