# HP Elitedesk 800 G4/G5 Desktop Mini as Server

## Custom scripts for Proxmox host and LXC setup and maintenance

### Proxmox Host:

1) UEFI 2.27 introduces an issue with displayport pin mapping by not activating pin 6 for audio in Linux and needs explicit patching until kernel includes it by default (not as of 6.8.12): /etc/modprobe.d/hda-jack-retask.conf, /usr/lib/firmware/hda-jack-retask.fw
2) Sets DP/HDMI as default audio: /etc/asound.conf
2) Adds useful set of aliases /home/(user)/.aliases
3) Add NFS exports: /etc/exports
4) Includes GPU, Keyboard, Audio passthrough in LXC conf for reference: /etc/pve/lxc/lxc-id.conf
5) Includes scritps for PVE host maintenance, backup-restore and other tweaks: /pve_maintenance (needs to be manually installed/pulled from github)

### LXC:

1) Sets DP/HDMI as default audio: /etc/asound.conf
2) Adds useful set of aliases /home/(user)/.aliases
3) Mounts NFS exports: /etc/auto.master, /etc/auto.mount

## LXC Environment automation

Use this command to run off github:
```
bash -c "$(wget -qLO - https://raw.githubusercontent.com/anirudhra/hpe800g4dm_server/main/pve_lxc_scripts/setup/install.sh)"
```

Use this command to download locally:
```
wget https://raw.githubusercontent.com/anirudhra/hpe800g4dm_server/main/pve_lxc_scripts/setup/install.sh
```

## LXC autologin in PVE Console
Type the following in LXC console to enable auto login:

```
GETTY_OVERRIDE="/etc/systemd/system/container-getty@1.service.d/override.conf"
mkdir -p $(dirname $GETTY_OVERRIDE)
cat <<EOF >$GETTY_OVERRIDE
  [Service]
  ExecStart=
  ExecStart=-/sbin/agetty --autologin root --noclear --keep-baud tty%I 115200,38400,9600 \$TERM
EOF
systemctl daemon-reload
systemctl restart $(basename $(dirname $GETTY_OVERRIDE) | sed 's/\.d//')
```

## PCIe Power Management on PVE host: ASPM

Check which devices support ASPM but not enabled
```
lspci -vv | awk '/ASPM/{print $0}' RS= | grep --color -P '(^[a-z0-9:.]+|ASPM )'
```

If necessary append the following to grub CMDLINE: 
```
pcie_aspm=force
```

## Power/Temperature monitoring on PVE host

Install and add powertune to autotune on startup through "crontab -e":

```
@reboot sleep 60 && /usr/sbin/powertop --auto-tune
```

Also add the following modules to /etc/modules:
```
coretemp
drivetemp
```

Run the following script to add detected CPU/HDD/NVMe sensors to PVE web page (may need after proxmox updates):
```
bash -c "$(wget -qLO - https://raw.githubusercontent.com/anirudhra/hpe800g4dm_server/main/pve_lxc_scripts/maintenance/pve-mod-gui-sensors.sh)"
```

## Enabling IOMMU/VT-d Virtualization on PVE host

Referenced from https://pve.proxmox.com/wiki/PCI(e)_Passthrough:

1) Add the following to /etc/default/grub "GRUB_CMDLINE_LINUX_DEFAULT" to modify kernel command line (remove i915.enable_gvt=1 if there are stability issues):
```
i915.enable_gvt=1 i915.enable_guc=2 intel_iommu=on iommu=pt
```
2) Run to pick changes:
```
upgrade-grub
```
4) Add following modules to /etc/modules:
```
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd #not needed if on kernel 6.2 or newer
```
6) Run to update initram:
```
update-initramfs -u -k all
```
8) Reboot the machine
9) Run following commands to veryify (should see lines along Directed I/O for IOMMMU):
```
dmesg | grep -i -e DMAR -e IOMMU
lsmod | grep -i vfio
cat /proc/cmdline
```

### Full Kernel Command line on PVE host for reference
Kernel command line from /etc/default/grub (remove i915.enable_gvt=1 if there are stability issues and add pcie_aspm=force if ASPM is not getting enabled automatically):
```
BOOT_IMAGE=/boot/vmlinuz-6.8.12-1-pve root=/dev/mapper/pve-root ro quiet i915.enable_gvt=1 i915.enable_guc=2 intel_iommu=on iommu=pt
```

## Switching PVE host/LXC between default console and GUI boot modes (if GUI is installed)

To set console/cli as default:
```
systemctl set-default multi-user.target
```

To set X11 login on boot as default:
```
systemctl set-default graphical.target
```

## Intel iGPU issues and solutions
Note: More information here (may need to create explicit xorg conf): https://wiki.archlinux.org/title/Intel_graphics
Wiki says enable_gvt=1 conflicts with any non-zero enable_guc option in kernal command line.

<br>Also check this page for missing Intel iGPU firmware for certain models like Celeron etc.: https://wiki.debian.org/Firmware#Firmware_missing_from_Debian

## PVE Helper Scripts archives
Repo also contains archives of the excellent and much popular PVE and LXC scripts from ttek and mrrudy (for kodi):

### ttek PVE
Copy of PVE Helper scripts from ttkek: https://tteck.github.io/Proxmox/

### mrrudy Kodi

Copy of Kodi installation PVE LXC from mrrudy: https://github.com/mrrudy/proxmoxHelper (modified here to additionally install kodi-inputstream-adaptive)
