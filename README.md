# Kingg-Server: Multi-User Hosting with LXD Containers

One Oracle Cloud VM is split into three isolated Ubuntu 24.04 servers (`server1`, `server2`, `server3`). Each user logs in to the host over SSH with a password and is automatically placed in their own server as **root**. Each server is published on its own domain (`server1.kingg.in`, `server2.kingg.in`, `server3.kingg.in`) through a reverse proxy on the host.

## Status

| Component | Status |
|---|---|
| LXD installed, 3 containers running | Done |
| Fixed internal IPs | Done |
| Per-user SSH login, forced into own container as root | Done |
| Apache inside `server1`, published via `server1.kingg.in` | Done and verified |
| Same proxy pattern for `server2` and `server3` | Ready to apply (same steps) |
| HTTPS (Let's Encrypt) | Recommended next step |

## Architecture

```
                                   ┌──────────────────────────────────────────┐
                                   │ Host: Kingg-Server-1 (Oracle Cloud VM)   │
user1 ─┐                           │                                          │
user2 ─┼─ SSH (password) ─────────►│ sshd ─ forced jump ─► server1 10.125.169.11 (user1, root)
user3 ─┘                           │                  ├──► server2 10.125.169.12 (user2, root)
                                   │                  └──► server3 10.125.169.13 (user3, root)
Browser ── server1.kingg.in ──────►│ Apache reverse proxy ─► container web apps
                                   └──────────────────────────────────────────┘
                                         lxdbr0 private bridge (NAT)
```

| Item | Value |
|---|---|
| Host | `Kingg-Server-1`, Oracle Cloud Always Free, ARM, Ubuntu 24.04 |
| Container technology | LXD 5.21 (snap), `dir` storage pool, bridge `lxdbr0` |
| Private subnet | `10.125.169.0/24` |
| Resources | No CPU, RAM or disk limits: containers use host resources on demand |
| Host login | Password, forced command into the container |
| Container login | Automatic, no second password (internal link set up once by the admin) |

| Host user | Container | Container IP | Domain |
|---|---|---|---|
| user1 | server1 | 10.125.169.11 | server1.kingg.in |
| user2 | server2 | 10.125.169.12 | server2.kingg.in |
| user3 | server3 | 10.125.169.13 | server3.kingg.in |

## How users connect

```bash
ssh user1@<HOST_PUBLIC_IP>
```

Users only need their username and password. They do not create, copy or manage any SSH keys. After entering the password, the user lands at `root@server1:~#`. Closing that session closes the connection, and the user never gets a shell on the host. Users can install any software inside their own container (web servers, Node, Docker, databases) without affecting other users.

## Setup guide (run as `ubuntu` on the host)

### 1. Install and initialise LXD

```bash
sudo snap install lxd
sudo lxd init
```

Answers used: no clustering, new storage pool `default` with backend **`dir`**, no MAAS, new bridge `lxdbr0`, IPv4 `auto`, IPv6 `none`, do not expose over the network, auto-update images `yes`.

### 2. Create the containers

```bash
lxc launch ubuntu:24.04 server1
lxc launch ubuntu:24.04 server2
lxc launch ubuntu:24.04 server3
```

### 3. Allow the bridge through the Oracle firewall

Oracle's default iptables rules block container DHCP and traffic. Without this, `lxc list` shows empty IPv4 addresses.

```bash
sudo iptables -I INPUT -i lxdbr0 -j ACCEPT
sudo iptables -I FORWARD -i lxdbr0 -j ACCEPT
sudo iptables -I FORWARD -o lxdbr0 -j ACCEPT
sudo netfilter-persistent save
lxc restart server1 server2 server3
```

Test: `lxc exec server1 -- ping -c2 google.com`

### 4. Fixed IPs

```bash
lxc stop server1 server2 server3
lxc config device override server1 eth0 ipv4.address=10.125.169.11
lxc config device override server2 eth0 ipv4.address=10.125.169.12
lxc config device override server3 eth0 ipv4.address=10.125.169.13
lxc start server1 server2 server3
```

If the command says "The device already exists", use `lxc config device set serverN eth0 ipv4.address=...` instead.

### 5. Host users

```bash
sudo adduser user1
sudo adduser user2
sudo adduser user3
```

Do **not** add them to the `lxd` group, because that is equivalent to root on the host.

### 6. Internal host-to-container link (admin only, one-time)

This step is for the server admin only. Users never see it and never type a key or a second password. It lets the host open the root session inside the matching container automatically when a user logs in, which is what makes the jump passwordless. Without it, the user would be asked for an extra password inside the container. Each host user has their own internal key, and it is authorised only for their own container.

```bash
for i in 1 2 3; do
  lxc exec server$i -- bash -c "mkdir -p /root/.ssh && chmod 700 /root/.ssh && touch /root/.ssh/authorized_keys && chmod 600 /root/.ssh/authorized_keys && sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin prohibit-password/' /etc/ssh/sshd_config && systemctl restart ssh"
  sudo -u user$i mkdir -p /home/user$i/.ssh
  sudo -u user$i ssh-keygen -t ed25519 -N "" -f /home/user$i/.ssh/id_ed25519 <<< y
  sudo cat /home/user$i/.ssh/id_ed25519.pub | lxc exec server$i -- tee -a /root/.ssh/authorized_keys
done
```

Trust each container's host key once and verify the hop:

```bash
sudo -u user1 ssh -o StrictHostKeyChecking=accept-new root@10.125.169.11 hostname
sudo -u user2 ssh -o StrictHostKeyChecking=accept-new root@10.125.169.12 hostname
sudo -u user3 ssh -o StrictHostKeyChecking=accept-new root@10.125.169.13 hostname
```

### 7. Host sshd: password login and forced jump

File `/etc/ssh/sshd_config.d/00-containers.conf`:

```
Match User user1
    PasswordAuthentication yes
    ForceCommand ssh -t -o StrictHostKeyChecking=accept-new root@10.125.169.11
    AllowTcpForwarding no
    X11Forwarding no
Match User user2
    PasswordAuthentication yes
    ForceCommand ssh -t -o StrictHostKeyChecking=accept-new root@10.125.169.12
    AllowTcpForwarding no
    X11Forwarding no
Match User user3
    PasswordAuthentication yes
    ForceCommand ssh -t -o StrictHostKeyChecking=accept-new root@10.125.169.13
    AllowTcpForwarding no
    X11Forwarding no
```

```bash
sudo sshd -t && sudo systemctl restart ssh
```

The `00-` prefix makes the file load before Oracle's `50-cloud-init.conf`, so these settings take priority. The `ubuntu` admin account is not in any Match block and keeps normal key-based access.

### 8. Protect against password guessing

```bash
sudo apt install -y fail2ban
sudo systemctl enable --now fail2ban
```

## Publishing a container on its domain

The host's Apache forwards each domain to the matching container, so only the host needs ports 80 and 443 open.

### 1. DNS

At the DNS provider for `kingg.in`, add A records pointing to the host's public IP:

```
server1   A   <HOST_PUBLIC_IP>
server2   A   <HOST_PUBLIC_IP>
server3   A   <HOST_PUBLIC_IP>
```

A single wildcard record `*  A  <HOST_PUBLIC_IP>` also works.

### 2. Open ports 80 and 443

Allow TCP 80 and 443 in the Oracle Cloud security list (ingress rules) and in the host's iptables (`/etc/iptables/rules.v4`).

### 3. Install a web server inside the container

Example for `server1` (the user, or an admin using `lxc exec server1 -- bash`):

```bash
apt update && apt install -y apache2
systemctl enable --now apache2
```

Check inside the container:

```bash
systemctl status apache2 --no-pager
curl -I http://localhost
```

Check from the host:

```bash
curl -I http://10.125.169.11
```

### 4. Create the proxy vhosts on the host

```bash
sudo a2enmod proxy proxy_http proxy_wstunnel headers rewrite ssl

for i in 1 2 3; do
sudo tee /etc/apache2/sites-available/server$i.kingg.in.conf > /dev/null <<EOF
<VirtualHost *:80>
    ServerName server$i.kingg.in
    ProxyPreserveHost On
    ProxyRequests Off
    RequestHeader set X-Forwarded-Proto "http"
    RewriteEngine On
    RewriteCond %{HTTP:Upgrade} =websocket [NC]
    RewriteRule /(.*) ws://10.125.169.1$i:80/\$1 [P,L]
    ProxyPass / http://10.125.169.1$i:80/
    ProxyPassReverse / http://10.125.169.1$i:80/
</VirtualHost>
EOF
done

sudo a2ensite server1.kingg.in server2.kingg.in server3.kingg.in
sudo apache2ctl configtest && sudo systemctl reload apache2
```

If an app listens on another port (for example Node on 3000), change `:80` in the matching vhost to that port.

### 5. Test the proxy

From the host, before or without DNS:

```bash
curl -I -H "Host: server1.kingg.in" http://localhost
```

A `200 OK` means the proxy reaches the container. Then open `http://server1.kingg.in` in a browser.

To test from a Windows PC before DNS is ready, add this line to `C:\Windows\System32\drivers\etc\hosts` (edit as administrator) and remove it later:

```
<HOST_PUBLIC_IP>  server1.kingg.in
```

### 6. HTTPS (recommended)

Once DNS resolves to the host:

```bash
sudo apt install -y certbot python3-certbot-apache
sudo certbot --apache -d server1.kingg.in -d server2.kingg.in -d server3.kingg.in
```

Certbot adds the HTTPS vhost, redirects HTTP to HTTPS and renews certificates automatically. Traffic between the host and the container stays on the private bridge. After enabling HTTPS, set the header in the 443 vhost to `X-Forwarded-Proto "https"` so apps such as Laravel and ERPNext know they are behind HTTPS.

## Daily administration

```bash
lxc list                          # status and IPs
lxc exec server1 -- bash          # admin shell into a container
lxc stop|start|restart server1
lxc snapshot server1 snap1        # backup point
lxc restore server1 snap1
lxc config set server1 limits.memory 6GB   # optional cap later
sudo passwd user1                 # reset a user's host password
```

## Resetting a forgotten password

The server owner (admin) manages all user passwords. Log in to the host as `ubuntu` and run:

```bash
sudo passwd user1
```

Enter the new password twice and tell the user. It takes effect immediately, with no restart needed. Users cannot change their host password themselves, because they are always placed inside their container. The `passwd` command there would only change the container's root password, not the host login, so password changes and resets go through the admin.

## Adding another user and server

Repeat the same pattern with a new number (for example `server4`, `user4`, IP `10.125.169.14`):

1. `lxc launch ubuntu:24.04 server4` and set its fixed IP (step 4).
2. `sudo adduser user4` (step 5).
3. Create the internal host-to-container link for `user4` (step 6).
4. Add a `Match User user4` block to `/etc/ssh/sshd_config.d/00-containers.conf` (step 7).
5. Add an Apache vhost for `server4.kingg.in` and an A record (or rely on the wildcard).

## Optional: host swap

Containers share host RAM with no limits, so a swap file is a useful safety net (none is configured by default).

```bash
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

## Troubleshooting

| Problem | Fix |
|---|---|
| `lxc list` shows empty IPv4 | Apply the iptables rules in step 3, then `lxc restart` the containers |
| Container has an IP but no internet | Check `lxc network show lxdbr0` has `ipv4.nat: "true"` and re-apply step 3 |
| `Permission denied (publickey)` on host login | Password login not enabled for that user: check step 7 and `sudo sshd -T -C user=user1,host=x,addr=1.1.1.1 \| grep passwordauthentication` |
| Login works but asks for a second password inside the container | The internal link from step 6 is missing for that user: redo step 6 for that number |
| User forgot their password | Admin runs `sudo passwd userN` on the host (see "Resetting a forgotten password") |
| SSH connection timed out | The port is not open in the OCI security list or iptables |
| Domain shows the wrong page or the host's default page | The vhost is not enabled or `ServerName` does not match: `sudo apache2ctl -S` |
| 502 or 503 from the domain | The web server inside the container is down or the vhost port is wrong: `curl -I http://10.125.169.11` from the host |
| Any SSH problem | `sudo journalctl -u ssh -n 20 --no-pager` |

## Security notes

- Containers are unprivileged, so root inside a container is not root on the host.
- Users have no host shell, no TCP forwarding and no X11. They must not be added to the `lxd` group.
- No resource limits means one heavy user can slow the host and the others. Add `limits.*` values if that becomes a problem.
- SSH is exposed publicly with password login for these three users: use strong passwords and keep `fail2ban` running.
- Back up containers with `lxc snapshot`, and keep an off-server copy of important data.
