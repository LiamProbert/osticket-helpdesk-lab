## Phase 2 - Ticket Resolution Library

Phase 1 was all infrastructure. Deploy the VM, install the stack, configure the helpdesk, wire up LDAP. Phase 2 was about actually using it.

The goal was to create realistic tickets through the user portal, work them through the staff panel as an agent, and resolve them properly. Not just closing tickets, but documenting the work, communicating with the user, and moving each ticket through the correct lifecycle.

I originally planned to create all eight tickets from the Phase 2 brief. In practice, three was enough:

- New Starter account request
- Active Directory password reset
- Microsoft Outlook installation request

The first two had mistakes. I got department routing wrong, missed an internal note, and put a password somewhere it shouldn't have been. By the third ticket I had the full workflow down. That progression ended up being more useful than three perfect tickets with no learning behind them.



## Testing LDAP Client Authentication

Phase 1 confirmed that `lprobert` could authenticate as a staff agent. Before creating any tickets I also needed to confirm that a normal AD user could log into the customer portal.

The test user was Toby, sitting in the HR OU in Active Directory, account active, email set to Toby@lab.local.

At first, Toby couldn't log into the user portal consistently. I'd already reset the password in Active Directory, logged into Windows as Toby to change the temporary password, rebooted the lab, and tried again in a fresh browser. Still wouldn't work.

The password was definitely correct, so I tested the account directly against LDAP from the osTicket VM:

```bash
ldapsearch -x \
-H ldap://192.168.122.10 \
-D "LAB\toby" \
-W \
-b "DC=lab,DC=local" \
"(sAMAccountName=toby)"
```

Came back clean:

```
result: 0 Success
numEntries: 1
userAccountControl: 512
badPwdCount: 0
```

Account was active, not locked, password was correct, DC01 was accepting the credentials, LDAP connection was working. The problem wasn't Active Directory.

Toby eventually got into the user portal through a fresh incognito session. The new ticket form auto-populated his AD details (email and full name), which confirmed that both client authentication and LDAP user lookup were working.



## Agent Login Session Issue

The same thing happened with `lprobert` on the staff panel. Correct password being rejected through the normal browser, even though the account had worked before.

Same test:

```bash
ldapsearch -x \
-H ldap://192.168.122.10 \
-D "LAB\lprobert" \
-W \
-b "DC=lab,DC=local" \
"(sAMAccountName=lprobert)"
```

Same result. Account valid, password valid, `badPwdCount: 0`. Worked immediately in an incognito window.

The problem was the normal browser session. Stale cookies or cached osTicket session data stored against `http://192.168.122.80` was interfering with the login. For the rest of Phase 2 I used separate incognito sessions for the user and agent portals where needed.

This also explained why repeated password resets never permanently fixed the issue. The credentials were never the problem.



## Ticket Workflow

The basic workflow for every ticket was:

1. Log into the user portal as Toby
2. Select the correct help topic
3. Submit the request
4. Log into the staff panel as `lprobert`
5. Pick up the ticket
6. Add an internal technician note
7. Send a separate customer-facing reply
8. Update the ticket status
9. Resolve the request

One of the main things I learned was that internal notes and replies are two separate submissions. An internal note records work for other technicians and the customer never sees it. A reply is the message that goes to the user. The normal pattern became:

```
Internal note → Customer reply → Further work → Internal note → Final reply → Resolved
```



## Ticket 1 - New Starter Account Request

### The Request

Submitted by Toby using the **New Starter** help topic. The request was for a fictional employee called Emma Wilson joining the HR department on Monday.

The ticket included all the details a technician would actually need: full name, department, manager, job title, start date, and requested username (`ewilson`). My first draft just said "a new employee is joining" which wouldn't have given anyone enough information to create the account.

### Ticket Not Visible to the Agent

When I logged into the staff panel as `lprobert`, the ticket wasn't in his queue. I checked the system using the local admin account and found it immediately.

The problem was department routing. The **New Starter** help topic was configured in Phase 1 to route to HR. But `lprobert` was an IT Support agent and didn't have access to the HR department. The ticket existed, he just couldn't see it.

I manually transferred the ticket to IT Support and it appeared straight away.

### Routing Lesson

The person submitting the ticket was in HR, but the actual work (creating an AD account) is an IT task. The help topic should route to IT Support, not HR, regardless of who submits it. This was my first practical example of the difference between the requester's department and the department responsible for the work.

