# Detecting Suspicious PowerShell Activity with Sysmon and Splunk
## What I did
I set up rich endpoint logging on a domain-joined Windows client using Sysmon, forwarded that data to a central Splunk server using the Splunk Universal Forwarder, simulated a common attacker technique (base64-encoded PowerShell execution), and then built a Splunk dashboard and a scheduled alert to detect that behavior automatically.
## Why I did this
I'm a cybersecurity student working toward my Security+ certification, and I wanted hands-on experience with the kind of detection engineering pipeline used in real security operations centers: log source → forwarding agent → SIEM → dashboard → alert. Sysmon and Splunk both show up constantly in blue team and SOC analyst job postings, so I wanted practical experience configuring, breaking, and fixing this exact stack rather than just knowing the theory.
## Environment
- **DC01** — Windows Server 2022, domain controller for `dbrikas.local`
- **SplunkServer** — Ubuntu 24.04, running Splunk Enterprise
- **CLIENT01** — Windows 11 Pro, domain-joined client, the target endpoint for this lab

## Step 1: Installing Sysmon

Windows' default logs are shallow — they'll tell you a process started, but not much else useful for security work. Sysmon (System Monitor), a free tool from Microsoft's Sysinternals suite, installs a kernel-level driver that captures much richer detail: full command lines, parent-child process relationships, network connections, file hashes, registry changes, and more.

Rather than use Sysmon's default logging (which captures everything and drowns you in noise), I used the SwiftOnSecurity config — a community-maintained ruleset tuned to capture attacker-relevant behavior while filtering out routine Windows noise.

```powershell
New-Item -ItemType Directory -Path C:\Tools\Sysmon -Force
cd C:\Tools\Sysmon
Invoke-WebRequest -Uri "https://download.sysinternals.com/files/Sysmon.zip" -OutFile "Sysmon.zip"
Expand-Archive -Path "Sysmon.zip" -DestinationPath "." -Force
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml" -OutFile "sysmonconfig.xml"

.\Sysmon64.exe -accepteula -i sysmonconfig.xml
```

**Hiccup — DNS resolution failure.** The downloads initially failed with "the remote name could not be resolved." Turned out DC01 (which handles DNS for the domain) wasn't running at the time — CLIENT01 had nowhere to send its DNS queries. Starting DC01 back up resolved it immediately, since domain-joined machines route external DNS lookups through their domain controller.

Once installed, I confirmed Sysmon was capturing events:

```powershell
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 5
```

Right away I could see Event ID 1 (Process Create) and Event ID 13 (Registry value set) entries — exactly the kind of data that matters for spotting things like a suspicious process launch or a malware persistence mechanism.

## Step 2: Installing the Splunk Universal Forwarder

Sysmon only logs locally — I needed a way to ship that data to SplunkServer for centralized searching, dashboards, and alerting. That's the job of the Splunk Universal Forwarder (UF): a lightweight agent installed on the endpoint that reads specified logs and forwards them to an indexer over TCP.

```powershell
msiexec.exe /i splunkforwarder.msi AGREETOLICENSE=Yes RECEIVING_INDEXER="192.168.1.221:9997" WINEVENTLOG_SYS_ENABLE=1 LAUNCHSPLUNK=1 SPLUNKUSERNAME=admin SPLUNKPASSWORD="..." /quiet /norestart
```

**Hiccup — the forwarder couldn't connect.** Testing the connection showed "configured but inactive." Digging in, I found port 9997 wasn't actually listening on SplunkServer at all — because Splunk itself wasn't started yet, and the receive port had never been enabled. Fixing that was a two-step process on SplunkServer:

```bash
sudo /opt/splunk/bin/splunk start
sudo /opt/splunk/bin/splunk enable listen 9997 -auth splunkadmin:<password>
```

After that, `Test-NetConnection` from CLIENT01 came back clean, and the forwarder showed as an Active forward.

**Hiccup — Sysmon logs specifically weren't showing up in Splunk.** The default forwarder install only ships the Windows System log, not Sysmon's Operational log. I added a custom `inputs.conf` to explicitly monitor the Sysmon channel:

```
[WinEventLog://Microsoft-Windows-Sysmon/Operational]
disabled = false
index = main
```

Even after restarting the forwarder, still nothing showed up — and the forwarder's internal log revealed why:

```
WinEventLogChannel::init: Init failed, unable to subscribe to Windows Event Log channel
'Microsoft-Windows-Sysmon/Operational': errorCode=5
```

Error code 5 is Access Denied. The Splunk Forwarder service runs as a special account called `NT SERVICE\SplunkForwarder`, and Sysmon's Operational log channel restricts read access to specific groups (Administrators, SYSTEM, and Event Log Readers) at the channel level — separate from normal file permissions. The forwarder's service account wasn't in any of those groups by default.

The fix was adding it explicitly:

```powershell
Add-LocalGroupMember -Group "Event Log Readers" -Member "NT SERVICE\SplunkForwarder"
Restart-Service SplunkForwarder
```

After that, Sysmon events started flowing into Splunk immediately — within a minute, hundreds of process-creation events showed up in a search for `sourcetype=*Sysmon*`.

## Step 3: Generating Suspicious Activity

With logging and forwarding confirmed working, I simulated a common real-world attacker technique: base64-encoded PowerShell execution. Attackers frequently obfuscate malicious commands this way to evade naive string-matching defenses and make casual log review harder. It's one of the most well-known signatures in endpoint detection, appearing in countless public detection rulesets.

```powershell
$command = 'Write-Host "Suspicious lab activity - simulated recon"; Get-Process | Select-Object -First 5'
$bytes = [System.Text.Encoding]::Unicode.GetBytes($command)
$encodedCommand = [Convert]::ToBase64String($bytes)

powershell.exe -NoProfile -EncodedCommand $encodedCommand
```

This spawns a new `powershell.exe` process using the `-EncodedCommand` flag — legitimate users almost never do this, so it's a strong signal when it shows up in logs. Sysmon caught it immediately as an Event ID 1, with the full base64 command line captured for analysis.

## Step 4: Dashboard and Alert

**Dashboard.** I built a search to pull out exactly the fields an analyst would want to see for this kind of event:

```
index=main host=CLIENT01 sourcetype="WinEventLog:Microsoft-Windows-Sysmon/Operational" EventCode=1 CommandLine="*EncodedCommand*"
| table _time, ComputerName, User, Image, ParentImage, CommandLine
```

Saved as a panel called "Encoded PowerShell Execution" on a new dashboard (`Endpoint Detection - CLIENT01`), this gives an at-a-glance view of who ran the suspicious command, from what process, and the full obfuscated command line.

**Alert.** Using the same search, I built a scheduled alert to catch this pattern automatically going forward:

- **Schedule:** every 5 minutes (`*/5 * * * *`)
- **Time range:** rolling 5-minute window (`-5m` to `now`)
- **Trigger:** fires once when results > 0
- **Action:** Add to Triggered Alerts, Medium severity

The alert fired correctly on its next scheduled run, catching the test event and logging it under Triggered Alerts with the right severity and timestamp.

## What This Demonstrates

This lab covers the full lifecycle of endpoint detection engineering on a small scale:

- Deploying enhanced host-level logging (Sysmon plus a curated detection-oriented config)
- Centralizing logs via a forwarding agent to a SIEM (Splunk)
- Troubleshooting real infrastructure problems along the way, including DNS dependency on a domain controller and service-account permissions on Windows event log channels
- Simulating a real attacker technique (encoded PowerShell) tied to a named, well-documented behavior
- Building both a visualization (dashboard) and an automated detection (alert) around that behavior, and verifying the detection actually fires end-to-end

Every problem hit along the way had a real, diagnosable cause. That troubleshooting process is, in a lot of ways, the more valuable part of the exercise: production detection pipelines break in exactly these kinds of quiet, non-obvious ways, and knowing how to trace a "why isn't this working" question down to its root cause is a core blue team skill.

**Next steps for this environment:** additional detection patterns (registry persistence, suspicious network connections, process lineage anomalies like `winword.exe` spawning `cmd.exe`), and wiring up email notifications for triggered alerts.
