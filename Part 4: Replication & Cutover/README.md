# Part 4 — Replication & Cutover

This phase covers replication using Azure Migrate and Azure Site Recovery: the EC2 instance is prepared for replication, its disk is synchronized to Azure, and a test migration validates that the replicated disk boots correctly as an Azure VM.

> **Current status:** replication reached a healthy, **Protected** state, and a **test migration succeeded** — the replicated disk boots correctly in Azure and is reachable via RDP. The final cutover (the permanent "Migrate" action, Steps 8–9 below) has not yet been performed on this project; those steps are documented here as the intended next action.

## What This Phase Covers

| Step | What Happens |
|---|---|
| Install mobility service | Agent installed manually on the EC2 instance |
| Register with replication appliance | Mobility service connects to the replication appliance over ports 443 and 9443 |
| Enable replication | Azure Migrate begins initial disk sync from EC2 to Azure |
| Troubleshoot private IP issue | Agent config files patched — the agent hardcodes the appliance's private IP by default |
| Monitor replication | Wait for status to reach **Protected** |
| Run test migration | Temporary VM created to verify the replicated disk boots correctly — **completed** |
| Perform cutover | Final VM created in the target resource group — **not yet performed** |

> **Important:** in this project, the mobility agent's config files hardcoded the replication appliance's **private** Azure IP by default — not a temporary glitch, but a persistent behavior that reverted on every service restart. This had to be fixed before replication could succeed at all, and is documented in full below since it isn't covered anywhere in Microsoft's own documentation.

## Prerequisites

- Phase 3 complete — EC2 discovered, assessment shows Ready for Azure
- Replication appliance registered and healthy (see `3-appliance-registration-discovery-assessment/`)
- EC2 public IP (Elastic IP) and replication appliance public IP available
- RDP access confirmed to both the EC2 instance and the replication appliance VM

## File Structure

```
4-replication-and-cutover/
└── README.md
```

Like Phase 3, this phase has no saved automation scripts. Every fix — including the private-IP config patch — was applied as a sequence of interactive PowerShell commands, run one at a time, rather than a packaged `.ps1` file. The exact commands are documented inline below.

## Replication Flow Overview

Replication is handled by the Azure Site Recovery mobility service agent running on the EC2 instance. The agent sends disk data to the replication appliance, which forwards it to Azure:

```
EC2 → Mobility Agent → Replication Appliance → Azure
```

In a normal (same-cloud or VPN-connected) deployment, Azure Migrate pushes the mobility agent to the source machine automatically. This project has **no VPN between AWS and Azure**, so that automated push fails, and the agent has to be installed manually — which is where most of this phase's real troubleshooting lives.

## Step 1 — Locate the Mobility Service Installer

Unlike a same-cloud deployment, the installer wasn't pulled from a Microsoft download link during the Replicate wizard — it was already present locally in the replication appliance's own folder structure, under `DRAppliance\...\Agents`, as `PushInstallAgentSetup.msi`.

## Step 2 — Copy the Installer to the EC2 Instance

Transferred via RDP local-drive sharing from the appliance session to the EC2 instance, into `C:\MobilityInstaller`.

> Direct file transfer between two separate RDP sessions is unreliable — clipboard-based transfers in particular caused corrupted/zero-byte files later in this phase (see Issue 3 below). Where possible, prefer a direct local-drive share over clipboard copy-paste.

## Step 3 — Install the Mobility Service on the EC2

```powershell
cd C:\MobilityInstaller
msiexec /i PushInstallAgentSetup.msi
```

Unlike some ASR deployment flows, this installer did **not** generate a registration string on its own. Registration instead required a separate **source configuration JSON file**, downloaded from the replication appliance's configuration manager, and provided during a manual registration step in the wizard. This distinction is what led to two of the issues documented below.

## Step 4 — Enable Replication in Azure Migrate

