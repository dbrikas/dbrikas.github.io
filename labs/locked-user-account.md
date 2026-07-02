---
---
# Resolving a Locked User Account in Active Directory

## What I did

I simulated a real account lockout scenario in my Active Directory lab — configuring an account lockout policy from scratch, triggering a real lockout by repeatedly entering the wrong password, then finding and fixing it the way a help desk technician or sysadmin would.

## Why I did this

Account lockouts are one of the most common tickets a help desk or IT support person deals with day to day. I wanted hands-on experience with the full cycle: understanding why lockouts happen, how the policy behind them is configured, and how to actually resolve one — not just click a button, but understand what's happening underneath.

## Environment

- Windows Server 2022 domain controller (`dbrikas.local`), built in a previous lab
- Test user account: John Smith (`jsmith`)

## What I actually did, step by step

### 1. Tried to trigger a lockout — and found the policy wasn't even configured
My first attempt was to just log in as John Smith with the wrong password several times and see what happened. Nothing did — it kept rejecting the password indefinitely with no lockout, no matter how many times I tried.

This turned out to be expected behavior, not a bug: Windows Server's default domain policy ships with **no account lockout threshold set** at all. Lockout protection isn't automatic — a domain administrator has to explicitly configure it. That was a useful thing to learn firsthand rather than just being told.

### 2. Configured the account lockout policy
Using Group Policy Management, I edited the Default Domain Policy and set:
- **Account lockout threshold**: 5 invalid logon attempts
- **Account lockout duration**: 30 minutes
- **Reset account lockout counter after**: 30 minutes

These are reasonable, realistic values — strict enough to slow down a brute-force guessing attempt, lenient enough that a real employee who mistypes their password a couple of times won't get locked out over nothing.

I then ran `gpupdate /force` to push the policy immediately instead of waiting for Windows' normal refresh cycle.

### 3. Triggered the lockout for real
With the policy in place, I attempted to log in as John Smith with the wrong password five times. On the fifth attempt, I got the message: *"The referenced account is currently locked out and may not be logged on to."* Exactly what a real locked-out user would see.

### 4. Found and unlocked the account
In Active Directory Users and Computers, I opened John Smith's account properties and confirmed the **Account** tab showed "This account is currently locked out on this Active Directory Domain Controller." I checked the **Unlock account** box and applied the change.

### 5. Ran into a real Windows security behavior
My first instinct to verify the fix was to log back in as John Smith directly on the domain controller. Windows blocked this with: *"The sign-in method you're trying to use isn't allowed."*

This is actually correct, intentional behavior — regular domain users aren't allowed to log on interactively to a domain controller by default, since DCs are sensitive infrastructure. In a real company, employees log into regular workstations, not the DC itself. I don't currently have a separate client VM in this lab, so I hit this limitation directly, which was a good reminder of how access is genuinely restricted in a real environment.

### 6. Verified the fix using PowerShell instead
Since an interactive login wasn't an option, I verified everything using PowerShell:

```powershell
Get-ADUser jsmith -Properties LockedOut, BadPwdCount, PasswordLastSet
```

The output confirmed:
- `LockedOut: False` — the unlock worked
- `BadPwdCount: 0` — reset cleanly
- `PasswordLastSet` showing the current date/time — the forced password change went through

## What I learned

- Security policies like account lockout aren't automatic — someone has to configure them deliberately, and the defaults are more permissive than I expected.
- The 5-attempts / 30-minute lockout window is a genuine security tradeoff between stopping brute-force attacks and not locking out real users for simple typos.
- Domain controllers restrict interactive logons from regular users by design — this isn't a limitation to work around, it's a real security boundary that reflects how production environments are actually run.
- Verifying a fix doesn't always mean clicking through a GUI — PowerShell gave me a faster, more precise way to confirm the account state than trying (and failing) to log in directly.

## What's next

Continuing this lab series with:
- Delegation of Control — granting a non-admin user specific permissions (like the ability to unlock accounts) without giving them full domain admin rights
