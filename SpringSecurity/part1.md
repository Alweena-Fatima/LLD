# Spring Security: Notes

## 1. What is Spring Security?
A framework that puts a security layer in front of your Spring application. It works as a **chain of filters** that every request passes through *before* reaching your controller.

It handles:
- Authentication (who are you?)
- Authorization (what can you do?)
- Password encoding
- Session management
- JWT / token support
- Protection against common attacks (CSRF, session fixation, etc.)

---

## 2. Authentication vs Authorization

| | Authentication | Authorization |
|---|---|---|
| **Question** | Who are you? | What are you allowed to do? |
| **Purpose** | Verify the identity of a user | Check the permissions of a verified user |
| **Based on** | Username/password, OTP, token | Roles / permissions (USER, MANAGER, ADMIN) |
| **Order** | Happens first | Happens after authentication |
| **Failure** | `401 Unauthorized` | `403 Forbidden` |

**Example roles**
- `USER`: view orders
- `MANAGER`: view + update orders
- `ADMIN`: view + update + delete orders

> Being logged in does **not** mean you can access everything.

### 401 vs 403
- **401 Unauthorized**: no valid authentication (not logged in, wrong password, invalid/expired token). *"Who are you?"*
- **403 Forbidden**: authenticated, but not permitted (e.g. a `ROLE_USER` calling a `DELETE` endpoint that needs `ADMIN`). *"I know you, but you can't do this."*

---

## 3. What is a vulnerability?
A weakness an attacker can exploit to access or modify data, bypass checks, or steal information.

**Example (broken access control):**
`GET /users/{id}` only checks that you are logged in, but not that `{id}` belongs to you. Any logged-in user can read other users' data.

---

## 4. Adding Spring Security

**Maven**
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

**Gradle**
```groovy
implementation 'org.springframework.boot:spring-boot-starter-security'
```

---

## 5. A Simple Example

### Without Spring Security
```java
@RestController
@RequestMapping("/user")
public class UserController {

    @GetMapping("/get")
    public String getUser() {
        return "User data";
    }
}
```

`GET http://localhost:8080/user/get` returns:
```
User data
```

### After adding the dependency
No config, no login controller, no users created, yet the endpoint is now protected. Spring Boot auto-configures:
- A default login mechanism (form login and HTTP Basic) <img width="1100" height="282" alt="image" src="https://github.com/user-attachments/assets/28e2a8c0-7c33-42da-9fc7-664934aa6785" />

- A default user named `user`
- A random password, **regenerated on every application start**, printed in the logs: <img width="568" height="160" alt="image" src="https://github.com/user-attachments/assets/91b01494-fba7-4e93-99de-b1b3703bc2a9" />

  ```
  Using generated security password: <random-uuid>
  ```

After logging in with `user` and that password, `GET /user/get` returns `User data` again.

### Override the default credentials
In `application.properties`:
```properties
spring.security.user.name=<name>
spring.security.user.password=<password>
```
> Fine for learning and testing. For real apps, use a proper `UserDetailsService` with a database and encoded passwords.

---

## 6. What just happened?
The request no longer goes straight to the controller. It passes through the **Security Filter Chain** first:

```
Client
  ↓  GET /user/get
Spring Security Filter Chain
  ↓  Authentication  (who are you?)
  ↓  Authorization   (are you allowed?)
UserController
  ↓
"User data"
```

**Key idea:** Spring Security acts *before* the request reaches your controller. If authentication or authorization fails, the controller is never called.

---

## Quick recap
1. Add `spring-boot-starter-security` → all endpoints are secured by default.
2. Authentication comes first (401 on failure), authorization second (403 on failure).
3. Default user is `user`, with a new random password each startup (see logs).
4. Override with `spring.security.user.name` / `spring.security.user.password`.
5. Everything is enforced by the filter chain, before your code runs.
