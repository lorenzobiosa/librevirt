# Custom AlmaLinux Installer with ZFS Root, KVM Hypervisor, and Secure Boot

This document outlines the steps required to build a custom AlmaLinux ISO that:

- Supports installing the root filesystem on ZFS, including ZFS RAID features.
- Allows filesystem, RAID level, and encryption selection via the installer GUI.
- Installs and configures a KVM hypervisor with encrypted storage pools.
- Includes signed ZFS kernel modules to enable boot with Secure Boot (similar to Proxmox).

---

## 1. Prepare the Build Environment

1. **Host System**: Use an AlmaLinux 8/9 VM or container matching your target OS version.
2. **Install Required Packages**:
   ```bash
   sudo dnf install -y lorax-composer rpm-build mock \
                       dracut-devel pesign mokutil git wget \
                       anaconda-blivet-gui python3-blivet-gui python3-pykickstart \
                       @virtualization qemu-kvm libvirt libvirt-client virt-install
   ```
3. **Enable and Start libvirtd**:

   ```bash
   sudo systemctl enable --now libvirtd.socket
   ```
4. **Generate Signing Keys**:

   ```bash
   openssl req -new -x509 -newkey rsa:4096 -keyout MOK.key \
               -out MOK.crt -days 3650 -nodes \
               -subj "/CN=My ZFS Module Signing/"
   ```

   * You will enroll `MOK.crt` into the shim database during installation using MokEnroll.

---

## 2. Download and Build ZFS Packages

1. **Clone the ZFS-on-Linux Repository**:

   ```bash
   git clone https://github.com/openzfs/zfs.git
   cd zfs
   git checkout stable-2.1   # or the latest stable branch compatible with RHEL
   ```
2. **Generate RPM Packages**:

   ```bash
   ./autogen.sh
   ./configure --prefix=/usr --with-config=kernel
   make rpm
   ```

   * This produces `zfs-dkms`, `kmod-zfs`, and their dependencies.

---

## 3. Sign the ZFS Kernel Modules

To allow module loading under Secure Boot, the `.ko` files must be signed.

1. **Install and Extract Modules**:

   ```bash
   find /usr/lib/modules/$(uname -r) -type f \
     -name 'zfs*.ko' -o -name 'spl*.ko' \
     -exec cp --parents {} /tmp/zfs-modules/ \;
   ```
2. **Sign Each Module**:

   ```bash
   for module in $(find /tmp/zfs-modules -type f -name '*.ko'); do
     /usr/src/kernels/$(uname -r)/scripts/sign-file sha256 \
       /path/to/MOK.key /path/to/MOK.crt "$module"
   done
   ```
3. **Automate Signing in Dracut**:

   * Create a Dracut module in `/usr/lib/dracut/modules.d/95zfs-sign/` that hooks into `install_modules()` to sign all ZFS modules during initramfs creation.

---

## 4. Customize Installer GUI for ZFS, RAID, and Encryption

To let users select ZFS filesystem, RAID levels, and LUKS or ZFS native encryption in Anaconda:

1. **Install Blivet GUI and Kickstart Packages**:

   ```bash
   sudo dnf install -y anaconda-blivet-gui python3-blivet-gui python3-pykickstart cryptsetup
   ```
2. **Extend Blivet Plugin**:

   * In `anaconda/pyanaconda/blivet_plugins/`, update `zfs_plugin.py` to register:

     * **Filesystem**: ZFS
     * **RAID Levels**: mirror, raidz, raidz2, raidz3
     * **Encryption Options**: LUKS (via `cryptsetup`) or ZFS native encryption (`encryption=on`)
   * Map GUI selections (\$fs\_type, \$raid\_level, \$encrypt\_type) to Kickstart variables.
3. **Modify Anaconda UI Layout**:

   * Add dropdown menus for filesystem, RAID, and encryption under `/usr/share/anaconda/ui/target.blivet` or companion XML.
