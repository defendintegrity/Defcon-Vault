# Diagnosing Connection Failures with Verbose SSH Debugging

Verbose debugging is the primary diagnostic tool for inspecting the SSH handshake process. When a connection fails, the error messages returned to the terminal—often generic reports like `Permission denied` or `Connection reset`—are intentionally opaque to prevent information leakage. By passing the `-v` (verbose) flag to the `ssh` client, you force it to output the internal state of the cryptographic negotiation, identity exchange, and authentication attempt.

### Triggering Diagnostic Output

The `-v` flag increases verbosity incrementally. Using `-v` provides a basic overview of the connection phase, while `-vv` or `-vvv` provides granular detail on every packet sent and received.

bash

```
# Basic diagnostic outputssh -v user@devops-pro-server# Maximum detail for deep-dive troubleshootingssh -vvv user@devops-pro-server
```

When diagnosing, `-vvv` is the standard. It logs exactly which files the client is reading, which public keys it is offering, and, crucially, how the server responds to each offer.

### Interpreting the Authentication Sequence

The core of a failed connection usually hides in the `debug1: Authentications that can continue` section. The SSH client negotiates methods in a specific order: `publickey`, `gssapi-keyex`, `gssapi-with-mic`, `password`, and so on.

A successful key-based authentication flow follows this pattern:

1. **Offer**: The client identifies a key file (e.g., `~/.ssh/id_ed25519`).
2. **Signature**: The client signs a challenge from the server using the private key.
3. **Acceptance**: The server confirms the signature matches the public key in `authorized_keys`.

If the connection fails, watch for these specific indicators:

- **"Offering public key: [path]":** If the key file you expect is not listed here, the client is either not configured to use it or lacks the required file permissions, causing the SSH agent to ignore it.
- **"Authentications that can continue: publickey,password":** If the list does _not_ include `publickey` but does include `password`, the server-side SSH daemon has likely been configured to disable public key authentication entirely (`PubkeyAuthentication no`).
- **"debug1: Next authentication method: publickey":** If the process hangs or drops immediately after this line, the server is likely rejecting the key before the signature check even begins. This is almost always due to incorrect ownership (e.g., non-root user owning the `.ssh` folder) or insecure permissions (e.g., `chmod 777 ~/.ssh`).

### Visualizing the Handshake Failure

The following sequence illustrates a common failure where the client offers a key, but the server rejects it due to invalid permissions or a missing entry in the `authorized_keys` file.

SSH_MSG_KEXINIT (Start Handshake)Key Exchange CompleteSSH_MSG_USERAUTH_REQUEST (Attempt publickey)SSH_MSG_USERAUTH_FAILURESSH_MSG_USERAUTH_REQUEST (Fallback to password)Prompt for passwordClient logs: "debug1: Authentications that can continue: publickey,password"SSH ClientSSH Daemon (Server)

### Analyzing Client-Side Configuration

When debugging, realize that your local `~/.ssh/config` file may be redirecting your connection or forcing the use of incorrect identities. When running with `-v`, look for the line:

`debug1: Reading configuration data /home/user/.ssh/config`

If this line appears, watch the subsequent output for `Applying options for...`. If you see `IdentityFile` being set to a key you aren't intending to use, that is your point of failure. You can override your config settings during a debug session to isolate the issue:

bash

```
# Ignore existing config files to test raw connectivityssh -vvv -F /dev/null user@devops-pro-server
```

If the connection succeeds with `-F /dev/null` but fails normally, your `~/.ssh/config` file is the culprit.

### Exercises

1. Perform a verbose connection to your project server. Identify the specific lines that confirm your public key was accepted.
2. Temporarily set your `~/.ssh` directory permissions to `777` and run a verbose connection attempt. Observe how the server logs (or client output) change regarding "refused" keys. Reset permissions to `700` immediately after.
3. Use the `-F /dev/null` flag to test if your local SSH configuration is masking the correct identity file.

