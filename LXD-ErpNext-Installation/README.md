# ERPNext v16 on LXD Containers (Kingg-Server)

ERPNext (Frappe v16) installed directly inside each of the 3 LXD containers. The host's Apache sends each domain to its container, and certbot on the host provides HTTPS.

| Container | IP | Domain |
|---|---|---|
| server1 | 10.125.169.11 | erpnext1.kingg.in |
| server2 | 10.125.169.12 | erpnext2.kingg.in |
| server3 | 10.125.169.13 | erpnext3.kingg.in |

```
browser ──HTTPS──► Host Apache (certbot SSL) ──HTTP──► Container Apache :80 ──► Frappe (8000 / 9000)
```

Stack per container: Ubuntu 24.04, Apache, Python 3.14, Node 24, MariaDB, Redis, Supervisor, Frappe Bench, ERPNext (`version-16`).

The steps below use `server1` and `erpnext1.kingg.in`. For server2 and server3, replace `server1` with `server2` / `server3` and `erpnext1` with `erpnext2` / `erpnext3`. Install one container at a time, because the asset build uses a lot of RAM.

## A. Host setup (once)

**1. DNS:** add A records `erpnext1`, `erpnext2`, `erpnext3` for `kingg.in` pointing to the host public IP. TCP 80 and 443 must be open in the Oracle security list and host iptables.

**2. Proxy vhosts on the host:**

```bash
sudo a2enmod proxy proxy_http proxy_wstunnel headers rewrite

for i in 1 2 3; do
sudo tee /etc/apache2/sites-available/erpnext$i.kingg.in.conf > /dev/null <<EOF
<VirtualHost *:80>
    ServerName erpnext$i.kingg.in
    ProxyPreserveHost On
    ProxyRequests Off
    RequestHeader set X-Forwarded-Proto "https"
    RewriteEngine On
    RewriteCond %{HTTP:Upgrade} =websocket [NC]
    RewriteRule /(.*) ws://10.125.169.1$i:80/\$1 [P,L]
    ProxyPass / http://10.125.169.1$i:80/
    ProxyPassReverse / http://10.125.169.1$i:80/
</VirtualHost>
EOF
done

sudo a2ensite erpnext1.kingg.in erpnext2.kingg.in erpnext3.kingg.in
sudo apachectl configtest && sudo systemctl reload apache2
```

## B. Install inside each container

On the host, enter the container as the `ubuntu` user:

```bash
lxc exec server1 -- su - ubuntu
```

**1. Packages**

```bash
sudo ip link set dev eth0 mtu 1400
sudo apt update && sudo apt upgrade -y
sudo apt install -y software-properties-common curl
sudo add-apt-repository -y ppa:deadsnakes/ppa
sudo apt update
sudo apt install -y apache2 python3.14 python3.14-venv python3.14-dev python3-setuptools python3-pip pkg-config mariadb-server libmysqlclient-dev redis-server xvfb libfontconfig fontconfig xfonts-75dpi npm supervisor
sudo systemctl enable --now apache2 mariadb redis-server
python3.14 -m ensurepip --upgrade
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc && source ~/.bashrc
python3.14 -m pip install --upgrade pip
```

The `mtu 1400` line is a fix for stalled downloads inside containers on Oracle. It lasts until the container restarts.

**2. MariaDB** (interactive, run alone). Set the root password and answer `Y` to the rest.

```bash
sudo mysql_secure_installation
```

**3. Node 24 and Yarn**

```bash
curl -4 -o- https://raw.githubusercontent.com/nvm-sh/nvm/master/install.sh | bash
source ~/.bashrc
sudo npm install -g yarn
nvm install 24 && nvm use 24
node -v
```

**4. wkhtmltopdf**

```bash
cd /tmp
wget https://github.com/wkhtmltopdf/packaging/releases/download/0.12.6.1-2/wkhtmltox_0.12.6.1-2.jammy_$(dpkg --print-architecture).deb
sudo dpkg -i wkhtmltox_*.deb || sudo apt --fix-broken install -y
cd ~
wkhtmltopdf --version
```

**5. Frappe Bench** (takes several minutes)

```bash
sudo -H pip3 install frappe-bench --break-system-packages
sudo apt install --reinstall -y python3-pkg-resources
sudo chown ubuntu:ubuntu /var/www
cd /var/www
bench init Frappe --frappe-branch version-16 --python python3.14
cd /var/www/Frappe
```

The `python3-pkg-resources` line fixes Supervisor failing with `No module named 'pkg_resources'` after the bench install.

**6. Site and ERPNext.** `bench new-site` is interactive: run it alone. It asks for the MariaDB root password, then a new Administrator password.

```bash
bench new-site erpnext1.kingg.in
```

```bash
bench use erpnext1.kingg.in
bench get-app erpnext --branch version-16
bench --site erpnext1.kingg.in install-app erpnext
bench --site erpnext1.kingg.in migrate
```

**7. Supervisor**

