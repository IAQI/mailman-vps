# mailman-vps Adaptation Plan

## Complete Plan: Adapt from Public Cloud to VPS Deployment

### Phase 1: Clean Up Repository Structure
- [ ] Remove Terraform directory (or move to `archive/` for reference)
- [ ] Update `.gitignore` to exclude deployment-specific files
  - [ ] Add `inventory` (user-specific)
  - [ ] Add `id_tf_keypair*` (SSH keys)
  - [ ] Add `*.openrc.txt` (OpenStack credentials - no longer needed)
  - [ ] Add `PCU-*` (Infomaniak specific files)
- [ ] Ensure working files stay out of repo

### Phase 2: Update Ansible Playbook
- [x] OpenStack environment checks are commented out
- [x] Bullseye-backports removal is in place
- [x] Fetchmail domain uses variable
- [x] SMTP sender rewriting configured
- [ ] Clean up the sender rewriting block in playbook.yml
  - Remove any references to old `sender_canonical_maps`
  - Ensure only `smtp_generic_maps` is used
  - Verify regex pattern for Mailman administrative addresses

### Phase 3: Update Documentation (README.md)
Replace current README with VPS-focused instructions:

#### Section 1: Introduction
- Brief description of what this deploys
- Mention it's configured for Infomaniak Email Service

#### Section 2: Prerequisites
- Debian 11+ VPS with root/sudo access
- Ansible installed locally
- SSH key pair for server access
- Domain name with DNS access
- Infomaniak Email Service account

#### Section 3: Email Service Setup
- Create email account at Infomaniak
- Configure email aliases for list addresses
- Document required aliases pattern

#### Section 4: VPS Setup
- Provision Debian VPS at your hosting provider
- Configure DNS records (A/AAAA)
- Set up SSH access
- Generate SSH keypair

#### Section 5: Deployment
- Create inventory file from example
- Configure variables
- Run Ansible playbook
- Verify installation

#### Section 6: Post-Installation
- Access web interface
- Create domain and lists
- Configure list settings (DMARC, Reply-To)

#### Section 7: Troubleshooting
- Document the fixes we implemented:
  - Bullseye-backports repository issue
  - Fetchmail domain configuration
  - SMTP sender rewriting for Infomaniak
  - Debugging commands

### Phase 4: Create Supporting Files

#### File 1: `inventory.example`
Template inventory file with:
```ini
mailman ansible_host=YOUR_VPS_IP

[all:vars]
ansible_python_interpreter=/usr/bin/python3
ansible_user=debian
ansible_ssh_private_key_file=~/.ssh/id_rsa  # or your key path
fqdn=mailman.<yourdomain>
mail_host=mail.infomaniak.com
mail_account=mailman@<yourdomain>
mail_password=<email_password>
mailman_user=mailman
mailman_password=<interface_password>
mailman_domain=<yourdomain>
mailman_email=<your_email>
```

#### File 2: `.gitignore`
```
# Deployment-specific files
inventory
id_tf_keypair
id_tf_keypair.pub
*.openrc.txt
PCU-*

# Terraform (archived)
terraform/.terraform/
terraform/*.tfstate
terraform/*.tfstate.backup
terraform/.terraform.lock.hcl

# macOS
.DS_Store

# Editor
.vscode/
.idea/
*.swp
*.swo
```

#### File 3: `DEPLOYMENT.md` (optional)
Detailed step-by-step deployment guide with:
- Screenshots/examples
- Common issues and solutions
- Infomaniak-specific notes

### Phase 5: Test & Document

#### Testing
- [ ] Verify playbook runs cleanly on fresh Debian 11 VPS
- [ ] Test email flow: incoming → Mailman → outgoing
- [ ] Verify sender rewriting works correctly
- [ ] Check all list functionality

#### Documentation
- [ ] Add section about Infomaniak SMTP sender rewriting requirement
- [ ] Document the regex pattern for administrative addresses
- [ ] Include example debugging commands
- [ ] Add note about monitoring logs

### Phase 6: Optional Improvements
- [ ] Add systemd service health checks
- [ ] Create backup/restore documentation
- [ ] Add Let's Encrypt/HTTPS setup instructions
- [ ] Document migration from Public Cloud to VPS
- [ ] Add monitoring/alerting suggestions

## Implementation Order
1. Start with `.gitignore` (protects sensitive files immediately)
2. Create `inventory.example` (gives users a template)
3. Update README.md (main documentation)
4. Clean up playbook.yml (finalize automation)
5. Test on fresh VPS
6. Create DEPLOYMENT.md if needed
7. Archive/remove terraform directory

## Notes
- Keep original repo reference for upstream updates
- Focus on Infomaniak-specific configurations
- Emphasize VPS simplicity vs cloud complexity
- Document cost savings motivation
