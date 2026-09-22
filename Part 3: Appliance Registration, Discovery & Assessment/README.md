# Part 3 — Appliance Registration, Discovery & Assessment

By this phase, all infrastructure is already deployed. The EC2 instance is running in AWS, and both Azure resource groups are in place from the AWS and Azure infrastructure phases. Phase 3 is where the migration workflow actually begins: the Azure Migrate appliances are registered, connectivity between AWS and Azure is validated, and discovery is performed against the source EC2 instance.

This phase bridges the gap between infrastructure provisioning and actual migration by establishing communication between the two environments and confirming the source machine can be successfully assessed.

> **Important:** Discovery depends on both network connectivity and OS-level configuration. Even with every port open, discovery will fail if WinRM or the local Windows Firewall aren't configured correctly — and the failure mode isn't always obvious from the portal's error messages.

## Video walkthrough:

## What This Phase Covers

| Step | What Happens |
|---|---|
| Register discovery appliance | Discovery appliance (`app-mg-jeb`) bound to the Azure Migrate project via registration key |
| Register replication appliance | Replication appliance (`app-rep-jb1`) registered separately, for the disk replication workflow used in Phase 4 |
| Complete WinRM configuration | `user_data` sets the EC2 Administrator password at first boot; WinRM enablement and firewall scoping are configured manually afterward |
| Add EC2 credentials | Windows credentials (`ec2winadmin`) added to the discovery appliance |
| Validate discovery source | Connectivity and credentials tested from the appliance to the EC2 instance |
| Start discovery | Appliance connects to EC2 over WinRM and collects inventory |
| Run assessment | Azure Migrate evaluates readiness and estimates cost |

## Prerequisites

- EC2 instance running, accessible via RDP (see `1-aws-infrastructure/`)
- Azure staging resource group and both appliance VMs deployed (see `2-azure-infrastructure/`)
- RDP access confirmed to the EC2 instance and both appliance VMs

## File Structure

```
3-appliance-registration-discovery-assessment/
└── README.md
```

Every fix below was applied as an interactive PowerShell command during live troubleshooting. The exact commands are documented inline so they can be reproduced directly.

## WinRM — What's Already Done vs. What Still Needs Configuration

The EC2 `user_data` block runs automatically on first boot, but only handles one piece of the puzzle:

```powershell
# Handled automatically by user_data at EC2 launch
net user Administrator "${var.admin_password}"   # Sets admin password
```

Everything else — actually enabling remote management — has to be done manually after the instance is up:

| Setting | Why It's Still Needed |
|---|---|
| WinRM service enabled | Not on by default on a stock Windows Server 2022 AMI — Azure Migrate's agentless discovery depends on it |
| Windows Firewall remote address | `winrm quickconfig` restricts WinRM access to the **local subnet only** by default — must be widened to allow the Azure appliance's IP range |
| Security Group port 5985 | The AWS Security Group has no inbound rule for WinRM until one is added explicitly |

**Key insight:** discovery failing doesn't always mean discovery is misconfigured — in this project, two of the three layers above (firewall scope, missing SG port) produced discovery failures that looked identical to a WinRM configuration problem, and each needed to be ruled out separately.

## Step 1 — Enable and Configure WinRM on the EC2 Instance

RDP into the EC2 instance, open PowerShell as Administrator, and run:

```powershell
winrm quickconfig -quiet
```

At this point, discovery validation from the appliance still failed. The error text pointed at the real cause: *"the WinRM firewall exception for public profiles limits access to remote computers within the same local subnet."*

```powershell
Set-NetFirewallRule -DisplayName "Windows Remote Management (HTTP-In)" -RemoteAddress Any
```

