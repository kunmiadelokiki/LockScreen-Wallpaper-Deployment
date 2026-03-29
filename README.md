# LockScreen Wallpaper Deployment via Microsoft Intune

Enterprise-grade lockscreen wallpaper deployment using Microsoft Intune Win32 app packaging, PowerShell automation, and the Windows PersonalizationCSP registry provider — delivering consistent corporate branding across all managed endpoints.

---

## Overview

Deploying a standardised lockscreen wallpaper across an enterprise fleet is a deceptively complex challenge. Microsoft deprecated the native Intune lockscreen wallpaper settings catalog option, leaving organisations without a supported, cloud-native method to enforce branded lockscreen imagery on Windows 10/11 devices.

This project engineers a complete workaround — leveraging the Windows `PersonalizationCSP` registry provider through a PowerShell-driven Win32 app deployment. Rather than relying on deprecated configuration profiles or fragile Group Policy approaches, this solution packages the wallpaper image and deployment logic into a single `.intunewin` package that can be silently deployed, validated, and monitored through the standard Intune app lifecycle.

The approach is production-tested, fully automated, and designed for zero-touch deployment at scale — requiring no end-user interaction and no local administrator intervention on target devices.

---

## Objectives

- **Enforce consistent corporate branding** across the lockscreen of every managed Windows endpoint, eliminating device-level variation caused by user personalisation or imaging inconsistencies.
- **Work around Microsoft's deprecated lockscreen settings** by directly configuring the `PersonalizationCSP` registry provider, ensuring the solution functions regardless of Intune settings catalog changes.
- **Deliver zero-touch deployment** through a fully automated Win32 app package that installs silently, requires no user interaction, and integrates into the standard Intune app lifecycle.
- **Provide reliable detection and validation** so Intune can accurately report deployment status and compliance across the estate.
- **Enable centralised management** of the lockscreen image through Intune's app management console — supporting versioning, redeployment, and group-based targeting.

---

## Technologies & Tools Used

| Component | Detail |
|---|---|
| Management Platform | Microsoft Intune (Endpoint Manager) |
| Deployment Method | Win32 App (.intunewin package) |
| Scripting Language | PowerShell 5.1+ |
| Packaging Tool | Microsoft Win32 Content Prep Tool (IntuneWinAppUtil.exe) |
| Registry Provider | Windows PersonalizationCSP |
| Target OS | Windows 10 & later (64-bit) |
| Detection Method | File-based detection rule |
| Logging | Intune Management Extension transcript logs |

---

## Environment / Prerequisites

Before deploying this solution, the following must be in place:

- **Microsoft Intune** — active tenant with app deployment permissions (Intune Administrator or equivalent RBAC role).
- **Entra ID (Azure AD)** — target devices must be Entra ID joined or Hybrid Entra ID joined and enrolled in Intune MDM.
- **Windows 10/11 enrolment** — endpoints must be enrolled via Autopilot, bulk enrolment, or manual join with the Intune Management Extension installed.
- **Licensing** — Microsoft Intune Plan 1 (included in Microsoft 365 E3/E5, Business Premium, or standalone).
- **Microsoft Win32 Content Prep Tool** — downloaded from the [official Microsoft repository](https://github.com/microsoft/Microsoft-Win32-Content-Prep-Tool) for `.intunewin` packaging.
- **Wallpaper image** — a branded lockscreen image designed at **1920 × 1080** resolution (JPEG format), with key design elements kept within safe margins to accommodate resolution differences across display hardware.

---

## Architecture / Policy Design

### Solution Architecture

The deployment follows a structured file-based approach that separates the wallpaper asset from the deployment logic, packaged together into a single distributable unit:

```
C:\ProgramData\Lockscreen Wallpaper\
├── install.ps1              # Deployment script
├── check.ps1                # Detection script
└── Data\
    └── LockScreenWallpaper.jpg   # Branded wallpaper (1920×1080)
```

### Deployment Flow

The solution operates through four distinct phases:

1. **Packaging** — The folder structure (scripts + image) is processed through `IntuneWinAppUtil.exe` to produce a single `.intunewin` package.
2. **Distribution** — The package is uploaded to Intune as a Win32 app, with install/uninstall commands, requirements, and detection rules configured.
3. **Execution** — On target devices, the Intune Management Extension downloads and executes `install.ps1`, which copies the image to a protected system directory and writes the necessary registry keys.
4. **Validation** — Intune's detection rule confirms deployment success by checking for the presence of the deployed image file.

### Registry Configuration

The `install.ps1` script writes three values to the `PersonalizationCSP` registry provider:

| Registry Path | Value Name | Type | Data |
|---|---|---|---|
| `HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\PersonalizationCSP` | LockScreenImageStatus | DWORD | 1 |
| `HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\PersonalizationCSP` | LockScreenImagePath | STRING | `C:\Windows\System32\Lockscreen.jpg` |
| `HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\PersonalizationCSP` | LockScreenImageUrl | STRING | `C:\Windows\System32\Lockscreen.jpg` |

The `LockScreenImageStatus` value of `1` activates the PersonalizationCSP override, instructing Windows to source the lockscreen image from the path specified in `LockScreenImagePath` rather than the default user-selected wallpaper.

### Design Rationale

The image is copied to `C:\Windows\System32\` for a deliberate reason — this directory is protected by SYSTEM-level NTFS permissions, preventing standard users from modifying or deleting the deployed wallpaper. The `PersonalizationCSP` registry path sits under `HKLM`, ensuring the setting applies machine-wide regardless of which user is logged in.

Using a Win32 app deployment rather than a PowerShell script profile provides several operational advantages: retry logic is handled natively by the Intune Management Extension, detection rules enable accurate compliance reporting, and the package integrates into Intune's standard app lifecycle (versioning, supersedence, uninstall).

---

## Step-by-Step Implementation

### Step 1: Prepare the Folder Structure

Create the source folder hierarchy that will contain the deployment scripts and wallpaper image. This structure must be exact — the `install.ps1` script references the `Data` subfolder relative to its own location.

- Create the parent directory: `C:\ProgramData\Lockscreen Wallpaper`
- Create the image subfolder: `C:\ProgramData\Lockscreen Wallpaper\Data`
- Place the branded wallpaper image (`LockScreenWallpaper.jpg`) inside the `Data` folder

### Step 2: Create the Deployment Script (install.ps1)

The installation script handles three operations: copying the image to the protected system directory, writing the PersonalizationCSP registry keys, and creating a validation marker file. All actions are wrapped in transcript logging for troubleshooting via the Intune Management Extension logs.

```powershell
$PackageName = "Lockscreen"
$Version = 1
$LockscreenIMG = "LockScreenWallpaper.jpg"

Start-Transcript -Path "$env:ProgramData\Microsoft\IntuneManagementExtension\Logs\$PackageName-install.log" -Force
$ErrorActionPreference = "Stop"

$RegKeyPath = "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\PersonalizationCSP"
$LockScreenPath = "LockScreenImagePath"
$LockScreenStatus = "LockScreenImageStatus"
$LockScreenUrl = "LockScreenImageUrl"
$StatusValue = "1"
$LockscreenLocalIMG = "C:\Windows\System32\Lockscreen.jpg"

if (!$LockscreenIMG) {
    Write-Warning "LockscreenIMG must have a value."
} else {
    if (!(Test-Path $RegKeyPath)) {
        New-Item -Path $RegKeyPath -Force
    }
    Copy-Item ".\Data\$LockscreenIMG" $LockscreenLocalIMG -Force
    New-ItemProperty -Path $RegKeyPath -Name $LockScreenStatus -Value $StatusValue -PropertyType DWORD -Force
    New-ItemProperty -Path $RegKeyPath -Name $LockScreenPath -Value $LockscreenLocalIMG -PropertyType STRING -Force
    New-ItemProperty -Path $RegKeyPath -Name $LockScreenUrl -Value $LockscreenLocalIMG -PropertyType STRING -Force
}

New-Item -Path "C:\ProgramData\AdminConfig\Validation\$PackageName" -ItemType "file" -Force -Value $Version
Stop-Transcript
```

**Key design decisions in this script:**

- **Transcript logging** — Every execution is logged to the Intune Management Extension logs directory, enabling remote troubleshooting without device access.
- **Error action preference** — Set to `Stop` to ensure any failure terminates execution immediately rather than continuing with a partial deployment.
- **Registry key creation** — The script creates the `PersonalizationCSP` key if it does not exist, handling both fresh deployments and redeployments cleanly.
- **Validation marker** — A version file is written to `C:\ProgramData\AdminConfig\Validation\` to support custom compliance checks and version tracking.

### Step 3: Create the Detection Script (check.ps1)

The detection script mirrors the installation logic — ensuring that if detection runs on a device where the deployment has partially failed, it re-applies the full configuration rather than reporting a false positive.

### Step 4: Package with IntuneWinAppUtil.exe

The Microsoft Win32 Content Prep Tool converts the source folder into a compressed, encrypted `.intunewin` package suitable for upload to Intune.

- Download and extract the Win32 Content Prep Tool from the [Microsoft GitHub repository](https://github.com/microsoft/Microsoft-Win32-Content-Prep-Tool)
- Run `IntuneWinAppUtil.exe` and provide the following inputs:

| Prompt | Value |
|---|---|
| Source folder | `C:\ProgramData\Lockscreen Wallpaper` |
| Setup file | `C:\ProgramData\Lockscreen Wallpaper\install.ps1` |
| Output folder | `C:\ProgramData\Intune Deployment Output` |
| Catalog folder | N (decline) |

The tool produces `install.intunewin` in the specified output directory.

### Step 5: Upload to Microsoft Intune

Navigate to the Intune admin centre and create a new Win32 app deployment.

- Open **Microsoft Intune admin centre** → Apps → Windows → + Add → Windows app (Win32)
- Upload the generated `.intunewin` package

### Step 6: Configure App Information

| Field | Value |
|---|---|
| Name | LockScreen Configuration |
| Description | Deploy a custom lockscreen wallpaper to client devices |
| Publisher | Olu |

### Step 7: Configure Program Settings

The installation command uses the `sysnative` path to ensure the 64-bit PowerShell host is invoked, regardless of whether the Intune Management Extension runs in a 32-bit context.

| Parameter | Value |
|---|---|
| Install command | `%SystemRoot%\sysnative\WindowsPowerShell\v1.0\powershell.exe -executionpolicy bypass -command .\install.ps1` |
| Uninstall command | `%SystemRoot%\sysnative\WindowsPowerShell\v1.0\powershell.exe -executionpolicy bypass -command .\uninstall.ps1` |
| Install behaviour | System |

### Step 8: Configure Requirements

| Parameter | Value |
|---|---|
| OS architecture | 64-bit |
| Minimum OS | Windows 10 1607 |

### Step 9: Configure Detection Rules

Detection is file-based — Intune checks for the presence of the deployed wallpaper image in the target directory.

| Parameter | Value |
|---|---|
| Rule type | File |
| Path | `C:\Windows\System32\` |
| File or folder | `Lockscreen.jpg` |
| Detection method | File or folder exists |
| 32-bit app on 64-bit | No |

### Step 10: Assign and Deploy

- Skip Dependencies and Supersedence (no settings required)
- Navigate to the **Assignments** tab
- Add the target Entra ID device group under **Required** assignments
- Review and create the deployment
- Wait for Intune to confirm the package upload before refreshing the page

---

## Configuration Details

### Image Deployment

The wallpaper image is copied from the Win32 app package to `C:\Windows\System32\Lockscreen.jpg`. This location is used because it is protected by SYSTEM-level NTFS permissions, preventing standard users from tampering with the deployed image. The 1920 × 1080 resolution provides broad compatibility across laptop and desktop displays, with Windows automatically scaling the image for non-native resolutions.

### PersonalizationCSP Registry Configuration

The `PersonalizationCSP` registry provider is the mechanism Windows uses to determine lockscreen image sourcing when managed by an MDM or configuration service provider. Three values are written:

- **LockScreenImageStatus (DWORD: 1)** — Activates the CSP-managed lockscreen override, instructing Windows to use the image path specified in the registry rather than the default user-selected wallpaper.
- **LockScreenImagePath (STRING)** — Points to the local file path of the deployed image. Windows reads this value at each lockscreen render to source the wallpaper.
- **LockScreenImageUrl (STRING)** — Required by the CSP schema for completeness. In local file deployments, this mirrors the `LockScreenImagePath` value.

### Intune Management Extension Logging

Both the install and detection scripts write full transcript logs to `C:\ProgramData\Microsoft\IntuneManagementExtension\Logs\`. These logs are accessible remotely through Intune's diagnostic collection feature, enabling support teams to troubleshoot deployment failures without requiring local device access.

---

## Testing & Validation

### Pre-Deployment Validation

- **Package integrity check** — Run the `.intunewin` package through a test deployment on a single device before targeting production groups. Verify that the install script executes without errors and all three registry values are written correctly.
- **Image quality verification** — Confirm the wallpaper renders correctly on representative hardware (varying screen sizes, resolutions, and aspect ratios). Ensure key branding elements remain visible and properly positioned.

### Post-Deployment Verification

- **Intune app status** — Monitor Apps → Windows → LockScreen Configuration → Device install status. Confirm all targeted devices report **Installed** status.
- **File presence check** — Verify `C:\Windows\System32\Lockscreen.jpg` exists on target devices, confirming the image copy operation succeeded.
- **Registry validation** — Confirm the three `PersonalizationCSP` values exist and contain the correct data:
  ```
  HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\PersonalizationCSP
  ├── LockScreenImageStatus = 1 (DWORD)
  ├── LockScreenImagePath = C:\Windows\System32\Lockscreen.jpg (STRING)
  └── LockScreenImageUrl = C:\Windows\System32\Lockscreen.jpg (STRING)
  ```
- **Visual confirmation** — Lock a test device (Win+L) and verify the branded wallpaper displays correctly on the lockscreen.
- **Transcript log review** — Check `C:\ProgramData\Microsoft\IntuneManagementExtension\Logs\Lockscreen-install.log` for any warnings or errors during execution.

---

## Results / Outcomes

- **Consistent corporate branding** across all managed endpoints — every device displays the same branded lockscreen wallpaper regardless of user personalisation settings.
- **Successful deployment at scale** with zero end-user interaction required — the Win32 app installs silently in the SYSTEM context without user prompts or restarts.
- **Reliable detection and reporting** through Intune's file-based detection rule, providing accurate compliance visibility across the estate.
- **Resilient to user interference** — the wallpaper is stored in a SYSTEM-protected directory and the registry keys are written under HKLM, preventing standard users from overriding the configuration.
- **Full audit trail** through transcript logging, enabling remote troubleshooting and deployment validation without device access.
- **Workaround for deprecated functionality** — the solution delivers lockscreen wallpaper management through a method that remains functional regardless of changes to the Intune settings catalog.

---

## Challenges & Solutions

### Challenge: Deprecated Intune Lockscreen Settings

**Problem:** Microsoft deprecated the native lockscreen wallpaper configuration option in the Intune settings catalog, removing the supported method for deploying branded lockscreen images to managed devices.

**Solution:** Engineered an alternative deployment path using the Windows `PersonalizationCSP` registry provider, which the deprecated settings catalog option was built on. By writing the registry values directly through a PowerShell script deployed as a Win32 app, the solution achieves the same outcome through a method that is independent of settings catalog availability.

### Challenge: 32-bit vs 64-bit PowerShell Context

**Problem:** The Intune Management Extension can execute scripts in a 32-bit process context on 64-bit systems, which causes registry writes to be redirected to `WOW6432Node` — resulting in the PersonalizationCSP values being written to the wrong registry location.

**Solution:** The install command explicitly invokes the 64-bit PowerShell host through the `%SystemRoot%\sysnative\` path, ensuring registry operations target the correct native registry hive regardless of the calling process architecture.

### Challenge: Detection Rule Accuracy

**Problem:** A detection rule based solely on registry key presence could return false positives if the registry values exist but the image file was not successfully copied (partial deployment failure).

**Solution:** The detection rule is file-based, checking for the physical presence of `Lockscreen.jpg` in `C:\Windows\System32\`. Since the image copy is the last critical operation in the deployment script (after registry writes), its presence confirms a complete, successful deployment.

---

## Best Practices / Recommendations

- **Use the `sysnative` path** in all Win32 app install commands that invoke PowerShell on 64-bit systems. This prevents silent failures caused by WOW64 registry redirection.
- **Design wallpapers at 1920 × 1080** with safe margins for key elements. While Windows scales lockscreen images, critical branding should remain visible across common display resolutions (1366×768, 1920×1080, 2560×1440).
- **Implement transcript logging** in all deployment scripts. Intune Management Extension logs are the primary remote troubleshooting tool for Win32 app failures.
- **Use file-based detection** over registry-based detection for deployments that involve both file operations and registry writes. File presence confirms the complete deployment chain.
- **Version your deployments** using the validation marker file (`C:\ProgramData\AdminConfig\Validation\`). When updating the wallpaper, increment the version number and use Intune's supersedence feature to manage the upgrade path.
- **Test on representative hardware** before broad deployment. Lockscreen rendering can vary across display hardware, drivers, and Windows versions.
- **Wait for upload confirmation** before refreshing the Intune console after submitting a Win32 app. Premature refresh can interrupt the upload process.

---

## Conclusion

This project demonstrates a production-ready solution for deploying branded lockscreen wallpapers across an enterprise Windows fleet — solving a real operational gap created by the deprecation of native Intune lockscreen configuration. By combining PowerShell automation, the Windows PersonalizationCSP registry provider, and Intune Win32 app packaging, the solution delivers reliable, scalable, and auditable lockscreen management without depending on deprecated platform features.

The architecture follows enterprise deployment best practices: silent execution in the SYSTEM context, file-based detection for accurate compliance reporting, transcript logging for remote troubleshooting, and NTFS-protected storage to prevent user tampering. The result is a zero-touch deployment that enforces consistent corporate branding across the entire managed estate.

---

## Author

| | |
|---|---|
| **Name** | Olu Adelokiki |
| **Title** | Lead EUC Engineer |
| **Organisation** | UserCompute |
| **Website** | [www.usercompute.com](https://www.usercompute.com) |
# LockScreen-Wallpaper-Deployment
Enterprise lockscreen wallpaper deployment using Microsoft Intune Win32 app packaging, PowerShell automation, and the Windows PersonalizationCSP registry provider.
