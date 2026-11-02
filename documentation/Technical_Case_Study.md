# Technical Case Study: Windows 11 Enterprise Imaging & Deployment

**Project 2 | Ian Van Amberg | October–November 2026**  
**Organization:** Northstar Office Solutions — fictional business scenario  
**Environment:** Oracle VirtualBox, self-directed, non-production  
**Outcome:** One reference image captured and successfully deployed to a separate virtual workstation; planned lab validation completed.

[Return to the project README](../README.md)

## 1. Objective and outcome

The scenario was a small business that needed a consistent Windows workstation baseline for new employees. My objective was to prepare a reference installation, generalize and capture it, apply the image to a new virtual disk, and validate the resulting workstation with documented evidence.

I completed the workflow using Windows 11 Enterprise Evaluation **26H2, build 26300.9457, x64**. DISM captured `NS-W11-ENT-BASE-01.wim` at **8,415,340,722 bytes (~7.84 GiB)** and successfully applied image index 1 to the target. The target booted through UEFI, completed OOBE, reached the desktop, retained its support files, and passed the planned operational and integrity checks.

The value of this project is the demonstrated process: prepare, identify disks, generalize, capture, deploy, troubleshoot, and verify. It does not establish rollout speed, fleet-scale reliability, or readiness for a production organization.

## 2. Architecture and tools

| Component | Recorded configuration |
| --- | --- |
| Reference VM | `NS-REF-01`; 4096 MB RAM; 2 vCPUs; 80 GB dynamically allocated system VDI |
| Target VM | `NS-DEPLOY-01`; 4096 MB RAM; 2 vCPUs; separate 80 GB dynamically allocated system VDI |
| Firmware/security configuration | UEFI; Secure Boot enabled in VM settings; virtual TPM 2.0 |
| Image-store disk | `NS-ImageStore-50GB.vdi`; 50 GB; GPT; NTFS; label `NS-IMAGESTORE` |
| Guest OS | Windows 11 Enterprise Evaluation 26H2; build 26300.9457; x64 |
| Boot media | `WinPE_amd64.iso`, created with Windows ADK Deployment Tools and the WinPE add-on |
| Reference preparation | Audit Mode, PowerShell, Sysprep |
| Imaging and partitioning | DISM, DiskPart |
| Boot and recovery | BCDBoot, REAgentC, XCOPY |
| Validation | PowerShell, `manage-bde`, DISM, SFC |
| Network | VirtualBox NAT; cable disconnected during preparation and initial deployment, connected for final network checks |

```text
NS-REF-01                    Separate image storage                 NS-DEPLOY-01
80 GB GPT system disk   -->  50 GB GPT/NTFS VDI                 -->  New 80 GB GPT disk
Audit Mode + Sysprep         NS-W11-ENT-BASE-01.wim                  DISM apply
Offline WinPE capture       Index 1; ~7.84 GiB                      BCDBoot + recovery setup
                                                                   UEFI boot -> OOBE -> validation
```

The image-store VDI was detached from the powered-off reference VM and attached to the target VM. The original reference system disk was not attached to the deployment VM. Guest Additions was retained because both VMs used similar VirtualBox hardware; physical-hardware compatibility was not established.

## 3. Phase 1 — Planning and deployment tooling

I assessed virtualization support, available storage, and host memory before creating the lab. With 16 GB host RAM and competing memory use, I planned to operate one Windows VM at a time.

The preparation work included:

- Obtaining Windows 11 Enterprise Evaluation media and recording a matching SHA-256 comparison.
- Installing VirtualBox, ADK Deployment Tools, and the WinPE add-on.
- Staging x64 WinPE with `copype` and creating the bootable ISO with `MakeWinPEMedia`.
- Organizing phase notes and screenshot evidence.

The first WinPE staging attempt failed to mount its image. DISM diagnostics returned error 740, indicating elevation was required. Reopening the Deployment and Imaging Tools Environment as administrator allowed staging to complete. The ISO was created in Phase 1 and its actual boot was verified in Phase 3.

## 4. Phase 2 — Reference workstation configuration

I installed Windows on `NS-REF-01` and entered Audit Mode from OOBE. The reference baseline included a deployment profile and a support-folder structure:

```text
C:\ProgramData\NS-Deployment\Image_Profile.txt
C:\Support\Docs\Workstation_Readiness.txt
C:\Support\Logs
C:\Support\Tools
```

The image profile identified `NS-W11-ENT-BASE-01`, the simulated organization, and the intended standardized-workstation use. These files provided concrete artifacts to inspect after deployment.

