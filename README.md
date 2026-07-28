# osTicket Helpdesk Lab

A fully configured helpdesk ticketing system deployed from scratch on a Fedora Server VM in my homelab. Integrated with Active Directory via LDAP so domain users can authenticate with their existing credentials. Built as a portfolio piece to demonstrate real 1st line helpdesk skills beyond just terminal work.

## Environment

| Component         | Details                            |
| ----------------- | ---------------------------------- |
| Ticketing System  | osTicket v1.18.2                   |
| OS                | Fedora Server 44                   |
| Web Server        | Apache (httpd)                     |
| Database          | MariaDB 11.8                       |
| Language          | PHP 8.5.8                          |
| Hypervisor        | KVM / virt-manager                 |
| Host Machine      | Lenovo ThinkPad T480, Fedora Linux |
| Domain Controller | Windows Server 2022 (DC01)         |
| Domain            | lab.local                          |

## What This Project Covers

**Phase 1 - Infrastructure and Configuration**

- Spun up a Fedora Server 44 VM on KVM
- Installed and configured the full LAMP stack
- Deployed osTicket via the web installer
- Dealt with SELinux blocking Apache write access during setup
- Created departments aligned with Active Directory OUs (IT Support, HR, Facilities)
- Built SLA plans reflecting a real priority structure (P1 Critical through P4 Low)
- Configured help topics with automatic department and priority routing
- Wrote canned responses for the most common ticket types
- Installed the LDAP plugin and connected osTicket to DC01 so AD users authenticate with domain credentials

**Phase 2 - Ticket Resolution Library**

- Tested LDAP client authentication by logging into the user portal as an AD user
- Created and resolved three realistic helpdesk tickets through the full lifecycle
- Demonstrated ticket submission, department routing, internal notes, customer replies, approval workflows, and resolution
- Documented mistakes made during the process and what I learned from them
- Linked the ticket work back to the [Helpdesk PowerShell Toolkit](https://github.com/LiamProbert/helpdesk-powershell-toolkit) (New-User.ps1, Disable-Leaver.ps1)

## Departments

| Department | Notes                                |
| ---------- | ------------------------------------ |
| IT Support | Default. Handles the bulk of tickets |
| HR         | Onboarding, offboarding, HR requests |
| Facilities | Hardware, printers, building access  |

## SLA Plans

| Plan        | Grace Period | Use Case                      |
| ----------- | ------------ | ----------------------------- |
| P1 Critical | 1 hour       | System down, team-wide impact |
| P2 High     | 4 hours      | Single user unable to work    |
| P3 Medium   | 8 hours      | Non-urgent requests           |
| P4 Low      | 24 hours     | General enquiries             |

## Help Topics

| Topic                    | Department | Priority | SLA       |
| ------------------------ | ---------- | -------- | --------- |
| Password Reset           | IT Support | Normal   | P2 High   |
| Hardware Fault           | IT Support | High     | P2 High   |
| Software Install Request | IT Support | Normal   | P3 Medium |
| New Starter              | IT Support | Normal   | P3 Medium |
| General Enquiry          | IT Support | Low      | P4 Low    |

## Canned Responses

| Title                                    | Department |
| ---------------------------------------- | ---------- |
| Password Reset - Confirmation            | IT Support |
| Hardware Fault - Logged and Assigned      | IT Support |
| Software Install - Request Received      | IT Support |
| New Starter - Account Setup Confirmation | HR         |
| General Enquiry - Response               | IT Support |
| Ticket Resolved - Closing                | IT Support |

## Ticket Resolution Library

Three tickets were created and resolved during Phase 2, each worked through the full lifecycle from user submission to agent resolution.

| Ticket                     | Help Topic               | SLA       | Notes                                                       |
| -------------------------- | ------------------------ | --------- | ----------------------------------------------------------- |
| New Starter Account Setup  | New Starter              | P3 Medium | Exposed a department routing mistake. Referenced New-User.ps1 |
| AD Password Reset          | Password Reset           | P2 High   | Documented a security mistake with temporary credentials      |
| Outlook Installation       | Software Install Request | P3 Medium | Full clean workflow with approval stage                       |

The progression matters more than the individual tickets. The first two had mistakes (wrong department routing, missed internal note, password in a ticket thread). By the third ticket I had the full workflow down properly.

Full documentation of each ticket, including the mistakes and what I learned: [osticket-live-practice.md](osticket-live-practice.md)

## LDAP Integration

The goal was to let Active Directory users log into osTicket with their domain credentials instead of managing a separate osTicket password.

The LDAP plugin is not bundled with osTicket and needs to be installed manually. It also requires the Net_LDAP2 PEAR library which is not installed by default on Fedora. Both needed to be sorted before the plugin would initialise at all.

Once the dependency issues were resolved, the plugin still failed to connect to the domain controller. The error was:

```
Unable to connect any listed LDAP servers
```

The configuration was correct. A manual `ldapwhoami` test from the terminal confirmed DC01 was reachable, the credentials worked, and LDAP was accepting binds. A packet capture showed zero packets leaving the server when osTicket tried to connect.

The actual cause was SELinux. PHP-FPM runs in a restricted security context (`httpd_t`) that does not have permission to make outbound LDAP connections by default. The audit log confirmed it:

```
avc: denied { name_connect } for comm="php-fpm" dest=389
```

Fix:

```
sudo setsebool -P httpd_can_connect_ldap on
```

After restarting Apache and PHP-FPM, the plugin connected successfully. Domain users can now authenticate against DC01 using their existing AD credentials.

## Key SELinux Issues Encountered

| Issue                                   | Cause                              | Fix                                      |
| --------------------------------------- | ---------------------------------- | ---------------------------------------- |
| Apache couldn't write to ost-config.php | Wrong SELinux file context         | `chcon -t httpd_sys_rw_content_t`        |
| PHP-FPM couldn't connect to LDAP        | httpd_t not allowed outbound LDAP  | `setsebool -P httpd_can_connect_ldap on` |

## Screenshots

### Configuration

| Screenshot | Description |
| ---------- | ----------- |
| ![Departments](screenshots/departments.png) | IT Support (default), HR, Facilities |
| ![SLA Plans](screenshots/sla-plans.png) | P1 Critical through P4 Low |
| ![Help Topics](screenshots/help-topics.png) | Five help topics with auto-routing |
| ![LDAP Config](screenshots/ldap-plugin-configuration.png) | LDAP plugin connected to DC01 |
| ![Agent Login](screenshots/lprobert-ldap-agent-login.png) | lprobert authenticated via LDAP |

### Tickets

| Screenshot | Description |
| ---------- | ----------- |
| ![New Starter Submission](screenshots/new-starter-ticket-submission.png) | Toby submitting a new starter request |
| ![New Starter Resolved](screenshots/new-starter-ticket-resolved.png) | Resolved after department transfer |
| ![Password Reset Submission](screenshots/password-reset-ticket-submission.png) | Password reset request from Toby |
| ![Password Reset Resolved](screenshots/password-reset-ticket-resolved.png) | Resolved with P2 High SLA |
| ![Software Install Submission](screenshots/software-install-ticket-submission.png) | Outlook install request |
| ![Software Install Resolved](screenshots/software-install-ticket-resolved.png) | Full thread: internal notes, replies, approval, resolution |

## Related Projects

- [Helpdesk PowerShell Toolkit](https://github.com/LiamProbert/helpdesk-powershell-toolkit) - The AD automation scripts used alongside this project
- [Active Directory Homelab](https://github.com/LiamProbert/active-directory-homelab) - The DC01 setup that LDAP connects to
