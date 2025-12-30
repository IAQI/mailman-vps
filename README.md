Mailman 3 on Debian VPS
========================

## Introduction

[Mailman, the GNU Mailing List Manager](https://www.list.org/) is free software for managing electronic mail discussion and e-newsletter lists.

This repository provides automated deployment of Mailman 3 on a Debian VPS using Ansible. It's specifically configured to work with [Infomaniak Email Service](https://www.infomaniak.com/en/hosting/service-mail/) as the SMTP relay.

**Architecture**: Instead of configuring MX records and piping emails directly to Mailman, this setup uses fetchmail to poll an email account, making it suitable for environments where you don't control the mail server.

## Prerequisites

Before you begin, you'll need:

- **Debian VPS** (tested on Debian 11 Bullseye) with root or sudo access
- **Ansible** installed on your local machine
- **SSH key pair** for server access
- **Domain name** with DNS management access
- **Infomaniak Email Service** account (or similar IMAP/SMTP email service)

## Email Service Setup

### 1. Create Email Account

Create an email account at [Infomaniak Email Service](https://www.infomaniak.com/en/hosting/service-mail/) for your domain:

```
mailman@yourdomain.com
```

Note the password - you'll need it for the inventory configuration.

### 2. Configure Email Aliases

For each mailing list you plan to create (e.g., "mylist"), add these aliases to the mailman email account:

Required aliases for list "mylist":
- `mylist`
- `mylist-bounces`
- `mylist-confirm`
- `mylist-join`
- `mylist-leave`
- `mylist-owner`
- `mylist-request`
- `mylist-subscribe`
- `mylist-unsubscribe`

Also create an alias for:
- `postorius` (web interface administrative emails)

These aliases ensure all list-related emails are delivered to the mailman account.

## VPS Setup

### 1. Provision Your VPS

Obtain a Debian 11+ VPS from any hosting provider. Recommended minimum specs:
- 1 GB RAM
- 20 GB storage
- Public IPv4 address (IPv6 optional)

### 2. Configure DNS

Create DNS records pointing to your VPS:

```
A     list.yourdomain.com    → your.vps.ip.address
AAAA  list.yourdomain.com    → your:ipv6::address  (if available)
```

### 3. Set Up SSH Access

Ensure you can SSH into your VPS:

```bash
# Test SSH access
ssh debian@your.vps.ip.address

# Or generate a new SSH key pair if needed
ssh-keygen -t ed25519 -f ~/.ssh/mailman_vps
ssh-copy-id -i ~/.ssh/mailman_vps.pub debian@your.vps.ip.address
```

## Deployment

### 1. Create Inventory File

Create a file named `inventory` in the repository root:

```ini
mailman ansible_host=your.vps.ip.address

[all:vars]
ansible_python_interpreter=/usr/bin/python3
ansible_user=debian
ansible_ssh_private_key_file=~/.ssh/your_ssh_key
fqdn=list.yourdomain.com
mail_host=mail.infomaniak.com
mail_account=mailman@yourdomain.com
mail_password=your_email_password
mailman_user=admin
mailman_password=your_admin_password
mailman_domain=yourdomain.com
mailman_email=you@yourdomain.com
```

**Variable Explanations**:
- `ansible_host`: Your VPS IP address
- `ansible_ssh_private_key_file`: Path to your SSH private key
- `fqdn`: Full domain name for the Mailman web interface
- `mail_host`: SMTP/IMAP server (use `mail.infomaniak.com` for Infomaniak)
- `mail_account`: Email account created in step 1
- `mail_password`: Password for the email account
- `mailman_user`: Admin username for Mailman web interface
- `mailman_password`: Admin password for web interface
- `mailman_domain`: Your domain name
- `mailman_email`: Your personal email address (site owner)

### 2. Run the Playbook

Deploy Mailman to your VPS:

```bash
# Run with diff mode to see changes
ansible-playbook playbook.yml -D

# Or run a dry-run first to preview
ansible-playbook playbook.yml --check
```

The playbook will:
- Install Postfix, Mailman3, fetchmail, Apache2, and certbot
- Configure Postfix to relay through Infomaniak SMTP
- Set up fetchmail to poll your email account
- Configure Mailman3 and the web interface
- Obtain Let's Encrypt SSL certificate
- Configure Apache2 to serve the web interface

**Expected output**:
```
PLAY RECAP *****************************************************
mailman    : ok=19   changed=17   unreachable=0    failed=0
```

## Post-Installation

### 1. Access Web Interface

Navigate to your Mailman instance:

```
https://list.yourdomain.com
```

Log in with the `mailman_user` and `mailman_password` you configured in the inventory.

### 2. Create a Domain

1. In the web interface, go to **Domains** → **Add Domain**
2. Enter your domain (e.g., `yourdomain.com`)
3. Set mail host to `list.yourdomain.com`
4. Save

### 3. Create Your First List

1. Go to **Lists** → **Create New List**
2. Enter list name (e.g., `mylist`)
3. Choose the domain you created
4. Set initial owner email
5. Create the list

### 4. Configure List Settings

Recommended settings for each list:

**DMARC Mitigations** (Settings → DMARC Mitigations):
- Set "DMARC mitigation action" to **"Replace From: with list address"**
- This helps with email deliverability

**Message Options** (Settings → Alter Messages):
- Set "Reply goes to list" to **"Reply to list"**
- Enable **"First strip Reply-To"**
- This ensures replies go to the list, not the original sender

### 5. Test Email Flow

Send a test email to your list:

1. Subscribe yourself to the list
2. Send an email to `mylist@list.yourdomain.com`
3. Check that you receive the email back from the list

## Troubleshooting

### Issue: Packages Won't Install (bullseye-backports)

**Symptom**: Package installation fails with repository errors.

**Solution**: The playbook automatically removes the bullseye-backports repository before installation. If you encounter issues, manually run:

```bash
ssh debian@your.vps.ip.address
sudo sed -i '/bullseye-backports/d' /etc/apt/sources.list
sudo apt update
```

### Issue: Emails Not Being Received

**Symptom**: Emails sent to list addresses don't appear in Mailman.

**Solution**: Check fetchmail status and logs:

```bash
# On the VPS
sudo systemctl status fetchmail
sudo tail -f /var/log/mail.log

# Verify email account credentials
sudo cat /etc/fetchmailrc
```

Ensure:
- Email aliases are configured in Infomaniak
- Fetchmail credentials are correct
- Fetchmail service is running

### Issue: Emails Rejected with "Sender mismatch"

**Symptom**: Outbound emails fail with `550 5.7.1 Sender mismatch` error.

**Context**: Infomaniak SMTP relay validates that both the envelope sender AND the From header match the authenticated account. See `PROBLEM_ANALYSIS.md` for detailed investigation.

**Solution**: Configure Postfix to rewrite both envelope and From headers. On the VPS:

```bash
# Rewrite envelope sender (already in playbook)
echo '/@list\.yourdomain\.com$/    mailman@yourdomain.com' | sudo tee /etc/postfix/smtp_generic

# Rewrite From headers for all outgoing mail
echo '/^From:/i    REPLACE From: Mailman List <mailman@yourdomain.com>' | sudo tee /etc/postfix/smtp_header_checks

# Enable header checks in main.cf
sudo postconf -e 'smtp_header_checks = regexp:/etc/postfix/smtp_header_checks'

# Reload Postfix
sudo postfix reload
```

**Trade-off**: All outgoing emails will show `From: mailman@yourdomain.com` instead of the original sender. Recipients won't see who sent the message in the From field. Subscription confirmations require users to send a new email (not reply) - custom templates are provided.

**Debugging**:
```bash
sudo tail -f /var/log/mail.log
sudo postqueue -p
echo "test" | sendmail -f mailman@yourdomain.com you@example.com
```

### Issue: Web Interface Not Accessible

**Symptom**: Cannot access `https://list.yourdomain.com`

**Solution**: Check Apache and SSL certificate:

```bash
# On the VPS
sudo systemctl status apache2
sudo certbot certificates

# Check Apache logs
sudo tail -f /var/log/apache2/mailman3-error.log
```

Ensure:
- DNS records point to your VPS
- Port 443 is open in firewall
- SSL certificate was successfully obtained

### General Debugging Commands

```bash
# Check all service statuses
sudo systemctl status postfix
sudo systemctl status mailman3
sudo systemctl status mailman3-web
sudo systemctl status fetchmail
sudo systemctl status apache2

# View logs
sudo tail -f /var/log/mail.log              # Postfix/fetchmail
sudo tail -f /var/log/mailman3/mailman.log  # Mailman core
sudo tail -f /var/log/mailman3/smtp.log     # Mailman SMTP
sudo tail -f /var/log/apache2/mailman3-error.log  # Web interface

# Reload Postfix after config changes
sudo postfix check  # Verify syntax first
sudo postfix reload
```

## Notes

- **HTTPS**: The playbook automatically obtains a Let's Encrypt certificate using certbot
- **Email Flow**: Incoming emails are fetched via IMAP every 5 minutes by fetchmail
- **Outbound Relay**: All outbound emails go through Infomaniak SMTP with SASL authentication
- **Sender Rewriting**: Envelope senders are rewritten to match the authorized email account

## Credits

This project is based on [mailman-cloud](https://github.com/reneluria/mailman-cloud) by René Luria, originally designed for Infomaniak Public Cloud. This version has been adapted to work with any generic Debian VPS.
