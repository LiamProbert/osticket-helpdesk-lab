# osTicket Setup

## Why osTicket

After finishing the PowerShell Helpdesk Toolkit, I wanted a second portfolio project that showed a different set of skills. The first project was entirely terminal based. This one needed to be more visual and cover something I would realistically interact with in a 1st line support role.

A helpdesk ticketing system was the obvious choice. The specific tool does not matter at entry level. The concepts transfer across ServiceNow, Zendesk, Freshdesk, or whatever an employer uses. What matters is being able to say I deployed and configured one from scratch and understand the full ticket lifecycle.

I went with osTicket over GLPI. GLPI is a heavier asset management suite where ticketing feels like an afterthought. osTicket is simpler, more focused on pure helpdesk ticketing, and closer to what I would actually be using day to day in a 1st line role.

---

## Project Scope

The project has two phases.

Phase 1 is the infrastructure and configuration. Spin up a VM, install the LAMP stack, deploy osTicket, and configure it to mirror a realistic helpdesk environment with departments, SLA plans, ticket categories, help topics, auto-routing rules, and canned responses. LDAP authentication tied to DC01 is also part of this phase so AD users can log in with their domain credentials.

Phase 2 is the ticket resolution library. Once the system is live, I create and resolve realistic helpdesk tickets based on real scenarios. Each ticket gets logged, categorised, assigned, resolved, and closed. This builds a history showing I understand the full workflow from start to finish.

---

## VM Setup

The VM was created using virt-manager (KVM) on my Fedora ThinkPad. This is the same host running DC01.

I kept the specs minimal. osTicket on a headless server does not need much, and DC01 is already using resources on the same host:

- 2048 MB RAM
- 1 vCPU
- 20 GB storage (qcow2)

I went with Fedora Server 44 (x86_64 DVD ISO). Server edition rather than Workstation because there is no need for a desktop environment. This box is accessed entirely over SSH.

During the Fedora installer I set the hostname to `osticket.lab.local`, used automatic partitioning on the 20 GB virtual disk, created an admin user with sudo access, and left the root account disabled. Time zone was set to Europe/London.

The VM landed on IP `192.168.122.80`, same virtual network as DC01 at `192.168.122.10`. That is important for the LDAP integration later.

Rather than using the virt-manager console, I just SSH in from my ThinkPad. Copy and paste works natively that way and the terminal experience is much better.

```bash
ssh lliiiamm@192.168.122.80
```

---

## System Update

Before installing anything, I fully updated the system:

```bash
sudo dnf update -y
```

This pulled in 334 package upgrades including a new kernel. Wanted everything up to date before building on top of it.

---

## LAMP Stack Installation

LAMP is not something I needed to learn in depth for this project. It is four pieces of software that osTicket needs to run:

- **Linux** is the OS (Fedora Server 44)
- **Apache** is the web server that serves the pages
- **MariaDB** is the database that stores ticket data
- **PHP** is the language osTicket is written in

Install them, configure them, move on. The focus of this project is osTicket itself, not becoming a LAMP expert.

```bash
sudo dnf install httpd mariadb-server php php-mysqlnd php-cli php-gd php-mbstring php-xml php-intl php-json php-ldap php-apcu -y
```

I included `php-ldap` for the DC01 integration later. `php-imap` was not available in the Fedora 44 repositories so I skipped it. That only affects email piping which is not needed for this setup.

Then started and enabled both services so they run on boot:

```bash
sudo systemctl enable --now httpd mariadb
```

Both came up fine. Apache listening on port 80, MariaDB reporting "Taking your SQL requests now..."

---

## MariaDB Configuration

### Securing the Installation

Ran the built-in hardening script:

```bash
sudo mariadb-secure-installation
```

Switched to unix_socket authentication, set a root password, removed anonymous users, disallowed remote root login, removed the test database, and reloaded privilege tables.

### Creating the osTicket Database

Created a dedicated database and user for osTicket to use:

```bash
sudo mariadb -u root -p
```

