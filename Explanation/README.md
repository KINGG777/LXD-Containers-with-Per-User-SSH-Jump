Multi-User Hosting Platform Using LXD Containers on Oracle Cloud

1. Project Overview

This project creates multiple isolated Linux hosting environments on a single Oracle Cloud Ubuntu server using LXD containers.

Each user gets:

- A separate Linux container.
- A separate host login account.
- Root access inside their assigned container.
- A dedicated private IP address.
- A dedicated domain name.
- Independent Apache, PHP, database, and application installations.
- No direct root access to the main Ubuntu host.

The main objective is to host multiple independent websites or applications on one cloud server while keeping their environments separated.

2. Architecture

Main server

Component| Configuration
Cloud provider| Oracle Cloud
Host OS| Ubuntu 24.04 LTS
Architecture| ARM64
Container manager| LXD
Container networking| lxdbr0
Container subnet| 10.125.169.0/24
Web server on host| Apache
Public web ports| 80 and 443
SSH port| 22

Container allocation

Container| Host username| Private IP| Domain
server1| user1| 10.125.169.11| server1.kingg.in
server2| user2| 10.125.169.12| server2.kingg.in
server3| user3| 10.125.169.13| server3.kingg.in

Architecture diagram

                 INTERNET
                     |
                     |
             Oracle Cloud VM
             Ubuntu 24.04 Host
             Public IP Address
                     |
          +----------+----------+
          |                     |
          |                     |
      SSH Port 22          Apache Proxy
          |                  80 / 443
          |                     |
     Host Login                 |
          |                     |
    +-----+-----+          Domain Routing
    |     |     |               |
  user1 user2 user3             |
    |     |     |               |
    |     |     |               |
    v     v     v               |
  LXD   LXD   LXD                |
 server1 server2 server3         |
    |     |     |                |
 .11   .12   .13                 |
    |     |     |                |
    +-----+-----+----------------+
              |
          lxdbr0 bridge

3. Understanding LXD

LXD is a system container manager.

It allows us to run multiple Linux environments inside one physical or virtual server.

For example:

- The main Ubuntu server is the host.
- "server1" is a separate Ubuntu environment.
- "server2" is another separate Ubuntu environment.
- "server3" is another separate Ubuntu environment.

Each container has its own:

- Filesystem.
- Hostname.
- Installed packages.
- Users and permissions.
- Network address.
- Running services.

Containers share the host's Linux kernel. They are isolated environments, but they are not equivalent to separate physical servers or virtual machines.

4. Install LXD

Update the host:

sudo apt update
sudo apt upgrade -y

Install LXD:

sudo snap install lxd

Check the installation:

lxd version

Initialize LXD:

sudo lxd init

During initialization, configure:

- Storage backend: "dir"
- Network bridge: "lxdbr0"
- IPv4 subnet: "10.125.169.0/24"
- IPv6: Disable if not required.

The storage pool stores container filesystems.

The network bridge connects containers to the host and provides their private networking.

5. Create the Containers

Launch three Ubuntu containers:

lxc launch ubuntu:24.04 server1
lxc launch ubuntu:24.04 server2
lxc launch ubuntu:24.04 server3

Check the containers:

lxc list

Expected result:

+---------+---------+-----------------------+
| NAME    | STATE   | IPV4                  |
+---------+---------+-----------------------+
| server1 | RUNNING | 10.125.169.11         |
| server2 | RUNNING | 10.125.169.12         |
| server3 | RUNNING | 10.125.169.13         |
+---------+---------+-----------------------+

The addresses shown above are the intended allocation. Verify actual addresses with "lxc list".

6. Configure Fixed Private IP Addresses

Containers should retain their assigned IP addresses.

Example for server1:

lxc config device set server1 eth0 ipv4.address 10.125.169.11

For server2:

lxc config device set server2 eth0 ipv4.address 10.125.169.12

For server3:

lxc config device set server3 eth0 ipv4.address 10.125.169.13

Restart the containers:

lxc restart server1
lxc restart server2
lxc restart server3

Verify:

lxc list

7. Create Separate Host Login Accounts

Create one Linux account for each container user.

sudo adduser user1
sudo adduser user2
sudo adduser user3

These accounts exist on the main Ubuntu host.

They are not the same as users inside the containers.

The intended login relationship is:

user1 → server1
user2 → server2
user3 → server3

Do not add these accounts to the host's "sudo" group. The intended access is root inside the assigned container, not administrative access to the host.

8. Understand SSH Access to Containers

This is an important part of the architecture.

We want a user to run:

ssh user1@YOUR_PUBLIC_IP

And automatically enter:

root@server1:~#

To achieve this, we establish an internal SSH connection from the host account into its assigned container.

How it works

There are two separate SSH connections:

Connection 1: External SSH

Your Computer
     |
     | SSH user1@PUBLIC_IP
     |
     v
Main Ubuntu Host

The host authenticates the password of "user1".

Connection 2: Internal SSH

Main Ubuntu Host
     |
     | SSH using user1's private key
     |
     v
LXD Container server1
     |
     v
root@server1

The internal SSH key allows the host account to authenticate as root inside its assigned container without asking for another password.

9. Generate the SSH Key for user1

Create the SSH directory:

sudo mkdir -p /home/user1/.ssh

Set ownership:

sudo chown -R user1:user1 /home/user1/.ssh

Set directory permissions:

sudo chmod 700 /home/user1/.ssh

Generate an Ed25519 key:

sudo -u user1 -H ssh-keygen -t ed25519 -N "" -f /home/user1/.ssh/id_ed25519

This creates two files:

File| Purpose
id_ed25519| Private key
id_ed25519.pub| Public key

The private key must remain on the host and must not be shared.

Set file permissions:

sudo chmod 600 /home/user1/.ssh/id_ed25519
sudo chmod 644 /home/user1/.ssh/id_ed25519.pub

10. Install the Public Key Inside server1

Create the SSH directory inside the container:

sudo lxc exec server1 -- mkdir -p /root/.ssh

Set permissions:

sudo lxc exec server1 -- chmod 700 /root/.ssh

Copy the public key:

sudo bash -c 'cat /home/user1/.ssh/id_ed25519.pub | lxc exec server1 -- tee -a /root/.ssh/authorized_keys'

Set authorized key permissions:

sudo lxc exec server1 -- chmod 600 /root/.ssh/authorized_keys

The container now trusts the corresponding private key held by user1 on the host.

11. Test Internal SSH Connection

Run this command from the main Ubuntu host:

sudo -u user1 -H ssh -i /home/user1/.ssh/id_ed25519 root@10.125.169.11 hostname

Expected output:

server1

If the hostname is returned successfully, the internal SSH connection is working.

Important: Actual SSH Issue We Resolved

Initially, the following commands failed:

ssh root@10.125.169.11

sudo ssh root@10.125.169.11

The reason was that SSH was looking for the wrong private key.

We also initially supplied the SSH directory instead of the private key:

ssh -i /home/user1/.ssh/ root@10.125.169.11

This is incorrect because "-i" must point to the actual private key file.

The working command was:

ssh -i /home/user1/.ssh/id_ed25519 root@10.125.169.11

This successfully opened the container and displayed:

root@server1:~#

Current progress: Direct SSH from the host into server1 has been successfully tested.

12. Configure Automatic SSH Login

Once the internal SSH test works, configure the host SSH server to automatically execute the internal SSH command when user1 logs in.

Create a configuration file:

sudo nano /etc/ssh/sshd_config.d/00-containers.conf

Add:

Match User user1
    PasswordAuthentication yes
    ForceCommand ssh -t -i /home/user1/.ssh/id_ed25519 -o IdentitiesOnly=yes root@10.125.169.11
    AllowTcpForwarding no
    X11Forwarding no

Add equivalent entries for the other users after generating and installing their individual keys:

Match User user2
    PasswordAuthentication yes
    ForceCommand ssh -t -i /home/user2/.ssh/id_ed25519 -o IdentitiesOnly=yes root@10.125.169.12
    AllowTcpForwarding no
    X11Forwarding no

Match User user3
    PasswordAuthentication yes
    ForceCommand ssh -t -i /home/user3/.ssh/id_ed25519 -o IdentitiesOnly=yes root@10.125.169.13
    AllowTcpForwarding no
    X11Forwarding no

Validate the SSH configuration:

sudo sshd -t

If there are no errors, reload SSH:

sudo systemctl reload ssh

Before relying on this configuration, verify its effective settings:

sudo sshd -T -C user=user1,host=localhost,addr=127.0.0.1 | grep -E 'passwordauthentication|forcecommand|allowtcpforwarding|x11forwarding'

Test from another terminal or computer:

ssh user1@YOUR_PUBLIC_IP

Expected destination:

root@server1:~#

Keep an existing host administration session open while testing SSH changes so that you do not lock yourself out.

This is a basic forced-command gateway. Before using it commercially, further restrict SSH features, review shell access and forwarding, and test the complete isolation design.

13. Install Apache Inside Each Container

The host Apache server will receive public web requests and forward them to the correct container.

Each container runs its own Apache server.

Enter server1:

lxc exec server1 -- bash

Update packages:

apt update
apt upgrade -y

Install Apache:

apt install apache2 -y

Enable Apache:

systemctl enable --now apache2

Check:

systemctl status apache2

Repeat for server2 and server3.

14. Configure Apache Reverse Proxy on the Host

The main host Apache server routes requests to the correct container based on the requested domain.

Install Apache on the host if it is not already installed:

sudo apt install apache2 -y

Enable the required modules:

sudo a2enmod proxy proxy_http proxy_wstunnel ssl rewrite headers

Restart Apache:

sudo systemctl restart apache2

Configure server1 domain

Create:

sudo nano /etc/apache2/sites-available/server1.conf

Add:

<VirtualHost *:80>
    ServerName server1.kingg.in

    ProxyPreserveHost On
    ProxyPass / http://10.125.169.11/
    ProxyPassReverse / http://10.125.169.11/

    ErrorLog ${APACHE_LOG_DIR}/server1-error.log
    CustomLog ${APACHE_LOG_DIR}/server1-access.log combined
</VirtualHost>

Configure server2 domain

Create:

sudo nano /etc/apache2/sites-available/server2.conf

Add:

<VirtualHost *:80>
    ServerName server2.kingg.in

    ProxyPreserveHost On
    ProxyPass / http://10.125.169.12/
    ProxyPassReverse / http://10.125.169.12/
</VirtualHost>

Configure server3 domain

Create:

sudo nano /etc/apache2/sites-available/server3.conf

Add:

<VirtualHost *:80>
    ServerName server3.kingg.in

    ProxyPreserveHost On
    ProxyPass / http://10.125.169.13/
    ProxyPassReverse / http://10.125.169.13/
</VirtualHost>

Enable the sites:

sudo a2ensite server1.conf
sudo a2ensite server2.conf
sudo a2ensite server3.conf

Validate:

sudo apache2ctl configtest

Reload:

sudo systemctl reload apache2

For applications using WebSockets, configure a WebSocket proxy rule for the application's actual WebSocket endpoint and port. For example, Frappe deployments commonly use Socket.IO on port 9000; a generic port-80 proxy alone does not configure that endpoint.

15. Configure DNS

In your DNS provider, create A records:

Type| Name| Value
A| server1| Oracle public IP
A| server2| Oracle public IP
A| server3| Oracle public IP

All domains point to the same public IP.

Apache identifies the requested domain and forwards the request to the corresponding container.

16. Configure HTTPS Using Certbot

Install Certbot:

sudo apt install certbot python3-certbot-apache -y

Request certificates:

sudo certbot --apache -d server1.kingg.in
sudo certbot --apache -d server2.kingg.in
sudo certbot --apache -d server3.kingg.in

Check certificate renewal:

sudo certbot renew --dry-run

The public HTTPS connection terminates at the host Apache reverse proxy. Backend traffic to the containers is HTTP over the private bridge unless backend TLS is configured separately.

17. Oracle Cloud Firewall Configuration

Allow the required inbound ports in the Oracle Cloud network security rules.

Port| Protocol| Purpose
22| TCP| SSH administration
80| TCP| HTTP
443| TCP| HTTPS

Do not expose container SSH ports publicly unless there is a specific requirement.

Use Ubuntu firewall rules and network policies to restrict access to private container addresses. The containers should not be able to access each other's private services or sensitive host services without an explicit requirement.

18. Container Resource Management

One physical server is shared by all containers.

Set CPU and memory limits according to the available server resources.

Example:

lxc config set server1 limits.cpu 2
lxc config set server1 limits.memory 2GiB

For server2:

lxc config set server2 limits.cpu 2
lxc config set server2 limits.memory 2GiB

