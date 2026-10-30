# Hashing User Password

When users create an account, we should **never** store their plain-text password directly in the database.

For example, we should NOT store:

```
Username: Meow
Password: hello123
```

Instead, we store a password hash:

```
Username: Meow
Password: $2a$10$....
```

This process is called **password hashing**.

```
Plain Password
      ↓
 Hashing Algorithm
      ↓
Password Hash
      ↓
Database
```

If the database is compromised, the attacker should not immediately obtain the original passwords.

---

## Two Types of Hashing Algorithms

Hashing algorithms can broadly be thought of as:

1. Fast Hashing Algorithms
2. Slow / Password Hashing Algorithms

The important difference is how quickly they can calculate a hash.

---

## 1. Fast Hashing Algorithms

Fast hashing algorithms are designed to calculate a hash very quickly.

Examples include:

- MD5
- SHA-1
- SHA-256
- SHA-512

For example:

```
"hello123"
     ↓
 SHA-256
     ↓
Hash
```

The same input produces the same hash:

```
hello123
   ↓
SHA-256
   ↓
<hash>
```

These algorithms are extremely useful for many purposes, such as:

- File integrity
- Checksums
- Data integrity
- Digital signatures
- General cryptographic operations

However, **fast hashing is not suitable for storing user passwords by itself.**

### Why?

Imagine an attacker gets our database:

```
Username    Password Hash
-------------------------------
Meow        <hash>
Tom         <hash>
Jerry       <hash>
```

Because SHA-256 is very fast, an attacker can try a huge number of possible passwords very quickly.

Conceptually:

```
Password 1 → SHA-256 → Compare
Password 2 → SHA-256 → Compare
Password 3 → SHA-256 → Compare
Password 4 → SHA-256 → Compare
       ...
```

The attacker can perform a large number of guesses in a short amount of time. Therefore:

> Fast hashing algorithms are too fast for password storage.

---

## 2. Slow Password Hashing Algorithms

Password hashing algorithms are intentionally designed to be computationally expensive.

Examples include:

- bcrypt
- scrypt
- Argon2
- PBKDF2

Instead of calculating a password hash extremely quickly, these algorithms deliberately require more computational work.

Conceptually:

```
Password
   ↓
Slow Password Hashing Algorithm
   ↓
Password Hash
```

The goal is to make password guessing more expensive for an attacker.

For example:

```
Guess 1 → expensive
Guess 2 → expensive
Guess 3 → expensive
Guess 4 → expensive
...
```

If an attacker needs to try millions of possible passwords, making each attempt expensive significantly increases the cost of the attack.

---

## Why Do We Want Password Hashing to Be Slow?

This may seem strange. Normally, when designing software, we want algorithms to be **fast**. But password hashing is a special case.

We want legitimate authentication to be reasonably fast:

```
User Login
    ↓
Password Hash Verification
    ↓
Small delay
    ↓
Login
```

But we want millions or billions of password guesses to become expensive:

```
Attacker
   ↓
Millions of guesses
   ↓
Each guess requires expensive computation
   ↓
Attack becomes much more costly
```

So the basic idea is:

```
Fast hashing
→ Good for many general cryptographic tasks

Slow password hashing
→ Designed specifically to make password guessing expensive
```

---

## Example: bcrypt

One commonly used password hashing algorithm is **bcrypt**.

For example:

```
Password:
        hello123
        ↓
        bcrypt
        ↓
$2a$10$................................
```

The database stores the resulting hash rather than `hello123`.

Notice that the stored bcrypt string contains information such as the algorithm/version and cost parameters, along with the salt and resulting hash.

---

## Encryption vs Hashing

### Encryption

Encryption is designed to be **reversible**:

```
Plain Text
    ↓
Encryption
    ↓
Cipher Text
    ↓
Decryption
    ↓
Plain Text
```

If you have the appropriate key, you can decrypt the data.

### Hashing

Hashing is designed as a **one-way** operation:

```
Password
    ↓
Hash Function
    ↓
Password Hash
```

We don't normally "decrypt" a password hash to obtain the original password. Instead, during login, the submitted password is verified against the stored hash.

---

## What Happens During Login?

Suppose the user originally created an account with:

```
Password:
hello123
```

We hash the password and store:

```
$2a$10$....
```

Later, the user logs in:

```
Username: Meow
Password: hello123
```

Spring Security's password encoder verifies the submitted password against the stored hash.

Conceptually:

```
Entered Password
      ↓
PasswordEncoder
      ↓
Compare with Stored Hash
      ↓
Match?
   ↙      ↘
 YES       NO
  ↓         ↓
Success   Failure
```

We do not need to retrieve the original password from the database.

Modern password-hashing algorithms such as bcrypt and Argon2 handle the salt as part of the password-hashing process, so developers generally don't need to manually create and manage a separate salt column.

---

## Spring Security `PasswordEncoder`

Spring Security provides:

```java
PasswordEncoder
```

We can configure an appropriate password encoder, for example:

```java
PasswordEncoder passwordEncoder =
        new BCryptPasswordEncoder();
```
<img width="1000" height="330" alt="image" src="https://github.com/user-attachments/assets/217c5305-460c-4d20-833f-372b60723fac" />


When creating a user:

```java
String hash =
        passwordEncoder.encode("hello123");
```
<img width="902" height="58" alt="image" src="https://github.com/user-attachments/assets/10bdc0c5-7684-467b-a1d8-3c13602f28b7" />

We store `hash` in the database rather than `hello123`.

During login, Spring Security can then use:

```java
passwordEncoder.matches(
    enteredPassword,
    storedHash
);
```

Conceptually:

```
Registration
────────────
hello123
   ↓
PasswordEncoder
   ↓
Password Hash
   ↓
Database
```

```
Login
─────
hello123
   ↓
PasswordEncoder
   ↓
Compare with stored hash
   ↓
Match → Authentication Successful
```

---

## Why Do We Need Salt?

Another important concept in password hashing is **salt**.

A salt is a random value added to a password before/during password hashing. Suppose two users have the same password:

```
Meow  → hello123
Tom   → hello123
```

If we used a simple unsalted deterministic hash, both could produce the same hash.

With password hashing schemes such as bcrypt, a unique random salt is used.

Conceptually:

```
hello123 + Random Salt
          ↓
       bcrypt
          ↓
      Hash A
```

and:

```
hello123 + Different Random Salt
          ↓
       bcrypt
          ↓
      Hash B
```

Therefore:

```
Same Password
      ↓
Different Salt
      ↓
Different Hash
```

This makes precomputed attacks such as **rainbow tables** much less useful.
