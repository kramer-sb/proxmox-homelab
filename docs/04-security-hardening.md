# 04 - Security Hardening (Proxmox VE)

Covers hardening the base Proxmox install: creating a non-root admin account for the web UI, adding MFA, disabling root's web login, and restricting SSH to key-based auth only.

## 1. New Admin User (Web UI)

1. **Datacenter > Users > Add**
   - Username: `admin`
   - Realm: `Proxmox VE authentication server` (not Linux PAM — this account only exists inside Proxmox)
   - Set a password (stored in password manager)
2. **Datacenter > Permissions > Add > User Permission**
   - Path: `/`
   - User: `admin@pve`
   - Role: `Administrator`
   - Propagate: checked
3. Verified login as `admin@pve` before continuing, then signed back in as `root` to finish hardening.

## 2. MFA (TOTP) on the Admin Account

1. **Datacenter > Two Factor > Add > TOTP**
2. User: `admin@pve`
3. Scanned QR code with Aegis Authenticator
4. Confirmed with a verify code + `admin@pve` password

> Switched away from Google Authenticator after a device issue wiped all stored TOTP data with no way to recover it. Aegis supports manual encrypted export to a file I control, so a future device loss doesn't mean losing every TOTP token again.

## 3. Disable Root Web Login

1. **Datacenter > Users > root > Edit**
2. Unchecked **Enabled**, clicked Ok
3. Confirmed: `admin@pve` web login still works with MFA; `root` can no longer log into the web UI at all

## 4. SSH Key-Only Access (root)

1. Generated a key pair on the local machine:
   ```
   ssh-keygen -t ed25519 -f ssh_key
   ```
   (passphrase set on the key)
2. Copied the public key (Windows `cmd`, not PowerShell):
   ```
   type ssh_key.pub
   ```
3. Added the public key to Proxmox via the web Shell:
   ```
   nano .ssh/authorized_keys
   ```
   pasted the key on its own line, saved (`Ctrl+S`), exited (`Ctrl+X`)
4. **Tested key login before disabling password auth** (so there was a fallback if the key didn't work):
   ```
   ssh root@10.0.0.52 -i ssh_key
   ```
5. Disabled password authentication:
   ```
   nano /etc/ssh/sshd_config
   ```
   set `PasswordAuthentication no`
6. Applied the change:
   ```
   systemctl restart sshd
   ```
7. Re-tested key login after the restart — success, no password prompt, key + passphrase only.
8. Backed up the private key (`ssh_key`, no extension) to password manager + USB drive.
   **Private key is never committed to this repo** — only the fact that key-only auth is configured is documented here.

## Result

- **Web UI:** `admin@pve` + MFA only. Root web login disabled.
- **SSH:** root, key-only, password auth disabled.
- **Fallback:** physical console access to the host remains as the last resort if both of the above are ever lost.
