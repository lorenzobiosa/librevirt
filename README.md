# Custom AlmaLinux Installer with ZFS Root, KVM Hypervisor, Secure Boot, and Open vSwitch Networking

This document outlines the steps required to build a custom AlmaLinux ISO that:

- Supports installing the root filesystem on ZFS, including ZFS RAID features.
- Allows filesystem, RAID level, and encryption selection via the installer GUI.
- Installs and configures a KVM hypervisor with encrypted storage pools.
- Uses Open vSwitch (OVS) for virtualized networking.
- Includes signed ZFS kernel modules to enable boot with Secure Boot (similar to Proxmox).
- Provides a web-based management UI using Cockpit with KVM support.

---

## 1. Prepare the Build Environment

1. **Host System**: Use an AlmaLinux 8/9 VM or container matching your target OS version.
2. **Install Required Packages**:
   ```bash
   sudo dnf install -y centos-release-nfv-openvswitch epel-release
   sudo dnf install -y https://zfsonlinux.org/epel/zfs-release-2-3$(rpm --eval "%{dist}").noarch.rpm
   sudo dnf update -y
   sudo dnf install -y osbuild-composer composer-cli rpm-build mock \
                       dracut-tools pesign mokutil git wget \
                       python3-blivet pykickstart \
                       @virtualization\ host qemu-kvm libvirt libvirt-client virt-install \
                       cockpit cockpit-machines cockpit-networkmanager openvswitch3.5 \
                       kernel-devel zfs
   ```

3. **Enable and Start Services**:

   ```bash
   sudo systemctl enable --now libvirtd.socket
   sudo systemctl enable --now cockpit.socket
   sudo systemctl enable --now openvswitch.service
   ```
4. **Generate Signing Keys**:

   ```bash
   openssl req -new -x509 -newkey rsa:4096 -keyout MOK.key \
               -out MOK.crt -days 3650 -nodes \
               -subj "/CN=My ZFS Module Signing/"
   ```

   * You will enroll `MOK.crt` into the shim database during installation using MokEnroll.

---

## 3. Sign the ZFS Kernel Modules

To allow module loading under Secure Boot, the `.ko` files must be signed.

1. **Extract Modules**:

   ```bash
   mkdir -p /tmp/zfs-modules
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

   * Create a Dracut module under `/usr/lib/dracut/modules.d/95zfs-sign/` that signs ZFS modules in `install_modules()`.

---

## 4. Customize Installer GUI for ZFS, RAID, and Encryption

To allow users to select ZFS, RAID RAID levels, and encryption in Anaconda:

1. **Install Blivet GUI and Cryptsetup**:

   ```bash
   sudo dnf install -y anaconda-blivet-gui python3-blivet-gui python3-pykickstart cryptsetup
   ```
2. **Extend Blivet Plugin**:

   * Add `zfs_plugin.py` to `anaconda/pyanaconda/blivet_plugins/` that registers:

     * **Filesystem**: ZFS
     * **RAID Levels**: single, mirror, raidz, raidz2, raidz3
     * **Encryption Options**: LUKS or ZFS native encryption
   * Map GUI choices (`$fs_type`, `$raid_level`, `$encrypt_type`) to Kickstart variables.
3. **Modify UI Layout**:

   * Update `/usr/share/anaconda/ui/target.blivet` or corresponding XML to include dropdowns for filesystem, RAID, and encryption when ZFS is selected.
4. **Test GUI**:

   ```bash
   anaconda --testmode --storage-only
   ```

---

## 5. Create a Kickstart Configuration for ZFS with Encryption, KVM, and Open vSwitch

Use a Kickstart file (`ks.cfg`) to automate ZFS pool creation, encryption, hypervisor setup, and OVS networking.

```kickstart
zerombr
clearpart --all --initlabel
bootloader --location=mbr

%packages
@^minimal
@virtualization
cockpit
cockpit-machines
cockpit-networkmanager
openvswitch
anaconda-blivet-gui
zfs
zfs-dkms
kmod-zfs
cryptsetup
%end

%pre --interpreter=/usr/bin/bash
# Variables from GUI plugin: $raid_level, $encrypt_type (luks|zfs)
ZPOOL_OPTS="-o ashift=12 -O compression=lz4 -O atime=off"
DEVICE_LIST=(/dev/vda1 /dev/vdb1)
case "$raid_level" in
  single) STRAT="single";;
  mirror) STRAT="mirror";;
  raidz)  STRAT="raidz";;
  raidz2) STRAT="raidz2";;
  raidz3) STRAT="raidz3";;
