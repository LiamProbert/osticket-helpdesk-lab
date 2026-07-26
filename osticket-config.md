# osTicket Setup Walkthrough
 
Full build documentation for the osTicket helpdesk lab, from blank VM to working ticketing system with Active Directory authentication.
 
## Project Scope
 
This project has two phases.
 
**Phase 1** is infrastructure and configuration: spin up a VM, install the LAMP stack, deploy osTicket, and configure it to mirror a realistic helpdesk environment with departments, SLA plans, help topics, auto-routing, and canned responses. LDAP authentication tied to DC01 is also part of this phase.
 
**Phase 2** is the ticket resolution library. Once the system is live, realistic helpdesk tickets are created, triaged, resolved, and closed to demonstrate the full ticket lifecycle from a 1st line support perspective.
 
## Why osTicket
 
After finishing the [PowerShell Helpdesk Toolkit](https://github.com/LiamProbert/helpdesk-powershell-toolkit), I wanted a second portfolio project covering a different skill set. The first project was entirely terminal based. This one needed to be more visual and cover something I would realistically interact with in a 1st line support role.
 
A helpdesk ticketing system was the obvious choice. The specific tool does not matter at entry level. The concepts transfer across ServiceNow, Zendesk, Freshdesk, or whatever an employer uses. What matters is deploying and configuring one from scratch and understanding the full ticket lifecycle.
 
I chose osTicket over GLPI. GLPI is a heavier asset management suite where ticketing feels like an afterthought. osTicket is simpler, more focused on pure helpdesk ticketing, and closer to what I would actually be using day to day.
 
## VM and LAMP Stack Setup
 
The VM was created using virt-manager (KVM) on my Fedora ThinkPad, the same host running DC01.
 
**VM Specs:**
- 2048 MB RAM, 1 vCPU, 20 GB storage (qcow2)
- Fedora Server 44 (x86_64), headless (no desktop environment)
- Hostname: `osticket.lab.local`
- IP: `192.168.122.80` (same virtual network as DC01 at `192.168.122.10`)
The LAMP stack (Apache, MariaDB, PHP) was installed with a single `dnf install` command. I included `php-ldap` upfront for the DC01 integration. `php-imap` was not available in the Fedora 44 repos but is only needed for email piping, which this setup does not use.
 
MariaDB was hardened using `mariadb-secure-installation` (unix_socket auth, root password set, anonymous users removed, remote root login disabled, test database removed). A dedicated `osticket` database and user were created for the application.
 
## osTicket Deployment
 
Downloaded osTicket v1.18.2 from GitHub, extracted it into Apache's web directory at `/var/www/html/osticket`, and ran the web installer.
 
### SELinux: Apache Write Access
 
The first SELinux issue hit immediately. Apache could not write to the config file during installation, even with correct file permissions. The SELinux file context needed to be updated:
 
```bash
sudo chcon -R -t httpd_sys_rw_content_t /var/www/html/osticket/upload/include/ost-config.php
```
 
After the installer finished, the setup directory was removed (security risk) and the config file was locked down to `0644`.
 
### Access Points
 
- Staff control panel: `http://192.168.122.80/osticket/upload/scp`
- User facing portal: `http://192.168.122.80/osticket/upload/`
## Phase 1 Configuration
 
### Departments
 
osTicket ships with three defaults (Support, Sales, Maintenance). I replaced them to reflect a realistic office environment aligned with the OUs already in Active Directory on DC01.
 
| Department | Type | Notes |
|---|---|---|
| IT Support | Public | Default department, where unrouted tickets land |
| HR | Public | Onboarding, offboarding, HR requests |
| Facilities | Public | Hardware, printers, building access |
 
IT Support as the default is standard practice. If a ticket does not match any routing rule, it goes to the biggest team first.
 
### SLA Plans
 
SLA (Service Level Agreement) defines how quickly a ticket needs a response based on its priority. Four plans covering a standard priority structure:
 
| SLA | Grace Period | Use Case |
|---|---|---|
| P1 Critical | 1 hour | System down, entire team affected |
| P2 High | 4 hours | Single user unable to work |
| P3 Medium | 8 hours | Non-urgent software or access requests |
| P4 Low | 24 hours | General enquiries |
 
The default 18 hour SLA was left as a fallback.
 
### Help Topics
 
Help topics are what users select when raising a ticket. Each one is tied to a department, priority, and SLA plan so tickets get routed automatically without an agent manually triaging every one.
 
| Help Topic | Department | Priority | SLA |
|---|---|---|---|
| Password Reset | IT Support | Normal | P2 High |
| Hardware Fault | IT Support | High | P2 High |
| Software Install Request | IT Support | Normal | P3 Medium |
| New Starter | HR | Normal | P3 Medium |
| General Enquiry | IT Support | Low | P4 Low |
 
### Canned Responses
 
Pre-written replies for common ticket types. Instead of typing the same thing out every time someone needs a password reset, you pick the canned response, fill in the placeholders, and send it.
 
Six responses were created covering password resets, hardware faults, software installs, new starters, general enquiries, and ticket closures. Each uses `[placeholder]` values the agent fills in when applying the response. Full examples are in the [README](README.md#canned-responses).
 
## LDAP Integration
 
The goal was to let Active Directory users from lab.local log into osTicket using their domain credentials instead of needing a separate osTicket account. In a real environment, you would not want staff managing two sets of credentials.
 
### Plugin Installation
 
osTicket does not include LDAP support out of the box. The official LDAP Authentication and Lookup plugin was downloaded from the osTicket plugins GitHub repo and placed into the plugins directory manually:
 
```bash
sudo curl -L -o /tmp/osticket-ldap.zip https://github.com/osTicket/osTicket-plugins/archive/refs/heads/develop.zip
sudo unzip /tmp/osticket-ldap.zip 'osTicket-plugins-develop/auth-ldap/*' -d /tmp/ldap-extract
sudo cp -r /tmp/ldap-extract/osTicket-plugins-develop/auth-ldap /var/www/html/osticket/upload/include/plugins/
sudo chown -R apache:apache /var/www/html/osticket/upload/include/plugins/
```
 
After that it appeared in Admin Panel > Manage > Plugins and was installed and enabled from there.
 
### Net_LDAP2 Dependency
 
First problem. Saving the LDAP plugin configuration threw:
 
```
Failed opening required 'include/Net/LDAP2.php'
```
 
The plugin depends on Net_LDAP2 and does not bundle it. Installed via PEAR:
 
```bash
sudo dnf install php-pear -y
sudo pear install Net_LDAP2
```
 
That installed it to `/usr/share/pear/Net/` but osTicket was not looking there. The plugin uses a relative path that resolves to `/var/www/html/osticket/upload/include/Net/LDAP2.php`. The files had to be copied to that exact location with correct ownership and SELinux context:
 
```bash
sudo cp -r /usr/share/pear/Net /var/www/html/osticket/upload/include/Net
sudo chown -R apache:apache /var/www/html/osticket/upload/include/Net
sudo chcon -R -t httpd_sys_content_t /var/www/html/osticket/upload/include/Net
```
 
### Plugin Configuration
 
The plugin was configured with the DC01 connection details:
 
- **Default Domain:** `lab.local`
- **DNS Servers:** `192.168.122.10`
- **LDAP Server:** `win-798ni1fmr23.lab.local`
- **Search User:** `CN=Administrator,CN=Users,DC=lab,DC=local`
- **Search Base:** `DC=lab,DC=local`
- **LDAP Schema:** Microsoft Active Directory
- **Authentication:** Staff and client both enabled
The search user is the account osTicket uses to look up users in AD when someone tries to log in. It is not the account people log in with. I used the Administrator account because this is a lab. In production you would create a dedicated service account with minimal read only permissions.
 
### Troubleshooting the Connection Failure
 
Saving the configuration returned:
 
```
Unable to connect any listed LDAP servers
Bind failed: Can't contact LDAP server
```
 
My first assumption was that the configuration was wrong. It was not.
 
**Step 1: Verify connectivity.** DNS resolved, DC01 was pingable, routing was fine. Network problem ruled out.
 
**Step 2: Check logs.** Apache's error log gave nothing useful. PHP-FPM's log was more interesting:
 
```
ldap_close(): Argument #1 ($ldap) must be of type LDAP\Connection, false given
```
 
This suggested the connection was never being established. PHP was trying to clean up a connection handle that was `false`, meaning the initial `ldap_connect()` was silently failing.
 
**Step 3: Add timeouts.** I patched explicit timeout values into the plugin source to surface failures faster:
 
```php
$info['options'] = array(
    'LDAP_OPT_PROTOCOL_VERSION' => 3,
    'LDAP_OPT_REFERRALS' => 0,
    'LDAP_OPT_TIMELIMIT' => 5,
    'LDAP_OPT_NETWORK_TIMEOUT' => 5,
);
```
 
The plugin now failed faster and more predictably, but it still failed. The timeout was not the cause. It was just exposing the real problem quicker.
 
**Step 4: Test LDAP manually.** If the config is right and the server is reachable, can anything on this box actually talk LDAP?
 
```bash
ldapwhoami -x -H ldap://win-798ni1fmr23.lab.local:389 -D "Administrator@lab.local" -W
```
 
Result:
 
```
u:LAB\Administrator
```
 
DNS working. Port 389 open. Credentials correct. AD accepting binds. The LDAP configuration was never the problem.
 
**Step 5: Packet capture.** So the terminal can connect but osTicket cannot. The difference is the user context. I ran a capture and triggered the connection from the browser:
 
```bash
sudo tcpdump -i any host 192.168.122.10 and \(port 389 or port 636\)
```
 
Result: **0 packets captured.** The connection was being blocked before it even left the server.
 
**Step 6: SELinux audit log.** At this point I was confident it was SELinux. Checked the audit log:
 
```bash
sudo ausearch -m AVC -ts recent | tail -50
```
 
Found the denial:
 
```
avc: denied { name_connect } for pid=1499 comm="php-fpm" dest=389
scontext=system_u:system_r:httpd_t:s0
tcontext=system_u:object_r:ldap_port_t:s0
```
 
SELinux was blocking PHP-FPM from opening outbound connections on port 389. The web server process was trying to reach DC01 but SELinux killed the connection before the packet ever left the machine. This is why the manual `ldapwhoami` test worked (running as a normal user, not under `httpd_t`) and the plugin did not.
 
### The Fix
 
```bash
sudo setsebool -P httpd_can_connect_ldap on
sudo systemctl restart php-fpm
sudo systemctl restart httpd
```
 
The `-P` flag makes it persistent across reboots. After restarting both services, the LDAP plugin connected successfully.
 
### Creating an AD-Backed Agent
 
With LDAP working, I created an osTicket agent linked to the `lprobert` Active Directory account. The authentication backend was set to "Use any available backend" (the LDAP plugin registers itself automatically, no separate dropdown option appears).
 
LDAP only handles authentication. It proves who someone is. It does not decide what they are allowed to do inside osTicket. Permissions, department assignment, and access levels are still configured within osTicket itself. I assigned `lprobert` to the IT Support department with user management, department visibility, search, and statistics permissions.
 
### Login Issues
 
The first login attempt with `lprobert` failed. I reset the password on DC01 using the PowerShell script from my [Helpdesk Toolkit](https://github.com/LiamProbert/helpdesk-powershell-toolkit), logged into Windows with the temporary password, changed it when prompted, and tried osTicket again.
 
Still failed. Multiple retries triggered the lockout threshold:
 
```
Maximum failed login attempts reached
```
 
I checked the database directly to confirm neither account was disabled:
 
```sql
USE osticket;
SELECT staff_id, username, backend, isactive, isadmin, dept_id FROM ost_staff;
```
 
Both `lliiiamm` and `lprobert` showed as active. The issue was most likely osTicket's internal failed attempt counter rather than anything on the AD side. After restarting Apache and PHP-FPM (which would have cleared the PHP session state), the login succeeded on the next attempt.
 
I cannot be 100% certain whether it was the service restart clearing session state or simply the failed attempt counter expiring on its own, since I did not isolate those two variables at the time. Either way, the root cause was not credentials or LDAP configuration. The authentication chain was working correctly once the counter reset.
 
### Result
 
Logged into osTicket using Active Directory credentials. The authentication request was passed to the LDAP plugin, validated against DC01, and accepted. No local osTicket password required.
 
```
User → osTicket → LDAP Plugin → DC01 (Active Directory) → Authenticated
```
 
### What Actually Caused the Connection Failure
 
The LDAP server, credentials, PHP extension, and osTicket plugin were all correct from the start. Nothing was misconfigured.
 
The failure was entirely caused by Fedora's SELinux policy. Even though a normal user could connect to LDAP from the terminal without any issues, PHP-FPM runs inside a restricted security context (`httpd_t`) that blocks outbound LDAP connections by default. The fix was not a configuration change. It was granting the web server's SELinux context permission to talk to LDAP.
 
This is the second time SELinux caused problems during this project (the first being Apache's write access to the config file during installation). Fedora's default security posture is restrictive, which is good in production, but it means you have to explicitly grant permissions for anything the web server needs to do beyond serving static pages.
 
## Phase 1 Complete
 
The infrastructure went from a blank Fedora Server VM to a fully configured helpdesk platform integrated with Active Directory, with departments, SLA plans, help topics, canned responses, and domain authentication all in place.
 
Phase 2 (creating, resolving, and documenting realistic helpdesk tickets) is next.
