# EC2 AMI Creation & Cross-Region Copying Lab

**Platform:** AWS  
**Services Used:** EC2, AMI (Amazon Machine Image)  
**Region:** US East (N. Virginia) → US East 2 (Ohio)  
**Course:** Adrian Cantrill's AWS Solutions Architect Associate

---

## Overview

This lab demonstrates how to create a custom Amazon Machine Image (AMI) from a configured EC2 instance and copy it to another AWS region. A WordPress server is used as the base instance, with a custom MOTD (Message of the Day) added via `cowsay` to confirm the AMI captures the full system state.

---

## Steps

### Step 1 — Launch an EC2 Instance with WordPress

- Launched an EC2 instance (Amazon Linux 2023)
- Installed WordPress on the instance

---

### Step 2 — Install Cowsay & Configure Custom MOTD

Added a `cowsay` greeting to the login banner to visually confirm the AMI is working after launch.

```bash
# Install cowsay
sudo dnf install -y cowsay

# Create a custom MOTD script
sudo nano /etc/update-motd.d/40-cow
```

Contents of `/etc/update-motd.d/40-cow`:

```bash
#!/bin/sh
cowsay "Amazon Linux 2023 AMI - Animals4Life"
```

```bash
# Make the script executable and apply it
sudo chmod 755 /etc/update-motd.d/40-cow
sudo update-motd
sudo reboot
```

---

### Step 3 — Create AMI & Launch Instance From It

- Created an AMI image from the configured WordPress EC2 instance
- Launched a new instance from the AMI (`InstanceFromAMI`)
- Connected to the new instance — the custom MOTD confirmed the AMI captured the full config:

```
 ______________________________________
< Amazon Linux 2023 AMI - Animals4Life >
 --------------------------------------
        \   ^__^
         \  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||
   ,     #_
   ~\_  ####_        Amazon Linux 2023
  ~~  \_#####\
  ~~     \###|
  ~~       \#/ ___   https://aws.amazon.com/linux/amazon-linux-2023
   ~~       V~' '->
    ~~~         /
      ~~._.   _/
         _/ _/
       _/m/'
```

- Tested the public IP in a browser → WordPress loaded successfully ✅

---

### Step 4 — Unblock Public Sharing for AMIs & Snapshots

> ⚠️ **Note:** This step was not present in Adrian Cantrill's course and appears to be a newer AWS default. If you find you can't edit AMI permissions, this is likely why.

By default, AWS now blocks public sharing of AMIs and EBS snapshots at the account level. To modify AMI access permissions, this block must be disabled first.

- Navigated to the **EC2 Dashboard**
- Under the **Data Protection and Security** tab, located two settings:
  - **Block public access for AMIs**
  - **Block public access for EBS snapshots**
- Clicked **Manage** on each
- Unchecked **"Block new public sharing"** on both
- This allowed AMI permission editing to proceed normally

---

### Step 5 — Copy AMI to Another Region

- Navigated to **EC2 → AMIs** in the AWS Console
- Selected the AMI and chose **Copy AMI** from the Actions dropdown
- Copied the AMI to **US East 2 (Ohio)**
- Waited for both the AMI and its associated snapshot to finish copying
- Verified the copy completed successfully in the Ohio region

---

### Step 6 — Cleanup

- Deregistered the AMI in both **N. Virginia** and **Ohio**
- Deleted the associated EBS snapshots in both regions

---

## Key Concepts

| Concept | Notes |
|---|---|
| **AMI** | A snapshot of an EC2 instance used to launch identical copies |
| **MOTD** | Customizable login banner; useful for confirming instance identity |
| **Cross-Region Copy** | AMIs can be shared across regions; snapshots are copied automatically |
| **Block Public Access** | AWS now blocks public AMI/snapshot sharing by default at the account level — must be disabled via EC2 Dashboard → Data Protection and Security before editing AMI permissions |
| **Cleanup** | Both the AMI *and* its snapshot must be deleted separately to avoid storage costs |

---

## Notes

- AMI creation captures the full disk state — installed software, configs, and custom scripts are all preserved
- Cross-region AMI copies are useful for disaster recovery and multi-region deployments
- Always delete snapshots alongside AMIs during cleanup — they are billed independently
