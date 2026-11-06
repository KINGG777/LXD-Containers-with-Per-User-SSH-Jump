# Kingg-Server: LXD Containers per User

One Oracle Cloud VM is split into 3 Ubuntu 24.04 containers. Each user logs in to the host with a password and lands directly in their own container as root. Each container is published on its own domain through the host's Apache.

| Host user | Container | IP | Domain |
|---|---|---|---|
| user1 | server1 | 10.125.169.11 | server1.kingg.in |
| user2 | server2 | 10.125.169.12 | server2.kingg.in |
| user3 | server3 | 10.125.169.13 | server3.kingg.in |

```
user ── SSH (password) ──► Host ──► own container (root)
browser ── serverN.kingg.in ──► Host Apache (proxy) ──► container
```

Connect: `ssh user1@<HOST_PUBLIC_IP>`

## Setup (run as `ubuntu` on the host)

### 1. Install LXD

```bash
sudo snap install lxd
sudo lxd init
```

Answers: storage backend `dir`, bridge `lxdbr0`, IPv4 `auto`, IPv6 `none`, other prompts default or `no`.

### 2. Create containers

```bash
lxc launch ubuntu:24.04 server1
lxc launch ubuntu:24.04 server2
lxc launch ubuntu:24.04 server3
```

### 3. Firewall rules for the bridge

```bash
sudo iptables -I INPUT -i lxdbr0 -j ACCEPT
sudo iptables -I FORWARD -i lxdbr0 -j ACCEPT
sudo iptables -I FORWARD -o lxdbr0 -j ACCEPT
sudo netfilter-persistent save
lxc restart server1 server2 server3
```

### 4. Fixed IPs

```bash
lxc stop server1 server2 server3
lxc config device override server1 eth0 ipv4.address=10.125.169.11
lxc config device override server2 eth0 ipv4.address=10.125.169.12
lxc config device override server3 eth0 ipv4.address=10.125.169.13
lxc start server1 server2 server3
```

### 5. Host users

```bash
sudo adduser user1
sudo adduser user2
sudo adduser user3
```

### 6. Internal host-to-container link (admin only)

Lets the host open the container root session automatically, so users type only their host password.

```bash
for i in 1 2 3; do
  lxc exec server$i -- bash -c "mkdir -p /root/.ssh && chmod 700 /root/.ssh && touch /root/.ssh/authorized_keys && chmod 600 /root/.ssh/authorized_keys && sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin prohibit-password/' /etc/ssh/sshd_config && systemctl restart ssh"
  sudo -u user$i mkdir -p /home/user$i/.ssh
  sudo -u user$i ssh-keygen -t ed25519 -N "" -f /home/user$i/.ssh/id_ed25519 <<< y
  sudo cat /home/user$i/.ssh/id_ed25519.pub | lxc exec server$i -- tee -a /root/.ssh/authorized_keys
  sudo -u user$i ssh -o StrictHostKeyChecking=accept-new root@10.125.169.1$i hostname
done
```

### 7. Password login and forced jump

`/etc/ssh/sshd_config.d/00-containers.conf`:

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

### 8. Domains

1. DNS: add A records `server1`, `server2`, `server3` for `kingg.in` pointing to the host public IP.
2. Open TCP 80 and 443 in the Oracle security list and host iptables.
3. Install a web server inside a container, for example `lxc exec server1 -- apt install -y apache2`.
4. Create the proxy vhosts on the host:

```bash
sudo a2enmod proxy proxy_http proxy_wstunnel headers rewrite

for i in 1 2 3; do
sudo tee /etc/apache2/sites-available/server$i.kingg.in.conf > /dev/null <<EOF
<VirtualHost *:80>
    ServerName server$i.kingg.in
    ProxyPreserveHost On
    ProxyRequests Off
    ProxyPass / http://10.125.169.1$i:80/
    ProxyPassReverse / http://10.125.169.1$i:80/
</VirtualHost>
EOF
done

sudo a2ensite server1.kingg.in server2.kingg.in server3.kingg.in
sudo apache2ctl configtest && sudo systemctl reload apache2
```

Test: `curl -I http://10.125.169.11` (from host) and open `http://server1.kingg.in`.

## Admin commands

```bash
lxc list                      # status and IPs
lxc exec server1 -- bash      # admin shell into a container
lxc restart server1
sudo passwd user1             # reset a forgotten user password
```

Users cannot change their host password themselves; the admin resets it with `sudo passwd`.

## Remove everything (in this order)

**1. Apache vhosts**
```bash
sudo a2dissite server1.kingg.in server2.kingg.in server3.kingg.in
sudo rm /etc/apache2/sites-available/server{1,2,3}.kingg.in.conf
sudo systemctl reload apache2
```

**2. SSH jump config**
```bash
sudo rm /etc/ssh/sshd_config.d/00-containers.conf
sudo systemctl restart ssh
```

**3. Containers**
```bash
lxc delete server1 server2 server3 --force
```

**4. LXD image, network, storage**
```bash
lxc image delete $(lxc image list -f csv -c f)
lxc profile device remove default eth0
lxc profile device remove default root
lxc network delete lxdbr0
lxc storage delete default
```

**5. LXD itself**
```bash
sudo snap remove --purge lxd
```

**6. Host users**
```bash
sudo pkill -u user1; sudo deluser --remove-home user1
sudo pkill -u user2; sudo deluser --remove-home user2
sudo pkill -u user3; sudo deluser --remove-home user3
```

**7. Firewall rules**
```bash
sudo iptables -D INPUT -i lxdbr0 -j ACCEPT
sudo iptables -D FORWARD -i lxdbr0 -j ACCEPT
sudo iptables -D FORWARD -o lxdbr0 -j ACCEPT
sudo netfilter-persistent save
```

**8. DNS**

Delete the `server1`, `server2`, `server3` A records at the DNS provider.
