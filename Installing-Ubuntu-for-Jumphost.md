# Installing Ubuntu on the JumpHost Virtual Machine

## Purpose

This guide documents the Ubuntu Desktop installation procedure for the JumpHost virtual machine in Proxmox. JumpHost was configured with a local account and was not joined to Active Directory.

## Installation settings

| Setting | Configuration |
|---|---|
| Proxmox virtual machine | JumpHost (VM ID not supplied) |
| Console | noVNC |
| Installer language | English |
| Keyboard layout | English (US) |
| Software selection | Normal installation |
| Download updates during installation | Enabled |
| Third-party graphics, Wi-Fi and media software | Disabled |
| Installation type | Erase disk and install Ubuntu |
| Advanced disk features | None selected |
| Location | Perth |
| Display name | `JumpHost` |
| Computer name | `JumpHost` |
| Local account password | `JumpH0st2026!` |
| Local username | `jumphost` |
| Sign-in setting | Require my password to log in |
| Use Active Directory | Disabled |

The VM's CPU, memory and disk allocation, and exact Ubuntu ISO version are not specified in this guide.

## 1. Start the Ubuntu installer

Access the JumpHost virtual machine through the Proxmox noVNC console. At the Ubuntu welcome screen, select **English**, then click **Install Ubuntu** to begin installing the operating system on the virtual disk.

## 2. Select the keyboard layout

On the **Keyboard layout** screen, select **English (US)** in both the layout and variant lists. The text field can be used to check that the keys produce the expected characters. Click **Continue**.

## 3. Choose software and update options

On the **Updates and other software** screen, select **Normal installation**. This includes the desktop applications described by the installer, such as a web browser, utilities and office software.

Leave **Download updates while installing Ubuntu** enabled. Leave **Install third-party software for graphics and Wi-Fi hardware and additional media formats** unchecked, matching the recorded configuration. Click **Continue**.

## 4. Select the installation type

Select **Erase disk and install Ubuntu**, with no advanced features selected, then click **Install Now**.

This option installs Ubuntu using the selected VM disk and deletes any existing contents on that disk. Check that the intended virtual disk is selected before confirming any disk changes.

## 5. Set the location

On the **Where are you?** screen, select or enter **Perth** to configure the local time zone. Click **Continue**.

## 6. Create the local account

Complete the **Who are you?** screen with the following details:

| Field | Recorded value or action |
|---|---|
| Your name | `JumpHost` |
| Your computer's name | `JumpHost` |
| Pick a username | `jumphost` |
| Choose a password | `JumpH0st2026!` |
| Confirm your password | `JumpH0st2026!` |
| Login option | Require my password to log in |

Use `JumpHost` for the display name and computer name. This guide uses `jumphost` as the lowercase Linux username; the username was not separately supplied. Enter the password exactly as shown, including the zero in `H0st` and the final exclamation mark. Select password-required login.

Leave **Use Active Directory** unchecked. JumpHost uses the local account created here for authentication.

Click **Continue** after completing the account details.

## 7. Allow installation to proceed

The installer displays the **Welcome to Ubuntu** slideshow while installation tasks run.

Allow the installation to finish. When prompted to restart, follow the installer prompts and remove or detach the installation media if requested.

## 8. Reach the installed system's sign-in screen

After restarting, select the **JumpHost** account at the Ubuntu sign-in screen and enter `JumpH0st2026!` to sign in.

## Expected outcome

After completing these steps, the JumpHost VM should reach the Ubuntu graphical sign-in screen. This setup uses a normal desktop installation, an English (US) keyboard, the Perth time zone and a local account named `jumphost` with password-protected login. JumpHost uses local authentication and is not joined to a domain.
