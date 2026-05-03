# Moving Beyond Individual Keys with SSH Certificates

SSH certificates replace the static, long-lived `authorized_keys` file with a dynamic, cryptographic trust model. Instead of distributing public keys to every server, you treat the server as a client that trusts a single, central authority—the Certificate Authority (CA)—to verify the identity of the person attempting to connect.

### The Shift from Trusting Keys to Trusting Authorities

In a traditional setup, the server keeps a list of authorized keys. If you have 500 servers, you must propagate any change to those 500 `authorized_keys` files. An SSH certificate changes this: the server is configured to trust a specific CA public key. When a user connects, they present a certificate signed by that CA. The server verifies the signature, checks the identity, and grants access if the certificate is valid.

This removes the need for manual key distribution and allows for ephemeral access—you can issue a certificate that is valid for only eight hours, effectively creating a "just-in-time" access model.

Requests access with public keyReturns signed SSH CertificateConnects with signed CertificateVerifies CA signatureAccess GrantedUserCertificate AuthorityServer

### Anatomy of an SSH Certificate

An SSH certificate is a standard SSH public key bundled with metadata. When you inspect a certificate using `ssh-keygen -L -f id_rsa-cert.pub`, you see critical fields that don't exist in a standard key:

- **Key ID:** A unique string for auditing (e.g., the user’s email or employee ID).
- **Valid After/Before:** The hard time limit for the certificate.
- **Principals:** A list of usernames allowed for this certificate. If a cert is signed for the principal `ubuntu`, it cannot be used to log in as `root`.
- **Extensions/Critical Options:** Fine-grained controls like `permit-pty` (allow terminal access) or `source-address` (lock the certificate to a specific IP or CIDR range).

### Configuring Server-Side Trust

To accept certificates, the server must be aware of the CA. You do not add the user's public key to `authorized_keys`. Instead, you update the `sshd_config` file on every host to point to the CA's public key.

Add the following to `/etc/ssh/sshd_config`:

ssh

```
# The public key of your CA used to sign user certificatesTrustedUserCAKeys /etc/ssh/trusted-user-ca-keys.pem
```

Once the service is reloaded, the server will trust any key presented that bears a valid signature from the private key corresponding to `trusted-user-ca-keys.pem`. This is a one-time configuration change per host.

### The Signing Workflow

The user generates a standard key pair and sends their public key to a signing service (the CA). The CA verifies the user's identity—often via OIDC (Okta, Google, GitHub)—and returns a certificate.

The signing command typically looks like this:

bash

```
# Sign the user's public key for 8 hours, allowing the 'devops' principalssh-keygen -s /path/to/ca_signing_key \           -I "user_identity_email" \           -n devops \           -V +8h \           id_rsa.pub
```

The output is `id_rsa-cert.pub`. The user keeps this file alongside their private key. When they run `ssh devops@server`, the SSH client automatically loads the certificate. If the certificate is expired, the connection is rejected by the local agent before it even reaches the server.

### Security Implications and Trade-offs

The primary advantage is **revocation and expiration**. If a developer loses their laptop, you don't need to rotate keys across your infrastructure. You simply wait for the certificate to expire or update the `KRL` (Key Revocation List) on your servers to invalidate specific serial numbers.

The tradeoff is the **availability of the CA**. If your CA signing service goes down, no one can generate new credentials. Consequently, enterprise implementations often use high-availability signing services and strict monitoring of the CA's private key, which is usually stored in a Hardware Security Module (HSM).

### Exercises

1. Create a "dummy" CA key pair using `ssh-keygen -t ed25519 -f ca_key`.
2. Generate a standard user key pair.
3. Sign the user key with the CA key to create a certificate.
4. Use `ssh-keygen -L -f user_key-cert.pub` to inspect the certificate fields and verify the "Principals" list.
5. Attempt to connect to a local VM after adding the CA public key to the VM's `TrustedUserCAKeys` configuration.