Reference checks covered the Windows identity, virtual TPM, Defender, Firewall profiles, present-device status, GPT disk layout, DISM `/CheckHealth`, and SFC `/verifyonly`. The reference system reported no integrity issues in those checks. Defender definition freshness was not established. I preserved Audit Mode and pre-Sysprep snapshots before proceeding.

## 5. Phase 3 — Generalization and WIM capture

### Storage and pre-capture checks

I created the separate 50 GB image-store VDI, initialized it as GPT, formatted its data volume as NTFS, and verified file write/read access. It appeared as `S:` in Audit Mode and `D:` in the capture WinPE session. The source Windows partition was `C:` in that WinPE session.

Pre-capture encryption checks found the reference system volume encrypting at 91.9% with protection off. I decrypted it for the lab and verified both source and image-store volumes at 0.0% encrypted before Sysprep. This was a separate observation from the later target encryption incident.

### Sysprep and offline startup

Sysprep was run with **Enter System Out-of-Box Experience**, **Generalize**, and **Shutdown** selected. The VM powered off and next booted into WinPE for offline capture.

The Sysprep log recorded creation of `Sysprep_succeeded.tag`. It also contained UEFI/BCD-related errors shortly before the success marker. I retained that qualification and treated target boot as a separate validation requirement. Phase 4 subsequently demonstrated successful target startup after BCDBoot configuration; the log was not treated as error-free.

### Capture failure and recovery

The first capture failed at **48%** with **error 112 / 0x80070070**, reporting insufficient disk space. Investigation recorded approximately **53.57 billion bytes free** on the destination and approximately **532.8 million bytes free** in the WinPE RAM environment. No incomplete WIM remained in the destination directory.

I created `D:\Scratch` and retried with an explicit scratch directory and a persistent log. The successful command was:

```bat
Dism /Capture-Image /ImageFile:"D:\Images\NS-W11-ENT-BASE-01.wim" /CaptureDir:C:\ /Name:"NS-W11-ENT-BASE-01" /Compress:max /CheckIntegrity /Verify /ScratchDir:D:\Scratch /LogPath:D:\Images\Capture_Retry.log
```

The retry completed at **100%**. Limited WinPE temporary space was a plausible contributor, supported by the successful retry, but the root cause was **not conclusively isolated**.

### Captured-image verification

```bat
Dism /Get-ImageInfo /ImageFile:"D:\Images\NS-W11-ENT-BASE-01.wim" /Index:1
```

| Metadata | Result |
| --- | --- |
| Image filename/name | `NS-W11-ENT-BASE-01.wim` / `NS-W11-ENT-BASE-01` |
| Image index | 1 |
| Edition | `EnterpriseEval` |
| Image version / service-pack build field | `10.0.26300` / `9457` |
| Architecture / installation type | x64 / Client |
| Compressed WIM file size | **8,415,340,722 bytes (~7.84 GiB)** |
| DISM-reported image size | 28,743,603,686 bytes (~26.77 GiB); distinct from compressed file size |
| Result | Metadata readable; command completed successfully |

The metadata, successful capture, and later successful apply form separate checks. Metadata readability alone was not treated as proof of a bootable deployed workstation.

## 6. Phase 4 — Deployment to NS-DEPLOY-01

### Disk identification and partitioning

The target booted WinPE with its new 80 GB system disk and the existing 50 GB image-store disk attached. In this session, the image store appeared as `C:`. I identified both disks before partitioning the new target disk.

| Target partition | Size | Format | Temporary WinPE letter |
| --- | --- | --- | --- |
| EFI System | 200 MB | FAT32 | `S:` |
| Microsoft Reserved | 16 MB | Not formatted | None |
| Windows | Approximately 78 GB | NTFS | `W:` |
| Recovery | 1600 MB | NTFS | `R:` |

DiskPart showed the expected partition/volume layout. The recovery partition was later assigned recovery type GUID `de94bba4-06d1-4d40-a16a-bfd50179d6ac` and attributes `0x8000000000000001`.

The following commands document the recorded lab sessions. Their drive letters are session-specific, not a generic deployment script.

### Image application and boot configuration

```bat
Dism /Apply-Image /ImageFile:"C:\Images\NS-W11-ENT-BASE-01.wim" /Index:1 /ApplyDir:W:\ /CheckIntegrity /Verify /ScratchDir:C:\Scratch /LogPath:C:\Images\Apply_Image.log
bcdboot W:\Windows /s S: /f UEFI
```

An initial apply attempt returned **error 87** because a space was missing between parameters. After correcting syntax, DISM completed at **100%** without repartitioning the target or modifying the WIM. BCDBoot reported that boot files were successfully created; the BCD store and `bootx64.efi` were checked on the EFI partition.

### Recovery image and registration

`Winre.wim` was present with hidden/system attributes. Copy attempts encountered both incorrect quoting and difficulty copying the hidden file. After checking attributes and correcting syntax, this command succeeded:

