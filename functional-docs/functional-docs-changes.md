# Additions to existing functional docs (from the Stalwart work, 2026-11-02)

Two small additions. Each shows where it goes. The real domain is replaced with `example.dev` as elsewhere in the repo.

## 1. `functional-docs/coredns.md`

Add a new section after "Adding a new `.lab` DNS entry":

```
## Names that are not `.lab`

The `lab` server block in the Corefile only applies `lab.hosts` to the `lab` zone. A line in `lab.hosts` for any other name is silently ignored, because queries for other zones fall through to the `.` block, which forwards upstream. To make CoreDNS answer for one name outside `.lab` (for example a mail server's real hostname so its certificate matches), add a server block for just that name to `/etc/coredns/Corefile`:

mail.cyber.example.dev {
    hosts {
        10.0.0.250 mail.cyber.example.dev
    }
    log
    errors
}

1. Back up first: `cp /etc/coredns/Corefile /etc/coredns/Corefile.bak-DATE`.
2. Append the block, then `systemctl restart coredns.service`.
3. Verify the new name, a normal public name and a `.lab` name:
   nslookup mail.cyber.example.dev 10.0.0.30
   nslookup example.dev 10.0.0.30
   nslookup gitea.lab 10.0.0.30
4. To undo, restore the backup and restart.

Only machines that use CoreDNS see the answer. The Windows management host deliberately does not (see note 07a), so it needs a hosts file entry instead.
```

(The block of commands in step 3 is shown plain in the real file the same way the existing steps are.)

## 2. `functional-docs/tailscale.md`

Add to section 5 ("Configure Split DNS for `.lab`"), after step 4:

```
A second entry works the same way for a single name outside `.lab`: Add nameserver > Custom, `10.0.0.30`, "Restrict to domain" on, and the full name (for example `mail.cyber.example.dev`, no leading dot). Only that name is sent to CoreDNS. This relies on CoreDNS having a server block for the name (see `coredns.md`).
```

Add to "Common pitfalls":

```
- **Tailscale not connected on the phone.** Names that depend on the Split DNS entries (`.lab` names and any extra single-name entry) fail with "DNS address could not be found" in a browser. Connect the Tailscale app first. If it is connected and a Chrome-based browser still fails, check the browser's own secure DNS setting and Android's Private DNS setting.
```
