## Errors & fixes
- Ubuntu's server download page promoted 26.04 → used "Previous releases" to get 24.04.5
- Unattended "Proceed with Unattended Installation" box in the VM creator blocked Finish → unchecked it
- Installer set / to only ~24 GB → edited the volume to use the full ~48 GB
- "Failed unmounting cdrom" on reboot → harmless, pressed Enter
- Guide's GPG command wraps across two lines in the PDF, so pasting it broke → pasted as one line
- Pasted the install command twice with no line break → merged into "-acurl", script printed help, reran it once
- Guide used 4.12 and -i, I used current 4.14 without -i so the manager and agent versions match
- Indexer step looked stuck for ~14 min → checked the log from a second SSH window, it was just slow
- manage_agents agent was tied to the VirtualBox host-only IP, so it never connected → removed it and re-added with "any"
- A second agent (my PC's host name) also showed up, so I removed it and kept one Active agent
## What FIM showed
- Added: rule 554, level 5, "File added to the system"
- Modified: rule 550, level 7, "Integrity checksum changed"
- Deleted: rule 553, level 7, "File deleted"
- Renaming a file in Windows shows up as a delete + an add, not a "renamed" event
## What surprised me
- Installer skipped the boot menu and went straight to the language screen
- Online sources state that the install takes just 10-15 minutes, while mine took almost 30
- Wazuh had alerts already there before the agents
- My PC has two IPs (Wi-Fi and VirtualBox host-only), and the wrong one broke the first agent
## Wazuh version
- 4.14.8


## Step A: Attack it

### Setup
- Windows Home doesn't support Remote Desktop Server (Pro/Enterprise only) → switched to SSH instead of RDP
- Installed OpenSSH Server via Windows Optional Features (OpenSSH Client was already present, needed Server separately)
- Started service: `Start-Service sshd`, set to auto-start: `Set-Service -Name sshd -StartupType 'Automatic'`
- Confirmed firewall rule `OpenSSH-Server-In-TCP` already existed, Enabled: True, Action: Allow
- Created local test account `testuser` (not tied to Microsoft account) for the attack target
- Kali import: VirtualBox's "Import Appliance" only accepts .ovf/.ova — had to use Machine → Add instead to register the .vbox directly
- Ping from Kali to Windows host got no replies — Windows Firewall blocks ICMP by default; not an issue, skipped since it's not required and SSH worked fine anyway

### Attack
- Tool: Hydra v9.7
- Command: `hydra -l testuser -P wordlist.txt ssh://x.x.x.x`
- Wordlist: 8 entries, real password (passwordHacked00_) placed last
- Target: Windows host, IP x.x.x.x
- Attacker: Kali VM, IP x.x.x.x, bridged network
- Attack timestamp (hydra): 15:54:19–15:54:20

### Result
- Started: 2026-10-02 15:54:19
- Finished: 2026-10-02 15:54:19
- Valid credentials found: testuser / passwordHacked00_

### What Wazuh caught
- Alert timestamp (rule 60122): 15:54:20.063
- Rule 60122 (level 5): Logon Failure - Unknown user or bad password — fired once per failed attempt
- Rule 60106 (level 3): Windows Logon Success — fired on the final successful login
- No single alert correlates multiple failures into "this looks like brute forcing"

### What it missed
- No source IP field present on the Logon Failure event — can't tell WHERE the attempts came from without that
- Auto-tagged MITRE ID (T1531, Account Access Removal) doesn't match what actually happened (brute force / credential access)