```bat
xcopy /h /y W:\Windows\System32\Recovery\Winre.wim R:\Recovery\WindowsRE
W:\Windows\System32\ReAgentc.exe /setreimage /path R:\Recovery\WindowsRE /target W:\Windows
```

The recovery directory had been created first. XCOPY reported one file copied, and the destination size matched **796,822,851 bytes**. REAgentC reported successful location registration. Offline status still showed WinRE disabled, so I left enablement verification for the running target OS. Phase 5 later reported WinRE enabled without additional registration repair.

### First boot and configuration preservation

I shut down WinPE, detached the image-store disk, and ejected the boot ISO. The target booted from its deployed system disk into OOBE, completed setup with a local lab administrator account, and reached the Windows desktop.

The deployed edition and architecture matched the reference image. I renamed the computer `NS-DEPLOY-01` and verified its workgroup identity. The deployment profile remained readable, and the workstation-readiness file was present. These checks demonstrated preservation of the selected support artifacts; no byte-for-byte file-hash comparison was recorded.

The snapshot `P04_Deployed_Windows11_Baseline` preserved this completed deployment before final validation.

## 7. Phase 5 — Post-deployment validation

| Check | Method | Recorded result and interpretation |
| --- | --- | --- |
| OS identity | OS inspection and DISM image-version output | Windows 11 Enterprise Evaluation x64; deployed build 26300.9457 |
| Deployment artifacts | Read image profile; `Test-Path` checks | Expected profile and readiness file preserved |
| Virtual TPM | `Get-Tpm` | Present, ready, and enabled: `True` |
| Defender | `Get-MpComputerStatus` | Service, antivirus, and real-time protection enabled; definition freshness not verified |
| Firewall | `Get-NetFirewallProfile` | Domain, Private, and Public profiles enabled; this does not establish domain membership |
| Driver/device health | Filter `Get-PnpDevice -PresentOnly` for non-OK status | No devices returned; limited to reported status of present virtual devices |
| WinRE | `reagentc /info` | Enabled; recovery path on disk 0, partition 4; recovery boot not tested |
| Network adapter and IP | `Get-NetAdapter`; `Get-NetIPConfiguration` | Adapter up with IPv4 configuration and default gateway |
| DNS | `Resolve-DnsName www.microsoft.com -Type A` and subsequent connection test | Name resolution succeeded |
| HTTPS-endpoint connectivity | `Test-NetConnection www.microsoft.com -Port 443` | `TcpTestSucceeded: True`; confirms TCP 443 connectivity, not TLS certificate or web-content validation |
| Component store | `DISM /Online /Cleanup-Image /ScanHealth` | No component-store corruption detected; operation completed successfully |
| Protected system files | `sfc /verifyonly` | No integrity violations; verification performed without a repair operation |

### Unexpected target BitLocker encryption

The target began encrypting automatically, reaching **82.6%, then 82.7%**, using XTS-AES 128. At the same time, `manage-bde` showed **Protection Off** and **no key protectors**. The precise trigger was not independently established.

For this lab, I ran:

```bat
manage-bde -off C:
manage-bde -status C:
```

After decryption completed, verification showed **Fully Decrypted**, **0.0% encrypted**, **Encryption Method: None**, **Protection Off**, and **no key protectors**.

This resolved the unexpected encryption activity for the lab. It did **not** deliver a production-ready encryption configuration or demonstrate enterprise recovery-key management. The final system volume was unencrypted.

### Preserved final state

After validation, I preserved `P05_Final_Validated_Workstation` while retaining the earlier deployment-baseline snapshot. The final snapshot evidence shows both VMs powered off. Snapshot creation was documented; a snapshot restore test was not part of the completed validation.

## 8. Issue and resolution register

| Issue | Finding and action | Closure evidence / remaining uncertainty |
| --- | --- | --- |
| WinPE staging failed | DISM diagnostic error 740; reran tooling with administrator privileges | Staging and ISO creation succeeded; WinPE later booted |
| Capture error 112 at 48% | Destination capacity available; added dedicated scratch folder and persistent log | Capture completed; temporary-space cause suspected, not proven |
| Sysprep UEFI/BCD log errors | Success marker existed, but boot still required independent validation | Target boot succeeded after BCDBoot; no claim that all Sysprep log errors were explained |
| Apply error 87 | Missing separator between parameters; corrected syntax | DISM apply reached 100% |
| `Winre.wim` copy failures | Corrected quoting; inspected hidden/system attributes; used `xcopy /h` | One file copied; destination size verified |
| Offline WinRE disabled | Deferred final status check until after setup | Online REAgentC reported enabled; actual recovery boot untested |
| Target encryption at 82.7%, protection off | Reviewed encryption/protection/key-protector state; decrypted for the lab | Fully decrypted at 0.0%; automatic trigger unresolved; production encryption outside scope |

