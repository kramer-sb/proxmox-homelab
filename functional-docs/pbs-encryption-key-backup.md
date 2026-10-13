# Backing Up and Verifying PBS Encryption Keys

Each Proxmox Backup Server storage that uses client-side encryption has its own key. Backups made with a key can only be restored with that key, so if the Proxmox host is lost and the only copy of a key was on it, those backups are unrecoverable. This doc covers saving a copy off the host and proving that the saved copy is exact. See `proxmox-backup-server.md` for which storages use keys and `backup-strategy.md` for why.

## Where the keys live

On the Proxmox host, in `/etc/pve/priv/storage/`, one `.enc` file per encrypted storage (for example `remote-backup.enc` and `usb-backup.enc`). The `.pw` files next to them are the storage passwords, not the encryption keys.

The keys are never committed to this repo, and neither are their fingerprints.

## Steps

Run all of these in the Proxmox host shell. The example uses `usb-backup`; swap in another storage name for the others.

1. List the key files:

   ```
   ls -l /etc/pve/priv/storage/
   ```

2. Note the fingerprint of each key. This is safe to share, because it only identifies the key:

   ```
   sha256sum /etc/pve/priv/storage/*.enc
   ```

3. Print the key to save, then copy all of it, from `{` to `}`:

   ```
   cat /etc/pve/priv/storage/usb-backup.enc
   ```

4. Paste it into a Secure Note in a password manager that is **not** hosted on the Proxmox host. Vaultwarden runs on this host, so a copy there is lost along with it. Name the note after the storage, for example "Proxmox usb-backup PBS encryption key". Never paste the key into a chat or a repo.

5. Start capturing the saved copy into a temporary file:

   ```
   cat > /tmp/check.enc
   ```

6. Paste the key from the password manager, then press Ctrl+D (twice if it does not finish).

7. Compare the fingerprints, with newlines stripped so a trailing newline does not cause a false mismatch. The two must be identical:

   ```
   tr -d '\n' < /tmp/check.enc | sha256sum
   tr -d '\n' < /etc/pve/priv/storage/usb-backup.enc | sha256sum
   ```

8. Delete the temporary file:

   ```
   rm /tmp/check.enc
   ```

If the fingerprints differ, the saved copy is wrong. Re-copy it from step 3 and repeat from step 5.

Repeat the whole procedure for every storage that has a key.

## When to redo this

- Whenever a key is generated or replaced for a storage.
- After restoring the host, to confirm the saved copies still match.
