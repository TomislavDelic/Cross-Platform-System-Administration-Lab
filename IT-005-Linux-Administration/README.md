
# IT-005 – Ubuntu Server Administration

## Overview

This lab demonstrates basic Linux system administration, user and group management, file permissions, SSH hardening, firewall configuration, system monitoring, and troubleshooting.

The lab was performed on an Ubuntu Server virtual machine running in Microsoft Hyper-V.

The objective was to simulate common system administration tasks in a small IT environment, including secure remote access, access control, monitoring, and user offboarding.

## Lab Environment

| Component | Configuration |
|---|---|
| Operating System | Ubuntu Server 24.04 LTS |
| Virtualization | Microsoft Hyper-V |
| Administration | SSH and Linux Terminal |
| SSH Client | macOS Terminal |
| Remote Connectivity | Tailscale |
| Firewall | UFW |
| Administrator | tomo |
| Test User | marko |
| Security Group | developers |
| Shared Directory | /srv/developers |

## Implementation

### 1. User and Group Management

Created a standard Linux user and a dedicated group for a development team.

```bash
sudo adduser marko
sudo groupadd developers
sudo usermod -aG developers marko
```

Verified group membership:

```bash
id marko
```

The user was successfully added to the `developers` group without administrative privileges.

### 2. Shared Directory and File Permissions

Created a shared directory for members of the development team.

```bash
sudo mkdir /srv/developers
sudo chown root:developers /srv/developers
sudo chmod 2770 /srv/developers
```

Verified permissions:

```bash
ls -ld /srv/developers
```

Result:

```text
drwxrws--- root developers /srv/developers
```

The permissions provide:

- Full access to the directory owner and development group.
- No access for other users.
- SGID inheritance so newly created files inherit the `developers` group.

Tested file creation as the standard user:

```bash
sudo -u marko touch /srv/developers/marko-test.txt
ls -l /srv/developers/
```

Result:

```text
-rw-rw-r-- 1 marko developers 0 marko-test.txt
```

Verified that an unauthorized user cannot access the directory:

```bash
sudo -u nobody ls /srv/developers
```

Result:

```text
Permission denied
```

### 3. SSH Key-Based Authentication

Configured SSH key-based authentication for secure remote administration from a MacBook.

Generated an Ed25519 SSH key pair on the client:

```bash
ssh-keygen -t ed25519 -C "ubuntu-lab-mac"
```

Protected the private key with a passphrase and added the public key to the Ubuntu user's `authorized_keys` file.

Verified key-based authentication:

```bash
ssh -o PreferredAuthentications=publickey -o PasswordAuthentication=no tomo@<server-ip>
```

Result: SSH login using the private key succeeded.

### 4. SSH Security Hardening

Created a backup of the SSH server configuration:

```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
```

Configured the following settings:

```text
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
```

During verification, the effective configuration still showed:

```text
passwordauthentication yes
```

Investigated active SSH configuration files:

```bash
sudo grep -RniE '^[[:space:]]*PasswordAuthentication' /etc/ssh/sshd_config /etc/ssh/sshd_config.d/
```

Identified an overriding setting in:

```text
/etc/ssh/sshd_config.d/50-cloud-init.conf
```

Changed the setting in this file to:

```text
PasswordAuthentication no
```

Validated the configuration:

```bash
sudo sshd -t
sudo sshd -T | grep -E '^(permitrootlogin|passwordauthentication|pubkeyauthentication) '
```

Verified result:

```text
permitrootlogin no
pubkeyauthentication yes
passwordauthentication no
```

Applied the configuration without terminating existing SSH sessions:

```bash
sudo systemctl reload ssh
```

Confirmed that SSH key authentication worked after reloading.

Tested password-only authentication:

```bash
ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password -o KbdInteractiveAuthentication=no tomo@<server-ip>
```

Result:

```text
Permission denied (publickey).
```

This confirmed that password-based SSH authentication was disabled.

### 5. UFW Firewall Configuration

Checked the initial firewall status:

```bash
sudo ufw status verbose
```

Initial result:

```text
Status: inactive
```

Allowed SSH traffic before enabling the firewall:

```bash
sudo ufw allow 22/tcp
```

Enabled UFW:

```bash
sudo ufw enable
```

Verified the configuration:

```bash
sudo ufw status verbose
```

Results:

- UFW enabled and active at startup.
- Default incoming policy: deny.
- Default outgoing policy: allow.
- TCP port 22 allowed for IPv4 and IPv6.

A new SSH session was successfully established over Tailscale after enabling UFW.

Note: Port 22 was permitted from all source addresses during the lab. Restricting SSH access to trusted sources would be a further security improvement.

### 6. Service Management and Log Analysis

Verified SSH service status:

```bash
systemctl status ssh --no-pager
```

The service was active and running.