### Summary

Verbose debugging turns a "black box" connection failure into a step-by-step audit log. By examining the exchange of authentication methods and verifying that the correct keys are being offered by the client, you can isolate whether the failure is local configuration, file permissions, or server-side policy. This diagnostic capability is essential before investigating server-side daemon logs.

# Identifying Weak or Compromised Key Pairs in Production

SSH key compromise is rarely a result of a brute-force attack on the cryptographic primitive itself. Instead, it occurs through poor lifecycle management: orphaned keys, weak entropy during generation, or insecure storage on local machines. Identifying these risks requires moving from passive management to proactive auditing of the `authorized_keys` files and local identity pools.

## Auditing Key Entropy and Algorithm Weakness

The security of an SSH key is fundamentally tied to the algorithm used and the entropy of the generation process. Using legacy algorithms like RSA with insufficient bit lengths (e.g., 1024 or 2048 bits) or outdated digital signature algorithms like DSA makes keys susceptible to precomputed attacks.

You can audit the strength of public keys by extracting their fingerprints and identifying the algorithm type. On a Linux host, loop through all `authorized_keys` files to identify non-compliant keys:

bash

```
# Locate all authorized_keys files and inspect the key typefind /home -name "authorized_keys" -exec ssh-keygen -lf {} \; 2>/dev/null | awk '{print $2, $3}'
```

Output showing `ssh-rsa` with keys below 3072 bits or any usage of `ssh-dss` signals an immediate rotation requirement. Modern production standards require the transition to `Ed25519` for all new and re-issued identities, as it is immune to many side-channel attacks that target RSA padding and offer superior performance.

## Identifying Orphaned and Stale Access

An "orphaned" key is one that remains in an `authorized_keys` file long after the user or the automated service that generated it has left the organization. These keys are the primary target for attackers because they are often "forgotten" and therefore unmonitored.

To identify stale access, you must correlate file metadata with access logs. The `sshd` daemon logs key fingerprints upon successful authentication. By comparing the `ssh-keygen -lf` output of existing keys against the `lastlog` or `/var/log/auth.log` files, you can identify keys that have not been used in a specific timeframe (e.g., 90 days).

YesNoExtract all public keys from authorized_keysGenerate unique fingerprint for each keyParse /var/log/auth.log for successful loginsKey in logs within 90 days?Keep key/Mark as activeFlag key as stale/OrphanedInitiate rotation or revocation process

## Detecting Unauthorized Modification via Fingerprint Mismatch

A subtle form of compromise involves the manual injection of an unauthorized public key into an existing `authorized_keys` file. Because `authorized_keys` is a simple text file, an attacker with sufficient permissions can append their own key to it, gaining persistent, low-profile access.

The most effective way to detect this is to maintain a version-controlled "Source of Truth" for your authorized keys. You can automate the validation by running a periodic script that compares the current state of a server's `authorized_keys` against your master repository:

1. **Export:** Extract the current `authorized_keys` from the production host.
2. **Sort:** Sort the keys alphabetically (the file order does not impact functionality).
3. **Compare:** Use `diff` to identify any keys present on the server that do not exist in your authorized repository.

bash

```
# Example logic for comparing remote keys to local source of truthcat ~/.ssh/authorized_keys | sort > /tmp/remote_keys.sortedcat ./trusted_repo/keys.pub | sort > /tmp/local_repo.sorteddiff /tmp/remote_keys.sorted /tmp/local_repo.sorted
```

Any discrepancy here is a high-severity security event. It indicates either unauthorized access or an out-of-band management practice that bypasses established security controls.

## Analyzing SSH Agent Exposure

Local compromise is often facilitated by SSH agents. If a developer uses `ssh-add -K` (on macOS) or a long-lived agent session on a jump box, a process running with the same user privileges can interact with the agent socket to authenticate as the user without ever knowing the private key passphrase.

