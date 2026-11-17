# OAuth 2.0 (HLD #24 – Lecture 61)

<!-- nav -->
⬅ [20 Distributed Transactions](20_Distributed_Transactions_Notes.md) · [Index](00_HLD_Index.md) · [25 Symmetric and Asymmetric Encryption](25_Symmetric_and_Asymmetric_Encryption_Notes.md) ➡
<!-- nav -->

Sources:
- Handwritten PDF (4 pages): https://drive.google.com/file/d/1DTuw_AyH2ByBV8tldT4kha87MXLplXdi/view?usp=drive_link
- Typed Zoho note (not readable automatically – JavaScript-rendered): https://notebook.zohopublic.in/public/notes/bietv949cfd82a5804e0ea1d18400d3ff6fa3

---

<!-- simple-explanation -->
## In simple words – OAuth 2.0

### The problem it solves
You open a new website and click **"Sign in with Google"**. The site now wants some of your Google data (name, email), but you don't want to give that site your **Google password**. OAuth 2.0 lets you **grant limited access to your data without sharing your password**. It stands for **Open Authorization**: it is about *what an app is allowed to do*, not about proving who you are.

### The four players
| Role | Meaning | Example |
|---|---|---|
| **Resource Owner** | The person who owns the data | You (Himanshu) |
| **Client** | The app asking for access | Instagram / the XYZ website |
| **Authorization Server** | Checks your identity, asks for your consent and issues tokens | Google's login server |
| **Resource Server** | Stores the protected data and serves it when shown a valid token | Google's API holding your name, DOB, etc. |

### The key idea: the access token
Instead of your password, the client gets an **access token**, a temporary key that works only for the permissions (**scopes**) you approved and expires after some time. There are several ways ("grant types") to obtain it. The most important is the **Authorization Code Grant**; others are Implicit, Password Credentials, Client Credentials and Refresh Token.

### The Authorization Code flow, step by step
1. **One-time registration:** the client app registers with the Authorization Server and receives a **client ID** (public) and a **client secret** (private).
2. You click **Sign in with Gmail** on the client.
3. The client **redirects your browser to the Authorization Server**.
4. **You log in there and give consent** ("Allow this app to see your email?"). The client never sees your password.
5. The Authorization Server redirects you back to the client with a short-lived **authorization code**.
6. The client's backend sends that code (plus its client ID and secret) to the Authorization Server and **exchanges it for an access token** (and a refresh token).
7. The client calls the Resource Server with the header `Authorization: Bearer <access_token>`.
8. The Resource Server **validates the token**. If valid → returns your data. If not → returns an error.
9. The client shows "sign-in successful" or "failed".

Why a code first and not the token directly? The code travels through the browser (which can be spied on), but it is useless without the client secret, which only the client's backend knows.

### The two main API calls
- **Register:** `POST /register` with the app name and **redirect URIs** (allowed callback URLs) → response gives `client_id` and `client_secret`.
- **Authorize:** `GET /authorize?response_type=code&client_id=…&redirect_uri=…&scope=…&state=…`
  - `scope` lists what access is requested.
  - `state` is a random value that the client checks when you come back, to prevent **CSRF** (forged requests).

### About tokens
- **Bearer token:** whoever "bears" (holds) the token can use it, so always send it over HTTPS in the `Authorization` header.
- The token can be a plain random string or a **JWT** (see [JWT Explained](26_JWT_Explained_Notes.md)).
- It **expires** (usually in seconds or minutes). When it does, the client uses the **refresh token** to get a new one without asking you to log in again.
<!-- simple-explanation -->


## 1. What is OAuth 2.0?
- **OAuth stands for OPEN AUTHORIZATION.**
- It is an **Authorization framework**.
- It **enables secure third-party access to user-protected data.**

**Example – "Sign in with Gmail":** a user on *XYZ website* clicks **"I want to login" → "Sign in Gmail"** (alternatively "Sign in FB"). XYZ talks to the user's **Gmail account** (the user is **already logged in** to Gmail), and afterwards XYZ shows *"Welcome user"*. XYZ never sees the user's Gmail password.

---

