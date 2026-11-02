# Windows 11 Enterprise Imaging & Deployment Lab

**Project 2 · Ian Van Amberg · Self-directed IT Support portfolio lab**  
**Scenario:** Northstar Office Solutions, a fictional small business  
**Status:** Imaging, deployment, and planned lab validation completed

I built and generalized a Windows 11 Enterprise reference workstation, captured it as a reusable WIM image, and deployed it to a separate virtual workstation. The deployed system booted through UEFI, completed Windows setup, preserved the reference support files, and passed the planned security-state, device, connectivity, and system-integrity checks.

This project demonstrates hands-on workstation deployment and troubleshooting in Oracle VirtualBox. It is an independently completed simulation, not employment, client work, or a production rollout.

**[Read the technical case study](documentation/Technical_Case_Study.md)** for the Phase 1–5 workflow, commands, troubleshooting decisions, validation boundaries, and evidence register.

## Results at a glance

| Deliverable or check | Recorded outcome |
| --- | --- |
| Operating system | Windows 11 Enterprise Evaluation **26H2, build 26300.9457, x64**, verified in the lab |
| Reusable image | `NS-W11-ENT-BASE-01.wim` — **8,415,340,722 bytes (~7.84 GiB)**; image index 1 |
| Image capture and application | Both DISM operations completed successfully at **100%** |
| Target startup | UEFI boot succeeded; OOBE completed; Windows desktop reached on `NS-DEPLOY-01` |
| Configuration preservation | Deployment profile and workstation-readiness file retained |
| Security and device checks | Virtual TPM ready; Defender and all Firewall profiles enabled; no present PnP devices reported a non-OK status |
| Recovery configuration | REAgentC reported WinRE enabled with a registered recovery path; recovery boot was not tested |
| Connectivity | Adapter, IP configuration, and DNS checks passed; TCP connection to an HTTPS endpoint on port 443 succeeded |
| System integrity | DISM `/ScanHealth`: no component-store corruption; SFC `/verifyonly`: no integrity violations |
| Final lab state | Unexpected BitLocker encryption resolved by full decryption; validated VirtualBox snapshot preserved |

## Business scenario and environment

The simulated requirement was to prepare a consistent workstation image for new employee computers at Northstar Office Solutions. I used a separate reference VM and deployment VM to demonstrate the complete process from preparation to validated first boot.

| Component | Configuration or purpose |
| --- | --- |
| Reference VM | `NS-REF-01`; 4 GB RAM, 2 vCPUs, 80 GB GPT system VDI |
| Deployment VM | `NS-DEPLOY-01`; 4 GB RAM, 2 vCPUs, separate 80 GB GPT system VDI |
| Virtual firmware | UEFI, Secure Boot enabled in VM configuration, virtual TPM 2.0 |
| Image storage | `NS-ImageStore-50GB.vdi`; 50 GB, GPT/NTFS; separate from both system disks |
| Deployment tools | Windows ADK Deployment Tools, WinPE add-on, Sysprep, DISM, DiskPart, BCDBoot, REAgentC, PowerShell |
| Networking | VirtualBox NAT; disconnected during image preparation and initial deployment, then connected for validation |

## Workflow

1. **Prepare the lab:** Assess host resources, verify installation-media SHA-256, install ADK/WinPE tools, and create bootable x64 WinPE media.
2. **Configure the reference:** Install Windows in Audit Mode, create the image profile and support folders, check the baseline, and preserve snapshots.
3. **Generalize and capture:** Run Sysprep with Generalize, OOBE, and Shutdown; boot WinPE; capture the Windows partition with DISM; inspect WIM metadata.
4. **Deploy to a new disk:** Provision EFI, MSR, Windows, and recovery partitions; apply the WIM; create UEFI boot files; copy and register the recovery image.
5. **Validate and preserve:** Complete OOBE, assign the target identity, verify support files, run security/device/network/integrity checks, resolve unexpected encryption, and save a final snapshot.

## Selected evidence

The gallery highlights three checkpoints. The [case-study evidence register](documentation/Technical_Case_Study.md#9-selected-evidence-register) links all selected screenshots.

| WIM capture completed | Image applied to target | Final integrity checks |
| --- | --- | --- |
| [![DISM image capture completed at 100 percent](screenshots/P03_24_DISM_Image_Capture_Success.png)](screenshots/P03_24_DISM_Image_Capture_Success.png) | [![DISM image application completed at 100 percent](screenshots/P04_05_DISM_Image_Application_Success.png)](screenshots/P04_05_DISM_Image_Application_Success.png) | [![DISM ScanHealth and SFC verifyonly passed](screenshots/P05_05_System_Integrity.png)](screenshots/P05_05_System_Integrity.png) |

## Troubleshooting highlights

| Issue | Investigation and resolution | Verified outcome |
| --- | --- | --- |
| DISM capture error 112 at 48% | Checked destination capacity and logs; retried with a dedicated disk-backed scratch folder. Limited WinPE temporary space was suspected, **not proven**. | Capture completed at 100%; WIM metadata readable. |
| DISM apply error 87 | Corrected a missing space between command parameters. | Image applied successfully without repartitioning or changing the WIM. |
| Recovery image copy failure | Inspected hidden/system attributes on `Winre.wim`; corrected command quoting and used `xcopy /h`. | Recovery image copied and destination file verified. |
| Unexpected BitLocker encryption | Observed 82.7% encryption with protection off and no key protectors; decrypted the target volume for the lab. | Fully Decrypted, 0.0% encrypted. **This is not production-ready encryption.** |

## What I learned

- Identify disks by capacity, role, and volume label before making changes; WinPE drive letters changed between capture and deployment.
- Separate image creation, boot configuration, first boot, and post-deployment validation into distinct checks.
- Investigate temporary working space as well as destination capacity when imaging fails.
- Read encryption progress, protection state, and key-protector status together.
- Record what a test proves: enabled WinRE is a configuration result, and TCP 443 reachability is narrower than a complete web-access test.

**Skills demonstrated:** Windows deployment, Audit Mode and Sysprep, offline imaging, UEFI/GPT partitioning, recovery configuration, PowerShell diagnostics, endpoint validation, structured troubleshooting, and technical documentation.

## Repository structure

```text
GitHub_Publication/
├── README.md
├── documentation/
│   └── Technical_Case_Study.md
└── screenshots/
    └── Selected Phase 3–5 PNG evidence
```

Reviewed phase notes may be retained in `documentation/` as supporting material. Windows installation media, captured WIMs, virtual disks, credentials, recovery keys, and unreviewed logs are excluded from the intended public repository.

## Scope and limitations

This was a simulated virtual lab with one reference VM and one target VM. No physical-hardware deployment, Active Directory domain join, Intune enrollment, enterprise policy application, or production network integration was demonstrated. VirtualBox Guest Additions remained in the image; hardware portability was not established.

WinRE boot and recovery operations were not functionally tested. Windows update compliance and Defender definition freshness were not verified. The final Windows volume was deliberately decrypted for the lab; enterprise encryption and recovery-key management were outside scope. Results apply to the checks performed on this virtual workstation, not a production-readiness certification.

## Attribution and licensing

Created by **Ian Van Amberg**. Northstar Office Solutions is fictional. Windows 11 Enterprise Evaluation and the other software used remain subject to their respective license terms. This repository documents the lab and does not distribute Windows images, installation media, virtual disks, or software licenses. Product names and screenshots are used for identification and documentation; no vendor affiliation or endorsement is implied. No open-source license is granted by this README; any reuse license for original project materials must be stated separately.
