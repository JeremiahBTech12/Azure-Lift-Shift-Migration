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
End-to-end migration of a Windows Server workload from AWS EC2 to Azure, using Azure Migrate, Azure Site Recovery, and Terraform to move it across clouds.

This project covers the complete migration lifecycle — provisioning infrastructure, discovering and assessing the source machine, replicating it, and executing cutover — along with troubleshooting the real-world issues that come up when migrating across clouds without private connectivity in place.
## Architecture Flow
 <img width="2823" height="3544" alt="migration-architecture aws to azre diagram 2" src="https://github.com/user-attachments/assets/7ffaec1f-ae6b-43e4-b50c-beafa1a42bbc" />

###

##  What This Project Demonstrates

###
• Cross-cloud migration from AWS to Azure using Azure Migrate and Azure Site Recovery — full end-to-end pipeline from discovery through a successful test migration

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


## Design Decisions

###

## Tools and Services Used
###
```

```
## Prerequisites

###
 Before deploying, install and configure:


## Estimated Cost

###

## Verification Checklist & Troubleshooting
###

## Thank you for following along watching me create real-world cloud solutions. This is only part of my full Azure cloud portfolio.
