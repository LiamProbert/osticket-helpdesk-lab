# osTicket Helpdesk Lab
 
A fully configured helpdesk ticketing system deployed from scratch on a Fedora Server VM in my homelab. Integrated with Active Directory via LDAP so domain users can authenticate with their existing credentials. Built to demonstrate real 1st line helpdesk skills beyond just terminal work.
 
## Environment
 
| Component | Details |
|---|---|
| Ticketing System | osTicket v1.18.2 |
| OS | Fedora Server 44 |
| Web Server | Apache (httpd) |
| Database | MariaDB 11.8 |
| Language | PHP 8.5.8 |
| Hypervisor | KVM / virt-manager |
| Host Machine | Lenovo ThinkPad T480, Fedora Linux |
| Domain Controller | Windows Server 2022 (DC01) |
| Domain | lab.local |
 
## What This Project Covers
 
### Infrastructure
 
- Spun up a Fedora Server 44 VM on KVM
- Installed and configured the full LAMP stack (Apache, MariaDB, PHP)
- Deployed osTicket via the web installer
- Dealt with SELinux blocking Apache write access during setup
### Configuration
 
- Created departments aligned with Active Directory OUs (IT Support, HR, Facilities)
- Built SLA plans reflecting a real priority structure (P1 Critical through P4 Low)
- Configured help topics with automatic department and priority routing
- Wrote canned responses for the most common ticket types
### LDAP Integration
 
- Installed the LDAP Authentication and Lookup plugin manually
- Resolved a missing Net_LDAP2 PEAR dependency that prevented the plugin from initialising
- Diagnosed and fixed SELinux blocking PHP-FPM from making outbound LDAP connections
- Connected osTicket to DC01 so AD users authenticate with domain credentials
## Departments
 
| Department | Notes |
|---|---|
| IT Support | Default. Handles the bulk of tickets |
| HR | Onboarding, offboarding, HR requests |
| Facilities | Hardware, printers, building access |
 
## SLA Plans
 
| Plan | Grace Period | Use Case |
|---|---|---|
| P1 Critical | 1 hour | System down, team-wide impact |
| P2 High | 4 hours | Single user unable to work |
| P3 Medium | 8 hours | Non-urgent requests |
| P4 Low | 24 hours | General enquiries |
 
## Help Topics
 
| Topic | Department | Priority | SLA |
|---|---|---|---|
| Password Reset | IT Support | Normal | P2 High |
| Hardware Fault | IT Support | High | P2 High |
| Software Install Request | IT Support | Normal | P3 Medium |
| New Starter | HR | Normal | P3 Medium |
| General Enquiry | IT Support | Low | P4 Low |
 
## Canned Responses
 
Pre-written replies agents can fire off for common ticket types. Each uses placeholders (e.g. `[name]`, `[password]`) that the agent fills in before sending.
 
| Title | Department | Example |
|---|---|---|
| Password Reset - Confirmation | IT Support | *"Hi [name], your password has been reset. Your temporary password is [password]. You'll be prompted to change it on first login. If you have any issues, reply to this ticket."* |
| Hardware Fault - Logged and Assigned | IT Support | *"Hi [name], your hardware fault has been logged and assigned to a technician. We aim to respond within [SLA]. Please don't attempt any repairs yourself."* |
| Software Install Request - Request Received | IT Support | *"Hi [name], your request to install [software] has been received. We'll check licensing and compatibility before proceeding. Expected turnaround: [SLA]."* |
| New Starter - Account Setup Confirmation | HR | *"Hi [name], the account for [new starter name] has been created. Login credentials have been sent to their manager. Start date: [date]."* |
| General Enquiry - Response | IT Support | *"Hi [name], thanks for getting in touch. [response]. If this doesn't resolve your query, reply to this ticket and we'll follow up."* |
| Ticket Resolved - Closing | IT Support | *"Hi [name], this ticket has been resolved. If the issue comes back or you need further help, reply to this ticket within 48 hours and it will reopen automatically."* |
 
## LDAP Integration
 
The goal was to let Active Directory users log into osTicket with their domain credentials instead of managing a separate osTicket password. The plugin is not bundled with osTicket and had to be installed manually, along with its Net_LDAP2 dependency.
 
Once installed, the plugin failed to connect to DC01. The configuration was correct. A manual `ldapwhoami` test from the terminal confirmed DC01 was reachable and accepting binds. A packet capture showed zero packets leaving the server when osTicket attempted the connection.
 
The cause was SELinux. PHP-FPM runs in a restricted security context (`httpd_t`) that does not have permission to make outbound LDAP connections by default. The audit log confirmed it:
 
```
avc: denied { name_connect } for comm="php-fpm" dest=389
```
 
Fix:
 
```bash
sudo setsebool -P httpd_can_connect_ldap on
```
 
After restarting Apache and PHP-FPM, the plugin connected and domain users could authenticate against DC01. Full troubleshooting walkthrough is in [`osticket-config.md`](osticket-config.md).
 
## SELinux Issues Encountered
 
This is the second project where Fedora's default SELinux policy required explicit configuration. Worth noting for anyone deploying osTicket on Fedora or RHEL-based systems.
 
| Issue | Cause | Fix |
|---|---|---|
| Apache couldn't write to ost-config.php | Wrong SELinux file context | `chcon -t httpd_sys_rw_content_t` |
| PHP-FPM couldn't connect to LDAP | `httpd_t` not allowed outbound LDAP | `setsebool -P httpd_can_connect_ldap on` |
 
## Related Projects
 
- [Helpdesk PowerShell Toolkit](https://github.com/LiamProbert/helpdesk-powershell-toolkit) - AD automation scripts used alongside this project
- [Active Directory Homelab](https://github.com/LiamProbert/active-directory-homelab) - The DC01 setup that LDAP connects to
 