```sql
CREATE DATABASE osticket;
CREATE USER 'osticket'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON osticket.* TO 'osticket'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

---

## osTicket Deployment

### Downloading osTicket

Grabbed osTicket v1.18.2 straight from GitHub:

```bash
curl -L -o /tmp/osticket.zip https://github.com/osTicket/osTicket/releases/download/v1.18.2/osTicket-v1.18.2.zip
```

### Extracting and Configuring

Extracted it into Apache's web directory:

```bash
sudo unzip /tmp/osticket.zip -d /var/www/html/osticket
```

Copied the sample config file to create the live one:

```bash
sudo cp /var/www/html/osticket/upload/include/ost-sampleconfig.php /var/www/html/osticket/upload/include/ost-config.php
```

Set ownership so Apache could access everything:

```bash
sudo chown -R apache:apache /var/www/html/osticket
```

### SELinux

This is where Fedora starts being difficult. SELinux is enforcing by default, which means Apache could not write to the config file even after setting the correct file permissions. The SELinux context needed updating:

```bash
sudo chcon -R -t httpd_sys_rw_content_t /var/www/html/osticket/upload/include/ost-config.php
```

The config file also needed write permissions temporarily so the web installer could do its thing:

```bash
sudo chmod 0666 /var/www/html/osticket/upload/include/ost-config.php
```

### Firewall

Opened up HTTP traffic:

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload
```

---

## Web Installer

Hit the installer from my ThinkPad browser:

```
http://192.168.122.80/osticket/upload/
```

The prerequisites check passed for everything except PHP IMAP, which I already knew about.

Filled in the installer with lab details. Helpdesk name set to "lab Helpdesk", default email set to `helpdesk@lab.local`, created an admin account, and pointed it at the `osticket` database with the credentials I set up earlier.

After it finished, I removed the setup directory because osTicket tells you to and it is a security risk leaving it there:

```bash
sudo rm -rf /var/www/html/osticket/upload/setup
```

Locked down the config file permissions:

```bash
sudo chmod 0644 /var/www/html/osticket/upload/include/ost-config.php
```

---

## Accessing osTicket

Staff control panel:

```
http://192.168.122.80/osticket/upload/scp
```

User facing portal:

```
http://192.168.122.80/osticket/upload/
```

Logged in with the admin credentials and the dashboard showed the default "osTicket Installed!" ticket. Everything working.

---

## Phase 1 Configuration

### Departments

osTicket comes with three default departments: Support, Sales, and Maintenance. I renamed them to reflect a realistic office environment and to align with the OUs I already have in Active Directory on DC01.

| Department | Type | Notes |
|---|---|---|
| IT Support | Public | Default department, where the bulk of tickets land |
| HR | Public | Onboarding, offboarding, HR requests |
| Facilities | Public | Hardware, printers, building access |

IT Support was kept as the default so anything unrouted lands there first. That is standard practice. If a ticket does not match any routing rule, it goes to the biggest team.

### SLA Plans

SLA stands for Service Level Agreement. In a helpdesk context it defines how quickly a ticket needs to be responded to based on its priority. If something critical is down, someone has to respond within an hour. If someone has a general question, 24 hours is fine.

I created four plans to match a standard priority structure:

| SLA | Grace Period | Use Case |
|---|---|---|
| P1 Critical | 1 hour | System down, entire team affected |
| P2 High | 4 hours | Single user unable to work |
| P3 Medium | 8 hours | Non-urgent software or access requests |
| P4 Low | 24 hours | General enquiries, nice to have requests |

The default SLA (18 hours) was left in place as a fallback.

### Help Topics

Help topics are what users select when they raise a ticket. Each one is tied to a department, priority, and SLA plan so tickets get routed automatically without an agent having to manually triage every one.

I removed the defaults and replaced them with topics a 1st line helpdesk would actually deal with daily:

| Help Topic | Department | Priority | SLA |
|---|---|---|---|
| Password Reset | IT Support | Normal | P2 High |
| Hardware Fault | IT Support | High | P2 High |
| Software Install Request | IT Support | Normal | P3 Medium |
| New Starter | HR | Normal | P3 Medium |
| General Enquiry | IT Support | Low | P4 Low |

### Canned Responses

Canned responses are pre-written replies that agents can fire off quickly for common ticket types. Instead of typing the same thing out every time someone needs a password reset, you pick the canned response, fill in the placeholders, and send it.

I created six covering the most common interactions:

| Title | Department |
|---|---|
| Password Reset - Confirmation | IT Support |
| Hardware Fault - Logged and Assigned | IT Support |
| Software Install - Request Received | IT Support |
| New Starter - Account Setup Confirmation | HR |
| General Enquiry - Response | IT Support |
| Ticket Resolved - Closing | IT Support |

