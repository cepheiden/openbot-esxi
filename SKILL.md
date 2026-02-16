---
name: esxi-debian-deploy
description: Zero-touch Debian 13 VM deployment on VMware ESXi 8. Builds custom preseed ISO, creates NVMe+vmxnet3 VM with serial console, and runs unattended installation. Use when deploying Debian VMs on ESXi, automating VM provisioning, or setting up serial console access for headless ESXi VM management.
---

# ESXi Debian 13 Zero-Touch Deploy

Deploy fully configured Debian 13 VMs on ESXi 8 in ~8 minutes with zero manual interaction.

## Requirements

- **ESXi 8.x** host with SSH and datastore access
- **govc** CLI ([github.com/vmware/govmomi](https://github.com/vmware/govmomi))
- **xorriso**, **isolinux** — for custom ISO build
- **sshpass** — for automated SSH/SCP
- Tools on agent host: `bash`, `python3`, `wget`

Install on Debian/Ubuntu:
```bash
apt install xorriso isolinux sshpass
# govc: https://github.com/vmware/govmomi/releases
```

## Usage

```bash
bash scripts/esxi-deploy.sh [hostname] [cpu] [ram_mb] [disk_gb] [serial_port]
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| hostname | random animal name | VM name |
| cpu | 2 | vCPU count |
| ram_mb | 2048 | Memory in MB |
| disk_gb | 20 | Disk size in GB |
| serial_port | random 8600-8699 | Telnet port for serial console |

**Example:**
```bash
bash scripts/esxi-deploy.sh webserver 4 4096 50 8610
```

## What It Does

1. **Generate preseed.cfg** — German locale, DHCP, configurable user + `root`, random password
2. **Build custom ISO** — Debian netinst + preseed, patched isolinux for auto-boot
3. **Upload ISO** to ESXi datastore
4. **Create VM** — NVMe disk (thin provisioned), dual NIC (E1000 for installer + vmxnet3 for production), serial port via telnet
5. **Boot + unattended install** — preseed handles everything
6. **Post-install** — Remove E1000, eject ISO, set boot to HDD
7. **Output credentials** — SSH + serial console access details

## Serial Console

Every VM gets a serial port accessible via telnet to the ESXi host:

```bash
telnet <ESXI_IP> <serial_port>
```

Works even when the VM has no network. Configured:
- GRUB: `GRUB_TERMINAL="console serial"`, serial 115200 8N1
- Kernel: `console=tty0 console=ttyS0,115200n8`
- Getty: `serial-getty@ttyS0.service` enabled

**ESXi firewall requirement** (activated automatically by the script):
```bash
esxcli network firewall ruleset set -e true -r remoteSerialPort
```

**Important:** Set serial port IP to the ESXi host IP, not `0.0.0.0`:
```
serial0.fileName = "telnet://<ESXI_IP>:<port>"
```

## Online Disk Resize

Grow a VM's disk without shutdown:

```bash
bash scripts/esxi-vm-resize-disk.sh <vm-name> <new-size-gb>
```

Requires `cloud-guest-utils` on the VM (pre-installed by the deploy script).

## Configuration

Edit variables at the top of `scripts/esxi-deploy.sh`:

```bash
ESXI_HOST="192.168.1.100"    # ESXi host IP
ESXI_USER="root"               # ESXi user
ESXI_DATASTORE="datastore1"    # Target datastore
NETWORK="VM Network"           # Port group name
DOMAIN="local"          # Domain for VMs
```

Credentials are resolved via Vaultwarden (`scripts/vw_ref.py`). Adapt the `ESXI_PASS` resolution to your credential store, or hardcode for testing.

## VM Configuration Details

| Component | Choice | Reason |
|-----------|--------|--------|
| Disk controller | NVMe | Faster than SCSI/SATA for modern guests |
| Production NIC | vmxnet3 | Paravirtualized, best performance |
| Installer NIC | E1000 | Kernel driver built-in, no firmware needed |
| Boot mode | BIOS | Simpler for automated installs |
| Provisioning | Thin | Saves datastore space |

## Preseed Highlights

- Locale: `de_DE.UTF-8`, keyboard `de`, timezone `Europe/Berlin`
- Partitioning: automatic, single root + swap
- Packages: `open-vm-tools`, `curl`, `sudo`, `qemu-guest-agent`, `cloud-guest-utils`
- SSH: `PermitRootLogin yes`, `PasswordAuthentication yes`
- Blacklisted modules: `floppy`, `pcspkr` (prevent I/O error loops in VMs)

Customize the preseed section in `esxi-deploy.sh` for different locales or packages.

## Gotchas

- **No heredoc in preseed `late_command`** — Shell expansion in the deploy script's heredoc destroys nested heredocs. Use `echo -e` or single-line commands instead.
- **Serial console only works after install** — The Debian installer uses VGA; serial output starts at first boot (GRUB + kernel).
- **ESXi firewall blocks serial by default** — The `remoteSerialPort` ruleset must be enabled.
- **Don't resize MBR partitions live** with extended/swap layout — Use `growpart` on the root partition or redeploy with larger disk.
- **E1000 removal requires shutdown** — The script handles this automatically post-install.

## References

- [references/preseed-template.cfg](references/preseed-template.cfg) — Full preseed config template
- [references/vmx-template.md](references/vmx-template.md) — VMX configuration reference
