# Module 12 - The Web's Language (HTTP/HTTPS)

> *Every website you've ever used runs on this. Understanding it properly changes how you see the web entirely.*

---

## What HTTP Actually Is

HTTP (HyperText Transfer Protocol) is a text-based request/response protocol. Your browser sends a request, the server sends back a response. That's it. Every page load, every form submission, every image download - all of it is HTTP requests and responses going back and forth.

It runs over TCP, typically on port 80 (HTTP) or 443 (HTTPS). It's stateless by design - each request is independent, the server doesn't automatically remember you from one request to the next. Cookies and sessions exist specifically to work around that statelessness.

---

## Requests and Responses

Every HTTP interaction has two parts: a request from the client and a response from the server.

A basic GET request looks like this:

```
GET /index.html HTTP/1.1
Host: www.example.com
User-Agent: Mozilla/5.0
Accept: text/html
Cookie: session=abc123
```

- **Method** - what action to perform (GET, POST, PUT, DELETE, etc.)
- **Path** - which resource to act on
- **Headers** - metadata about the request
- **Body** - data sent with the request (only on POST/PUT)

The server responds:

```
HTTP/1.1 200 OK
Content-Type: text/html
Set-Cookie: session=abc123; HttpOnly; Secure
Content-Length: 1234

<html>...</html>
```

- **Status code** - result of the request
- **Headers** - metadata about the response
- **Body** - the actual content

---

## HTTP Methods

| Method | What it does |
|--------|-------------|
| GET | Retrieve a resource - parameters go in the URL |
| POST | Submit data - parameters go in the request body |
| PUT | Create or replace a resource |
| DELETE | Delete a resource |
| HEAD | Same as GET but returns only headers, no body |
| OPTIONS | Ask what methods the server supports |
| PATCH | Partial update to a resource |

GET vs POST matters for security. GET parameters end up in the URL and get logged by servers, proxies, and browsers. Passwords in a GET request show up in server logs, browser history, and referrer headers. POST sends data in the body - not in the URL - which is why login forms use POST.

---

## Status Codes

The three-digit status code tells you exactly what happened. You'll read these constantly in Burp Suite and proxy logs.

| Range | Meaning | Common Examples |
|-------|---------|----------------|
| 2xx | Success | 200 OK, 201 Created, 204 No Content |
| 3xx | Redirect | 301 Moved Permanently, 302 Found (temp redirect) |
| 4xx | Client error | 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found |
| 5xx | Server error | 500 Internal Server Error, 503 Service Unavailable |

The difference between 401 and 403 matters. **401 Unauthorized** means you need to authenticate - no credentials or wrong ones. **403 Forbidden** means the server knows who you are but won't let you in - authenticated but not authorized. That distinction tells you something about how the access control works.

500 errors during testing are interesting - they often mean you've sent input the server didn't expect, which can indicate injection vulnerabilities.

---

## Cookies and Sessions

HTTP is stateless. Cookies are how servers remember who you are between requests.

When you log in, the server creates a session on its end and sends you back a session token in a `Set-Cookie` header. Your browser stores that cookie and automatically sends it with every subsequent request to that domain. The server looks up the session token, finds your session, and knows who you are.

```
Login request:
POST /login
username=john&password=secret

Server response:
Set-Cookie: session=eyJhbGciOiJIUzI1NiJ9...; HttpOnly; Secure; SameSite=Strict

All future requests automatically include:
Cookie: session=eyJhbGciOiJIUzI1NiJ9...
```

The session cookie is essentially a temporary identity token. If you have someone's session cookie, you can make requests as them - no username or password needed. This is session hijacking, and it works directly from packet captures on unencrypted connections or through XSS attacks.

Cookie flags matter:
- **HttpOnly** - JavaScript can't access the cookie, blocks XSS-based theft
- **Secure** - only sent over HTTPS, never over HTTP
- **SameSite** - controls cross-site sending, mitigates CSRF

---

## HTTPS - HTTP Inside a Tunnel

HTTPS is just HTTP with TLS encryption wrapped around it. The HTTP part works identically - same methods, same headers, same status codes. The TLS layer handles:

- **Encryption** - traffic between client and server is encrypted, unreadable to anyone in the middle
- **Authentication** - the server's certificate proves you're talking to the real server and not an impostor
- **Integrity** - traffic can't be modified in transit without detection

```
HTTP:
[Browser] --------- cleartext HTTP ---------> [Server]
           anyone on the path can read it

HTTPS:
[Browser] == TLS handshake == [Server]
[Browser] ===== encrypted HTTP ============> [Server]
           traffic is encrypted end-to-end
```

HTTPS protects data in transit. It does not protect against weak passwords, SQL injection, broken access control, or any server-side vulnerability. A site can be fully HTTPS and completely insecure in every other way.

---

## Burp Suite - Web Proxy

Burp Suite is the standard tool for web application testing. It sits between your browser and the target server as a proxy - every request goes through Burp before it reaches the server, and every response goes through Burp before your browser sees it.

```
[Your Browser] --> [Burp Suite Proxy] --> [Web Server]
                         |
                    intercept, read,
                    modify, replay
```

