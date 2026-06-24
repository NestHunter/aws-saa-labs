# Amazon EBS Volume Lab

This lab demonstrates the core lifecycle of an Amazon EBS volume: creating it, attaching it to an EC2 instance, formatting and mounting it on Linux, persisting the mount across reboots, capturing a snapshot, and using that snapshot to transfer data to a second instance in a different Availability Zone.

The detailed step-by-step walkthrough is in [amazon-ebs-volumes.md](./amazon-ebs-volumes.md).

---

## Prerequisites

- An AWS account with permissions to create EC2 instances, EBS volumes, and snapshots
- Basic familiarity with the AWS Management Console
- Basic Linux command-line skills (the lab uses `lsblk`, `mkfs`, `mount`, and `fstab`)
- Two EC2 instances running Amazon Linux 2 (or a similar Linux AMI), each in a **different Availability Zone** within the same region
- SSH access to both instances

---

## Lab Steps (High Level)

### Part 1 — Create and attach the volume

1. Launch two EC2 instances in the same region, placing each one in a **different Availability Zone** (for example, `us-east-1a` and `us-east-1b`).
2. Create a **10 GB gp2 EBS volume** in the same AZ as the first instance.
3. Attach the volume to the first instance.

### Part 2 — Format, mount, and write data

4. SSH into the first instance and list block devices to confirm the volume is attached:
   ```bash
   lsblk
   ```
5. Format the volume as **ext4**:
   ```bash
   sudo mkfs -t ext4 /dev/xvdf
   ```
6. Create a mount point and mount the volume:
   ```bash
   sudo mkdir /data
   sudo mount /dev/xvdf /data
   ```
7. Make the mount **persistent across reboots** by adding an entry to `/etc/fstab`.
8. Create test files and directories under `/data` to use as verification data.

### Part 3 — Snapshot and cross-AZ restore

9. Take a **snapshot** of the volume from the EC2 console or CLI.
10. Once the snapshot is complete, create a **new EBS volume** from it — this time in the **second AZ**.
11. Attach the new volume to the second instance.
12. SSH into the second instance, mount the volume, and verify that the test files are present.

---

## What This Lab Demonstrates

| Concept | Why It Matters |
|---|---|
| EBS volumes are AZ-scoped | A volume can only be directly attached to an instance in the same AZ — you cannot attach an `us-east-1a` volume to an instance in `us-east-1b` |
| Formatting and mounting on Linux | EBS volumes arrive as raw block devices; they must be formatted before use |
| Persistent mounts with `/etc/fstab` | Without an `fstab` entry, a mounted volume will not remount after a reboot |
| Snapshots are region-scoped | A snapshot can be used to create a new volume in any AZ within the same region, which is the standard way to move EBS data across AZs |

---

## Related Lab Notes

- [Enabling Outbound Internet Access from a Private Subnet with a NAT Gateway](../labs/private-subnet-nat-gateway.md)
- [Network Evaluation Challenge — VPC Troubleshooting Skills Assessment](../labs/network-evaluation-challenge.md)