## 9. Selected evidence register

The selected images establish key deployment milestones. Phase 1–5 notes supply additional context, including security and connectivity results whose dedicated screenshots are outside this curated selection. Each image below supports its stated checkpoint; the final integrity screenshot is not evidence for every security or network check.

| Evidence file | What it demonstrates |
| --- | --- |
| [P03_24_DISM_Image_Capture_Success.png](../screenshots/P03_24_DISM_Image_Capture_Success.png) | Capture reached 100% with successful completion |
| [P03_25_Captured_WIM_Verification.png](../screenshots/P03_25_Captured_WIM_Verification.png) | Exact compressed file size, image index, edition, architecture, and build metadata |
| [P04_04_Target_Disk_Partitioned.png](../screenshots/P04_04_Target_Disk_Partitioned.png) | Target partition and volume layout before final recovery-type configuration |
| [P04_05_DISM_Image_Application_Success.png](../screenshots/P04_05_DISM_Image_Application_Success.png) | Successful image application at 100% |
| [P04_07_UEFI_Boot_Files_Created.png](../screenshots/P04_07_UEFI_Boot_Files_Created.png) | BCDBoot success and EFI boot-file presence |
| [P04_11_First_Boot_OOBE.png](../screenshots/P04_11_First_Boot_OOBE.png) | Target reached first-boot Windows setup; later desktop completion is recorded in phase notes |
| [P04_15_Deployment_Files_Verified.png](../screenshots/P04_15_Deployment_Files_Verified.png) | Readable deployment profile and presence of the support readiness file |
| [P05_03_BitLocker_Remediation.png](../screenshots/P05_03_BitLocker_Remediation.png) | Final fully decrypted target volume at 0.0% |
| [P05_05_System_Integrity.png](../screenshots/P05_05_System_Integrity.png) | DISM ScanHealth and SFC verifyonly results; deployed image build |
| [P05_06_Final_Validated_Snapshot.png](../screenshots/P05_06_Final_Validated_Snapshot.png) | Supplementary evidence of preserved snapshots and powered-off VMs |

## 10. Lessons and transferable skills

- **Disk identification:** The image store moved from `S:` in Windows to `D:` during capture and `C:` during apply. Rechecking capacity, labels, and contents prevented treating letters as permanent identities.
- **Evidence-based troubleshooting:** Destination free space did not explain error 112 by itself. I tested a scratch-space correction and documented both its success and the remaining uncertainty.
- **Deployment sequencing:** Successful capture and readable metadata were intermediate milestones. Apply, EFI boot files, actual startup, and final checks each needed independent verification.
- **Security-state interpretation:** Encryption progress alone did not mean the target had a completed, protected BitLocker configuration.
- **Recovery validation:** Copying `Winre.wim`, registering its path, and seeing WinRE enabled are distinct from demonstrating an actual recovery boot.
- **Support documentation:** Consistent file names, phase notes, logs, snapshots, and a curated evidence register made the work easier to explain and review.

The project provided practical experience with Windows workstation preparation, offline deployment, PowerShell inspection, command-line troubleshooting, and documented technical handoff.

## 11. Scope, limitations, and possible extensions

Completed testing applies to one simulated VirtualBox deployment using evaluation software. The lab did not demonstrate physical-device deployment, diverse hardware drivers, Active Directory, Intune, organizational policy enforcement, production-network integration, enterprise monitoring, or fleet-scale automation.

Windows update compliance and Defender definition freshness were not verified. WinRE was not booted for functional recovery testing. The final target was deliberately unencrypted, and production BitLocker policy, key protectors, and recovery-key management were not implemented. No deployment time savings, production reliability metrics, or business impact measurements are claimed.

Possible future work includes a functional WinRE boot test, documented update compliance, testing on authorized physical hardware, managed encryption with recovery-key handling, and a separate domain-join or Intune enrollment lab. These are proposed extensions, not completed results.

## 12. Attribution and licensing

This is a self-directed IT Support portfolio project by **Ian Van Amberg**. **Northstar Office Solutions is fictional.** The work does not represent employment, a real client engagement, or a production business deployment.

Windows 11 Enterprise Evaluation and all other software remain subject to their respective license terms. This portfolio contains documentation and selected evidence; it does not distribute Windows installation media, captured WIMs, virtual disks, product keys, or recovery keys. Product names and screenshots identify the tools used and imply no vendor affiliation or endorsement. A reuse license for original project materials, if chosen, must be specified separately; this case study does not grant software redistribution rights.
