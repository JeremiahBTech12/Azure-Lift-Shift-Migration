# AWS EC2 to Azure Using Migrate
![Terraform](https://img.shields.io/badge/Terraform-7B42BC?style=flat&logo=terraform&logoColor=white)
![AWS](https://img.shields.io/badge/AWS-FF9900?style=flat&logo=amazonaws&logoColor=white)
![Azure](https://img.shields.io/badge/Azure-0078D4?style=flat&logo=microsoftazure&logoColor=white)
![PowerShell](https://img.shields.io/badge/PowerShell-5391FE?style=flat&logo=powershell&logoColor=white)
![Azure Migrate](https://img.shields.io/badge/Azure_Migrate-0078D4?style=flat&logo=microsoftazure&logoColor=white)

## Video walkthrough: 
Jeremiah Brown | Azure Cloud Engineer | Linkedin: https://www.linkedin.com/in/jeremiah-brown12/

## Project Overview

###
Cloud migration is one of the most frequent and highest-value projects in cloud engineering. Organizations shift workloads between providers for reasons ranging from cost optimization and compliance requirements to platform consolidation or mergers that leave them running infrastructure across multiple clouds. Migrations from AWS to Azure are a routine part of this landscape.

Azure Migrate is Microsoft’s built-in tool for handling this process. It identifies source machines, evaluates whether they’re ready to move, keeps their disks synced through continuous background replication, and executes the final cutover with as little downtime as possible. 
end-to-end migration of a Windows Server workload from AWS EC2 to Azure, using Azure Migrate, Azure Site Recovery, and Terraform to move it across clouds.

This project covers the complete migration lifecycle — provisioning infrastructure, discovering and assessing the source machine, replicating it, and executing cutover — along with troubleshooting the real-world issues that come up when migrating across clouds without private connectivity in place.
## Architecture Flow
 <img width="2823" height="3544" alt="migration-architecture aws to azre diagram 2" src="https://github.com/user-attachments/assets/7ffaec1f-ae6b-43e4-b50c-beafa1a42bbc" />

###

##  What This Project Demonstrates

###
• Cross-Cloud migration from AWS to Azure using Azure Migrate and Azure Site Recovery — full end-to-end pipeline from discovery through a successful test migration

• Infrastructure as Code on both clouds — separate Terraform roots for AWS (EC2, VPC, security groups, IAM) and Azure (VNets, appliance VMs, storage, Recovery Services Vault)

• WinRM and Windows Firewall configuration for agentless OS-level discovery — including resolving the default WinRM firewall scope restriction that blocks cross-subnet connections

• Disk replication using Azure Site Recovery (EC2 EBS volume → Azure managed disk), reaching a healthy, protected replication state and a working test migration

• Manual Mobility Service installation and registration for a cross-cloud scenario with no VPN or private connectivity between AWS and Azure, including direct patching of agent configuration files

• Real-world troubleshooting across multiple distinct troubleshooting issues, spanning:

-  Azure VM quota limits 

- An undocumented bug where the Mobility Service agent hardcodes the replication appliance’s private Azure IP into its runtime config and bypasses DNS — discovered, diagnosed, and worked around through direct log analysis and file patching

- A missing NSG inbound rule (port 9443) blocking the replication data channel despite correct OS-level firewall rules

- EC2 public IP drift after instance restarts, resolved by attaching an Elastic IP

- Hands-on diagnosis using PowerShell and Azure CLI — service status checks, firewall rule inspection, NSG rule auditing, TCP connectivity testing (Test-NetConnection), and live log analysis (svagents logs) to isolate network-layer vs. application-layer failures


## Important Design Decisions

###

- Two resource groups — staging and target. Migration infrastructure (discovery and replication appliances, replication storage cache, Recovery Services Vault) lives in rg-migrate-source-jeremiah, separate from the migrated VM’s home in rg-migrate-target-jeremiah. This keeps the temporary migration tooling cleanly separable from the actual workload once cutover happens.

- No VPN between clouds. All traffic — discovery, replication, and the mobility agent’s data channel — flows over the public internet using the EC2 instance’s public IP (stabilized with an Elastic IP after early testing revealed it changed on every restart). This kept the lab cost-effective and infrastructure-light, but it’s also the direct cause of the majority of the hard problems in this project: Azure Migrate’s tooling assumes private connectivity exists, and getting it to work without one meant working around defaults at almost every layer — DNS resolution, appliance-to-source routing, and the mobility agent’s own hardcoded IP behavior.

- Non-overlapping CIDRs. AWS VPC uses 10.0.0.0/16; Azure VNet uses 10.1.0.0/16. Kept deliberately distinct so no routing conflict would arise if private connectivity were added later.

- WinRM automated via user_data. The EC2 instance’s user_data script sets the Administrator password at first boot, but WinRM itself still required manual configuration afterward (winrm quickconfig) plus a firewall scope fix, since the default WinRM firewall rule only permits connections from the same local subnet — a gap not covered by initial provisioning.

- Manual Mobility Service installation, with direct config patching. Push-installation from the replication appliance consistently failed in this cross-cloud setup: the appliance’s discovered-server record stores the source machine’s OS-reported private IP, which is unreachable from Azure without a VPN. Manual installation on the EC2 instance was required, and even then the agent’s own generated configuration files hardcoded the replication appliance’s private Azure IP and bypassed DNS entirely — an undocumented behavior that required directly patching the agent’s JSON config with the appliance’s public IP before registration would succeed.

  
## Tools and Services Used
###
| Category | Tools |
|----------|-------| 
| Infrastructure as Code | Terraform | 
| Source cloud | AWS - EC2, VPC, IAM, Security Groups, Elastic IP | 
| Target cloud | Azure - Azure Migrate, Azure Site Recovery, Recovery Services Vault, VNet, Network Security Groups, Managed Disks | 
| OS | Windows Server 2022 | 
| Scripting/CLI | PowerShell, Azure CLI, AWS CLI | 
| Migration tool | Azure Migrate (discovery, assessment, appliance managment) | 
| Replication engine | Azure Site Recovery (Mobility Service agent, replication appliance) |
| Networking/Connectivity | Public internet (no VPN) -WinRM, RDP, HTTPS/9443 replication traffic |


## Prerequisites

###
- AWS account with programmatic access
- Azure subscription with sufficient vCPU quota
- Terraform installed
- AWS CLI installed and configured
- Azure CLI installed and authenticated
- RDP access capability for Windows VMs
- Refer to each phase README for detailed prerequisites and deployment steps.
  


## Estimated Cost
Costs reflect a short-lived lab environment with resources running only during active testing and migration.

###
|Resource |Estimated Cost | 
|----------------------------------------------------|------------------------------------------| 
|EC2 t3.large (Windows Server 2022) |~$0.1072/hour | 
|Discovery appliance – Standard_D8s_v3 |~$0.7520/hour |
|Replication appliance – Standard_E16s_v5 |~$1.7440/hour | 
|Replication appliance OS disk (650GB, Standard LRS) |(~$0.44/day while it exists)| 
|Storage account – replication cache (~30–50GB LRS) |~$0.02–0.04/day | 
|Target VM post-cutover – Standard_D2s_v3 |~$0.1880/hour | 
|**Total for a full day with all appliances running**|**~$50–55** |

##
Destroy all resources immediately after completing the lab. See teardown instructions in each phase README.


## Refer to each phase README for detailed walkthroughs, configurations, and troubleshooting.
[Part 1: AWS Infrastructure](https://github.com/JeremiahBTech12/Azure-Lift-Shift-Migration/blob/main/Part%201%3A%20AWS%20Infrastructure/README.md) [Part 2: Azure Infrastructure](https://github.com/JeremiahBTech12/Azure-Lift-Shift-Migration/blob/main/Part%202%3A%20Azure%20Infrastructure/README.md) [Part 3: Appliance Registration, Discovery & Assessment](https://github.com/JeremiahBTech12/Azure-Lift-Shift-Migration/blob/main/Part%203%3A%20Appliance%20Registration%2C%20Discovery%20%26%20Assessment/README.md) [Part 4: Replication & Cutover](https://github.com/JeremiahBTech12/Azure-Lift-Shift-Migration/blob/main/Part%204%3A%20Replication%20%26%20Cutover/README.md)
