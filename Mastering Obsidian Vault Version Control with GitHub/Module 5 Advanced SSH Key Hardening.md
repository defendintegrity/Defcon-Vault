# Implementing SSH Key Constraints and Restriction Options

SSH key constraints are directives added to the `authorized_keys` file that dictate exactly what a public key is permitted to do once it successfully authenticates. By default, an SSH key grants full shell access to the user account it is associated with. Constraints transform a "master key" into a "scoped key," effectively applying the Principle of Least Privilege to remote system access.

## Implementing Key Options

Constraints are prefixed to the public key string within the `~/.ssh/authorized_keys` file. When the SSH daemon processes a login attempt, it parses these options before executing the command or shell.

A standard entry in `authorized_keys` looks like this: `ssh-ed25519 AAAAC3... user@host`

To add a constraint, you prepend the option followed by the key: `restrict,command="rsync --server" ssh-ed25519 AAAAC3... user@host`

### The 'restrict' Keyword

The `restrict` option is the most powerful tool in the hardening toolkit. It disables all features except for those explicitly re-enabled by subsequent options. It acts as a "deny-all" policy for the specific key. Using `restrict` automatically disables:

- Port forwarding
- X11 forwarding
- Agent forwarding
- TTY allocation
- Permit-open requests

### Scoping with command="..."

The `command` option forces the remote server to execute a specific program or script immediately upon authentication, overriding whatever command the client attempted to run. This is essential for building restricted service accounts, such as backup servers or deployment bots.

Example for a deployment user restricted to only running a specific CI/CD script:

text

```
restrict,command="/usr/local/bin/deploy-app.sh" ssh-ed25519 AAAAC3... devops-bot@ci-server
```

### IP and Hostname Validation

You can limit access to specific source network origins using `from="..."`. This prevents a stolen private key from being used unless it originates from a trusted infrastructure segment, such as a hardened jump host or a VPN subnet.

- `from="10.0.5.0/24"`: Restricts access to this CIDR block.
- `from="*.internal.example.com"`: Restricts by DNS pattern.
- `from="192.168.1.5,10.0.0.5"`: Allows multiple comma-separated values.

### Preventing Resource Abuse

If you are allowing shell access, you can prevent key holders from creating persistent tunnels or modifying the environment via SSH flags.

- `no-port-forwarding`: Prevents the client from using the server as a SOCKS proxy or tunnel endpoint.
- `no-x11-forwarding`: Disables GUI application tunneling.
- `no-pty`: Prevents the allocation of a pseudo-terminal (prevents interactive shell access).
- `no-agent-forwarding`: Prevents the remote server from accessing the SSH agent running on your local machine.

NoYesNoYesClient sends SSH requestKey has constraints?Grant full accessApply restrictionsAction permitted?Deny requestExecute command or shellLog event

## Practical Implementation Patterns

### Pattern: The Dedicated Backup Key

For a backup server that only needs to read files via rsync, use a combination of `restrict` and `command`. By omitting `no-pty`, you don't even need to worry about shell access because the `command` constraint overrides it.

text

```
# Only allow rsync from the backup server IPrestrict,from="192.168.10.50",command="rsync --server --sender ." ssh-ed25519 AAAAC3... backup-user@backup-host
```

### Pattern: The Hardened Jump Box Access

If you are using a bastion host, you want to ensure that if the key is compromised, it cannot be used to pivot to other machines using port forwarding.

text

```
# Prevent port forwarding to other internal resourcesno-port-forwarding,no-agent-forwarding,no-x11-forwarding ssh-ed25519 AAAAC3... admin@workstation
```

## Exercises

1. **Configure a Restricted Key**: Create a new Ed25519 key pair. Add your public key to your own `~/.ssh/authorized_keys` file on a test server, but prepend it with `restrict,no-pty`. Attempt to log in via `ssh user@host`. You should be able to connect, but the session should immediately exit because a TTY cannot be allocated.
2. **Verify IP Lockdown**: Modify the same key entry to include `from="127.0.0.1"`. Attempt to connect from your local machine (using your actual IP). Observe the authentication failure in the server's SSH logs (`/var/log/auth.log` or `journalctl -u ssh`).

