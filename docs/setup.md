# Lab setup: Ubuntu server

I built this Ubuntu Server VM as the foundation for my security monitoring lab. My Windows laptop has a limited 8 GB RAM, so I allocated 4 GB RAM to the VM. I will evaluate Wazuh's performance with this allocation of resources before adding another endpoint.

## Current status

I installed Ubuntu Server and verified its version, hostname, available memory, and root filesystem space. 

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

After logging in, I ran these commands to confirm the server was working properly and checked the resources were available before installing Wazuh:
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

<img width="662" height="213" alt="05-ubuntu-system-verification" src="https://github.com/user-attachments/assets/87573ce7-3376-4bb0-9902-627cedd1dd3a" />

## Remaining setup

The VM's 4 GB allocation is below the [Wazuh quickstart recommendation](https://documentation.wazuh.com/current/quickstart.html) of 8 GiB for the combined central components. During Wazuh deployment, resource use will need to be checked consistently. NAT is currently being used for installation downloads, a proper network will have to be configured before running security tests and verifying SSH.
