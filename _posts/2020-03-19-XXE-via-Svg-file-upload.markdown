---
layout: post
title:  "XXE via .svg file upload"
date:   2020-03-19 16:52:46 +0100
categories: writeup portswigger
---
# *What's an XXE?*
XXE stands for XML external entity and basically it's a common web vulnerability that allows an attacker to inject external entities in an [XML file](https://en.wikipedia.org/wiki/XML). The vulnerable server will then parse the injected payload allowing the attacker to read files or also make a request impersonating the server. More about XML, External entities and XXE's **[here](https://portswigger.net/web-security/xxe)**.

`Svg` instead stands for [Scalable Vector Ghrapic](https://en.wikipedia.org/wiki/Scalable_Vector_Graphics). It's a common image format for vector graphics, mainly used for logos but most importantly... it's based on `XML`!<br/>
This means that a site that allows us to upload an `.svg` file is basically allowing us to upload an XML, and thus it may be vulnerable to XXE!

An svg file look's like this:
{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<svg height="100" width="100">
  <circle cx="50" cy="50" r="40" stroke="black" stroke-width="3" fill="red" />
  <text x="50" y="50" text-anchor="middle">Text Here!</text>
</svg> 
{% endhighlight %}
And when opened with a browser we see this:
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<svg xmlns="http://www.w3.org/2000/svg" height="100" width="100">
  <circle cx="50" cy="50" r="40" stroke="black" stroke-width="3" fill="red"></circle>
  <text x="50" y="50" text-anchor="middle">Text Here!</text>
</svg> 


# *Exploiting an svg upload*
So let's see what an exploited svg file looks like. Let's take the file above as an example. Basically we want to inject a entity that reads a file. We will end with something like this:
{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE example [ <!ENTITY xxe SYSTEM "file:///filepath/goes/here" > ]>
<svg xmlns="http://www.w3.org/2000/svg" height="100" width="100">
  <circle cx="50" cy="50" r="40" stroke="black" stroke-width="3" fill="red" />
  <text x="50" y="50" text-anchor="middle">&xxe;</text>
</svg> 
{% endhighlight %}
As we see, the trick is simple. We just need to be sure that the `entity` is referred inside a `<text>` tag, otherwise we won't be able to read anything. 

# *What this looks like in practice*
As a practical example I will show you how to solve port swigger's lab "[Exploiting XXE via image file upload](https://portswigger.net/web-security/xxe/lab-xxe-via-file-upload)".<br/>
The description says:
>>This lab lets users attach avatars to comments and uses the Apache Batik library to process avatar image files.
>>To solve the lab, upload an image that displays the contents of the /etc/hostname file after processing. Then use the "Submit solution" button to submit the value of the server hostname.

So the lab simulates a blog with a vulnerable comment section that looks like this:

![comments](/assets/comment_section.png)

We can see it has an avatar upload function. Let's try to upload the first shown svg file... and it works! Our comment will show up with this title:<br/>
![first_upload](/assets/first_upload.png)<br/>
The text inside is tiny, but we can always right click and open in a new tab to make it readable.

Now let's try the malicious one. We put the requested file path `/etc/hostname` in the external entity and then upload the file. We open in a new tab and Bang!:<br/>
![second_upload](/assets/second_upload.png)<br/>
We just got reading access to an internal file. If we submit the value we just read we solve the lab, Congratulations!

Hope reading this helped to understand xxe's in this particular case. <br/>*Thank you for reading!*