Each one uses placeholders in square brackets like `[name]` and `[password]` which the agent fills in when applying the response to a ticket.

---

## LDAP Integration

The goal was to let Active Directory users from lab.local log into osTicket using their domain credentials instead of needing a separate osTicket account. In a real environment, you would not want staff managing two sets of credentials.

### Plugin Installation

osTicket does not include LDAP support out of the box. The official LDAP Authentication and Lookup plugin had to be downloaded from the osTicket plugins GitHub repo and dropped into the plugins directory manually:

```bash
sudo curl -L -o /tmp/osticket-ldap.zip https://github.com/osTicket/osTicket-plugins/archive/refs/heads/develop.zip
sudo unzip /tmp/osticket-ldap.zip 'osTicket-plugins-develop/auth-ldap/*' -d /tmp/ldap-extract
sudo cp -r /tmp/ldap-extract/osTicket-plugins-develop/auth-ldap /var/www/html/osticket/upload/include/plugins/
sudo chown -R apache:apache /var/www/html/osticket/upload/include/plugins/
```

After that it showed up in Admin Panel > Manage > Plugins and I installed and enabled it from there.

### Net_LDAP2 Dependency

The first problem. Trying to save the LDAP plugin configuration threw:

```
Failed opening required 'include/Net/LDAP2.php'
```

The plugin depends on a PHP library called Net_LDAP2 and does not bundle it. I installed it through PEAR:

```bash
sudo dnf install php-pear -y
sudo pear install Net_LDAP2
```

That installed it to `/usr/share/pear/Net/` but osTicket was not looking there. The plugin code uses `require('include/Net/LDAP2.php')` as a relative path, which resolves to `/var/www/html/osticket/upload/include/Net/LDAP2.php`. The file needed to be in that exact location.

Copied it in and set the ownership and SELinux context:

```bash
sudo cp -r /usr/share/pear/Net /var/www/html/osticket/upload/include/Net
sudo chown -R apache:apache /var/www/html/osticket/upload/include/Net
sudo chcon -R -t httpd_sys_content_t /var/www/html/osticket/upload/include/Net
```

After that the library error was gone.

### Configuring the Plugin

Created a plugin instance with the DC01 connection details:

- **Default Domain:** `lab.local`
- **DNS Servers:** `192.168.122.10`
- **LDAP Server:** `win-798ni1fmr23.lab.local`
- **Search User:** `CN=Administrator,CN=Users,DC=lab,DC=local`
- **Search Base:** `DC=lab,DC=local`
- **LDAP Schema:** Microsoft Active Directory
- **Staff Authentication:** Enabled
- **Client Authentication:** Enabled

The search user is the account osTicket uses to look up users in AD when someone tries to log in. It is not the account people log in with. I used the Administrator account because this is a lab. In production you would create a dedicated service account with minimal permissions.

Saving this time gave a different error:

```
Unable to connect any listed LDAP servers
Bind failed: Can't contact LDAP server
Unable to bind to server win-798ni1fmr23.lab.local
```

### Troubleshooting the Connection Failure

My first assumption was that the configuration was wrong. It was not.

I confirmed basic connectivity from the osTicket VM. DNS resolved, the domain controller was reachable, routing was fine. That ruled out a network problem.

The Apache log gave nothing useful. The PHP-FPM log was more interesting and contained repeated errors like:

```
ldap_close(): Argument #1 ($ldap) must be of type LDAP\Connection, false given
```

This suggested the LDAP connection was not even being established before PHP tried to clean it up. The plugin was failing silently before it could return a meaningful error.

I added explicit timeout values to the plugin source code so failures would surface faster rather than waiting for PHP to hit its execution limit:

```php
$info['options'] = array(
    'LDAP_OPT_PROTOCOL_VERSION' => 3,
    'LDAP_OPT_REFERRALS' => 0,
    'LDAP_OPT_TIMELIMIT' => 5,
    'LDAP_OPT_NETWORK_TIMEOUT' => 5,
);
```

The plugin was now failing faster and more predictably. But it was still failing. The timeout was not the cause, it was just exposing the real problem faster.

Next I tested the LDAP connection manually from the terminal to see if DC01 was actually reachable and accepting connections:

