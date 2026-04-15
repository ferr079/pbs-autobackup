# pbs-autobackup

Automated Proxmox Backup Server orchestration with Wake-on-LAN. Designed for homelabs where the backup server runs on a dedicated machine that stays powered off until needed.

## What it does

```
WOL → boot PBS host → enable storage → vzdump all CTs/VMs → wait → prune → GC → disable storage → shutdown
```

The entire cycle runs unattended. Your backup server boots, backs up everything, cleans up, and shuts down. Zero manual intervention, zero idle power draw.

## Why

Most Proxmox homelab setups either:
- Leave the backup server running 24/7 (wasting power)
- Run manual backups (forgetting for weeks)

This script solves both problems. Schedule it as a weekly cron job and forget about it.

## Requirements

- Proxmox VE cluster with PBS configured as storage (can be disabled by default)
- PBS host with Wake-on-LAN enabled in BIOS
- SSH key access from the orchestrator to PBS host (for prune/GC/shutdown)
- `wakeonlan` package installed on the orchestrator
- PVE API tokens with `PVEAuditor` + `Datastore.Allocate` permissions
- `python3`, `curl`, `openssl` (standard on Debian)

## Quick start

```bash
# Install
sudo cp pbs-backup /usr/local/bin/
sudo chmod +x /usr/local/bin/pbs-backup

# Configure
sudo mkdir -p /etc/pbs-backup
sudo cp pbs-backup.env.example /etc/pbs-backup/pbs-backup.env
sudo chmod 600 /etc/pbs-backup/pbs-backup.env
# Edit with your IPs, MACs, and API tokens

# Test
pbs-backup --dry-run

# Schedule (weekly Sunday 3am)
echo "0 3 * * 0 root /usr/local/bin/pbs-backup >> /var/log/pbs-backup.log 2>&1" | sudo tee /etc/cron.d/pbs-backup
```

## Configuration

Copy `pbs-backup.env.example` to `/etc/pbs-backup/pbs-backup.env`:

| Variable | Required | Description |
|----------|----------|-------------|
| `PBS_HOST_MAC` | Yes | MAC address of the PBS host (for WOL) |
| `PBS_HOST_IP` | Yes | IP of the PBS host (for boot check + shutdown) |
| `PBS_IP` | Yes | IP of PBS server (can differ if PBS runs in a CT) |
| `PBS_STORAGE` | Yes | PVE storage name pointing to PBS |
| `PVE_NODES` | Yes | Nodes to backup (see example) |
| `BACKUP_MODE` | No | `snapshot` (default), `suspend`, or `stop` |
| `COMPRESS` | No | `zstd` (default), `lzo`, `gzip`, or `none` |
| `MAX_BOOT_WAIT` | No | Boot timeout in seconds (default: 180) |
| `MAX_TASK_WAIT` | No | Backup timeout in seconds (default: 3600) |
| `PBS_PRUNE_JOB` | No | PBS prune job name (skipped if empty) |
| `PBS_DATASTORE` | No | PBS datastore for GC (skipped if empty) |

### Multi-node setup

```bash
# One line per node: name|ip|api_token
PVE_NODES="pve1|192.168.1.251|user@pam!backup=xxx-xxx
pve2|192.168.1.252|user@pam!backup=yyy-yyy
pve3|192.168.1.253|user@pam!backup=zzz-zzz"
```

## Telegram alerts (optional)

`guardian-backup` wraps `pbs-backup` and sends a Telegram notification after each run.

```bash
sudo cp guardian-backup /usr/local/bin/
sudo chmod +x /usr/local/bin/guardian-backup
sudo cp guardian.env.example /etc/pbs-backup/guardian.env
sudo chmod 600 /etc/pbs-backup/guardian.env
# Edit with your Telegram bot token and chat ID

# Use guardian-backup instead of pbs-backup in cron
echo "0 3 * * 0 root /usr/local/bin/guardian-backup" | sudo tee /etc/cron.d/pbs-backup
```

## Exit codes

| Code | Meaning |
|------|---------|
| 0 | All backups completed successfully |
| 1 | Partial failure (some backups failed, others OK) |
| 2 | Critical failure (PBS host unreachable or storage error) |

## How it works

1. **WOL** — Sends magic packet to wake the PBS host
2. **Boot wait** — Polls ping + PBS API until responsive (configurable timeout)
3. **Enable storage** — Re-enables the PBS storage on PVE via API
4. **Backup** — Starts `vzdump --all` on each node in parallel via PVE API
5. **Wait** — Polls task status every 30s until all complete or timeout
6. **Prune** — Runs configured prune job on PBS (optional)
7. **GC** — Runs garbage collection on PBS datastore (optional)
8. **Disable storage** — Disables PBS storage to stop polling when host is off
9. **Shutdown** — Gracefully shuts down the PBS host via SSH

The storage is disabled after backup so PVE doesn't log connection errors while the PBS host is powered off.

## License

MIT