Finally, the Security Group itself needed an inbound rule added for TCP 5985 (done via Terraform in `1-aws-infrastructure/`, not shown here since it's not a PowerShell fix).

## Step 2 — Register the Discovery and Replication Appliances

Registering the two appliances surfaced two separate issues, both false-negatives rather than real configuration problems:

**Azure AD login loop.** The appliance configuration manager repeatedly looped back to the login prompt during registration with no clear error. Diagnosed via:

```powershell
w32tm /query /status
```

This showed a broken/unsynced time source — Azure AD token validation is time-sensitive and fails silently when the VM's clock has drifted. Fixed with:

```powershell
w32tm /unregister
w32tm /register
w32tm /resync /force
```

**False-negative registration failure in the GUI.** The configuration manager's own registration step (`PS_Configurator.ps1`) reported failure in its GUI wrapper. Running the exact same command manually surfaced the real result:

```powershell
& 'C:\Program Files\Microsoft Azure Site Recovery Process Server\home\svsystems\bin\PS_Configurator.ps1' `
    -Register `
    -RcmUri https://pod01-rcm1.eus.hypervrecoverymanager.windowsazure.com `
    -ApplianceJsonPath 'C:\ProgramData\Microsoft Azure\Config\appliance.json'
```

This returned `"ErrorCode": "Success"` — the GUI was misreporting an `ApplianceComponentAlreadyRegistered` internal transition as a failure rather than a no-op success. Registration had actually already succeeded.

## Step 3 — Add EC2 Credentials and Validate the Discovery Source

Windows credentials for the EC2 instance were added to the discovery appliance, targeting the instance's public IP. Validation initially failed with "unable to connect — bad credentials" — but the credentials were correct. The EC2 instance's public IP had changed after a stop/start cycle (no Elastic IP was attached yet at this point in the project). Fixed by updating the discovery source with the current public IP from the AWS console, and permanently resolved afterward by attaching an Elastic IP (see `1-aws-infrastructure/`).

## Step 4 — Start Discovery and Run Assessment

Once WinRM, appliance registration, and the discovery source were all validated, discovery and assessment ran through the standard Azure Migrate workflow with no further issues: the EC2 instance (`EC2AMAZ-UK30H1V`) appeared in inventory, and the resulting assessment showed **~100% Azure-ready** with an estimated cost of **~$39.6/month**.



## Troubleshooting

### MSI install failure over an RDP-redirected drive
Running the appliance installer directly from `\\tsclient\...` (the RDP session's redirected local drive) caused repeated MSI error 1645.
**Fix:** copy the extracted installer to the VM's own local disk first, then run it from there.
```powershell
Copy-Item -Path "\\tsclient\C\Users\jerem\Downloads\AzureMigrateInstaller" -Destination "C:\AzureMigrateInstaller" -Recurse
cd "C:\AzureMigrateInstaller"
.\AzureMigrateInstaller.ps1
```

### Replication appliance CPU validation kept failing across multiple VM sizes
**Root cause:** the appliance's prerequisite check validates **physical CPU cores**, not vCPUs — and several Azure VM families report roughly half their advertised vCPU count as physical cores.
**Fix:** verify actual physical cores directly rather than trusting the size label:
```powershell
Get-WmiObject -Class Win32_Processor | Select-Object NumberOfCores, NumberOfLogicalProcessors
```
This confirmed, for example, that `Standard_E8s_v3` (8 vCPUs) reports only 4 physical cores — insufficient for the appliance's 8-core minimum — while `Standard_E16s_v3`/`Standard_E16s_v5`-class sizing reliably clears it. (Full sizing/quota saga documented in `2-azure-infrastructure/`.)



## Key Resource Identifiers

###
	•	Discovery appliance: app-mg-jeb
	•	Replication appliance: app-rep-jb1
	•	Discovered server: EC2AMAZ-UK30H1V
	•	Discovery source credential: ec2winadmin / ec2winadmin2


[Part 4: Replication & Cutover](https://github.com/JeremiahBTech12/Azure-Lift-Shift-Migration/blob/main/Part%204%3A%20Replication%20%26%20Cutover/README.md)
