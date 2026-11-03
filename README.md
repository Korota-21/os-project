# Project 8 – SSH and Authentication Hardening with MFA

**Authors:** Marwa, Abeer, Reham

Hardening SSH access on a Linux VM using key-based authentication, TOTP multi-factor authentication, account lockout against brute-force attacks, and source IP restriction.

---

## Table of Contents

1. [Objective](#objective)
2. [Environment & Versions](#environment--versions)
3. [Prerequisites](#prerequisites)
4. [Setup Instructions](#setup-instructions)
5. [Testing](#testing)
6. [Results](#results)
7. [Limitations & Challenges](#limitations--challenges)
8. [Cleanup & Recovery](#cleanup--recovery)
9. [Conclusion](#conclusion)

---

## Objective

Secure remote access to a Linux virtual machine by hardening the SSH service:

- Enforce **key-based authentication** (Ed25519)
- Add **TOTP MFA** (Google Authenticator)
- Configure an **account lockout policy** against brute-force attacks
- Apply **source IP restrictions**

## Environment & Versions

| Component | Details |
|---|---|
| OS | Ubuntu 26.04.1 LTS (Virtual Machine, VirtualBox) |
| SSH | OpenSSH (default Ubuntu repository) |
| MFA module | Google Authenticator (`libpam-google-authenticator`) |
| Network | Dual adapter: NAT (internet) + Host-Only (client connections) |
| Client | Windows command line, IP `192.168.56.1` |

## Prerequisites

- An active user account with `sudo` privileges
- An Ed25519 SSH key pair generated on the client machine
- The Google Authenticator app installed on a mobile device

## Setup Instructions

### 1. Generate and transfer the SSH key

On the client machine:

```bash
ssh-keygen -t ed25519
scp ~/.ssh/id_ed25519.pub user@<IP>:~/
```

On the server, add the key to the authorized keys:

```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
cat ~/id_ed25519.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

### 2. Install the MFA module

```bash
sudo apt install libpam-google-authenticator -y
```

### 3. Configure MFA

Run the following as the target user and follow the interactive prompts (scan the QR code with the Google Authenticator app):

```bash
google-authenticator
```

### 4. Apply SSH security policies

Create the config file:

```bash
sudo nano /etc/ssh/sshd_config.d/00-security.conf
```

Add the following rules:

```text
PasswordAuthentication no
PermitRootLogin no
PubkeyAuthentication yes
KbdInteractiveAuthentication yes
AuthenticationMethods publickey,keyboard-interactive
AllowUsers sec_user@192.168.56.1
```

> The file is named `00-security.conf` so it is loaded first and overrides the default Ubuntu SSH configuration.

### 5. Configure PAM (Faillock & TOTP)

Edit the PAM SSH file:

```bash
sudo nano /etc/pam.d/sshd
```

Apply the jump logic:

```text
auth required pam_faillock.so preauth silent deny=3 unlock_time=120
auth [success=1 default=ignore] pam_google_authenticator.so
auth [default=die] pam_faillock.so authfail
auth sufficient pam_faillock.so authsucc
```

### 6. Restart the service

```bash
sudo systemctl restart ssh
```

## Security Controls Summary

| Control | Implementation |
|---|---|
| Cryptographic hardening | Weak KEX, MACs, and ciphers removed after analysis with `ssh-audit` |
| Key-based auth | Ed25519 keys; password login and direct root login disabled |
| MFA | `libpam-google-authenticator` + `AuthenticationMethods publickey,keyboard-interactive` |
| Lockout policy | `pam_faillock`: 120-second lock after 3 consecutive failed attempts |
| Source restriction | `AllowUsers sec_user@192.168.56.1` |

## Testing

Tests were run iteratively with a clean user account (`sec_user`):

- Attempting unauthorized **root login** (denied)
- Logging in with a **valid key and MFA code** (allowed)
- Intentionally **failing the MFA prompt** to trigger the lockout
- Connecting from **unauthorized loopback addresses** to verify source restriction (denied)

## Results

All security implementations worked as expected. The cryptographic baseline improved significantly, brute-force attacks are mitigated through key-only access and `faillock`, and MFA successfully requires secondary validation.

> See the attached **Evidence** folder for screenshots and logs.

## Limitations & Challenges

- **PAM execution order:** Initially, the standard `required` control flag caused valid MFA attempts to fail, because PAM continued on to the `authfail` faillock module. This was resolved with PAM jump logic (`[success=1 default=ignore]`), which skips the failure condition after a successful TOTP entry.
- **Config precedence:** Overriding default Ubuntu SSH settings required naming the custom file `00-security.conf` so that it is read first.

## Cleanup & Recovery

To revert the system to its default state:

1. Delete the custom SSH configuration:
   ```bash
   sudo rm /etc/ssh/sshd_config.d/00-security.conf
   ```
2. Revert `/etc/pam.d/sshd`: remove the faillock/TOTP lines and uncomment `#@include common-auth`.
3. Unlock any locked accounts:
   ```bash
   sudo faillock --user <username> --reset
   ```
4. Restart SSH:
   ```bash
   sudo systemctl restart ssh
   ```

## Conclusion

By combining strong cryptography, key-based authentication, TOTP MFA, brute-force protection, and strict IP whitelisting, the server is protected against both automated attacks and unauthorized credential reuse.
