<img width="1099" height="245" alt="image" src="https://github.com/user-attachments/assets/9ead036c-f133-488b-951d-085309ec0865" />
<br>

Marco Bianchi, a systems administrator, recently deployed several internal services, including a firewall console, employee portal, social platform, and SSH access to critical infrastructure. Due to tight deadlines and operational pressure, Marco reused weak, predictable, and pattern-based passwords across multiple systems.  

Your objective is to conduct a password security assessment to identify weaknesses in Marco’s authentication practices.

<img width="1893" height="830" alt="image" src="https://github.com/user-attachments/assets/0325dee9-4ef2-4789-b33e-48e3653af2b4" />

# What is the password for Level 1?

<img width="1886" height="849" alt="image" src="https://github.com/user-attachments/assets/3606281a-cf7d-4e0b-8b2e-866e6bc906e3" />

<br>

```
hydra -l admin \
-P /usr/share/seclists/Passwords/Common-Credentials/10k-most-common.txt \
firewall.thm -s 5001 \
http-post-form "/login:username=^USER^&password=^PASS^:F=Invalid credentials."
```

<img width="1252" height="323" alt="image" src="https://github.com/user-attachments/assets/bc09ff24-5ce7-4fc1-b149-9b55303af8c9" />

Result:

```
12345
```
## What is the password for Level 2?

<img width="1639" height="360" alt="image" src="https://github.com/user-attachments/assets/57502cbd-fb17-40a5-9d63-23d8a677b7cf" />

<img width="1438" height="848" alt="image" src="https://github.com/user-attachments/assets/62b975b0-606c-4e43-a75d-e194ed1edaa1" />
<br>

Then go to `Employee Login` at the top-right-corner of the page:

<img width="1449" height="635" alt="image" src="https://github.com/user-attachments/assets/217a3e85-c0a5-4f23-a869-89d6942b0019" />

THM mentioned Marco built an internal Employee Login panel on jobs.thm:5002 and used common company keywords as passwords, So we take a look at jobs.thm:5002 and use Hydra with a custom keyword list from the page.

Create:

```
cat > company.txt <<EOF
innovation
excellence
security
digital
cloud
future
talent
security123
innovation123
cloud123
digital123
future2025
EOF
```

Then:

```
hydra -l marco -P company.txt jobs.thm -s 5002 \
http-post-form "/login:username=^USER^&password=^PASS^:F=Invalid Credemtials"
```
<img width="1251" height="325" alt="image" src="https://github.com/user-attachments/assets/5e7d0a0f-c684-4ccf-87e5-189e81451a92" />

## What is the password for Level 3?

<img width="1589" height="334" alt="image" src="https://github.com/user-attachments/assets/67d7608a-3ce4-43d2-8f1a-d12cbcdee928" />

<img width="1455" height="846" alt="image" src="https://github.com/user-attachments/assets/981919e9-08a7-4ecd-a2d1-b3300e4cdf3c" />
<br>

The challenge hint suggested deriving Marco’s password using personal information from `jobs.thm:5002`. 

<img width="1460" height="532" alt="image" src="https://github.com/user-attachments/assets/c0e42d48-dd21-4b2c-a89f-d779bc779c20" />

After checking the employee profile using credentials from Level 2 task, the following details were identified:

- First Name: Marco
- Surname: Bianchi
- Nickname: marky
- Birthdate: 14021995

Initial attempts involved manually testing common password patterns such as combinations of the nickname, surname, and birth year (e.g., `marky1995!`, `Marco140295`). However, these attempts were unsuccessful.

To automate password generation based on personal information, the `CUPP` (Common User Passwords Profiler) tool was used. The collected details were supplied to CUPP in order to generate a targeted wordlist containing realistic password combinations.

```
python3 cupp.py -i
```

<img width="1247" height="422" alt="image" src="https://github.com/user-attachments/assets/c29c60f6-6c62-4f4f-866f-ac22dc7ccf37" />
<br>

Just press Enter for all unknown fields like 
- partner
- child
- pet
- company

<img width="843" height="213" alt="image" src="https://github.com/user-attachments/assets/bd672be0-6cd7-48bd-bdaa-2ba99f6339b1" />
<br>

Wait few minutes for it to be generated and the wordlist was then used with Hydra to brute-force the login form on `social.thm:5003`.

