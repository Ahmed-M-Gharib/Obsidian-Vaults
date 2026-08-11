---
type: study-note
subject: Linux-04-Linux-System-Admin
category: devops
status: active
---

# 17 - Virtual Consoles

> [!quote] Deck's content
> Accessed with Ctrl-Alt-F_key. Consoles 1-6 accept logins. X server starts on the console 7.

**[EXTRA]** Genuinely relevant caveat the deck omits: on modern systemd-based RHEL/Fedora systems, this virtual-console-to-key mapping is managed by `systemd-logind`, and the exact console assigned to the graphical session can differ from the classic "console 7" convention the deck describes (which reflects older, pre-systemd/pre-Wayland conventions). The underlying concept - multiple independent text login sessions accessible via `Ctrl-Alt-F<n>`, separate from any graphical session - still holds and remains genuinely useful for recovering a system whose graphical desktop or main SSH session has become unresponsive.

```bash
who       # each virtual console session shows up as tty1, tty2, etc in "who" output
```

### Self-Check Q and A

1. **Q: A server's graphical desktop environment freezes completely. Why is knowing about virtual consoles operationally useful here, beyond just being a historical curiosity?**
   A: **[EXTRA]** Switching to a different virtual console (`Ctrl-Alt-F2` for example) provides a completely independent text-mode login session, unaffected by whatever caused the graphical session to hang - allowing you to log in, diagnose, and potentially kill the frozen graphical session or restart the display manager without needing to power-cycle the machine.

---

# 18 - System Shutdown

> [!quote] Deck's content
> It only requires reboot or shutdown when you need to add or remove hardware, upgrade to a new version of Ubuntu, or upgrade your kernel. `shutdown -k now` doesn't really shut down, only sends warning messages and disables logins. `shutdown -h time` halts after shutdown. `poweroff`. `init 0`.

```bash
shutdown -k now              # warn only, don't actually shut down
shutdown -h now                # halt now
shutdown -r now                 # reboot now
shutdown -h +10 "maintenance in 10 minutes"   # scheduled, with a broadcast message
poweroff
init 0
```

> [!important] `init 0` is a legacy reference on modern RHEL
> **[EXTRA]** The deck's `init 0` reflects the older SysV init system's runlevel model, where runlevel 0 meant "halt." Modern RHEL/Fedora (and most current distros) use systemd as PID 1, where the equivalent operations are `systemctl poweroff` and `systemctl reboot` - `init 0` and `shutdown` commands still work on systemd systems (systemd provides backward-compatible shims for them), but the underlying mechanism and the actual unit-dependency-based shutdown sequence is entirely systemd's, not the old SysV init's.

```bash
systemctl poweroff       # modern systemd-native equivalent
systemctl reboot
systemctl status          # see current system state / any failed units before deciding to reboot
```

### Self-Check Q and A

1. **Q: `init 0` still works to shut down a modern RHEL 9 system. Does this mean the system is actually using SysV init under the hood?**
   A: **[EXTRA]** No - modern RHEL uses systemd as PID 1. `init 0` (and the classic `shutdown` command syntax) still works because systemd provides backward-compatible command shims that translate these legacy invocations into the equivalent systemd operations (`systemctl poweroff`), preserving compatibility with old scripts and muscle memory without actually running SysV init.

---

# Beyond the Slides - Essential Linux Knowledge for a DevOps Engineer

**[EXTRA]** This entire section is additional to both decks. RHSA1 is an entry-level fundamentals course and deliberately stops short of package management, text processing, I/O redirection, service management, and remote access - all things a working DevOps engineer uses daily from week one on the job.

## Package Management

Since this note follows RHEL conventions (per the distros section), the RHEL-family tools:

```bash
dnf install httpd              # install a package (modern RHEL/Fedora - replaces yum)
dnf remove httpd                 # uninstall
dnf update                        # update all packages
dnf search keyword                  # search available packages
dnf list installed                   # list what's installed
dnf info httpd                        # package details
rpm -qa                                 # list all installed packages (lower-level than dnf)
rpm -qi httpd                            # query info about a specific installed package
rpm -ql httpd                             # list files owned by a package
```

