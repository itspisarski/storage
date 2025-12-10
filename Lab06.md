# Lab: AWS Snapshots & AMIs (EC2 & EBS Backup Strategies)

## 1. Overview
In this lab, you will learn the two critical ways to back up resources in AWS:
1.  **EBS Snapshots:** Point-in-time backups of specific data volumes.
2.  **AMIs (Amazon Machine Images):** Full instance backups that include the OS, boot configuration, and all attached volumes.

### **Lab Objectives**
* Create a dedicated data volume and populate it with "critical" files.
* Take a **Data-Consistent Snapshot** of an EBS volume.
* Create a **Full EC2 Backup (AMI)** of a running instance.
* **Restore** a volume from a snapshot.
* **Launch** a new instance from an AMI.

---

## 2. Prerequisites
* **AWS Account** with EC2 permissions.
* **EC2 Instance** (Amazon Linux 2023 or Ubuntu) running.
* **SSH Client** to connect to the instance.

---

## 3. Phase 1: Setup (Create Data)

We need a distinction between the "OS Drive" (Root) and a "Data Drive."

### Step 1: Create and Attach a Volume
1. Go to the **EC2 Console** -> **Volumes** -> **Create Volume**.
2. **Size:** 1 GB (Small is fine).
3. **Availability Zone:** Must match your running instance (e.g., `us-east-1a`).
4. Click **Create**.
5. Select the new volume -> **Actions** -> **Attach Volume**.
6. Select your instance and click **Attach**.

### Step 2: Format and Write Data
SSH into your instance and run the following commands to set up the disk.

```bash
# 1. Identify the new disk (usually the last one, e.g., nvme1n1 or xvdf)
lsblk

# 2. Format with XFS (Change /dev/nvme1n1 if your device name differs)
sudo mkfs -t xfs /dev/nvme1n1

# 3. Mount it
sudo mkdir /data
sudo mount /dev/nvme1n1 /data

# 4. Create a "Critical" file
sudo sh -c 'echo "Important Database Config v1" > /data/config.txt'
cat /data/config.txt
```

---

## 4. Phase 2: EBS Volume Snapshot (Data Backup)

This method is best for backing up secondary drives without stopping the server.

### Step 1: Freeze I/O (Best Practice)
To ensure data consistency (no half-written files), freeze the filesystem first.

```bash
sudo fsfreeze -f /data
```

### Step 2: Take the Snapshot (AWS Console)
1. Go to **EC2 Console** -> **Volumes**.
2. Select the 1 GB volume (Look for the "In-use" status).
3. **Actions** -> **Create Snapshot**.
4. **Description:** `Lab-Data-Snapshot-1`.
5. Click **Create Snapshot**.

### Step 3: Unfreeze I/O
**Go back to your SSH terminal immediately** to unfreeze the drive.

```bash
sudo fsfreeze -u /data
```

---

## 5. Phase 3: EC2 Instance Snapshot (AMI)

This method is best for "Golden Images" (saving server configurations) or full disaster recovery.

### Step 1: Create Image
1. Go to **EC2 Console** -> **Instances**.
2. Select your instance.
3. **Actions** -> **Image and templates** -> **Create Image**.
4. **Image name:** `Web-Server-Backup-v1`.
5. **No reboot:** *Uncheck* this (We WANT it to reboot to ensure the OS is in a clean state).
6. Click **Create Image**.

*Note: Your SSH connection will drop briefly as the instance reboots.*

---

## 6. Phase 4: The Restore Tests

Now we verify that our backups actually work.

### Test A: Restore Data from EBS Snapshot
**Scenario:** You accidentally deleted the file.

**1. Simulate Disaster**
Run this to delete your data:

```bash
sudo rm /data/config.txt
```

**2. Restore via Console**
* Go to **Snapshots** in the Console.
* Select `Lab-Data-Snapshot-1`.
* **Actions** -> **Create Volume from snapshot**.
* Select the **same AZ** as your instance.
* Click **Create Volume**.

**3. Attach & Verify**
* Go to **Volumes**, select the new volume, and **Attach** it to your instance.
* Run the following to verify the data is back:

```bash
# List disks (You will see a new one, e.g., nvme2n1)
lsblk

# Mount recovery drive
sudo mkdir /recovery
sudo mount /dev/nvme2n1 /recovery

# Verify file exists
cat /recovery/config.txt
```

### Test B: Launch Instance from AMI
**Scenario:** The main server crashed completely.

1. Go to **AMIs** in the left sidebar.
2. Select `Web-Server-Backup-v1`.
3. Click **Launch instance from AMI**.
4. Proceed through the launch wizard (select keypair, etc.).
5. **Result:** When this new instance boots, it will be an *exact clone* of your original server, including the `/data` mount and files.

---

## 7. Cleanup

To avoid costs, perform the following cleanup steps:

1. **Terminate** the restored instance from Test B.
2. **Detach and Delete** the volumes created in Phase 1 & 4.
3. **Deregister** the AMI (`Web-Server-Backup-v1`).
4. **Delete** the Snapshots associated with the AMI and the Volume.
