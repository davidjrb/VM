
# How to Create a Minimal Debian VM on a KVM Host (Text-Mode, No GUI)

This guide explains how to set up a **lean Debian VM** on a KVM/QEMU host in **pure text mode** (no graphical environment). The final VM will have a **serial console** so you can use `virsh console`, and it will be suitable as a “golden template” for cloning.

---

## 1. Prerequisites & Setup

1. **Install KVM & Libvirt** (on the host, e.g., Ubuntu):
   ```bash
   sudo apt update
   sudo apt install qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils virtinst
   ```
   - Log out/in (or reboot) so your user is added to the `libvirt` and `kvm` groups.

2. **(Optional) Check Nested Virtualization**  
   If this host is itself a VM and you plan on nesting more VMs, ensure:
   ```bash
   cat /sys/module/kvm_amd/parameters/nested   # For AMD
   cat /sys/module/kvm_intel/parameters/nested # For Intel
   ```
   It should show `1` or `Y`.

3. **Download a Debian netinst ISO**  
   Place it somewhere on the host, for example:
   ```
   /srv/shared/debian-12.x.x-amd64-netinst.iso
   ```

---

## 2. Create a Disk for the Debian VM

```bash
sudo qemu-img create -f qcow2 /var/lib/libvirt/images/<VM_NAME>.qcow2 20G
```
- `<VM_NAME>` can be something like `debian-golden`.
- `20G` is thin-provisioned (it only occupies space as data is written).

---

## 3. Launch virt-install With Serial Console

Use `--location` (not `--cdrom`) so you can pass `--extra-args="console=ttyS0"` for a text-based installer:

```bash
sudo virt-install \
  --name <VM_NAME> \
  --ram 2048 \
  --vcpus 2 \
  --cpu host \
  --disk path=/var/lib/libvirt/images/<VM_NAME>.qcow2,size=20,format=qcow2 \
  --location /srv/shared/debian-12.x.x-amd64-netinst.iso \
  --os-variant debian12 \
  --graphics none \
  --extra-args="console=ttyS0"
```

### Key Points
- **`--location`** extracts kernel/initrd from the ISO, allowing kernel args.
- **`--graphics none`** + `console=ttyS0` ensures a **serial** text-mode installer.
- Adjust `--ram`, `--vcpus`, or disk size as needed.

`virt-install` starts the VM and attaches you to its serial console automatically. If you see a blank screen, press **Enter** to prompt output.

---

## 4. Debian netinst Installation Steps

1. **Language/Keyboard/Locale**: Pick what you need.  
2. **Hostname**: e.g., `<VM_HOSTNAME>`.  
3. **Root Password & New User**: Debian typically prompts for both.  
4. **Partitioning**: “Guided - use entire disk” is simplest.  
5. **Software Selection**:
   - **Uncheck** “Debian desktop environment” if you want minimal.
   - **Check** “SSH server” and keep “standard system utilities.”
6. **Install GRUB**: Usually to `/dev/vda`.
7. **Finish Installation**: The system reboots.

---

## 5. First Boot & Serial Console

Upon reboot, you should see:

```
Booting 'Debian GNU/Linux'
Loading Linux ...
Loading initial ramdisk ...
Debian GNU/Linux 12 <VM_HOSTNAME> ttyS0
<VM_HOSTNAME> login:
```

- Log in with the credentials you set.
- If the screen is blank, press **Enter** a few times.

---

## 6. Post-Install Basics

- If you **didn’t** get “SSH server” installed, do:
  ```bash
  sudo apt update
  sudo apt install openssh-server
  ```
- By default, Debian might not add your user to `sudo`. Fix:
  ```bash
  su -
  usermod -aG sudo <YOUR_USERNAME>
  exit
  ```
  Log out/in again to apply the group change.

- **Update** your system:
  ```bash
  sudo apt update
  sudo apt upgrade
  ```

---

## 7. Verify SSH Connectivity

1. **Check VM IP**:
   ```bash
   ip addr
   ```
   or from the host:
   ```bash
   sudo virsh domifaddr <VM_NAME>
   ```
2. **SSH** from the host:
   ```bash
   ssh <YOUR_USERNAME>@<VM_IP>
   ```
   If “Connection refused,” ensure `openssh-server` is running.

#### Passwordless SSH (Optional)

1. **Generate or re-use a key** on the host:
   ```bash
   ssh-keygen -t ed25519
   ```
2. **Copy key** to VM:
   ```bash
   ssh-copy-id -i ~/.ssh/id_ed25519.pub <YOUR_USERNAME>@<VM_IP>
   ```
3. **Login**:
   ```bash
   ssh <YOUR_USERNAME>@<VM_IP>
   ```
   No password needed.

---

## 8. Autostart on Host Reboot

If you want the VM to start automatically when the host reboots:

```bash
sudo virsh autostart <VM_NAME>
```

Check:

```bash
sudo virsh list --all --autostart
```

---

## 9. Optional Packages

- **net-tools, iputils-ping**: Network utilities that might be missing in minimal installs.
  ```bash
  sudo apt install net-tools iputils-ping
  ```
- **nano or vim**: For easier text editing.
  ```bash
  sudo apt install nano
  ```
- **fish**: Another shell.
  ```bash
  sudo apt install fish
  chsh -s /usr/bin/fish
  ```
- **ca-certificates**: For HTTPS validation.
  ```bash
  sudo apt install ca-certificates
  ```

---

## 10. Cloning the “Golden” VM

Once you’re happy with the minimal Debian setup, you can treat it as a **golden template**. To clone:

1. **Shut down** the VM if running:
   ```bash
   sudo virsh shutdown <VM_NAME>
   ```
2. **Copy** the disk:
   ```bash
   sudo cp /var/lib/libvirt/images/<VM_NAME>.qcow2 /var/lib/libvirt/images/<NEW_VM_NAME>.qcow2
   ```
3. **Import** the clone:
   ```bash
   sudo virt-install \
     --name <NEW_VM_NAME> \
     --import \
     --disk /var/lib/libvirt/images/<NEW_VM_NAME>.qcow2,format=qcow2 \
     --memory 2048 \
     --vcpus 2 \
     --os-variant debian12 \
     --network network=default \
     --graphics none
   ```
This spawns a new VM with its own MAC address, IP, and hostname. You can then customize each cloned VM further.

---

## Summary

1. **Create** a qcow2 disk.  
2. **Install** Debian with `virt-install --location ... --extra-args="console=ttyS0"`.  
3. **Pick** minimal software (SSH, standard utilities).  
4. **Reboot** into a fully text-based Debian VM with `virsh console`.  
5. **Configure** SSH, sudo, and any packages you want.  
6. **Autostart** if desired.  
7. **Clone** the disk for future VMs.

That’s all! You now have a **lightweight** Debian server VM using only a **serial console**—no GUI required.
```
