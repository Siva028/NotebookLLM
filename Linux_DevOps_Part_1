# PART 1 — LINUX FUNDAMENTALS FOR DEVOPS (Beginner)

> Conventions used throughout the guide
> • Commands shown in modern syntax first; deprecated alternatives noted in *italics*.
> • Distro markers: **[Debian/Ubuntu]** primary, **[RHEL/Rocky/Alma]** noted where it diverges.
> • `$` prompt = unprivileged user, `#` prompt = root/sudo.

---

## 1. Filesystem & Navigation

### 1.1 `pwd` — Print Working Directory

⚙️  **Syntax**
```
pwd [-L|-P]
```
- `-L` (default): show the logical path including symlinks
- `-P`: resolve symlinks and show the physical path

📖 **What it does**
Prints the absolute path of the current directory.

🎯 **DevOps use case**
In CI/CD scripts, you often need to anchor relative paths. `pwd -P` is critical when the build agent's workspace is symlinked (Jenkins agents do this) — relative `../` traversal differs between logical and physical paths.

💻 **Example**
```bash
$ cd /var/log
$ pwd
/var/log

$ ln -s /var/log /tmp/logs && cd /tmp/logs
$ pwd
/tmp/logs
$ pwd -P
/var/log
```

⚠️  **Pitfalls**
- Bash's built-in `pwd` and `/bin/pwd` behave slightly differently for symlink resolution. Inside scripts, prefer `pwd -P` for predictability.

🔗 **Related**: `cd`, `readlink -f`, `realpath`

---

### 1.2 `ls` — List Directory Contents

⚙️  **Syntax**
```
ls [OPTIONS] [PATH...]
```
Most useful flags:

| Flag | Meaning |
|------|---------|
| `-l` | long listing (perms, owner, size, mtime) |
| `-a` | include hidden files (dotfiles) |
| `-A` | hidden files but not `.` / `..` |
| `-h` | human-readable sizes (with `-l`) |
| `-S` | sort by size |
| `-t` | sort by modification time, newest first |
| `-r` | reverse sort |
| `-R` | recursive |
| `-i` | show inode number |
| `-d` | list directory itself, not contents |
| `-1` | one entry per line (great for piping) |
| `--color=auto` | colorize output |

🎯 **DevOps use case**
- `ls -lah` is the daily go-to.
- `ls -ltr` shows newest files at the **bottom** — perfect when tailing log directories to spot the most recently rotated file.
- `ls -1 | xargs ...` is the safe way to pipe filenames when there are no spaces; for safety prefer `find -print0 | xargs -0`.

💻 **Example**
```bash
$ ls -lah /var/log | head -5
total 124M
drwxr-xr-x 11 root   root   4.0K May 21 06:25 .
drwxr-xr-x 14 root   root   4.0K Mar 10 09:12 ..
-rw-r-----  1 syslog adm     12M May 21 11:42 auth.log
-rw-r-----  1 syslog adm    105M May 21 11:42 syslog
drwxr-x---  2 root   adm    4.0K May 21 06:25 apt
```

The 10-character mode string: `drwxr-xr-x`
- char 1: file type (`-` regular, `d` dir, `l` symlink, `b` block, `c` char, `s` socket, `p` pipe)
- chars 2–4: owner perms (read/write/execute)
- chars 5–7: group perms
- chars 8–10: other perms

⚠️  **Pitfalls**
- Don't parse `ls` output in scripts — use `find`, `stat`, or shell globbing instead. `ls` output format varies by locale and version.
- `ls *` will fail with "Argument list too long" in directories with hundreds of thousands of files. Use `find . -maxdepth 1` instead.

🔗 **Related**: `tree`, `stat`, `find`, `du -sh *`

---

### 1.3 `cd` — Change Directory

⚙️  **Syntax**
```
cd [DIR]
cd -          # previous directory
cd ~user      # home directory of user
cd            # own home
```

🎯 **DevOps use case**
`cd -` toggles between two directories — invaluable when bouncing between `/etc/nginx` and `/var/log/nginx` during incident response.

💻 **Example**
```bash
$ cd /etc/nginx
$ cd /var/log/nginx
$ cd -
/etc/nginx
$ cd -
/var/log/nginx
```

⚠️  **Pitfalls**
- `cd` is a shell builtin, not an executable. It cannot be invoked via `sudo cd /root` — use `sudo -i` or `sudo bash -c 'cd /root && ...'`.
- `cd` with no args goes to `$HOME`; under `sudo` that means root's home unless you used `-E` and `HOME` was preserved.

🔗 **Related**: `pushd`, `popd`, `dirs`

---

### 1.4 `tree` — Recursive Directory Visualisation

⚙️  **Syntax**
```
tree [-L depth] [-d] [-a] [-h] [-I pattern] [PATH]
```

Install: `sudo apt install tree` **[Debian/Ubuntu]** / `sudo dnf install tree` **[RHEL]**

🎯 **DevOps use case**
Documenting an Ansible role's directory layout or a Terraform module structure for a runbook.

💻 **Example**
```bash
$ tree -L 2 -d /etc/nginx
/etc/nginx
├── conf.d
├── modules-available
├── modules-enabled
├── sites-available
└── sites-enabled
```

⚠️  **Pitfalls**
- Not installed by default on minimal images. In a Dockerfile audit, fall back to `find . -type d | sed -e 's;[^/]*/;|____;g;s;____|; |;g'`.

🔗 **Related**: `find`, `ncdu`

---

### 1.5 `stat` — Display File Status

⚙️  **Syntax**
```
stat [-c FORMAT] [-L] FILE
```

📖 **What it does**
Shows detailed metadata: size, blocks, inode, perms, UID/GID, access/modify/change/birth times.

🎯 **DevOps use case**
- Investigating "who/what modified this config?" — `Modify` time pinpoints the change.
- Scripting permission audits with `stat -c '%a %U %G %n' FILE`.

💻 **Example**
```bash
$ stat /etc/passwd
  File: /etc/passwd
  Size: 2891      	Blocks: 8          IO Block: 4096   regular file
Device: 252,1	Inode: 657421      Links: 1
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2026-05-21 06:25:11.000000000 +0000
Modify: 2026-04-12 14:02:03.000000000 +0000
Change: 2026-04-12 14:02:03.000000000 +0000
 Birth: 2024-09-01 10:11:00.000000000 +0000
```