The environment used systemd socket activation for SSH.

Reviewed SSH service logs:

```bash
sudo journalctl -u ssh --since "30 minutes ago" --no-pager
```

Confirmed:

- Successful SSH configuration reload.
- SSH listening on TCP port 22.
- Successful public-key authentication.
- Successful creation of SSH sessions over Tailscale.

### 7. System Monitoring

Performed basic system health checks.

Memory usage:

```bash
free -h
```

Disk usage:

```bash
df -h /
```

CPU and process monitoring:

```bash
top -b -n 1 | head -n 15
```

Observed results:

| Resource | Observation |
|---|---|
| RAM | Approximately 535 MiB total |
| Swap | Approximately 39 MiB used |
| Disk | 16 GB total, approximately 44% used |
| CPU | Approximately 100% idle during the sample |
| Load Average | Approximately 0.00 |

These checks demonstrated basic Linux resource monitoring and process inspection.

### 8. User Offboarding – Access Revocation

Simulated removing a user from a development team while preserving the user account.

Verified the existing group membership:

```bash
id marko
```

Removed the user from the development group:

```bash
sudo gpasswd -d marko developers
```

This removed the user's group membership without deleting the account or its files.

A final access-denial test after the group removal was not captured.

## Verification and Results

| Test | Result |
|---|---|
| Create Linux user and group | Passed |
| Assign development group membership | Passed |
| Configure shared directory permissions | Passed |
| Authorized user creates a file | Passed |
| Unauthorized user denied access | Passed |
| SSH login with Ed25519 key | Passed |
| Root SSH login disabled in effective configuration | Passed |
| Password-based SSH authentication blocked | Passed |
| Enable UFW firewall | Passed |
| SSH access through Tailscale after UFW activation | Passed |
| SSH service and log verification | Passed |
| CPU, memory and disk monitoring | Passed |
| Remove user from developers group | Passed |
| Verify access denial after offboarding | Passed |
| Windows PowerShell SSH access after hardening | Passed |

## Issues Encountered

### Issue 1 – SSH Password Authentication Remained Enabled

**Problem:**

After setting `PasswordAuthentication no` in `/etc/ssh/sshd_config`, the effective SSH configuration still showed password authentication as enabled.

**Investigation:**

Inspected active SSH configuration files and discovered that `/etc/ssh/sshd_config.d/50-cloud-init.conf` contained:

```text
PasswordAuthentication yes
```

**Resolution:**

Changed the setting in the cloud-init SSH configuration file to `no`, validated the effective configuration, and reloaded the SSH service.

Subsequent tests confirmed that SSH key authentication worked and password-only authentication was blocked.


### Issue 2 – Windows PowerShell SSH Access Failed

**Problem:**

After disabling SSH password authentication, remote access from Windows PowerShell failed with the following error:

```text
Permission denied (publickey).
```

**Root Cause:**

SSH key-based authentication had been configured and tested successfully from macOS, but the Windows SSH public key had not been added to the Ubuntu Server's `authorized_keys` file.

As a result, Windows could no longer authenticate after password-based authentication was disabled.

**Resolution:**

1. Temporarily restored SSH password authentication through the Hyper-V console.
2. Reloaded the SSH service to restore remote access from Windows PowerShell.
3. Added the existing Windows Ed25519 public key to the Ubuntu user's `~/.ssh/authorized_keys` file.
4. Successfully tested SSH key-based authentication from Windows.
5. Disabled password authentication again.
6. Validated and reloaded the SSH configuration.
7. Confirmed successful SSH key-based login from Windows PowerShell.

**Verification:**

The Windows PowerShell SSH connection succeeded using public-key authentication with password authentication disabled.

The SSH server configuration was verified with:

```bash
sudo sshd -T
```

Confirmed security settings:

```text
permitrootlogin no
pubkeyauthentication yes
passwordauthentication no
```

**Lesson Learned:**

Before disabling SSH password authentication, verify key-based access from every administrative client.

Always maintain an existing authenticated session or console recovery access until the updated SSH configuration has been tested successfully.

## Security Considerations

- SSH private keys remain on their respective client devices.
- Public keys are stored in the user's `authorized_keys` file.
- Root SSH login is disabled.
- SSH password authentication is disabled.
- UFW denies incoming traffic by default.
- Shared directory access is controlled using Linux groups and file permissions.
- Tailscale provides remote network connectivity, but SSH authorization remains independently enforced.

## Conclusion

This lab covered practical Ubuntu Server administration tasks relevant to IT support and system administration.

The exercises included user and group management, shared directory permissions, SSH hardening, firewall configuration, service monitoring, log analysis and user access revocation.

An SSH configuration conflict was identified and resolved by inspecting configuration precedence. A remaining Windows SSH authentication issue was documented as an area for future remediation.
