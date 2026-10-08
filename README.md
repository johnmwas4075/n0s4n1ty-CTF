# File upload Vulnerability - CTF Write-up

## Overview

**Challenge:** n0s4n1ty
**Platform:** CyLab
**Category:** Web Security / File Upload / Privilege Escalation
**Objective:** Exploit an unrestricted file upload vulnerability to obtain command execution and retrieve the flag.

---

## 1. Initial Reconnaissance

The challenge presented a web application with a file upload functionality.

The application appeared to be designed for uploading image files. After uploading a file, the application redirected to:

```text
/uploads
```
The `/uploads` page displayed the location of the uploaded file.
<img width="692" height="65" alt="image" src="https://github.com/user-attachments/assets/6e85603b-3719-478a-b7e3-f0a72f34a1fd" />
I investigated whether the application properly validated the type of file being uploaded.

---

## 2. Testing the File Upload

The application did not appear to perform adequate validation of the uploaded file type.

Since the server accepted files without properly restricting their extensions, I tested whether it was possible to upload a PHP file instead of an image.

I created a PHP file containing a simple script and uploaded it through the application's upload functionality.

The application accepted the file and made it available under the `/uploads` directory.

This indicated an **unrestricted file upload vulnerability**.

<img width="692" height="65" alt="image" src="https://github.com/user-attachments/assets/5300d017-a676-4ae4-bfb7-486342dce584" />
---

## 3. Accessing the Uploaded PHP File

After the upload was successful, I navigated to the uploaded file:

```text
/uploads/exploit.php
```

Instead of treating the file as an ordinary uploaded document, the server processed it as a PHP script.

This gave me the ability to execute commands on the server through the uploaded script.

### Screenshot

<img width="1000" height="127" alt="image" src="https://github.com/user-attachments/assets/d0d284ea-6f3d-4d78-80ce-d251d41844c3" />


```text
![Uploaded PHP file](images/exploit.php)
```

---

# 4. Obtaining Command Execution

The ability to execute the uploaded PHP file effectively provided server-side command execution.

I used this access to investigate the privileges of the web server account.

The account executing the PHP script was:

```text
www-data
```

I then checked which commands this account could execute with `sudo`.

---

## 5. Privilege Enumeration

I executed:

```bash
sudo -l
```

The server returned:

```text
Matching Defaults entries for www-data on challenge:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User www-data may run the following commands on challenge:
    (ALL) NOPASSWD: ALL
```

The important finding was:

```text
(ALL) NOPASSWD: ALL
```

This indicated that the `www-data` account could execute commands as any user through `sudo` without requiring a password.

This represented a severe **privilege escalation / excessive sudo privilege** vulnerability.
<img width="889" height="168" alt="image" src="https://github.com/user-attachments/assets/4f1be934-8be7-4249-9502-5e95d3d0e50a" />


---

# 6. Retrieving the Flag

Since the `www-data` account had unrestricted sudo privileges, I was able to access files belonging to the root user.

I executed:

```bash
sudo cat /root/flag.txt
```

The contents of the file contained the challenge flag.
<img width="807" height="224" alt="image" src="https://github.com/user-attachments/assets/307447c8-6acf-47b0-a33e-ea8e11ab035a" />


---

# 7. Attack Chain

The complete attack path was:

```text
File Upload Functionality
          ↓
File Type Not Properly Validated
          ↓
Upload Malicious PHP File
          ↓
Access /uploads/exploit.php
          ↓
PHP Code Execution
          ↓
Command Execution as www-data
          ↓
sudo -l
          ↓
www-data has NOPASSWD: ALL
          ↓
Privilege Escalation
          ↓
Read /root/flag.txt
          ↓
Capture Flag
```

---

# 8. Vulnerability Analysis

The challenge involved multiple security weaknesses that could be chained together.

### 8.1 Unrestricted File Upload

The application accepted a PHP file despite apparently expecting image uploads.

A secure application should validate uploaded files using multiple mechanisms, including:

* Allowlisting permitted file extensions
* Validating the actual file type/MIME type
* Inspecting file contents
* Renaming uploaded files
* Storing uploads outside the executable web root
* Preventing uploaded files from being executed as server-side code

Simply checking the filename extension is also insufficient by itself.

---

### 8.2 Remote Code Execution

Because the uploaded PHP file was placed in a web-accessible directory and interpreted by the server, accessing:

```text
/uploads/exploit.php
```

resulted in execution of the PHP code.

This transformed the file upload vulnerability into server-side command execution.


---

### 8.3 Excessive Sudo Privileges

The `sudo -l` output showed:

```text
(ALL) NOPASSWD: ALL
```

for `www-data`.

This granted the web server account unrestricted sudo privileges without authentication.

From a security perspective, this is extremely dangerous because compromise of the web application would immediately provide a path to privileged system access.

---

# 9. Key Security Lessons

### 1. File uploads must be properly restricted

Applications should not allow users to upload arbitrary executable files when the intended functionality is limited to images or documents.

### 2. Uploaded files should not be executable

Even if an attacker manages to upload a malicious file, the server should be configured so that user-uploaded content cannot be interpreted as executable server-side code.

### 3. Web server accounts should have minimal privileges

Accounts such as `www-data` should operate with the minimum permissions required for the application.

### 4. Avoid unrestricted sudo permissions

Granting:

```text
NOPASSWD: ALL
```

to a web server account effectively removes an important security boundary.

### 5. Vulnerabilities can be chained

The challenge demonstrated how several weaknesses can combine:

```text
Unrestricted Upload
        +
PHP Execution
        +
Excessive Sudo Privileges
        =
Privileged Server Access
```

Individually, each weakness is serious. Combined, they resulted in complete compromise of the challenge environment.

---

# 10. Methodology Summary

| Stage                 | Action                                 | Result                           |
| --------------------- | -------------------------------------- | -------------------------------- |
| Reconnaissance        | Investigated file upload functionality | Upload endpoint identified       |
| File Upload Testing   | Uploaded PHP file                      | File accepted                    |
| File Execution        | Accessed `/uploads/exploit.php`        | PHP code executed                |
| Command Execution     | Executed commands through the script   | Server access obtained           |
| Privilege Enumeration | Ran `sudo -l`                          | `www-data` had unrestricted sudo |
| Privilege Escalation  | Used available sudo privileges         | Root-level access obtained       |
| Flag Retrieval        | Read `/root/flag.txt`                  | Flag captured                    |

---

# 12. Conclusion

The **n0s4n1ty** challenge demonstrated how an unrestricted file upload vulnerability can be escalated into full server compromise when combined with excessive system privileges.

The initial vulnerability was identified by testing whether the application's upload functionality properly validated file types. Because PHP files were accepted and stored in a web-accessible directory, I was able to upload and execute a PHP script.

This provided command execution as the `www-data` user. I then performed privilege enumeration using:

```bash
sudo -l
```

The resulting configuration showed:

```text
(ALL) NOPASSWD: ALL
```

for `www-data`, meaning the web server account had unrestricted sudo privileges. I then used the elevated privileges to read:

```text
/root/flag.txt
```

and retrieve the flag.

### The main lesson from this challenge is that **secure file upload controls, proper separation of uploaded content from executable code, and the principle of least privilege are all critical to preventing web application compromise from escalating into full system access**.
