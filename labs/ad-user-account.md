# Building My First Active Directory Lab (From Scratch)

## What I did

I built a Windows Server 2022 domain controller from the ground up in a virtual machine on my own computer, then used it to create my first Active Directory user account. This is the same core setup that IT teams use at most companies to manage employee logins, permissions, and computer access.

## Why I did this

I'm a cybersecurity student working toward my Security+ certification, and I wanted hands-on experience with Active Directory instead of just reading about it. AD shows up in a huge percentage of IT and security job postings, so I wanted to be able to say "I've actually built and managed one" rather than just "I know what it is."

## Environment

- **Host machine**: 32GB RAM, running VirtualBox
- **Virtual machine**: Windows Server 2022 Standard Evaluation (free 180-day license from Microsoft), 4GB RAM allocated, 60GB disk
- **Domain created**: `dbrikas.local`

## What I actually did, step by step

### 1. Set up the virtual machine
I installed VirtualBox and downloaded the free Windows Server 2022 evaluation ISO directly from Microsoft. I created a new VM, gave it 4GB of RAM and a 60GB virtual hard disk, and attached the ISO so it would boot into the Windows Server installer.

### 2. Installed Windows Server
I ran through the actual Windows Server installation screens myself (rather than letting it auto-install) so I could see and understand every step — accepting the license, choosing the "Desktop Experience" edition for a full GUI, and picking where to install the OS on the virtual disk.

**A real problem I ran into:** partway through, the VM hit a black screen after a reboot and I restarted it too early, which corrupted that install attempt and sent me back to the beginning of setup. On the second attempt, I let it sit through the reboot patiently instead of resetting it, and it completed successfully. I also had to manually delete leftover disk partitions from the failed attempt before reinstalling cleanly. Small thing, but it's a good reminder that "it looks stuck" doesn't always mean it's actually stuck — first boots after a big install can genuinely take several minutes with no visual feedback.

### 3. Installed the Active Directory Domain Services role
Using Server Manager, I added the AD DS server role, which is the Windows feature that turns a regular server into a domain controller.

### 4. Promoted the server to a Domain Controller
This is the step that actually creates the domain. I chose to create a brand new forest and named the domain `dbrikas.local` (the `.local` suffix is the standard convention for internal lab/test domains that aren't meant to be reachable on the public internet). I kept the DNS server and Global Catalog options enabled, since a domain controller needs DNS to function properly, and set a separate recovery password (DSRM) used only for disaster recovery scenarios.

I got a couple of expected warnings during this step — like DNS delegation not being possible — which is normal for a standalone lab domain and not something to worry about.

### 5. Verified the domain was live
After the server rebooted, I confirmed under Server Manager → Local Server that the domain now showed as `dbrikas.local` instead of a plain workgroup, and that both AD DS and DNS were listed as installed roles.

### 6. Created my first user account
Using Active Directory Users and Computers, I created a new user in the domain (a fictional employee, "John Smith"), set an initial password, and required the password to be changed at next logon — which is the realistic default for a new hire account in a real company.

## What I learned

- Active Directory isn't just a concept — it's the actual authority a company relies on to decide who can log into what. Setting one up made concepts like "domain," "domain controller," and "forest" click in a way that reading definitions never did.
- Troubleshooting is part of the job. My install didn't go perfectly on the first try, and figuring out *why* (resetting too early during a slow reboot) and fixing it was arguably more valuable than if everything had worked immediately.
- Small config choices (like the `.local` domain suffix, or DSRM password) exist for specific reasons tied to how DNS and disaster recovery actually work — not just arbitrary setup steps.

## What's next

I'm continuing this lab series with:
- Simulating and resolving a locked user account
- Delegating administrative control over an organizational unit

I'll be documenting each of these the same way as I build them out.
