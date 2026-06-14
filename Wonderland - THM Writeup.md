<img width="1909" height="460" alt="image" src="https://github.com/user-attachments/assets/c187bcd7-e24c-4406-b847-fa82b4a3d367" />

# Wonderland - Enter Wonderland and capture the flags.

## 1. Recon

```
nmap -sC -sV -oN wonderland.txt 10.49.151.224
```

<img width="1342" height="406" alt="image" src="https://github.com/user-attachments/assets/3ea8946c-70ef-408a-b0e1-f051065626ad" />

- Nmap found SSH on 22 and HTTP on 80.
- HTTP title hinted: “Follow the white rabbit.”

I checked the port 80:

<img width="1916" height="865" alt="image" src="https://github.com/user-attachments/assets/fc0a513c-a23d-4b39-8f72-bdda6ab2bdb8" />

## 2. Web Enumeration

```
feroxbuster -u 'http://10.49.151.224' -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-lowercase-2.3-medium.txt
```

<img width="1341" height="802" alt="image" src="https://github.com/user-attachments/assets/398ce243-7602-4868-b4e2-4bbae1ab309c" />

- Feroxbuster discovered:
```/img/
/r/
/r/a/
/r/a/b/
/r/a/b/b/
/poem/
```
- The `/r/a/b/b/` path suggested spelling “rabbit”, so I continued manually to:
`/r/a/b/b/i/t/`

<img width="1664" height="819" alt="image" src="https://github.com/user-attachments/assets/552cd1cd-40dc-413b-b285-6c1d5e259f1a" />

## 3. Finding Credentials

<img width="1298" height="390" alt="image" src="https://github.com/user-attachments/assets/80afed88-57e6-49dc-81da-0b1341602630" />

- Viewing the source of `/r/a/b/b/i/t/` revealed hidden credentials:
`alice:HowDothTheLittleCrocodileImproveHisShiningTail`

## 4. Initial Access
- Used the credentials to SSH as alice.

<img width="1342" height="596" alt="image" src="https://github.com/user-attachments/assets/92a46709-9a7f-46f1-84a5-07561333aec5" />

## 5. Alice to Rabbit

<img width="1344" height="180" alt="image" src="https://github.com/user-attachments/assets/3fd329ba-5853-400b-bfe2-d2c648ad7e3f" />

- `sudo -l` showed alice could run:
(rabbit) /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py

<img width="1336" height="340" alt="image" src="https://github.com/user-attachments/assets/6c6b1e02-5c42-4fbb-b27f-e1870f6f2cd0" />

- The script imported random.

<img width="1341" height="158" alt="image" src="https://github.com/user-attachments/assets/a7af25da-9e3f-4501-b1d5-98d28a457398" />

- Created a malicious random.py in `/home/alice` to hijack the import and spawn a shell as rabbit.

## 6. Rabbit to Hatter

<img width="1336" height="286" alt="image" src="https://github.com/user-attachments/assets/cc3d52a9-363c-4d6d-a481-47c1c7bf8337" />

- In /home/rabbit, found SUID binary teaParty.

<img width="1336" height="435" alt="image" src="https://github.com/user-attachments/assets/3c20efb1-86e4-4b25-9df9-a268d2611eee" />

- Running it showed it used date output.
- Created a fake date binary in /tmp and modified PATH.
- Executing teaParty spawned a shell as hatter.

## 7. Hatter to Root

<img width="1345" height="599" alt="image" src="https://github.com/user-attachments/assets/eff2c71f-a114-40c8-b9d4-f2e75411ff0a" />

- Found hatter’s password in /home/hatter/password.txt.
- getcap showed:
`/usr/bin/perl = cap_setuid+ep`

- Used Perl capability abuse to set UID to 0 and spawn a root shell.

## 8. Grab the Flags

<img width="1338" height="94" alt="image" src="https://github.com/user-attachments/assets/af7831ea-6d9f-4c70-b599-9df61deb6791" />


- user.txt was located at `/root/user.txt`
- root.txt was located at `/home/alice/root.txt`

## Key takeaway

Wonderland is a pretty classic room, and the steps became obvious from the outputs:

- /r/a/b/b/ → guessed /i/t/ because it spells rabbit 🐇
- HTML source → gave alice:password
- sudo -l → Python script imports random → module hijack
- teaParty SUID + printed date → PATH hijack with fake date
- getcap showed /usr/bin/perl cap_setuid+ep → perl to root

It was a good reminder that privilege escalation is not always one technique. 
This room chained together Python module hijacking, SUID PATH hijacking, and Linux capabilities. 
The flag locations were also intentionally swapped, so enumeration mattered even after getting root.
