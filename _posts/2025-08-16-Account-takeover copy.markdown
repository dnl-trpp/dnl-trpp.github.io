---
layout: post
title:  "Account takeover by abusing fast login URLs"
description: "Turns out generating endless login URLs isn’t such a great idea."
date:   2025-08-16 16:00:00 +0100
categories: writeup
image:
  path: /assets/img/dnl-airport.jpg
  alt: Me at the airport, so focused on my screen that I don't notice my flight is half an hour late
---

## *TL;DR; (TikTok Length for Dopamine Rush)*

*`redacted.com` featured a 'one-time login' URL generator that could create unlimited login links for any user account. The security flaw was that each URL relied on just a 10-character alphanumeric token. An attacker could spam the generation endpoint to create hundreds of valid tokens, then brute-force their way into any account by drastically improving their odds of guessing a working code.*

## Introduction

I hate catching flights. Not the flying part, but those soul-crushing two-hour waits that make me feel like I'm burning daylight for no reason. Fortunately, even if I'm not good at multitasking, I'm still able to work on my laptop without missing (almost) any of my gate announcements. So here I am, fresh off updating this blog's theme during an airport wait and ready to share a relatively simple account takeover I found during a (not so) recent pentest. 

The client that was vulnerable is not in business anymore and the bug got fixed within a week, but my *lawyer-dodging* instincts are telling me to call the target `redacted.com`. I might slightly change some technical details but nothing will change substantially. Let's go!

## It's not a bug, it's a feature

The platform had your typical user registration setup so nothing groundbreaking there. But what caught my attention was their instant messaging feature between users, even though it wasn't really the main focus of the app. 

> Use the [plus addressing](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/plus-addressing-in-exchange-online) trick to register multiple accounts for a site, but receive all emails in the same inbox.
{: .prompt-tip }

So I started poking around with my usual testing methodology, and when I tried sending a message to another user, I noticed the request had this structure:
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

The `to` parameter is just the User ID of whoever you're messaging. The user IDs were incremental and could be enumerated, but that by itself wasn't really a vulnerability. 

> From now on we'll have two accounts involved in this story.  We'll call them account one and account two with emails respectively of `daniel.trippa+account1@gmail.com` and `daniel.trippa+account2@gmail.com`
{: .prompt-info }

Pretty standard stuff here. The request just made the other user receive your message in their inbox. Nothing obviously exploitable jumped out at me from this functionality, so I quickly moved on to test other things. That's when I noticed something unexpected in my Gmail inbox (I don't remember the exact wording, but it was something like this):
```
From: notifications@redacted.com
To: daniel.trippa+account2@gmail.com
Subject: Redacted.com: You received a message while you where away!

daniel.trippa+account1@gmail.com sent you a message!
Click here to answer.
```

The `Click here` had a link structured like this: `https://redacted.com/messages/quick_view=<code>` where `<code>` was a 10-character alphanumeric string, something like `EcG9A4hLL0`.

Clicking the link took me straight to a screen showing the incoming message. Pretty neat feature, right? 
Well, yeah, until I realized something weird: I wasn't logged into `account2` (the one that received the message), but I could still view the message. I double-checked this and confirmed that even without any session cookies, you could visit the link and read the message content. 

## More features, not more bugs

Now, the functionality I just described could already be considered an IDOR vulnerability on its own, since you could theoretically enumerate messages from random users by brute-forcing those alphanumeric codes. 

This is easier said than done. A 10-character alphanumeric string has a keyspace size of `62^10`, which is... well, let's just say it's huge but not impossible to brute force locally. The problem is, even with top-tier hardware, unlimited bandwidth, and a server that somehow doesn't break under pressure, you might squeeze out around 30,000 requests per second over the network. At that rate, you'd exhaust the keyspace on average in about... `~443,000` years. That's more than a lifetime, as you probably now.

So with just the IDOR alone, we're not looking at a critical vulnerability. But here's where things get interesting. After visiting the link and viewing the message, I hit the website's home button and noticed something: I was now logged in as `account two` (the message recipient). At first, I thought maybe I already had a valid session, but after retesting through Burp Suite, I confirmed that visiting the link actually triggered the server to respond with a `Set-Cookie` header containing valid session cookies. 

## The takeover

Having an IDOR that automatically logs you into someone else's account is already pretty bad, but it gets way worse when the code protecting this functionality isn't actually that secure. The account takeover procedure would be simple:
- Send a message to a user (your victim)
- Brute-force the `<code>` in `https://redacted.com/messages/quick_view=<code>` until you're logged into the victim's account.

The catch is, as I mentioned before, that code should theoretically be secure since you'd have to brute-force it over the network.

But there's another lovely "feature" that makes this whole thing actually exploitable. After playing around with the links, I noticed that sending a second message generated a brand new link (delivered to the receiver's inbox) without invalidating the previous one! The links expire after 24 hours, but that's not enough to prevent a takeover, as we'll see.

It took me a moment to realize the real issue: being able to generate multiple valid tokens massively improves the odds of guessing one. But how much? The short version is that each new token linearly "shrinks" the keyspace, reducing the time needed to guess a valid one.

In practice, generating `x` tokens reduces the expected time to find a valid one as if the keyspace had been divided by a factor of `x`. For example, with `62⁵` tokens, the effective search space "shrinks" from `62¹⁰` down to `62⁵`. That is equivalent to working with a 5-character code.

At 30,000 req/s, generating those tokens takes ~8.5 hours, and brute-forcing another ~8.5 hours. That’s a full account takeover **expected** in ~17 hours instead of 443,000 years.

> Note that this is a probabilistic approach. In principle, fully exhausting the keyspace would still require on the order of 62¹⁰ requests. However, if the tokens are sampled from a uniform distribution, the likelihood of needing that many attempts is extremely small. I’ll soon publish a deep dive exploring this in more detail.
{: .prompt-info }

## Some limitations
Here's the reality check though: the server, my hardware, and my connection were nowhere near capable of `30,000` requests per second. In optimal conditions, I was pulling maybe `200-300 req/s`. The vulnerability is still technically exploitable under these conditions, but we're talking about days of brute-forcing. And that's not even considering that tokens expire after 24 hours, which caps the number of tokens you can realistically generate within your attack window.

At this point, I didn't actually carry out the full attack since it would've required way too many resources for a simple proof of concept. Instead, I reported the vulnerability with a detailed explanation of how this could be a real exploitable issue in the wild. The team accepted the finding and patched it within a week. Their fix was pretty straightforward: just use much longer, more secure codes. The email link still creates a valid session when visited, which arguably isn't the optimal solution, but it's definitely not exploitable anymore.

## Conclusions

Overall the vulnerability was very simple, but I spent some time in investigating how the 
keyspace can be reduced by generating multiple valid codes. The main lesson here is simple: if your site has functionality that lets users generate unlimited login links for any account, you better make sure those links are truly unguessable.

Key takeaways:
* Test for cross-account interactions
* If you can generate mutiple valid tokens (e.g. OTP codes) that might be a powerful primitive for exhausting the keyspace!
* Don't miss your gate announcements at the airport ✈️



