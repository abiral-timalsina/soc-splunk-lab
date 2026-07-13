# SOC Home Lab — Splunk SIEM Deployment (Phase 1)

## Overview

This document covers Phase 1 of a SOC home lab project: deploying Splunk Enterprise as a
centralized SIEM, configuring a Windows 10 endpoint to forward Sysmon and Windows Event Logs
to it, and verifying end-to-end log ingestion.

**Lab environment:**
- Splunk Enterprise 10.4.1 — Ubuntu 24 VM (VMware)
- Splunk Universal Forwarder 10.4.1 — Windows 10 x64 VM
- Sysmon v15.21 with the [SwiftOnSecurity config](https://github.com/SwiftOnSecurity/sysmon-config)
- Kali Linux 2026.1, Metasploitable2, Wazuh 4.14.6 (present in the lab, used in later phases)

---

## Architecture

```
+------------------+          Sysmon + WinEventLog          +----------------------+
|  Windows 10 x64  |  ------------------------------------> |     Ubuntu 24        |
|  (target/victim)  |     Universal Forwarder, port 9997     |  Splunk Enterprise    |
|  - Sysmon         |                                        |  (SIEM / indexer)    |
|  - Universal      |                                        |  index: main         |
|    Forwarder      |                                        |  Web UI: :8000       |
+------------------+                                          +----------------------+
```

---

## Step 1: Install Splunk Enterprise (Ubuntu)

1. Downloaded Splunk Enterprise 10.4.1 (.deb) via `wget` using the "Copy wget link" option
   from splunk.com, to avoid a slow browser download.
2. Installed with:
   ```bash
   sudo dpkg -i splunk-10.4.1-5a009d941268-linux-amd64.deb
   ```
3. Started Splunk for the first time, accepting the license and creating an admin account:
   ```bash
   sudo /opt/splunk/bin/splunk start --accept-license --run-as-root
   ```
4. Enabled Splunk to auto-start on boot (important — a VM restart otherwise leaves Splunk
   stopped):
   ```bash
   sudo /opt/splunk/bin/splunk enable boot-start -user root
   ```
5. Confirmed the web UI was reachable at `http://localhost:8000`.

*Screenshot: `images/01-splunk-install-terminal.png`*
*Screenshot: `images/02-splunk-web-home.png`*

---

## Step 2: Configure Splunk to Receive Forwarded Data

In Splunk Web → **Settings → Forwarding and receiving → Configure receiving → New Receiving
Port**, added port `9997` (Splunk's standard forwarder-to-indexer port).

*Screenshot: `images/03-receiving-port-9997.png`*

---

## Step 3: Install Sysmon (Windows 10)

1. Downloaded Sysmon from Microsoft Sysinternals.
2. Downloaded the SwiftOnSecurity `sysmonconfig-export.xml` — a community-maintained config
   tuned for high-quality security event tracing (used instead of Sysmon's bare defaults,
   which log too little useful detail for detection work).
3. Installed Sysmon with the config applied immediately:
   ```powershell
   .\Sysmon64.exe -accepteula -i sysmonconfig-export.xml
   ```
4. Verified the Sysmon64 service was running and generating events via:
   ```powershell
   Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 5
   ```

*Screenshot: `images/04-sysmon-install.png`*

---

## Step 4: Install the Universal Forwarder (Windows 10)

1. Downloaded Splunk Universal Forwarder 10.4.1 (64-bit .msi).
2. Ran the installer:
   - Accepted the license, kept the default install path.
   - Skipped SSL certificate setup (not needed for a home lab).
   - Kept **Virtual Account** for the service logon (later changed — see Troubleshooting).
   - Enabled **Security Log** and **System Log** as Windows Event Log inputs (the installer's
     built-in checklist does *not* include Sysmon — that has to be added manually).
   - Set local admin credentials for the forwarder's own management interface.
   - Skipped the optional deployment server screen.
   - Set the **Receiving Indexer** to the Ubuntu Splunk server's IP and port 9997
     (`192.168.108.131:9997`).
3. Confirmed the install completed successfully.

*Screenshot: `images/05-forwarder-install.png`*

---

## Step 5: Add Sysmon as a Log Source

The installer's checkbox list only covers Application/Security/System/Forwarded Events/Setup
logs — Sysmon writes to its own event channel
(`Microsoft-Windows-Sysmon/Operational`) and has to be added by hand.

Created `inputs.conf` in
`C:\Program Files\SplunkUniversalForwarder\etc\system\local\`:

```ini
[WinEventLog://Microsoft-Windows-Sysmon/Operational]
disabled = false
index = main
sourcetype = WinEventLog:Sysmon
```

Restarted the forwarder service to apply:
```powershell
net stop SplunkForwarder
net start SplunkForwarder
```

---

## Step 6: Verify Ingestion

Ran a search in Splunk Web:
```
index=main sourcetype="WinEventLog:Sysmon"
```

Initial result: **0 events**, despite:
- The config file being present and syntactically correct
- `btool` confirming Splunk had loaded the Sysmon input block correctly
- Sysmon itself actively generating events at the OS level (confirmed via `Get-WinEvent`)
- Security and System logs arriving in Splunk normally (2,000+ events)

---

## Troubleshooting: Sysmon Data Not Arriving

**Symptom:** All other log sources (Security, System) worked immediately. Only the Sysmon
event channel produced zero events, with no errors logged by the forwarder for that input.

**Diagnosis process:**
1. Verified the forwarder-to-indexer network path was healthy (Security/System logs were
   flowing — ruled out a connectivity or receiving-port issue).
2. Verified `inputs.conf` existed, was saved correctly (not `.txt`), and had correct syntax.
3. Used `splunk.exe btool inputs list --debug` to confirm Splunk had actually parsed and
   loaded the Sysmon input stanza from the right file — it had, ruling out a config syntax
   or precedence problem.
4. Confirmed Sysmon was generating events locally via `Get-WinEvent`, ruling out Sysmon
   itself being the problem.
5. Checked `splunkd.log` for any Sysmon/WinEventLog-related errors — found none, which
   pointed toward a **silent permissions failure** rather than a config error.

**Root cause:** The Universal Forwarder was running under a **Virtual Account** (the
installer's default, recommended for most cases). Virtual Accounts have access to most local
data, but the `Microsoft-Windows-Sysmon/Operational` channel has a more restrictive default
ACL than Security/System — normally readable only by SYSTEM, Administrators, and the
"Event Log Readers" group. The Virtual Account wasn't implicitly covered by that ACL, so
reads silently failed with no logged error.

**Fix:** Changed the SplunkForwarder service logon account to **Local System**
(Services → SplunkForwarder → Properties → Log On tab → Local System account), then
restarted the service. Local System has guaranteed read access to all Windows Event Log
channels.

**Result:** Within one polling interval, Sysmon events began arriving in Splunk
(2,900+ events on first check, EventCode 1 process-creation events with full command-line,
parent process, and hash details from the SwiftOnSecurity config).

*Screenshot: `images/06-sysmon-events-confirmed.png`*

**Lesson for SOC work:** "No data" from a log source is rarely a config-file problem once
the config validates — it's very often a permissions/service-account issue. `btool` is the
right tool to confirm Splunk *parsed* a config; it does not confirm the process has
*permission* to read the source.

---

## Phase 1 Status: Complete

- [x] Splunk Enterprise installed, running, and configured to auto-start on boot
- [x] Receiving port 9997 configured
- [x] Universal Forwarder installed on Windows 10 target
- [x] Sysmon installed with SwiftOnSecurity config
- [x] Security, System, and Sysmon logs all confirmed flowing into Splunk
- [x] Root-caused and resolved a silent permissions failure on the Sysmon input

## Next: Phase 2

Phase 2 will cover generating attack traffic from the Kali VM against the Windows 10 target
(brute-force authentication attempts), and building Splunk searches/detections to identify
it — including mapping the detection to the relevant MITRE ATT&CK technique.