- Azure Migrate → Migration and modernization → **Replicate**
- **Yes, with another cloud provider (AWS, GCP, etc.)**
- On-premises appliance: `app-rep-jb1`
- Select the EC2 instance from the discovered machines list
- Target settings:
  - Resource group: `rg-migrate-target-jeremiah`
  - Replication storage account: `stmigratejeremiah`
  - Virtual network: `vnet-migrate-jeremiah`
  - Subnet: `snet-migrate`
- OS type: Windows
- Click **Replicate**

This is also where the connectivity failure in Issue 1 below first appeared.

## Troubleshooting

### Issue 1: Push-install targets the source machine's private IP
Enabling replication initially failed with:
```
Error 322008
Connectivity failure - Mobility service installation failed on the source machine (Name: IP: 10.0.1.254)
```
**Root cause:** Azure Migrate's discovered-server record stores the EC2 instance's OS-reported (WMI) **private** IP as its canonical identity for replication targeting — separate from, and overriding, the public IP that discovery itself used successfully. The replication appliance, sitting in Azure with no VPN to AWS, physically cannot reach that address.

Additional Security Group and firewall rules were opened as a genuine prerequisite (though not sufficient alone to fix the routing problem):
```powershell
# On the EC2 instance
netsh advfirewall firewall set rule group="File and Printer Sharing" new enable=yes
Enable-NetFirewallRule -DisplayGroup "Windows Management Instrumentation (WMI)"
```
(Security Group rules for ports 445, 135, and 49152–65535 added via Terraform/AWS CLI — see `1-aws-infrastructure/`.)

**Fix:** manual mobility service installation (Steps 1–3 above), since automated push-install cannot succeed cross-cloud without private connectivity.

### Issue 2: "Invalid source config file provided"
The source config JSON, downloaded from the appliance configuration manager for manual registration, repeatedly failed validation.
**Root cause:** clipboard-based file transfer through nested RDP sessions was silently producing corrupted or zero-byte copies.
**Fix:** verified the file directly on the EC2 instance before trusting it:
```powershell
Get-Content "C:\MobilityInstaller\<config-filename>.json" -Raw
```
Re-saved without a UTF-8 byte-order mark to rule out an encoding-level rejection as well:
```powershell
$content = Get-Content "C:\MobilityInstaller\<config-filename>.json" -Raw
[System.IO.File]::WriteAllText("C:\MobilityInstaller\config_clean.json", $content, (New-Object System.Text.UTF8Encoding $false))
```

### Issue 3: Mobility agent hardcodes the appliance's private Azure IP (undocumented)
**Symptom:** even after clean registration, the agent's live log (`svagents_curr_*.log`) showed repeated connection failures to `10.1.1.5:443` — the replication appliance's **private** Azure IP — and separately, `Couldn't resolve host name: repl-jeremiah`.

**Root cause:** the mobility agent writes the replication appliance's IP directly into its own runtime configuration files (`settings.json` and `SourceConfig.json`, under `...\agent\Application Data\etc\`). If the appliance's own registration used its private IP, that's what gets baked in — and the agent connects to it directly, with **no DNS lookup**, so a hosts-file entry alone cannot override it.

**This behavior is not documented by Microsoft** and is easy to misdiagnose as a DNS or firewall problem rather than an application-level config issue.

Found every affected file:
```powershell
Get-ChildItem "C:\Program Files (x86)\Microsoft Azure Site Recovery\agent" -Recurse -Filter "*.json" |
    Where-Object {(Get-Content $_.FullName -Raw) -match "10\.1\.1\.5"}
```

Patched both files to use the appliance's public IP instead:
```powershell
$publicIp = "20.106.129.133"
$files = @(
    "C:\Program Files (x86)\Microsoft Azure Site Recovery\agent\Application Data\etc\settings.json",
    "C:\Program Files (x86)\Microsoft Azure Site Recovery\agent\Application Data\etc\SourceConfig.json"
)
foreach ($file in $files) {
    (Get-Content $file -Raw) -replace "10\.1\.1\.5", $publicIp | Set-Content $file -Encoding UTF8
}
```