### Summary

SSH certificates move the complexity of identity management away from individual servers and into a centralized, short-lived, and auditable signing process. By using certificate-based authentication, you decouple user identity from server configuration, enabling secure access that scales with your infrastructure.

# Configuring a Centralized SSH Certificate Authority

SSH certificates solve the scaling bottleneck inherent in managing `authorized_keys` files across hundreds or thousands of servers. Instead of distributing individual public keys to every host, you establish a centralized Certificate Authority (CA). When a user needs access, they present their public key to the CA, which signs it, creating a short-lived, cryptographically verifiable identity document—a certificate—that any server trusting your CA will automatically accept.

### The Mechanics of Trust

An SSH certificate is simply a public key bundled with metadata: a list of valid principals (usernames), a list of permitted source IP addresses, and, crucially, an expiration timestamp. When a server receives an authentication request, it checks three things:

1. Is the certificate signed by a CA I trust?
2. Has the current timestamp passed the certificate's expiry?
3. Is the user requesting a principal (login name) permitted by this certificate?

Because the server only needs to store the CA’s public key, your infrastructure configuration remains static even as your workforce grows.

Requests signature for public keyVerifies user identityReturns signed certificate (with expiry)Connects via SSH with certificateValidates CA signature & expiryAccess GrantedUserSSH Certificate AuthorityServer

### Establishing the Certificate Authority

The CA is not a complex piece of software; it is a standard SSH key pair kept in an offline or highly restricted, HSM-backed environment. The private key acts as the "Master Key" for your infrastructure.

To create the CA, generate a standard Ed25519 key pair:

bash

```
# Generate the CA private keyssh-keygen -t ed25519 -f ca_key -C "SSH-CA"
```

The `ca_key.pub` file is what you distribute to your fleet. Add this file to `/etc/ssh/trusted-user-ca-keys.pem` on your target hosts and update the `sshd_config` to point to it:

text

```
# /etc/ssh/sshd_config on target serversTrustedUserCAKeys /etc/ssh/trusted-user-ca-keys.pem
```

Once the daemon reloads, any certificate signed by your `ca_key` is now a valid "key" for logging into that server.

### Signing User Certificates

When a developer joins or a rotation is required, you sign their public key using your CA. The command is `ssh-keygen -s`. You must specify the duration (`-V`) and the permitted principals (`-n`).

bash

```
# Sign the user's public key with a 12-hour lifespanssh-keygen -s ca_key -I "user_identity_email@company.com" -V +12h -n ubuntu,admin id_ed25519.pub
```

This produces a file named `id_ed25519-cert.pub`. When the user attempts to log in, the SSH client automatically uses this certificate if it resides in the same directory as their private key.

### Tradeoffs and Constraints

The primary advantage is "no-touch" key management: no more `ssh-copy-id` and no more stale keys in `authorized_keys`. However, this introduces a single point of failure. If the CA private key is compromised, every server in your infrastructure is effectively wide open.

> **Operational Guardrail:** Always store the CA private key on a FIPS 140-2 Level 3 hardware module (like a YubiKey or HSM) and implement a strictly defined, audited signing process to prevent unauthorized certificate issuance.

### Summary

Transitioning to a CA-based architecture shifts your security model from _managing files on servers_ to _managing identity at the point of entry_. By utilizing short-lived certificates, you eliminate the need for manual key rotation, as access automatically expires without server-side intervention. Next, we will cover how to programmatically revoke access and handle the automation of these lifecycle procedures.

# Automating Key Revocation and Expiration Procedures

SSH key revocation is the process of invalidating a public key before its natural expiration, while expiration is the enforcement of a cryptographic "Time-to-Live" (TTL) on the key itself. In an enterprise environment, relying on manual removal of keys from `authorized_keys` files is a security liability that cannot scale. Instead, you must shift toward automated lifecycle management, primarily through certificate-based authentication and automated pruning scripts.

