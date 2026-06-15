# Escalator Pentesting Report

## Target Information

- **Target IP:** `192.168.75.2`
- **Accessible services discovered:** FTP (`21`), SSH (`22`), HTTP (`80`)
- **Assessment type:** Authorized lab / ethical hacking exercise

---

## Walkthrough

### 1. Finding the target IP

The first step was to identify live hosts on the local network. The host discovery screenshot shows `arp-scan --localnet` being used and returning `192.168.75.2` as a live system. fileciteturn0file0L1-L8

Command used:

```bash
sudo arp-scan --localnet
```

### 2. Running an Nmap scan

After identifying the target, a full port and service scan was run using the command shown in the document. The scan identified:

- **Port 21/tcp**: FTP (`vsftpd`)
- **Port 22/tcp**: SSH (`OpenSSH 7.2p2 Ubuntu 4ubuntu2.10`)
- **Port 80/tcp**: HTTP (`Apache httpd 2.4.18`) fileciteturn0file0L9-L11

Command used:

```bash
nmap -sC -sV -p- -oN initial 192.168.75.2
```

This scan was important because it immediately showed three possible attack surfaces: FTP, SSH, and the web server.

### 3. Visiting the web application

Browsing to `http://192.168.75.2` displayed a page with the text:

- `HackMe Please!`
- `Local#1` fileciteturn0file0L1-L11

This suggested the host was intentionally vulnerable and likely part of a CTF-style or lab machine.

### 4. Testing FTP access

The next step was to test the FTP service. The screenshots show a successful connection to FTP and an **anonymous login** using the username `anonymous`. This confirmed that anonymous FTP access was enabled on the target. fileciteturn0file0L9-L11

Command used:

```bash
ftp 192.168.75.2
```

This was the first major weakness because anonymous FTP can expose internal files to unauthenticated users.

### 5. Discovering exposed web files

The screenshots then show browsing to:

```text
http://192.168.75.2/files/
```

The exposed directory listing contained:

- `life.c`
- `linp.sh`
- `linpeas.sh`
- `shell.php`
- `template.html` fileciteturn0file0L9-L11

This was a critical finding. Directory indexing was enabled, and highly sensitive files were exposed over HTTP. In particular:

- `linpeas.sh` is a well-known privilege escalation enumeration script.
- `shell.php` strongly suggests a web shell or server-side execution capability.
- The file listing indicated weak operational security and poor access control.

### 6. Getting a reverse shell

The next screenshot sequence shows that a reverse shell was obtained from the target, with a Netcat listener receiving an incoming connection from `192.168.75.2`. After connection, the shell prompt shows the compromised user context as `uid=33(www-data)` on the Ubuntu target. fileciteturn0file0L9-L11

Representative listener command:

```bash
nc -lvnp 4444
```

The evidence indicates that remote code execution was achieved through the web server, leading to an interactive shell as the Apache user `www-data`.

### 7. Enumerating for privilege escalation

Once shell access as `www-data` was established, local enumeration was performed. The screenshots show:

- A search for SUID binaries with:
  ```bash
  find / -user root -perm -4000 2>/dev/null
  ```
- Use of local file inspection in the filesystem
- Discovery of an `important.txt` file in a home directory
- The presence of a hidden script named `.runme.sh` fileciteturn0file0L9-L11

The `important.txt` output included a clue telling the tester to run a script to see the data. This led to inspection of the hidden script.

### 8. Recovering credentials from `.runme.sh`

The screenshots show the script contents being displayed. The script prints a message and reveals a value associated with user `shrek`. A separate screenshot then shows the web application returning the highlighted string plus the word `youaresmart`, which is then used as the SSH password for the `shrek` account. fileciteturn0file0L9-L11

This appears to have been the credential path:

- Username: `shrek`
- Password: `youaresmart`

This demonstrates another weakness: sensitive credential material or credential clues were retrievable from server-side files.

### 9. Logging in over SSH as `shrek`

Using the recovered password, SSH access was obtained as the user `shrek`. The screenshots clearly show a successful SSH login to the target using the `shrek` account. fileciteturn0file0L9-L11

Command used:

```bash
ssh -p 22 shrek@192.168.75.2
```

At this point, initial web access was converted into a legitimate shell for a local user, which made privilege escalation easier.

### 10. Checking sudo permissions

After logging in as `shrek`, `sudo -l` was run. The screenshots show that user `shrek` was allowed to execute:

```text
(root) NOPASSWD: /usr/bin/python3.5
``` fileciteturn0file0L9-L11

This is a serious misconfiguration because Python can be used to spawn a root shell.

Command used:

```bash
sudo -l
```

### 11. Escalating to root

The screenshots then show privilege escalation using Python to spawn `/bin/bash` as root:

```bash
sudo /usr/bin/python3.5 -c 'import pty; pty.spawn("/bin/bash")'
```

After this, the prompt changes to `root@ubuntu:/#`, confirming root access. The final screenshot shows `cat root.txt`, proving full system compromise and successful completion of the target. fileciteturn0file0L9-L11

### 12. Proof of root access

Proof shown in the document includes:

- A root shell prompt: `root@ubuntu:/#`
- Successful access to `root.txt` in the root context fileciteturn0file0L9-L11

---

## Vulnerabilities Identified

1. **Anonymous FTP enabled**
   - Allowed unauthenticated access to server content.

2. **Directory indexing enabled on the web server**
   - Exposed internal files directly through `/files/`.

3. **Sensitive files exposed over HTTP**
   - Included tooling and files that helped advance the attack.

4. **Remote code execution / web shell exposure**
   - The reverse shell evidence indicates server-side command execution was possible.

5. **Weak credential handling**
   - User credential clues were obtainable from scripts or related files.

6. **Overly permissive sudo configuration**
   - User `shrek` could run Python as root without a password.

---

## Remediation

### 1. Disable anonymous FTP access

If FTP is not required, disable it entirely. If it is required, anonymous access must be turned off.

Example in `vsftpd.conf`:

```conf
anonymous_enable=NO
```

Also restrict FTP to authorized users only and log all access attempts.

### 2. Disable directory listing on Apache

The `/files/` directory should not have been browsable. Disable directory indexing in Apache.

Example:

```apache
Options -Indexes
```

This prevents users from seeing file listings when no index page exists.

### 3. Remove sensitive files from the web root

Files such as enumeration scripts, shell files, and internal notes should never be stored in directories accessible from the web server. Move administrative files outside the document root and enforce strict file permission policies.

### 4. Investigate and remove web shell / command execution paths

Any uploaded or planted shell file such as `shell.php` must be removed immediately. Review:

- file upload functionality
- writable web directories
- PHP execution paths
- Apache logs
- suspicious outbound connections

Deploy file integrity monitoring and alerting for unexpected web-accessible scripts.

### 5. Rotate compromised credentials

The `shrek` account password should be reset immediately, and all user passwords should be reviewed for strength and reuse. Secrets must not be embedded in scripts or retrievable from files in user-accessible locations.

### 6. Harden sudo permissions

Remove dangerous NOPASSWD rules unless absolutely required. In this case, allowing Python as root effectively grants full root access.

Avoid entries like:

```text
(root) NOPASSWD: /usr/bin/python3.5
```

Use narrowly scoped administrative commands instead of interpreters like Python, Perl, Bash, or editors.

### 7. Apply least privilege and file permission controls

Review ownership and permissions for:

- user home directories
- hidden scripts
- web content
- administrative tooling

Sensitive helper scripts should not be world-readable or exposed to lower-privileged users.

### 8. Monitor and audit privilege escalation paths

Run regular audits for:

- SUID binaries
- sudoers entries
- exposed scripts
- world-readable secrets
- unexpected files in web directories

---

## Vulnerability Report Email

**Subject:** Critical vulnerabilities identified on 192.168.75.2

**To:** System Owner / Security Team

Hello,

During an authorized security assessment of host `192.168.75.2`, I identified multiple vulnerabilities that allowed complete compromise of the system, including root-level access.

### Summary

The target exposed several critical weaknesses: anonymous FTP access, directory listing of sensitive web files, evidence of remote code execution through the web server, recoverable user credentials, and an insecure sudo configuration that allowed passwordless execution of Python as root. By chaining these issues together, I was able to obtain a shell as `www-data`, pivot to the `shrek` user over SSH, and escalate privileges to root. fileciteturn0file0L9-L11

### Steps to Reproduce

1. Discover the host on the local network using:
   ```bash
   sudo arp-scan --localnet
   ```

2. Enumerate open ports and services:
   ```bash
   nmap -sC -sV -p- -oN initial 192.168.75.2
   ```