Restarted the mobility services to reload the patched config:
```powershell
Get-Service -Name "InMage*", "svagents" | Restart-Service
```

> **Important caveat, confirmed by direct testing:** `settings.json` is regenerated live from Azure's cloud-side RCM registration on **every service restart**, silently reverting this patch each time. `SourceConfig.json` holds; `settings.json` does not. As of this project, there is no permanent fix — the appliance's own "NAT IP Address" field, which would be the correct place to set this, is locked/uneditable in this appliance version. The patch above must be re-applied after any restart of the mobility services until Microsoft exposes that field.

### Issue 4: DNS resolution failure for the appliance hostname
Separately from Issue 3, the log also showed `Couldn't resolve host name: repl-jeremiah` — the appliance's hostname has no public DNS entry.
**Fix:** added a hosts-file entry mapping it directly to the public IP:
```powershell
Add-Content -Path C:\Windows\System32\drivers\etc\hosts -Value "20.106.129.133 repl-jeremiah"
```
This resolved the hostname-lookup failures but, on its own, did **not** fix Issue 3 — the two are separate problems that needed separate fixes.

### Issue 5: Missing NSG rule for the replication data channel
Even after the IP and hostname were both correct, connections still timed out (`system:10060`) on port 9443 — the ASR replication *data* channel, separate from the port 443 registration channel.
**Root cause:** the Azure NSG (`nsg-migrate-repl-jeremiah`) had no inbound rule for port 9443 — only RDP (3389) and HTTPS (443).
**Fix:**
```bash
az network nsg rule create \
  --resource-group rg-migrate-source-jeremiah \
  --nsg-name nsg-migrate-repl-jeremiah \
  --name Allow-ASR-9443 \
  --destination-port-ranges 9443 \
  --priority 310 \
  --access Allow \
  --protocol Tcp
```
Verified the appliance was actually listening on the port before assuming the NSG was the fix (ruling out an application-level failure first):
```powershell
Get-NetTCPConnection -LocalPort 9443 -State Listen -ErrorAction SilentlyContinue
```
Confirmed end-to-end connectivity from both sides after the NSG change:
```powershell
Test-NetConnection -ComputerName 20.106.129.133 -Port 9443
```
This was the fix that finally let replication proceed.

## Step 5 — Monitor Replication Progress

| Status | What It Means |
|---|---|
| Initial replication in progress | Full disk copy underway |
| Protected | Initial sync complete, delta sync running — ready for test migration |
| Critical / Warning | An error occurred — check the machine's details, and the agent log directly if the portal status seems stale |

Live agent log, useful for confirming actual progress rather than trusting the portal's percentage (which lagged behind reality more than once in this project):
```powershell
$log = Get-ChildItem "C:\Program Files (x86)\Microsoft Azure Site Recovery\agent" -Filter "svagents_curr_*.log" |
    Sort-Object LastWriteTime -Descending | Select-Object -First 1 -ExpandProperty FullName
Get-Content $log -Tail 20
```

## Step 6 — Run a Test Migration (Completed)

- Replicating machines list → EC2 instance → **Test migration**
- Selected VNet: `vnet-migrate-jeremiah`
- Azure created the test VM in `rg-migrate-target-jeremiah`

**Result:** the test VM appeared with **no public IP and no inbound RDP rule** — neither is created by default for a test migration. Fixed manually:
- Created and associated a new Standard SKU static public IP via the VM's Network Interface
- Added an inbound NSG rule for port 3389

RDP into the test VM confirmed the Windows desktop loaded correctly and the disk was bootable, validating the full pipeline end-to-end.

**Clean up:** test migration VMs are temporary and billed separately from the appliances — use **Clean up test migration** in Azure Migrate once verification is done, rather than leaving the test VM running indefinitely.

