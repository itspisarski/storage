# Lab: High-Performance Flash (NVMe Instance Store)

**Objective:**
In this lab, you will work with raw "Instance Store" volumes. You will compare their performance against standard EBS volumes and demonstrate the "ephemeral" (volatile) nature of physical flash storage on the cloud.

---

## 📋 Prerequisites & Setup

**1. Infrastructure (Cost Warning ⚠️)**
* You must launch an instance type that includes **Instance Store** (look for the "d" in the name).
* **Recommended:** `m5d.large` or `c5d.large`.
* **OS:** Amazon Linux 2 or 2023.

**2. Launch Instructions**
* Go to EC2 Console > Launch Instance.
* Name: `Flash-Lab`.
* Instance Type: `m5d.large` (or any type with "1 x 75 NVMe SSD" or similar listed).
* Key Pair: Select yours.
* Launch.

---

## Part 1: Identify the Flash Storage

Instance Store volumes look different from EBS volumes.

### 1. List Block Devices
Run the list block command.

```bash
lsblk
```

### 2. Analyze Output
You will likely see two distinct drives:
1.  `nvme0n1` (Roots disk): This is your **EBS** volume (Network attached).
2.  `nvme1n1` (Ephemerals): This is the **Instance Store** (Physical Flash). It might be unformatted and unmounted.

---

## Part 2: Format and Mount

Just like the first lab, we must format the raw flash drive to use it.

### 1. Format the Drive
We use XFS for performance. (Replace `/dev/nvme1n1` with your specific device name from Step 1).

```bash
sudo mkfs -t xfs /dev/nvme1n1
```

### 2. Mount the Drive
Create a directory specifically for high-speed scratch data.

```bash
sudo mkdir /scratch
sudo mount /dev/nvme1n1 /scratch
sudo chown ec2-user:ec2-user /scratch
```

### 3. Verify
```bash
df -h
```
*You should see your `/scratch` partition ready.*

---

## Part 3: The Performance Test (Benchmark)

We will use `fio` (Flexible I/O Tester) to race the **EBS volume** against the **Flash Instance Store**.

### 1. Install FIO
```bash
sudo dnf install fio -y
```

### 2. Test 1: Standard EBS (Root Volume)
We will run a write test on your home folder (which sits on EBS).

```bash
# Write a 1GB file to EBS
fio --name=ebs_test \
    --filename=~/ebs_test_file \
    --size=1G --rw=write --bs=1M --direct=1 \
    --ioengine=libaio --iodepth=4 --runtime=60 --group_reporting
```
> **Note the Result:** Look at the `bw=` (Bandwidth) line. It will likely be around **100-200 MiB/s** (depending on the EBS type).

### 3. Test 2: Local Flash (Instance Store)
Now run the exact same test on the mounted flash drive.

```bash
# Write a 1GB file to NVMe Flash
fio --name=flash_test \
    --filename=/scratch/flash_test_file \
    --size=1G --rw=write --bs=1M --direct=1 \
    --ioengine=libaio --iodepth=4 --runtime=60 --group_reporting
```
> **Note the Result:** Look at the `bw=` line. On an `m5d` instance, this is often **over 400-1000 MiB/s**. It is significantly faster and has lower latency.

---

## Part 4: The "Volatile" Test (Data Loss)

This is the most critical lesson. Physical instance stores are tied to the hardware power state.

### 1. Create Data
Create a critical file on the flash drive.

```bash
echo "This data is super important" > /scratch/my_data.txt
cat /scratch/my_data.txt
```

### 2. Stop the Instance
Go to the AWS Console.
* **Action:** Stop Instance (Do not Terminate, just Stop).
* Wait for it to say "Stopped".

### 3. Start the Instance
* **Action:** Start Instance.
* Wait for it to run.
* SSH back into the server.

### 4. Check the Data
Run `lsblk` and check your folder.

```bash
lsblk
ls /scratch
```

**Observation:**
1.  The mount `/scratch` is gone.
2.  Even if you remount it, **the file is gone.**
3.  The disk was likely reset to a blank slate (or encrypted with a new key) by AWS during the hardware reset.

---

## 🧹 Cleanup

Since `m5d` instances are not free, terminate this immediately after finishing.

1.  **Terminate Instance** via AWS Console.

---

## 🧠 Key Takeaways

| Feature | EBS (Standard) | Instance Store (Flash) |
| :--- | :--- | :--- |
| **Location** | Networked Data Center | Physically inside the Server |
| **Speed** | Fast (Network limited) | Extremely Fast (PCIe limited) |
| **Persistence** | Data survives reboot/stop | **Data DOES NOT survive stop** |
| **Best Use** | Databases, OS, Persistent Files | Cache, Buffers, Temporary Scratch Data |