```bash
ldapwhoami -x -H ldap://win-798ni1fmr23.lab.local:389 -D "Administrator@lab.local" -W
```

After entering the password it came back with:

```
u:LAB\Administrator
```

This was the key moment. DNS resolution worked, port 389 was open, the credentials were correct, and Active Directory was accepting binds. There was nothing wrong with the LDAP configuration at all.

So the terminal could connect but osTicket could not. The difference was the user context. The terminal was running as my normal Linux user. Apache and PHP-FPM run under a restricted SELinux context called `httpd_t`.

I ran a packet capture to confirm:

```bash
sudo tcpdump -i any host 192.168.122.10 and \(port 389 or port 636\)
```

Then tried saving the LDAP config in the browser. Result: 0 packets captured. The connection was being blocked before it even left the server.

Checked the SELinux audit logs:

```bash
sudo ausearch -m AVC -ts recent | tail -50
```

Found exactly what I expected:

```
avc: denied { name_connect } for pid=1499 comm="php-fpm" dest=389
scontext=system_u:system_r:httpd_t:s0
tcontext=system_u:object_r:ldap_port_t:s0
```

SELinux was blocking PHP-FPM from opening outbound connections on port 389. The web server was trying to reach DC01 but SELinux killed the connection before the packet ever left the machine.

### The Fix

One command:

```bash
sudo setsebool -P httpd_can_connect_ldap on
```

The `-P` flag makes it persistent across reboots. Then restarted both services:

```bash
sudo systemctl restart php-fpm
sudo systemctl restart httpd
```

Saved the LDAP configuration again. No errors. Connection test passed.

### Creating an AD-Backed Agent

With LDAP connectivity working I created an osTicket agent linked to an Active Directory account.

I created a new agent for `lprobert` and set the authentication backend to "Use any available backend." The LDAP plugin registers itself automatically as an available backend so no separate LDAP option appears in the dropdown. The plugin handles authentication in the background when it is available.

LDAP only proves who someone is. It does not decide what they are allowed to do once they have logged in. That is still controlled inside osTicket. I assigned `lprobert` to the IT Support department and configured the appropriate permissions including user management, department visibility, search, and statistics.

### Login Issues

The first login attempt failed.

I went back to DC01 and used the Password Reset PowerShell script I had written earlier in the project to reset the password, logged into Windows with the temporary password, changed it when prompted, and tried osTicket again.

Still failed. Closed the browser completely, opened a fresh incognito window, tried again. Still no luck.

Eventually even the local administrator account stopped working and osTicket returned:

```
Maximum failed login attempts reached
```

Too many failed attempts had locked the authentication state. I connected to the MariaDB database to check the state of the accounts:

```bash
sudo mariadb -u root -p
```

```sql
USE osticket;
SELECT staff_id, username, backend, isactive, isadmin, dept_id FROM ost_staff;
```

Both accounts, `lliiiamm` and `lprobert`, showed as active. Nothing was disabled at the database level. After restarting the services and clearing the failed authentication state, I tried once more.

This time it worked.

### Result

Logged into osTicket successfully using Active Directory credentials. The authentication request was passed to the LDAP plugin, validated against DC01, and accepted. No local osTicket password was required.

The full authentication chain:

```
Active Directory → LDAP → osTicket
```

Exactly how it would work in a real enterprise environment.

### What Actually Caused the Issue

The LDAP server, credentials, PHP extension, and osTicket plugin were all correct from the start. Nothing was misconfigured.

The failure was entirely caused by Fedora's SELinux policy. Even though a normal user could connect to LDAP from the terminal without any issues, PHP-FPM runs inside a restricted security context that does not have permission to create outbound LDAP connections by default. The fix was not changing anything in the LDAP configuration. It was allowing the web server's SELinux context to talk to LDAP.

This is the second time SELinux has caused problems during this project. The first was during the initial osTicket deployment when Apache could not write to the config file. Fedora's default security posture is restrictive, which is good in production, but it means you have to explicitly grant permissions for anything the web server needs to do beyond serving static pages.

---

## Phase 1 Complete

The infrastructure went from a blank Fedora Server VM to a fully configured helpdesk platform integrated with Active Directory, with departments, SLA plans, help topics, canned responses, and domain authentication all in place.

Everything remaining is Phase 2: creating, resolving, and documenting realistic helpdesk tickets before publishing to GitHub.
