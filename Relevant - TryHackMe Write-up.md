# TryHackMe Relevant Write-up

<img width="1513" height="351" alt="image" src="https://github.com/user-attachments/assets/8b1fa4f2-7160-4008-8602-87abc7f22323" />

## Overview
Short summary of the room and attack chain.

## Target Information
- Target IP: 10.48.184.169
- Hostname: Relevant
- OS: Windows Server 2016
- Difficulty: Medium
- Goal: Get user.txt and root.txt

## Enumeration

### Nmap Scan
```bash
nmap -p- --min-rate 5000 -T4 relevant.thm -oN allports.txt
```
<img width="1340" height="384" alt="image" src="https://github.com/user-attachments/assets/b3393f9f-0c6f-43a6-8c67-8e2300a4e6fc" />

```
nmap -sC -sV -p <ports> relevant.thm -oN detail.txt
```

<img width="1339" height="505" alt="image" src="https://github.com/user-attachments/assets/8c6172a5-94ce-473a-90a1-fb69da278a63" /> 

<img width="1382" height="855" alt="image" src="https://github.com/user-attachments/assets/e5a93114-9339-4d6c-9fe3-bc968d98a98a" />

Key ports discovered:

* 80 - IIS
* 445 - SMB
* 3389 - RDP
* 49663 - IIS web service

## SMB Enumeration

After identifying SMB running on port 445, I attempted anonymous share enumeration using `smbclient`. 

The scan revealed several default administrative shares, as well as an unusual custom share named nt4wrksv:

<img width="1338" height="279" alt="image" src="https://github.com/user-attachments/assets/6cdf0737-9792-4958-a94d-262f4e494818" />

The -N option allowed authentication without providing a password, which confirmed that anonymous access was permitted. Since `nt4wrksv` was not a default Windows share, I treated it as a high-priority target for further enumeration.

Inside the share, I discovered a file named passwords.txt, which later led to credential disclosure. This confirmed that the SMB service was misconfigured and exposed sensitive files to unauthenticated users.

## Credential Discovery

<img width="1338" height="404" alt="image" src="https://github.com/user-attachments/assets/84a01b3b-99bd-4406-80ca-c4235648cef0" />

Found some encoded credentials. Decode them like this:
```
echo 'Qm9iIC0gIVBAJCRXMHJEITEyMw==' | base64 -d
echo 'QmlsbCAtIEp1dzRubmFNNG40MjAhJCQk' | base64 -d
```
<img width="1339" height="165" alt="image" src="https://github.com/user-attachments/assets/5ef269ef-3d76-456c-bd33-a0b19e535d64" />

The passwords.txt file contained base64-encoded credentials for two users. After decoding the file, I attempted to validate whether the credentials could be reused against exposed services.

Since RDP was available on port `3389`, I tested the recovered credentials against the Remote Desktop service:
```
xfreerdp /u:<USER> /p:<PASSWORD> /v:relevant.thm
```
However, the credentials did not provide successful RDP access.

## Verifying SMB Write Access and Web Exposure

After that, I tested whether the share allowed file uploads. I created a simple test file on my Kali machine:

```
echo "test-from-kali" > test.txt
```
Then I connected to the SMB share anonymously and uploaded the file:
```
smbclient //relevant.thm/nt4wrksv -N
```
Inside the SMB session:
```
put test.txt
ls
```

<img width="1339" height="358" alt="image" src="https://github.com/user-attachments/assets/ee0fe26b-7c1f-4df3-b21a-35c1552aa18e" />

The upload succeeded, confirming that the share was writable by an unauthenticated user.

Next, I checked whether the uploaded file was accessible through the IIS web service running on port 49663:
```
curl http://relevant.thm:49663/nt4wrksv/test.txt
```
The server returned the contents of the .txt file we created and uploaded earlier: `test-from-kali` 

<img width="1341" height="497" alt="image" src="https://github.com/user-attachments/assets/bb661da8-6a09-4ae9-b030-7942c02c5dd2" />
```
49663 = web-accessible
80    = not mapped
```

This confirmed that the SMB share was not only writable, but also mapped to a web-accessible IIS directory. This was a critical finding because it meant that an attacker could upload a server-side script, such as an ASPX web shell, and execute it through the browser.

In other words, the test.txt file was used as a safe proof-of-concept to confirm the attack path before uploading a web shell.

## Initial Foothold

Explain uploading `shell.aspx`, getting command execution, and then using `nc.exe` for reverse shell.

## User Flag

Path:

```cmd
C:\Users\Bob\Desktop\user.txt
```

## Privilege Escalation

Explain:

```cmd
whoami /priv
```

Finding:

```text
SeImpersonatePrivilege Enabled
```

Then explain using PrintSpoofer to get SYSTEM.

## Root Flag

Path:

```cmd
C:\Users\Administrator\Desktop\root.txt
```

## Vulnerabilities Found

* Anonymous SMB access
* Sensitive credential disclosure
* Writable SMB share
* SMB share mapped to IIS web directory
* Remote Code Execution via ASPX upload
* SeImpersonatePrivilege abuse

## Remediation

* Disable anonymous SMB access
* Remove sensitive files from shares
* Restrict write permissions
* Do not expose SMB directories through IIS
* Harden IIS app pool permissions
* Patch/mitigate token impersonation abuse

## Lessons Learned

* Full port scans matter
* SMB write access becomes critical when tied to web access
* `whoami /priv` is important on Windows privesc

````

Big tip: **don’t include the actual flag values** in the public write-up. Replace them with:

```text
THM{REDACTED}
````


