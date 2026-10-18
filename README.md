# From Windows Home to Hyper-V

**My journey building a SUSE & Ubuntu Linux home lab**

*A hands-on tutorial covering Windows Home-to-Pro upgrade, VirtualBox freezes, Hyper-V setup, virtual disks, and lessons learned.*

> **Project status:** SUSE OS installation has not yet been confirmed. The external SSD reported **Warning / Full Repair Needed**; preserve important files and address drive health before continuing.

---

# At a glance

| **My environment**     | **Details**                                                                          |
|------------------------|--------------------------------------------------------------------------------------|
| Windows host           | Upgraded from Windows Home to Windows Pro                                            |
| Hardware               | Intel Core i5; 16 GB RAM                                                             |
| External disk          | 2 TB Seagate One Touch, D:, exFAT; contains important files                          |
| Original platform      | Oracle VirtualBox, SUSE installer stalled                                            |
| New platform           | Microsoft Hyper-V; VM named suse; Generation 2                                       |
| Latest confirmed state | Hyper-V VM reported Running / Operating normally; SUSE OS installation not confirmed |

# The journey in six milestones

- 01 Planned a Linux lab for Linux Administration → Azure Administrator → DevOps.

- 02 Prepared Ubuntu and SUSE ISO files on my external D: drive.

- 03 Tried VirtualBox; SUSE Agama installation became unresponsive.

- 04 Upgraded Windows Home to Pro and enabled the built-in Hyper-V feature.

- 05 Created a Generation 2 SUSE VM and attached a virtual hard disk on D:.

- 06 Discovered an AVHDX disk chain and a Full Repair Needed warning on D:; paused for data safety.

Status note: This guide records the verified setup and safe next steps; it does not present the SUSE OS installation or SSD repair as complete.

# 1 Why I built this lab

My goal was to refresh Linux administration from the fundamentals, practise with different operating systems, and later apply the skills to Azure and DevOps. I chose SUSE and Ubuntu because they let me compare installation methods, package managers, systemd, network configuration, storage and enterprise administration workflows.

## The initial storage plan

I had a separate 2 TB external SSD at D:. It was formatted as exFAT and already contained important files. I deliberately chose not to format the SSD. I created a lab folder with ISO images and virtual machine files.

> D:\VirtualBox-Lab\\
>
> ISO\\ (Ubuntu and SUSE installers)
>
> VM's\\ (original VirtualBox machines)
>
> D:\Hyper-V\\ (Hyper-V virtual disk files)

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th><p><strong>SAFETY RULE</strong></p>
<p>The physical D: SSD and a VHDX file stored on D: are not the same disk. A Linux installer may partition a virtual VHDX presented to the guest, but it must never be given raw access to the physical D: drive.</p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

# 2 My first attempt: VirtualBox

I created a SUSE VM using a downloaded ISO. Windows displayed the ISO with a WinRAR icon, but the filename was an ISO, so VirtualBox could boot it directly without extracting it.

The Agama installer opened in its browser interface. At one point it showed a 40 GB VBOX HARDDISK with GPT partitions. The initial layout included an 8 MB BIOS Boot partition, a Btrfs root filesystem and 2 GB swap. To practise modern booting, I aimed to use UEFI and an EFI System Partition instead.

After the installer started, the progress page stopped changing. I checked CPU, disk and network activity, but the browser-based installation remained frozen and even switching to the installer terminal did not respond. I therefore chose to try another hypervisor instead of endlessly reinstalling.

## What the VirtualBox logs showed

| **Evidence**                    | **What I learned**                                                                        |
|---------------------------------|-------------------------------------------------------------------------------------------|
| EFI: relocate module            | Virtual firmware activity was visible; this alone did not establish a successful OS boot. |
| NAT: Link up                    | The virtual network link came up, but it did not prove that repositories were reachable.  |
| VMSVGA: 3D disabled             | An informational graphics message, not evidence of a fatal error.                         |
| IPv6 MTU / async-flush warnings | Warnings worth investigating, but insufficient to identify one confirmed root cause.      |

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th><p><strong>LESSON LEARNED</strong></p>
<p>A frozen installer does not necessarily mean the hypervisor is broken. Gather the install stage and logs first. Changing hypervisors can be a useful isolation test, but it is not a proven fix.</p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