Check the active connections to your agent socket:

bash

```
# Check who owns the agent socket filels -l $SSH_AUTH_SOCK# Check for unauthorized processes accessing the socketlsof $SSH_AUTH_SOCK
```

If you detect suspicious processes attached to the agent socket, assume the identity is compromised. The immediate response must be to kill the agent process, remove the identity, and rotate the key pair, as the private key data itself may have been extracted via memory dumping.

## Summary

Auditing production keys is an exercise in identifying gaps between your policy and the reality of the disk. Use `ssh-keygen` to enforce algorithm standards, cross-reference authentication logs to identify stale access, and use repository-based diffing to catch unauthorized key injections. In the next section, we will cover how to operationalize this by writing scripts that perform these checks and auto-cleanup of identified risks.

# Implementing Automated Key Scanning and Cleanup Scripts

Automated key management prevents "key sprawl"—the accumulation of orphaned, unused, or unauthorized public keys within `authorized_keys` files. As infrastructure scales, manual auditing becomes impossible, leading to security debt where decommissioned user accounts or compromised developer machines retain persistent access to your production environment.

### Identifying Stale Keys

The primary strategy for cleanup is comparing the active list of authorized keys against a "Source of Truth," such as an LDAP directory, an IAM user list, or a central configuration management repository (like Ansible or SaltStack).

Before deleting, you must differentiate between keys currently in use and those that are merely present. You can track actual usage by monitoring the `authorized_keys` file's interaction with the SSH daemon. Because standard SSH logs do not explicitly map a successful login to a specific line in `authorized_keys`, you must implement a "last-seen" mechanism. A common approach involves using `command` restrictions in your `authorized_keys` file to log key fingerprints upon authentication.

### Designing the Cleanup Script

A robust cleanup script operates in three phases: Discovery, Verification, and Pruning.

1. **Discovery**: Scrape all `~/.ssh/authorized_keys` files from designated targets.
2. **Verification**: Compare extracted keys against your identity provider.
3. **Pruning**: Remove entries that do not match current, active employee or service identities.

The following Python snippet demonstrates how to parse a key file and identify fingerprints.

python

```
import subprocessdef get_key_fingerprint(key_line):    """Calculates the fingerprint of an SSH public key string."""    # We pass the key through ssh-keygen -lf to get the fingerprint    process = subprocess.Popen(        ['ssh-keygen', '-lf', '/dev/stdin'],        stdin=subprocess.PIPE,        stdout=subprocess.PIPE,        stderr=subprocess.PIPE,        text=True    )    stdout, stderr = process.communicate(input=key_line)    return stdout.split()[1] if stdout else None# Example usage:key_content = "ssh-ed25519 AAAAC3Nza... user@workstation"print(f"Fingerprint: {get_key_fingerprint(key_content)}")
```

### Implementing Automated Scanning

Rather than running scripts ad-hoc, integrate scanning into your CI/CD pipeline or as a cron-based compliance job. The scanning logic should treat the `authorized_keys` file as an immutable artifact managed by automation—if a key exists on the server but is not in your repository, the script should automatically comment it out or delete it.

YesNoFetch current authorized_keys from serverParse list of fingerprintsIs fingerprint in Central Registry?Keep key activeArchive key to backupRemove entry from authorized_keysLog audit event to SIEM

### Handling False Positives and Emergencies

Aggressive automation can lead to self-inflicted lockouts. Always implement these safety measures:

- **Dry-run mode**: The script must support a flag that logs intended changes without modifying files.
- **Emergency override**: Maintain at least one "break-glass" key that is explicitly excluded from the cleanup logic.
- **Backup**: Always back up the current `authorized_keys` file to a secure directory (e.g., `/var/backups/ssh_keys/`) before applying changes.

### Exercises