## 2. 4 important roles / actors
| Role | Example in the notes |
|---|---|
| **Resource Owner** | the user (e.g. *Himanshu*) who owns the data |
| **Client** | the app that wants access (e.g. *Facebook / Instagram* – "Sign in with Gmail") |
| **Authorization Server** | the server that authenticates the owner and issues tokens (e.g. **Gmail authorization server** – "Authorization Resource") |
| **Resource Hosting Server** | the server holding the protected data (e.g. **Gmail resource server** – holds DOB, name, … of Himanshu) |

---

## 3. Authorization Grant types
*(Mechanism used by clients to obtain an **Access Token**.)*
1. **Authorization Code Grant** ⭐⭐ *(most important part)*
2. **Implicit Grant**
3. **Resource Owner Password Credentials Grant**
4. **Client Credentials Grant**
5. **Refresh Token Grant** ⭐

---

## 4. End-to-end flow using Authorization Code Grant
Actors in the diagram: **Resource Owner** (user), **Client** (e.g. Instagram – "Sign in using Gmail"), **Gmail Authorization Server**, **Gmail Resource Server** (user is already logged in to Gmail).

1. **Registration** – client registers itself with the Authorization Server.
2. Authorization Server returns **Client ID & Secret**.
3. Resource Owner chooses **Sign in using Gmail** on the client.
4. Client **redirects the owner to the Authorization Server**.
5. **Owner authenticates and provides consent** (at the Authorization Server).
6. Authorization Server **provides an Authorization Code** (to the client, via the redirect URI).
7. Client **requests a token using the Authorization Code**.
8. *(page edge cut off in the scan)* Authorization Server returns the **access token** (and refresh token) to the client.
9. *(page edge cut off in the scan)* Client calls the **Resource Server** with the access token to get the owner's data.
10. Resource Server **validates the token** (with the Authorization Server).
11. Result: **valid or invalid**.
12. **If the token is valid → provides the details, else a 400 error is thrown.**
13. Client shows **sign-in successful or failure**.

> ⚠️ Steps 8–9 sit at the page boundary of the scan and are not fully visible; they are reconstructed from the standard flow (and consistent with steps 7 and 10).

---

## 5. API request and response

### 1) Registration process
**Request**
```
POST /register
body:
{
  "client_name":  {my app name},          // e.g. Instagram
  "redirect_uris": ["https://mysample.com/", "https://mywebsite.com/1"]   // 3 URI (callback URIs)
}
```
**Response**
```
{
  "client_name":   {my app name},
  "client_id":     {unique id assigned to client},
  "client_secret": {confidential info, known only to client and authorization server,
                    used for client authentication}
}
```

### 2) Fetch Authorization Code
**Request**
```
GET /authorize?
    response_type = code
  & client_id     = {id which the client gets during registration}
  & redirect_uri  = {callback URI like https://...}
  & scope         = scope1 scope2 scope3
  & state         = {some random value like sj11}
```
- `state` – a random value the client sends and then verifies on the callback (protects against CSRF).

---

## 6. Tokens
- **Bearer:** a security mechanism – it means the **client should add the TOKEN in the `Authorization` header whenever it wants to access the protected resources** (`Authorization: Bearer <token>`).
- **Token value:** can be either a **JWT token or a plain string**.
- **Expiry time:** generally in **seconds**. After expiry the **Refresh token should be used to get new tokens.**

---

## Quick revision
- OAuth 2.0 = **authorization** framework for delegated, secure third-party access (no password sharing).
- Actors: **Resource Owner, Client, Authorization Server, Resource Server**.
- Grants: Authorization Code (most important), Implicit, Password Credentials, Client Credentials, Refresh Token.
- Auth-code flow: register → get client id/secret → user signs in & consents → **auth code** → exchange for **access token** → call resource server with `Bearer` token → server validates token.
- `state` = anti-CSRF random value; tokens expire → use **refresh token**.
- Related: [JWT Explained](26_JWT_Explained_Notes.md) (token format).

---

<!-- nav -->
⬅ [20 Distributed Transactions](20_Distributed_Transactions_Notes.md) · [Index](00_HLD_Index.md) · [25 Symmetric and Asymmetric Encryption](25_Symmetric_and_Asymmetric_Encryption_Notes.md) ➡
<!-- nav -->
