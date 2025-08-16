---
layout: post
title:  "Breaking the One-Time Promise"
description: "How I exploited simultaneous login code generation to compromise user accounts"
date:   2025-09-16 16:00:00 +0100
categories: writeup
image:
  path: /assets/img/dnl-airport.jpg
  alt: Me at the airport, so focused on my screen that I don't notice my flight is half an hour late
---

## *TL;DR; (TikTok Length for Dopamine Rush)*

*`redacted.com` featured a 'one-time login' URL generator that could create unlimited login links for any user account. The security flaw was that each URL relied on just a 10-character alphanumeric token. An attacker could spam the generation endpoint to create hundreds of valid tokens, then brute-force their way into any account by drastically improving their odds of guessing a working code.*

## Introduction

I hate catching flights. Not the flying part, but those soul-crushing two-hour waits that make me feel like I'm burning daylight for no reason. Fortunately, even if I'm not good at multitasking, I'm still able to work on my laptop without missing my gate announcements. So here I am, fresh off updating this blog's theme during another airport wait and ready to share a vulnerability I found during a (not so) recent pentest. 

The client that was vulnerable is not in business anymore and the bug got fixed within a week, but my *lawyer-dodging* instincts are telling me to call the target `redacted.com`. I might slightly change some technical details but nothing will change substantially. Let's go!

## It's not a bug, it's a feature

So the platform allowed users of the service to register an account autonomously. One functionality they offered was an instant messaging between users, even if this was not the main focus of the app. After diving deep into my usual testing routine and exploration, I discovered that when you sent a message to another user, the request structure looked like the following:
```http
POST /api/message HTTP/1.1
Host: redacted.com
Cookie: Session=<session-one-token>
Content-Type: application/json
Content-Length: [...]

{
  "to": 12345678,
  "text": "Hello my friend"
}
```

The `to` parameter is the User ID of another user. The user IDs were incremental and could be enumerated, but this alone was not a vulnerability on the site.


