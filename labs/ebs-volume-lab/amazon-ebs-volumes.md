# Amazon EBS Volumes — Step-by-Step Lab Walkthrough

This document contains the detailed steps for the EBS Volume Lab. For a high-level overview, see [README.md](./README.md).

---

## What Is Amazon EBS?

Amazon Elastic Block Store (EBS) provides persistent block storage for use with EC2 instances. Unlike instance store volumes (which are temporary and lost when the instance stops), EBS volumes persist independently of the instance lifecycle — they can be stopped, detached, snapshotted, and reattached.

**Key characteristics:**

- EBS volumes are **AZ-scoped**: a volume exists in one Availability Zone and can only be attached to an instance in that same AZ
- Volumes are **network-attached**: they connect to your EC2 instance over the AWS network, not a physical cable
- **Snapshots** are stored in Amazon S3 (managed by AWS) and are **region-scoped**: a snapshot can be used to create a new volume in any AZ within the same region
- EBS volumes must be **formatted** before use — they arrive as raw block devices

---

## Lab Environment

| Resource | Details |
|---|---|
| Instance 1 | EC2 in AZ 1 (e.g., `us-east-1a`) |
| Instance 2 | EC2 in AZ 2 (e.g., `us-east-1b`) |
| Volume A | 10 GB gp2, created in AZ 1 |
| Volume B | Created from snapshot of Volume A, in AZ 2 |

---

## Part 1 — Launch Instances and Create the Volume

### Step 1: Launch two EC2 instances in different AZs

1. Open the **EC2 Console** and choose **Launch Instance**.
2. Select an Amazon Linux 2 AMI (or any compatible Linux AMI).
3. Choose an instance type (t2.micro or t3.micro is sufficient for this lab).
4. Under **Network settings**, confirm the subnet corresponds to **AZ 1** (e.g., `us-east-1a`).
5. Ensure **Auto-assign Public IP** is enabled so you can SSH in.
6. Create or select a key pair.
7. Launch the instance.
8. Repeat the process for the **second instance**, selecting a subnet in **AZ 2** (e.g., `us-east-1b`).

### Step 2: Create a 10 GB gp2 EBS volume in AZ 1

1. In the EC2 Console, navigate to **Elastic Block Store → Volumes**.
2. Choose **Create volume**.
3. Set the following:
   - **Volume type:** gp2
   - **Size:** 10 GiB
   - **Availability Zone:** same AZ as Instance 1 (e.g., `us-east-1a`)
4. Choose **Create volume**.

### Step 3: Attach the volume to Instance 1

1. Select the newly created volume.
2. Choose **Actions → Attach volume**.
3. Select Instance 1 from the instance list.
4. Note the device name (typically `/dev/xvdf` or `/dev/sdf` — the console suggests one).
5. Choose **Attach volume**.

---

## Part 2 — Format, Mount, and Persist the Volume

SSH into Instance 1 to complete the following steps.

### Step 4: Verify the volume is attached

```bash
lsblk
```

You should see a new block device listed (for example, `xvdf` with 10G), separate from the root volume.

### Step 5: Check whether the volume already has a filesystem

```bash
sudo file -s /dev/xvdf
```

If the output is `/dev/xvdf: data`, the volume is empty and has no filesystem — this is expected for a brand-new volume.

### Step 6: Format the volume as ext4

```bash
sudo mkfs -t ext4 /dev/xvdf
```

This writes an ext4 filesystem to the block device. Only do this on a new or intentionally wiped volume — formatting will erase any existing data.

### Step 7: Create the mount point and mount the volume

```bash
sudo mkdir /data
sudo mount /dev/xvdf /data
```

Verify the mount:

```bash
df -h
```

`/dev/xvdf` should now appear in the output, mounted at `/data`.

### Step 8: Make the mount persistent with /etc/fstab

Without an `fstab` entry, the mount will not survive a reboot.

First, retrieve the volume's UUID:

```bash
sudo blkid /dev/xvdf
```

Copy the UUID value from the output, then open `/etc/fstab`:

```bash
sudo nano /etc/fstab
```

Add the following line at the end of the file (replace `YOUR-UUID-HERE` with the actual UUID):

```
UUID=YOUR-UUID-HERE  /data  ext4  defaults,nofail  0  2
```