- **Access (atime)**: last read
- **Modify (mtime)**: last content change
- **Change (ctime)**: last metadata or content change (perms, owner, content)
- **Birth (crtime)**: file creation (ext4+, not always populated)

⚠️  **Pitfalls**
- Many filesystems are mounted `noatime` or `relatime` for performance — `atime` may not reflect actual reads.
- `ctime` cannot be set by the user; it's what forensics relies on.

🔗 **Related**: `ls -l`, `find -mtime`, `touch`

---

### 1.6 `file` — Determine File Type by Content

⚙️  **Syntax**
```
file [-b] [-i] FILE...
```
- `-b` brief (no filename prefix)
- `-i` MIME type output

🎯 **DevOps use case**
A binary lands in `/tmp` during incident response — `file` tells you whether it's an ELF executable, shell script, gzipped tarball, or PE32 (Windows binary uploaded by mistake).

💻 **Example**
```bash
$ file /usr/bin/python3 /etc/hosts /tmp/payload
/usr/bin/python3: symbolic link to python3.12
/etc/hosts:       ASCII text
/tmp/payload:     ELF 64-bit LSB pie executable, x86-64, dynamically linked
```

🔗 **Related**: `strings`, `readelf`, `ldd`

---

### 1.7 `readlink` / `realpath` — Resolve Symlinks

⚙️  **Syntax**
```
readlink [-f] PATH      # -f resolves recursively, even non-existent components
realpath PATH           # always resolves to canonical absolute path
```

🎯 **DevOps use case**
- Build scripts: `SCRIPT_DIR="$(cd "$(dirname "$(readlink -f "$0")")" && pwd)"` — the canonical "where am I really" idiom.
- Verifying which binary `kubectl` actually points to when multiple versions are installed.

💻 **Example**
```bash
$ readlink /usr/bin/python3
python3.12
$ readlink -f /usr/bin/python3
/usr/bin/python3.12
$ realpath /etc/../var/log/./syslog
/var/log/syslog
```

🔗 **Related**: `ln -s`, `which`, `command -v`

---

### 1.8 Filesystem Hierarchy Standard (FHS)

