---
---
# Delegation of Control in Active Directory

## What I did

I set up a realistic "help desk" scenario in Active Directory: created a dedicated Organizational Unit (OU), created a non-admin user to act as a help desk employee, and delegated a specific, limited set of permissions to that user — the ability to reset passwords and read user information for accounts inside that OU only. I then verified the delegation actually worked by inspecting the underlying permissions directly with PowerShell, rather than just trusting the wizard's success message.

## Why I did this

This lab is about a core security principle: **least privilege**. In a real company, you don't want every help desk employee to be a full Domain Admin just so they can unlock accounts and reset passwords. Delegation of Control is how Active Directory lets you hand out narrow, specific permissions instead of all-or-nothing access. Understanding this concept — and being able to actually configure and verify it — is directly relevant to Security+ and to how real organizations are structured.

## Environment

- Windows Server 2022 domain controller (`dbrikas.local`)
- Existing user: John Smith (`jsmith`) — a regular employee
- New user created for this lab: Alex Rivera (`arivera`) — playing the role of a help desk employee

## What I actually did, step by step

### 1. Created an Organizational Unit
I created a new OU called **IT Support**. An OU is essentially a container inside Active Directory used to group and organize users, computers, or other objects — and importantly, it's the scope boundary you delegate permissions against. You don't delegate control over the whole domain; you delegate control over a specific OU.

### 2. Created a delegated user and moved a target user into scope
I created Alex Rivera inside the IT Support OU — this is the account that will receive delegated permissions. I also moved John Smith's existing account into the same OU, so there would be a real account for Alex to actually manage.

### 3. Ran the Delegation of Control Wizard
Right-clicking the IT Support OU and selecting **Delegate Control** opens a wizard that walks through:
- Choosing which user or group to delegate to (Alex Rivera)
- Choosing which specific tasks to grant

I deliberately selected only:
- **Reset user passwords and force password change at next logon**
- **Read all user information**

And left broader permissions unchecked — things like creating/deleting user accounts or managing group membership. This mirrors a real help desk role: enough access to solve the most common tickets (password resets), without the ability to create new accounts, delete accounts, or restructure groups.

### 4. Verified the delegation with PowerShell
Rather than just trusting the wizard's "success" confirmation screen, I inspected the actual Access Control List (ACL) on the OU to see what permissions were really granted under the hood:

```powershell
Import-Module ActiveDirectory
Get-Acl "AD:\OU=IT Support,DC=dbrikas,DC=local" | Select-Object -ExpandProperty Access | Where-Object {$_.IdentityReference -like "*arivera*"}
```

The output showed concrete Access Control Entries (ACEs) tied to `DBRIKAS\arivera`, including:
- **ReadProperty** rights — matching the "read user information" permission
- **WriteProperty** rights — part of the password reset capability
- An **ExtendedRight** entry — the specific reset-password right

This confirmed the delegation genuinely took effect at the permission level, not just in the wizard's UI.

## What I learned

- Delegation of Control is scoped to an OU, not the whole domain — which means good OU structure (grouping users logically) is a prerequisite for good, granular security delegation. You can't delegate cleanly if everyone's dumped into one flat "Users" container.
- The wizard is really just a friendly interface over raw Windows ACL entries. Understanding that underlying structure — and being able to query it directly — is a more durable skill than knowing which checkboxes to click.
- "Least privilege" isn't an abstract security buzzword — it's a concrete, configurable thing in AD, and it's genuinely quick to set up once you understand OUs and the delegation wizard.

## What's next

This wraps up my initial Active Directory lab series (user creation, account lockout resolution, and delegation of control). Next, I'm planning to add a Windows client VM to the lab so I can test scenarios like this one from an actual employee's perspective, rather than working entirely from the domain controller.
