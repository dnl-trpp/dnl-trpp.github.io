---
layout: post
title:  "Pollute only after Cleaning! (0623 Intigriti Challenge)"
date:   2023-06-21 15:00:00 +0100
categories: writeup Intigriti
---

### Introduction

As soon as my phone got the notification, I knew I would have to cancel my plans for the next two days. I I thought this as a joke, but i ended up spending waaay to much time on this. After going through multiple rabbit🐇 holes, I spend around 20 hours total to finally come up with a solution.

As usual, I would like to go through my steps again and not only disclosing the results. That said, let's start.

(At the end of this article you will find a TL;DR; and the final payload)

### The Challenge
At the time of writing, the challenge page is aviliable [here](https://challenge-0623.intigriti.io/challenge/index.html). The page welcomes us with a textfield and a button. Once pressed, the input is reflected on the screen. So what about the standard payload `<script>alert(document.domain)</script>`? Obviously this doesn't work, but why? 

![hey](../assets/hey0623.png){: .right }

So the reason out input is sanitized, is because `setHTML` of the new [Sanitizer API](https://developer.mozilla.org/en-US/docs/Web/API/HTML_Sanitizer_API) is used. It actually allows other tags like `img`, but at the same time also strips a lot of attributes to prevent script execution. Also note that the challenge code is loaded with a [defer](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/script#defer) attribute. This has nothing to do with the actual challenge, but it ensures that the sanitizing code is executed after the `DOMContentLoaded`.

But let's take a step back by looking at the source code.

At the top of the page, two scripts are loaded:

```html
<script src="./static/jquery-2.2.4.js"></script>
<script src="./static/jquery-deparam.js"></script>
```

Look at that! `jquery 2.2.4` is not the latest version, maybe there is some known cve? Also `deparam`  seems an interesting library but first, let's go through the challenge quickly:
* First, a check on `document.domain` is performed, and based on the results, the recaptcha propriety is set. 
```javascript
if (document.domain === 'challenge-0623.intigriti.io') {
    window.recaptcha = false
}
if (document.domain === 'localhost') {
    window.recaptcha = true
}
```

* Then, the name variable is cleared, and a param object is construted using the `deparam` function and the parameters we passed in the URL
```javascript
window.name = null;
window.params = $.deparam(location.search.slice(1))
```

* We have some code that handles sumbitting and button pressing, but we don't need that.

* Finally, after the DOM is loaded the following is executed. The name variable is extracted from our params and assigned to name. Then, only  `window.recaptcha` is set, the google reCaptcha library is loaded . After that, our input variable `name` is sanitized and reflected on screen. The `try-catch` statment is only for compatibility purposes for older browsers that don't support the Sanitizer API. (Understanding this took me a while)
    ```javascript
    name = window.params.name;
    if (name && name !== 'undefined' && name !== undefined) {
        const modal = document.getElementById('modal');
        modal.style.display = 'flex';
        const modalContent = document.getElementById('modalContent');
        // recaptcha is still under development
        if (window.recaptcha) {
            const script = document.createElement('script');
            script.src = 'https://www.google.com/recaptcha/api.js';
            script.async = true;
            script.defer = true
            document.head.appendChild(script);
        }
        try {
            modalContent.setHTML(name + " 👋", {sanitizer: new Sanitizer({})}); // no XSS
        } catch {
            modalContent.textContent = name + " 👋"
        
    }
    ```

### Polluting the Planet

So what can we do now? Probably the challenge isn't asking us to find a zero day in the Sanitizer library, so let's try to approach this differently. Just because this is a challenge, the way recaptcha is implemented is strange, and probably requires us to find a bypass to activate it (When loaded on the domain, `window.recaptcha` is set to `false`). After some messing around with other stuff, I found that the `deparam` lib is vulnerable to a very known `Prototype pollution`. At this point, the first hint by `intigriti` was published, and it also clearly hinted that this was the right way.

A Poc for this vulnerability is found [here](https://github.com/BlackFan/client-side-prototype-pollution/blob/master/pp/jquery-deparam.md). I won't go through what prototype pollution is, as there are plenty of resources online, but in a essence it allows us to set "global" proprieties on all objects. More precisely it allows us to overwrite the prototype of the `Object` class. 

On the challenge site, passing `?__proto__[test]=hello`  as parameter, sets the `test` proprety of `Object.prototype` to the string `hello`.

### The window.name rabbit🐇 hole
> This paragraph has to do with my researching process, and some pitfalls. If you are only interested in the solution, you can skip this paragraph.
{: .prompt-info }

Since the start of the challenge, I had some eyes for the `name` variable. The variable is reset at the start with `window.name = null` and above all, some of you may know that this isn't a variable like another. [window.name](https://developer.mozilla.org/en-US/docs/Web/API/Window/name) can be used for cross domain comunication in some cases, but what struck me was this note on the official documentation:
```
Note: window.name converts all stored values to their string representations using the toString method.

```
Wait, what? I didn't know about that. Apparently, setting something to a variable called `name` automatically calls `toString`. As a consequence, you can't assign objects or numbers (or anything else) to the variable. 
Our challenge code assignes the parameter we are passing in to the name variable. If somehow we could pass an object as a parameter, the `toString()` method on this object will be called and guess what, we can pollute that!
>You can test this now in your browser! Open the developer tools and type `Object.prototype.toString = x=>alert(1)`. This will overwrite (or pollute) the toString of the Object class with an alert. After that assign an object to `name`. (`name = {}` for example). An allert pops!

After some digging, I found that conviniently `$.deparam` supports the passing of object! Simply use a key containing `[]`. (This was before I knew about the prototype pollution). 
I came up with the following payload : `?name[test]=test&__proto__[toString]=function(){alert(1)}`
Unfortunately, this doesn't fire. The reason is that we are not reassigning the alert function to `toString`, we are literally reassinging the string `"function(){alert(1)}"`. At the end, there was no way to bypass this. It seems that our prototype pollution only allows us to set a proprety to a string, an object or an array. I also confirmed this by looking through the `deparam` source code.
After hours spent on this, I decided to move on.

### The reCaptcha gadget
As you might now, a prototype pollution vulnerability is not useful if we don't have a gadget. In this case a `gadget` means some piece of code that, once we pollute Object.prototype, behaves diffrently. Previously I was trying to use the `name` variable as a gadget, but was unsuccessful. 
At some point, I found out that google reCaptcha is a known gadget for prototype pollution! 
PoC can be found [here](https://github.com/BlackFan/client-side-prototype-pollution/blob/master/gadgets/recaptcha.md). 
Basically by polluting the `srcdoc` propriety, we can directly gain script execution!
This works but we have to fulfill two conditions:
* In our application we have to set `window.recaptcha` to true (or something that evaluates to true ;D )
* There has to exist an HTML element with a `class` attribute set to `g-recaptcha` and `data-sitekey` set to anything we like
The second point has to do with the inner workings of google's reCaptcha library. I observed by reproducing locally that the `XSS` doesn't fire if both conditions aren't met.
Let's focus on the first point first!

### Cleaning before polluting!
It is very clear that somehow we have to bypass the `if(window.captcha)` check. The only way seems to have document.domain set to `localhost`. After some researching, I concluded that the document.domain variable has some severe security config. For instance, it can never be set to a random value, but only on a parent domain.
Let's take a step back, what if we directly pollute the recaptcha variable. We set it to a string, and a string will be evaluated to true right? Wrong. The thing is, an object (`window` in this case) get's a propriety from `Object.prototype` only if it's not defined lower in it's inheritance chain (don't know if it's called so). In other words, to pollute window.recaptcha, we have to make sure that it's not locally defined in the code. 
To do so, it is enough to have `document.domain` set to literally anything diffrent that `challenge-0623.intigriti.io`. I spent a lot of time researching. at the end I came up with a little trick I read some time ago. Ready?
We append a dot at the end of our domain. Yes, just a dot. The reason is, the dot is the Top Level Domain for ALL domains, but we don't have to put it in the URL, as browsers understand anyway.
If we add it, everything works like fine, but technically `document.domain` will be different from `challenge-0623.intigriti.io`.
With that in place our plan is:
* make `window.recaptcha` undefined by visiting `challenge-0623.intigriti.io.` instead of `challenge-0623.intigriti.io`
* pollute `window.recaptcha` using the usual deparam vulnerability (add `__proto__[recaptcha]=true` in the URL parameters)

> Everytime I find something like this, it is the result of multiple testing and experiments. Usually, the console in the developer tools helps a lot!
{: .prompt-tip }

### The Sanitizer({}) rabbit🐇 hole
> This paragraph has to do with my researching process, and some pitfalls. If you are only interested in the solution, you can skip this paragraph.
{: .prompt-info }

Now, we have almost everything. To use reCaptcha as a gadget, we are just missing an html element with a class set to `g-recaptcha` and a custom attribute `data-sitekey`. Now as for the class, the Sanitizer allows that in the default configuration, but it strips any custom attribute.

After some digging, I found a very interesting fact. The sanitizer is created using the following line:
```javascript
new Sanitizer({})
```
{: .nolineno }
as opposed to
```javascript
new Sanitizer()
```
{: .nolineno }
Now, this may seem as identical, but there is a small difference. The Sanitizer constructor accepts a config object. If this object is empty, it behaves as it it's not there at all, and defaults to the standard configuration. 
But... what if the empty object, has some proprieties by default. A polluted Object prototype, will pass the proprieties in the default constructor, if the Sanitizer is passed the empty object! I checked this in the console:

![sani](../assets/Sani_tests.png){: w="600" h="400" }
_Experimenting with different instantiations of the Sanitizer Object in the Developer Console_

I though this was it, configurate the Sanitizer to allow custom tags and let's go. Sadly, this was not the way. For some reason I wasn't able to allow custom elements, nor tags or anything. Even when I tried to reproduce locally with normal javscript code.
After enough hours lost on this, I move on.
### Pollute the world!

So really, only an HTML element with a custom attribute is missing right? I will jump to the conclusion, as I'm tired of writing, but at the end we don't need an HTML element. The `sitekey` variable just needs to be defined. As it's starts as `undefined` I guess, we can pollute it. So passing `__proto__[sitekey]=something` will be enough to fire to execute the reCaptcha API correctly, and Fire the XSS!

### Final payload
As a quick recap. Or TL;DR;
* We clean `window.recaptcha` by using a dot at the end of the domain
* We pollute `window.recaptcha` using a prototype pollution vulnerability in the `deparam` function
* We pollute the `srcdoc` propriety, as seen in the PoC that uses reCaptcha as a gadget
* We pollute the `sitekey` propriety, as it is required for `reCaptcha` to work
* We pass an html element (img for example) with a `class` attribute set to `g-recaptcha` in the name variable
  
We put everything togheter and we get:
```
https://challenge-0623.intigriti.io./challenge/?name=%3Cdiv%20class%3D%22g-recaptcha%22/%3E&__proto__[recaptcha]=true&__proto__[srcdoc][]=%3Cscript%3Ealert(document.cookie)%3C/script%3E&__proto__[sitekey]=test
```

You can visit the link above by clicking [here](https://challenge-0623.intigriti.io./challenge/?name=%3Cdiv%20class%3D%22g-recaptcha%22/%3E&__proto__[recaptcha]=true&__proto__[srcdoc][]=%3Cscript%3Ealert(document.cookie)%3C/script%3E&__proto__[sitekey]=test) and you will get:

![fired](../assets/fired0623.png){: width="600" height="400" }
_POPUP!_

### Conlusion
As with any challenge, I learned a lot. And I probably learned even more by trying and going thorugh some rabbit holes, exploring what made me uncomfortable and what I initially discarded.

Some takeaways:
* Go through the documentation
* Reporduce and experiment locally
* Don't be afraid to try something new
* Stay hydrated🍺

