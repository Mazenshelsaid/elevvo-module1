# Elevvo Cloud & DevOps Track - Module 1: Hardened Bastion & Production Service

Environment: Ubuntu Linux VM on VMware, client: Windows PowerShell.
VM IP: `192.168.222.129`

## 1. SSH Key Generation

Command (on the client):

```powershell
ssh-keygen -t ed25519 -C "devops@elevvo-lab"
```

Output (randomart image, the private key is passphrase-protected):

```
The key's randomart image is:
+--[ED25519 256]--+
|                 |
|   . .           |
|    +            |
| . o .           |
|... =   S        |
|= .+ = o. .      |
|+=o B oo+E +     |
|O=+o B.oB.+      |
|=X+ ooB*..       |
+----[SHA256]-----+
```

> TODO: paste the full terminal output here (the "Generating public/private ed25519 key pair" lines and the fingerprint line), and the output of `ssh-keygen -lf ~/.ssh/id_ed25519.pub`.

Public key copied to the server:

```powershell
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh devops@192.168.222.129 "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

## 2. Least Privilege User

```bash
sudo adduser devops
sudo usermod -aG sudo devops
id devops
```

> TODO: paste the output of `id devops`.

## 3. Daemon Hardening

Snippet of `/etc/ssh/sshd_config` (see `configs/sshd_config.snippet`):

```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
```

Validation and restart:

```bash
sudo sshd -t
sudo systemctl restart ssh
sudo sshd -T | grep -Ei "permitrootlogin|passwordauthentication"
```

> TODO: paste the real output (expected: `permitrootlogin no` and `passwordauthentication no`).

## 4. Firewall (UFW)

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 8080/tcp
sudo ufw enable
sudo ufw status verbose
```

> TODO: paste the output of `sudo ufw status verbose`.

## 5. Service Management (systemd)

Dedicated unprivileged user and web root:

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin webapp
sudo mkdir -p /opt/webapp
echo "<h1>Hello from Elevvo Module 1</h1>" | sudo tee /opt/webapp/index.html
sudo chown -R webapp:webapp /opt/webapp
```

Unit file: [`configs/webapp.service`](configs/webapp.service)

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now webapp.service
systemctl status webapp.service
```

> TODO: paste the output of `systemctl status webapp.service` (must show `active (running)` and `enabled`).

Restart policy test:

```bash
sudo pkill -f "http.server 8080"
sleep 6
systemctl status webapp.service
journalctl -u webapp.service -n 20 --no-pager
```

> TODO: paste the output showing the service restarted itself.

The service is reachable from the host machine at `http://192.168.222.129:8080`.

## 6. Access Verification

Screenshot: `screenshots/access-verification.png`

It shows:
1. Successful login with the key: `ssh devops@192.168.222.129`
2. Password login rejected:
   `ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password devops@192.168.222.129`
   -> `Permission denied (publickey).`
3. Root login rejected: `ssh root@192.168.222.129`

![Access verification](screenshots/access-verification.png)

## 7. The Why

**Why disable root login instead of using a complex root password?**
Direct root logins bypass the audit trail. With a named user plus `sudo`, every privileged command is logged against a person, and the system stays protected even if a password leaks (defense in depth).

**Why Ed25519 instead of RSA-4096?**
Ed25519 uses elliptic curve cryptography, giving smaller and faster keys with no practical attack surface. RSA is mostly kept as a compatibility fallback for legacy systems.