For contrast, the Debian-family equivalents referenced earlier in this note:

```bash
apt install httpd
apt remove httpd
apt update && apt upgrade
apt search keyword
dpkg -l
```

## I/O Redirection and Pipes

Not covered anywhere in either deck, despite being fundamental to virtually every real command-line workflow:

```bash
command > file           # redirect stdout to a file, OVERWRITING it
command >> file            # redirect stdout to a file, APPENDING
command 2> file              # redirect stderr (errors) specifically
command > file 2>&1            # redirect BOTH stdout and stderr to the same file
command < file                  # feed a file as stdin

command1 | command2              # PIPE - command1's stdout becomes command2's stdin
```

```mermaid
graph LR
    C1["ps aux"] -->|"stdout piped as stdin"| C2["grep nginx"] -->|"stdout piped as stdin"| C3["wc -l"]
```

```bash
ps aux | grep nginx | wc -l      # count running nginx processes - classic pipe chain
cat access.log | grep "500" | wc -l   # count error responses in a log file
```

## Text Processing - grep, sed, awk

The single largest practical gap in the deck for a DevOps engineer, who spends enormous amounts of time parsing logs and config files:

```bash
grep "ERROR" logfile              # find lines matching a pattern
grep -i "error" logfile             # case-insensitive
grep -v "DEBUG" logfile               # INVERT - show lines NOT matching
grep -r "TODO" /path/to/code/           # recursive search through a directory
grep -c "ERROR" logfile                   # count matching lines
grep -n "ERROR" logfile                    # show line numbers

sed 's/old/new/' file                        # substitute first occurrence per line
sed 's/old/new/g' file                         # substitute ALL occurrences per line
sed -i 's/old/new/g' file                        # edit the file IN PLACE

awk '{print $1}' file                              # print the first whitespace-separated field of each line
awk -F: '{print $1}' /etc/passwd                     # use : as the field separator - print just usernames
```

## Environment Variables and PATH

```bash
echo $PATH                       # colon-separated list of directories the shell searches for commands
export MY_VAR="value"              # set an environment variable, visible to child processes
env                                  # list all current environment variables
which command                         # show which PATH directory a command actually resolves to
```

> [!important] Why PATH order matters
> **[EXTRA]** `PATH` is searched left to right, and the FIRST matching executable found wins. A very common real-world issue: a user or CI environment has an unexpected or outdated version of a tool (Python, Node, a custom script) shadowing the intended one earlier in PATH - `which toolname` immediately reveals which actual binary is being invoked.

## Service Management with systemd

Not covered at all in either deck despite being the actual mechanism behind the shutdown discussion:

```bash
systemctl status httpd           # check a service's current state
systemctl start httpd              # start it now
systemctl stop httpd                # stop it now
systemctl restart httpd              # restart
systemctl enable httpd                # start automatically on boot
systemctl disable httpd                 # don't start automatically on boot
systemctl enable --now httpd             # enable AND start in one command
journalctl -u httpd                        # view that service's logs
journalctl -u httpd -f                       # follow that service's logs live
```

## SSH and Remote Access

```bash
ssh user@hostname                          # connect to a remote machine
ssh -i keyfile.pem user@hostname             # connect using a specific private key
scp file.txt user@hostname:/remote/path/       # copy a file to a remote machine
ssh-keygen -t ed25519                            # generate a new SSH keypair
```

## Process Management

```bash
ps aux                # list all running processes
top / htop              # live, interactively updating process view
kill <pid>                 # send SIGTERM (graceful stop request)
kill -9 <pid>                # send SIGKILL (unconditional termination, per the interrupting-execution section above)
```

## Disk and Filesystem Basics

```bash
df -h              # disk space usage per mounted filesystem, human-readable
du -sh /path/         # total size of a directory, human-readable
mount / umount           # attach/detach a filesystem to the directory tree
lsblk                      # list block devices (disks and their partitions)
```

## Scheduled Tasks - cron

```bash
crontab -e                    # edit the current user's scheduled jobs
crontab -l                      # list current scheduled jobs
# 0 2 * * * /path/to/backup.sh   <- example: run daily at 2:00 AM
```

---
