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

<img width="960" height="564" alt="1" src="https://github.com/user-attachments/assets/e2db5a1b-2b60-4d0f-83a6-c9b02e26a188" />

## 2. Select the keyboard layout

On the **Keyboard layout** screen, select **English (US)** in both the layout and variant lists. The text field can be used to check that the keys produce the expected characters. Click **Continue**.

<img width="960" height="564" alt="2" src="https://github.com/user-attachments/assets/8a81f88b-44f5-4cf0-bdf1-a82226e60d60" />


## 3. Choose software and update options

On the **Updates and other software** screen, select **Normal installation**. This includes the desktop applications described by the installer, such as a web browser, utilities and office software.

Leave **Download updates while installing Ubuntu** enabled. Leave **Install third-party software for graphics and Wi-Fi hardware and additional media formats** unchecked, matching the recorded configuration. Click **Continue**.

<img width="960" height="564" alt="3" src="https://github.com/user-attachments/assets/1cec0f1c-c059-479a-8821-95cb451db211" />

## 4. Select the installation type

The installer reported that no operating systems were detected. Select **Erase disk and install Ubuntu**, with no advanced features selected, then click **Install Now**.

This option installs Ubuntu using the selected VM disk and deletes any existing contents on that disk. Check that the intended virtual disk is selected before confirming any disk changes. The disk confirmation dialog and partition details are not included in the screenshots.

<img width="960" height="564" alt="4" src="https://github.com/user-attachments/assets/63f547b3-dd0d-4762-9ccf-3063cb80debe" />

## 5. Set the location

On the **Where are you?** screen, select or enter **Perth** to configure the local time zone. Click **Continue**.

<img width="960" height="564" alt="5" src="https://github.com/user-attachments/assets/478e6745-4131-47f5-a9f7-4d53687568cf" />


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

<img width="960" height="564" alt="6" src="https://github.com/user-attachments/assets/de149168-744a-4cc7-9f6e-4648da59594f" />


## 7. Allow installation to proceed

The installer displays the **Welcome to Ubuntu** slideshow while installation tasks run. The captured progress message reads **Retrieving file 27 of 27**.

Allow the installation to finish. When prompted to restart, follow the installer prompts and remove or detach the installation media if requested. The completion and restart prompts are not captured in the supplied screenshots.

<img width="960" height="564" alt="7" src="https://github.com/user-attachments/assets/6280529b-20e6-4f2a-baa8-5e0865cc4e30" />


## 8. Reach the installed system's sign-in screen

The final screenshot shows the Ubuntu sign-in screen with the **admin-computer** account and a password field. Select the account, enter the password created during installation and sign in.

The captured screen demonstrates that the system reached the graphical sign-in stage. A desktop session after entering the password is not shown in this screenshot set.

<img width="960" height="564" alt="8" src="https://github.com/user-attachments/assets/947eed56-5492-4535-9acb-37d3f5e90595" />


## Recorded outcome

The Admin VM progressed through the Ubuntu installation process and reached the graphical sign-in screen. The recorded setup used a normal desktop installation, an English (US) keyboard, the Perth time zone and a local account named `admin-computer` with password-protected login. Active Directory configuration and domain membership verification are documented separately.
