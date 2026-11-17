# JWT (JSON Web Token) Explained (HLD #26 – Lecture 63)

<!-- nav -->
⬅ [25 Symmetric and Asymmetric Encryption](25_Symmetric_and_Asymmetric_Encryption_Notes.md) · [Index](00_HLD_Index.md) · [27 Thundering Herd](27_Thundering_Herd_Notes.md) ➡
<!-- nav -->

Sources:
- Handwritten PDF (11 pages): https://drive.google.com/file/d/1jURIUs6grcQBkXl_mnx3nvZDlpHAqof8/view?usp=drive_link
- Typed Zoho note (JavaScript-rendered, not readable automatically): https://notebook.zohopublic.in/public/notes/bietvcd2cffdd8c7d449c839c007fecf8d562

---

<!-- simple-explanation -->
## In simple words – JWT (JSON Web Token)

### What is a JWT?
A JWT is a small piece of text that says **"this is who the user is and what they may do"**, and is **digitally signed** so nobody can change it without being noticed. After you log in, the server hands you a JWT. You attach it to every later request, and the server trusts it because the signature checks out.

Why people like it:
- **Compact** – it fits in an HTTP header.
- **Stateless / self-contained** – the token itself carries the user's information, so the server doesn't need to look anything up in a database.
- Has a built-in **expiry time**, and you can add **custom claims** (e.g. the user's role).
- Used for **authentication**, **authorization** and **Single Sign-On (SSO)**.

### What came before: session IDs
Earlier, after login the server created a **session ID**, a random string, saved the user's details in a **database** against it, and gave the ID to the browser in a cookie. On every request the server had to **look up** the session in the database. That is **stateful**: it adds latency and is awkward when you have many servers that all need access to the same session store. JWT removes that lookup because the information travels inside the token.

### The login flow with JWT
1. The client sends username and password to the **authentication server**.
2. It verifies them and returns a **JWT**.
3. The client calls the **resource server** (your API) with `Authorization: Bearer <JWT>`.
4. The resource server validates the token's signature and expiry and, if fine, sends the data.

### The three parts: `Header . Payload . Signature`
Each part is Base64URL-encoded and they are joined by dots, e.g. `aaaa.bbbb.cccc`.
1. **Header** – metadata: token type, the **signing algorithm** (`alg`, e.g. RSA or HMAC) and a **key id** (`kid`) that says which key to use for verification.
2. **Payload** – the **claims** (facts about the user and token):
   - *Registered claims* (standard names): `iss` (who issued it), `sub` (who it is about), `aud` (who it is meant for), `exp` (expiry time), `nbf` (not valid before), `iat` (issued at), `jti` (unique ID).
   - *Public claims* – custom ones that many parties understand (e.g. `email`, `roles`).
   - *Private claims* – custom ones only for your own systems.
3. **Signature** – created by signing `Base64(header) + "." + Base64(payload)`:
   - **HMAC (symmetric):** one secret key signs and verifies.
   - **RSA (asymmetric):** sign with the **private key**; anyone verifies with the **public key**. This is handy because many services can verify tokens without being able to forge them.
   If anyone alters even one character of the header or payload, the signature check fails.

### Single Sign-On (SSO)
The user logs in **once** at the authentication server, gets a JWT, and presents the same token to App 1, App 2 and App 3. Each app just validates the signature, so there is a single login for many applications.

### Pitfalls to know
1. **Hard to revoke:** a token stays valid until `exp`. Options: keep a **blacklist** of revoked tokens, make tokens **very short-lived**, use them **only once**, or change the signing key (which logs out *everyone*).
2. **Encoded ≠ encrypted:** anyone can Base64-decode the payload and read it. Never put secrets in it; use **JWE** (JSON Web Encryption) if the content must be hidden.
3. **`"alg": "none"` attack:** an attacker sets the algorithm to "none" to claim no signature is needed. The server must **reject** unsigned tokens.
4. **Embedded key (JWK) attack:** an attacker puts *their own* public key inside the token, signs it with their own private key, and a careless server "verifies" it with the key from the token. **Never trust a key that comes inside the token.** Use the `kid` to fetch the genuine public key from the auth server's **JWKS endpoint** (`/.well-known/jwks.json`).

Related: [OAuth 2.0](24_OAuth_2.0_Notes.md), [Encryption & Digital Signature](25_Symmetric_and_Asymmetric_Encryption_Notes.md).
<!-- simple-explanation -->


## 1. What is JWT?
- JWT provides **a secure way of transmitting information between parties as a JSON object.**
- The information can be **verified because it's digitally signed** – using **RSA (public/private key pair)** etc.

### Advantages
- **Compact** – because of its size it can be sent inside an **HTTP header** itself and its transmission is fast.
- **Self-contained / stateless** – the **payload contains all the required information about the user, so it avoids querying the database.**
- Can be signed using **symmetric (HMAC)** or **asymmetric (RSA)**.
- **Built-in expiry mechanism.**
- **Custom claims** (additional data, e.g. roles) can be added in the JWT.

## 2. Where is JWT used?
1. **Authenticating** (confirming the user's identity).
2. **Authorization** (checks the user's permission before giving access to resources).
3. **SSO – Single Sign-On** (authenticate once, access multiple applications).

### Stateless flow
```
 1. Client ── login with username & password ──► Authentication Server (a third-party app that supports token generation and validation)
 2. Auth Server ── generates JWT and returns it ──► Client
 3. Client ── GET /details + JWT in the Authorization header ──► Resource Server (your backend code)
        Resource Server internally calls the Auth Server's endpoints – verify(token) – to validate various parts of the token
 4. Resource Server authenticates the user using the token and sends the response to the client
```

---

## 3. What was used before JWT? – Session ID (JSESSIONID)
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

---

## 4. JWT structure  *(JWT / JWS / JWE)*
`aaaaaa.bbbbbbbbb.ccccccc` = **① Header . ② Payload . ③ Signature**

### ① Header – metadata about the token
```json
{ "type": "JWT", "alg": "RSA", "kid": "sfdsf234234" }
```
- `type` – type of token (generally JWT).
- `alg` – **signing algorithm** used, like **RSA (asymmetric key)** or **HMAC (symmetric key)**.
- `kid` – key id (used to find the right public key).

### ② Payload – the claims
(User information or any additional information is kept here.) Example:
```json
{ "iss": "https://example.com/auth", "sub": "1234567890", "exp": 1711945509,
  "email": "abc@gmail.com", "jti": "uniq-id-12312312" }
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

### ③ Signature – **JWS (JSON Web Signature)**
1. **Base64-encode the JWT header and payload separately.**
2. **Concatenate** the encoded header and payload with `"."` (e.g. `xxxxxx.yyyyyy`) – this is known as the **message**.
3. Use **RSA (asymmetric) or HMAC (symmetric)** to create the **digital signature** of that message:
   - **RSA:** sign with the **private key**, verify with the **public key** (`Sign(Message, PrivateKey)`, `Verify(Message, PublicKey)`).
   - **HMAC:** a single **secret key** both signs and verifies.
   - e.g. `RSASHA256(Base64URL(header) + "." + Base64URL(payload), key)`
4. **Base64-encode the signature** and **concatenate using "."**.

Final token: `Base64(Header) . Base64(Payload) . Base64(Signature)`

### Sample API request
```
curl --location --request GET "https://localhost:80/api/resource" \
     --header "Authorization: Bearer abcd.efgh.ijkl"
```

---

## 5. SSO (Single Sign-On) with JWT
User logs in **once** at the **Authentication Server** → gets a **JWT token** → presents the same JWT to **App1 (www.app1.com), App2, App3** (each app validates it against the auth server / its public key). **Login happens only in one place; the user can sign in to multiple apps.**

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