1. **Develop a Parser**: Write a script that reads a local `authorized_keys` file and outputs a JSON list of all public key fingerprints found within it.
2. **Implement Dry-Run**: Add a command-line argument to your script that prints "Would remove key: [fingerprint]" instead of executing the removal.
3. **Simulate a Registry**: Create a local file `active_keys.txt` containing trusted fingerprints and configure your script to identify keys in `authorized_keys` that are missing from `active_keys.txt`.

### Summary

Automated cleanup turns security from a periodic manual task into a continuous background process. By ensuring that only verified keys remain in your production environment, you significantly reduce the attack surface. This process assumes that you have already established a centralized identity management system, which we will build upon as we look toward preventing brute-force access in the final sections of our security hardening strategy.

# Securing the SSH Daemon Against Brute Force Attacks

Brute force attacks on SSH rely on automated scripts attempting to guess credentials or exploit weak configurations by cycling through common usernames and passwords. When you rely solely on public-key authentication, you significantly reduce the surface area for these attacks, but the `sshd` process remains exposed to connection attempts that consume CPU, memory, and log space.

### Hardening the SSH Daemon Configuration

The `/etc/ssh/sshd_config` file is the control center for your daemon's security posture. To defend against brute force, you must shift from a permissive default state to a restrictive, "deny-by-default" model.

#### Disabling Password Authentication

Even if you use keys, if password authentication is enabled, an attacker can attempt to guess a system user's password. This is a common entry point for compromised accounts.

bash

```
# Set these directives in /etc/ssh/sshd_configPasswordAuthentication noChallengeResponseAuthentication noUsePAM no
```

_Note: Ensure you have successfully tested your SSH key access before disabling password authentication to avoid being locked out of the server._

#### Limiting Access to Specific Users and Groups

Don't allow the entire user base to attempt SSH connections. Use the `AllowUsers` or `AllowGroups` directives to whitelist only those identities that strictly require remote access.

bash

```
# Only allow specific usersAllowUsers devops-admin deploy-bot# Or allow entire groupsAllowGroups ssh-access
```

#### Reducing MaxAuthTries

By default, SSH allows six failed attempts per connection. Lowering this to two or three forces a delay, as the client must drop the connection and re-initiate the handshake, significantly slowing down automated brute-force scripts.

bash

```
MaxAuthTries 2
```

### Implementing Connection Rate Limiting

While `sshd_config` handles the authentication logic, it is inefficient at managing high-volume connection storms. Integrating an external monitor like `fail2ban` creates a feedback loop where the firewall dynamically blocks IPs that exhibit malicious behavior.

YesNoYesNoIncoming SSH RequestSSH DaemonAuthenticated?Grant SessionLog Failure to /var/log/auth.logFail2ban MonitorThreshold Reached?Update Firewall: Drop IP

`fail2ban` works by monitoring log files for repeated failure patterns (e.g., "Failed password for..."). Once a threshold of failed attempts from a single IP is reached, it instructs `iptables` or `nftables` to drop all traffic from that host for a set duration.

### Reducing the Attack Surface with Listen Directives

By default, `sshd` binds to all available network interfaces (`0.0.0.0`). If your server has a public and a private interface, you should restrict `sshd` to only listen on the management/private interface.

bash

```
# In /etc/ssh/sshd_configListenAddress 192.168.1.50
```

If you must expose SSH to the public internet, change the default port from `22` to a high-numbered port (e.g., `2222`). While this is "security through obscurity" and does not stop a dedicated attacker, it eliminates 99% of "noise" from automated bots scanning port 22.

### Summary

Securing the SSH daemon requires a multi-layered approach: disabling legacy authentication mechanisms, restricting the scope of users who can connect, and implementing automated defensive barriers that mitigate the impact of volumetric brute force attacks. By hardening the daemon configuration and offloading connection-denial to the firewall layer, you ensure that the server's resources are reserved exclusively for authorized administrative traffic.

# Practical Exercise: Performing a Security Audit on the Project Infrastructure

