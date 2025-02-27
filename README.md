# Termux QEMU Alpine ARM64 with EFI

This repository contains instructions and scripts to run Alpine Linux on an ARM64 architecture using QEMU in Termux with EFI support.

## Prerequisites

1. **Termux**: Install Termux from the Google Play Store or F-Droid.
2. **QEMU**: Install QEMU in Termux by running:
   ```sh
   pkg install qemu-system-aarch64
   ```
3. **Alpine Linux**: Download the Alpine Linux ARM64 image from the official Alpine Linux website.

## Steps

### 1. Download Alpine Linux Image

Download the latest stable version of Alpine Linux for ARM64:
```sh
wget http://dl-cdn.alpinelinux.org/alpine/latest-stable/releases/aarch64/alpine-virt-3.21.0-aarch64.iso -O alpine.iso
```

### 2. Download qemu-efi-aarch64

Download the QEMU EFI for ARM64:
```sh
wget http://ftp.us.debian.org/debian/pool/main/e/edk2/qemu-efi-aarch64_2024.11-5_all.deb -O qemu-efi-aarch64.deb
```

Extract the EFI file using `7zip` or a similar tool and copy `QEMU_EFI.fd` file:
```sh
7z x qemu-efi-aarch64.deb -o./qemu-efi
cp qemu-efi/usr/share/qemu-efi/QEMU_EFI.fd .
```

### 3. Create EFI Flash Images

Create flash images for EFI:
```sh
dd if=/dev/zero of=flash0.img bs=1M count=64
dd if=QEMU_EFI.fd of=flash0.img conv=notrunc
dd if=/dev/zero of=flash1.img bs=1M count=64
```

### 4. Create System Disk

Create a disk image for the system:
```sh
qemu-img create -f qcow2 alpine.qcow2 20G
```

### 5. Install System

Start the installation process:
```sh
qemu-system-aarch64 \
  -M virt -m 2G -cpu cortex-a72 -smp 2 \
  -bios QEMU_EFI.fd \
  -drive id=cdrom,media=cdrom,if=none,file=alpine.iso \
  -device virtio-scsi-device -device scsi-cd,drive=cdrom \
  -drive id=hd0,media=disk,if=none,file=alpine.qcow2 \
  -device virtio-scsi-pci -device scsi-hd,drive=hd0 \
  -nic user,model=virtio-net-pci,hostfwd=tcp::2222-:22,hostfwd=tcp::6900-:5900 \
  -nographic
```

Wait for the system to boot and log in as `root`.

### 6. Initialize Network

Initialize the network interface:
```sh
setup-interfaces
```
When prompted, choose the following options:
```
Available interfaces are: eth0.
Enter '?' for help on bridges, bonding and vlans.
Which one do you want to initialize? (or '?' or 'done') [eth0]
Ip address for eth0? (or 'dhcp', 'none', '?') [dhcp]
Do you want to do any manual network configuration? [no]
```
Then bring up the interface:
```sh
ifup eth0
```

### 7. Install Alpine Linux

Run the Alpine setup script:
```sh
setup-alpine
```
During the setup, you can install `openssh` and allow `root` login.

### 8. Shut Down

Shut down the system after installation:
```sh
poweroff
```

### 9. Start System

Start the installed system:
```sh
qemu-system-aarch64 \
  -M virt -m 2G -cpu cortex-a72 -smp 2 \
  -drive file=flash0.img,format=raw,if=pflash \
  -drive file=flash1.img,format=raw,if=pflash \
  -drive id=hd0,media=disk,if=none,file=alpine.qcow2 \
  -device virtio-scsi-pci -device scsi-hd,drive=hd0 \
  -nic user,model=virtio-net-pci,hostfwd=tcp::2222-:22,hostfwd=tcp::6900-:5900 \
  -nographic
```

## Post-Installation

After installing the system, you can write a script to start the virtual machine and run the service in the background. You can also use RVNC Viewer to connect to the virtual machine via VNC or use SSH to log in to the virtual machine at `127.0.0.1:2222`.

### Start Script

Create a start script `start-vm.sh`:
```sh name=start-vm.sh
#!/bin/bash

qemu-system-aarch64 \
  -M virt -m 2G -cpu cortex-a72 -smp 2 \
  -drive file=flash0.img,format=raw,if=pflash \
  -drive file=flash1.img,format=raw,if=pflash \
  -drive id=hd0,media=disk,if=none,file=alpine.qcow2 \
  -device virtio-scsi-pci -device scsi-hd,drive=hd0 \
  -nic user,model=virtio-net-pci,hostfwd=tcp::2222-:22,hostfwd=tcp::6900-:5900 \
  -device virtio-gpu \
  -vnc :0 &

# Print message
echo "QEMU virtual machine has started and is providing a graphical interface via VNC."
echo "Please use a VNC client to connect to localhost:5900 to view the virtual machine interface."
echo "If you need to stop the virtual machine, you can use the 'kill' command to terminate the process."
```

Note: If you install a VNC server inside the virtual machine, you need to log in via port `6900`.
