# Development-of-a-Secure-and-Fully-Integrated-Network-Services-Infrastructure-Using-Linux-Ubuntu
---

## Overview

This project documents the full installation, configuration, and verification of a Linux-based server environment using **Ubuntu 22.04 LTS** on Oracle VM VirtualBox. It covers DNS, DHCP, Email, Web, LDAP servers, and SSL/TLS encryption.

---

## Table of Contents

1. [Environment Setup](#1-environment-setup)
2. [DNS Server — BIND9](#2-dns-server--bind9)
3. [DHCP Server](#3-dhcp-server)
4. [Email Server — Postfix & Dovecot](#4-email-server--postfix--dovecot)
5. [Web Server — Apache](#5-web-server--apache)
6. [LDAP Server — OpenLDAP & PHPLDAPAdmin](#6-ldap-server--openldap--phpldapadmin)
7. [SSL/TLS Encryption](#7-ssltls-encryption)
8. [Network Configuration Summary](#8-network-configuration-summary)

---

## 1. Environment Setup

| Component | Detail |
|---|---|
| Host OS | Windows 11 Pro |
| Hypervisor | Oracle VM VirtualBox 7.0 |
| Guest OS | Ubuntu 22.04 LTS (64-bit) |
| RAM Allocated | 2048 MB |
| Username | tp046712 |
| Hostname | tp046712 |

### Steps
- Downloaded VirtualBox from [virtualbox.org](https://www.virtualbox.org/)
- Downloaded Ubuntu from [ubuntu.com/download/desktop](https://ubuntu.com/download/desktop)
- Created VM, allocated RAM (≤ half of host), set virtual hard disk
- Installed Ubuntu with "Erase disk and Install Ubuntu" option
- Updated OS regularly with `sudo apt-get update`

---

## 2. DNS Server — BIND9

### What is DNS?
DNS (Domain Name System) translates human-readable domain names (e.g. `tp046712.local`) into IP addresses. Four components work together: DNS Resolver → Root Nameserver → TLD Nameserver → Authoritative Nameserver.

### What is BIND?
BIND (Berkeley Internet Name Domain) is the most widely used DNS server software on Linux. BIND9 can act as a resolver, authoritative nameserver, or stub resolver.

### Key Configuration Files

| File | Purpose |
|---|---|
| `/etc/bind/named.conf.options` | Main options: forwarders, listen interfaces, recursion |
| `/etc/bind/named.conf.local` | Zone declarations |
| `/etc/bind/db.tp046712.local` | Forward zone file with SOA, NS, A records |

### Configuration Highlights

**Forwarders** (`named.conf.options`):
```
forwarders {
    8.8.8.8;
    8.8.4.4;
};
listen-on port 53 { 127.0.0.1; };
allow-recursion { 127.0.0.1; };
```

**Zone Declaration** (`named.conf.local`):
```
zone "tp046712.local" {
    type master;
    file "/etc/bind/db.tp046712.local";
};
```

**Zone File Records** (`db.tp046712.local`):

| Type | Name | Purpose | Value |
|---|---|---|---|
| SOA | Start of Authority | Identifies zone | tp046712.local |
| NS | Name Server | Authoritative nameserver | tp046712.local |
| A | Address | Maps FQDN to IP | 10.0.2.15 |

### Key Commands

```bash
sudo apt install bind9
sudo apt install dnsutils
sudo nano /etc/bind/named.conf.options
sudo named-checkconf              # Validate config syntax
sudo service bind9 restart
sudo ufw allow Bind9
nslookup tp046712.local           # Test DNS resolution
```

---

## 3. DHCP Server

### What is DHCP?
DHCP (Dynamic Host Configuration Protocol) automatically assigns IP addresses, subnet masks, default gateways, and DNS server addresses to network hosts. The handshake process: DISCOVER → OFFER → REQUEST → ACKNOWLEDGE.

### Key Configuration Files

| File | Purpose |
|---|---|
| `/etc/dhcp/dhcpd.conf` | DHCP scope, range, and options |
| `/etc/netplan/01-networkmanager-all.yaml` | Network interface configuration |

### Subnet Configuration

```
subnet 10.0.2.0 netmask 255.255.255.0 {
    range 10.0.2.100 10.0.2.200;
    option routers 10.0.2.1;
    option domain-name-servers 127.0.0.53;
}
INTERFACESv4="enp0s3";
```

### Netplan Configuration

```yaml
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    enp0s3:
      dhcp4: true
```

### Key Commands

```bash
sudo apt install isc-dhcp-server
sudo nano /etc/dhcp/dhcpd.conf
sudo netplan apply
sudo dhcpd -t                          # Test config syntax
sudo systemctl start isc-dhcp-server
sudo systemctl enable isc-dhcp-server
sudo systemctl status isc-dhcp-server
```

---

## 4. Email Server — Postfix & Dovecot

### What is Postfix?
Postfix is a Mail Transfer Agent (MTA) for Linux. It handles sending and routing email via SMTP. Configured here as an "Internet Site" with SASL and TLS authentication.

### What is Dovecot?
Dovecot is a Mail Delivery Agent (MDA) supporting IMAP and POP3. It handles incoming mail retrieval and integrates with Postfix for SASL authentication.

### Key Configuration Files

| File | Purpose |
|---|---|
| `/etc/postfix/main.cf` | Postfix main configuration |
| `/etc/dovecot/conf.d/10-mail.conf` | Dovecot mail location |
| `/etc/dovecot/conf.d/10-master.conf` | Dovecot SASL socket config |

### Postfix Settings

```
myhostname = tp046712.local
mydestination = $myhostname, tp046712.local, localhost.localdomain, localhost
home_mailbox = Maildir/
```

### SMTP Authentication (SASL)

```bash
sudo postconf -e 'smtpd_sasl_type = dovecot'
sudo postconf -e 'smtpd_sasl_path = private/auth'
sudo postconf -e 'smtpd_sasl_auth_enable = yes'
sudo postconf -e 'smtpd_sasl_security_options = noanonymous'
sudo postconf -e 'smtpd_sasl_local_domain = $myhostname'
sudo postconf -e 'broken_sasl_auth_clients = yes'
```

### Dovecot SASL Socket (`10-master.conf`)

```
unix_listener /var/spool/postfix/private/auth {
    mode = 0666
    user = postfix
    group = postfix
}
```

### Key Commands

```bash
sudo apt install postfix
sudo apt install dovecot-core dovecot-imapd dovecot-pop3d
sudo systemctl restart postfix
sudo systemctl restart dovecot
sudo systemctl status postfix
sudo systemctl status dovecot
# Test sending
echo "Test Email" | mail -s "Test Subject" tp046712
```

### Mail Client Setup (Mozilla Thunderbird)
- Protocol: IMAP
- Incoming: `tp046712.local`, Port 143, STARTTLS
- Outgoing: `tp046712.local`, Port 25, STARTTLS

---

## 5. Web Server — Apache

### What is Apache?
Apache HTTP Server is the most widely used web server on Linux. Used in the LAMP stack, it serves static and dynamic web content.

### Key Configuration Files

| File | Purpose |
|---|---|
| `/etc/apache2/sites-available/tp046712.local.conf` | Virtual host configuration |
| `/var/www/tp046712.local/public_html/index.html` | Website content |

### Virtual Host Configuration

```apache
<VirtualHost 10.0.2.15:80>
    ServerName tp046712.local
    ServerAdmin webmaster@tp046712.local
    DocumentRoot /var/www/tp046712.local/public_html
    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

### Key Commands

```bash
sudo apt install apache2
sudo mkdir -p /var/www/tp046712.local/public_html
sudo chown -R www-data:www-data /var/www/tp046712.local/public_html
sudo chmod -R 755 /var/www/tp046712.local/public_html
sudo a2ensite tp046712.local.conf
sudo a2enmod rewrite
sudo systemctl restart apache2
```

### Accessing the Website
- By IP: `http://10.0.2.15`
- By domain: `http://tp046712.local`

---

## 6. LDAP Server — OpenLDAP & PHPLDAPAdmin

### What is LDAP?
LDAP (Lightweight Directory Access Protocol) is a protocol for querying and managing directory services — storing user accounts, groups, and organisational data.

### What is PHPLDAPAdmin?
A web-based GUI tool for managing an OpenLDAP directory. Allows browsing the LDAP tree, creating/editing entries, and managing organisational units.

### Key Configuration Files

| File | Purpose |
|---|---|
| `/etc/ldap/ldap.conf` | Base DN and server URI |
| `/etc/phpldapadmin/config.php` | PHPLDAPAdmin configuration |

### LDAP Configuration (`/etc/ldap/ldap.conf`)

```
BASE    dc=tp046712,dc=local
URI     ldap://tp046712.local
```

### Organisational Structure Created

```
dc=tp046712, dc=local
└── ou=groups
    ├── cn=admin  (GID: 500, posixGroup)
    └── cn=users  (GID: 501, posixGroup)
└── ou=Users
```

### Key Commands

```bash
sudo apt-get install slapd ldap-utils
sudo dpkg-reconfigure slapd        # Interactive reconfiguration
sudo apt install phpldapadmin
sudo nano /etc/phpldapadmin/config.php
sudo systemctl status slapd
sudo slapcat                       # View LDAP directory contents
ldapsearch -x                      # Test LDAP connection
ldapwhoami -H ldap:// -x           # Verify LDAP authentication
```

### PHPLDAPAdmin Login
- URL: `http://tp046712.local/phpldapadmin`
- Username: `cn=admin,dc=tp046712,dc=local`

---

## 7. SSL/TLS Encryption

Self-signed X.509 certificates were generated for both Postfix and Apache to enable encrypted communications.

### Generating Certificates

```bash
# For Postfix
sudo openssl req -new -x509 -nodes \
  -out /etc/ssl/certs/postfix.pem \
  -keyout /etc/ssl/private/postfix.key \
  -days 3650

# For Apache
sudo openssl req -new -x509 -nodes \
  -out /etc/ssl/certs/apache.pem \
  -keyout /etc/ssl/private/apache.key \
  -days 3650
```

### Postfix TLS Config (`/etc/postfix/main.cf`)

```
smtpd_tls_cert_file = /etc/ssl/certs/postfix.pem
smtpd_tls_key_file  = /etc/ssl/private/postfix.key
```

### Apache SSL Setup

```bash
sudo a2enmod ssl
sudo nano /etc/apache2/sites-available/default-ssl.conf
# Set:
# SSLCertificateFile    /etc/ssl/certs/apache.pem
# SSLCertificateKeyFile /etc/ssl/private/apache.key
```

### Testing SSL/TLS

```bash
openssl s_client -starttls smtp -connect tp046712.local:25
```

> Note: A "self-signed certificate" verification error is expected since these are not CA-issued certificates.

---

## 8. Network Configuration Summary

| Setting | Value |
|---|---|
| Interface | enp0s3 |
| IP Address | 10.0.2.15 |
| Subnet Mask | 255.255.255.0 |
| Broadcast | 10.0.2.255 |
| Default Gateway | 10.0.2.2 |
| Loopback | 127.0.0.1 |
| MAC Address | 08:00:27:90:97:d0 |
| DNS Server | 127.0.0.53 |
| Domain | tp046712.local |

---

## Troubleshooting Summary

| # | Issue | Resolution |
|---|---|---|
| 1 | BIND syntax error (`dnssec-enable` deprecated) | Removed deprecated option; verified with `sudo named-checkconf` |
| 2 | Zone file empty on first open | Used `sudo cp /etc/bind/db.local` as template |
| 3 | BIND service errors on status check | Fixed zone file config; restarted service |
| 4 | YAML indentation error in Netplan | Corrected indentation; re-ran `sudo netplan apply` |
| 5 | DHCP config syntax error | Fixed option name spelling; verified with `sudo dhcpd -t` |
| 6 | Dovecot failed to restart | Fixed config indentation errors |
| 7 | Apache virtual host config empty | Copied default config as template with `sudo cp` |
| 8 | PHPLDAPAdmin PHP deprecated function error | Replaced buggy files with newer version (1.2.6.4) |

