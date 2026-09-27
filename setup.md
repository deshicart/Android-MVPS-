# Android VPS-Like Server Setup (Termux + Debian + OpenSSH + Tailscale)

A complete, copy-paste-friendly guide for creating a VPS-like Linux environment on an Android phone **without root**.

> **Important:** This is not a real hardware VPS partition. Debian runs inside Termux using `proot-distro`, so Android and Debian still share the phone's CPU, RAM, battery and storage.
>
> Recommended architecture:
>
> **Android → Termux → Debian (proot) → OpenSSH/SFTP**
>
> Remote access:
>
> **Remote Android → Tailscale → Host Android → Debian SSH**

---

## 0. What this guide gives you

After completing this guide:

- Debian Linux runs inside Android without root.
- A normal Linux user named `server` is created.
- SSH runs on port `2222`.
- Remote SSH access works through Tailscale.
- SFTP works with Material Files or another SFTP client.
- Files can be created/edited remotely.
- The host phone can still be used normally.
- The server can work while the screen is off, provided Android does not kill Termux/Tailscale.
- Internet OFF = remote access unavailable.
- Internet ON again = Tailscale can reconnect automatically.
- No public SSH port needs to be opened on the router.

### Recommended first test

On a small test phone:

- RAM: about 2 GB
- Storage: about 32 GB
- Do NOT try to hard-limit RAM/storage during the first test.
- First verify that SSH, Tailscale, SFTP, screen-off and reconnect all work.

After the setup is proven, resource limits can be added separately.

---

# 1. Requirements

## Host Android phone

Install:

1. Termux
2. Tailscale

Optional later:

3. Termux:Boot

Use a current Termux build from a trusted source such as F-Droid or the official Termux GitHub project.

## Remote Android phone

Install:

1. Tailscale
2. Termux (for SSH testing)
3. Material Files (optional, for SFTP)

---

# 2. Important Android settings

Before testing the server:

### Disable battery optimization

Android Settings → Apps → Termux → Battery → choose:

- Unrestricted
- Don't optimize
- Allow background activity

The exact wording depends on the phone manufacturer.

Do the same for Tailscale if your phone provides the option.

### Keep Android powered ON

The phone may have its screen OFF, but it must not be fully powered OFF.

If the phone is shut down:

- SSH will stop.
- Tailscale will stop.
- The server will not be reachable.

---

# 3. Install Termux packages

Open Termux on the HOST phone.

Run:

```bash
pkg update -y
```

Then:

```bash
pkg upgrade -y
```

Install all required Termux packages:

```bash
pkg install -y proot-distro openssh nano curl wget git tmux
```

Verify:

```bash
command -v proot-distro
command -v ssh
command -v nano
command -v curl
command -v wget
command -v git
command -v tmux
```

All commands should print a path.

---

# 4. Give Termux storage permission

Run:

```bash
termux-setup-storage
```

Android will show a permission dialog.

Allow storage access.

Verify:

```bash
ls -la ~/storage
```

You should see directories such as:

```text
shared
downloads
pictures
movies
music
dcim
```

---

# 5. Install Debian

Check whether Debian is already installed:

```bash
proot-distro list
```

If Debian is not installed, run:

```bash
proot-distro install debian
```

Verify again:

```bash
proot-distro list
```

You should see Debian as installed.

---

# 6. Enter Debian as root

From the normal Termux prompt, for example:

```text
~ $
```

run:

```bash
proot-distro login debian
```

You should now see something similar to:

```text
root@localhost:~#
```

## IMPORTANT

Once you see:

```text
root@localhost:~#
```

you are already inside Debian.

**Do NOT run `proot-distro login debian` again.**

To return to Termux:

```bash
exit
```

---

# 7. Update Debian

Inside Debian, run:

```bash
apt update
```

Then:

```bash
apt upgrade -y
```

Do not continue until these commands finish successfully.

---

# 8. Install ALL Debian packages needed by this guide

Run:

```bash
apt install -y openssh-server sudo nano curl wget git iproute2 ca-certificates
```

Verify the important commands:

```bash
command -v ssh
command -v sshd
command -v sudo
command -v nano
command -v curl
command -v wget
command -v git
command -v ss
```

Each command should print a path.

---

# 9. Create the server user

Check whether the user already exists:

```bash
id server
```

If it says that the user does not exist, create it:

```bash
adduser server
```

During setup:

- Enter a strong password.
- Full Name can be left blank.
- Other information can be left blank.
- Confirm with `Y` when asked.

