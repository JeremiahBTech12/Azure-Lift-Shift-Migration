## AWS Infrastructure

###
This folder provisions the source environment for the migration: a Windows Server 2022 EC2 instance representing a legacy on-premises workload, along with the networking and access configuration required to make it discoverable and reachable by Azure Migrate.

## Video walkthrough: https://www.loom.com/share/7d36a6eed5544a2d81019ca3c59360c2

## What’s Deployed
###
	•	EC2 instance — t3.large (2 vCPU, 8 GiB), Windows Server 2022, deployed via Terraform (main.tf)
	•	VPC — 10.0.0.0/16, deliberately non-overlapping with the Azure VNet (10.1.0.0/16) used later, in case private connectivity is ever added
	•	Security Group — inbound rules for RDP, WinRM, and later, SMB/RPC (see below)
	•	Elastic IP — attached after initial testing revealed the instance’s default public IP changes on every stop/start


## Folder Setup

This lab uses two separate Terraform roots — one for AWS resources, one for Azure resources. Keep them separate so you can destroy each side independently.

Windows (PowerShell):
```powershell
New-Item -ItemType Directory -Path "$HOME\aws-to-azure-migrate"
cd "$HOME\aws-to-azure-migrate"
New-Item -ItemType Directory -Path aws-side, azure-side
New-Item -ItemType File aws-side\main.tf, aws-side\variables.tf, aws-side\outputs.tf, aws-side\terraform.tfvars
New-Item -ItemType File azure-side\main.tf, azure-side\variables.tf, azure-side\outputs.tf, azure-side\terraform.tfvars
```


## How to Deploy

###
 - Initialize Terraform:

terraform init

 - Review the plan:

terraform plan

 - Deploy:

terraform apply

Deployment takes 3–5 minutes. Windows instances take an additional 5 minutes to fully initialize after Terraform completes — wait before attempting RDP.

## Design Decisions

###
No VPN to Azure. All traffic — discovery, replication, and the mobility agent’s data channel — crosses the public internet on the EC2 instance’s public IP. This keeps the lab free of a VPN gateway or peering setup, but it’s also the direct cause of most of the harder problems documented in 4-replication-and-cutover/.
Elastic IP over default public IP. Initially the instance ran with AWS’s default (non-static) public IP. After a routine stop/start cycle changed the IP and broke an already-configured discovery source, an Elastic IP (52.4.247.242) was attached to eliminate that class of failure permanently.

## Issues Encountered & Fixes

###
1. WinRM not enabled by default
Azure Migrate’s agentless discovery relies on WinRM to query the Windows OS remotely. A stock Windows Server 2022 AMI doesn’t have this configured out of the box.
Fix: ran winrm quickconfig -quiet on the instance to enable and configure the WinRM listener.

2. Local Windows Firewall silently scoped to same-subnet only
Even after WinRM was enabled, discovery validation from the Azure-side appliance still failed. The default Windows Remote Management (HTTP-In) firewall rule only permits connections from the local subnet — a restriction that isn’t surfaced anywhere in Azure Migrate’s own error messages, so it looks like a WinRM configuration problem rather than a firewall scope problem.
Fix ran in Powershell:

```powershell
Set-NetFirewallRule -DisplayName "Windows Remote Management (HTTP-In)" -RemoteAddress Any
```

3. Security Group missing the WinRM port
Port 5985 (WinRM HTTP) wasn’t open on the instance’s Security Group, so even with WinRM and the local firewall fixed, nothing outside the VPC could reach it.
Fix: added an inbound rule for TCP 5985 from the Azure appliance’s public IP range.

4. Public IP drift breaking a previously-validated discovery source
After stopping and restarting the instance for a later phase of the project, the discovery source in Azure Migrate started failing with “unable to connect — bad credentials.” The credentials were actually fine — the instance’s public IP had changed (no Elastic IP was attached yet), so Azure was trying to reach a host that no longer existed at that address.
Fix: attached an Elastic IP to the instance and updated the discovery source with the new static address, permanently eliminating IP drift as a failure mode.

5. Additional inbound rules required for the Mobility Service agent
Later in the project (see 4-replication-and-cutover/), enabling replication required additional Security Group rules beyond what discovery needed: SMB (445), RPC endpoint mapper (135), and the dynamic RPC port range (49152–65535), plus enabling File and Printer Sharing and WMI rules in the local Windows Firewall. These were ultimately not sufficient on their own to fix replication — the real blocker was a cross-cloud IP-routing issue documented in the replication README — but they were still legitimate, necessary prerequisites.

## Key Resource Identifiers

###
	•	EC2 Instance ID: i-0fa3d97db560ad288
	•	Elastic IP: 52.4.247.242
	•	Security Group: sg-0a749a021281b54f3
	•	VPC CIDR: 10.0.0.0/16

 [Part 2: Azure Infrastructure](https://github.com/JeremiahBTech12/Azure-Lift-Shift-Migration/blob/main/Part%202%3A%20Azure%20Infrastructure/README.md)
