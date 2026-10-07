# Lab setup: Ubuntu server and Windows endpoint

I built this Ubuntu Server VM as the foundation for my security monitoring lab. My Windows laptop has a limited 8 GB RAM, so I allocated 4 GB RAM to the VM. I am evaluating Wazuh's performance with this allocation and a Windows test VM before running security scenarios.

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

After logging in, I verified the hostname and Ubuntu version and checked the available memory and root filesystem space before installing Wazuh.

The screenshot below records the verification results:

<img width="662" height="213" alt="05-ubuntu-system-verification" src="https://github.com/user-attachments/assets/6993afb0-1738-4361-b2b3-253598d18555" />

## Lab networking and SSH access

I kept Adapter 1 on NAT for installation downloads and added Adapter 2 on the host-only network for SSH and lab communication.

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

I set the file permissions with `sudo chmod 600 /etc/netplan/99-soc-lab.yaml` and applied the configuration with `sudo netplan try`. The host-only interface has no default gateway or DNS server; the NAT interface retains its download connection. I verified that `enp0s8` was UP with the assigned address.

### Troubleshooting the initial timeout

The first SSH attempt timed out because the Windows host-only address was assigned to the wrong adapter. I restored Ethernet 2 to `192.168.56.1/24` and assigned Ethernet 3 `192.168.149.1/24`, matching the VM's selected host-only network. The SSH connection then succeeded.

### Connection verification

On Ubuntu, `ssh.socket` was active. From my Windows laptop's PowerShell, I connected with:

```powershell
ssh socadmin@192.168.149.10
```

I accepted the host key and authenticated with my private Ubuntu password. The screenshot shows the Ubuntu welcome message and the remote `socadmin@soc-wazuh-server` shell prompt, confirming a successful authenticated SSH session.

<img width="654" height="682" alt="07-ubuntu-lab-network" src="https://github.com/user-attachments/assets/10b922d8-0ef9-498b-be71-e9e93fd98379" />

Networking references: [VirtualBox host-only networking](https://docs.oracle.com/en/virtualization/virtualbox/7.2/user/networkingdetails.html), [Netplan static addresses](https://netplan.readthedocs.io/en/stable/examples/#how-to-configure-a-static-ip-address), and [Ubuntu SSH socket activation](https://discourse.ubuntu.com/t/sshd-now-uses-socket-based-activation-ubuntu-22-10-and-later/30189/1).

## Wazuh installation attempt

I ran `sudo bash ./wazuh-install.sh -a` inside Ubuntu for the combined manager, indexer, and dashboard deployment. The assistant reported version 4.14.8. The first attempt failed with `ERROR: Wazuh-manager could not be started.` The installer removed the components and reported `Installation cleaned.`

My VM's 4 GB allocation is below the 8 GiB recommendation for a combined deployment in the [Wazuh quickstart](https://documentation.wazuh.com/current/quickstart.html#requirements).

### Diagnostic findings

The service journal showed a successful initial start followed by a restart that exceeded the 45-second startup limit and reported a leftover Python process. The kernel journal search returned no out-of-memory kill messages. The [Wazuh 4.14.8 service template](https://github.com/wazuh/wazuh/blob/v4.14.8/src/init/templates/wazuh-manager.service) sets `TimeoutSec=45` so I retried by changing the timeout to 180 seconds which ended up working.

<img width="934" height="474" alt="08-wazuh-service-verification" src="https://github.com/user-attachments/assets/986c1f5a-2bfb-491c-a687-ac59c2d32f72" />

## Windows endpoint networking

I configured `soc-win-test` with 2048 MB RAM and two network adapters. Adapter 1 uses NAT for internet access. Adapter 2 uses `VirtualBox Host-Only Ethernet Adapter #2`, the same host-only network as Ubuntu.

Inside the Windows VM, I identified the network cards with:

```powershell
Get-NetAdapter | Select-Object Name, MacAddress
```

I matched the output against the cards configured in VirtualBox:

| Windows VM interface | MAC address | VirtualBox connection | Intended IP configuration |
|---|---|---|---|
| Ethernet | `08-00-27-EE-34-D8` | Adapter 1: NAT | Automatic IP address and DNS |
| Ethernet 2 | `08-00-27-E6-9B-B7` | Adapter 2: host-only #2 | Static `192.168.149.20/24`; gateway and DNS left blank |

### Windows version and IP verification

This screenshot records the full hostname, Windows version, and corrected IP configuration:

<img width="883" height="490" alt="010-windows-endpoint-verification" src="https://github.com/user-attachments/assets/a4441485-5d94-4f3a-bd3d-93ec945e88b2" />

These checks verify the endpoint's name, OS, and IP addresses. They do not yet establish that it can reach the Wazuh manager or that an agent is enrolled.

## Remaining setup

I need to capture the successful Wazuh deployment checks and test the Windows endpoint's connection to the manager before installing and verifying the Windows agent and Sysmon. Before running security scenarios, I need to disconnect the NAT adapters' virtual cables while keeping the host-only network connected.
