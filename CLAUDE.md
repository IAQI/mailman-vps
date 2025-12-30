# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This repository automates the deployment of GNU Mailman 3 (mailing list manager) on a Debian VPS using Ansible. Originally designed for Infomaniak Public Cloud with Terraform, the project has been adapted for generic VPS deployment.

**Key Architecture**: Uses fetchmail to poll an Infomaniak email account rather than traditional MX-based delivery. All outbound emails relay through Infomaniak SMTP servers.

## Current Status

**ACTIVE ISSUE**: Infomaniak SMTP relay rejects list messages with "550 5.7.1 Sender mismatch" error. See `PROBLEM_ANALYSIS.md` for detailed investigation.

**Working Components**:
- Email receiving via fetchmail
- Mailman core processing
- Web interface (HTTPS with Let's Encrypt)
- Envelope sender rewriting to mailman@iaqi.org

**Known Problem**: Infomaniak validates both envelope sender AND message headers for consistency. List messages fail because:
- Envelope sender: mailman@iaqi.org (correct, rewritten by smtp_generic_maps)
- From header: original sender email (e.g., huebli@gmail.com)
- Result: Sender mismatch rejection

## Common Commands

### Deployment

```bash
# Run the full playbook
ansible-playbook playbook.yml -D

# Dry-run to preview changes
ansible-playbook playbook.yml --check

# Verbose output for debugging
ansible-playbook playbook.yml -vv

# Syntax check
ansible-playbook playbook.yml --syntax-check
```

### Debugging Email Issues

```bash
# On the VPS server:

# Check Postfix queue
sudo postqueue -p

# Examine message headers in queue
sudo postcat -qh <MESSAGE_ID>

# View mail logs
sudo tail -f /var/log/mail.log

# Check service status
sudo systemctl status postfix
sudo systemctl status mailman3
sudo systemctl status fetchmail

# Test direct send (bypasses Mailman)
echo "test" | sendmail -f mailman@iaqi.org recipient@example.com

# Reload Postfix after config changes
sudo postfix reload

# Check Postfix configuration
sudo postconf | grep smtp_generic
sudo postconf | grep smtp_header
```

### Mailman Administration

```bash
# Access the web interface at https://<your-domain>
# Login with credentials from inventory file (mailman_user/mailman_password)

# Mailman command-line
sudo -u list mailman shell

# View Mailman logs
sudo tail -f /var/log/mailman3/mailman.log
sudo tail -f /var/log/mailman3/smtp.log
```

## Architecture

### Email Flow

```
Incoming:
  Sender → Infomaniak Email Service (IMAP)
         → Fetchmail (polls every 5 min)
         → Postfix (local delivery)
         → Mailman3 (LMTP)

Outgoing:
  Mailman3 → Postfix
           → smtp_generic_maps (rewrites envelope sender)
           → Infomaniak SMTP relay (validates sender)
           → Recipients
```

### Component Stack

- **Postfix**: MTA configured as smarthost relay
- **Mailman3**: Core mailing list engine
- **Mailman3-web**: Django-based web interface (Postorius/HyperKitty)
- **Fetchmail**: Polls Infomaniak IMAP for incoming mail
- **Apache2**: Web server with mod_proxy_uwsgi
- **Certbot**: Manages Let's Encrypt SSL certificates

## Configuration Files

### Inventory Variables (inventory file)

Required variables:
- `fqdn`: Fully qualified domain name (e.g., list.yourdomain.com)
- `mail_host`: Infomaniak SMTP server (mail.infomaniak.com)
- `mail_account`: Mailman email account (e.g., mailman@yourdomain)
- `mail_password`: Email account password
- `mailman_user`: Web interface admin username
- `mailman_password`: Web interface admin password
- `mailman_domain`: Domain for list addresses
- `mailman_email`: Site owner email address

### Postfix Configuration

Key settings in `/etc/postfix/main.cf` (managed by Ansible):

```
# SASL authentication for Infomaniak relay
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_sasl_auth_enable = yes
smtp_tls_security_level = encrypt

# Mailman LMTP transport
owner_request_special = no
transport_maps = hash:/var/lib/mailman3/data/postfix_lmtp
local_recipient_maps = proxy:unix:passwd.byname $alias_maps hash:/var/lib/mailman3/data/postfix_lmtp
relay_domains = hash:/var/lib/mailman3/data/postfix_domains

# Envelope sender rewriting (CURRENT STATE)
smtp_generic_maps = regexp:/etc/postfix/smtp_generic
```

The `/etc/postfix/smtp_generic` file contains:
```
/@list\.iaqi\.org$/    mailman@iaqi.org
```

### Templates

- `templates/fetchmail/fetchmailrc.j2`: Fetchmail IMAP configuration with idle/fetchall
- `templates/postfix/sasl_passwd.j2`: SASL credentials for SMTP relay

## Important Implementation Details

### Sender Rewriting Strategy

The playbook configures sender rewriting via `smtp_generic_maps` which rewrites the envelope sender (MAIL FROM) for all emails from list addresses to the authorized mailman account. This is required because Infomaniak SMTP relay only accepts mail from authenticated account addresses.

**Current limitation**: Only rewrites envelope, not message headers. See PROBLEM_ANALYSIS.md for investigation into header validation issues.

### Django Site Domain

The playbook directly modifies the SQLite database to set the correct domain for the web interface:
```bash
sqlite3 /var/lib/mailman3/web/mailman3web.db \
  'update django_site set name = "...", domain = "..." where id = 1;'
```

This ensures generated URLs in emails and redirects are correct.

### Bullseye-Backports Issue

The playbook explicitly removes bullseye-backports repository before package installation to avoid conflicts with Mailman3 packages. This is a workaround for Debian package repository issues.

### Email Aliases Required

For each mailing list (e.g., "mylist"), configure these aliases in Infomaniak email service to point to the mailman account:
- mylist
- mylist-bounces
- mylist-confirm
- mylist-join
- mylist-leave
- mylist-owner
- mylist-request
- mylist-subscribe
- mylist-unsubscribe

Also create an alias for "postorius" (web interface).

## Migration Notes (from Public Cloud to VPS)

This project originally used Terraform to provision infrastructure on Infomaniak Public Cloud. The Terraform code remains in the `terraform/` directory but is no longer the primary deployment method.

**Current approach**: Deploy to any Debian VPS with Ansible only.

**TODO items** from PLAN.md:
- Clean up sender rewriting configuration in playbook.yml
- Archive or remove Terraform directory
- Update .gitignore for VPS deployment
- Create inventory.example template

## Troubleshooting

### Sender Mismatch Errors

If you see `550 5.7.1 Sender mismatch` in `/var/log/mail.log`:

1. **Verify envelope rewriting is working**:
   ```bash
   grep "from=<mailman@" /var/log/mail.log
   # Should show mailman@iaqi.org, not list addresses
   ```

2. **Check actual message headers**:
   ```bash
   sudo postqueue -p  # Get message ID
   sudo postcat -qh <MESSAGE_ID>
   # Look at From: header - does it match envelope sender?
   ```

3. **Test direct send** (bypasses Mailman):
   ```bash
   echo "test" | sendmail -f mailman@iaqi.org your-email@example.com
   # This should succeed if SASL auth is working
   ```

See `PROBLEM_ANALYSIS.md` for comprehensive investigation of this issue.

### Service Health Checks

```bash
sudo systemctl status postfix
sudo systemctl status mailman3
sudo systemctl status mailman3-web
sudo systemctl status fetchmail
sudo systemctl status apache2
```

### Log Locations

- Postfix: `/var/log/mail.log`
- Mailman: `/var/log/mailman3/mailman.log`, `/var/log/mailman3/smtp.log`
- Apache: `/var/log/apache2/mailman3-error.log`, `/var/log/apache2/mailman3-access.log`
- Fetchmail: Check syslog or `/var/log/mail.log`

## Development Workflow

When modifying the playbook:

1. Make changes to `playbook.yml` or templates
2. Test with `--check` first: `ansible-playbook playbook.yml --check`
3. Apply with diff mode: `ansible-playbook playbook.yml -D`
4. Monitor logs on the server during deployment
5. Test email flow after changes

**For Postfix config changes specifically**:
- Changes to main.cf require: `sudo postfix reload`
- Changes to map files (smtp_generic, sasl_passwd) require: `sudo postmap <file>` then `sudo postfix reload`
- Always check syntax first: `sudo postfix check`
