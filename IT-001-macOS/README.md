# IT-001 – macOS Workstation Administration

## Objective
Configure and verify local user accounts, permissions, and basic security settings on a macOS workstation.

## Lab Environment
- Device: MacBook Pro 14-inch (Apple M5)
- Operating System: macOS Tahoe 26.1
- Administrator: Tomislav
- Standard User: labuser

## Tasks Completed
- Verified the macOS hardware and operating system configuration.
- Created and tested a standard local user account.
- Verified user identity and group membership using Terminal.
- Confirmed that labuser does not have administrator privileges.
- Checked Secure Token, APFS Volume Ownership, and FileVault status.

## Verification
- Username: labuser
- UID: 502
- Primary Group: staff
- Administrator Membership: No
- Secure Token (primary administrator): Enabled
- FileVault: Disabled

## Result
PASSED – Local user administration and security baseline verification completed.

## Terminal Commands and Verification

The following commands were used to verify the local user account and its group membership in macOS.

### User Identity

```bash
id -un labuser
id -u labuser
id -gn labuser
id -Gn labuser
```

These commands display the username, user ID (UID), primary group, and group memberships.

**Verified results:**
- Username: labuser
- UID: 502
- Primary group: staff
- The user belongs to standard local system groups.

### Administrator Group Membership

```bash
dseditgroup -o checkmember -m labuser admin
```

This command checks whether labuser belongs to the local admin group.

**Verified result:**

```text
no labuser is NOT a member of admin
```

### Result

Confirmed that labuser is a standard macOS account without local administrator group membership. User identity and group membership were verified using Terminal commands.

## Screenshots
### Local User Accounts

<img src="IT-001-01-Local-Users.png" alt="macOS local user accounts" width="650">
