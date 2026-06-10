<img width="1749" height="385" alt="image" src="https://github.com/user-attachments/assets/0b2b1739-60c7-46cf-b9ef-18f86e77a89f" />

# Tryhackme - Lookback

The Lookback company has just started the integration with Active Directory. Due to the coming deadline, the system integrator had to rush the deployment of the environment. Can you spot any vulnerabilities?

Sometimes to move forward, we have to go backward. So if you get stuck, try to look back!

### Start with Nmap

<img width="1345" height="603" alt="image" src="https://github.com/user-attachments/assets/f018aab9-d6e8-46ff-8af9-2a84663818c2" />

Important clues from your scan:
```
80    HTTP      Microsoft IIS 10.0
443   HTTPS     Outlook / OWA
3389  RDP       Microsoft Terminal Services
Hostname: WIN-12OU7A66M7.thm.local
```

First add the hostname properly:
```
sudo nano /etc/hosts
```
Add:
```
10.49.138.119 WIN-12OU7A66M7.thm.local WIN-12OU7A66M7 lookback.thm
```

I checked port 80 but there was nth there, so I check port 443 `https://WIN-12OU7A66M7.thm.local/owa`
 
<img width="1913" height="861" alt="image" src="https://github.com/user-attachments/assets/f0d88cdb-5ca2-480a-be7a-4fac705abbaa" />

That’s the OWA login page. Good sign — Exchange/Outlook is definitely alive. 🔥

After discovering the OWA/Exchange login, I tested a small set of common weak credentials. The pair `admin:admin` was accepted, confirming weak credential usage. However, this account did not have mailbox or RDP access, so I continued enumerating other HTTPS paths.

Nikto also identified/confirmed the weak credential issue, which helped validate that admin:admin was part of the intended path rather than only a lucky guess.

<img width="1915" height="865" alt="image" src="https://github.com/user-attachments/assets/9f8c7f8a-bd96-4e9b-b0e3-75c3f024a22c" />

It does not say “wrong password.” It says:

`A mailbox couldn't be found for THM\admin`

That usually means:

`THM\admin exists and the password worked,
but this account has no Exchange mailbox.`

So `admin:admin` may actually be valid domain creds, just not valid for OWA mailbox access.