esac
# Encrypt VM storage if using LUKS
if [[ "$encrypt_type" == "luks" ]]; then
  cryptsetup luksFormat /dev/vdc1
  cryptsetup open /dev/vdc1 cryptvm
  VM_DEV="/dev/mapper/cryptvm"
else
  VM_DEV="/dev/vdc1"
fi
# Create system ZFS pool
ezpool create -f $ZPOOL_OPTS rpool $STRAT ${DEVICE_LIST[@]}
# Create VM storage ZFS pool
ezpool create -f -o mountpoint=/var/lib/libvirt/images vm_pool $VM_DEV
# Create datasets
zfs create -o mountpoint=/ rpool/ROOT
zfs create -o mountpoint=/home rpool/HOME
%end

%post --interpreter=/usr/bin/bash
# Enable services
systemctl enable --now libvirtd.service
systemctl enable --now openvswitch.service
# Configure default OVS bridge
ovs-vsctl add-br ovsbr0 || true
nmcli connection add type ovs-bridge con-name ovsbr0 ifname ovsbr0 || true
nmcli connection up ovsbr0 || true
%end
```

---

## 6. Build the Custom ISO with Lorax

1. **Prepare a Lorax Template**:

   * Include `ks.cfg`, Blivet plugin, and ZFS/LUKS repositories.
   * Reference these in the Lorax XML template.
2. **Run Lorax Composer**:

   ```bash
   lorax-composer --tree=/path/to/repo \
                  --ks=/path/to/ks.cfg \
                  --product="AlmaLinux KVM Hypervisor ZFS SecureBoot OVS" \
                  --version=8.9 \
                  --release=custom1 \
                  /path/to/output/
   ```

   * The ISO will appear in `/path/to/output/images/`.

---

## 7. Integrate Secure Boot (Shim and MOK)

This section explains how to include UEFI boot files in the ISO and enroll your MOK certificate:

1. **Prepare a directory for EFI files**:

   ```bash
   mkdir -p /tmp/iso-efi/EFI/BOOT
   ```
2. **Copy shim, GRUB, and MOK certificate**:

   ```bash
   cp /usr/share/shim/shimx64.efi /tmp/iso-efi/EFI/BOOT/shimx64.efi
   cp /usr/share/grub2/x86_64-efi/grubx64.efi /tmp/iso-efi/EFI/BOOT/grubx64.efi
   cp /path/to/MOK.crt /tmp/iso-efi/EFI/BOOT/MOK.crt
   ```
3. **Merge EFI files into the Lorax tree before ISO build**:

   * If using a custom tree directory for Lorax (`--tree=/path/to/repo`), add:

     ```bash
     cp -r /tmp/iso-efi/EFI/BOOT /path/to/repo/EFI/
     ```
   * Ensure the Lorax template XML includes:

     ```xml
     <addon>
       <name>EFI Files</name>
       <type>file</type>
       <source>/path/to/repo/EFI</source>
       <dest>/EFI</dest>
     </addon>
     ```
4. **Build the ISO**:

   ```bash
   lorax-composer --tree=/path/to/repo \
                  --ks=/path/to/ks.cfg \
                  ...
   ```
5. **Enroll MOK Certificate**:

   * Boot from the ISO. At the shim prompt, press a key to enroll a new key.
   * Run:

     ```bash
     mokutil --import MOK.crt
     ```
   * Reboot, select “Enroll MOK” from the blue screen, and follow prompts.
6. **Verify enrollment and module signatures**:

   ```bash
   mokutil --list-enrolled
   modinfo -F signer zfs
   ```

---

## 8. Testing and Debugging Testing and Debugging

* **Dracut Debug**:

  ```bash
  dracut -f --regenerate-all --debug
  ```
* **UEFI Shell**: Mount the ISO in an EFI shell to confirm shim, GRUB, and MOK certificate.
* **Cockpit**: Access at `https://<host>:9090/` to manage VMs and OVS bridges.

---

### Summary

1. Build host: install `lorax`, `mock`, `dracut-devel`, virtualization and networking tools, Blivet GUI.
2. Compile ZFS RPMs; create MOK keys; sign modules.
3. Extend Anaconda GUI for ZFS, RAID, and encryption.
4. Kickstart: create encrypted ZFS pools, VM storage, and OVS bridge.
5. Build ISO with `lorax-composer` including hypervisor and OVS.
6. Embed shim/GRUB, enroll MOK for Secure Boot.
7. Test: installation flow, encryption, KVM, OVS, and Cockpit UI.