## Revocation via Certificate Revocation Lists (CRL)

When using an SSH Certificate Authority (CA), you do not manage individual public keys on every host. Instead, you issue short-lived certificates. If a device is lost or an employee leaves the company, you need a mechanism to invalidate that certificate immediately, even if it hasn't technically "expired." This is achieved through a Certificate Revocation List (CRL).

The SSH daemon (`sshd`) can be configured to check a `RevokedKeys` file. This file contains a list of public key fingerprints or serial numbers that are explicitly forbidden from access.

To implement this:

1. Generate the CRL file containing the serial numbers of revoked certificates.
2. Distribute the file to all target servers (e.g., using configuration management like Ansible or SaltStack).
3. Update the `sshd_config` on the target hosts:
    
    text
    
    ```
    # /etc/ssh/sshd_configRevokedKeys /etc/ssh/revoked_keys
    ```
    
4. Reload the SSH service. Any key present in this file will be rejected by the server, even if the signature is valid.

## Time-Bound Access with Certificate Expiration

The most effective way to automate key management is to eliminate "permanent" keys entirely. When signing a user key with your CA, you define a `valid_after` and `valid_before` constraint. This forces the user to re-authenticate against your identity provider (like Okta or Active Directory) to obtain a new, short-lived certificate (e.g., valid for 8 hours).

When generating a certificate with `ssh-keygen -s`, you apply these constraints:

bash

```
# Sign a user public key with a 12-hour expiration windowssh-keygen -s ca_key -I user_identity -V +12h -n username user_key.pub
```

By setting the `-V` (validity) flag, you offload the burden of revocation to the expiration clock. If a key is compromised, you only need to secure the identity provider; the SSH certificate will naturally expire without manual intervention on every single server in your infrastructure.

## Automating Key Pruning for Legacy Keys

If your infrastructure still relies on static `~/.ssh/authorized_keys` files, you must implement an automated "pruning" process. This prevents "key sprawl," where access rights remain active long after they are required.

A robust automation approach involves a central repository (often a Git-backed store) of approved public keys, mapped to user IDs and expiration dates. A scheduled job—running via cron or a CI/CD pipeline—pushes the current state of "authorized" keys to servers.

bash

```
#!/bin/bash# Example logic for an automated sync script# 1. Fetch authorized keys from a central database# 2. Check the 'expires_at' attribute for each key# 3. If current_date > expires_at, exclude from deployment# 4. Push the filtered list to the target host's authorized_keys
```

This ensures that any key missing an entry in your central database, or one that has passed its expiration date, is effectively deleted from the target server during the next synchronization cycle.

NoYesExpiredActiveIdentity ProviderGenerate Short-Lived CertificateCertificate Valid?Deny AccessSSH Session EstablishedSession DurationDrop Connection

## Exercises

- **Implement CRL**: Create a dummy `revoked_keys` file containing a base64-encoded public key. Configure a local SSH daemon to use the `RevokedKeys` directive and verify that `ssh` connection attempts with that key are denied.
- **Certificate Expiration**: Generate a certificate using `ssh-keygen` with a very short validity (e.g., `-V +1m` for one minute). Attempt to connect to a host and observe the connection failure once the one-minute window passes.

## Summary

Automation in SSH key management moves security from a reactive, manual task to a proactive, policy-driven system. By combining CRLs for emergency revocation and short-lived certificates for inherent expiration, you minimize the window of opportunity for attackers and eliminate the maintenance overhead of managing thousands of individual public keys. In the next section, we will see how to tie these identities directly into your version control workflows to maintain a clean audit log of who accessed what and when.

# Integrating SSH Keys with Version Control Systems

Integrating SSH keys with Version Control Systems (VCS) like Git hinges on establishing a trusted, authenticated tunnel between your local environment and the remote provider (GitHub, GitLab, Bitbucket). Unlike HTTPS, which often relies on Personal Access Tokens (PATs) or cached passwords, SSH authentication uses your cryptographic key pair to prove identity without transmitting credentials over the network.

