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
