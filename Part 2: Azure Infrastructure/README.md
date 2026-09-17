## Azure Infrastructure

###
This folder provisions the target-side Azure resources: resource groups, networking, and the two appliance VMs (discovery and replication) that Azure Migrate uses to talk to the AWS source environment. 

## What’s Deployed

###
Two resource groups, deliberately separated by purpose:

	•	rg-migrate-source-jeremiah — staging resources: appliances, replication storage cache, Recovery Services Vault
	•	rg-migrate-target-jeremiah — home for the final migrated VM
	
Keeping these separate means the staging environment can be torn down cleanly after cutover without touching the actual workload.

	- VNet — 10.1.0.0/16, non-overlapping with the AWS VPC by design
	- Discovery appliance VM — vmm-al-jeremiah, Standard_D8s_v3 (8 vCPU, 32 GiB)
	- Replication appliance VM — vm-mig-repl-jeremiah, ultimately Standard_E16s_v5 (16 vCPU, 128 GiB) after extensive resizing (below)
	- Recovery Services Vault — Migrate-Project-Jeremiah-MigrateVault-647598466

## Issues Encountered & Fixes

###

- VM size unavailable in region (SkuNotAvailable)
The desired size, Standard_A4_v2, had no available capacity in East US at deploy time — an availability issue, not a quota issue.
Fix: substituted a modern equivalent.

- Insufficient disk space
The appliance requires 600GB+ free for staging replicated disk data; the default OS disk (~127GB) fell far short.
Fix: set disk_size_gb = 650 on the OS disk in Terraform, then manually extended the Windows partition via Disk Management — resizing the underlying Azure disk does not automatically grow the Windows partition on top of it.

- Cascading Azure quota walls
Getting to a working VM size meant clearing several independent quota limits, each with different rules:
	•	Subscription-wide “Total Regional Cores” maxed at 16/16 — required an explicit increase request (approved instantly).
	•	Family-specific quota (Standard ESv3: 10/10) was maxed separately from the regional total, and flagged “Not adjustable” — a legacy-family restriction Microsoft applies to push self-service requests toward newer families (Esv5/Esv6/Esv7) instead.


## Key Resource Identifiers

###
	•	Subscription ID: 9b96a1c5-2501-4714-9009-28c82098129c
	•	Resource groups: rg-migrate-source-jeremiah, rg-migrate-target-jeremiah
	•	Discovery appliance VM: vmm-al-jeremiah (Standard_D8s_v3)
	•	Replication appliance VM: vm-mig-repl-jeremiah (Standard_E16s_v5)
	•	Recovery Services Vault: Migrate-Project-Jeremiah-MigrateVault-647598466



[Part 3: Appliance Registration, Discovery & Assessment](https://github.com/JeremiahBTech12/Azure-Lift-Shift-Migration/blob/main/Part%203%3A%20Appliance%20Registration%2C%20Discovery%20%26%20Assessment/README.md) 
