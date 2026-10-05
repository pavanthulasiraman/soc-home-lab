# Lab setup: Ubuntu server

I built this Ubuntu Server VM as the foundation for my security monitoring lab. My Windows laptop has a limited 8 GB RAM, so I allocated 4 GB RAM to the VM. I will evaluate Wazuh's performance with this allocation of resources before adding another endpoint.

## Current status

I installed Ubuntu Server and verified its version, hostname, available memory, and root filesystem space. I also configured the host-only lab network and successfully logged in to the VM over SSH from my Windows laptop.

## Environment

| Setting | Value |
|---|---|
| Host operating system | Windows |
| Host RAM | 8 GB |
| Virtualization software | Oracle VirtualBox 7.2.20 |
| VM name and hostname | `soc-wazuh-server` |
| Guest operating system | Ubuntu Server 24.04.5 LTS, AMD64 |
| Assigned memory | 4096 MB |
| Assigned processors | 4 virtual CPUs |
| Virtual disk | 50 GB, dynamically allocated |
| Installation network | NAT with DHCP |
| Lab network | Adapter 2: host-only, using VirtualBox Host-Only Ethernet Adapter #2 |
| Windows host-only address | `192.168.149.1/24` |
| Ubuntu host-only address | `192.168.149.10/24` on `enp0s8` |
| Storage layout | ext4 mounted at `/`, without LVM |
| Lab administrator account | `socadmin` |

## Installation steps

1. Download the Ubuntu Server install image ISO from the [official release page](https://releases.ubuntu.com/24.04/).
2. In VirtualBox, create `soc-wazuh-server` with the settings above and attach the ISO. Disable unattended installation so the Ubuntu installer can be configured manually.
3. Start the VM and choose **Try or Install Ubuntu Server**. Select the appropriate language and keyboard layout, and choose the standard **Ubuntu Server** installation.
4. Accept DHCP networking. Leave the proxy blank and use the default Ubuntu archive mirror after its connection test succeeds.
5. Select the 50 GB **VBOX_HARDDISK** under **Use an entire disk**. Disable **Set up this disk as an LVM group**, review the storage summary, and confirm installation to this virtual disk.
6. Set the hostname to `soc-wazuh-server`, create the `socadmin` account with a private password, and skip Ubuntu Pro. Select OpenSSH server with password authentication for initial lab administration, and leave featured server snaps unselected.
7. Wait for installation and updates to finish. Reboot, remove the installation ISO from the virtual optical drive if prompted, and log in.

Installation references: [VirtualBox VM creation](https://docs.oracle.com/en/virtualization/virtualbox/7.2/user/create-vm.html) and [Ubuntu installer guide](https://canonical-subiquity.readthedocs-hosted.com/en/latest/tutorial/screen-by-screen.html).

## Verification

After logging in, I ran these commands to verify the hostname and Ubuntu version and check the available memory and root filesystem space before installing Wazuh:

```bash
hostname
lsb_release -d
free -h
df -h /
```

My verification results were:

| Check | Observed result |
|---|---|
| Hostname | `soc-wazuh-server` |
| Ubuntu version | Ubuntu 24.04.5 LTS |
| Memory visible to Ubuntu | 3.8 GiB total, 3.4 GiB available |
| Swap | 3.8 GiB total, unused at verification |
| Root filesystem | `/dev/sda2`, 49G total, 40G available, 15% used |

Memory usage and free disk space will change as software is installed and services run. 

The screenshot below records the commands I ran and their output:

<img width="662" height="213" alt="05-ubuntu-system-verification" src="https://github.com/user-attachments/assets/6993afb0-1738-4361-b2b3-253598d18555" />

## Lab networking and SSH access

I kept Adapter 1 on NAT for installation downloads and added Adapter 2 on the host-only network. This gives my laptop a direct connection to the Ubuntu VM for SSH.

I used VirtualBox Host-Only Ethernet Adapter #2 with DHCP disabled and assigned the following static addresses:

| Device | Interface | Address |
|---|---|---|
| Windows laptop | Ethernet 3: VirtualBox Host-Only Ethernet Adapter #2 | `192.168.149.1/24` |
| Ubuntu VM | `enp0s8`: Adapter 2, host-only | `192.168.149.10/24` |

I configured the Ubuntu address in `/etc/netplan/99-soc-lab.yaml`:

```yaml
network:
  version: 2
  ethernets:
    enp0s8:
      dhcp4: false
      addresses:
        - 192.168.149.10/24
```

I set the file permissions with `sudo chmod 600 /etc/netplan/99-soc-lab.yaml`, applied the configuration with `sudo netplan try`. The host-only interface has no default gateway or DNS server; the NAT interface retains its download connection. I checked `ip -br address` and verified that `enp0s8` was UP with the assigned address.

### Troubleshooting the initial timeout

The first SSH connection attempt timed out. I compared the VM's selected host-only network with Windows' actual addresses using `ipconfig /all`. The Windows' adapter names differed from VirtualBox's numbering: Ethernet 2 was the first host-only adapter, while Ethernet 3 was host-only adapter #2. The `192.168.149.1` address had been assigned to Ethernet 2 while the VM was connected to adapter #2.

I restored Ethernet 2 to `192.168.56.1/24` and assigned Ethernet 3 `192.168.149.1/24`. After correcting the Windows addresses, the SSH connection succeeded.

### Connection verification

On Ubuntu, `ssh.socket` was active. Ubuntu 24.04 uses socket activation, so an inactive `ssh` service does not by itself mean SSH is unavailable.

From my Windows laptop's PowerShell, I ran:

```powershell
ssh socadmin@192.168.149.10
```

I accepted the host key and authenticated with my private Ubuntu password. The screenshot shows the Ubuntu welcome message and the remote `socadmin@soc-wazuh-server` shell prompt, confirming a successful authenticated SSH session.

<img width="654" height="682" alt="07-ubuntu-lab-network" src="https://github.com/user-attachments/assets/10b922d8-0ef9-498b-be71-e9e93fd98379" />

Networking references: [VirtualBox host-only networking](https://docs.oracle.com/en/virtualization/virtualbox/7.2/user/networkingdetails.html), [Netplan static addresses](https://netplan.readthedocs.io/en/stable/examples/#how-to-configure-a-static-ip-address), and [Ubuntu SSH socket activation](https://discourse.ubuntu.com/t/sshd-now-uses-socket-based-activation-ubuntu-22-10-and-later/30189/1).

## Remaining setup

The host-only network and SSH access are now verified. Wazuh deployment and the Windows endpoint setup are still pending. Before running security scenarios, disconnect the NAT adapter's virtual cable while keeping the host-only network connected.