## Configuring Git for SSH Transport

Git natively understands SSH. When you clone, push, or pull, Git looks at the remote URL. If it starts with `git@github.com:...` or similar, Git invokes the `ssh` command behind the scenes.

To ensure Git uses the correct key for a specific repository—especially when managing multiple identities, such as a work account and a personal account—you must define an SSH host alias in your `~/.ssh/config` file.

ssh

```
# ~/.ssh/config# Personal accountHost github.com-personal    HostName github.com    User git    IdentityFile ~/.ssh/id_ed25519_personal# Work accountHost github.com-work    HostName github.com    User git    IdentityFile ~/.ssh/id_ed25519_work
```

Once this is defined, you don't use the standard remote URL. You update your local repository remote to use the alias:

bash

```
# Update the remote to use the alias defined in the configgit remote set-url origin git@github.com-work:company-org/project-repo.git
```

## The Authentication Handshake

When you execute `git push`, the following sequence occurs. The SSH client identifies the `Host` alias, looks up the associated `IdentityFile`, and initiates a challenge-response handshake with the VCS server.

Request signature for Host aliasReturns identity signatureEstablishes SSH connection with signatureVerifies signature against stored public keyGrants push/pull accessLocal MachineSSH AgentGit Server

## Managing Multiple Identities and Agent Forwarding

A common friction point arises when using SSH Agent. If you have multiple keys loaded into your `ssh-agent`, the client may attempt to present them to the VCS server in an order that triggers a "too many authentication attempts" error, causing the server to disconnect.

You can explicitly instruct your SSH configuration to send only the key defined for that host, preventing this collision:

ssh

```
Host github.com-work    HostName github.com    User git    IdentityFile ~/.ssh/id_ed25519_work    IdentitiesOnly yes
```

Setting `IdentitiesOnly yes` is a security best practice for enterprise environments. It ensures that even if you have keys loaded for other servers (e.g., your personal project keys or server management keys), the SSH client strictly uses the specific key defined for that Git host.

## Handling SSH Keys in CI/CD Pipelines

In an enterprise context, you often need a build server (like Jenkins, GitHub Actions, or GitLab CI) to interact with VCS repositories. Never copy your personal private keys to a CI/CD environment. Instead, generate a dedicated "Deploy Key."

1. **Scope:** Create an Ed25519 key pair specifically for that repository or pipeline.
2. **Access:** Add only the public key to the repository's "Deploy Keys" settings in your VCS platform.
3. **Storage:** Store the private key as an encrypted "Secret" or "Variable" within your CI/CD platform’s management console.

During the build, the CI/CD runner injects this key into its temporary `ssh-agent`, performs the required git operations, and discards the key once the job finishes. This limits the blast radius if the CI environment is compromised.

## Summary

Integrating SSH with Git shifts the burden of authentication from manual token management to cryptographic identity. By utilizing `~/.ssh/config` for granular control and `IdentitiesOnly` for security, you decouple your various identities and prevent key leakage across projects. These configurations serve as the foundation for the more advanced identity management strategies discussed next, specifically moving toward ephemeral, short-lived certificates.

# Practical Exercise: Implementing Certificate-Based Access for the DevOps-Pro Team

SSH certificates replace the static trust model of individual `authorized_keys` files with a dynamic, cryptographic verification system. Instead of distributing public keys to every host, a central Certificate Authority (CA) signs a user's short-lived public key. When a user connects to a server, the server verifies the certificate’s signature against the CA’s public key. If the signature is valid and the certificate has not expired, access is granted.

## The Certificate Authority Workflow

Implementing certificate-based access requires three distinct actors: the CA (which holds the private signing key), the user (who requests a signed certificate), and the target host (which trusts the CA).

