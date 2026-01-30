# Setting up Synaptics Fingerprint Sensor (ID 06cb:00be) on Arch Linux (Lenovo IdeaPad Flex 5)

This guide provides a detailed walkthrough for enabling your Synaptics fingerprint sensor, which requires a specific community-developed driver (synaTudor), as it's not supported by the mainline libfprint out-of-the-box.

## Disclaimer and Important Warnings

- **Risk of System Instability**: Modifying PAM (Pluggable Authentication Modules) files incorrectly can lock you out of your system.
- **Backup**: Always have a live USB handy with Arch Linux installed, or a reliable backup of your system, before making critical system changes.
- **Device ID**: This guide is specifically for the 06cb:00be Synaptics sensor. If your `lsusb` output shows a different ID, these steps might not apply.

## Prerequisites

- Arch Linux installed and updated.
- Active internet connection.
- The `yay` AUR helper installed. (If you don't have it, install it:
  ```bash
  sudo pacman -S --needed base-devel git
  git clone https://aur.archlinux.org/yay.git
  cd yay
  makepkg -si
  cd ..
  rm -rf yay
  ```
- You have confirmed your fingerprint sensor's ID is 06cb:00be by running `lsusb`.

## Phase 1: Install synaTudor (The Community Driver)

This driver is essential for your specific Synaptics sensor to function on Linux.

### Install Build Dependencies

These packages are required to compile the synaTudor driver.

```bash
sudo pacman -S git meson cmake pkg-config libusb glib2 dbus innoextract openssl libseccomp cryptopp
```

**Note**: `cryptopp` is the correct package name on Arch for the libcrypto++ dependency.

### Clone the synaTudor Repository

This downloads the driver's source code to your system.

```bash
git clone https://github.com/Popax21/synaTudor.git
cd synaTudor
```

### Build and Install synaTudor

This compiles the driver and installs it to the appropriate system directories.

```bash
rm -rf build              # Clean up any previous failed build attempts
meson build               # Configure the build system
cd build                  # Navigate into the build directory
ninja                     # Compile the driver
sudo ninja install        # Install the compiled driver to system paths
```

You should see output indicating successful compilation and installation, including a line like `Run-time dependency libfprint-2-tod-1 found: YES` during `meson build`.

## Phase 2: Install fprintd and its Dependencies

fprintd is the daemon that handles fingerprint authentication on Linux. Your 06cb:00be sensor requires a special "Touch-On-Die" (TOD) version of libfprint.

### Install libfprint-tod-git and fprintd from the AUR

This step ensures you have the correct libfprint version that synaTudor integrates with, and installs the fprintd daemon itself.

```bash
yay -S libfprint-tod-git fprintd
```

`yay` will prompt you to confirm building packages; generally, press `n` unless you want to inspect the PKGBUILD. This process will take some time as it compiles from source.

## Phase 3: Enroll Your Fingerprint

Now that the driver and daemon are installed, you can enroll your fingerprint(s).

### Stop fprintd (for clean enrollment)

It's good practice to stop the service before enrollment to prevent conflicts, though it's often D-Bus activated and might not be actively running.

```bash
sudo systemctl stop fprintd.service
```

**Note**: It might output "Unit fprintd.service not loaded." or "not active, cannot stop." This is expected and fine.

### Enroll Your Fingerprint

This command will guide you through the enrollment process.

```bash
fprintd-enroll
```

Follow the on-screen prompts carefully. You'll typically be asked to place or swipe your finger multiple times (e.g., 8-10 times).

If this step works, it means your sensor, the driver, and fprintd are communicating correctly!

### Verify Enrollment (Optional but Recommended)

Test if your enrolled fingerprint is recognized.

```bash
fprintd-verify
```

Place your enrolled finger on the sensor. It should output "Verify succeeded."

## Phase 4: Configure PAM (Pluggable Authentication Modules)

This is the most critical step for enabling fingerprint authentication for login, sudo, and other system prompts.

**IMPORTANT PAM WARNINGS**:
- Incorrect PAM edits can lock you out of your system! Double-check every change.
- Always have a live USB or recovery shell accessible.

### A. For sudo (Password First, Fingerprint Fallback)

This configuration makes the terminal prompt for your password first, and if that fails or is skipped, it then prompts for your fingerprint.

#### Edit the sudo PAM file

```bash
sudo nano /etc/pam.d/sudo
```

Replace the existing auth section (the lines starting with `auth`) with the following block. Ensure the `account`, `password`, and `session` sections remain as `include system-auth`.

```
#%PAM-1.0
# First, try password authentication.
# 'sufficient' means if this succeeds, no more 'auth' modules are processed.
# 'try_first_pass' means it will prompt for a password if one hasn't been provided.
auth       sufficient  pam_unix.so try_first_pass

# If pam_unix.so fails (incorrect password or empty input), then try fingerprint.
# 'sufficient' here means if fingerprint succeeds, it's enough to authenticate.
auth       sufficient  pam_fprintd.so

# This line acts as a final fallback for other authentication methods defined system-wide,
# but it will only be reached if both pam_unix.so and pam_fprintd.so above explicitly fail.
auth       include     system-auth

account    include     system-auth
password   include     system-auth
session    include     system-auth
```

Save the file: Press `Ctrl + O`, then `Enter`, then `Ctrl + X`.

#### Test sudo

Open a new terminal window and run `sudo ls`. It should prompt for your password first.

### B. For Display Manager (Login Screen)

This enables fingerprint authentication at your graphical login screen.

**Crucial for GNOME (GDM) Users and Login Keyring**: Ensure your Login Keyring password matches your user's login password through the "Passwords and Keys" application (search for "Keys" or "Passwords"). This is critical for the keyring to unlock automatically.

#### Identify your Display Manager's PAM file

- GNOME (GDM): `/etc/pam.d/gdm-password`
- KDE Plasma (SDDM): `/etc/pam.d/sddm`
- LightDM: `/etc/pam.d/lightdm` (or sometimes `/etc/pam.d/lightdm-greeter`)

#### Edit the relevant PAM file

Example for GDM:

```bash
sudo nano /etc/pam.d/gdm-password
```

Add the fingerprint module: Add the line `auth sufficient pam_fprintd.so` at the very top of the auth section.

Example (`gdm-password` modified):

```
#%PAM-1.0

# Add this line at the very top of the 'auth' section for fingerprint login
auth       sufficient  pam_fprintd.so

auth       include     system-local-login
auth       optional    pam_gnome_keyring.so

account    include     system-local-login

password   include     system-local-login
password   optional    pam_gnome_keyring.so use_authtok

session    include     system-local-login
session    optional    pam_gnome_keyring.so auto_start
```

**For GNOME (GDM) Users - Additional Steps**:
- Add user to input group: This is often required for desktop environments to interact with input devices.
  ```bash
  sudo usermod -aG input $USER
  ```
  (Replace `$USER` with your actual username).
- Enable Fingerprint Login in GNOME Settings: Go to Settings > Users. There should be a "Fingerprint Login" option. Ensure it's enabled.
- Keyring Auto-Unlock: Ensure `session optional pam_gnome_keyring.so auto_start` is present in the session section of your display manager's PAM file or in `system-auth` (it usually is by default). If your keyring password matches your user password, this should handle automatic unlocking.

Save the file(s).

### C. For Polkit (GUI Authentication Prompts)

This enables fingerprint authentication for graphical prompts that ask for administrator privileges (e.g., installing software, changing system settings).

#### Copy the default polkit-1 PAM file (if it doesn't exist in /etc/pam.d/)

```bash
sudo cp /usr/lib/pam.d/polkit-1 /etc/pam.d/
```

#### Edit the polkit-1 PAM file

```bash
sudo nano /etc/pam.d/polkit-1
```

Add the fingerprint module: Add the line `auth sufficient pam_fprintd.so` at the very top of the auth section.

Example:

```
#%PAM-1.0
auth       sufficient pam_fprintd.so             # Add this line at the top of 'auth'
auth       include  system-auth
account    include  system-auth
password   include  system-auth
session    include  system-auth
```

Save the file.

## Phase 5: Finalize and Test

### Start and Enable fprintd Service

```bash
sudo systemctl enable fprintd.service
sudo systemctl start fprintd.service
```

You can check its status with `systemctl status fprintd.service`. It should show "active (running)".

### Reboot Your System

A full reboot is crucial for PAM changes (especially for display managers) and group membership changes to take effect.

```bash
sudo reboot
```

### Test Your Fingerprint

- At your login screen (try logging in with your fingerprint).
- In a terminal (try `sudo ls`).
- For a GUI authentication prompt (e.g., try opening a graphical package manager that requires admin privileges).

