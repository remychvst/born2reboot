*This activity has been created as part of the 42 curriculum by rchavast.*

# Born2beRoot

## Description

Born2beRoot is a system administration project whose goal is to set up a first server
from scratch, on a virtual machine, following a strict set of security rules.

The virtual machine is configured with:
- An encrypted LVM setup (at least 2 partitions).
- A hardened SSH service running on port `4242` (no root login).
- A firewall (`UFW` on Debian / `firewalld` on Rocky) allowing only port `4242`.
- A strict password policy and `sudo` configuration.
- A `monitoring.sh` script displaying system information every 10 minutes via `wall`.

## Instructions

### Prerequisites
- [VirtualBox](https://www.virtualbox.org/) (or UTM on Apple Silicon).
- An ISO of the latest stable version of **Debian** or **Rocky Linux**.

### Setup
1. Create a new VM in VirtualBox, mount the ISO, and install the OS in **minimal / server** mode
   (no graphical environment — no X.org / Wayland).
2. During installation, set up manual partitioning with encrypted LVM
   (see partitioning table below).
3. After installation, connect to the machine and apply the configuration:
   - `sudo`, SSH, firewall, password policy (see `Project description` section).
   - Deploy `monitoring.sh` and its `cron` job.
4. Reset all passwords (root + user) once every config file is in place.
5. To get the signature required for submission:
```bash
   sha1sum <name_of_vm>.vdi   # or shasum on macOS
```
   Paste the result into `signature.txt` at the root of the repo.

### Testing the setup
```bash
# Check SSH & firewall (Debian)
ss -tunlp
sudo ufw status

# Check SSH & firewall (Rocky)
ss -tunlp
sudo firewall-cmd --list-all

# Check LVM
sudo lvs

# Run monitoring script manually
sudo bash monitoring.sh
```

## Resources

- [Debian Administrator's Handbook](https://debian-handbook.info/)
- [Rocky Linux documentation](https://docs.rockylinux.org/)
- [Arch Wiki – LVM](https://wiki.archlinux.org/title/LVM)
- [UFW documentation](https://help.ubuntu.com/community/UFW)
- [firewalld documentation](https://firewalld.org/documentation/)
- [SELinux User's and Administrator's Guide](https://docs.redhat.com/)
- [AppArmor wiki](https://gitlab.com/apparmor/apparmor/-/wikis/home)
- man pages: `sudo(8)`, `sshd_config(5)`, `pam_pwquality(8)`, `crontab(5)`

**AI usage:** I used an AI assistant to clarify PAM configuration syntax (`pam_pwquality`),
to review my `monitoring.sh` script for bash syntax errors, and to clarify the differences
between AppArmor and SELinux before writing this section. The LVM partitioning, the
sudo/SSH configuration, and the logic of `monitoring.sh` were written and tested by me
on the VM.

## Project description

### Why Debian / Rocky?

| | Debian | Rocky Linux |
|---|---|---|
| Package manager | `apt` / `dpkg` (`.deb`) | `dnf` / `yum` (`.rpm`) |
| Release model | Very stable, slower release cycle | Downstream rebuild of RHEL, enterprise-oriented |
| Default MAC framework | AppArmor | SELinux |
| Default firewall tool | UFW (front-end to iptables/nftables) | firewalld (front-end to nftables) |
| Learning curve | Beginner-friendly | Steeper (SELinux contexts, `dnf` ecosystem) |
| Typical use case | Servers, desktops, general purpose | Enterprise servers, RHEL-compatible environments |

I chose **Debian** for this project because of its simplicity, its large documentation base,
and because AppArmor is easier to approach for a first system administration project.

### AppArmor vs SELinux

Both are **Mandatory Access Control (MAC)** systems that restrict what a process can do,
beyond standard Unix permissions.

- **AppArmor** (Debian) works by attaching security *profiles* to specific binaries/paths.
  It's path-based, easier to write and read, but less granular.
- **SELinux** (Rocky) works with *labels* (contexts) attached to every file, process, and port.
  It's more powerful and fine-grained, but has a steeper learning curve and can be harder to debug
  (`ausearch`, `audit2allow`, etc.).

On my VM, `sudo aa-status` confirms AppArmor is enabled and loaded at startup.

### UFW vs firewalld

- **UFW** (Uncomplicated Firewall) is a simplified front-end for `iptables`/`nftables`,
  used on Debian. Rules are simple (`ufw allow 4242/tcp`) and human-readable.
- **firewalld** (used on Rocky) manages rules dynamically through *zones* and *services*,
  without reloading the whole ruleset, and integrates with SELinux and NetworkManager.

I configured UFW to deny all incoming traffic by default and allow only port 4242/tcp.

### VirtualBox vs UTM

- **VirtualBox** (Oracle) is a free, cross-platform (Windows/Linux/Intel-Mac) type-2 hypervisor,
  widely used and well documented, but not natively supported on Apple Silicon (ARM).
- **UTM** is based on QEMU and is used as an alternative on Apple Silicon Macs, since VirtualBox
  doesn't run natively there. It uses `.qcow2` disk images instead of `.vdi`.

### Design choices

**Partitioning (LVM + LUKS):**

| Partition | Mount point | Approx. size |
|---|---|---|
| `/boot` | `/boot` | ~500MB |
| LVM PV (encrypted) | — | remaining disk |
| `root-vg` | `/` | XX GB |
| `swap-vg` | `[SWAP]` | XX MB |
| `home-vg` | `/home` | XX GB |

**Users & groups:**
- `root` (disabled for SSH login)
- `rchavast` — member of `sudo` and `user42` groups

**Services installed:** `openssh-server` (port 4242), `ufw`, `sudo`, `cron`

**Password policy:** 30-day expiration, 2-day minimum between changes, 7-day warning,
min. 10 characters, uppercase/lowercase/digit required, no more than 3 identical
consecutive characters, no username in password.

**Sudo hardening:** 3 login attempts, custom error message, full input/output logging
in `/var/log/sudo/`, TTY mode required, restricted `secure_path`.