# 3 Upgrading Windows Home to Pro

I moved from Windows Home to Windows Pro so I could use the built-in Hyper-V role. I confirmed in our troubleshooting conversation that the upgrade was completed. I did not record which license, purchase or entitlement method I used, so this document does not claim a particular free upgrade method.

## Tutorial: verify the Windows edition and activation

> **1.** Open Settings → System → Activation (Windows 11) or Settings → Update & Security → Activation (Windows 10).
>
> **2.** Verify that the installed edition is Pro and review its activation status.
>
> **3.** For a legitimate edition upgrade, use an existing valid Pro product key through Change product key, or follow the Microsoft Store upgrade process and associated licensing instructions.
>
> **4.** Restart if Windows requests it, then recheck the edition and activation status.
>
> winver
>
> \# Or open the Activation page from Run:
>
> ms-settings:activation

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th><p><strong>LICENSING NOTE</strong></p>
<p>Windows Pro is not generally a free upgrade from Home. A valid Pro digital entitlement, product key or eligible organizational/education licensing is required. Do not publish or share activation keys in the tutorial.</p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

# 4 Enabling Hyper-V

After completing the Windows Pro upgrade, I enabled Hyper-V successfully.

## Option A — Windows Features (GUI)

> **1.** Press Windows + R, enter optionalfeatures and press Enter.
>
> **2.** Expand Hyper-V and check Hyper-V Platform and Hyper-V Management Tools.
>
> **3.** Select OK and restart the computer when prompted.
>
> **4.** Open Start, search for Hyper-V Manager, and launch it.

## Option B — PowerShell (Administrator)

> Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All

Restart Windows if prompted. Verify the feature and VM tooling with these commands:

> Get-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V
>
> Get-Command Get-VM
>
> Get-VMSwitch

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th><p><strong>COMPATIBILITY NOTE</strong></p>
<p>Once the Windows hypervisor is enabled, VirtualBox or VMware Workstation may behave differently depending on their versions and configuration. I therefore used Hyper-V for this stage of my Linux lab.</p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

# 5 Creating the SUSE virtual machine in Hyper-V

The following steps reproduce the VM creation workflow. The values reflect my lab where confirmed. Storage recommendations are marked separately from what I actually used.

## Step A — launch the New Virtual Machine Wizard

> **1.** Open Hyper-V Manager and select the local host.
>
> **2.** From Actions, choose New → Virtual Machine.
>
> **3.** Give the VM the name suse. If you change the VM configuration location, choose a healthy NTFS volume with enough space.
>
> **4.** Choose Generation 2 for UEFI-capable virtual hardware.
>
> **5.** Set startup memory to 4096 MB (4 GB). For my 16 GB host, this was a practical starting point.
>
> **6.** Choose Default Switch for basic network access, if available.
>
> **7.** Create a new VHDX or attach an existing suitable VHDX. Use a unique filename to avoid conflicts.
>
> **8.** For installation options, select Install an operating system from a bootable image file and browse to the SUSE ISO.
>
> **9.** Finish the wizard. While the VM is powered off, open Settings → Processor and assign 2 virtual processors if appropriate.

| **Hyper-V setting**      | **My recorded value**                                                             |
|--------------------------|-----------------------------------------------------------------------------------|
| VM name                  | suse                                                                              |
| Generation               | Generation 2                                                                      |
| RAM                      | 4096 MB                                                                           |
| Virtual network          | Default Switch                                                                    |
| VM status later observed | Running; Operating normally; configuration version 12.0                           |
| Disk                     | 100 GB virtual disk later created/attached on D:; attached path observed as AVHDX |

## Step B — firmware and virtual DVD drive

> **1.** With the VM off, open suse → Settings → SCSI Controller → DVD Drive.
>
> **2.** Choose Image file, browse to the SUSE ISO, and Apply.
>
> **3.** Under Firmware, place the DVD drive ahead of the virtual hard disk for installation boot.
>
> **4.** Under Security, select the supported Linux Secure Boot template if appropriate; for isolated diagnosis, Secure Boot may be temporarily disabled.
>
> **5.** Connect → Start; review the SUSE boot menu before installation.

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th><p><strong>INSTALLATION BOUNDARY</strong></p>
<p>The VM was created and shown as Running, but I did not obtain confirmed evidence of a completed SUSE operating-system installation. A Hyper-V status of Operating normally describes VM operation, not successful guest OS installation.</p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

