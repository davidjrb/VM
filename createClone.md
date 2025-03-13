Below is a **generic** step-by-step on shutting down your golden VM, then cloning it into a new VM (called “mockingbird”). It assumes:

- You have a **host** running KVM/libvirt.
- You have a **“golden” Debian VM** that you want to clone.
- The golden VM is set to autostart, so it boots when the host reboots.

---

## 1. (Optional) Reboot the Host

If your host system prompts for a reboot (for kernel updates, etc.), do:

```bash
sudo reboot
```

- Once it’s back online, SSH back in.  
- Since the golden VM is autostart-enabled, it should be running again.

---

## 2. Shut Down the Golden VM

1. **Check current VMs**:
   ```bash
   sudo virsh list --all
   ```
   Your golden VM should appear as “running.”

2. **Shut it down** gracefully:
   ```bash
   sudo virsh shutdown debian-golden
   ```
   or forcefully:
   ```bash
   sudo virsh destroy debian-golden
   ```
   (Graceful shutdown is preferred.)

3. **Confirm** it’s off:
   ```bash
   sudo virsh list --all
   ```
   Now it should say “shut off.”

---

## 3. Copy the Disk to a New Name

Assuming your golden VM disk is `/var/lib/libvirt/images/debian-golden.qcow2`:

```bash
sudo cp /var/lib/libvirt/images/debian-golden.qcow2 \
        /var/lib/libvirt/images/mockingbird.qcow2
```

You now have a cloned disk named **mockingbird.qcow2**.

---

## 4. Create the New “mockingbird” VM With `virt-install --import`

Run:

```bash
sudo virt-install \
  --name mockingbird \
  --import \
  --disk /var/lib/libvirt/images/mockingbird.qcow2,format=qcow2 \
  --memory 2048 \
  --vcpus 2 \
  --os-variant debian12 \
  --network network=default \
  --graphics none
```

### Explanation
- **`--import`**: Tells virt-install you already have a disk with an OS installed, so no installer is needed.
- The VM automatically gets a **new MAC address**, so it won’t conflict with the golden VM.
- **`--network network=default`**: Uses the default NAT network unless you specify otherwise.

---

## 5. Verify & Access the New VM

1. **Check console**:
   ```bash
   sudo virsh console mockingbird
   ```
   Press **Enter** if it looks blank—eventually, you should see a login prompt.

2. **Credentials**: Use the same username/password you set in the golden VM (unless you changed them on the clone’s disk).

3. **IP address**:
   ```bash
   ip addr
   ```
   or from the host:
   ```bash
   sudo virsh domifaddr mockingbird
   ```
   That IP can be used to SSH in.

4. **(Optional) Change Hostname**: Inside the VM, edit `/etc/hostname` and/or `/etc/hosts` if you want “mockingbird” to show up as the system’s hostname.

---

### Summary

1. **Reboot** host (if you have pending updates).  
2. **Shut down** your golden VM.  
3. **Copy** the disk file to a new name.  
4. **Run `virt-install --import`** to create the new VM from that disk.  
5. **Access** the console or SSH to manage your clone.  

You’ve successfully created a new VM called **mockingbird** from your Debian “golden” template.
