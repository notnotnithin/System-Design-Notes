# JWT (JSON Web Token) Explained (HLD #26 – Lecture 63)

<!-- nav -->
⬅ [25 Symmetric and Asymmetric Encryption](25_Symmetric_and_Asymmetric_Encryption_Notes.md) · [Index](00_HLD_Index.md) · [27 Thundering Herd](27_Thundering_Herd_Notes.md) ➡
<!-- nav -->

Sources:
- PDF: https://drive.google.com/file/d/1jURIUs6grcQBkXl_mnx3nvZDlpHAqof8/view?usp=drive_link
- Zoho: https://notebook.zohopublic.in/public/notes/bietvcd2cffdd8c7d449c839c007fecf8d562

---

## 1. What is JWT?
- JWT provides **a secure way of transmitting information between parties as a JSON object.**
- The information can be **verified because it's digitally signed** – using **RSA (public/private key pair)** etc.

### Advantages
- **Compact** – because of its size it can be sent inside an **HTTP header** itself and its transmission is fast.
- **Self-contained / stateless** – the **payload contains all the required information about the user, so it avoids querying the database.**
- Can be signed using **symmetric (HMAC)** or **asymmetric (RSA)**.
- **Built-in expiry mechanism.**
- **Custom claims** (additional data, e.g. roles) can be added in the JWT.

<!-- isw:26-0 -->
**In simple words**

JWT gives **a secure way of transmitting information between parties as a JSON object.** The information can be **verified because it is digitally signed**, using **RSA (a public/private key pair)** and similar methods.
<!-- /isw:26-0 -->

<!-- isw:26-1 -->
**In simple words**

- **Compact:** because of its size it can travel inside an **HTTP header** and is fast to transmit.
- **Self-contained / stateless:** **the payload holds all the needed information about the user, so a database query is avoided.**
- It can be signed with **symmetric (HMAC) or asymmetric (RSA)** keys.
- It has a **built-in expiry mechanism.**
- **Custom claims** (extra data, for example roles) can be added.
<!-- /isw:26-1 -->





