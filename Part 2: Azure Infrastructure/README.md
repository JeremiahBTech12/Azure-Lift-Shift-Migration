## Azure Infrastructure

###
This folder provisions the target-side Azure resources: resource groups, networking, and the two appliance VMs (discovery and replication) that Azure Migrate uses to talk to the AWS source environment. 

## What’s Deployed

###
Two resource groups, deliberately separated by purpose:

	•	rg-migrate-source-jeremiah — staging resources: appliances, replication storage cache, Recovery Services Vault
	•	rg-migrate-target-jeremiah — home for the final migrated VM
	
Keeping these separate means the staging environment can be torn down cleanly after cutover without touching the actual workload.
After cutover, the migration infrastructure - appliances, storage cache, vault replication state - can be destroyed cleanly without touching the migrated VM. This is the ideal pattern for any real-world migration engagement and makes teardown significantly safer.


| Resource | Details |
|---|---|
| Virtual Network | `10.1.0.0/16` — target network |
| Subnet | `10.1.1.0/24` (snet-migrate) |
| Azure Migrate Project | Created manually in the Azure portal — control plane for discovery, assessment, and replication |
| Storage Account | Standard LRS — replication cache during disk sync |
| Log Analytics Workspace | Stores discovery data and performance metrics |
| Recovery Services Vault | Orchestrates replication via Azure Site Recovery |
| Migration Appliance VM | Windows Server VM used for discovery and Azure Migrate appliance setup — `vmm-al-jeremiah`, Standard_D8s_v3 (8 vCPU, 32 GiB) |
| Replication Appliance VM | High-capacity VM `vm-mig-repl-jeremiah`, ultimately Standard_E16s_v5 (16 vCPU, 128 GiB), used for replication processing |
| Network Interfaces | Attached to appliance VMs with public IPs for connectivity |
| Public IPs | Enables access to appliance VMs |
| Network Security Groups | Allow RDP and required inbound traffic for appliance access |

##	Prerequisites
###
- Azure subscription
- Azure CLI installed and authenticated
- Terraform installed
- Part 1 deployed and EC2 instance running

## File Structure
### 
```
part-2-azure-infrastructure/
├── main.tf                    # All Azure resources
├── variables.tf               # Input variable definitions
├── outputs.tf                 # Resource group, vault, storage outputs
├── terraform.tfvars           # Your variable values (subscription, region, network ranges, VM sizes, credentials, etc.)
└── README.md
```

## Deploy
This phase deploys both the Azure infrastructure and the appliance VMs required for discovery and replication. 

- Initialize Terraform:

terraform init

- Review the plan:

terraform plan

- Deploy:

terraform apply
Deployment takes approximately 5–10 minutes depending on VM provisioning time.


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
	•	Family-specific quota (Standard ESv3: 10/10) was maxed separately from the regional total, and flagged “Not adjustable” 


## Key Resource Identifiers

###
	•	Subscription ID: 9b96a1c5-2501-4714-9009-28c82098129c
	•	Resource groups: rg-migrate-source-jeremiah, rg-migrate-target-jeremiah
	•	Discovery appliance VM: vmm-al-jeremiah (Standard_D8s_v3)
	•	Replication appliance VM: vm-mig-repl-jeremiah (Standard_E16s_v5)
	•	Recovery Services Vault: Migrate-Project-Jeremiah-MigrateVault-647598466

## Teardown
Important: Do not destroy Part 2 resources until after Part 4 (cutover) is complete and you have stopped replication in the Azure Migrate portal.


### Stop replication first in the Azure portal
### Azure Migrate → Replicating Machines → Stop Replication

### Then destroy
terraform destroy
If terraform destroy fails on the Recovery Services Vault with a "vault is not empty" error:

Go to Azure portal → Recovery Services Vault → rsv-migrate-[yourname]
Click Replication items → delete all items
Click Backup items → delete all items
Retry terraform destroy
Both appliance VMs, their NICs, NSGs, public IPs, and both resource groups are managed by Terraform in this phase.

[Part 3: Appliance Registration, Discovery & Assessment](https://github.com/JeremiahBTech12/Azure-Lift-Shift-Migration/blob/main/Part%203%3A%20Appliance%20Registration%2C%20Discovery%20%26%20Assessment/README.md) 
