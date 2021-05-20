# Fedora 34 on Dell XPS 9570 Reference

*This guide is for personal reference only, and has not guarantee to be correct, up-to-date and reliable. Follow at your own risk*

## Update System

Before we start, we make sure that Fedora is up to date:

```bash
$ sudo dnf update
```

## Install Additional Packages

We'll need a few additional packages to comletely configure and optimise our installation, as well as add additional tools to be used on the workstation.

```bash
$ sudo dnf install mc\
	vim-editorconfig\
	vim-enhanced.x86_64\
	skopeo\
	wavemon.x86_64\
	htop\
	lm_sensors.x86_64\
	sysfsutils\
	linux-firmware.noarch\
	powertop\
	tlp tlp-rdw\
	tuned-utils\
	thermald\
	kernel-devel\
	kernel-headers\
	gcc\
	make\
	dkms\
	acpid\
	\libglvnd-glx\
	libglvnd-opengl\
	libglvnd-devel\
	pkgconfig\
	xrandr

```

## Setup Hardware

### Enable Power Management Tools

Complete these steps while the laptop is *UNPLUGGED* from power.

Enable `powertop` service:

```bash
$ sudo systemctl start powertop
$ sudo systemctl enable powertop
$ sudo powertop --auto-tune
```

Enable the `tlp` service:

```bash
$ sudo systemctl enable tlp.service
```

### Optimise Power Management 

#### TLP Radio Control

Enable NetworkManager dispatcher to put radios under `tlp` control:

```bash
$ sudo systemctl enable NetworkManager-dispatcher.service
```

Disable rfkill service and socket:

```bash
$ sudo systemctl mask systemd-rfkill.service
$ sudo systemctl mask systemd-rfkill.socket 
```

Restart the `tlp` service:

```bash
$ sudo systemctl restart tlp.service
```

#### TLP CPU Frequency Scaling

This will tame the CPU when on battery a bit. We don't need to be running at
full tilt of 4.1 GHz when on battery.  You can check the supported frequencies
with `tlp-stat -p`.

Tweak CPU frequency scaling in `tlp` by creating a drop-in in `/etc/tlp.d`:

```bash
$ sudo tee /etc/tlp.d/00-cpu-scaling.conf << 'EOL'
CPU_SCALING_MIN_FREQ_ON_AC=800000
CPU_SCALING_MAX_FREQ_ON_AC=4100000
CPU_SCALING_MIN_FREQ_ON_BAT=800000
CPU_SCALING_MAX_FREQ_ON_BAT=2500000
EOL
```

Restart the `tlp` service:

```bash
$ sudo systemctl restart tlp.service
```

#### USB Device Power Management

The Linux kernel (appears to) defaults to enabling autosuspend on USB devices
when power management is active. We can add a `udev` rule to override this 
behaviour for a particular USB device, and blacklist the device in the `tlp` configuration.

This *should* stop devices like a USB mouse from "sticking" when on battery.

We will need to determine the Vendor and Product ID's for the usb device we
want to configure using the `lsusb` command.  

Find your USB device ID:

```bash
$ sudo lsbusb
Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Bus 001 Device 004: ID 27c6:5395 Shenzhen Goodix Technology Co.,Ltd. Fingerprint Reader
Bus 001 Device 003: ID 8087:0029 Intel Corp. AX200 Bluetooth
Bus 001 Device 005: ID 0c45:671d Microdia Integrated_Webcam_HD
Bus 001 Device 007: ID 046d:c52b Logitech, Inc. Unifying Receiver
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
```

In this case the USB wireless mouse I want to configure has an ID of
`046d:c52b`, which is a concatenation of the Vendor ID: `046d` and the Product
ID: `c52b`.

Blacklist the USB devices from being suspended when on battery by creating a
drop-in in `/etc/tlp.d`:

```bash
$ sudo tee /etc/tlp.d/01-usb-blacklist.conf << 'EOL'
#Logitech, Inc. Unifying Receiver
USB_BLACKLIST="046d:c52b"
EOL
```

Override the default udev behaviour of auto-suspending by creating a drop-in
rule in `/etc/udev/rules.d` (replace `idVendor` and `idProduct` as required):

```bash
$ sudo tee /etc/udev/rules.d/71-mouse-pm.rules << 'EOL'
#Logitech, Inc. Unifying Receiver
ACTION=="add", SUBSYSTEM=="usb", ATTR{idVendor}=="046d", ATTR{idProduct}=="c52b", TEST=="power/control", ATTR{power/control}="on"
ACTION=="change", SUBSYSTEM=="usb", ATTR{idVendor}=="046d", ATTR{idProduct}=="c52b", TEST=="power/control", ATTR{power/control}="on"
ACTION=="bind", SUBSYSTEM=="usb", ATTR{idVendor}=="046d", ATTR{idProduct}=="c52b", TEST=="power/control", ATTR{power/control}="on"
EOL
```

An example rules file can be found [here](etc/udev/rules.d/71-logitech-mouse-pm.rules)

#### Optimise Intel iGPU

GRUB_CMDLINE_LINUX="rhgb quiet mem_sleep_default=deep i915.enable_gvt=1"

### Install NVidia Proprietary GPU Driver and Tools (Optional)

[TODO]

#### References

[RPMFusion NVIDIA Howto](https://rpmfusion.org/Howto/NVIDIA)
[RPMFusion NVIDIA Optimus Howto](https://rpmfusion.org/Howto/Optimus)

## Microsoft Teams

Install Mirosoft Teams using the Microsoft provided package repository for Fedora.

Install the repo signing key:

```bash
$ sudo rpm --import https://packages.microsoft.com/keys/microsoft.asc
```

Add the repository:

```bash
$ sudo tee /etc/yum.repos.d/ms-teams.repo << 'EOL'
[ms-teams]
name=Microsoft Teams
baseurl=https://packages.microsoft.com/yumrepos/ms-teams
enabled=1
gpgcheck=1
gpgkey=https://packages.microsoft.com/keys/microsoft.asc
EOL
```

Update repo metadata:

```bash
$ sudo dnf update
```

Install from repo:

```bash
$ sudo dnf install teams
```

## Install Snap Store

You can use the Snap store to install "snaps" that provides some non-distribution packaged apps.

Install `snapd`:

```bash
$ sudo dnf install snapd
```

Reboot your machine to enable `snapd` service to properly initialise

Install the Snap store app:

```bash
$ sudo snap install snap-store
```

### Useful snaps to install

- Prospect Mail - "Unofficial" Microsoft Outlook Client, wraps O365 Web Based Outlook in an Electron App.



## Useful commands

### Rebuild initramfs

```bash
$ sudo dracut -fv
```

### Update grub config

```bash
$ sudo grub2-mkconfig -o /boot/efi/EFI/fedora/grub.cfg
```

```bash
$ cat /sys/kernel/debug/dri/0/gt/uc/guc_info
$ cat /sys/kernel/debug/dri/0/gt/uc/huc_info
$ modinfo -p i915
$ systool -m i915 -av
```

## References

[Fedora Workstation 33 with btrfs-luks full disk encryption](https://mutschler.eu/linux/install-guides/fedora-btrfs/)