```bash
hydra -l marco -P marco.txt social.thm -s 5003 http-post-form "/login:username=^USER^&password=^PASS^:Invalid credentials"
```

<img width="1248" height="288" alt="image" src="https://github.com/user-attachments/assets/75432d27-852d-45a9-bc89-ca9a68bf330c" />

The attack successfully identified the valid credentials:

- Username: `marco`
    
- Password: `Bianchi2495`

One important observation was that the username was case-sensitive and required `marco` instead of `Marco`.

<img width="1856" height="841" alt="image" src="https://github.com/user-attachments/assets/d1cde300-862c-45fe-8a8f-66c462ffa918" />
<br>

## What is the password for Level 4?

<img width="1623" height="401" alt="image" src="https://github.com/user-attachments/assets/7c27c89c-37ed-40f2-b389-8fd81fd2c265" />
<br>

The challenge description stated that uploaded files were automatically renamed using the SHA256 hash of the original filename and stored as `(SHA256).png`.

After logging into `social.thm:5003` using Marco’s credentials, the profile page was inspected. Inspecting the `<img>` tag for Marco’s avatar revealed the following hashed filename:

<img width="1891" height="845" alt="image" src="https://github.com/user-attachments/assets/7e083a0a-9f62-43ad-9850-e87899a8903a" />
<br>

```text
d34a569ab7aaa54dacd715ae64953455d86b768846cd0085ef4e9e7471489b7b.png
```

The hash value identified as SHA256. A dictionary attack was then performed using Hashcat with SHA256 mode (`-m 1400`) and the `rockyou.txt` wordlist.

```bash
hashcat -m 1400 level4_hash.txt /usr/share/wordlists/rockyou.txt
```

<img width="1252" height="84" alt="image" src="https://github.com/user-attachments/assets/f2d4ace6-5b39-430d-a392-f4ab6cb2d35c" />
<br>

Successfully cracked the hash and recovered the original filename:

```text
family
```

#### What is the password for Level 5?

<img width="1578" height="354" alt="image" src="https://github.com/user-attachments/assets/10256b9d-96ae-4292-b098-a1ac92c382a6" />
<br>

Marco’s social profile revealed a post describing his password pattern:

<img width="647" height="254" alt="image" src="https://github.com/user-attachments/assets/438c36e5-3f60-4eb5-b87e-ab1cdc8e91ca" />
<br>

Marco is referring to THIS pattern: `Capitalize CompanyKeyword + Number/Year + !`

<img width="841" height="339" alt="image" src="https://github.com/user-attachments/assets/39cab64a-ce7e-434a-8120-7e901e7a51ad" />
<br>

The visible company keywords on the post were:

- `security`
- `excellence`
- `innovation`
- `digital`
- `cloud`
- `future`
- `talent`

Based on this pattern, a small targeted wordlist was created using the keywords with the year 2024 and an exclamation mark. If 

```
cat > ssh_wordlist.txt << EOF
Security2024!
Excellence2024!
Innovation2024!
Digital2024!
Cloud2024!
Talent2024!
Future2024!
EOF
```

The SSH service was then brute-forced using Hydra with the username `marco`.

```
hydra -l marco -P ssh_wordlist.txt ssh://social.thm
```

<img width="1256" height="289" alt="image" src="https://github.com/user-attachments/assets/7f4a8dbc-1dfc-4031-93c4-c60b9a1742b2" />

<br>

The valid SSH password was found to be:

```
Security2024!
```

This credential was confirmed by successfully logging in through SSH as Marco:

```
ssh marco@social.thm
>>>> password: Security2024!
```

### Key Takeaways / Lessons Learned

- Personal information can be used to generate targeted password wordlists.
    
- Weak password patterns based on names, dates, and common keywords are highly predictable.
    
- `CUPP` can automate OSINT-based password generation using victim details.
    
- Hydra can brute-force web login forms and SSH services using targeted wordlists.
    
- Browser developer tools can reveal hidden resources such as uploaded file paths and hashed filenames.
    
- Uploaded files renamed with hashes may still leak information if predictable filenames are used.
    
- Small targeted wordlists are often more effective than huge random brute-force attempts.
    
- Real-world password habits reused across services can lead to credential compromise.

