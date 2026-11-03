# Enterprise Mail Server Suite

A complete, self-hosted enterprise mail server stack built on Debian Linux. The suite integrates Exim as the message transfer agent, Dovecot as the IMAP and POP3 server, Roundcube as the webmail client, and supporting services for authentication, spam filtering, and automated email workflows.

This project was developed as part of academic coursework in network infrastructure and demonstrates the design, configuration, and hardening of a production-grade mail system in a controlled lab environment.

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Components](#components)
- [Features](#features)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Testing and Verification](#testing-and-verification)
- [Troubleshooting](#troubleshooting)
- [Project Structure](#project-structure)
- [License](#license)
- [Author](#author)

## Overview

Email remains a critical component of enterprise communication, and understanding its underlying infrastructure is essential for network and security professionals. This project builds a functional mail server suite from the ground up, covering the full lifecycle of an email message: submission, routing, delivery, storage, retrieval, and web-based access.

The implementation emphasizes:

- Standards-compliant SMTP, IMAP, and POP3 services
- Secure authentication and encrypted transport
- Integration between independent services through well-defined interfaces
- Automation of administrative and notification workflows
- Reproducibility through documented configuration steps

## Architecture

The suite follows a modular design in which each service has a single, well-defined responsibility. Messages flow through the following stages:
Sender
|
v
[ Exim MTA ] <-- SMTP submission (port 587), authentication via Dovecot SASL
|
+--> Local delivery --> [ Dovecot LMTP ] --> Maildir storage
|
+--> Remote delivery --> Internet (MX resolution)
|
v
[ Roundcube ] <-- IMAP over TLS (port 993) --> [ Dovecot ]
|
v
User (browser)

text

Supporting services include a relational database for virtual users and Roundcube metadata, a spam and antivirus filter chain, and TLS certificates for encrypted communication.

## Components

| Component | Role | Version |
|-----------|------|---------|
| Debian | Operating system | [e.g. 12 Bookworm] |
| Exim | Message transfer agent (SMTP) | [e.g. 4.96] |
| Dovecot | IMAP and POP3 server, SASL provider, LMTP delivery agent | [e.g. 2.3.19] |
| Roundcube | Webmail client | [e.g. 1.6.x] |
| MariaDB or MySQL | Backend database for virtual users, aliases, and Roundcube | [e.g. 10.11] |
| SpamAssassin | Spam filtering | [e.g. 4.0] |
| ClamAV | Antivirus scanning | [e.g. 1.0] |
| OpenSSL | TLS certificate generation and management | System package |
| Certbot | Automated certificate renewal (optional) | [e.g. 2.x] |

## Features

- Full SMTP submission with STARTTLS and authentication
- IMAP and POP3 access with TLS enforcement
- Virtual domain and mailbox management stored in a database
- Webmail access through Roundcube with a consistent, responsive interface
- Spam and malware filtering on inbound messages
- Automated email workflows for administrative notifications and alerts
- DKIM signing and SPF and DMARC alignment for outbound mail
- Per-user quota enforcement and mailbox size reporting
- Logging and monitoring integration compatible with Nagios and journald

## Prerequisites

Before beginning the installation, ensure the following are available:

- A Debian-based server with root or sudo access
- A registered domain name with control over DNS records
- At least 2 GB of RAM and 20 GB of disk space
- A static public IP address, or a relay service for outbound delivery
- Ports 25, 465, 587, 993, and 995 open in the firewall
- A valid TLS certificate, either self-signed for lab use or issued by a public CA

## Installation

The following steps assume a fresh Debian installation. Adjust package versions and paths to match your environment.

### 1. System preparation

```
sudo apt update && sudo apt upgrade -y
sudo apt install -y exim4 dovecot-imapd dovecot-pop3d dovecot-lmtpd \
    mariadb-server roundcube roundcube-mysql spamassassin clamav \
    openssl certbot
```
2. Hostname and DNS
Set the server's fully qualified domain name:

```
sudo hostnamectl set-hostname mail.example.com
Configure the following DNS records at your registrar or DNS provider:

Type	Name	Value
A	mail.example.com	[server public IP]
MX	example.com	mail.example.com (priority 10)
TXT	example.com	v=spf1 mx -all
TXT	_dmarc.example.com	v=DMARC1; p=quarantine; rua=mailto:postmaster@example.com
TXT	default._domainkey.example.com	[DKIM public key, generated in step 6]
```
3. Database setup
Create the database and a dedicated user for virtual mail management:

``` sql
CREATE DATABASE mailserver CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'mailuser'@'localhost' IDENTIFIED BY 'strong-password-here';
GRANT ALL PRIVILEGES ON mailserver.* TO 'mailuser'@'localhost';
FLUSH PRIVILEGES;
Import the schema files from this repository:
```
```bash
mysql -u mailuser -p mailserver < sql/schema.sql
mysql -u mailuser -p mailserver < sql/seed-domains.sql
```
4. Exim configuration
Copy the configuration templates into place:

```bash
sudo cp config/exim/exim4.conf.template /etc/exim4/exim4.conf.template
sudo cp config/exim/conf.d/* /etc/exim4/conf.d/
sudo update-exim4.conf
sudo systemctl restart exim4
```
The templates configure Exim to:

Authenticate users against Dovecot SASL

Deliver local mail through Dovecot LMTP

Apply SpamAssassin and ClamAV scanning to inbound messages

Sign outbound mail with DKIM

5. Dovecot configuration
```bash
sudo cp config/dovecot/*.conf /etc/dovecot/conf.d/
sudo systemctl restart dovecot
```
The configuration enables:

Maildir storage under /var/mail/vhosts/

SQL-backed user and password lookups

SASL authentication for Exim

LMTP delivery socket for local messages

TLS enforcement on all client connections

6. DKIM key generation
```bash
sudo mkdir -p /etc/opendkim/keys/example.com
sudo opendkim-genkey -b 2048 -d example.com -s default \
    -D /etc/opendkim/keys/example.com
sudo chown -R opendkim:opendkim /etc/opendkim
```
Publish the contents of the generated .txt file as a TXT record at default._domainkey.example.com.

7. Roundcube configuration
```bash
sudo cp config/roundcube/config.inc.php /etc/roundcube/
sudo cp config/roundcube/plugins/* /var/lib/roundcube/plugins/
Update the database credentials and the IMAP host in config.inc.php to match your environment, then point your web server to /var/lib/roundcube/public_html.
```
## Configuration
Virtual users and domains
Add a new domain:

```sql
INSERT INTO domains (domain, active) VALUES ('example.com', 1);
Add a mailbox:
```
```bash
./scripts/add-mailbox.sh user@example.com 'strong-password'
The script hashes the password with Dovecot's doveadm pw utility and inserts the record into the users table.
```
Aliases and forwarding
Aliases are stored in the aliases table and support both local delivery and remote forwarding:

```sql
INSERT INTO aliases (source, destination, domain_id)
VALUES ('sales@example.com', 'alice@example.com', 1);
```
Quotas
Per-user quotas are defined in the users table and enforced by Dovecot:

```sql
UPDATE users SET quota = 2048 WHERE email = 'alice@example.com';
```
The value is expressed in megabytes.

## Email Automation
The suite includes scripts that generate and send automated email messages. These are useful for administrative notifications, monitoring alerts, and scheduled reports.

Sending mail from the command line

```bash
echo "Server backup completed successfully." | \
    mail -s "Backup Notification" admin@example.com
```
Automated report script
```scripts/send-report.sh``` generates a daily summary of mailbox usage and sends it to the administrator:

```bash
#!/bin/bash
REPORT=$(doveadm quota get -A | awk '{print $1, $2, $4, $5}')
echo "$REPORT" | mail -s "Daily Mailbox Quota Report" admin@example.com
Schedule it with cron:
```
```cron
0 6 * * * /usr/local/bin/send-report.sh
Triggered notifications
Exim routers can be configured to trigger scripted workflows on specific conditions, such as a message arriving at a monitoring address:
```
```text
trigger_notification:
  driver = accept
  condition = ${if eq{$local_part}{alerts}{yes}{no}}
  transport = notification_pipe
The corresponding transport pipes the message to a script for processing:
```
```text
notification_pipe:
  driver = pipe
  command = /usr/local/bin/handle-alert.sh
  user = mail
  group = mail
```
## Security Hardening
The following measures are implemented in this configuration and should be reviewed before deployment:

TLS is enforced on all submission and retrieval ports

Dovecot SSL settings use only TLS 1.2 and above, with strong cipher suites

SMTP authentication is required for relay; open relay is disabled

Fail2ban rules are provided in config/fail2ban/ to block brute-force attempts on SMTP, IMAP, and Roundcube

DKIM, SPF, and DMARC records align outbound mail with receiving servers' policies

ClamAV and SpamAssassin scan inbound messages before delivery

Mailbox directories use restrictive permissions: owned by vmail:vmail, mode 0700

Roundcube sessions use secure cookies and HTTPS-only transport

System accounts have password authentication disabled where possible

## Testing and Verification
Send a local message
```bash
swaks --to alice@example.com --from test@example.com \
      --server 127.0.0.1 --auth LOGIN \
      --auth-user alice@example.com --auth-password 'password'
```
Verify SMTP TLS
```bash
openssl s_client -starttls smtp -connect mail.example.com:587
Verify IMAP TLS
```
```bash
openssl s_client -connect mail.example.com:993
```
Check SPF, DKIM, and DMARC alignment
Send a message to a service such as check-auth@verifier.port25.com and review the reply. The expected result is pass for all three checks.

Confirm delivery
```bash
sudo tail -f /var/log/exim4/mainlog
sudo doveadm mailbox status -u alice@example.com messages INBOX
```
## Troubleshooting
Messages are deferred with a connection timeout to remote hosts. Outbound port 25 may be blocked by your hosting provider. Configure a smart host relay in /etc/exim4/update-exim4.conf.conf or use an authenticated relay service.

Dovecot authentication fails for valid users. Confirm the SQL query in config/dovecot/dovecot-sql.conf.ext matches your schema column names. Run doveadm auth test alice@example.com to see the exact failure.

Roundcube shows a blank page after login. Check the PHP error log (/var/log/nginx/error.log or /var/log/apache2/error.log) and confirm the config.inc.php database credentials are correct.

Outbound mail lands in spam at major providers. Review SPF, DKIM, and DMARC records, confirm your server's PTR record matches its hostname, and check your IP against major blocklists.

ClamAV fails to start after installation. The signature database can take several minutes to download. Run sudo freshclam manually and wait for completion before starting the service.

## Project Structure
```text
enterprise-mail-server-suite/
├── config/
│   ├── exim/
│   │   ├── exim4.conf.template
│   │   └── conf.d/
│   ├── dovecot/
│   │   ├── 10-auth.conf
│   │   ├── 10-mail.conf
│   │   ├── 10-ssl.conf
│   │   └── dovecot-sql.conf.ext
│   ├── roundcube/
│   │   ├── config.inc.php
│   │   └── plugins/
│   └── fail2ban/
├── sql/
│   ├── schema.sql
│   └── seed-domains.sql
├── scripts/
│   ├── add-mailbox.sh
│   ├── send-report.sh
│   └── handle-alert.sh
├── LICENSE
└── README.md
```

## License
This project is licensed under the MIT License. See the LICENSE file for details.

## Author
Victor Mawira

Email: victor.mawira@strathmore.edu

GitHub: https://github.com/MawiraVictor

LinkedIn: https://www.linkedin.com/in/victor-mawira

Acknowledgments
Developed as part of coursework in computer networks and cybersecurity at Strathmore University. Configuration references drawn from the official Exim, Dovecot, and Roundcube documentation.
