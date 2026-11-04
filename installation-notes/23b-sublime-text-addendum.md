# 23b - Sublime Text on kali-linux (addendum)

Status: **Installed** from Sublime's official apt repository on VM 207 (`kali-linux`, `10.0.0.90`). The clean version is the "Sublime Text" section in [kali-linux.md](../functional-docs/kali-linux.md).

## Goal

Add a lightweight GUI text editor to the Kali VM for editing scripts and notes.

## Decisions

- **Sublime's own apt repository.** Sublime Text is not in the Kali repos, so the editor comes from the repository Sublime publishes at `download.sublimetext.com`. Installing through apt means `apt full-upgrade` keeps it updated.
- **Stable channel.** The dev channel is restricted to licensed users, and Sublime is free to evaluate.
- **Instructions from Sublime's install page** (https://www.sublimetext.com/docs/linux_repositories.html). The repo file uses the newer `.sources` format with the key stored in `/etc/apt/keyrings/`, not the older `trusted.gpg.d` method.

## What I did

1. **Created the keyrings folder and imported the signing key:**
   ```bash
   sudo mkdir -p /etc/apt/keyrings
   wget -qO - https://download.sublimetext.com/sublimehq-pub.gpg | sudo tee /etc/apt/keyrings/sublimehq-pub.asc > /dev/null
   ```
2. **Added the stable repository:**
   ```bash
   echo -e 'Types: deb\nURIs: https://download.sublimetext.com/\nSuites: apt/stable/\nSigned-By: /etc/apt/keyrings/sublimehq-pub.asc' | sudo tee /etc/apt/sources.list.d/sublime-text.sources
   ```
3. **Updated and installed:**
   ```bash
   sudo apt update
   sudo apt install sublime-text -y
   ```
4. **Launched it** with `subl`, or from the Xfce applications menu.

## Notes

- Sublime is a GUI app, so launch it from the RDP desktop session or the Proxmox console, not from a plain SSH terminal.
- No license key is needed to start. Sublime shows an occasional purchase prompt while evaluating.
- The `.sources` file and key are not secrets. Nothing in this note needs redacting.