A security audit of the SSH infrastructure ensures that access control policies are not just theoretical, but strictly enforced. The process requires a systematic evaluation of three distinct areas: the identity layer (the keys themselves), the access layer (authorized files and configurations), and the observability layer (logs and active sessions).

## Auditing Authorized Keys

The `authorized_keys` file is the primary surface area for privilege escalation. An audit must identify keys that lack ownership metadata, use deprecated algorithms, or provide excessive permissions.

Use the following loop to inspect every user's directory on the server, extracting key types and identifying comments. This helps you spot "orphaned" keys—keys that belong to former team members or unknown services.

bash

```
# Audit script: List all public keys with their owners and typesfor user_dir in /home/*; do    auth_file="$user_dir/.ssh/authorized_keys"    if [ -f "$auth_file" ]; then        echo "Auditing: $user_dir"        awk '{print $1, $3}' "$auth_file"    fidone
```

If you encounter `ssh-rsa` keys, prioritize them for rotation. Modern security standards mandate `ed25519` for all new deployments due to its immunity to collision attacks and smaller key size.

## Analyzing SSH Daemon Configurations

The `sshd_config` file dictates the security posture of the daemon. A comprehensive audit checks for specific directives that, if left at their default or improperly configured, expose the server to brute-force or unauthorized entry.

Check for these vulnerabilities:

- `PermitRootLogin`: Must be set to `no` or `prohibit-password`.
- `PasswordAuthentication`: If your infrastructure relies strictly on key-based access, this must be set to `no`.
- `MaxAuthTries`: Lowering this to `3` restricts the number of attempts an attacker has to guess a passphrase or present a key before the connection is dropped.

bash

```
# Validate critical security directivesgrep -E "PermitRootLogin|PasswordAuthentication|MaxAuthTries" /etc/ssh/sshd_config
```

## Reviewing Access Logs and Evidence

Security audits depend on identifying _what actually happened_ versus _what was intended_. SSH logs provide the history of authentication attempts, including failures that indicate reconnaissance activity.

On most Linux distributions, `auth.log` or `journalctl` captures the state of the SSH daemon. Specifically, look for high frequencies of `Failed password` or `Connection closed by authenticating user` patterns, which often signal a botnet probing for weak credentials.

Accepted PasswordFailed PasswordAccepted PublickeyRetrieve Auth LogsAnalyze Event TypeFlag for Policy ViolationDetect Brute Force PatternVerify Key FingerprintUpdate Fail2Ban or FirewallCross-Reference with Active Team List

## Executing the Infrastructure Audit

To perform a rigorous check on the "DevOps-Pro" project infrastructure, run the following sequence to map existing access against current team membership:

1. **Generate a Fingerprint Manifest:** Run `ssh-keygen -l -f` on every key found in `authorized_keys` to create a master list of all authorized fingerprints.
2. **Cross-Reference:** Compare this list against the current HR or IAM (Identity and Access Management) list. Any fingerprint not tied to an active, authorized user must be removed immediately.
3. **Permission Check:** Ensure that `.ssh/` directories are `700` and `authorized_keys` files are `600`. Any broader permissions (like `644`) allow other local users on the server to read or tamper with your access list.

## Exercises

- **Orphan Discovery:** Identify every key in your home directory that has not been used in the last 30 days. Use the `ssh -v` verbose flag to compare the keys you are currently using against the ones present on the remote host.
- **Daemon Hardening:** Modify your `sshd_config` to disable all password-based authentication and verify the change by intentionally attempting an SSH connection without an agent or key loaded. The server must reject the attempt before it prompts for a password.

## Summary

A security audit transitions your infrastructure from "working" to "hardened." By auditing the key store, validating daemon directives, and analyzing historical logs, you establish a baseline of trust. This practice directly informs the maintenance of your certificate-based access systems discussed in the previous module, ensuring that even if automated systems manage access, the underlying server configuration remains impenetrable.