## Summary

Hardening individual SSH keys via `authorized_keys` options is a critical layer of defense-in-depth. By using `restrict` as a baseline and selectively enabling features, you minimize the blast radius of a compromised key. The upcoming lessons will expand on this by shifting from static, long-lived keys to ephemeral certificates that carry these constraints by design.

# Configuring Key Rotation Policies and Lifecycle Management

Key rotation is the systematic replacement of cryptographic keys to minimize the blast radius of a potential compromise. Even if a private key is silently exfiltrated, its utility is time-bound, effectively limiting the window of opportunity for an attacker. Lifecycle management ensures that keys are retired predictably, preventing the accumulation of "zombie" access points—keys that remain authorized on hosts long after an employee has departed or a project has concluded.

## The Mechanics of Controlled Rotation

Effective rotation requires a period of overlap where both the old and new keys are authorized. Without this overlap, a forced switch often results in broken automation pipelines or locked-out administrators. The standard lifecycle follows a transition from _Authorized_ to _Staged_ to _Retired_.

### Implementing Rotation Overlap

To rotate a key without downtime, append the new public key to the remote `authorized_keys` file before removing the old one. This ensures that the transition is seamless.

1. **Deploy New Key:** Upload the new public key to all servers.
2. **Verify Connectivity:** Confirm the new key provides access.
3. **Remove Old Key:** Delete the specific entry for the old public key from `authorized_keys`.

### Automating Lifecycle with Key Constraints

You can harden rotation by attaching `valid-after` and `valid-before` constraints directly to the authorized key entry. These are added as options at the beginning of the line in the `~/.ssh/authorized_keys` file.

text

```
# Example of a time-constrained keyvalid-after="2023-10-01 00:00:00",valid-before="2023-12-31 23:59:59" ssh-ed25519 AAAAC3... user@workstation
```

By enforcing these constraints, the SSH daemon will refuse to authenticate the key outside of the specified window, effectively forcing a rotation policy at the protocol level.

## Lifecycle Management Workflow

Managing the lifecycle of a key pair involves tracking metadata: creation date, owner, purpose, and expiration policy. For individual users, this is often tracked in a configuration management system or a secure vault. For infrastructure, the rotation frequency should be tied to the sensitivity of the resource.

Generate Key PairAuthorized on HostDeployment of New KeyBoth Keys ValidOld Key RemovedKey Pair DestroyedProvisionedActiveTransitionRetired

### Establishing Rotation Cadence

The frequency of rotation should be based on your risk profile:

- **High-Security Environment:** Rotate every 30–90 days.
- **Standard DevOps Infrastructure:** Rotate every 6–12 months or upon role change.
- **Emergency Rotation:** Immediate revocation if a private key is moved to an insecure location, a laptop is lost, or a developer leaves the team.

## Auditing and Revocation

Revocation is the immediate invalidation of a key. Because SSH does not have a native, global "Certificate Revocation List" (CRL) for standard public keys, revocation requires active removal of the public key string from the `authorized_keys` file across your fleet.

To perform a fleet-wide revocation, utilize configuration management tools like Ansible or SaltStack. A manual approach is prone to "ghosting"—leaving an old key active on a forgotten server.

bash

```
# Example snippet for an Ansible task to remove a compromised key- name: Remove compromised key from authorized_keys  lineinfile:    path: /home/{{ user }}/.ssh/authorized_keys    state: absent    regexp: '.*{{ lookup("file", "compromised_key.pub") }}'
```

This ensures that once a key reaches its end-of-life or is flagged for removal, the change is propagated identically across every host in your infrastructure, leaving no lingering access.

## Summary

Key rotation transforms SSH security from a static "set and forget" configuration into a dynamic, manageable process. By utilizing time-based constraints, you can prevent key abuse, while clear lifecycle stages (Provisioned, Active, Retired) prevent the proliferation of unauthorized access. Always favor automated removal over manual deletion to ensure consistency across your environment.

