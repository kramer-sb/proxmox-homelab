# Trusting Caddy's Local CA on Windows (removes self-signed cert warnings)

## Background

Every `.lab` service fronted by Caddy (`tls internal`) - currently Gitea,Vaultwarden, and Uptime Kuma - presents a self-signed certificate. This is expected and correct: `.lab` is not a publicly resolvable domain, so no public Certificate Authority can verify control of it, and Caddy self-signs
using its own local CA instead. The connection is still fully encrypted; Windows just doesn't trust the issuer by default, so it shows a "Not secure" / certificate warning on every visit.

Clicking through the warning each time is safe and is all the course requires. This doc covers the **optional** fix: importing Caddy's local root CA certificate into Windows' Trusted Root store so the warning goes away permanently for that service.

## Important: this is per-Caddy-instance, not global

**Each Caddy install generates its own independent local CA.** Gitea's native Caddy install, Uptime Kuma's native Caddy install, and Vaultwarden's Docker Compose Caddy container are three separate Caddy instances - each with its own root certificate. Trusting Gitea's CA on Windows does **not** automatically trust Kuma's or Vaultwarden's. This process must be repeated once per service (or at least once per distinct Caddy instance, if any services ever end up sharing one).

## Steps

### 1. Retrieve the root certificate from the LXC/container

Enter the relevant LXC from the Proxmox host shell (using `pct enter <CTID>` - more reliable than the noVNC console for typing):

```
pct enter <CTID>
```

Print the certificate contents:

```
cat /var/lib/caddy/.local/share/caddy/pki/authorities/local/root.crt
```

If that path doesn't exist (e.g. for the Docker-based Vaultwarden Caddy
container, whose data path will differ), locate it instead with:

```
find / -name root.crt 2>/dev/null
```

For a Dockerized Caddy (Vaultwarden), the equivalent is usually retrieved
with:

```
docker exec <caddy-container-name> cat /data/caddy/pki/authorities/local/root.crt
```

### 2. Copy the full certificate text

Select and copy the **entire** output, including both the `-----BEGIN CERTIFICATE-----` and `-----END CERTIFICATE-----` lines. Omitting either line makes Windows reject the file as invalid.

### 3. Save it as a `.crt` file on Windows

1. Open **Notepad** (not Word).
2. Paste the certificate text exactly as copied.
3. **File > Save As**, set "Save as type" to **All Files**, name it    something identifiable (e.g. `GiteaCaddyRootCA.crt`, `KumaCaddyRootCA.crt` VaultwardenCaddyRootCA.crt`), and save it  somewhere easy to find (Desktop/Downloads).

### 4. Install into the Windows certificate store

1. Double-click the saved `.crt` file. A Certificate dialog opens.
2. Click **Install Certificate...**
3. Choose **Local Machine** (not "Current User") so it's trusted system-wide. Click Next.
4. Approve the Windows admin/UAC prompt.
5. Select **"Place all certificates in the following store"**, click  **Browse**, choose **Trusted Root Certification Authorities**. Click Next, then Finish.
6. Windows shows a Security Warning with a thumbprint, asking you to  confirm trust - click **Yes**.
7. Confirm the "The import was successful" message.

### 5. Restart the browser and verify

Fully close and reopen the browser (not just the tab), then visit the `https://<service>.lab` URL. It should now show a normal padlock with no warning.

## Repeating for other services

Repeat steps 1–5 for each `.lab` service you want warning-free:

- [x] `gitea.lab`
- [ ] `kuma.lab`
- [ ] `vaultwarden.lab`

## Notes

- This only affects trust on the **Windows management host** where the   certificate was imported. Any other device (phone, another PC) would need the same import repeated to browse `.lab` services without warnings from that device.
- If a service's Caddy instance is ever rebuilt or its data directory wiped, it will generate a **new** local CA, and the old imported certificate in Windows will need to be replaced (delete the old entry from Trusted Root Certification Authorities via `certmgr.msc`, then re-import the new one).