### Missed Internal Note

I intended to add an internal note documenting the account creation and referencing the `New-User.ps1` script from the PowerShell Helpdesk Toolkit. But I moved to the reply section and resolved the ticket without submitting it.

The ticket still showed the submission, routing, transfer, customer communication, and resolution. I decided not to recreate it just to make the screenshot look perfect. The mistake was corrected in the following tickets.



## Ticket 2 - Active Directory Password Reset

### The Request

Submitted by Toby using the **Password Reset** help topic. Standard "can't log into my account this morning" request. Routed to IT Support with a P2 High SLA.

### Internal Note

This time the internal note went in properly. It documented that the user's identity had been verified, the account was checked in Active Directory, the account was active and not disabled, the password was reset, and the user would be required to change it at next login.

### Security Mistake

The customer reply included the temporary password in the ticket thread.

Even in a lab, this was bad practice. Ticket replies get stored in the database, the ticket history, email notifications, logs, backups, and screenshots. A temporary password should never be placed directly into a ticket.

The correct response should say something like:

```
Your password has been reset. The temporary password will be provided through 
the agreed secure method. You will be required to change it at next login.
```

The password itself goes through a separate secure channel.

I kept the screenshot because the ticket still demonstrated the full password reset workflow with proper P2 SLA application. The mistake was documented rather than hidden.



## Ticket 3 - Microsoft Outlook Installation

This was where the full workflow clicked.

### The Request

Submitted by Toby using the **Software Install Request** help topic. Needed Outlook installed on a new laptop (`HR-LAPTOP-02`). Routed correctly to IT Support with a P3 Medium SLA.

### First Internal Note

Before replying to Toby, I recorded the technician assessment. Confirmed the user and device details, noted that manager approval was required before the install could go ahead, and put the ticket on hold.

### First Customer Reply

Told Toby his request had been received and that approval was needed. Clear, professional, no technical detail that wouldn't mean anything to him.

### Second Internal Note

Documented that approval was received and the installation was completed. Noted that Outlook was installed on `HR-LAPTOP-02`, configured with Toby's `lab.local` account, and tested successfully.

### Final Customer Reply

Confirmed the install was done and the application was ready to use. Ticket set to Resolved.

### Result

The final ticket thread showed the full lifecycle:

1. Toby submitted the request
2. Agent reviewed the ticket
3. Internal note documented the assessment
4. Toby received an update about approval
5. Approval received
6. Internal note documented the completed work
7. Toby received a completion message
8. Ticket resolved

This was the cleanest ticket from Phase 2 and the best demonstration that I understood how the workflow actually works.



## Internal Notes vs Customer Replies

This was one of the main lessons from the ticket library.

**Internal notes** are for technicians and other agents. Troubleshooting steps, diagnostics run, scripts executed, approval status, handover information. The customer never sees these.

**Customer replies** go to the requester. Confirmation that the request was received, progress updates, resolution confirmation, instructions. Clear, non-technical where possible.

The two are separate submissions. Writing both and only submitting one doesn't save the other.



## Resolved vs Closed

**Resolved** means the technician believes the work is done. The ticket can still be reopened if the user says the issue isn't fixed or more work is needed.

**Closed** means the lifecycle is fully finished. No further work expected. If something else comes up, the user creates a new ticket.

For the Phase 2 tickets, Resolved was enough to demonstrate the work was completed.



## What I Learned

The first two tickets had mistakes. During the New Starter ticket I didn't understand why `lprobert` couldn't see the request (department routing), and I failed to submit the internal note before resolving. During the Password Reset ticket I put a temporary password directly into the ticket thread.

By the Outlook install, I completed the workflow properly: user submission, correct routing, internal documentation, customer communication, approval stage, technical completion, resolution.

The main things I took away:

- LDAP authentication can be tested independently of osTicket using `ldapsearch`
- Correct credentials don't rule out stale browser sessions
- The requester's department is not necessarily the department responsible for the work
- Agents can only see tickets in departments they have access to
- Help topic routing needs to reflect the team doing the work, not the team requesting it
- Internal notes and replies are separate actions
- Internal notes hold the technical detail, replies hold the user-facing communication
- Passwords should never be stored in ticket threads
- Resolved and Closed are different stages with different meanings
- A complete ticket history should let another technician understand exactly what happened without asking anyone
