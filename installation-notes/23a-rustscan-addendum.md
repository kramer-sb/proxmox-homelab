# 23a - RustScan on kali-linux (addendum)

Status: **Working**. RustScan 2.3.0 is installed from the Kali repo on VM 207 (`kali-linux`, `10.0.0.90`) and scanned the Gitea LXC (`10.0.0.10`) successfully. The clean version is the "RustScan" section in [kali-linux.md](../functional-docs/kali-linux.md).

## Goal

Add a fast port scanner to the Kali VM for lab practice. RustScan finds open ports quickly, then hands only those ports to nmap for the slower work (version detection, scripts).

## What I did

1. **Installed it from the Kali repo**: `sudo apt update`, then `sudo apt install rustscan -y`.
2. **Checked the version**: `rustscan --version` printed `rustscan 2.3.0`.
3. **Ran a first scan** against the Gitea LXC, a machine in my own lab, and it found ports 22, 80 and 443 open. That matches what the Gitea LXC should have open (SSH, plus Caddy on 80 and 443).
4. **Fixed the command** after the first attempt failed (see the problems table). The working command is `rustscan -a 10.0.0.10 --ulimit 5000 -- -sV`.

## How the command is split

Everything before the `--` is a RustScan option. Everything after the `--` is passed straight to nmap.

```
rustscan -a 10.0.0.10 --ulimit 5000 -- -sV
         ^^^^^^^^^^^^^^^^^^^^^^^^^^    ^^^^
         RustScan options              nmap options
```

## Problems I hit

| Problem | Cause | Fix |
| --- | --- | --- |
| `Error Exit code = 255` after the three open ports were listed | I wrote `-- -sv ulimit -n 5000`, so RustScan handed `-sv ulimit -n 5000` to nmap, which does not understand it | Put the RustScan options before the `--` and only nmap options after it |
| nmap did not run version detection | I typed `-sv` in lowercase; the nmap flag is `-sV` with a capital V | Use `-sV` |
| Warning: "File limit is lower than default batch size" and "Your file limit is very small" | The shell's open file limit was lower than RustScan's default batch size | Add `--ulimit 5000` before the `--`. Lowering the batch size with `-b 500` also works |

## Notes

- Only scan machines in the lab. Scanning anything outside the home network is not something to do without permission.
- The port scan itself finishes before the "Starting Script(s)" line. If nmap fails afterward, the open ports shown above that line are still valid results.