3. Confirm exposed services:
   - FTP on port 21
   - SSH on port 22
   - HTTP on port 80

4. Browse to the web application and identify the exposed `/files/` directory:
   ```text
   http://192.168.75.2/files/
   ```

5. Confirm anonymous FTP access:
   ```bash
   ftp 192.168.75.2
   ```

6. Use the exposed web content to gain remote code execution and catch a reverse shell as `www-data`.

7. Perform local enumeration and discover the hidden script `.runme.sh` and related credential clues.

8. Recover the `shrek` account password (`youaresmart`) and log in via SSH:
   ```bash
   ssh shrek@192.168.75.2
   ```

9. Run:
   ```bash
   sudo -l
   ```

10. Exploit the sudo misconfiguration:
    ```bash
    sudo /usr/bin/python3.5 -c 'import pty; pty.spawn("/bin/bash")'
    ```

11. Confirm root access and read `root.txt`. fileciteturn0file0L9-L11

### Impact

The impact is **critical**. An attacker could gain full control of the system. Specifically, the issues allow:

- unauthenticated access to server content
- discovery of sensitive internal files
- remote code execution on the web server
- theft or recovery of user credentials
- SSH access as a local user
- full privilege escalation to root

This level of compromise allows an attacker to read, modify, or delete data; install persistence; pivot to other systems; and fully control the host. fileciteturn0file0L9-L11

### Proof of Root Access

Proof of compromise was demonstrated by obtaining a root shell and reading `root.txt` from the target system. The screenshots show the shell prompt changed to `root@ubuntu:/#` and the root flag file was successfully accessed. fileciteturn0file0L9-L11

Please treat this issue as urgent and remediate all linked weaknesses rather than addressing them in isolation.

Regards,  
Security Tester

---

## Ethical Hacking Report

### The importance of obtaining proper authorization before testing

Security testing must only be performed with clear, explicit authorization from the owner of the system. Authorization defines the permitted scope, protects both the tester and the organization, and ensures that the activity is understood as legitimate defensive work rather than unauthorized access. Without authorization, the exact same technical actions used in a pentest could be illegal and unethical.

In a professional setting, authorization should ideally be written and should clearly define:

- which hosts and applications are in scope
- what testing methods are allowed
- what hours testing may occur
- who should be contacted in case of service disruption
- whether exploitation and privilege escalation are permitted

### The legal and ethical boundaries of vulnerability testing

Even with permission, ethical hacking has boundaries. A tester should only assess systems that are in scope and should avoid unnecessary damage. The goal is to identify risk, not to cause harm.

A responsible tester should:

- avoid destructive actions unless explicitly approved
- collect only the minimum evidence needed to prove the issue
- avoid modifying or deleting production data
- avoid persistence mechanisms unless specifically authorized
- stop and report immediately if testing affects availability or data integrity
- avoid accessing third-party or unrelated systems discovered during the test

In this case, moving from enumeration to exploitation to privilege escalation is acceptable only because the assessment is understood as an authorized lab exercise.

### How to report vulnerabilities responsibly and avoid causing harm

Responsible reporting means communicating findings clearly, accurately, and privately to the system owner or security team. A good report should explain:

- what the vulnerability is
- how it was discovered
- how it can be reproduced
- what the business or technical impact is
- how it can be fixed

The report should include enough evidence to validate the issue, but it should not unnecessarily expose secrets, credentials, or exploit details beyond what is needed for remediation. Sensitive findings should be shared only with authorized recipients.

To avoid causing harm, the tester should:

- use the least intrusive method that still proves the issue
- avoid public disclosure before remediation
- protect any collected evidence
- recommend practical remediations
- support remediation validation after fixes are applied

Ethical hacking is valuable because it helps organizations improve security before a real attacker can exploit the same weaknesses. That value depends on discipline, authorization, accuracy, and responsible disclosure.

---

## Conclusion

This assessment demonstrated a full compromise chain on `192.168.75.2`:

1. discover host
2. enumerate services
3. identify anonymous FTP and exposed web files
4. obtain a reverse shell as `www-data`
5. enumerate locally and recover credentials
6. SSH in as `shrek`
7. abuse `sudo` rights on Python
8. escalate to root

The machine was compromised because multiple small and large weaknesses existed at the same time. Fixing only one issue would reduce risk, but the correct response is full hardening across FTP, web exposure, credentials, permissions, and sudo policy.
