# 💻 Linux Server Domain Controller

## Setup: Samba 4 AD/DC Implementation

This project outlines the comprehensive steps to transform a standard Linux server into a robust Primary Domain Controller (PDC) using Samba 4. This solution provides an open-source, cost-effective alternative to proprietary Active Directory services, offering centralized user authentication, group policy management, and DNS resolution for heterogeneous networks including Windows, Linux, and macOS clients.

This will be my personal server configurations change as needed:

Linux Server:

Password: admin_21
Internal network IP: 192.168.6.1
Bridge IP: 172.30.20.46/25 (enp0s3)
DHCP server: 192.168.6.2
DHCP range: 192.168.70.106-126
DNS: 192.168.6.1
NetBIOS: LS06 | lab06.lan
Domain: lab06.lan
FQDN: LS06.lab06.lan

![image](./images/1.0_linux-vBox.png)

---

## Features

- **Centralized Authentication**: Manage user accounts and authenticate clients against the Samba 4 AD/DC.
- **Group Policy Management**: Implement and enforce network-wide policies for enhanced security and configuration consistency.
- **Integrated DNS**: Samba 4 integrates seamlessly with BIND for robust DNS services within the domain.
- **Flexible Client Support**: Join Windows, Linux, and macOS clients to the domain.
- **Open Source Solution**: Leverage the power and flexibility of open-source software for enterprise-grade domain services.

---

## Getting Started

These instructions will get you a copy of the project up and running on your local machine for development and testing purposes.

### Prerequisites

To replicate this project, you will need the following:

- A virtual machine or physical server with a minimum of **2GB RAM** and **2 CPU cores**.
- A clean installation of **Ubuntu Server 24.04 LTS** (or a similar Debian/RPM-based distribution).
- A separate client machine to test the domain join process.

---

## Sprints

**Sprint 1:**

- Install the server

**Sprint 2:**

- Join Windows client
- Join Linux client

**Sprint 3:**

- Install Domain Controller (Samba, Kerberos)
- Create domain user accounts
- Manage policies (GPOs)
  - Minimum 8-character passwords
  - 5-minute lockout after 3 failed attempts
- Set up shared folders
- Scheduled tasks (crontab)
- Disk management (fstab)
- Process Management (ps -aux | htop)
**Sprint 4:**
- Create trust relationships between domains (with peer)

---

## Sprint 1: Installation and Initial configuration

Follow these steps to set up your Linux server as a Samba 4 Domain Controller.

### 1.1. Server Preparation

First, ensure your server is up-to-date correctly.

```bash
# Upgrade to new system version
sudo apt update

# Update the Linux version if needed (will restart)
# yes | sudo do-release-upgrade
sudo apt update && sudo apt upgrade -y

# Install necessary tools
sudo apt install -y vim net-tools ntp
```

Now correctly configure it with a hostname and static IP address.

```yaml
# Set hostname
sudo hostnamectl set-hostname LS06.lab06.lan

# Edit /etc/hosts to include your static IP and FQDN
echo "192.168.6.1    LS06.lab06.lan    LS06" | sudo tee /etc/hosts

# Edit netplan and set an static ip address
echo "network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: no
      addresses:
        - 172.30.20.46/25
      gateway4: 172.30.20.1
      nameservers:
        addresses:
          - 10.239.3.7
          - 10.239.3.8
          - 8.8.8.8
    enp0s8:
      dhcp4: no
      addresses:
        - 192.168.6.1/24
      gateway4: 192.168.6.1
      nameservers:
        addresses:
          - 127.0.0.1" | sudo tee /etc/netplan/50-cloud-init.yaml

# Apply changes
sudo netplan apply
```

Now install samba and kerberos.

```bash
# Install Samba and Dependencies
sudo apt install -y samba krb5-config winbind

```

---

## 2. Sprint 2: Join clients to the domain (Windows and Linux)

## 2.1. Join Windows to the domain

First we start with the base that we have a windows already installed

### 2.1.1 Configure Network Settings

Settings → Network & Internet → Ethernet/Wi-Fi

Ethernet/Wi-Fi
We Select Edit on IP assignment
Click Manual
Configure:
IP address: 192.168.6.20
Subnet mask: 255.255.255.0
Gateway: 192.168.6.1 (optional)
Primary DNS: 192.168.6.1
Secondary DNS: 10.239.3.7

After setting up the network configuration we check we have connectivity with the server

´´´bash
ping 192.168.6.1
nslookup lab06.lan
nslookup ls06.lab06.lan
´´´

### 2.1.2 Join Domain

Settings ->
![image](./images/2.1.1_.png)

System ->
![image](./images/2.1.2_.png)

Open Settings ->
![image](./images/2.1.3_.png)

 About ->
![image](./images/2.1.4_.png)

Related settings ->
![image](./images/2.1.5_.png)

-> Change settings -> Change
![image](./images/2.1.6_.png)

Click Domain or workgroup
Click Domain
Type: lab06.lan
Click OK
Use credentials:
Username: Administrator
Password: Admin_21
Click OK
![image](./images/2.1.7_.png)

Now the Client has joined the domain

---

## 3. Sprint 3: Install samba and kerberos, create and manage users, GPOs/PSOs, create shared folders, create programmed tasks and disk management

install dc(samba, kerberos)
install domain users
shared folders
manage policies (GPOs)(PSOs)
    Hide Control Panel.
    Set wallpaper to school logo
    Minimum 8-character passwords.
    5-minute lockout after 3 failed attempts.
programmed tasks (crontab)
disk management (fstab)

### 3.1. Configure Kerberos

During the installation of krb5-config, you will be prompted to enter your Kerberos realm.

Default Kerberos Realm: domain name in uppercase (LAB06.LAN).

Kerberos servers for your realm: LS06.lab06.lan

![image](./images/3.2_kerberos1.png)

Administrative server for your Kerberos realm: LS06.lab06.lan

![image](./images/3.2_kerberos2.png)

### 3.2. Configure Samba

Before provisioning, move or remove the default smb.conf file.

```bash
sudo mv /etc/samba/smb.conf /etc/samba/smb.conf.bkp

# Now, provision the domain (realm in uppercase)
sudo samba-tool domain provision --use-rfc2307 --interactive

realm=LAB06.LAN
domain=LAB06
server-role=dc
dns-backend=SAMBA_INTERNAL
dns-forwarder=192.168.6.1
adminpass="admin_21" 

```

![image](./images/3.3_samba.png)

Now change the dns.

```bash
sudo echo "nameserver 127.0.0.1
nameserver 192.168.6.1" | sudo tee >> /etc/resolv.conf
```

### 3.3. Restart Services

Restart services

```bash
# Stop and disable non-DC services (if they are running)
sudo systemctl stop smbd nmbd winbind
sudo systemctl disable smbd nmbd winbind

# Enable and start the Active Directory DC service
sudo systemctl unmask samba-ad-dc
sudo systemctl enable samba-ad-dc
sudo systemctl start samba-ad-dc
```

![image](./images/3.4_services-restarted.png)

### 5. Check domain

![image](./images/3.5_domain-check.png)