| Path | Purpose | DevOps relevance |
|------|---------|------------------|
| `/` | Root of the filesystem tree | Mount point of root FS |
| `/bin`, `/sbin` | Essential binaries (symlinked to `/usr/{bin,sbin}` on modern distros) | Where core tools live |
| `/etc` | System-wide configuration | **Where 90% of your config changes happen** |
| `/var` | Variable data: logs, spool, caches | `/var/log`, `/var/lib/docker`, `/var/lib/postgresql` |
| `/var/log` | System and app logs | First stop for troubleshooting |
| `/tmp` | World-writable temp, cleared on boot (often tmpfs) | Don't store anything you need to keep |
| `/run` | Runtime data (tmpfs); replaces `/var/run` | PID files, sockets |
| `/usr` | User binaries, libraries, docs (read-only after install) | `/usr/local/bin` for site-local installs |
| `/usr/local` | Locally-compiled software (out of package manager's way) | Where `make install` defaults |
| `/opt` | Add-on application packages (vendor-supplied) | Datadog agent, Splunk, etc. install here |
| `/home` | User home directories | Often a separate partition |
| `/root` | Root user's home | Not under `/home` so it's available in single-user mode |
| `/boot` | Kernel, initramfs, bootloader files | Don't fill this — kernel updates will fail |
| `/proc` | Virtual FS — kernel and process info | Live system introspection |
| `/sys` | Virtual FS — kernel objects (devices, modules) | Tuning hardware/kernel knobs |
| `/dev` | Device files | `/dev/null`, `/dev/sda`, `/dev/random` |
| `/mnt`, `/media` | Mount points (manual / removable) | Where to mount NFS shares, USB drives |
| `/srv` | Site-specific service data | Less commonly used in cloud-native shops |

🎯 **DevOps reality check**
- `/proc/<pid>/` is your X-ray vision into a running process — environment, open files, cgroup, mountinfo, status.
- `/proc/meminfo`, `/proc/cpuinfo`, `/proc/loadavg` — what `free`, `top`, `uptime` actually read.
- `/sys/class/net/<iface>/` — link state, MTU, statistics for a NIC.
- `/run/systemd/` — systemd runtime state.

---

### 1.9 Inodes, Hard Links, Symbolic Links

📖 **Concept**
- Every file is an **inode** (a metadata record on disk: perms, owner, size, block pointers).
- A **filename** is just a directory entry pointing to an inode.
- A **hard link** = additional directory entry pointing to the same inode. Indistinguishable from the "original."
- A **symbolic link** = a tiny file whose content is a path string. Resolved at access time.

⚙️  **Syntax**
```
ln  SOURCE   LINK_NAME       # hard link
ln -s SOURCE LINK_NAME       # symbolic link
ln -sf SOURCE LINK_NAME      # force overwrite existing
```

🎯 **DevOps use case**
- Deployment pattern: `app -> releases/v2.3.1/`. Atomic switchover by repointing the symlink:
  ```bash
  ln -sfn /opt/app/releases/v2.3.2 /opt/app/current
  ```
  `-n` treats the target as a regular file even if it's an existing dir-symlink (prevents creating the link **inside** the old target).
- `/etc/nginx/sites-enabled/foo -> ../sites-available/foo` is the classic enable/disable pattern.

💻 **Example**
```bash
$ echo "hello" > a.txt
$ ln a.txt b.txt          # hard link
$ ln -s a.txt c.txt       # symlink
$ ls -li
657421 -rw-r--r-- 2 root root 6 May 21 11:50 a.txt
657421 -rw-r--r-- 2 root root 6 May 21 11:50 b.txt
657422 lrwxrwxrwx 1 root root 5 May 21 11:50 c.txt -> a.txt
```
Note `a.txt` and `b.txt` share inode `657421` and a link count of `2`. Deleting `a.txt` leaves the data intact — `b.txt` still references the inode.

⚠️  **Pitfalls**
- Hard links **cannot cross filesystems** (different inode tables). Symlinks can.
- Hard links to directories are forbidden on most filesystems (would create cycles).
- Symlinks to relative paths break if you move the symlink. Prefer absolute paths in production.
- `rsync -a` preserves symlinks-as-symlinks; `rsync -aL` follows them and copies the contents.

🔗 **Related**: `stat`, `find -inum`, `unlink`

---

## 2. File Operations

### 2.1 `touch` — Create empty files / update timestamps

⚙️  **Syntax**
```
touch [-a|-m] [-t STAMP|-d DATE] [-c] FILE...
```
- `-a` only atime, `-m` only mtime
- `-c` don't create if missing
- `-t [[CC]YY]MMDDhhmm[.ss]` or `-d "2026-01-15 10:00"` for explicit timestamp

🎯 **DevOps use case**
- `touch /forcefsck` (legacy) to force a filesystem check on next boot.
- Marking sentinel files: `touch /var/run/myapp.started` from a systemd `ExecStartPost=`.
- Bumping the mtime of a Makefile target to force a rebuild.

💻 **Example**
```bash
$ touch deploy.lock
$ touch -d "2 days ago" old.log
$ stat -c %y old.log
2026-05-19 11:52:14.000000000 +0000
```

🔗 **Related**: `install`, `mktemp`

---

### 2.2 `cp` — Copy files

⚙️  **Syntax**
```
cp [OPTIONS] SOURCE... DEST
```

| Flag | Meaning |
|------|---------|
| `-r`, `-R` | recursive (for directories) |
| `-a` | archive: preserves perms, owners, timestamps, symlinks (`-dR --preserve=all`) |
| `-p` | preserve mode, ownership, timestamps |
| `-i` | interactive — prompt before overwrite |
| `-n` | no-clobber — never overwrite |
| `-u` | update — only copy if source is newer |
| `-v` | verbose |
| `--reflink=auto` | use COW on supported FSes (btrfs, XFS reflink) |
| `--sparse=always` | preserve sparse files |

🎯 **DevOps use case**
- `cp -a` for migrating a service's data directory between hosts (with rsync as a better alternative for large or remote copies).
- `cp -au src/ dst/` for incremental local backups.

💻 **Example**
```bash
$ cp -av /etc/nginx /backup/nginx-$(date +%F)
'/etc/nginx' -> '/backup/nginx-2026-05-21'
'/etc/nginx/nginx.conf' -> '/backup/nginx-2026-05-21/nginx.conf'
...
```

⚠️  **Pitfalls**
- `cp src/ dst/` and `cp src dst/` differ when `src` ends in `/` — see GNU cp's "trailing slash" rules. Test in a scratch directory.
- Without `-a`, ownership and SELinux contexts may be lost.

🔗 **Related**: `rsync`, `install`, `dd`

---

### 2.3 `mv` — Move / rename files

⚙️  **Syntax**
```
mv [-f|-i|-n] [-v] SOURCE... DEST
```

📖 Within the same filesystem, `mv` is an atomic rename (inode unchanged). Across filesystems, it becomes a copy-then-delete — and is **not** atomic.

🎯 **DevOps use case**
- Atomic config swap: write `/etc/nginx/nginx.conf.new`, validate, then `mv nginx.conf.new nginx.conf && nginx -s reload`.
- Log rotation pattern: `mv app.log app.log.1 && touch app.log` (but use `logrotate` in production).

⚠️  **Pitfalls**
- "Atomic across filesystems" is a myth — confirm with `stat -c %m` (mount point) or `df` that source and dest are on the same FS.
- `mv` doesn't follow `--reflink`; copies across FS are full byte copies.

🔗 **Related**: `rename`, `cp`, `install`

---

### 2.4 `rm` — Remove files

⚙️  **Syntax**
```
rm [-f] [-i] [-r] [-v] FILE...
```
- `-r` recursive (directories), `-f` force (no prompts), `-i` interactive

🎯 **DevOps use case**
Cleaning up old release artifacts in a deployment script. Always combine with `set -euo pipefail` and **always** parameter-check before `rm -rf`.

⚠️  **Pitfalls — read carefully**
- `rm -rf /` is a meme but easy to produce by accident: `rm -rf "$DIR/"` where `$DIR=""`. **Always** guard:
  ```bash
  : "${DIR:?DIR not set}"
  rm -rf -- "${DIR:?}"/
  ```
  `${VAR:?msg}` aborts if `VAR` is empty or unset.
- `rm` does not free space if a process still has the file open (see `lsof | grep deleted`).
- Some `rm` implementations refuse `/`; pass `--no-preserve-root` to bypass (don't).

🔗 **Related**: `find -delete`, `shred`, `trash-cli`

---

### 2.5 `mkdir` / `rmdir`

⚙️  **Syntax**
```
mkdir [-p] [-m MODE] [-v] DIR...
rmdir [-p] DIR...
```
- `-p`: create parents as needed; do not error if exists
- `-m`: set mode at creation (avoid the race between `mkdir` + `chmod`)

🎯 **DevOps use case**
```bash
install -d -m 0750 -o app -g app /var/lib/myapp
```
`install -d` is `mkdir -p` plus mode/owner/group in one syscall — preferred in idempotent provisioning.

💻 **Example**
```bash
$ mkdir -p /var/lib/myapp/{data,cache,logs}
$ tree /var/lib/myapp
/var/lib/myapp
├── cache
├── data
└── logs
```

🔗 **Related**: `install -d`, `mktemp -d`

---

### 2.6 `shred` — Securely overwrite files

⚙️  **Syntax**
```
shred [-n N] [-z] [-u] FILE...
```
- `-n N` overwrite N times (default 3)
- `-z` final pass with zeros
- `-u` deallocate and remove file after overwriting

🎯 **DevOps use case**
Decommissioning a host that held secrets on bare-metal disks. **Useless on SSDs and copy-on-write filesystems** (btrfs, ZFS, log-structured FS) — the blocks may be remapped. For SSDs use `blkdiscard` or vendor secure-erase.

⚠️  **Pitfalls**
- `shred` is also ineffective on journaled filesystems where metadata is journaled (ext3/4 with `data=journal`).
- For real assurance, encrypt at rest (LUKS) and destroy the key.

🔗 **Related**: `blkdiscard`, `dd if=/dev/zero of=/dev/sdX`, `cryptsetup luksErase`

---

### 2.7 `cat`, `less`, `more`, `head`, `tail`

⚙️  **Syntax**
```
cat  [-n] [-A] FILE...               # concatenate / dump
less [-N] [-S] [-R] FILE             # pager (preferred over more)
head [-n N] [-c BYTES] FILE
tail [-n N] [-c BYTES] [-f] [-F] FILE
```

| Tool | Best for |
|------|----------|
| `cat` | Small files, piping; **never** for big logs in a terminal |
| `less` | Interactive paging; `/` to search, `n`/`N` next/prev, `G` end, `g` start, `F` follow |
| `head -n 50` | First N lines |
| `tail -n 50` | Last N lines |
| `tail -f` | Follow file as it grows (won't re-open on rotation) |
| `tail -F` | Follow **and** re-open on rotation — what you want for live logs |

🎯 **DevOps use case**
- `journalctl -u nginx -f` for systemd-managed services (preferred over `tail -F`).
- `tail -F /var/log/nginx/error.log` for non-journald logs.
- `cat -A` (or `cat -vET`) to spot CRLF/Windows line endings, tabs, trailing whitespace in config files.

💻 **Example**
```bash
$ tail -F /var/log/syslog &
$ tail -n 100 /var/log/auth.log | grep -i 'failed password'
```

⚠️  **Pitfalls**
- "Useless use of cat": `cat file | grep pattern` → write `grep pattern file`.
- `tail -f` on a rotated file = silent log loss. Use `-F`.
- `less` respects `LESSOPEN`; `less /tmp/file.gz` will decompress on the fly.

🔗 **Related**: `multitail`, `lnav`, `journalctl -f`

---

### 2.8 `find` — The Swiss Army Knife

⚙️  **Syntax**
```
find [PATH...] [EXPRESSION]
```

Key operands:

| Operand | Meaning |
|---------|---------|
| `-name PATTERN` | filename glob (case-sensitive); `-iname` for case-insensitive |
| `-type f\|d\|l\|s\|b\|c\|p` | regular file / dir / symlink / socket / block / char / pipe |
| `-mtime [+\|-]N` | modified N days ago (`+7` = older than 7d, `-1` = within 1d) |
| `-mmin [+\|-]N` | minutes-precision version |
| `-size [+\|-]N[kMG]` | size filter |
| `-perm MODE` | permissions |
| `-user U` / `-group G` | owner |
| `-empty` | zero-byte files / empty directories |
| `-maxdepth N` / `-mindepth N` | depth control |
| `-prune` | don't descend into matching directories |
| `-exec CMD {} \;` | exec per-match |
| `-exec CMD {} +` | exec once with batched args (faster) |
| `-delete` | delete matches |
| `-print0` | NUL-separated for safe piping into `xargs -0` |

🎯 **DevOps use cases**

```bash
# 1. Delete logs older than 14 days
find /var/log/myapp -type f -name '*.log' -mtime +14 -delete

# 2. Find files >1G under /var, sorted by size
find /var -type f -size +1G -exec du -h {} + | sort -h

# 3. Files modified in the last 10 minutes (incident triage)
find /etc -type f -mmin -10

# 4. Reset perms on a tree (be very careful)
find /srv/web -type d -exec chmod 0755 {} +
find /srv/web -type f -exec chmod 0644 {} +

# 5. Skip a noisy directory
find / -path /proc -prune -o -name 'core.*' -print

# 6. Find world-writable files (security audit)
find / -xdev -type f -perm -o+w 2>/dev/null
```

⚠️  **Pitfalls**
- `-mtime 7` means *exactly* 7 days, not "in the last 7 days" — use `-mtime -7`.
- `-name` does **not** use regex; use `-regex` (and `-regextype posix-extended`).
- Default action when no action is given is `-print`. When using `-prune`, you typically need an explicit `-print`.
- Always quote `-name` patterns to prevent shell expansion: `find . -name '*.log'`.

🔗 **Related**: `locate`, `fd`, `xargs`, `parallel`

---

### 2.9 `locate`, `which`, `whereis`, `type`

| Tool | What it finds | Source of truth |
|------|---------------|-----------------|
| `locate PATTERN` | Files matching pattern | Pre-built DB (`/var/lib/mlocate`), refresh with `updatedb` |
| `which CMD` | First executable in `PATH` | Live `PATH` search (external tool, may not see aliases) |
| `whereis CMD` | Binary, source, and manpage locations | Standard system paths |
| `type CMD` | What the shell will actually run | Shell builtin — sees aliases, functions, builtins |

🎯 **DevOps use case**
- Use `type -a` to disambiguate: is `ls` an alias to `ls --color=auto`? Is `cd` a builtin?
- Use `command -v CMD` in scripts — it's POSIX and reliable; `which` is not portable.

💻 **Example**
```bash
$ type -a ls
ls is aliased to 'ls --color=auto'
ls is /usr/bin/ls

$ command -v terraform
/usr/local/bin/terraform
```

⚠️  **Pitfalls**
- `locate` may be missing on minimal images; install `mlocate` or `plocate`.
- `which` was deprecated on Debian 12+ in favor of `command -v`.

---

## 3. Text Processing (CRITICAL for DevOps)

> Text processing is the single most important skill for daily Linux DevOps work. Logs, configs, API responses, build output — everything is text.

### 3.1 `grep` — Pattern Search

⚙️  **Syntax**
```
grep [OPTIONS] PATTERN [FILE...]
```

| Flag | Meaning |
|------|---------|
| `-i` | case-insensitive |
| `-v` | invert (lines NOT matching) |
| `-r`, `-R` | recursive; `-R` follows symlinks |
| `-n` | line numbers |
| `-H` | always print filename |
| `-h` | never print filename |
| `-l` | only filenames containing match |
| `-L` | only filenames NOT containing match |
| `-c` | count matches per file |
| `-w` | match whole words |
| `-x` | match whole line |
| `-E` | extended regex (same as `egrep`) |
| `-F` | fixed strings (same as `fgrep`); fastest, no regex |
| `-P` | Perl-compatible regex (where supported) |
| `-A N` | print N lines **after** each match |
| `-B N` | N lines **before** |
| `-C N` | N lines of context (both sides) |
| `-o` | only the matching part of the line |
| `--include='*.py'` | only files matching glob |
| `--exclude-dir=.git` | skip directories |
| `--color=auto` | highlight matches |
| `-z` | NUL-separated input (multi-line records) |

🎯 **DevOps use cases**

```bash
# 1. Find a config key across /etc
grep -rni --color 'listen' /etc/nginx

# 2. Last 50 errors in an app log, with 2 lines of context after
tail -n 10000 app.log | grep -A2 -i 'error'

# 3. Count failed SSH logins
grep -c 'Failed password' /var/log/auth.log

# 4. Inverse: lines NOT containing healthcheck
grep -v ' /healthz ' access.log

# 5. Exit-code use in scripts (0 if found, 1 if not, 2 on error)
if grep -q 'PermitRootLogin no' /etc/ssh/sshd_config; then
    echo "secure"
fi

# 6. Recursive search excluding noise
grep -rn 'TODO' src/ --include='*.go' --exclude-dir=vendor
```

⚠️  **Pitfalls**
- Default regex is **BRE** (Basic Regular Expression): `+`, `?`, `|`, `(...)` are literal characters unless escaped. Use `-E` (ERE) for the syntax you probably expect.
- `grep -P` is not available everywhere (busybox grep lacks it).
- A single literal dot in a pattern matches any character: use `-F` for literal strings.
- `grep` exits non-zero when no match found — under `set -e` this kills your script. Use `grep || true` or `grep -c`.

🔗 **Related**: `ripgrep` (`rg`), `ag`, `awk`, `sed`

---

### 3.2 `sed` — Stream Editor

⚙️  **Syntax**
```
sed [OPTIONS] 'SCRIPT' [FILE...]
```

| Flag | Meaning |
|------|---------|
| `-n` | suppress default output; print only what `p` requests |
| `-E` (or `-r`) | extended regex |
| `-i[SUFFIX]` | in-place edit (with optional backup suffix) |
| `-e SCRIPT` | add a script; multiple `-e` allowed |
| `-f FILE` | read script from file |

Core commands within a script:

| Cmd | Action |
|-----|--------|
| `s/PATTERN/REPLACEMENT/FLAGS` | substitute |
| `d` | delete current line |
| `p` | print current line |
| `q` | quit |
| `N` | append next line to pattern space (multiline) |
| `a TEXT` | append after match |
| `i TEXT` | insert before match |
| `c TEXT` | replace match |

Substitute flags: `g` global, `i` case-insensitive, `N` Nth occurrence, `p` print, `w FILE` write.

🎯 **DevOps use cases**

```bash
# 1. Replace a value in a config (in-place, with backup)
sed -i.bak 's/^Port .*/Port 2222/' /etc/ssh/sshd_config

# 2. Delete blank lines
sed -i '/^$/d' file.conf

# 3. Delete comments and blank lines (useful for config diffs)
sed -E '/^\s*(#|$)/d' /etc/nginx/nginx.conf

# 4. Print only lines 100-150
sed -n '100,150p' big.log

# 5. Insert a header line
sed -i '1i # Generated by Ansible — do not edit' /etc/myapp.conf

# 6. Replace only the first match per file
sed -i '0,/foo/s//bar/' file

# 7. Multi-file in-place edit
find . -name '*.yaml' -exec sed -i 's/v1.2.3/v1.2.4/g' {} +
```

⚠️  **Pitfalls**
- `-i` without a suffix works in GNU sed; BSD sed (macOS) **requires** `-i ''`. Write scripts for the target environment.
- `&` in the replacement = the whole match. To insert a literal `&`, use `\&`.
- Choose a delimiter that doesn't conflict with your data: `sed 's|/old/path|/new/path|g'` is cleaner than escaping slashes.
- `sed -i` breaks symlinks — it writes to a temp file and renames, replacing the link with a regular file.

🔗 **Related**: `awk`, `perl -pi -e`, `ed`

---

### 3.3 `awk` — Text Processing Powerhouse

⚙️  **Syntax**
```
awk [OPTIONS] 'PROGRAM' [FILE...]
```

Program structure: `PATTERN { ACTION }`. Either may be omitted.

Built-in variables:

| Var | Meaning |
|-----|---------|
| `$0` | entire current line |
| `$1`, `$2`, ... | fields (default split on whitespace) |
| `NF` | number of fields on this line |
| `NR` | line number (record number) |
| `FNR` | line number within current file |
| `FS` | input field separator (default whitespace; use `-F` to set) |
| `OFS` | output field separator |
| `RS`, `ORS` | input/output record separator (default `\n`) |
| `FILENAME` | current input filename |

Special blocks: `BEGIN { ... }` runs before input; `END { ... }` runs after.

🎯 **DevOps use cases**

```bash
# 1. Print column 7 (request URL) from nginx access log
awk '{print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head

# 2. Sum a column (e.g., bytes transferred = col 10)
awk '{sum += $10} END {print sum}' access.log

# 3. Filter by field and print specific cols
awk '$9 >= 500 {print $1, $7, $9}' access.log    # all 5xx responses

# 4. Custom delimiter — parse /etc/passwd
awk -F: '{print $1, $7}' /etc/passwd            # user, shell

# 5. Multiple actions per pattern
awk -F: '$3 >= 1000 && $7 != "/usr/sbin/nologin" {print $1}' /etc/passwd

# 6. Format-rich report
df -h | awk 'NR>1 && $5+0 > 80 {printf "%-20s %s used\n", $6, $5}'

# 7. Per-key aggregation with associative arrays
awk '{count[$1]++} END {for (k in count) print count[k], k}' access.log \
  | sort -rn | head
```

📌 **Anatomy of an awk one-liner**

```bash
awk -F: '$3 >= 1000 {print $1}' /etc/passwd
#    │   └── pattern ─┘ └── action ──┘
#    └── field separator
```

⚠️  **Pitfalls**
- Field numbering starts at 1, not 0. `$0` is the whole line.
- Numeric vs string comparison: `$3 >= 1000` compares numerically only if `$3` looks like a number. Force numeric with `+0`: `$3+0 > 80`.
- `awk` vs `gawk` vs `mawk`: features like `gensub()` are GNU-only. Be explicit in scripts.
- Default `FS` is "any amount of whitespace, leading/trailing trimmed" — different from `-F' '` (single space).

🔗 **Related**: `cut`, `perl -lane`, `miller` (`mlr`)

---

### 3.4 `cut`, `sort`, `uniq`, `wc`, `tr`, `paste`, `join`, `comm`

#### `cut`
```
cut -d DELIM -f FIELDS
cut -c POS
```
```bash
$ cut -d: -f1,7 /etc/passwd          # users and shells
$ cut -c1-10 file.txt                # first 10 chars per line
```
⚠️  Unlike `awk`, `cut -d ' '` treats every single space as a delimiter (consecutive spaces produce empty fields).

#### `sort`
```
sort [-n] [-r] [-k FIELD] [-t DELIM] [-u]
```
- `-n` numeric, `-h` human-numeric (`10K`, `2.3G`), `-r` reverse
- `-k N` sort by Nth field, `-k 2,2n` field 2 numeric only
- `-u` deduplicate
- `-t :` set delimiter
- `--parallel=N` parallel sort for big files

```bash
$ sort -t: -k3 -n /etc/passwd | head    # users by UID
$ du -sh /var/log/* | sort -h           # by human-readable size
```

#### `uniq`
```
uniq [-c] [-d] [-u]
```
- Operates on **adjacent** lines — always `sort` first.
- `-c` count occurrences, `-d` only duplicates, `-u` only unique

```bash
$ awk '{print $1}' access.log | sort | uniq -c | sort -rn | head
   4521 10.0.0.5
   3812 10.0.0.7
    921 192.168.1.42
```

#### `wc`
```
wc [-l|-w|-c|-m]
```
- `-l` lines, `-w` words, `-c` bytes, `-m` chars

```bash
$ wc -l /var/log/syslog
27194 /var/log/syslog
```

#### `tr`
```
tr [OPTIONS] SET1 [SET2]
```
- `-d` delete chars in SET1
- `-s` squeeze repeated chars
- `-c` complement of SET1

```bash
$ echo "Hello World" | tr 'a-z' 'A-Z'
HELLO WORLD
$ tr -d '\r' < windows.txt > unix.txt        # strip CRs
$ tr -s ' ' < file                          # collapse runs of spaces
```

#### `paste`
Merges files side-by-side.
```bash
$ paste -d, names.txt ages.txt
alice,30
bob,25
```

#### `join`
SQL-like join on a key field; **both inputs must be sorted** on the key.
```bash
$ join -t: -1 1 -2 1 <(sort -t: -k1 /etc/passwd) <(sort -t: -k1 /etc/shadow)
```

#### `comm`
Compare two **sorted** files; outputs three columns (only-in-1, only-in-2, in-both).
```bash
$ comm -23 <(sort a.txt) <(sort b.txt)    # lines only in a.txt
```

#### `diff` & `patch`
```bash
$ diff -u old.conf new.conf > my.patch
$ patch < my.patch                       # apply
$ patch -R < my.patch                    # reverse (undo)
$ diff -ruN dir1/ dir2/                  # recursive, new files included
```

---

### 3.5 `xargs` — Build & Execute Command Lines

⚙️  **Syntax**
```
xargs [OPTIONS] [COMMAND [INITIAL-ARGS...]]
```

| Flag | Meaning |
|------|---------|
| `-0` | NUL-separated input (pair with `find -print0`) |
| `-n N` | use N args per command invocation |
| `-I {}` | placeholder for substitution |
| `-P N` | run N commands in parallel |
| `-t` | echo command before running |
| `-r` | don't run if input empty (GNU; `--no-run-if-empty`) |
| `-a FILE` | read args from FILE instead of stdin |

🎯 **DevOps use cases**

```bash
# 1. Safe deletion: NUL-separated piping
find /tmp -type f -name '*.cache' -print0 | xargs -0 rm -f

# 2. Parallel compression across many files
find . -name '*.log' -print0 | xargs -0 -P 8 -n 1 gzip

# 3. Restart all services matching a pattern
systemctl list-units --type=service --state=running --no-legend \
  | awk '{print $1}' | grep 'app-' | xargs -r -n1 systemctl restart

# 4. -I substitution for non-trailing position
ls /backups/*.tar.gz | xargs -I {} cp {} /mnt/dr/{}.copy

# 5. Combine with kubectl
kubectl get pods --no-headers -o custom-columns=:.metadata.name \
  | xargs -I {} kubectl logs {} --tail=20
```

⚠️  **Pitfalls**
- **Always** use `-0` with `find -print0` if filenames could contain spaces or newlines.
- Default behavior batches inputs to fit `ARG_MAX`. To run once-per-input, use `-n 1` or `-I {}`.
- `-P` is **not** ordered — output from parallel runs will interleave.
- With `-I {}`, the placeholder appears literally where you put it; `-n` and `-L` are ignored.

🔗 **Related**: `parallel`, `find -exec`, `printf | xargs`

---

## 4. Permissions & Ownership

### 4.1 Permission Model Recap

Every file has:
- **Owner** (user) — UID
- **Group** — GID
- **Mode** — 9 bits (rwx for user, group, other) + 3 special bits (SUID, SGID, sticky)

Modes can be expressed as:
- **Octal**: 3 digits, each summing `r=4 + w=2 + x=1`. E.g. `0644` = rw-r--r--.
- **Symbolic**: `u`/`g`/`o`/`a` (user/group/other/all) + `+`/`-`/`=` + `r`/`w`/`x`. E.g. `u+x,g-w`.

The optional 4th octal digit (leading) sets special bits: `4000` SUID, `2000` SGID, `1000` sticky.

### 4.2 `chmod` — Change Mode

⚙️  **Syntax**
```
chmod [-R] MODE FILE...
```

🎯 **DevOps use cases**

```bash
chmod 0755 /opt/app/bin/run.sh                # executable for owner, rx for everyone
chmod u+x,g-w deploy.sh                       # symbolic
chmod -R u=rwX,g=rX,o= /var/lib/myapp         # capital X: set x only on dirs and already-x files
chmod 0600 ~/.ssh/id_ed25519                  # private keys MUST be 600
```

The **capital `X`** in symbolic mode is special: applies `x` only to directories or files that already have at least one `x` bit. Indispensable when re-permissioning a tree.

⚠️  **Pitfalls**
- `chmod -R 777 .` is one of the most damaging "fixes" — it breaks SSH, sudo, and security models, and is rarely the actual fix.
- SUID/SGID on shell scripts is ignored by the kernel on Linux (security feature). Wrap in a compiled binary if you really need it.

🔗 **Related**: `umask`, `chown`, `find -perm`, `setfacl`

---

### 4.3 `chown` & `chgrp`

⚙️  **Syntax**
```
chown [-R] [-h] OWNER[:GROUP] FILE...
chgrp [-R] GROUP FILE...
```

- `-h` operate on symlinks themselves, not their targets
- `:GROUP` (no user) changes only group
- `--reference=FILE` copy ownership from another file

```bash
chown -R app:app /var/lib/myapp
chown :nginx /var/log/nginx/*.log         # group only
```

⚠️  Recursive chown across a tree containing symlinks can produce surprises with `-L`/`-P`. Default is don't-follow.

---

### 4.4 `umask` — Default Permission Mask

The umask is **subtracted** from the default creation mode (`0666` for files, `0777` for dirs).

```bash
$ umask
0022
$ umask 0027              # group-readable but not other-readable
$ touch /tmp/test
$ stat -c %a /tmp/test
640
```

🎯 **DevOps use case**
Set `umask 0027` in `/etc/profile.d/hardening.sh` for tighter default permissions on multi-user systems.

---

### 4.5 Special Bits: SUID, SGID, Sticky

| Bit | Octal | On file | On directory |
|-----|-------|---------|--------------|
| SUID | `4000` | runs with owner's UID (e.g. `passwd`) | N/A |
| SGID | `2000` | runs with owner's GID | new files inherit dir's GID |
| Sticky | `1000` | (legacy) | only owner can delete their files (e.g. `/tmp`) |

How to spot them in `ls -l`:
- `-rwsr-xr-x` SUID (lowercase `s` = SUID+x; uppercase `S` = SUID without x)
- `-rwxr-sr-x` SGID
- `drwxrwxrwt` sticky on `/tmp`

🎯 **DevOps use case**
- SGID on shared project directories so all new files belong to the team group:
  ```bash
  chgrp dev /srv/project && chmod 2775 /srv/project
  ```
- Audit SUID binaries — anything unexpected is a privilege escalation risk:
  ```bash
  find / -xdev -perm -4000 -type f 2>/dev/null
  ```

---

### 4.6 ACLs — `getfacl` / `setfacl`

When traditional UGO permissions aren't expressive enough (e.g., "user `alice` needs rw, user `bob` needs r, group `devs` needs rwx"), use ACLs.

Requires the filesystem to be mounted with `acl` option (default on ext4 in modern distros).

⚙️  **Syntax**
```
setfacl [-R] [-m|-x] ACL_SPEC FILE
getfacl FILE
```

ACL spec format: `[d:][u|g|o|m]:[name]:perms`
- `d:` = default ACL (inherited by new entries in a directory)
- `m:` = mask (caps the max effective perms granted by ACLs)

🎯 **DevOps use cases**

```bash
# Grant alice rw on a specific file
setfacl -m u:alice:rw /var/lib/myapp/secret.conf

# Default ACL on a directory — new files inherit
setfacl -d -m g:devs:rwx /srv/project
setfacl -d -m o::--- /srv/project

# View
getfacl /var/lib/myapp/secret.conf

# Remove
setfacl -x u:alice /var/lib/myapp/secret.conf

# Wipe all ACLs
setfacl -b FILE
```

The `+` after the mode in `ls -l` indicates an ACL is set: `-rw-r--r--+`.

⚠️  **Pitfalls**
- `chmod` modifies the ACL mask and can silently reduce ACL-granted permissions. Re-apply ACLs after `chmod`.
- ACLs are preserved by `cp -p`/`cp -a`, `rsync -A`, `tar --acls`, **not** by default `cp` or scp.

---

### 4.7 `sudo`, `/etc/sudoers`, `visudo`

`sudo` lets specified users run commands as root (or another user) with fine-grained control.

Configuration lives in `/etc/sudoers` and `/etc/sudoers.d/*`. **Always edit with `visudo`** — it validates syntax before saving, preventing a broken sudoers file from locking you out.

⚙️  **Syntax**
```
sudo [-u USER] [-i] [-E] COMMAND
sudo -l                 # list what you're allowed to run
sudo -k                 # invalidate cached credentials
visudo [-c]             # check sudoers syntax
visudo -f /etc/sudoers.d/myrule
```

🎯 **DevOps use cases**

```bash
# Grant the deploy user passwordless restart of nginx (place in /etc/sudoers.d/deploy)
deploy ALL=(root) NOPASSWD: /bin/systemctl restart nginx, /bin/systemctl reload nginx

# Run a command as another service user
sudo -u postgres psql

# Launch a login shell as root (loads root's environment)
sudo -i
```

⚠️  **Pitfalls**
- **Never** `sudo NOPASSWD: ALL` for unprivileged users — equivalent to giving them root.
- `Defaults env_reset` (the default) wipes most environment vars. Use `sudo -E` to preserve, or `env_keep` in sudoers (sparingly).
- `sudo` without a TTY may behave differently (no password prompt → fails). Use `requiretty` cautiously.

🔗 **Related**: `su -`, `polkit`, `doas`

---

## 5. Users & Groups

### 5.1 The Files

| File | Content | Notable |
|------|---------|---------|
| `/etc/passwd` | user info, world-readable | hashed passwords NOT here anymore |
| `/etc/shadow` | password hashes, expiry policy, root-only | `!` or `*` = locked / no password |
| `/etc/group` | groups and membership | |
| `/etc/gshadow` | group passwords (rarely used) | |

**`/etc/passwd` format** (colon-separated):
```
username:x:UID:GID:GECOS:home_dir:login_shell
```
- `x` is a placeholder — actual hash is in `/etc/shadow`
- `GECOS` is the user description (historically — full name, room, phone)

**`/etc/shadow` format**:
```
username:hash:lastchange:min:max:warn:inactive:expire:reserved
```
The hash field starts with `$ID$` indicating algorithm: `$6$` SHA-512, `$y$` yescrypt (modern default), `$1$` MD5 (insecure).

**`/etc/group` format**:
```
groupname:x:GID:user1,user2,...
```

### 5.2 User Management

⚙️  **Commands**
```
useradd  [-m] [-s SHELL] [-g PRIMARY_GROUP] [-G GROUPS] [-u UID] [-d HOME] [-c COMMENT] USER
usermod  [-aG GROUP] [-s SHELL] [-L] [-U] [-l NEWNAME] USER
userdel  [-r] USER                   # -r removes home + mail spool
passwd   USER                        # set password
chage    [-l] [-E DATE] [-M DAYS] USER   # password aging
```

🎯 **DevOps use cases**

```bash
# Create a service user (no login, no home shell)
useradd -r -s /usr/sbin/nologin -d /var/lib/myapp -M myapp

# Create an interactive admin user
useradd -m -s /bin/bash -G sudo,docker alice
passwd alice

# Add an existing user to a group (note -aG together!)
usermod -aG docker bob

# Lock / unlock a user
usermod -L bob       # lock (prepends ! to hash)
usermod -U bob       # unlock
passwd -l bob        # lock (alternative)

# Show password aging
chage -l alice
```

⚠️  **Pitfalls**
- `usermod -G groups` **replaces** group memberships. Always use `-aG` to **append**. This is the #1 mistake made when adding users to `docker` or `sudo`.
- Group membership changes only take effect on **new** login sessions. Re-login or use `newgrp`/`sg`.
- `useradd` without `-m` won't create a home directory — common surprise. Some distros set `CREATE_HOME yes` in `/etc/login.defs`, others don't.
- `adduser` (Debian) is an interactive wrapper. In scripts use `useradd`.

### 5.3 Group Management

```
groupadd [-g GID] [-r] GROUP
groupmod [-n NEWNAME] [-g NEWGID] GROUP
groupdel GROUP
gpasswd  -a USER GROUP        # add member
gpasswd  -d USER GROUP        # remove member
gpasswd  -A USER GROUP        # designate group admin
```

```bash
groupadd -r dockerlite                    # system group (low GID)
gpasswd -a deploy dockerlite
```

### 5.4 Identity & Session Inspection

| Command | Purpose |
|---------|---------|
| `id [USER]` | UID/GID and groups of user (current if omitted) |
| `groups [USER]` | group memberships |
| `who` | who is currently logged in |
| `w` | who + what they're doing + load avg |
| `last -n 20` | recent logins (reads `/var/log/wtmp`) |
| `lastb` | failed login attempts (reads `/var/log/btmp`, root only) |
| `lastlog` | last login per user |
| `finger USER` | user info (often not installed) |

🎯 **DevOps use case**
After an incident, `last -F` (full timestamps) shows exactly when each user logged in. Combine with `auth.log` greps for a forensic timeline.

```bash
$ id alice
uid=1001(alice) gid=1001(alice) groups=1001(alice),27(sudo),998(docker)

$ w
 11:55:32 up 14 days,  3:42,  2 users,  load average: 0.42, 0.51, 0.55
USER     TTY      FROM              LOGIN@   IDLE   JCPU   PCPU WHAT
alice    pts/0    10.0.0.5         09:21    0.00s  0.12s  0.02s w
bob      pts/1    10.0.0.7         11:48    7:01   0.05s  0.03s -bash
```

---

## ✅ Part 1 — Quick Reference Card

### Filesystem & Navigation
```bash
pwd                    # current dir (use -P to resolve symlinks)
ls -lah                # long, all, human sizes
ls -ltr                # by mtime, oldest first
stat FILE              # full metadata
file FILE              # detect type
readlink -f FILE       # canonical path
tree -L 2 -d DIR       # 2-deep dir tree
```

### File operations
```bash
cp -av SRC DST                              # archive copy, verbose
mv -v SRC DST                               # rename / move
rm -rf -- "${DIR:?}"/                       # SAFE recursive delete
mkdir -p a/b/c                              # create parents
install -d -m 0750 -o app -g app /var/lib/myapp   # idempotent mkdir + chown + chmod
tail -F /var/log/app.log                    # follow with reopen
head -n 50 FILE / tail -n 50 FILE
```

### find
```bash
find PATH -type f -mtime +14 -delete
find PATH -type f -size +1G -exec du -h {} +
find PATH -type f -mmin -10
find PATH -type f -name '*.log' -print0 | xargs -0 -P8 gzip
find /  -xdev -perm -4000 -type f 2>/dev/null   # audit SUID
```

### Text processing
```bash
grep -rni 'PATTERN' DIR
grep -A2 -B2 'PATTERN' file
sed -i.bak 's/old/new/g' file
sed -E '/^\s*(#|$)/d' file                  # strip comments + blanks
awk -F: '$3>=1000 {print $1}' /etc/passwd
awk '{count[$1]++} END {for (k in count) print count[k], k}' access.log | sort -rn
cut -d: -f1,7 /etc/passwd
sort -h | uniq -c | sort -rn
tr -d '\r' < win.txt > unix.txt
diff -u old new > my.patch && patch < my.patch
```

### Permissions
```bash
chmod 0644 FILE                             # rw-r--r--
chmod -R u=rwX,g=rX,o= DIR                  # capital X = x only on dirs
chown -R app:app DIR
umask 0027
setfacl -m u:alice:rw FILE
getfacl FILE
find / -xdev -perm -4000 -type f            # SUID audit
```

### Users & Groups
```bash
useradd -r -s /usr/sbin/nologin -M myapp    # service user
useradd -m -s /bin/bash -G sudo alice       # admin user
usermod -aG docker bob                      # APPEND (not overwrite!)
passwd USER
chage -l USER
id USER
last -F -n 20
w
```

---

**END OF PART 1.** Confirm to proceed to **Part 2 — Intermediate Linux for DevOps** (Process Management, Package Management, systemd, Disk & Storage, Networking).
