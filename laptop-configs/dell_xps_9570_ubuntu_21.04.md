# Ubuntu 21.04 on Dell XPS 9570 Reference

*This guide is for personal reference only, and has not guarantee to be correct, up-to-date and reliable. Follow at your own risk*

Chose to bump the Ubuntu installation to 21.04 as opposed to the 20.04 LTS release purely for the changes in the later releases that are more appropriate for development needs, and better hardware support in later Linux kernels.

[Custom Install - Encrypted BTRFS System](#custom-install---encrypted-btrfs-system)

# Custom Install - Encrypted BTRFS System

A custom installation of Ubuntu 21.04 that provides an encrypted BTRFS system disk and swap, with automated system snapshot (like Apple's Time Machine).

Start with the article [here](https://mutschler.eu/linux/install-guides/ubuntu-btrfs/). Some customisations applied.

# Post-installation configuration

## Update System

Bring the machine up to date with the latest packages and firmware:

```bash
$ sudo apt update
$ sudo apt upgrade
$ sudo apt dist-upgrade
$ sudo fwupdmgr update
$ sudo apt autoremove
$ sudo apt autoclean
```

Reboot after updates:

```bash
$ sudo reboot now
```

## Install software required to finish configuring:

```bash
$ sudo apt install -y mc vim vim-editorconfig sysfsutils hwinfo powertop tlp tlp-rdw acpica-tools buildah podman skopeo wavemon
```

## Fix console and grub font sizes

The XPS 15 9570 has a native screen resolution of 3840x2160 which makes the Grub boot menu almost unreadable, as well as the native linux console. We will adjust the configuration to make this more readable. 

### Grub

Create a new Grub bootloader font based on the DejaVu Sans truetype font:

```bash
$ sudo apt install fonts-dejavu
$ sudo grub-mkfont --output=/boot/grub/fonts/DejaVuSansMono48.pf2 --size=48 /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf
```

Change the Grub defaults:

```bash
$ vi /etc/default/grub
```

Uncomment `GRUB_GFXMODE` and update it to the right resolution, and add a line containing the `GRUB_FONT`

```bash
GRUB_GFXMODE=3840x2160
GRUB_FONT=/boot/grub/fonts/DejaVuSansMono48.pf2
```

Update the Grub configuration:

```bash
$ sudo update-grub
```

### Early Grub Config (Optional)

There is an early Grub startup that decrypts the disk using the user supplied password. This console is still tiny. To change this we need to configure the early Grub config that eventually load the config in the boot partition.

[TODO]

[References]

(https://wiki.archlinux.org/title/GRUB/Tips_and_tricks#Manual_configuration_of_core_image_for_early_boot

### Linux Console

Reconfigure the `console-setup` package to change the console defaults:

```bash
$ sudo dpkg-reconfigure console-setup
```

* Select 'UTF-8' encoding
* Select 'Guess optimal character set'
* Select 'Terminus' for the console font
* Select '16x32' Font size

## Speed up LUKS decryption in Grub (Optional)

If the system was installed with whole disk encryption, at boot the system may not have enough compute resource or entropy available, and LUKS disk decryption may take quite some time. We can speed this up by reducing the number of Key Deriviation Function iterations required for the key in slot 0 (the password slot).

```bash
$ sudo cryptsetup luksChangeKey --key-slot=0 --pbkdf-force-iterations 5000 /dev/nvme0n1p3
```

Provide the existing password and re-enter the same password, or set a new **STRONG** password.

## Configure power management

Complete these steps while the laptop is *UNPLUGGED* from power.

Enable the `tlp` service:

```bash
$ sudo systemctl enable tlp.service
```

Enable NetworkManager dispatcher to put radios under `tlp` control:

```bash
$ sudo systemctl enable NetworkManager-dispatcher.service
```

Apply `powertop` tuning:

```bash
$ sudo powertop --auto-tune
```

### Tweak CPU frequency scaling


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

### USB Device power management


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

Reload `udev` rules:

```bash
$ sudo udevadm control --reload
```

## Configure Software Tools

### Git

Setup global git username details:

```bash
$ git config --global user.name "John Doe"
$ git config --global user.name "jdoe@example.com"
```


