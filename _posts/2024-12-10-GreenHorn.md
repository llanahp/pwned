---
date: 2024-12-10
categories: [Linux, Easy]
---

![greenhorn](https://github.com/user-attachments/assets/82138d6e-2c7f-4b3a-86fd-3717c751d086)

---

| **Created by** | **Page**     | **Difficulty** | **OS**  |
|-------------|--------------|----------------|---------|
| [nirza](https://app.hackthebox.com/users/800960)         | [Hack The Box](https://www.hackthebox.com/)     | Easy           | Linux   |

---









- I start with a scan of the open ports,
    
    ```bash
    nmap -p- --open -vvv -sS --min-rate 5000 -Pn 10.10.11.25 -oG scan
    ```
    
    ![image](https://github.com/user-attachments/assets/6f2f5e7c-24f4-4531-a59c-8d26f1f86fc3)

    
- I continue with a scan of the versions and technologies that are running on the open ports that we have found
    
    ```bash
    nmap -p22,80.3000 -sCV 10.10.11.25 -oN ports
    ```
    
    ![image 1](https://github.com/user-attachments/assets/fdda1208-e17c-4626-a382-549c4abb577e)

    
- I found the domain `greenhorn.htb` and add it to the file in the `/etc/hosts` file
- I look at what technologies they are using on the website.
    
    ```bash
    whatweb http://greenhorn.htb/
    ```
    
    ![image 2](https://github.com/user-attachments/assets/2fa50a6e-16be-47a2-a450-d7560a34e161)

    
    ![image 3](https://github.com/user-attachments/assets/1b0a5a1c-b03a-4e0c-bef7-0422a111d7ad)

    
- I find a panel to log in and little else. I look at the other site that uses port 3000 and I see that it is running a `Gitea`
- Looking at the GreenHorn repository files I find a hash apparently in sha512, when I run it through `crackStation` it tells me that the plaintext password of this hash is `iloveyou1`
    
    ![image 4](https://github.com/user-attachments/assets/fb67702a-f85c-46e6-af9e-1f344008d721)

    
    ![image 5](https://github.com/user-attachments/assets/17ee1c12-073b-456d-ad49-7247d9f8d0ca)

    
- Thanks to this password and searching for the Pluck version I find a vulnerability that allows to make a RCE. [Pluck v4.7.18 - Remote Code Execution (RCE)](https://www.exploit-db.com/exploits/51592)

- I use this exploit to create a revershell and manage to log in as www-data
    
    ![image 6](https://github.com/user-attachments/assets/9c11ea0c-44e5-4d2e-ba01-5bdf037fce88)

    
- I see that you exist as a junior user. I find in the junior directory a pdf that I can read, bring it to my place and open it.
    
    ![image 7](https://github.com/user-attachments/assets/95c82188-01b2-4130-a005-a080aaf5514a)

    
- I find a fuzzy password. After searching for various tools that can reverse this I found [Deprix](https://github.com/spipm/Depix)
    
    ![image 8](https://github.com/user-attachments/assets/90bee5b7-d7d6-4ebb-8a04-73c088dd57b3)

    
- To this tool we pass an image of the blur we want to reverse, using pdfimages, and a sample of the ones it has. After several different combinations I manage to get an image in which we can get to know the password. We try with “sidefromsidetheothersidesidefromsidetheotherside” and now we are root and we can read the second flag.
    
    ```jsx
    apt-get install poppler-utils
    pdfimages Usin.pdf output.png
    python3 depix.py -p  /home/rufo/Downloads/output.png-000.ppm -s images/searchimages/debruinseq_notepad_Windows10_closeAndSpaced.png
    ```
    
    ![image 9](https://github.com/user-attachments/assets/106e4e60-9d24-4fa6-82f3-dccd1a2dab22)

    
    ![image 10](https://github.com/user-attachments/assets/9f65f136-0e69-4bb3-a2bf-7f62d0e8d1bd)
