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

![image](./images/0_linux-vBox.png)

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

## 1. Installation and Initial configuration

Follow these steps to set up your Linux server as a Samba 4 Domain Controller.

---

### 1. Server Preparation

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

```bash
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
          - 127.0.0.1
          - 10.239.3.7
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

### 2. Configure Kerberos

During the installation of krb5-config, you will be prompted to enter your Kerberos realm.

Default Kerberos Realm: domain name in uppercase (LAB06.LAN).

Kerberos servers for your realm: LS06.lab06.lan

![image](./images/2_kerberos1.png)

Administrative server for your Kerberos realm: LS06.lab06.lan

![image](./images/2_kerberos2.png)

### 3. Configure Samba

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

![image](./images/3_samba.png)

Now change the dns.

```bash
sudo echo "nameserver 127.0.0.1
nameserver 192.168.6.1" | sudo tee >> /etc/resolv.conf
```

### 4. Restart Services

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

![image](./images/4_services-restarted.png)

### 5. Check domain

![image](./images/5_domain-check.png)

## 2. sprint 2

sprint 2