## Step 7 — Perform Cutover (Not Yet Performed)

The following is the intended next step, not yet executed on this project:

- Replicating machines list → EC2 instance → **Migrate**
- Shut down machines before migration: **No** (lab-only choice — in production, shutting down the source first prevents split-brain writes to two live copies of the same disk)
- Click **Migrate**

This finalizes the last delta sync and creates the permanent target VM in `rg-migrate-target-jeremiah` from the latest replicated state — distinct from, and replacing, the temporary test-migration VM.

## Step 8 — Attach Public IP and Verify (Not Yet Performed)

Per Azure Migrate's default behavior, the migrated VM lands with no public IP or inbound NSG rule attached, for security reasons. Based on the same fix applied to the test-migration VM in Step 6, the same two actions will be needed here: attach a public IP via the VM's Network Interface, and associate an NSG with an inbound rule for RDP (3389).

## Estimated Cost (This Phase)

Costs reflect the appliances and target VM only while actively running.

| Resource | Estimated Cost |
|---|---|
| Replication appliance VM (`Standard_E16s_v5`) | ~$1.744/hour |
| Discovery appliance VM (`Standard_D8s_v3`) | ~$0.752/hour |
| Target VM post-cutover (`Standard_D2s_v3`) | ~$0.188/hour |
| Replication appliance OS disk (650GB, Standard LRS) | ~$13.50/month (~$0.44/day while it exists) |

The replication appliance is by far the largest cost driver in this phase — deallocate or delete both appliance VMs, and delete the 650GB disk, immediately after cutover is complete and verified.

## Migration Outcome (Current State)

Replication reached a healthy, Protected status, and a test migration confirmed the replicated disk boots correctly and is reachable via RDP in Azure. Full cutover has not yet been performed — see Step 7 above for the remaining action.









## Replication & Cutover

###
This folder covers the actual replication of the EC2 instance’s disk to Azure and the resulting test migration. This is where the project’s most significant technical finding surfaced: an undocumented bug in the Azure Site Recovery Mobility Service agent that hardcodes a private IP address unreachable in a no-VPN, cross-cloud topology — along with the full diagnostic path used to isolate and work around it.

## Result

###
Replication reached “Protected” / Healthy” status, and a subsequent test migration succeeded — the VM appeared correctly in rg-migrate-target-jeremiah, and was reachable via RDP after a public IP and NSG rule were added (test-migration VMs don’t get either by default). Full cutover (the permanent “Migrate” action) has not yet been performed.

## Root Cause: No Private Connectivity Between Clouds

###
This lab deliberately has no VPN or peering between the AWS VPC and Azure VNet, to keep the environment simple and low-cost. Azure Site Recovery’s default architecture assumes the replication appliance can reach the source machine over a private network — an assumption that breaks in almost every direction once that assumption is false. Nearly everything below traces back to this one design choice.

## Issues Encountered & Fixes

###
1. Automated push-install targets the source machine’s private IP
Enabling replication through the standard wizard failed with a connectivity error (Error 322008) targeting 10.0.1.254 — the EC2 instance’s private AWS IP, unreachable from Azure without a VPN.
Root cause: Azure Migrate’s discovered-server record stores the OS-reported (WMI) private IP as the machine’s canonical identity for replication targeting — separate from, and overriding, the public IP that discovery itself used successfully. This isn’t editable through the standard UI.

2. Additional firewall/SG ports required (but not sufficient)
Opened SMB (445), RPC endpoint mapper (135), and the dynamic RPC range (49152–65535) on the EC2 Security Group, and enabled File and Printer Sharing / WMI rules in the local Windows Firewall. These are genuine prerequisites for push-install, but did not resolve the failure on their own — confirming the core issue was routing, not ports.

