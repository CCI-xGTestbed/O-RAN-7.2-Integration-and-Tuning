# Comprehensive Deployment Guide: srsRAN with Benetel RAN650 O-RU

This guide documents the complete process to deploy srsRAN (commit 122a) with a Benetel 650 O-RU using an Intel E810 NIC with PTP synchronization, DPDK acceleration, and SR-IOV configuration.

---

## Table of Contents

1. [Hardware Requirements](#hardware-requirements)
2. [Server Installation and Basic Setup](#server-installation-and-basic-setup)
   - [BIOS Configuration](#bios-configuration)
   - [OS Installation](#os-installation)
   - [Real-time Kernel Installation](#real-time-kernel-installation)
3. [Intel E810 NIC Setup](#intel-e810-nic-setup)
   - [Driver Installation](#driver-installation)
   - [Firmware Updates](#firmware-updates)
   - [PTP Tools Installation](#ptp-tools-installation)
4. [DPDK Installation](#dpdk-installation)
   - [GRUB Configuration for Hugepages and IOMMU](#grub-configuration-for-hugepages-and-iommu)
   - [Disable NTP and Complete Setup](#disable-ntp-and-complete-setup)
5. [Open5GS Core Installation](#open5gs-core-installation)
   - [MongoDB Installation](#mongodb-installation)
   - [Open5GS Installation](#open5gs-installation)
   - [Create Network Tunnel for UEs](#create-network-tunnel-for-ues)
   - [Configure Open5GS WebUI](#configure-open5gs-webui)
6. [O-RU Connectivity Setup](#o-ru-connectivity-setup)
7. [PTP Configuration and Synchronization](#ptp-configuration-and-synchronization)
8. [SR-IOV and DPDK Configuration](#sr-iov-and-dpdk-configuration)
   - [Manual SR-IOV Configuration](#manual-sr-iov-configuration)
   - [Determine the VF PCI Address](#determine-the-vf-pci-address)
   - [Bind VF to VFIO-PCI for DPDK](#bind-vf-to-vfio-pci-for-dpdk)
   - [Automate SR-IOV Configuration at Boot](#automate-sr-iov-configuration-at-boot)
9. [O-RU Firmware Upgrade](#o-ru-firmware-upgrade)
10. [srsRAN Configuration](#srsran-configuration)
    - [Configure the gNB](#configure-the-gnb)
    - [Configure the O-RU](#configure-the-o-ru)
11. [System Testing and Verification](#system-testing-and-verification)
    - [Starting the System](#starting-the-system)
    - [Verification](#verification)
12. [Troubleshooting and Lessons Learned](#troubleshooting-and-lessons-learned)

---

## Hardware Requirements

The detailed hardware and software components can be found in the README.md file. 

---

## Server Installation and Basic Setup

### BIOS Configuration

Before installing the OS, configure the server BIOS:

1. **System Setup > System BIOS > System Profile Settings**
   - Set the profile to **Performance** mode
   - Finish
2. **Boot Settings > Boot Mode** set to **UEFI**
   - Finish
3. **Device Settings > RAID Controller > Action/Configure > Auto Configure RAID 0**
   - Confirm/Yes

### OS Installation

Install Ubuntu 22.04.5 LTS Server version with standard installation options:

- **Your name:** `[YOUR_DISPLAY_NAME]`
- **Username:** `[YOUR_USERNAME]`
- **Server name:** `[YOUR_SERVER_HOSTNAME]`

After installation, run initial updates:

```bash
sudo apt install net-tools
sudo apt update
sudo apt upgrade
```

Verify that the PTP card is detected:

```bash
ifconfig
ip a
```

Configure IDRAC for remote server management. The login username for IDRAC is `root` and the password is the machine password configured during Ubuntu installation.

### Real-time Kernel Installation

Install the real-time kernel:

```bash
sudo apt install linux-image-5.15.0-1032-realtime \
                 linux-headers-5.15.0-1032-realtime \
                 linux-modules-extra-5.15.0-1032-realtime
```

Verify the `gnss` module is installed:

```bash
find /lib/modules/5.15.0-1032-realtime -name "*gnss*"
```

> **Note:** After installation, do **not** reboot yet — we need to prepare the GRUB configuration first. It will keep giving some warnings; just `Tab` into `Ok` and keep going.

---

## Intel E810 NIC Setup

### Driver Installation

Identify the PCI addresses for the NIC:

```bash
lspci | grep -i ethernet
```

> **Note:** Record the PCI bus prefix shown here (e.g., `0d:00`). This will be referred to as `[YOUR_PCI_BUS]` throughout the guide. The full Physical Function (PF) PCI address will be `0000:[YOUR_PCI_BUS].0` (referred to as `[YOUR_PF_PCI_ADDRESS]`).

Check the interface mapping:

```bash
ls -l /sys/class/net/ | grep [YOUR_PCI_BUS]
```

Install dependencies:

```bash
sudo apt update
sudo apt install build-essential linux-headers-$(uname -r) git cmake pkg-config \
    libnl-3-dev libnl-route-3-dev libcap-dev libpcap-dev libnuma-dev
```

Install the ICE driver v1.13.7:

```bash
git clone https://github.com/intel/ethernet-linux-ice.git
cd ethernet-linux-ice
git checkout v1.13.7

# Remove existing modules
sudo rmmod irdma
sudo rmmod ice

# Build and install
cd src
sudo make -j install
```

Verify installation:

```bash
modinfo ice
sudo modprobe ice
sudo ethtool -i enp13s0f0
```

Since we installed `ice` for the generic Ubuntu kernel, we now need to build it again for the realtime kernel:

```bash
sudo make BUILD_KERNEL=5.15.0-1032-realtime KSRC=/lib/modules/5.15.0-1032-realtime/build
sudo make BUILD_KERNEL=5.15.0-1032-realtime install
sudo depmod -a 5.15.0-1032-realtime
```

Verify that the gnss module is loaded properly:

```bash
modinfo -k 5.15.0-1032-realtime gnss
```

You can see all dependencies for `ice` with:

```bash
modinfo -k 5.15.0-1032-realtime ice | grep depends
```

### Firmware Updates

Update the NIC firmware to version 4.40 (or latest may work):

1. Download the NVM Update Utility for Intel E810 from:
   <https://www.intel.com/content/www/us/en/download/19624/812366/non-volatile-memory-nvm-update-utility-for-intel-ethernet-network-adapter-e810-series.html>
2. Transfer it to the server.
3. Extract and update:

```bash
unzip E810_NVMUpdatePackage_v4_40.zip
cd E810/Linux_x64
chmod 755 nvmupdate64e
sudo ./nvmupdate64e
```

Follow the on-screen prompts to update the firmware. When prompted to back up NVM images, select **Y**.

Verify the update:

```bash
sudo ethtool -i enp13s0f0 | grep firmware
```

Expected output:

```
firmware-version: 4.40 0x8001c96a 1.3534.0
```

### PTP Tools Installation

There is a similar procedure in the [srsRAN Documentation](https://docs.srsran.com/projects/project/en/latest/tutorials/source/oranRU/source/index.html).

**Install LinuxPTP:**

```bash
git clone git://git.code.sf.net/p/linuxptp/code linuxptp-4.3
cd linuxptp-4.3
make
sudo make install
```

**Install Synce4l:**

```bash
git clone http://github.com/intel/synce4l synce4l
cd synce4l
sudo apt-get install libnl-genl-3-dev
make
sudo make install
```

**Verify PTP capability:**

```bash
# Get subsystem device ID
cat /sys/class/net/enp13s0f0/device/subsystem_device

# Check timestamping capabilities
ethtool -T enp13s0f0

# Find all interfaces with the same subsystem device (replace {} with output of the first command)
grep {} /sys/class/net/*/device/subsystem_device | awk -F"/" '{print $5}'
```

We have provided our PTP Configuration file for usage with phc2sys in this repository, you are free to use it when you go to run.

---

## DPDK Installation

Install DPDK version 23.11:

```bash
sudo apt install build-essential tar wget python3-pip libnuma-dev \
                 meson ninja-build python3-pyelftools

wget https://fast.dpdk.org/rel/dpdk-23.11.tar.xz
tar xvf dpdk-23.11.tar.xz dpdk-23.11/
cd dpdk-23.11/

meson setup build
cd build
ninja

sudo meson install
sudo ldconfig
```

### GRUB Configuration for Hugepages and IOMMU

Create a custom GRUB entry for the real-time kernel (Not significantly different from the procedure found on the [srsRAN Documentation](https://docs.srsran.com/projects/project/en/latest/tutorials/source/dpdk/source/index.html):

```bash
sudo nano /etc/grub.d/40_custom
```

Add the following content:

> **Note:** Replace `[YOUR_BOOT_PARTITION]` with your boot partition (e.g., `hd0,gpt2`) and `[YOUR_ROOT_LV_PATH]` with your root logical volume path (e.g., `/dev/mapper/ubuntu--vg-ubuntu--lv`). Run `lsblk` and `cat /etc/fstab` to determine the correct values for your system.

```bash
#!/bin/sh
exec tail -n +3 $0
# This file provides an easy way to add custom menu entries. Simply type the
# menu entries you want to add after this comment. Be careful not to change
# the 'exec tail' line above.

menuentry 'Realtime Kernel with 1G Hugepages + IOMMU for DPDK' {
    insmod gzio
    insmod part_gpt
    insmod ext2
    insmod lvm
    set root='[YOUR_BOOT_PARTITION]'
    linux /vmlinuz-5.15.0-1032-realtime root=[YOUR_ROOT_LV_PATH] ro default_hugepagesz=1G hugepagesz=1G hugepages=2 intel_iommu=on iommu=pt
    initrd /initrd.img-5.15.0-1032-realtime
}
```

Modify `/etc/default/grub`:

```bash
sudo nano /etc/default/grub
```

Update with:

```
GRUB_DEFAULT="Realtime Kernel with 1G Hugepages + IOMMU for DPDK"
GRUB_TIMEOUT_STYLE=menu
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR=`lsb_release -i -s 2> /dev/null || echo Debian`
GRUB_CMDLINE_LINUX_DEFAULT="default_hugepagesz=1G hugepagesz=1G hugepages=2 intel_iommu=on iommu=pt"
GRUB_CMDLINE_LINUX=""
```

Apply the changes:

```bash
sudo chmod +x /etc/grub.d/40_custom
sudo update-grub
```

### Disable NTP and Complete Setup

Disable the NTP service to prevent interference with PTP:

```bash
sudo systemctl stop systemd-timesyncd
sudo systemctl disable systemd-timesyncd
sudo timedatectl set-ntp false
```

Now reboot the system:

```bash
sudo reboot
```

After reboot, verify hugepages configuration:

```bash
grep huge /proc/cmdline
```

---

## Open5GS Core Installation

### MongoDB Installation

```bash
sudo apt update
sudo apt install gnupg

curl -fsSL https://pgp.mongodb.com/server-6.0.asc | \
    sudo gpg -o /usr/share/keyrings/mongodb-server-6.0.gpg --dearmor

echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-6.0.gpg ] \
    https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/6.0 multiverse" | \
    sudo tee /etc/apt/sources.list.d/mongodb-org-6.0.list

sudo apt update
sudo apt install -y mongodb-org
sudo systemctl start mongod
sudo systemctl enable mongod
```

### Open5GS Installation

Install dependencies:

```bash
sudo apt install python3-pip python3-setuptools python3-wheel ninja-build \
    build-essential flex bison git cmake libsctp-dev libgnutls28-dev \
    libgcrypt-dev libssl-dev libmongoc-dev libbson-dev libyaml-dev \
    libnghttp2-dev libmicrohttpd-dev libcurl4-gnutls-dev libnghttp2-dev \
    libtins-dev libtalloc-dev meson

if apt-cache show libidn-dev > /dev/null 2>&1; then
    sudo apt-get install -y --no-install-recommends libidn-dev
else
    sudo apt-get install -y --no-install-recommends libidn11-dev
fi
```

Build and install Open5GS:

```bash
git clone https://github.com/open5gs/open5gs
cd open5gs
meson build --prefix=`pwd`/install
ninja -C build
ninja -C build install
```

### Create Network Tunnel for UEs

Create a tunnel script in your desired directory:

```bash
sudo nano tunn.sh
```

Content of `tunn.sh`:

```bash
#!/bin/bash
sudo ip tuntap add name ogstun mode tun
sudo ip addr add 10.45.0.1/16 dev ogstun
sudo ip addr add 2001:db8:cafe::1/48 dev ogstun
sudo ip link set ogstun up

sudo sysctl -w net.ipv4.ip_forward=1
sudo sysctl -w net.ipv6.conf.all.forwarding=1

sudo iptables -t nat -A POSTROUTING -s 10.45.0.0/16 ! -o ogstun -j MASQUERADE
sudo ip6tables -t nat -A POSTROUTING -s 2001:db8:cafe::/48 ! -o ogstun -j MASQUERADE

sudo ufw disable
sudo iptables -I INPUT -i ogstun -j ACCEPT
```

Make it executable and run:

```bash
chmod +x tunn.sh
./tunn.sh
```

### Configure Open5GS WebUI

```bash
cd ~/open5gs/webui
HOSTNAME=[YOUR_SERVER_IP] npm run dev
```

Access the web interface from another computer at `http://[YOUR_SERVER_IP]:3000` and configure UE subscriber information with:

- IMSI matching your test SIM
- Authentication keys
- Network slice configuration
- APN settings

> **Note:** The default password for the Open5GS WebUI is `1423`.

---

## O-RU Connectivity Setup

Set up connectivity to the Benetel RAN650 O-RU:

```bash
sudo ip link set enp13s0f0 up
sudo ip addr add 10.10.0.1/24 dev enp13s0f0
sudo ifconfig enp13s0f0 mtu 9600 up
```

Verify connectivity:

```bash
ping 10.10.0.100
ssh root@10.10.0.100
```

Record the O-RU MAC address for later configuration. This will be referred to as `[YOUR_O-RU_MAC]` throughout the guide:

```bash
ssh root@10.10.0.100 "ifconfig eth0 | grep ether"
```

Example output:

```
ether [YOUR_O-RU_MAC]
```

---

## PTP Configuration and Synchronization

Set up PTP in hardware timestamping mode.

**Terminal 1 — Start ptp4l:**

```bash
sudo su
/usr/local/sbin/ptp4l -i enp13s0f0 -m \
    --domainNumber 24 \
    --masterOnly 1 \
    --slaveOnly 0 \
    --network_transport L2 \
    --tx_timestamp_timeout 1000 \
    --logAnnounceInterval -3 3 \
    --clockClass 6
```

**Terminal 2 — Start phc2sys:**

```bash
sudo su
/usr/local/sbin/phc2sys -s enp13s0f0 -w -m \
    -f [YOUR_PATH_TO_PTP_CONFIG]/srs-ptp-gm.cfg
```

**Monitor synchronization on the O-RU:**

```bash
ssh root@10.10.0.100
cd /tmp/logs/
tail -f /var/log/pcm4l
```

Wait until you see consistent offset values below ±100ns in the logs.

---

## SR-IOV and DPDK Configuration

The SR-IOV configuration allows sharing of the physical NIC port between PTP (running on the Physical Function) and DPDK (running on a Virtual Function).

**Step 1:** Check the MAC address of the E810 NIC:

```bash
ip a
```

**Step 2:** Reboot the Dell server and enter BIOS (F2):

1. Go to **System BIOS Settings > Integrated Devices** and make sure **SR-IOV Global Enable** is **Enabled**.
2. Go to **Device Settings**, select the NIC with the corresponding MAC address.
3. Go to **Device Level Configuration** and set **Virtualization Mode** to **SR-IOV** (default is None).
4. Reboot.

**Step 3:** Verify that SR-IOV capability is present:

```bash
sudo lspci -vvv -s [YOUR_PF_PCI_ADDRESS] | grep -i "single root" -A 10
ls /sys/class/net/enp13s0f0/device/ | grep sriov
cat /sys/class/net/enp13s0f0/device/sriov_totalvfs
```

### Manual SR-IOV Configuration

Create a Virtual Function (VF) from the Physical Function (PF):

```bash
echo 1 > /sys/class/net/enp13s0f0/device/sriov_numvfs
```

Verify the VF was created:

```bash
ip link show enp13s0f0
```

Set VF MAC address and disable spoof checking. Choose a MAC address for your DU — this will be referred to as `[YOUR_DU_MAC]` throughout the guide:

```bash
ip link set enp13s0f0 vf 0 mac [YOUR_DU_MAC] spoofchk off
```

Configure VLAN tag if needed:

```bash
ip link set enp13s0f0 vf 0 vlan 3 spoofchk off
```

### Determine the VF PCI Address

```bash
cd /home/[YOUR_USERNAME]/[YOUR_DPDK_PATH]/dpdk-23.11/usertools/
./dpdk-devbind.py --status
```

> **Note:** Record the VF PCI address from the output. This will be referred to as `[YOUR_VF_PCI_ADDRESS]` throughout the guide. It will typically be one sub-function after the PF (e.g., if PF is `0000:0d:00.0`, VF might be `0000:0d:01.0`).

### Bind VF to VFIO-PCI for DPDK

```bash
cd /home/[YOUR_USERNAME]/[YOUR_DPDK_PATH]/dpdk-23.11/usertools/
./dpdk-devbind.py --bind=vfio-pci [YOUR_VF_PCI_ADDRESS]
```

### Automate SR-IOV Configuration at Boot

Create a startup script:

```bash
sudo nano /usr/local/bin/setup-sriov-dpdk.sh
```

Content (adjust values for your specific machine configuration):

```bash
#!/bin/bash

# Enable 1 VF
echo 1 > /sys/class/net/enp13s0f0/device/sriov_numvfs

# Set VF MAC address and disable spoof check
ip link set enp13s0f0 vf 0 mac [YOUR_DU_MAC] spoofchk off

# Bind VF to DPDK-compatible driver
/home/[YOUR_USERNAME]/[YOUR_DPDK_PATH]/dpdk-23.11/usertools/dpdk-devbind.py --bind=vfio-pci [YOUR_VF_PCI_ADDRESS]
```

Make executable:

```bash
sudo chmod +x /usr/local/bin/setup-sriov-dpdk.sh
```

Create a systemd service:

```bash
sudo nano /etc/systemd/system/setup-sriov-dpdk.service
```

Content:

```ini
[Unit]
Description=Setup SR-IOV and DPDK VF Binding
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/setup-sriov-dpdk.sh
RemainAfterExit=true

[Install]
WantedBy=multi-user.target
```

Enable and start the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable setup-sriov-dpdk.service
sudo systemctl start setup-sriov-dpdk.service
```

---

## O-RU Firmware Upgrade

The O-RU firmware needs to be upgraded to version **1v1.4.0**.

```bash
ssh root@10.10.0.100
cd /tmp/
mkdir sw_build
cd sw_build
```

Transfer the `RAN650-1v1.4.0.zip` file to the O-RU using SCP or another file transfer method.

```bash
unzip ../RAN650-1v1.4.0.zip
eeprom2ram
./usr/sbin/swm-install.sh
./usr/sbin/swm-activate.sh
reboot
```

After the O-RU reboots, verify the new firmware:

```bash
ssh root@10.10.0.100
cat /etc/benetel-rootfs-version
```

Expected output:

```
RAN650-1v1.4.0
```

---

## srsRAN Configuration

### Configure the gNB

Please refer to the provided configuration CU and DU configuration files.

### Configure the O-RU

SSH into the O-RU and edit the configuration file:

```bash
ssh root@10.10.0.100
vi /etc/ru_config.cfg
```

Key settings to update:

```ini
mimo_mode=1_2_3_4_4x2
prach_format=short
c_plane_du_mac=[YOUR_DU_MAC]
u_plane_du_mac=[YOUR_DU_MAC]
u_plane_du_vlan_uplink=3
u_plane_du_vlan_downlink=3
c_plane_du_vlan=3
bandwidth_hz=100000000
centre_frequency_hz=3900000000
tx_power_dbm=27.000000
tdd_pattern_1=DDDSUU
tdd_pattern_2=UUUU
```

> **Important:** Ensure the MAC addresses, VLAN tags, and TDD pattern match between the gNB and O-RU configurations.

---

## Troubleshooting and Lessons Learned

### MTU Size Issues

The MTU size might reset from 9600 to 9000 when starting the gNB. The SR-IOV approach resolves this by separating the interfaces.

### PTP Synchronization Issues

If hardware timestamping fails with "driver rejected most general HWTSTAMP filter" errors, verify:

- Proper ICE driver installation (version 1.13.7)
- Proper firmware version (4.40)
- NTP is disabled
- IRDMA module is unloaded

### UE Attachment Failures

If UEs can see the network but can't attach:

- Verify O-RU firmware is updated to 1v1.4.0
- Try setting `tx_power_dbm` to `27` in `ru_config.cfg`
- Try setting `iq_scaling` to `16` in `ran650_du.yml`
- Use `short` PRACH format in `ru_config.cfg`

### Real-time Kernel Issues

If NIC interfaces disappear after installing the real-time kernel, reinstall the ICE driver after booting into the real-time kernel.