The `nofail` option prevents the instance from failing to boot if the volume is not attached. Save and close the file.

Test the `fstab` entry without rebooting:

```bash
sudo mount -a
```

If there are no errors, the configuration is correct.

### Step 9: Create test data under /data

```bash
sudo mkdir /data/project-files
sudo touch /data/project-files/notes.txt
echo "EBS lab test data" | sudo tee /data/project-files/notes.txt
```

You can create additional files or directories to make the verification step in Part 3 more meaningful.

---

## Part 3 — Snapshot the Volume and Restore It in AZ 2

### Step 10: Take a snapshot of Volume A

1. In the EC2 Console, navigate to **Elastic Block Store → Volumes**.
2. Select Volume A (the one attached to Instance 1).
3. Choose **Actions → Create snapshot**.
4. Add a description (e.g., `ebs-lab-snapshot`) and choose **Create snapshot**.
5. Navigate to **Elastic Block Store → Snapshots** and wait for the snapshot status to show **completed**.

> Snapshots are stored incrementally in Amazon S3 (managed by AWS). The first snapshot captures all data on the volume; subsequent snapshots capture only the changes.

### Step 11: Create Volume B from the snapshot in AZ 2

1. Select the completed snapshot.
2. Choose **Actions → Create volume from snapshot**.
3. Set the following:
   - **Volume type:** gp2
   - **Size:** 10 GiB (or larger — you can increase size from a snapshot, but not decrease it)
   - **Availability Zone:** AZ 2 (e.g., `us-east-1b`) — the AZ where Instance 2 is running
4. Choose **Create volume**.

### Step 12: Attach Volume B to Instance 2

1. Select Volume B.
2. Choose **Actions → Attach volume**.
3. Select Instance 2 and note the device name.
4. Choose **Attach volume**.

### Step 13: Mount Volume B on Instance 2 and verify the data

SSH into Instance 2.

Verify the volume is visible:

```bash
lsblk
```

Mount it:

```bash
sudo mkdir /data
sudo mount /dev/xvdf /data
```

Check that the test data is present:

```bash
ls /data/project-files/
cat /data/project-files/notes.txt
```

You should see the files created in Step 9. This confirms that the snapshot captured the data correctly and the restore worked across Availability Zones.

---

## Key Concepts Reinforced

### EBS volumes are AZ-scoped
Volume A (created in AZ 1) could not be directly attached to Instance 2 (in AZ 2). The snapshot-and-restore workflow is the standard way to move EBS data across AZs.

### Snapshots are region-scoped
A snapshot can be used to create a new volume in any AZ within the same region. It can also be copied to a different region if cross-region data movement is needed.

### Formatting is required before first use
A new EBS volume has no filesystem. Running `mkfs` before mounting is a required step. Skipping it means the volume cannot be mounted.

### fstab ensures persistence
Mounting a volume manually does not survive a reboot. Adding a UUID-based entry to `/etc/fstab` with `nofail` ensures the volume remounts automatically and that a missing volume does not cause boot failures.

### Device name vs. UUID
Using the UUID in `fstab` (rather than the device name like `/dev/xvdf`) is more reliable because device names can change between reboots or instance types, while UUIDs remain constant for the lifetime of the volume.

---

## Cleanup

To avoid ongoing charges after completing the lab:

1. Unmount and detach both volumes from their instances.
2. Delete Volume A and Volume B from **Elastic Block Store → Volumes**.
3. Delete the snapshot from **Elastic Block Store → Snapshots**.
4. Terminate Instance 1 and Instance 2 if they are no longer needed.

---

## Relevance to AWS SAA Exam

| SAA Topic | How It Appeared in This Lab |
|---|---|
| EBS volume types | gp2 used here; exam also covers gp3, io1/io2, st1, sc1 |
| EBS AZ scope | Volumes are AZ-locked; snapshots unlock cross-AZ movement |
| EBS snapshots | Incremental, region-scoped, used to create volumes in any AZ |
| Linux volume management | `lsblk`, `mkfs`, `mount`, `/etc/fstab` — common in exam scenarios |
| Persistent storage vs. instance store | EBS persists independently; instance store is ephemeral |
| Data migration between AZs | Snapshot → create new volume in target AZ → attach and mount |
