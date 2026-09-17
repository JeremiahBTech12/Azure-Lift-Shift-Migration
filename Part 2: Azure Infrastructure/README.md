Azure Infrastructure
This folder provisions the target-side Azure resources: resource groups, networking, and the two appliance VMs (discovery and replication) that Azure Migrate uses to talk to the AWS source environment. The bulk of the hands-on troubleshooting in this project happened here, specifically around sizing and provisioning the replication appliance VM — a process that turned into an extended, multi-day exercise in Azure compute quota management and VM family compatibility.
What’s Deployed
	•	Two resource groups, deliberately separated by purpose:
	•	rg-migrate-source-jeremiah — staging resources: appliances, replication storage cache, Recovery Services Vault
	•	rg-migrate-target-jeremiah — home for the final migrated VM
Keeping these separate means the staging environment can be torn down cleanly after cutover without touching the actual workload.
	•	VNet — 10.1.0.0/16, non-overlapping with the AWS VPC by design
	•	Discovery appliance VM — vmm-al-jeremiah, Standard_D8s_v3 (8 vCPU, 32 GiB)
	•	Replication appliance VM — vm-mig-repl-jeremiah, ultimately Standard_E16s_v5 (16 vCPU, 128 GiB) after extensive resizing (below)
	•	Recovery Services Vault — Migrate-Project-Jeremiah-MigrateVault-647598466
Issues Encountered & Fixes
1. Terraform output typo
outputs.tf referenced .ip_addresss (extra “s”) instead of .ip_address, breaking terraform output for the replication appliance’s public IP.
Fix: corrected the attribute reference.
2. VM size unavailable in region (SkuNotAvailable)
The lab documentation’s suggested size, Standard_A4_v2, had no available capacity in East US at deploy time — an availability issue, not a quota issue.
Fix: substituted a modern equivalent, Standard_D4s_v3.
3. Subscription quota exceeded (DSv3 family)
Standard_D4s_v3 would have pushed the DSv3 family core count over the subscription’s approved limit (10 cores).
Fix: switched to Standard_D2s_v3, small enough to fit the remaining headroom.
4. The real sizing problem: appliance requirements vs. Azure’s core-counting quirks
The replication appliance’s prerequisite check requires 8 physical cores and 16GB+ RAM minimum — and Azure’s own vCPU-to-physical-core ratio varies by VM generation in ways that aren’t obvious from the size name alone:

The appliance’s checker reads NumberOfCores from Get-WmiObject -Class Win32_Processor, not logical processor count — so a “vCPU” figure in a VM size name isn’t a reliable proxy for what the appliance actually requires. This alone caused four separate resize cycles before the pattern was identified and a size was picked with enough headroom to guarantee 8 physical cores outright.
5. Insufficient disk space
The appliance requires 600GB+ free for staging replicated disk data; the default OS disk (~127GB) fell far short.
Fix: set disk_size_gb = 650 on the OS disk in Terraform, then manually extended the Windows partition via Disk Management — resizing the underlying Azure disk does not automatically grow the Windows partition on top of it.
6. Cascading Azure quota walls
Getting to a working VM size meant clearing several independent quota limits, each with different rules:
	•	Subscription-wide “Total Regional Cores” maxed at 16/16 — required an explicit increase request (approved instantly via the standalone Quotas app at portal.azure.com/#view/Microsoft_Azure_Capacity/QuotaMenuBlade, after the subscription-embedded quota page falsely reported “insufficient permissions” despite Owner-level access).
	•	Family-specific quota (Standard ESv3: 10/10) was maxed separately from the regional total, and flagged “Not adjustable” — a legacy-family restriction Microsoft applies to push self-service requests toward newer families (Esv5/Esv6/Esv7) instead.
	•	Several newer SKUs (EDSv5, DDSv5) showed zero approved quota and regional unavailability (“high demand”), ruling them out entirely.
7. VM resize architectural conflicts
	•	Attempting to resize from an ESv3 size (has a local resource disk) to an Ev5 size (no local resource disk) failed outright: “Unable to resize the VM since changing from resource disk to non-resource disk VM size… is not allowed.” Fix: used terraform apply -replace="azurerm_windows_virtual_machine.replication" to force a destroy/recreate instead of an in-place resize.
	•	Standard_E8ds_v7 failed to boot with a Hypervisor Generation mismatch — v7-series sizes require Gen2 images, but the VM’s image reference was Gen1. Fix: moved to a Gen1-compatible family (Edsv4, later E16s_v5).
8. ARM vs. x64 trap
az vm list-skus surfaced some sizes (E16ps_v5, E16pds_v5) as available with no restrictions — but the p suffix denotes ARM-based (Azure Cobalt) processors, which cannot run the x64 Windows installer binaries required by the appliance software. Confirmed and avoided via CLI before attempting deployment.
Key Resource Identifiers
	•	Subscription ID: 9b96a1c5-2501-4714-9009-28c82098129c
	•	Resource groups: rg-migrate-source-jeremiah, rg-migrate-target-jeremiah
	•	Discovery appliance VM: vmm-al-jeremiah (Standard_D8s_v3)
	•	Replication appliance VM: vm-mig-repl-jeremiah (Standard_E16s_v5)
	•	Recovery Services Vault: Migrate-Project-Jeremiah-MigrateVault-647598466



[Part 3: Appliance Registration, Discovery & Assessment](https://github.com/JeremiahBTech12/Azure-Lift-Shift-Migration/blob/main/Part%203%3A%20Appliance%20Registration%2C%20Discovery%20%26%20Assessment/README.md) 