```bash
sudo systemctl enable --now supervisor
sudo bench setup supervisor --yes
sudo ln -sf /var/www/Frappe/config/supervisor.conf /etc/supervisor/conf.d/Frappe.conf
sudo supervisorctl reread && sudo supervisorctl update
sudo supervisorctl status
```

All Frappe processes should show `RUNNING`.

**8. Apache inside the container**

```bash
sudo a2enmod proxy proxy_http proxy_wstunnel rewrite headers alias

sudo tee /etc/apache2/sites-available/erpnext1.kingg.in.conf > /dev/null <<'EOF'
<VirtualHost *:80>
    ServerName erpnext1.kingg.in

    Alias /assets /var/www/Frappe/sites/assets
    <Directory /var/www/Frappe/sites/assets>
        Require all granted
    </Directory>
    ProxyPass /assets !

    ProxyPreserveHost On
    ProxyRequests Off
    RewriteEngine On
    RewriteCond %{HTTP:Upgrade} =websocket [NC]
    RewriteRule /socket.io/(.*) ws://127.0.0.1:9000/socket.io/$1 [P,L]
    ProxyPass /socket.io http://127.0.0.1:9000/socket.io
    ProxyPassReverse /socket.io http://127.0.0.1:9000/socket.io

    ProxyPass / http://127.0.0.1:8000/
    ProxyPassReverse / http://127.0.0.1:8000/

    ErrorLog ${APACHE_LOG_DIR}/erpnext1.kingg.in-error.log
    CustomLog ${APACHE_LOG_DIR}/erpnext1.kingg.in-access.log combined
</VirtualHost>
EOF

sudo a2ensite erpnext1.kingg.in.conf
sudo apachectl configtest && sudo systemctl reload apache2
```

**9. Verify**

```bash
bench --site erpnext1.kingg.in list-apps
curl -I -H "Host: erpnext1.kingg.in" http://127.0.0.1
```

Expected: `frappe` and `erpnext` listed, and `200 OK` from gunicorn.

Then `exit` to leave the container.

## C. HTTPS (on the host)

Once DNS resolves to the host:

```bash
sudo apt install -y certbot python3-certbot-apache
sudo certbot --apache -d erpnext1.kingg.in -d erpnext2.kingg.in -d erpnext3.kingg.in
```

Certbot finds the host vhosts by `ServerName`, adds the HTTPS copy and a redirect, and renews automatically. The containers need no certificate. For a domain that isn't ready yet, run certbot only with the domains that are.

Then inside each container, tell Frappe its public URL:

```bash
cd /var/www/Frappe
bench --site erpnext1.kingg.in set-config host_name https://erpnext1.kingg.in
sudo supervisorctl restart all
```

Open `https://erpnext1.kingg.in`, log in as `Administrator`, and finish the setup wizard.

## Daily commands

```bash
sudo supervisorctl status                 # inside a container
sudo supervisorctl restart all
bench --site erpnext1.kingg.in backup     # backup to /var/www/Frappe/sites/erpnext1.kingg.in/private/backups
bench --site erpnext1.kingg.in set-admin-password NewPassword
```

## Remove everything (in this order)

**1. HTTPS certificate (on the host)**
```bash
sudo certbot certificates
sudo certbot delete --cert-name erpnext1.kingg.in
```
Repeat for any other certificate name that `certbot certificates` lists.

**2. Host vhosts (on the host)**
```bash
sudo a2dissite erpnext{1,2,3}.kingg.in erpnext{1,2,3}.kingg.in-le-ssl
sudo rm -f /etc/apache2/sites-available/erpnext{1,2,3}.kingg.in*.conf
sudo systemctl reload apache2
```
Some names may print "does not exist"; that is fine.

**3. Inside each container (`lxc exec server1 -- su - ubuntu`)**

Supervisor:
```bash
sudo supervisorctl stop all
sudo rm -f /etc/supervisor/conf.d/Frappe.conf
sudo supervisorctl reread && sudo supervisorctl update
```

Container Apache vhost:
```bash
sudo a2dissite erpnext1.kingg.in.conf
sudo rm -f /etc/apache2/sites-available/erpnext1.kingg.in.conf
sudo systemctl reload apache2
```

Site, database and bench (asks for the MariaDB root password):
```bash
cd /var/www/Frappe
bench drop-site erpnext1.kingg.in
cd /var/www && rm -rf Frappe
```

**4. Packages inside the container (optional)**
```bash
sudo pip3 uninstall -y frappe-bench --break-system-packages
rm -rf ~/.nvm
sudo apt remove -y wkhtmltox
sudo apt remove --purge -y mariadb-server redis-server supervisor
sudo apt autoremove -y
```
Keep `apache2` if the container's other web content uses it.

**5. DNS**

Delete the `erpnext1`, `erpnext2` and `erpnext3` A records at the DNS provider.

To wipe a whole server instead of removing items one by one, delete the container on the host with `lxc delete server1 --force` (see the LXD README).