# 6 Creating and attaching virtual hard disks

I wanted to practise enterprise Linux storage tasks, so I created a 100 GB virtual hard disk on the external SSD and attached it to the SUSE VM. In the earlier Hyper-V wizard, the disk was described as fixed size; later commands revealed that the disk attached to the VM was an AVHDX differencing disk. The effective disk chain needs verification before any cleanup.

## Tutorial: create an additional VHDX through Hyper-V

> **1.** In Hyper-V Manager, choose Actions → New → Hard Disk.
>
> **2.** Select VHDX format. For future practice disks, choose Dynamically expanding unless fixed allocation is specifically needed.
>
> **3.** Choose a distinctive filename, such as suse-data-01.vhdx, and a healthy supported storage location.
>
> **4.** Choose a capacity, such as 20 GB for a storage lab or 100 GB for the OS disk; finish the wizard.
>
> **5.** Power off the VM, open suse → Settings → SCSI Controller → Hard Drive → Add, and attach the existing VHDX.
>
> **6.** Before changing any data inside Linux, identify all disks by size and purpose with lsblk; never assume /dev/sda always means the OS disk.

## Tutorial: verify Hyper-V paths before modifying files

> Get-VM -Name "suse"
>
> Get-VMHardDiskDrive -VMName "suse" \|
>
> Select-Object VMName, Path, ControllerType
>
> Get-Volume -DriveLetter D

These are the exact checks I ran. In my session, the disk path was not a plain VHDX. It pointed to a differencing disk:

> D:\Hyper-V\New Virtual Hard Disk_829B7788-50BF-402D-936A-3E7CD4CED5E0.avhdx

An AVHDX file usually belongs to a Hyper-V checkpoint or differencing chain. It must not be deleted or detached independently without understanding its parent relationship.

## The filename-conflict problem I encountered

When creating a virtual disk, Hyper-V returned:

> Failed to create the virtual hard disk.
>
> The file exists. (0x80070050)

A safe first response is to check the selected folder for a file with the same name and choose a unique new VHDX filename. Never delete an unknown disk merely to clear the error; it may be attached to an existing VM.

> Get-ChildItem "D:\Hyper-V" -File \|
>
> Where-Object { \$\_.Extension -in ".vhdx", ".avhdx" } \|
>
> Select-Object Name, Length, LastWriteTime

# 7 The critical discovery: external disk health warning

My final PowerShell check showed that the VM was running with 4 GB memory, but the external D: volume was not healthy. This changed the priority from “finish Linux installation” to “protect my existing files.”

| **Get-Volume property** | **Observed result** |
|-------------------------|---------------------|
| Drive letter / label    | D: / One Touch      |
| File system             | exFAT               |
| Volume size             | 1.82 TB             |
| Free space              | 1.39 TB             |
| HealthStatus            | Warning             |
| OperationalStatus       | Full Repair Needed  |

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th><p><strong>STOP AND PROTECT DATA</strong></p>
<p>Do not keep writing to D:, formatting the disk, deleting AVHDX/VHDX files, or running repair switches before important files are safely backed up. “Full Repair Needed” is a Windows volume-health warning, not a confirmed diagnosis of failed SSD hardware.</p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

## A safe recovery sequence

> **1.** If the guest operating system is responsive, shut it down normally. If the guest is still only a frozen installer, plan a controlled VM power-off and accept that the incomplete installation might need repeating.
>
> **2.** Do not unplug the external drive while Hyper-V still has disk files open.
>
> **3.** Back up important existing files from D: to a separate healthy disk and verify the copies. If Windows reports read errors or disconnections, stop and reassess instead of forcing repairs.
>
> **4.** Once backups are secured and the VM is off, run a read-only filesystem inspection: chkdsk D: (no /f or /r).
>
> **5.** Inspect Hyper-V checkpoints and the AVHDX parent disk relationship before any move, merge or deletion.
>
> Get-VM -Name "suse" \| Select-Object Name, State
>
> Get-VMSnapshot -VMName "suse"
>
> Get-VMHardDiskDrive -VMName "suse" \| ForEach-Object {
>
> Get-VHD -Path \$\_.Path \|
>
> Select-Object Path, ParentPath, VhdType, Size
>
> }
>
> \# Only after backup and a stopped VM:
>
> chkdsk D:

Windows may report filesystem errors; any actual repair operation should be considered only after data protection is verified. Manual AVHDX deletion can break the VM checkpoint chain.

## What I would change for a reliable long-term lab

- Use a healthy NTFS volume for active Hyper-V VM files and VHDX/AVHDX chains. Keep existing important data and installation ISO files separate.

- If I need VM storage on the external SSD, prepare a dedicated supported volume only after a verified backup and planned migration. Do not reformat the current exFAT volume impulsively.

- Leave enough host free RAM and SSD capacity, and avoid USB disconnections while a VM is running.

# 8 My next steps and study outcomes

## Verified, pending and unresolved

| **Item**                              | **Status**                                             |
|---------------------------------------|--------------------------------------------------------|
| Windows Home → Pro upgrade            | Completed, specific activation path not documented     |
| Hyper-V feature enabled               | Completed                                              |
| Hyper-V SUSE VM created               | Confirmed via Get-VM                                   |
| Virtual disk attached                 | Confirmed via Get-VMHardDiskDrive; AVHDX path observed |
| SUSE OS installed successfully        | Not confirmed                                          |
| External D: filesystem healthy        | No — Full Repair Needed warning recorded               |
| Existing external-SSD files protected | Backup completion not recorded                         |

## My immediate checklist

- [ ] Stop VM activity safely and keep the external SSD attached until all disk handles are released.

- [ ] Copy important D: files to another healthy disk and verify the backup.

- [ ] Inspect the D: filesystem and checkpoint chain without deleting or repairing anything yet.

- [ ] Choose healthy NTFS storage for the next SUSE installation attempt.

- [ ] Finish SUSE, confirm UEFI boot and inspect GPT, Btrfs and swap.

- [ ] Install Ubuntu Server 24.04 as a separate Linux VM and compare administration commands.

- [ ] Add a separate disposable VHDX for Linux partitions, filesystems and LVM practice.

## What this journey taught me

Building a home lab is not just about getting an operating system installed. I practised distinguishing a physical drive from a virtual hard disk, learned why OS installers can freeze, discovered the difference between BIOS and UEFI partition layouts, explored Hyper-V generations and VHDX files, and saw why storage-health checks must come before disk operations. My biggest lesson is to verify the environment and protect data before retrying an installation.

# References and further reading

**Microsoft Support — Upgrade Windows Home to Pro** [<u>Open official documentation</u>](https://support.microsoft.com/en-us/windows/deployment/install-upgrade/upgrade-windows-home-to-windows-pro)

**Microsoft Learn — Install Hyper-V on Windows** [<u>Open official documentation</u>](https://learn.microsoft.com/en-us/windows-server/virtualization/hyper-v/get-started/install-hyper-v?pivots=windows&tabs=gui)

**Microsoft Learn — Create a virtual machine in Hyper-V** [<u>Open official documentation</u>](https://learn.microsoft.com/en-us/windows-server/virtualization/hyper-v/get-started/Create-a-virtual-machine-in-Hyper-V)

**Microsoft Learn — Using checkpoints** [<u>Open official documentation</u>](https://learn.microsoft.com/en-us/windows-server/virtualization/hyper-v/checkpoints)

**Microsoft Learn — AVHDX / checkpoint troubleshooting** [<u>Open official documentation</u>](https://learn.microsoft.com/en-us/troubleshoot/windows-server/virtualization/hyper-v-snapshots-checkpoints-differencing-disks)

**Microsoft Learn — chkdsk command** [<u>Open official documentation</u>](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/chkdsk)

Editorial note: The chronology, logs, error codes and settings above are from my own documented lab session. Generic tutorial steps and safety recommendations were cross-checked against Microsoft documentation. No license key, password or personal files are included.
