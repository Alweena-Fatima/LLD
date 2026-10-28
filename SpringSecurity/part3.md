# Spring Security Authentication Fundamentals: Core Concepts and Stateful vs Stateless Auth

## 1. Login Flow (Simplified)

```
Client -> Username + Password -> Spring Security -> Find User
       -> Verify Password -> Authentication Successful -> Authenticated User
```

Example request:

```
POST /login
{ "username": "Meow", "password": "mypassword" }
```

---

## 2. Core Concepts

| Concept | Meaning | Example |
|---|---|---|
| **Identity** | *Who* is this entity (user, service, system)? | Username `Meow`, User ID `101` |
| **Credentials** | *Proof* of the claimed identity | Password, API key, certificate, JWT |
| **Principal** | The currently authenticated user/entity (active identity) | `Meow`, often a `UserDetails` object |
| **Authorities** | Permissions granted to the principal (`GrantedAuthority`) | `ROLE_USER`, `READ_PRIVILEGE` |
| **Authentication** | (1) the *process* of verifying identity; (2) the *object* holding the authenticated identity | Principal + Authorities + authenticated flag |
| **Role** | Higher-level grouping of permissions; a `GrantedAuthority` with `ROLE_` prefix | `ROLE_USER`, `ROLE_MANAGER`, `ROLE_ADMIN` |
| **SecurityContext** | Holds the current `Authentication` object | |
| **SecurityContextHolder** | Utility class to access the current `SecurityContext` | |

**Key idea:** username = identity, password = credential proving it.

### Principal
```java
Authentication authentication =
        SecurityContextHolder.getContext().getAuthentication();
Object principal = authentication.getPrincipal();
```
The principal can be a custom `UserDetails` containing username, password, authorities and other user info.

### Authorities vs Roles
- Authorities are fine-grained: `READ_PRIVILEGE`, `WRITE_PRIVILEGE`.
- Roles group permissions, e.g.:
  - `ROLE_USER` -> view products
  - `ROLE_MANAGER` -> view + update
  - `ROLE_ADMIN` -> view + update + delete
- Roles are stored as `GrantedAuthority` with the `ROLE_` prefix by convention.

### Authentication Object
Contains:
- Principal
- Credentials
- Authorities
- Authenticated status

Whenever you check whether a user may access something, you inspect the `Authentication` object and its authorities.
<img width="577" height="335" alt="image" src="https://github.com/user-attachments/assets/d66d0a16-5972-4430-be16-a10da0a2f5f0" />
<img width="382" height="462" alt="image" src="https://github.com/user-attachments/assets/402dd965-c04f-4380-9309-52aa64656d41" />


### SecurityContext and SecurityContextHolder
```java
SecurityContext context = SecurityContextHolder.getContext();
Authentication authentication = context.getAuthentication();
String username = authentication.getName(); // usually the username; depends on setup
```
- By default the context is tied to the **current thread** (`ThreadLocal` strategy).
- When the thread returns to the thread pool, the context is cleared.
- `SecurityContextHolder` gives access to the security info of the current request/thread.

---

## 3. Authentication State

**Problem:** API requests are independent. How does the server know that `/getUser`, `/orders`, `/payment` after `/login` belong to the same user?
Suppose a user logs into an application.
```

Username: user1
Password: ********
```
After successful authentication, the application needs to recognize this user when they make subsequent requests.
For example:
```

POST /login
      ↓
GET /getUser
      ↓
GET /orders
      ↓
POST /payment
```
The important question is:

How does the application know that all these requests belong to the authenticated user?

An API request is independent by itself. The server does not automatically remember previous requests.

So, after the user logs in, there must be some mechanism that allows the server to identify the user on subsequent requests.
Two approaches: **stateful** and **stateless**.

---

## 4. Stateful Authentication (Session-Based)

The server **maintains authentication state** between requests.

**Flow**
1. User logs in with username/password.
2. Server authenticates and creates a session (e.g. `ABC123`).
3. Server stores: `Session ABC123 -> Authentication -> Principal user1, ROLE_USER`.
4. Server sends the session ID in a cookie: `Set-Cookie: JSESSIONID=ABC123`.
5. Browser automatically sends `Cookie: JSESSIONID=ABC123` on later requests.
6. Server looks up session -> Authentication -> identifies `user1` -> authorization -> controller.

**JSESSIONID** is the usual session identifier in Java web apps.

**Note:** session expiration is configurable (e.g. a banking app may use a short inactivity timeout). It does not mean a fixed re-login interval.
<img width="917" height="656" alt="image" src="https://github.com/user-attachments/assets/8d3eac22-a013-41e0-8fba-fb0b32ef0e0c" />

---

## 5. Stateless Authentication (JWT)

The server **does not keep authentication state in a server-side session**.

**Flow**
1. User logs in; server authenticates.
2. Server generates a **JWT** and returns it to the client.
3. Client stores it and sends it with each request: `Authorization: Bearer <JWT>`.
4. Request pipeline: Security Filter -> Validate JWT -> Extract user info -> Create Authentication -> SecurityContext -> Authorization -> Controller.

No session lookup by `JSESSIONID` is needed. Identity and authority info come from the validated token (subject to the app's validation and security rules).

---

## 6. Problem with Stateful Auth in Multi-Server Setups

Production apps run on multiple servers behind a load balancer (for CPU/memory/network capacity and traffic distribution):

```
            Load Balancer
          /      |       \
     Server 1  Server 2  Server 3
```
Why do we need multiple servers?

Because a large application may have thousands or millions of users. A single server may not have enough CPU, memory, or network capacity to handle all the requests.

So, multiple servers are used to distribute the traffic. This is where stateful authentication can create a problem.
**Scenario**
1. `POST /login` goes to **Server 1**, which creates session `ABC123` in its own memory.
2. Next request `GET /orders` (with `JSESSIONID=ABC123`) is routed to **Server 2**.
3. Server 2 has no such session -> cannot identify the user -> asks for login again -> creates a new session.

## 7. Why Stateless Helps

- Every request carries the JWT, so **any server** can handle it. Authorization: Bearer <JWT>
- Each server validates the token independently, with no shared or sticky session storage.
- Easier to scale horizontally.

---

## 8. Quick Comparison

| | Stateful (Session) | Stateless (JWT) |
|---|---|---|
| State stored | Server-side session | Client holds token |
| Client sends | `JSESSIONID` cookie | `Authorization: Bearer <JWT>` |
| Lookup needed | Yes, session store | No, validate token |
| Multi-server scaling | Harder (shared session or sticky sessions) | Easier |

---
