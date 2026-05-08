<img width="1509" height="389" alt="image" src="https://github.com/user-attachments/assets/5476ac46-be86-492b-9608-9bde5fe257d1" />

***

The Net Sec Challenge room tested the core enumeration and network security skills covered in the Network Security module, mainly using Nmap, Telnet, and Hydra.

## Challenge Questions

Answer the following questions using:

- Nmap
- Telnet
- Hydra

***

### 🔹 What is the highest port number being open less than 10,000?

Performed a scan:

```bash
nmap -p- --min-rate 1000 10.149.160.48
```

<img width="1286" height="340" alt="image" src="https://github.com/user-attachments/assets/8e762efc-dd04-4321-9ac5-1d59d1bf04e7" />

### 🔹 There is an open port outside the common 1000 ports; it is above 10,000. What is it?

<div style="border-left: 4px solid #2ea043; padding: 10px 15px; background-color: #0d1117; border-radius: 6px;">

**Answer:** `8080`

</div>

### 🔹 How many TCP ports are open?

<div style="border-left: 4px solid #2ea043; padding: 10px 15px; background-color: #0d1117; border-radius: 6px;">

**Answer:** `6`

</div>

### 🔹 What is the flag hidden in the HTTP server header?

<img width="1284" height="248" alt="image" src="https://github.com/user-attachments/assets/251ed0fa-81e4-429a-a18c-03528fa63c59" />

<br>

<div style="border-left: 4px solid #2ea043; padding: 10px 15px; background-color: #0d1117; border-radius: 6px;">

**Answer:** `THM{web_server_25352}`

</div>

### 🔹 What is the flag hidden in the SSH server header?

Used Telnet to grab the SSH banner:

```bash
telnet 10.49.160.108 22
```
<img width="1283" height="227" alt="image" src="https://github.com/user-attachments/assets/2a0ad874-10bb-4aaf-8524-33f5f927fbeb" />

<br>

<div style="border-left: 4px solid #2ea043; padding: 10px 15px; background-color: #0d1117; border-radius: 6px;">

**Answer:** `THM{946219583339}`

</div>

### 🔹 We have an FTP server listening on a nonstandard port. What is the version of the FTP server?

Performed service/version detection on port 10021:

```bash
nmap -sV -p 10021 10.49.160.108
```
<img width="1284" height="285" alt="image" src="https://github.com/user-attachments/assets/b04fce05-9c7f-4f94-b3de-6ca62102ef32" />

<br>

<div style="border-left: 4px solid #2ea043; padding: 10px 15px; background-color: #0d1117; border-radius: 6px;">

**Answer:** `vsftpd 3.0.5`

</div>

### 🔹 We learned two usernames using social engineering: eddie and quinn. What is the flag hidden in one of these two account files and accessible via FTP?

Used Hydra to brute-force the FTP credentials for both users on port 10021.

```bash
hydra -l eddie -P /usr/share/wordlists/rockyou.txt ftp://10.49.160.108 -s 10021

hydra -l quinn -P /usr/share/wordlists/rockyou.txt ftp://10.49.160.108 -s 10021
```
<img width="1292" height="522" alt="image" src="https://github.com/user-attachments/assets/df3cae1e-cbd0-4024-a2e3-f033a74244e7" />

<br>

Next, Logged into the FTP service using the discovered passwords.

First, accessed the FTP server as `eddie`, but no flag file was found.

<img width="1284" height="500" alt="image" src="https://github.com/user-attachments/assets/2d9df087-9fab-4af0-9663-4b9b86887992" />

<br>

Then logged in as `quinn` and found a file named `ftp_flag.txt`.

<img width="1283" height="654" alt="image" src="https://github.com/user-attachments/assets/95bd1fd8-fb1a-44c6-aa74-e95bf1312a44" />

<br>

Downloaded the file from ftp using:

```bash
get ftp_flag.txt
```

and then we open and obtained the flag:

<img width="1286" height="106" alt="image" src="https://github.com/user-attachments/assets/47bbf05e-676d-46b7-b4f2-5213478f8d72" />
<br>
<div style="border-left: 4px solid #2ea043; padding: 10px 15px; background-color: #0d1117; border-radius: 6px;">

**Answer:** `THM{321452667098}`

</div>

### 🔹 Browsing to http://10.49.160.108:8080 displays a small challenge that will give you a flag once you solve it. What is the flag?

The web challenge required performing an Nmap scan while minimizing IDS detection.

A NULL scan was used to reduce detection rates and successfully complete the challenge:

```bash
nmap -sN 10.49.160.108
```
<img width="956" height="523" alt="image" src="https://github.com/user-attachments/assets/2d879059-b4be-48f7-a2ac-c8922e6c8a74" />
<br>
<div style="border-left: 4px solid #2ea043; padding: 10px 15px; background-color: #0d1117; border-radius: 6px;">

**Answer:** `THM{f7443f99}`

</div>






















