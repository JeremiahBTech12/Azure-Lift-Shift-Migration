Appliance Registration, Discovery & Assessment
This folder covers installing and registering the Azure Migrate discovery and replication appliances, validating the EC2 instance as a discovery source, and generating the initial assessment. This phase surfaced several installer-level bugs and false-negative errors that had nothing to do with the underlying infrastructure being wrong.
What’s Documented Here
	•	Discovery appliance install (AzureMigrateInstaller.ps1) and registration on vmm-al-jeremiah
	•	Replication appliance install (DRInstaller.ps1) and registration on vm-mig-repl-jeremiah, registered as app-rep-jb1
	•	Discovery source configuration and credential validation against the EC2 instance
	•	Assessment generation (~100% Azure-ready, ~$39.6/mo baseline estimate)
Issues Encountered & Fixes
1. MSI install failure over an RDP-redirected drive
Running the installer directly from \\tsclient\... (a drive mapped through the RDP session’s local resource redirection) caused repeated MSI error 1645 during setup — a known class of issue where Windows Installer doesn’t reliably handle UNC/redirected-drive paths for certain install actions.
Fix: copied the extracted installer to the VM’s local C: drive first, then ran it from there.
2. Portal UI mismatch with lab documentation
Azure Migrate has both a “classic” and a newer “unified” experience, and screens referenced in written lab guidance (Discover, Create assessment) didn’t match what the newer UI showed. Fix: used the “click here” link on the Migration and modernization tool’s Overview page to switch to the classic experience, which matched the documented flow.
3. “This host has already been used as DR Appliance” — false-positive on a fresh VM
The replication installer (DRInstaller.ps1) refused to proceed on the first replication appliance VM, even though the replication agent had never actually been installed on it. Registry and log investigation (HKLM:\SOFTWARE\Microsoft\Azure Site Recovery Appliance...) didn’t turn up a clean way to clear the flag.
Root cause, found on the second attempt: running the general-purpose AzureMigrateInstaller.ps1 before DRInstaller.ps1 writes appliance-registration markers that the DR installer’s “already registered” check reads — even if the DR components themselves were never installed.
Fix (process change): for a VM that will be a dedicated replication appliance, run only DRInstaller.ps1. Do not run the general Azure Migrate installer first. The second attempt, following this rule, succeeded cleanly on the first try.
4. Azure AD login loop during registration
The appliance configuration manager repeatedly looped back to the login prompt when authenticating to Azure AD, without a clear error.
Diagnosis: w32tm /query /status showed a broken/unsynced time source — Azure AD token validation is time-sensitive and fails silently when the VM’s clock has drifted.

Fix: (Powershell)
'''
w32tm /unregister
w32tm /register
w32tm /resync /force
'''

5. False-negative registration error in the GUI
The appliance configuration manager’s registration script (PS_Configurator.ps1) reported failure in its GUI wrapper. Running the same underlying command manually showed it had actually returned "ErrorCode": "Success" — the GUI was misinterpreting an ApplianceComponentAlreadyRegistered internal transition as a failure rather than a no-op success.
Fix: ran the configurator script directly to confirm the true result, then proceeded past the misleading GUI error.
6. Stale IP in discovery source credentials
See the AWS Infrastructure README (Elastic IP fix) — this surfaced here first, as an “unable to connect – bad credentials” error on an already-validated discovery source, and was traced to IP drift rather than a credentials problem.
Key Resource Identifiers
	•	Discovery appliance: app-mg-jeb
	•	Replication appliance: app-rep-jb1
	•	Discovered server: EC2AMAZ-UK30H1V
	•	Discovery source credential: ec2winadmin / ec2winadmin2


[Part 4: Replication & Cutover](https://github.com/JeremiahBTech12/Azure-Lift-Shift-Migration/blob/main/Part%204%3A%20Replication%20%26%20Cutover/README.md)