# Utilizing Hardware Security Modules and YubiKeys for Key Storage

Hardware Security Modules (HSMs) and security tokens like the YubiKey act as an isolated environment for cryptographic operations. When you store an SSH private key on your laptop's disk, it is technically accessible to any process or user with sufficient permissions on your operating system. By contrast, an HSM performs the cryptographic signing operation internally; the private key never leaves the device's hardware, meaning even a complete compromise of your machine’s memory or filesystem cannot expose the key itself.

### The PKCS#11 Interface

SSH interacts with security hardware primarily through the PKCS#11 standard. This is a cryptographic API that allows applications to use a token to perform functions like `C_Sign` (signing data). To use a YubiKey for SSH, you must instruct your `ssh-agent` to load a provider library that bridges the gap between the SSH protocol and the physical device.

On most Linux distributions, this library is provided by `opensc`. On macOS, it is often bundled with the system’s native smartcard support.

bash

```
# Locate the PKCS#11 module (e.g., /usr/lib/x86_64-linux-gnu/opensc-pkcs11.so)# Load the key into your running agentssh-add -s /usr/lib/x86_64-linux-gnu/opensc-pkcs11.so
```

Once loaded, the `ssh-agent` acts as a proxy. When you connect to a server, the server sends a challenge; the agent forwards that challenge to the YubiKey, the hardware signs it using the internal private key, and the signature is returned to the server.

### Hardware-Backed Key Generation

While you can import existing keys onto some tokens, the most secure workflow is generating the key pair _on_ the device. This ensures the private key is created in an environment from which it cannot be exported.

1. **Initialize the YubiKey:** Ensure the PIV (Personal Identity Verification) application is enabled and you have set your PIN and Management Key.
2. **Generate the Key:** Use `ssh-keygen` with the `pkcs11` type to interface directly with the device's slot.
3. **Pin Protection:** Unlike standard file-based keys, a hardware key requires you to enter the device’s PIN whenever it performs a signing operation. This provides a physical second factor (something you have) and a knowledge factor (the PIN).