For server3:

lxc config set server3 limits.cpu 2
lxc config set server3 limits.memory 2GiB

These are examples, not recommended fixed allocations. Check the VM's actual CPU and RAM before applying limits.

Monitor container resource usage:

lxc info server1
lxc info server2
lxc info server3

Check host resources:

free -h
df -h
top

19. Useful LXD Commands

List containers

lxc list

Enter a container

lxc exec server1 -- bash

Stop a container

lxc stop server1

Start a container

lxc start server1

Restart a container

lxc restart server1

View container information

lxc info server1

View container configuration

lxc config show server1

View container logs

lxc info server1 --show-log

Create a snapshot

lxc snapshot server1 before-update

Restore a snapshot

lxc restore server1 before-update

Snapshots are stored locally and are not a replacement for off-server backups.

20. Troubleshooting

SSH permission denied

Check the key files:

sudo ls -la /home/user1/.ssh/

Check the authorized keys inside the container:

sudo lxc exec server1 -- ls -la /root/.ssh/

Test internal SSH:

sudo -u user1 -H ssh -i /home/user1/.ssh/id_ed25519 root@10.125.169.11 hostname

Container is not running

lxc list
lxc start server1

Apache is not working

Inside the container:

systemctl status apache2

On the host:

sudo systemctl status apache2
sudo apache2ctl configtest

Check connectivity

From the host:

ping -c 4 10.125.169.11

Test the container's Apache service:

curl -I http://10.125.169.11

Check SSH configuration

sudo sshd -t
sudo systemctl status ssh

Check Apache logs

sudo tail -f /var/log/apache2/error.log

21. Security Recommendations

Before using this architecture for multiple customers:

1. Keep LXD and Ubuntu updated.
2. Never expose container management APIs publicly.
3. Do not give container users host sudo privileges.
4. Keep each user's private SSH key protected.
5. Restrict container-to-container network traffic.
6. Restrict container access to host services.
7. Configure SSH forwarding and shell restrictions carefully.
8. Apply CPU, memory and disk limits.
9. Use independent backups stored outside the VM.
10. Monitor CPU, memory, storage and network activity.
11. Use strong authentication and Fail2ban or equivalent protection.
12. Test container escape and tenant isolation assumptions before commercial use.

Remember: LXD containers share the host kernel. For stronger customer isolation, consider separate virtual machines or dedicated hosts.

22. Adding a New Customer (server4)

The same process can be followed for another customer.

Example allocation:

Parameter| Value
Container| server4
Host user| user4
Private IP| 10.125.169.14
Domain| server4.kingg.in

Create the container:

lxc launch ubuntu:24.04 server4

Create the host account:

sudo adduser user4

Configure the fixed IP:

lxc config device set server4 eth0 ipv4.address 10.125.169.14

Generate a separate SSH key for user4, install its public key inside server4, and configure the corresponding SSH forced-command entry.

Then configure Apache and DNS for "server4.kingg.in".

23. Final Login and Traffic Flow

SSH flow

User's Computer
      |
      | ssh user1@PUBLIC_IP
      |
      v
Ubuntu Host
      |
      | Host authenticates user1
      |
      | Forced internal SSH
      |
      v
LXD server1
      |
      v
root@server1

Website traffic flow

Browser
   |
   | https://server1.kingg.in
   |
   v
Oracle Cloud Public IP
   |
   v
Host Apache Reverse Proxy
   |
   | Private IP: 10.125.169.11
   |
   v
LXD server1
   |
   v
Container Apache
   |
   v
Website / Application

24. Current Project Status

Task| Status
Oracle Ubuntu host| Configured
LXD installation| Configured
LXD bridge| Configured
Containers| Created
Container private IPs| Intended allocation; verify
Host user accounts| Created as part of setup
user1 SSH key| Created
Public key inside server1| Installed
Direct SSH into server1| Successfully tested
Automatic SSH login| Pending verification
Container Apache| Pending verification
Host reverse proxy| Pending verification
DNS| Pending verification
HTTPS| Pending verification

Current working command:

ssh -i /home/user1/.ssh/id_ed25519 root@10.125.169.11

Successful destination:

root@server1:~#

Next task: Verify the internal connection as "user1", configure automatic SSH login through "ForceCommand", and then complete web hosting and domain routing.

---

End of README.md