Generate short-lived key pairSend public key + identity infoVerify identitySign public key (return certificate)Connect using signed certificateVerify CA signature on certificateAccess grantedUserCertificate AuthorityProject Server

## Setting Up the Certificate Authority

The CA is simply a host—or a hardened offline server—that holds a private key dedicated to signing. To initialize this, generate the CA key pair:

bash

```
# Generate the CA key (typically RSA or Ed25519)ssh-keygen -t ed25519 -f ca_key -C "DevOps-Pro-CA"# Keep ca_key secure and offline. Only the public key goes to the servers.# View the public keycat ca_key.pub
```

Distribute `ca_key.pub` to every host that the DevOps-Pro team needs to access. On each target host, edit `/etc/ssh/sshd_config`:

text

```
# Add this line to trust the CATrustedUserCAKeys /etc/ssh/ca_key.pub
```

Restart the SSH daemon (`systemctl restart sshd`) to apply the configuration. From this point forward, the host will accept any key signed by this specific `ca_key`.

## Issuing Certificates to DevOps-Pro Team Members

When a developer needs access, they do not give you their `id_ed25519.pub`. Instead, they generate a fresh key pair, and you sign their public key with your CA.

bash

```
# Developer generates their client keyssh-keygen -t ed25519 -f ~/.ssh/devops_project_key# CA Administrator signs the key# -s: Specify the CA key# -I: Key ID (for logging/auditing)# -V: Validity period (e.g., +8h for 8 hours)# -n: Principal names (users allowed to login as)ssh-keygen -s ca_key -I "developer_user@company.com" -V +8h -n ubuntu,root ~/.ssh/devops_project_key.pub
```

The output creates a file named `devops_project_key-cert.pub`. The user places this file in the same directory as their private key (`devops_project_key`). When they run `ssh ubuntu@project-server`, the SSH client automatically detects the `-cert.pub` file and presents it to the server.

## Managing Revocation and Expiration

The primary advantage of this approach is the `-V` (Validity) flag. By forcing short TTLs (Time-To-Live), you eliminate the need to manually remove keys when an employee leaves or a project concludes. Access expires automatically.

For emergency revocation, use a `KRL` (Key Revocation List). If a certificate is compromised before it expires, you can generate a list of revoked serial numbers and push it to all servers:

bash

```
# Generate a KRL filessh-keygen -k -f /tmp/krl_file -u ca_key.pub serial_number_to_revoke# Point the sshd config to this file on every host# RevocationList /etc/ssh/ssh_revoked_keys
```

## Automating the Request Flow

In a mature DevOps environment, manual signing is a bottleneck. Instead, use an intermediate service (like HashiCorp Vault or Smallstep) that automates the `ssh-keygen -s` process. Users authenticate against an OIDC provider (like Okta or Google), and if the OIDC claim is valid, the service automatically signs their key for a limited window.

## Exercises

1. **Verify CA Trust:** Modify a local test VM's `sshd_config` to include the `TrustedUserCAKeys` directive pointing to your generated CA public key. Restart the service and verify that a standard, unsigned key is rejected, while a signed certificate is accepted.
2. **Implement Short-Lived Access:** Sign a test certificate with a validity window of only 5 minutes. Connect to the host, verify access, wait 6 minutes, and confirm that the subsequent connection attempt is rejected.
3. **Inspect the Certificate:** Use `ssh-keygen -L -f id_ed25519-cert.pub` to decode a signed certificate. Observe the Principal, Validity window, and Key ID fields to ensure they match your issuance policy.

## Summary

Certificate-based authentication replaces static identity management with a scalable, policy-driven model. By offloading trust to a CA and enforcing short expiration windows, you minimize the surface area for compromised keys and eliminate the administrative overhead of maintaining `authorized_keys` across distributed infrastructure. Future modules will cover auditing these certificate exchanges to maintain a complete trail of system access.