ssh user@serverSend ChallengeSign Challenge (via PKCS#11)Prompt for Physical Touch/PINTouch/Enter PINSignatureSignatureAccess GrantedUserHostAgentYubiKey

### Constraints and Considerations

Using hardware security introduces specific operational constraints:

- **Latency:** Signing operations are slower than software-based RSA or Ed25519 operations because the data must be transferred over the USB/NFC interface.
- **Physical Presence:** Most enterprise-grade tokens require a physical touch to authorize a signing operation. This effectively mitigates remote attacks where an attacker might attempt to use your socket-forwarded agent to authenticate to other servers without your knowledge.
- **Key Slot Limits:** Tokens have a finite number of slots. You cannot store an unlimited number of keys. Use the `ssh-add -L` command to list currently available keys provided by your hardware.

### Exercises

1. **Identify the Provider:** Locate the PKCS#11 shared library file on your system (usually found in `/usr/lib/` or via `brew list opensc`).
2. **Verify Token Presence:** Use `ssh-add -I` (or `pkcs11-tool -L`) to confirm your system detects the security token.
3. **Session Test:** Configure your local `ssh-agent` to point to the token and attempt a connection to a test server. Observe the requirement to touch the device.

### Summary

Transitioning to hardware-backed keys shifts the security boundary from the software layer (which is inherently vulnerable to malware and privilege escalation) to the physical layer. By requiring a PIN and physical presence for every cryptographic handshake, you neutralize the risk of private key theft. In the next module, we will explore how these individual identity keys can be replaced by enterprise-scale certificate authorities to simplify management across large infrastructure fleets.

# Auditing SSH Access Logs for Unauthorized Attempts

SSH authentication logs provide a high-fidelity audit trail for every connection attempt to your server. By default, most Linux distributions route these logs through `sshd` to the system logger, typically stored in `/var/log/auth.log` (Debian/Ubuntu) or `/var/log/secure` (RHEL/CentOS). Analyzing these logs is the primary mechanism for identifying brute-force attacks, unauthorized key usage, and suspicious account activity.

## Interpreting SSH Authentication Log Patterns

Every SSH event follows a predictable lifecycle. When a user connects, the daemon logs the stage of the handshake, the authentication method used, and the final result.

Consider these standard log entries:

text

```
# Successful public key authenticationAccepted publickey for admin from 192.168.1.50 port 52412 ssh2: RSA SHA256:f+b4...# Failed password attempt (when passwords are disabled)Failed password for invalid user guest from 10.0.0.5 port 44322 ssh2# Connection closed before authenticationConnection closed by 185.191.171.14 port 43212 [preauth]
```

The `[preauth]` flag is a critical indicator. It signifies that the connection was terminated before the client successfully authenticated. A high frequency of `[preauth]` entries from a single IP address is the signature of a brute-force or port-scanning bot.

## The Log Processing Workflow

To audit these logs effectively, you must move from manual tailing to structured pattern matching. The goal is to isolate anomalies—such as logins at unusual hours, access from unexpected geographies, or a sudden spike in failed attempts.

High FailuresUnusual Time/IPSuccessful LoginRaw SSH Logs: /var/log/auth.logFilter: grep 'Failed' or 'Accepted'Analyze PatternsFlag/Block IP via IPTablesAlert System AdministratorLog Session Metadata

### Filtering with Grep and Awk

For ad-hoc audits, you can extract relevant data without complex tools. To find the top IP addresses currently attempting unauthorized access:

bash

```
# Count failed attempts by IPgrep "Failed password" /var/log/auth.log | awk '{print $(NF-3)}' | sort | uniq -c | sort -nr
```

This command parses the log for failed password attempts, isolates the IP address field, counts occurrences, and sorts them by frequency. If you see a single IP with hundreds of attempts, you are witnessing an automated brute-force attack.

## Identifying Key-Based Anomaly Indicators

While public key authentication is inherently more secure than passwords, misconfigured servers can still be exploited. When auditing key usage, look for:

1. **Unexpected Fingerprints**: If you maintain a strict whitelist, any log entry showing a successful `publickey` authentication with a fingerprint that does not match your known keys indicates a compromised or unauthorized identity.
2. **Post-Authentication Behavior**: Even after a successful login, check the system logs for immediate privilege escalation attempts (e.g., `sudo` failures).
3. **Ghost Sessions**: If an `Accepted publickey` entry appears, but there is no corresponding `session opened` or `received disconnect` event for an extended period, the session might be being used as a persistent reverse proxy or tunnel.

## Automating Surveillance

While manual audits catch point-in-time incidents, production environments require continuous monitoring. Tools like `fail2ban` function by acting as an automated log-parser that updates firewall rules in real-time.

When you configure `fail2ban`, you are essentially automating the process of identifying the `Failed password` or `Invalid user` strings in your logs and triggering a `DROP` command for those source IPs.

> **Security Warning:** Do not rely solely on log analysis for security. If your logs show evidence of a successful unauthorized login, you must assume the private key associated with that user account is compromised. Rotate the key immediately rather than simply blocking the IP.

## Exercises

1. **Identify Attackers**: Run a frequency count on your current `auth.log` (or `secure` log) to identify the top 5 source IPs attempting to connect to your server.
2. **Search for Specific Key Usage**: Grep the logs for a specific known-good public key fingerprint to verify exactly when that user last accessed the system.
3. **Analyze Preauth Spikes**: Find all logs tagged `[preauth]` within the last hour and calculate the total number of unique IP addresses attempting to probe your server.

## Summary

Auditing SSH logs allows you to observe the behavior of attackers and verify the integrity of your authentication workflow. By identifying `[preauth]` patterns and analyzing successful authentication fingerprints, you maintain visibility over who is accessing your infrastructure. This data forms the input for your automated defense strategies.

# Practical Exercise: Restricting Key Access to Specific IP Ranges

The `authorized_keys` file acts as the primary gatekeeper for SSH access. By default, it grants access to anyone possessing the corresponding private key, regardless of their network location. Constraining this access to specific IP ranges turns the public key from a "global master key" into a "context-aware credential," significantly narrowing the attack surface if a private key is ever exfiltrated.

## The `from` Directive in `authorized_keys`

SSH provides a built-in option within the `authorized_keys` file to restrict a key's validity to specific hosts or subnets. This is prepended directly to the key entry. When the SSH daemon processes a connection, it inspects this directive before accepting the public key. If the source IP does not match the criteria, the key is ignored, even if the cryptographic signature is valid.

The syntax follows this pattern: `from="pattern-list" ssh-rsa AAAA...user@host`

The `pattern-list` supports comma-separated entries, including:

- **Specific IPs:** `192.168.1.50`
- **CIDR blocks:** `10.0.0.0/24`
- **Wildcards:** `*.example.com` or `*.168.1.*`
- **Negation:** `!192.168.1.10` (excludes this specific host)

### Implementing IP Restrictions

To implement this on the `DevOps-Pro` infrastructure, edit the `~/.ssh/authorized_keys` file on the remote server. Assume you want to restrict the `deploy-admin` key so it only works from the corporate VPN subnet (`10.50.0.0/16`) and a specific jump host (`192.0.2.22`).

Open the file and modify the line:

text

```
# Before:ssh-ed25519 AAAAC3Nza... user@workstation# After:from="10.50.0.0/16,192.0.2.22" ssh-ed25519 AAAAC3Nza... user@workstation
```

Once saved, any attempt to connect using that private key from an IP outside those ranges will be rejected immediately by the server, forcing the connection to fall back to other authentication methods or terminate entirely.

## Security Implications and Flow

When you apply the `from` constraint, you are essentially creating a conditional logic gate within the SSH daemon's authentication phase.

YesNoYesNoClient initiates SSH connectionServer receives requestIP in authorized_keys 'from' list?Perform Cryptographic ChallengeReject key and try next methodSignature Valid?Grant Shell AccessDeny Access

It is vital to recognize that this is a **supplemental** security layer. It does not replace the requirement for a strong passphrase or a secure private key. If an attacker gains access to a machine _within_ your allowed IP range, the `from` restriction will not protect you. Therefore, this should be paired with network-level firewalls.

## Practical Constraints and Best Practices

1. **Avoid Single Points of Failure:** If you restrict a key to a single IP (e.g., your current laptop's dynamic IP), you will lock yourself out the moment your ISP rotates your address. Always use a stable subnet or a known VPN/Bastion host IP.
2. **Order of Operations:** The SSH daemon checks `authorized_keys` entries sequentially. If you have multiple keys for the same user, an unrestricted key listed above a restricted one will still grant access. Always ensure your most restrictive policies are enforced globally or at the top of the file.
3. **Auditing:** Because this happens at the `sshd` process level, unauthorized attempts from restricted IPs will appear in your system logs (usually `/var/log/auth.log` or `journalctl -u ssh`). Monitor these logs to identify if a compromised key is being used from an unexpected network location.

## Exercises

1. **Verify Current Connectivity:** From your local terminal, attempt an SSH connection to your `DevOps-Pro` server. Confirm the connection succeeds.
2. **Apply Restriction:** Edit the `authorized_keys` file on the server. Prepend `from="127.0.0.1"` to your existing public key line. Save and exit.
3. **Test the Lockout:** Attempt to connect from your local terminal again. Note the connection failure.
4. **Correction:** Re-edit the file and change the restriction to allow your actual local IP address (or a broader CIDR block covering your current network). Confirm that connectivity is restored.

## Summary

Restricting SSH keys by IP range adds a layer of location-based context to your authentication. By utilizing the `from` directive in `authorized_keys`, you ensure that even if a private key is leaked, it is useless to an attacker outside your defined network perimeter. In the next module, we will scale this security approach by moving away from individual keys toward SSH certificates and centralized authorities.