3. Manual Mobility Service installation required
Since automated push-install cannot succeed cross-cloud without private connectivity, the Mobility Service agent (PushInstallAgentSetup.msi) was located in the replication appliance’s own DRAppliance folder structure, transferred to the EC2 instance via RDP local-drive sharing, and installed manually.

4. “Invalid source config file” during manual registration
The source configuration JSON (downloaded from the appliance configuration manager, required to register the manually-installed agent) repeatedly failed validation.
Root cause: clipboard-based file transfer through the RDP session was silently producing corrupted or zero-byte copies.
Fix: verified file integrity with Get-Content and file size before use, and re-saved the file without a UTF-8 byte-order mark to rule out encoding as a contributing factor.

5. The core bug: private IP hardcoded in the agent’s runtime config, bypassing DNS
Even after successful manual installation, the agent’s own generated configuration (RcmProxyTransportSettings.IpAddresses in settings.json, under C:\Program Files (x86)\Microsoft Azure Site Recovery\agent\Application Data\etc\) hardcoded the replication appliance’s private Azure IP (10.1.1.5) — unreachable from AWS. The agent connects to this address directly, with no DNS lookup, so a hosts-file override has no effect on it.
Diagnosed via svagents_curr_*.log, which showed repeated Curl error (28) Timeout and (6) Couldn't resolve host against both the private IP and the appliance’s hostname.
Fix attempts and what actually held:
	•	Patched settings.json directly via PowerShell, replacing the private IP with the appliance’s public IP (20.106.129.133). This did not hold — controlled testing proved settings.json is regenerated live from Azure’s cloud-side RCM registration on every service restart, silently reverting the patch each time.
	•	The appliance’s own “NAT IP Address” configuration field (under Appliance Connectivity in the configuration manager) was investigated as a proper fix, but found to be locked/uneditable in this appliance version — FQDN-only, not user-configurable.
	•	A hosts-file entry mapping the appliance’s hostname (repl-jeremiah) to its public IP was added as a secondary fix (Add-Content to C:\Windows\System32\drivers\etc\hosts), addressing a related DNS-resolution-failure symptom documented separately by another engineer who’d hit the same class of issue.
	•	After the hosts fix and a restart, the agent log showed it now correctly attempting the public IP — but the connection still timed out (system:10060).

6. Final root cause: a missing NSG rule, not a routing or DNS problem
With the agent finally attempting the correct (public) IP and still failing, the last blocker turned out to be simpler than everything above it: the Azure NSG nsg-migrate-repl-jeremiah had no inbound rule for port 9443 (the ASR replication data channel) — only RDP (3389) and HTTPS (443) were open.
Fix: 
'''
az network nsg rule create \
  --resource-group rg-migrate-source-jeremiah \
  --nsg-name nsg-migrate-repl-jeremiah \
  --name Allow-ASR-9443 \
  --destination-port-ranges 9443 \
  --priority 310 \
  --access Allow \
  --protocol Tcp
'''

The appliance VM’s local Windows Firewall already had the correct 9443 rule (confirmed via Get-NetTCPConnection showing something listening) — it was the network-layer NSG that was blocking traffic before it ever reached the OS. Verified end-to-end connectivity with Test-NetConnection from both the local machine and the EC2 instance, both returning True, before retrying replication.

7. Test migration housekeeping
The test-migration VM landed in rg-migrate-target-jeremiah as expected, but with no public IP and no inbound RDP rule — neither is created by default for test migrations. Fixed by manually creating and associating a Standard SKU static public IP, and adding an inbound NSG rule for port 3389.

## Key Resource Identifiers

###
	•	Replication appliance hostname: repl-jeremiah
	•	Replication appliance public IP: 20.106.129.133
	•	Replication appliance private IP: 10.1.1.5
	•	NSG: nsg-migrate-repl-jeremiah
	•	Target resource group: rg-migrate-target-jeremiah


[README](https://github.com/JeremiahBTech12/Azure-Lift-Shift-Migration/blob/main/README.md)
