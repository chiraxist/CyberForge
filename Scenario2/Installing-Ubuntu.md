# Installing Ubuntu on the Admin Virtual Machine

## Purpose

This guide documents the installation of Ubuntu Desktop on the Admin virtual machine in Proxmox. It covers the installer settings, local account creation and the first sign-in screen shown in the supplied screenshots.

## Installation settings

| Setting | Selection shown |
|---|---|
| Proxmox virtual machine | Admin, VM ID 105 |
| Console | noVNC |
| Installer language | English |
| Keyboard layout | English (US) |
| Software selection | Normal installation |
| Download updates during installation | Enabled |
| Third-party graphics, Wi-Fi and media software | Disabled |
| Installation type | Erase disk and install Ubuntu |
| Advanced disk features | None selected |
| Location | Perth |
| Display name | `admin-computer` |
| Local username | `admin-computer` |
| Sign-in setting | Require my password to log in |
| Use Active Directory | Selected on the account setup screen |

The screenshots do not show the VM's CPU, memory or disk allocation, or the exact Ubuntu ISO version.

## 1. Start the Ubuntu installer

Access the Admin virtual machine through the Proxmox noVNC console. At the Ubuntu welcome screen, select **English**, then click **Install Ubuntu** to begin installing the operating system on the virtual disk.

**Screenshot 1:** Ubuntu welcome screen with English selected and the Install Ubuntu option available.

## 2. Select the keyboard layout

On the **Keyboard layout** screen, select **English (US)** in both the layout and variant lists. The text field can be used to check that the keys produce the expected characters. Click **Continue**.

**Screenshot 2:** English (US) selected as the keyboard layout and variant.

## 3. Choose software and update options

On the **Updates and other software** screen, select **Normal installation**. This includes the desktop applications described by the installer, such as a web browser, utilities and office software.

Leave **Download updates while installing Ubuntu** enabled. Leave **Install third-party software for graphics and Wi-Fi hardware and additional media formats** unchecked, matching the recorded configuration. Click **Continue**.

**Screenshot 3:** Normal installation selected, update downloads enabled and third-party software disabled.

## 4. Select the installation type

The installer reported that no operating systems were detected. Select **Erase disk and install Ubuntu**, with no advanced features selected, then click **Install Now**.

This option installs Ubuntu using the selected VM disk and deletes any existing contents on that disk. Check that the intended virtual disk is selected before confirming any disk changes. The disk confirmation dialog and partition details are not included in the screenshots.

**Screenshot 4:** Erase disk and install Ubuntu selected.

## 5. Set the location

On the **Where are you?** screen, select or enter **Perth** to configure the local time zone. Click **Continue**.

**Screenshot 5:** Perth selected as the installation location.

## 6. Create the local account

Complete the **Who are you?** screen with the following details:

| Field | Recorded value or action |
|---|---|
| Your name | `admin-computer` |
| Your computer's name | A generated hostname is present; the full value is not visible |
| Pick a username | `admin-computer` |
| Choose a password | Enter a password for the local account |
| Confirm your password | Re-enter the same password |
| Login option | Require my password to log in |

The password is masked in the screenshot and is not recorded in this guide. Automatic login was not selected.

The **Use Active Directory** checkbox is selected. The installer states that domain details will be entered in the next step, but that screen is not included in this set of screenshots. This selection alone does not establish that the machine successfully joined a domain during installation.

Click **Continue** after completing the account details.

**Screenshot 6:** Local account details, password-required login and the Use Active Directory option selected.

## 7. Allow installation to proceed

The installer displays the **Welcome to Ubuntu** slideshow while installation tasks run. The captured progress message reads **Retrieving file 27 of 27**.

Allow the installation to finish. When prompted to restart, follow the installer prompts and remove or detach the installation media if requested. The completion and restart prompts are not captured in the supplied screenshots.

**Screenshot 7:** Installation in progress, with the Ubuntu slideshow and file retrieval status visible.

## 8. Reach the installed system's sign-in screen

The final screenshot shows the Ubuntu sign-in screen with the **admin-computer** account and a password field. Select the account, enter the password created during installation and sign in.

The captured screen demonstrates that the system reached the graphical sign-in stage. A desktop session after entering the password is not shown in this screenshot set.

**Screenshot 8:** Ubuntu sign-in screen for the local admin-computer account.

## Recorded outcome

The Admin VM progressed through the Ubuntu installation process and reached the graphical sign-in screen. The recorded setup used a normal desktop installation, an English (US) keyboard, the Perth time zone and a local account named `admin-computer` with password-protected login. Active Directory configuration and domain membership verification are documented separately.