If the user already exists, do NOT run `adduser server` again.

Set/reset the password if necessary:

```bash
passwd server
```

Add the user to sudo:

```bash
usermod -aG sudo server
```

Verify:

```bash
id server
```

Verify the home directory:

```bash
ls -la /home/server
```

---

# 10. Configure SSH

Create a backup first:

```bash
cp /etc/ssh/sshd_config /etc/ssh/sshd_config.backup
```

Open the SSH configuration:

```bash
nano /etc/ssh/sshd_config
```

Make sure these settings exist:

```text
Port 2222
PasswordAuthentication yes
PubkeyAuthentication yes
```

If you see an existing line beginning with `#Port`, replace it with:

```text
Port 2222
```

If you see:

```text
#PasswordAuthentication yes
```

change it to:

```text
PasswordAuthentication yes
```

If `PubkeyAuthentication` is missing, add:

```text
PubkeyAuthentication yes
```

### Save nano

Press:

```text
CTRL + O
```

Press Enter.

Then:

```text
CTRL + X
```

---

# 11. Validate SSH configuration BEFORE starting SSH

This step prevents many errors.

Run:

```bash
/usr/sbin/sshd -t
```

### If there is no output

The configuration is valid.

Continue.

### If you get an error

STOP.

Do not continue to Tailscale.

Fix the error first.

---

# 12. Generate SSH host keys

Run:

```bash
ssh-keygen -A
```

Verify:

```bash
ls -la /etc/ssh/ssh_host_*
```

You should see SSH host key files.

---

# 13. Prepare SSH runtime directory

Run:

```bash
mkdir -p /run/sshd
chmod 755 /run/sshd
```

---

# 14. Start SSH

Start the SSH daemon:

```bash
/usr/sbin/sshd
```

Check whether it is running:

```bash
pgrep -a sshd
```

Then check port 2222:

```bash
ss -tln | grep ':2222'
```

You should see something similar to:

```text
LISTEN 0 128 0.0.0.0:2222 0.0.0.0:*
```

or:

```text
LISTEN 0 128 [::]:2222 [::]:*
```

---

# 15. If port 2222 is NOT listening

Run the following in this exact order:

```bash
/usr/sbin/sshd -t
```

Then:

```bash
mkdir -p /run/sshd
chmod 755 /run/sshd
```

Then:

```bash
/usr/sbin/sshd
```

Then:

```bash
ss -tln | grep ':2222'
```

If it still does not listen, run:

```bash
/usr/sbin/sshd -D -e
```

This runs SSH in the foreground and prints the real error.

Do not close that terminal before recording the error.

---

# 16. Test SSH locally

Still inside Debian, run:

```bash
ssh -p 2222 server@127.0.0.1
```

The first time you may see:

```text
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

Type:

```text
yes
```

Enter the `server` user's password.

You should enter the server user's shell.

Verify:

```bash
whoami
```

Expected:

```text
server
```

Verify:

```bash
pwd
```

Expected:

```text
/home/server
```

Exit the SSH session:

```bash
exit
```

---

# 17. Test sudo

From the `server` user:

```bash
sudo -v
```

Enter the server user's password.

Then:

```bash
sudo whoami
```

Expected:

```text
root
```

Do not use root SSH for normal administration. Use the `server` user and `sudo`.

---

# 18. Install Tailscale on the HOST Android

Tailscale should run at the Android/Termux level, not inside the Debian proot environment.

First leave Debian:

```bash
exit
```

You should be back at:

```text
~ $
```

Install Tailscale using the Android Tailscale app.

Open the Tailscale app.

Sign in to the same Tailscale account that you will use on the remote Android phone.

Enable the VPN connection when Android asks.

---

# 19. Install Tailscale on the REMOTE Android

On the second Android phone:

1. Install Tailscale.
2. Sign in to the same Tailscale account.
3. Enable the VPN.
4. Make sure both phones appear in the Tailscale device list.

The host phone should receive a Tailscale IP, normally in the:

```text
100.x.x.x
```

range.

Record the host phone's Tailscale IP.

Example:

```text
100.100.100.100
```

Your actual IP will be different.

---

# 20. Test Tailscale connectivity

On the REMOTE phone, open Termux.

Install SSH client:

```bash
pkg update -y
pkg install -y openssh
```

Test the host's Tailscale IP:

```bash
ping -c 4 YOUR_HOST_TAILSCALE_IP
```

Example:

```bash
ping -c 4 100.100.100.100
```

If ping is blocked or does not respond, do not automatically assume SSH is broken. Continue with the SSH test.

---

# 21. Remote SSH test

On the REMOTE phone:

```bash
ssh -p 2222 server@YOUR_HOST_TAILSCALE_IP
```

Example:

```bash
ssh -p 2222 server@100.100.100.100
```

Enter the `server` password.

Verify:

```bash
whoami
```

Expected:

```text
server
```

Then:

```bash
hostname
```

Then:

```bash
pwd
```

Expected:

```text
/home/server
```

---

# 22. Create a test folder

On the remote SSH session:

```bash
mkdir -p ~/test-server
```

Create a test file:

```bash
echo "Android VPS test successful" > ~/test-server/test.txt
```

Read it:

```bash
cat ~/test-server/test.txt
```

Expected:

```text
Android VPS test successful
```

---

# 23. SFTP with Material Files

On the REMOTE Android:

Open Material Files.

Choose:

```text
Add storage
```

Then choose:

```text
SFTP
```

Use:

```text
Host: YOUR_HOST_TAILSCALE_IP
Port: 2222
Username: server
Password: YOUR_SERVER_PASSWORD
```

Example:

```text
Host: 100.100.100.100
Port: 2222
Username: server
```

Connect.

You should be able to access:

```text
/home/server
```

You can now:

- Create folders
- Upload files
- Download files
- Rename files
- Delete files
- Edit files

---

# 24. SSH key authentication (recommended)

Password authentication is useful for the first test.

After the setup works, SSH keys are recommended.

## Generate a key on the REMOTE phone

In remote Termux:

```bash
pkg install -y openssh
```

Check whether a key already exists:

```bash
ls -la ~/.ssh
```

If you do not already have an appropriate key, create one:

```bash
ssh-keygen -t ed25519
```

Press Enter to accept the default location.

You may set a passphrase.

---

# 25. Copy the public key to the server

Display the public key:

```bash
cat ~/.ssh/id_ed25519.pub
```

Copy the entire single line.

On the HOST Debian server, log in as `server` and run:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys
```

Paste the public key as one line.

Save:

```text
CTRL + O
Enter
CTRL + X
```

Then:

```bash
chmod 600 ~/.ssh/authorized_keys
```

Verify:

```bash
ls -la ~/.ssh
```

---

# 26. Test SSH key login

From the REMOTE phone:

```bash
ssh -p 2222 server@YOUR_HOST_TAILSCALE_IP
```

If the key is correctly configured, it should authenticate using the key.

---

# 27. Disable password authentication after key testing

Only do this AFTER confirming that key login works.

Inside Debian:

```bash
nano /etc/ssh/sshd_config
```

Change:

```text
PasswordAuthentication yes
```

to:

```text
PasswordAuthentication no
```

Keep:

```text
PubkeyAuthentication yes
```

Save the file.

Validate:

```bash
/usr/sbin/sshd -t
```

---

# 28. Restart SSH after configuration changes

Because this is a proot environment, do not depend on systemd.

Find SSH processes:

```bash
pgrep -a sshd
```

Stop the running SSH daemon:

```bash
pkill sshd
```

Prepare the runtime directory:

```bash
mkdir -p /run/sshd
chmod 755 /run/sshd
```

Start SSH again:

```bash
/usr/sbin/sshd
```

Verify:

```bash
ss -tln | grep ':2222'
```

Then test from the REMOTE phone:

```bash
ssh -p 2222 server@YOUR_HOST_TAILSCALE_IP
```

---

# 29. Screen-off test

This is an important test.

On the HOST phone:

1. Make sure Termux is running.
2. Make sure Debian SSH is running.
3. Make sure Tailscale is connected.
4. Enable the Android screen lock.
5. Turn the screen off.

Wait about 1–2 minutes.

From the REMOTE phone:

```bash
ssh -p 2222 server@YOUR_HOST_TAILSCALE_IP
```

If it connects, screen-off operation works.

---

# 30. Prevent Android from killing Termux

On the HOST Android:

Android Settings → Apps → Termux → Battery.

Choose:

```text
Unrestricted
```

or the equivalent option.

Also allow background activity if your phone provides that setting.

For Tailscale, use the equivalent unrestricted/background setting when available.

---

# 31. Optional: Termux wake lock

On the HOST phone, outside Debian, install the Termux API package only if you specifically need the wake-lock feature.

If `termux-wake-lock` is available in your Termux installation:

```bash
termux-wake-lock
```

This can help keep the process alive, but it may increase battery consumption.

To release it:

```bash
termux-wake-unlock
```

Do not assume that a wake lock can prevent every Android manufacturer from killing background processes.

---

# 32. Internet OFF/ON test

This setup intentionally depends on Internet connectivity.

### Test 1 — Internet OFF

On the HOST phone:

- Turn Wi-Fi/mobile data OFF.

Remote SSH should become unavailable.

### Test 2 — Internet ON

Turn Internet back ON.

Wait for Tailscale to reconnect.

Then from the REMOTE phone:

```bash
ssh -p 2222 server@YOUR_HOST_TAILSCALE_IP
```

It should become reachable again.

If not, open the Tailscale app on the host and verify its connection.

---

# 33. Starting Debian after leaving Termux

Whenever you open Termux again:

```bash
proot-distro login debian
```

Then start SSH:

```bash
mkdir -p /run/sshd
chmod 755 /run/sshd
/usr/sbin/sshd
```

Check:

```bash
ss -tln | grep ':2222'
```

---

# 34. Avoid starting duplicate SSH daemons

Before starting SSH, check:

```bash
pgrep -a sshd
```

If SSH is already running, you usually do not need to start another copy.

Check the port:

```bash
ss -tln | grep ':2222'
```

If it already shows LISTEN, SSH is already active.

---

# 35. Useful server commands

## Enter Debian

From Termux:

```bash
proot-distro login debian
```

## Check RAM

Inside Debian:

```bash
free -h
```

## Check disk usage

```bash
df -h
```

## Check CPU/processes

```bash
ps aux
```

## Check SSH

```bash
pgrep -a sshd
```

## Check port

```bash
ss -tln | grep ':2222'
```

## Check current user

```bash
whoami
```

## Check IP information

```bash
hostname -I
```

---

# 36. Installing development software later

After the base server is stable, you can install development software inside Debian.

For example:

```bash
apt update
apt install -y nodejs npm
```

Check:

```bash
node --version
npm --version
```

Git is already installed:

```bash
git --version
```

Do NOT install large software stacks until the base SSH/Tailscale/SFTP system has been tested successfully.

---

# 37. PHP / database / web server

You do not need XAMPP for this Linux server.

A normal Linux stack can be:

```text
Nginx or Apache
PHP
MariaDB
Git
Node.js
```

Install these only when needed.

Example:

```bash
apt update
apt install -y nginx php-fpm mariadb-server
```

Do not run this until the base server is confirmed stable.

---

# 38. Running long-lived terminal programs

Install tmux:

```bash
apt update
apt install -y tmux
```

Start:

```bash
tmux
```

Detach:

```text
CTRL + B
then D
```

Reattach:

```bash
tmux attach
```

List sessions:

```bash
tmux ls
```

This is useful for long-running commands.

---

# 39. Important limitation: RAM is shared

If the Android phone has 2 GB RAM, Debian does NOT automatically receive a physically separate 0.5 GB RAM partition.

For example:

```text
Android
   |
   +-- Termux
   |
   +-- Debian
   |
   +-- Other Android apps
```

They share the phone's physical resources.

Similarly, a "4 GB VPS" on an 8 GB phone would be a software/resource limit, not a real hardware partition.

---

# 40. Important limitation: storage is shared

If the phone has 32 GB storage, Debian does not automatically own a separate physical 1 GB disk partition.

The Linux environment uses Android storage.

For the first test, do not attempt complicated storage quotas.

First prove:

1. SSH
2. Tailscale
3. SFTP
4. Screen-off access
5. Internet reconnect
6. Normal Android usage

Then add resource isolation.

---

# 41. Future resource-limited test

After the basic setup is stable, resource limits can be added separately.

For example, the desired test target may be approximately:

```text
RAM:     512 MB
Storage: 1 GB
```

Later, on the main phone:

```text
RAM:     approximately 4 GB
Storage: approximately 50 GB
```

These should be implemented as software limits/filesystem limits where technically appropriate.

Do NOT claim that this creates a physical VPS partition.

---

# 42. Security rules

For this setup:

### Do

- Use Tailscale.
- Use a normal `server` user.
- Use SSH keys.
- Use `sudo` for administrative commands.
- Keep SSH on port 2222 if desired.
- Disable password authentication after key login works.
- Keep Android and Termux updated.

### Avoid

