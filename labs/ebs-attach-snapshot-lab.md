# EBS Attach, Detach, Snapshot & Cross-AZ Restore Lab

## Overview
In this lab I practiced attaching an EBS volume to an EC2 instance, inspecting and creating a filesystem, detaching and reattaching it to a second EC2, creating a snapshot, and restoring that snapshot as a new volume in a different Availability Zone.

**Services used:** EC2, EBS  
**Region:** us-east-1 (example)  
**Date completed:** April 2026

---

## Concepts Demonstrated
- EBS volumes persist data across instance stops and reboots
- EBS volumes are AZ-locked — they can only be attached to EC2 instances in the same AZ
- Snapshots are regional — they can be used to create volumes in any AZ within the same region
- Nitro-based EC2 instances present EBS volumes as NVMe devices (`nvme1n1`) rather than `xvdf`
- Device names like `/dev/sdf` become symbolic links to the actual NVMe device on Nitro instances

---

## Architecture
```
AZ us-east-1a                          AZ us-east-1b
┌─────────────────┐                   ┌─────────────────┐
│   EC2 Instance 1│                   │   EC2 Instance 3│
│                 │                   │                 │
│  ┌───────────┐  │    Snapshot →     │  ┌───────────┐  │
│  │ EBS Vol 1 │  │   New Volume      │  │ EBS Vol 2 │  │
│  │ (XFS)     │  │ ─────────────►    │  │ (XFS)     │  │
│  └───────────┘  │                   │  └───────────┘  │
└─────────────────┘                   └─────────────────┘
        │
        │ Detach & reattach
        ▼
┌─────────────────┐
│   EC2 Instance 2│
│  (same AZ)      │
│  ┌───────────┐  │
│  │ EBS Vol 1 │  │
│  └───────────┘  │
└─────────────────┘
```

---

## Steps

### 1. Attach EBS Volume to First EC2
- Created a new EBS volume in the AWS Console
- Attached it to EC2 Instance 1, specifying `/dev/sdf` as the device name
- On a Nitro-based instance, this appeared as `/dev/nvme1n1` (not `/dev/xvdf` as shown in older tutorials)

### 2. Inspect the Volume
```bash
sudo file -s /dev/nvme1n1
```
Output confirmed the volume already had an **XFS filesystem** from a previous use:
```
/dev/nvme1n1: SGI XFS filesystem data (blksz 4096, inosz 512, v2 dirs)
```

> **Note:** Running `sudo file -s /dev/sdf` returned `symbolic link to nvme1n1`, confirming that AWS automatically creates a symlink from the specified device name to the actual NVMe device on Nitro instances.

### 3. Create Filesystem (on fresh volume)
Since this was a pre-existing filesystem, formatting was not needed. On a blank volume you would run:
```bash
sudo mkfs -t xfs /dev/nvme1n1
```

### 4. Mount the Volume
```bash
# Create mount point at root level (important — not inside home directory)
sudo mkdir /ebstest

# Mount the volume
sudo mount /dev/nvme1n1 /ebstest

# Verify
df -h
```

> **Gotcha:** Initially created the directory inside the home folder (`~/ebstest`) instead of at root (`/ebstest`). The mount command requires the full absolute path from root.

### 5. Make Mount Persistent Across Reboots
Find the volume UUID (more reliable than device name on Nitro instances):
```bash
sudo blkid /dev/nvme1n1
```

Add an entry to `/etc/fstab`:
```
UUID=**blkid**  /ebstest  xfs  defaults,nofail  0  2
```

### 6. Detach and Reattach to Second EC2
- Unmounted the volume and detached it from EC2 Instance 1 in the AWS Console
- Attached it to EC2 Instance 2 (same AZ)
- Ran `sudo file -s /dev/nvme1n1` on the new instance — XFS filesystem was still intact, confirming EBS data persists across detach/reattach

### 7. Create a Snapshot
- In the AWS Console, navigated to **Volumes** and created a snapshot of the volume
- Snapshots are stored in S3 behind the scenes and are **regional** (not AZ-specific)

### 8. Restore Snapshot in a Different AZ
- Created a new volume from the snapshot, selecting a **different AZ** (e.g. us-east-1b)
- Attached the new volume to EC2 Instance 3 running in that AZ
- Mounted it and verified the filesystem and data were intact

### 9. Cleanup
- Unmounted volumes
- Detached and deleted EBS volumes
- Deleted the snapshot
- Terminated EC2 instances
- Deleted VPC and subnets in which these instances were deployed.

---

## Key Takeaways

**EBS vs Instance Store**
| | EBS | Instance Store |
|---|---|---|
| Persists on stop | ✅ Yes | ❌ No |
| Persists on reboot | ✅ Yes | ✅ Yes |
| Persists on terminate | ❌ No (by default) | ❌ No |
| Can detach/reattach | ✅ Yes | ❌ No |
| Cross-AZ migration | Via snapshot | Not possible |

**Nitro vs Xen Device Naming**
| Specified | Xen instance | Nitro instance |
|---|---|---|
| `/dev/sdf` | `/dev/xvdf` | `/dev/nvme1n1` |
| `/dev/sdg` | `/dev/xvdg` | `/dev/nvme2n1` |

---

## Issues Encountered
- **Device naming confusion:** Following along with a tutorial using a Xen-based instance (`xvdf`) while running a Nitro instance (`nvme1n1`). Solution: always run `lsblk` to confirm the actual device name.
- **Mount point location:** Created `/ebstest` inside the home directory instead of at root. Solution: always prefix directory paths with `/` when creating mount points.

---

## References
- [AWS EBS Documentation](https://docs.aws.amazon.com/ebs/)
- [Adrian Cantrill AWS SAA Course](https://learn.cantrill.io)