4. **Test GUI**:

   ```bash
   anaconda --testmode --storage-only
   ```

   * Verify “ZFS”, RAID, and “Encrypt with LUKS/ZFS” options appear and produce correct storage definitions.

---

## 5. Create a Kickstart Configuration for ZFS with Encryption and KVM

Use a Kickstart file (`ks.cfg`) to automate pool creation, encryption, and hypervisor setup.

```kickstart
zerombr
clearpart --all --initlabel
bootloader --location=mbr

%packages
@^minimal
@virtualization
anaconda-blivet-gui
zfs
zfs-dkms
kmod-zfs
cryptsetup
%end

%pre --interpreter=/usr/bin/bash
# GUI-provided variables: $raid_level, $encrypt_type (luks|zfs), $vm_storage
ZPOOL_OPTS="-o ashift=12 -O compression=lz4 -O atime=off"
DEVICES=(/dev/vda1 /dev/vdb1)
case "$raid_level" in
  mirror) STRAT="mirror";;
  raidz)  STRAT="raidz";;
  raidz2) STRAT="raidz2";;
  *)      STRAT="single";;
esac
# Create encrypted container if using LUKS
if [[ "$encrypt_type" == "luks" ]]; then
  cryptsetup luksFormat /dev/vdc1
  cryptsetup open /dev/vdc1 cryptvm
  VM_DEV="/dev/mapper/cryptvm"
else
  VM_DEV="/dev/vdc1"
fi
# ZFS pool for system
zpool create -f $ZPOOL_OPTS rpool $STRAT ${DEVICES[@]}
# ZFS pool for VM storage
zpool create -f -o mountpoint=/var/lib/libvirt/images vm_pool $VM_DEV
# Create root datasets
zfs create -o mountpoint=/ rpool/ROOT
zfs create -o mountpoint=/home rpool/HOME

%post --interpreter=/usr/bin/bash
```

# Enable and start virtualization

```
systemctl enable --now libvirtd.service
```

---

## 6. Build the Custom ISO with Lorax

1. **Prepare a Lorax Template**:
   - Include your `ks.cfg`, custom Blivet plugin, and ZFS/LUKS repos in the tree.
   - Reference these in the Lorax XML template.
2. **Run Lorax Composer**:
   ```bash
   lorax-composer --tree=/path/to/repo \
                  --ks=/path/to/ks.cfg \
                  --product="AlmaLinux KVM Hypervisor ZFS SecureBoot" \
                  --version=8.9 \
                  --release=custom1 \
                  /path/to/output/
   ```

* The ISO will be in `/path/to/output/images/`.

---

## 7. Integrate Secure Boot (Shim and MOK)

1. **Include UEFI Boot Files**:

   * Place `shim.efi`, `grubx64.efi`, and `MOK.crt` under `EFI/BOOT/` in the ISO.
2. **Enroll MOK Certificate**:

   ```bash
   mokutil --import MOK.crt
   ```

   * Reboot and follow the UEFI UI to enroll.
3. **Verify**:

   ```bash
   mokutil --list-enrolled
   modinfo -F signer zfs
   ```

---

## 8. Testing and Debugging

* **Proxmox Reference**:

  ```text
  https://git.proxmox.com/git/pve-zfs.git
  ```
* **Dracut Debug**:

  ```bash
  dracut -f --regenerate-all --debug
  ```
* **UEFI Shell**: Mount the ISO in EFI shell to confirm shim, GRUB, MOK.

---

### Summary

1. Build host: `lorax`, `mock`, `dracut-devel`, virtualization tools, Blivet GUI.
2. Compile ZFS RPMs; create MOK keys; sign modules.
3. Extend Anaconda GUI for ZFS, RAID, encryption.
4. Kickstart: create ZFS pools, LUKS or ZFS encryption, VM storage pool.
5. Build ISO with `lorax-composer` including hypervisor and custom UI.
6. Embed shim/GRUB, enroll MOK for Secure Boot.
7. Test: installation, encryption, KVM, signed modules.

```
```