Setup is straightforward:

```
1. Start Burp Suite
2. Go to Proxy > Options - confirm it's listening on 127.0.0.1:8080
3. Configure your browser to use 127.0.0.1:8080 as HTTP proxy
4. Install Burp's CA certificate in your browser (for HTTPS interception)
5. Turn Intercept on in Proxy > Intercept
```

Now every request your browser makes shows up in Burp. You can read it, modify any field, then forward it to the server.

Key Burp features:

**Intercept** - catch and modify requests before they're sent. Change parameter values, headers, cookies - anything in the raw request.

**Repeater** - take a captured request and resend it manually as many times as you want with different values. Useful for testing how a parameter responds to different input.

**Intruder** - automated attack tool. Mark positions in a request, load a payload list, fire. Used for brute-forcing login forms, fuzzing parameters, testing for injection.

**Decoder** - encode/decode base64, URL encoding, HTML entities, hex. Web apps pass data around in various encodings - this decodes it quickly.

**Scanner** - automated vulnerability scanning (Pro version only).

---

## Intercepting and Modifying Requests

The simplest example: intercepting a login form to see what gets sent.

```
Captured POST request in Burp:

POST /login HTTP/1.1
Host: target.com
Content-Type: application/x-www-form-urlencoded
Cookie: session=guest_token_here

username=john&password=mypassword&role=guest
```

With Burp's Intercept on, this request is frozen before it reaches the server. You can change any value:

- Change `role=guest` to `role=admin`
- Change `username=john` to `username=administrator`
- Modify cookie values
- Add or remove headers

Forward it, and the server receives your modified version. If the application only enforces access control on the client side (trusting values the user sends), this kind of parameter tampering works.

---

## Brute-Forcing with Intruder

Burp Intruder automates sending modified requests. Common use case: testing a login form with a password list.

```
1. Capture a login request in Burp
2. Send it to Intruder (right-click > Send to Intruder)
3. In Positions tab, mark the password field as a payload position
   username=admin&password=§password§
4. In Payloads tab, load a wordlist (rockyou.txt or a custom list)
5. Start attack
6. Watch the responses - different status code or response length = likely success
```

The "Cluster Bomb" attack type tries every combination across multiple payload positions simultaneously. If you have a username list and a password list, Cluster Bomb tries every username with every password.

Response analysis is the key part. A failed login returns a 200 with "Invalid credentials." A successful one returns a 302 redirect to the dashboard. Filter by status code or response length to find the hit.

---

## Key HTTP Headers for Security

Headers carry a lot of security-relevant information in both directions:

**Request headers worth knowing:**
- `Host` - the domain being requested (important for virtual hosting)
- `Authorization` - credentials for HTTP auth
- `Cookie` - session tokens and other stored values
- `Referer` - where the request came from
- `X-Forwarded-For` - original client IP when going through a proxy

**Response headers worth knowing:**
- `Set-Cookie` - sets cookies with their flags
- `Content-Security-Policy` - tells the browser what resources are allowed to load
- `X-Frame-Options` - controls whether the page can be framed (clickjacking protection)
- `Strict-Transport-Security` - forces HTTPS for future requests
- `Server` - reveals web server software and version (useful for fingerprinting)

Missing security headers are findings in themselves during a web assessment. A site without CSP is more vulnerable to XSS. No HSTS means downgrade attacks are possible.

---

## Quick Reference

| Method | Use |
|--------|-----|
| GET | Retrieve data, params in URL |
| POST | Submit data, params in body |

| Status Code | Meaning |
|------------|---------|
| 200 | OK |
| 301/302 | Redirect |
| 401 | Needs authentication |
| 403 | Authenticated but forbidden |
| 404 | Not found |
| 500 | Server error |

| Cookie Flag | What it prevents |
|------------|----------------|
| HttpOnly | XSS-based cookie theft |
| Secure | Transmission over HTTP |
| SameSite | CSRF attacks |

---

## Lab

```bash
# 1. Set up Burp Suite as a browser proxy
# Start Burp > Proxy > Options > confirm 127.0.0.1:8080
# Configure browser proxy settings to match
# Install Burp CA cert for HTTPS

# 2. Visit any HTTP site with Intercept on
# Read the raw request - find the User-Agent, Host, Cookie headers

# 3. Submit a login form on a test site (DVWA, HackTheBox, etc.)
# Capture the POST request in Burp
# Can you see the password in plaintext in the request body?

# 4. Send the captured request to Repeater
# Modify a parameter value and resend
# Compare the responses

# 5. Check security headers on a real site
curl -I https://example.com
# Look for: Strict-Transport-Security, Content-Security-Policy, X-Frame-Options
```

**Things to think about:**

- A login form uses GET instead of POST. Where does the password end up, and why is that a problem?
- You capture a session cookie without HttpOnly set. What attack does that enable, and how would you execute it?
- HTTPS is enabled on a site. A parameter in a POST request is being passed directly to a SQL query. Does HTTPS protect against this? Why or why not?

---

*Next: more network protocols, deeper into the stack - where lower-level attacks and advanced exploitation techniques live.*