- Publicly exposing SSH port 22.
- Using root SSH for normal work.
- Sharing the SSH private key.
- Sharing the server password.
- Running unknown scripts with `sudo`.
- Installing untrusted packages/scripts.

---

# 43. Quick diagnostic checklist

If SSH says:

```text
Connection refused
```

Run INSIDE Debian:

```bash
pgrep -a sshd
```

Then:

```bash
ss -tln | grep ':2222'
```

If nothing appears:

```bash
/usr/sbin/sshd -t
```

Then:

```bash
mkdir -p /run/sshd
chmod 755 /run/sshd
```

Then:

```bash
/usr/sbin/sshd
```

Then:

```bash
ss -tln | grep ':2222'
```

---

## If `ss` command is missing

Install:

```bash
apt update
apt install -y iproute2
```

Then:

```bash
ss -tln | grep ':2222'
```

---

## If SSH says host keys are missing

Run:

```bash
ssh-keygen -A
```

Then:

```bash
mkdir -p /run/sshd
/usr/sbin/sshd
```

---

## If SSH says `/run/sshd` does not exist

Run:

```bash
mkdir -p /run/sshd
chmod 755 /run/sshd
```

Then:

```bash
/usr/sbin/sshd
```

---

## If SSH configuration is invalid

Run:

```bash
/usr/sbin/sshd -t
```

Fix the reported line.

Then test again:

```bash
/usr/sbin/sshd -t
```

Only continue when there is no output.

---

## If port 2222 is already in use

Check:

```bash
ss -tln | grep ':2222'
```

Find the process:

```bash
ss -tlnp | grep ':2222'
```

Do not randomly change ports. First determine which process is using it.

---

# 44. Final verification checklist

The setup is considered successful only when ALL of these work:

### Host phone

```bash
proot-distro login debian
```

Then:

```bash
ss -tln | grep ':2222'
```

Shows LISTEN.

### Local SSH

```bash
ssh -p 2222 server@127.0.0.1
```

Works.

### Remote SSH

```bash
ssh -p 2222 server@YOUR_HOST_TAILSCALE_IP
```

Works.

### SFTP

Material Files can connect using:

```text
Host: YOUR_HOST_TAILSCALE_IP
Port: 2222
User: server
```

### File creation

```bash
mkdir -p ~/server-test
echo "success" > ~/server-test/test.txt
```

Works.

### Screen off

Remote SSH still works while host screen is OFF.

### Internet reconnect

SSH becomes available again after host Internet is restored and Tailscale reconnects.

### Normal phone use

Facebook, WhatsApp, Gmail, YouTube, camera, Excel and other normal apps continue working.

---

# 45. Daily quick-start

When you want to use the server:

### Termux

```bash
proot-distro login debian
```

Then:

```bash
mkdir -p /run/sshd
chmod 755 /run/sshd
```

Check whether SSH is already running:

```bash
ss -tln | grep ':2222'
```

If nothing appears:

```bash
/usr/sbin/sshd
```

Check again:

```bash
ss -tln | grep ':2222'
```

Now connect remotely using:

```bash
ssh -p 2222 server@YOUR_HOST_TAILSCALE_IP
```

---

# 46. Architecture summary

```text
                    INTERNET
                       |
                +------+------+
                |   Tailscale |
                +------+------+
                       |
              HOST ANDROID PHONE
                       |
                 +-----+-----+
                 |  Termux  |
                 +-----+-----+
                       |
                 proot-distro
                       |
                  +----+----+
                  | Debian  |
                  +----+----+
                       |
                 OpenSSH :2222
                       |
                 user: server
                       |
              /home/server
```

Remote access:

```text
REMOTE ANDROID
      |
   Tailscale
      |
      v
HOST ANDROID
      |
  Termux/Debian
      |
OpenSSH :2222
      |
   server
```

---

# 47. Most important rule

Do not skip installation/verification steps.

The correct order is:

```text
Termux packages
        ↓
Storage permission
        ↓
Debian
        ↓
Debian update
        ↓
Debian packages
        ↓
server user
        ↓
SSH configuration
        ↓
SSH validation
        ↓
SSH host keys
        ↓
/run/sshd
        ↓
sshd
        ↓
ss -tln verification
        ↓
Local SSH test
        ↓
Tailscale
        ↓
Remote SSH test
        ↓
SFTP
        ↓
SSH keys
        ↓
Disable password authentication
        ↓
Screen-off test
        ↓
Internet OFF/ON test
        ↓
Optional development software
```

Do not move to the next major section if the verification command from the current section fails.