## 2. Where is JWT used?
1. **Authenticating** (confirming the user's identity).
2. **Authorization** (checks the user's permission before giving access to resources).
3. **SSO – Single Sign-On** (authenticate once, access multiple applications).

### Stateless flow

<!-- diagram:26-0 -->
**JWT login and request flow**

![Diagram](diagrams/26_JWT_Explained_Notes-01.png)
<!-- /diagram:26-0 -->


```
 1. Client ── login with username & password ──► Authentication Server (a third-party app that supports token generation and validation)
 2. Auth Server ── generates JWT and returns it ──► Client
 3. Client ── GET /details + JWT in the Authorization header ──► Resource Server (your backend code)
        Resource Server internally calls the Auth Server's endpoints – verify(token) – to validate various parts of the token
 4. Resource Server authenticates the user using the token and sends the response to the client
```

<!-- isw:26-2 -->
**In simple words**

Three uses: **(1) authenticating** (confirming the user's identity), **(2) authorization** (checking the user's permission before giving access to resources) and **(3) SSO, Single Sign-On** (authenticate once, access several applications).
<!-- /isw:26-2 -->

<!-- isw:26-3 -->
**In simple words**

(1) The **client logs in with username and password** to the **authentication server** (a third-party app that supports token generation and validation). (2) The authentication server **generates a JWT and returns it** to the client. (3) The client sends **`GET /details` with the JWT in the Authorization header** to the **resource server** (your backend code); the resource server internally calls the auth server's endpoints, `verify(token)`, to validate the parts of the token. (4) The resource server **authenticates the user using the token** and sends the response.
<!-- /isw:26-3 -->





---

## 3. What was used before JWT? – Session ID (JSESSIONID)

<!-- diagram:26-1 -->
**Session ID flow (stateful)**

![Diagram](diagrams/26_JWT_Explained_Notes-02.png)
<!-- /diagram:26-1 -->


*The Authorization server gives back to the client a unique random key (Session ID); the server keeps the session data – expiry time, user ID, roles, user details – i.e. it is **stateful**.*

Flow:
1. Login with username and password → Authorization Server.
2. **Create SessionID and save it in DB** with other user info needed for auth.
3. **Return SessionID in cookies.**
4. Client **sends requests with cookies (SessionID)** to the Resource Server.
5. Resource Server **queries the DB to look up the Session ID and does validation.**
6. Send the response.

**Disadvantages**
- **Stateful** – relies on **server-side state management**; causes problems in **distributed systems.**
- The session id is **just a unique random value** – the server has to **perform a DB query to fetch the details.**
- **Extra latency** because of the DB query.

<!-- isw:26-4 -->
**In simple words**

Before JWT there was the **session ID (JSESSIONID):** the authorization server gives the client a unique random key (the session ID), and **the server keeps the session data (expiry time, user ID, roles, user details)**, so it is **stateful.**

**Flow:** (1) login with username and password to the authorization server; (2) **a session ID is created and saved in the DB** with the other user information needed for auth; (3) **the session ID is returned in a cookie;** (4) the client **sends requests with the cookie** to the resource server; (5) the resource server **queries the DB to look up the session ID and validates;** (6) the response is sent.

**Disadvantages:** it is **stateful and relies on server-side state management,** which causes problems in **distributed systems;** the session ID is **just a unique random value,** so the server has to **query the DB to fetch the details;** and the DB query adds **extra latency.**
<!-- /isw:26-4 -->


---

## 4. JWT structure  *(JWT / JWS / JWE)*

<!-- diagram:26-2 -->
**JWT = header . payload . signature**

![Diagram](diagrams/26_JWT_Explained_Notes-03.png)
<!-- /diagram:26-2 -->


`aaaaaa.bbbbbbbbb.ccccccc` = **① Header . ② Payload . ③ Signature**

### ① Header – metadata about the token
```json
{ "typ": "JWT", "alg": "RSA", "kid": "sfdsf234234" }
```
- `typ` – type of token (generally JWT, always added here).
- `alg` – **signing algorithm** used, like **RSA (asymmetric key)** or **HMAC (symmetric key)**.
- `kid` – key id (used to find the right public key).

<!-- isw:26-6 -->
**In simple words**

The header holds **metadata about the token:** `typ` (the type of token, generally JWT), `alg` (the **signing algorithm** used, like **RSA (asymmetric key)** or **HMAC (symmetric key)**) and `kid` (a key id, used to find the right public key).
<!-- /isw:26-6 -->


### ② Payload – the claims
(User information or any additional information is kept here.) Example:
```json
{ "iss": "https://example.com/auth", "sub": "1234567890", "exp": 1711945509,
  "email": "conceptcoding@xyz.com", "jti": "unique_id_12312312" }
```
**Claims are of 3 kinds**
1. **Registered claims** (predefined name and meaning):
   - `iss` (Issuer) – entity that issued the JWT.
   - `sub` (Subject) – identifies the user.
   - `aud` (Audience) – identifies the recipient the token is intended for.
   - `exp` (Expiration time) – after this time the token is not valid.
   - `nbf` (Not Before) – before this time the token should not be accepted.
   - `iat` (Issued At) – time at which the token was issued.
   - `jti` (JWT ID) – unique JWT id.
2. **Public claims** – custom claims that can be **shared and understood by multiple parties** (e.g. email `abc@gmail.com`, roles).
3. **Private claims** – custom claims **intended for internal use only**, not standardised, and other parties are not expected to understand them.

IAM note: the **auth server is the IAM (Identity & Access Management)**; resource servers 1 and 2 don't do IAM – they just receive the JWT from the user and trust the signature.

<!-- isw:26-7 -->
**In simple words**

The payload holds the **claims** (user information or any additional information), for example `iss`, `sub`, `exp`, `email`, `jti`. Claims are of **3 kinds:**
1. **Registered claims** (predefined name and meaning): `iss` (issuer, the entity that issued the JWT), `sub` (subject, identifies the user), `aud` (audience, the recipient the token is meant for), `exp` (expiration time, after which the token is not valid), `nbf` (not before, the token must not be accepted before this time), `iat` (issued at) and `jti` (a unique JWT id).
2. **Public claims:** custom claims that can be **shared and understood by several parties** (for example email, roles).
3. **Private claims:** custom claims **only for internal use,** not standardised; other parties are not expected to understand them.

The IAM note: **the auth server is the IAM (Identity & Access Management);** resource servers 1 and 2 do not do IAM, they just receive the JWT from the user and trust the signature.
<!-- /isw:26-7 -->


### ③ Signature – **JWS (JSON Web Signature)**
1. **Base64-encode the JWT header and payload separately.**
2. **Concatenate** the encoded header and payload with `"."` (e.g. `xxxxxx.yyyyyy`) – this is known as the **message**.
3. Use **RSA (asymmetric) or HMAC (symmetric)** to create the **digital signature** of that message:
   - **RSA:** sign with the **private key**, verify with the **public key** (`Sign(Message, PrivateKey)`, `Verify(Message, PublicKey)`).
   - **HMAC:** a single **secret key** both signs and verifies.
   - e.g. `RSASHA256(Base64URL(header) + "." + Base64URL(payload), key)`
4. **Base64-encode the signature** and **concatenate using "."**.

Final token: `Base64(Header) . Base64(Payload) . Base64(Signature)`

<!-- isw:26-8 -->
**In simple words**

The steps: (1) **Base64-encode the header and the payload separately.** (2) **Join** them with a `"."` (for example `xxxxxx.yyyyyy`); this is the **message.** (3) Use **RSA (asymmetric) or HMAC (symmetric)** to create the **digital signature** of the message: with **RSA** you sign with the **private key** and verify with the **public key**; with **HMAC** a single **secret key** both signs and verifies (for example `RSASHA256(Base64URL(header) + "." + Base64URL(payload), key)`). (4) **Base64-encode the signature** and join it with a ".". The final token is `Base64(Header) . Base64(Payload) . Base64(Signature)`.
<!-- /isw:26-8 -->


### Dummy JWT sample
```
eyJhbGciOiJIUzI1NiIsInR5cCI6I.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6Ikpva.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV
```
The three dot-separated parts are the Base64 header, payload and signature.

### Sample API request
```
curl --location --request GET "https://exampleHost.com:12345/api/resource" \
     --header "Authorization: Bearer <JWT>"
```

<!-- isw:26-5 -->
**In simple words**

A JWT looks like `aaaaaa.bbbbbbbbb.ccccccc`, which is **① Header . ② Payload . ③ Signature.**
<!-- /isw:26-5 -->

<!-- isw:26-9 -->
**In simple words**

The client sends the token in the header: `Authorization: Bearer <JWT>` (the sample is a `curl` GET to `https://exampleHost.com:12345/api/resource`).
<!-- /isw:26-9 -->





---

## 5. SSO (Single Sign-On) with JWT

<!-- diagram:26-3 -->
**Single sign-on**

![Diagram](diagrams/26_JWT_Explained_Notes-04.png)
<!-- /diagram:26-3 -->


User logs in **once** at the **Authentication Server** → gets a **JWT token** → presents the same JWT to **App1 (www.app1.com), App2, App3** (each app validates it against the auth server / its public key). **Login happens only in one place; the user can sign in to multiple apps.**

<!-- isw:26-10 -->
**In simple words**

The user logs in **once** at the **authentication server,** gets a **JWT,** and presents the **same JWT** to **App1 (www.app1.com), App2 and App3** (each app validates it against the auth server or its public key). **Login happens in only one place, and the user can sign in to several apps.**
<!-- /isw:26-10 -->


---

## 6. Challenges with JWT (it's stateless)
1. **Token invalidation** – say I blacklist one user: how do I invalidate their token **before its expiration?**
   - (a) The server keeps a **list of blacklisted tokens** (DB/cache) and looks it up while validating.
   - (b) **Change the secret key** – but that **invalidates the JWT for all users.**
   - (c) Tokens should be **very short-lived** ⭐.
   - (d) Tokens should be **used only once** ⭐.
2. **JWT is encoded, not encrypted** – so it's less secure (anyone can base64-decode the payload).
   - (a) Use **JWE (JSON Web Encryption)** – i.e. **encrypt the payload part.**
3. **Unsecured JWT with `"alg": "none"`** – such tokens **should be rejected** (the header says no signature is needed).
4. **JWK exploit** – a **public key embedded inside the token** (a `jwk` header with `n` = modulus, `e` = exponent, together forming the public key, plus `kid`) **must not be used to verify the signature** – an attacker could sign with their own private key and ship their own public key in the token; the resource server then "verifies" a forged token.
   ```json
   { "typ": "JWT", "alg": "RSA", "jwk": { "n": "…", "e": "…", "kid": "…" } }
   ```
   - **Solution (5):** use the **`kid` in the header to look up the key at `https://<auth-server-domain>/.well-known/jwks.json`** and find the real **public key** from the auth server (the resource server trusts only keys from the auth server, never from the token).

<!-- isw:26-11 -->
**In simple words**

Because JWT is stateless, there are these challenges:
1. **Token invalidation:** if one user is blacklisted, how do we invalidate their token **before it expires?** Options: (a) the server keeps a **list of blacklisted tokens** (DB / cache) and checks it while validating; (b) **change the secret key**, but that **invalidates the JWT of all users;** (c) tokens should be **very short-lived** ⭐; (d) tokens should be **used only once** ⭐.
2. **JWT is encoded, not encrypted,** so anyone can base64-decode the payload. Solution: **JWE (JSON Web Encryption),** that is **encrypt the payload part.**
3. **Unsecured JWT with `"alg": "none"`:** such tokens **should be rejected** (the header says no signature is needed).
4. **JWK exploit:** a **public key embedded inside the token** (a `jwk` header with `n` = modulus, `e` = exponent and `kid`) **must not be used to verify the signature,** because an attacker could sign with their own private key and put their own public key in the token, and the resource server would "verify" a forged token.
5. **Solution:** use the **`kid` in the header to look up the key at `https://<auth-server-domain>/.well-known/jwks.json`** and get the real **public key** from the auth server (the resource server trusts only keys from the auth server, never keys from the token).
<!-- /isw:26-11 -->


---

## Quick revision
- JWT = **Header.Payload.Signature**, base64url-encoded; signed (HS256 symmetric / RS256 asymmetric); **stateless, compact, has `exp`.**
- Before JWT: **session id** (stateful → DB lookup, hard in distributed systems).
- Claims: **registered** (`iss, sub, aud, exp, nbf, iat, jti`), **public**, **private**.
- Use for **authentication, authorization, SSO.**
- Pitfalls: **can't revoke easily** (blacklist/short life/one-time use/rotate key), **encoded ≠ encrypted** (use JWE), **reject `alg:none`**, **never trust an embedded `jwk`** – fetch keys via `kid` from the auth server's **JWKS endpoint.**
- Related: [OAuth 2.0](24_OAuth_2.0_Notes.md), [Encryption & Digital Signature](25_Symmetric_and_Asymmetric_Encryption_Notes.md), [API Gateway (JWT filter)](40_41_API_Gateway_Part1_Part2_Notes.md).

---

<!-- nav -->
⬅ [25 Symmetric and Asymmetric Encryption](25_Symmetric_and_Asymmetric_Encryption_Notes.md) · [Index](00_HLD_Index.md) · [27 Thundering Herd](27_Thundering_Herd_Notes.md) ➡
<!-- nav -->
