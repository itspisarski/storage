# Lab: Configure Network File System (NFS) on AWS EC2

**Objective:**
In this lab, you will move from block storage (local disk) to file storage (network share). You will configure a central Linux server to share directories over the network and connect multiple client servers to access that data simultaneously.

---

## 📋 Prerequisites & Setup

**1. Infrastructure**
You need **3 EC2 Instances** (Amazon Linux 2 or 2023 recommended) in the same VPC/Subnet.
* **Server A:** Will act as the **NFS Server** (Target).
* **Server B:** Client 1.
* **Server C:** Client 2 (Optional, but best for testing concurrency).

**2. Security Group Rules (Critical)**
NFS traffic will be blocked by default. You must edit the Security Group attached to **Server A** (The NFS Server).

* **Type:** `NFS`
* **Protocol:** `TCP`
* **Port:** `2049`
* **Source:** The Private IP range of your VPC (e.g., `172.31.0.0/16`).
    * *Note: If you are lazy for the lab, you can use the specific Private IPs of Server B and C, but using the VPC CIDR is standard practice.*

---

## Part 1: Configure the NFS Server
*Perform these steps on **Server A**.*

### 1. Install NFS Utilities
```bash
sudo dnf update -y
sudo dnf install nfs-utils -y
```

### 2. Create Share Directories
We will create two separate shares to test different permissions:
* `/srv/nfs/read_write` for shared team data.
* `/srv/nfs/read_only` for static configuration data.

```bash
sudo mkdir -p /srv/nfs/read_write
sudo mkdir -p /srv/nfs/read_only
```

### 3. Configure File Permissions
NFS maps remote `root` users to a specific local user called `nobody` (nfsnobody). We must ensure this user owns the directory so clients can write to it.

```bash
sudo chown nobody:nobody /srv/nfs/read_write
sudo chmod 777 /srv/nfs/read_write
```

### 4. Define Exports (Share Rules)
Edit the `/etc/exports` file. This file controls which folders are shared and with whom.

```bash
sudo nano /etc/exports
```

Add the following lines. Replace `*` with your VPC CIDR (e.g., `172.31.0.0/16`) for better security, or keep `*` to allow any IP to try connecting.

```text
/srv/nfs/read_write  *(rw,sync,no_subtree_check)
/srv/nfs/read_only   *(ro,sync,no_subtree_check)
```

> **Flag Breakdown:**
> * `rw`: Read and Write access.
> * `ro`: Read Only access.
> * `sync`: Request changes are written to disk before replying (reliability).
> * `no_subtree_check`: Improves reliability when sharing entire subdirectories.

### 5. Start the Service
Enable the NFS server and apply the configuration.

```bash
sudo systemctl enable --now nfs-server
sudo exportfs -r  # Re-reads the /etc/exports file
```

---

## Part 2: Configure NFS Clients
*Perform these steps on **Server B** and **Server C**.*

### 1. Install NFS Utilities
The clients need the NFS software to understand the protocol.

```bash
sudo dnf install nfs-utils -y
```

### 2. Create Mount Points
Create the local folders where the remote data will appear.

```bash
sudo mkdir -p /mnt/team_data
sudo mkdir -p /mnt/static_data
```

### 3. Mount the Shares
Mount the remote folders to your local directories.
*(Replace `NFS_SERVER_PRIVATE_IP` with the actual Private IP of Server A, e.g., `172.31.10.5`)*

```bash
# Mount the Read-Write Share
sudo mount -t nfs NFS_SERVER_PRIVATE_IP:/srv/nfs/read_write /mnt/team_data

# Mount the Read-Only Share
sudo mount -t nfs NFS_SERVER_PRIVATE_IP:/srv/nfs/read_only /mnt/static_data
```

### 4. Verify Connection
Run this command to check if the mounts are active:
```bash
df -h
```
*You should see the NFS Server IP listed at the bottom of the output.*

---

## Part 3: Testing Scenarios

### Test 1: Concurrent File Access (The "Chat" Test)
We will prove that both clients are looking at the same file in real-time.

1.  **On Server B (Client 1):** Create a file and start "watching" it.
    ```bash
    touch /mnt/team_data/chat.log
    tail -f /mnt/team_data/chat.log
    ```
    *(The terminal will hang here, waiting for data. Keep this window open.)*

2.  **On Server C (Client 2):** Write text into that same file.
    ```bash
    echo "Hello from Server C!" >> /mnt/team_data/chat.log
    ```

3.  **Result:** Look back at **Server B**. You should see the message appear instantly. This confirms shared access.

### Test 2: Permission Enforcement
We will prove that the `/mnt/static_data` share is truly Read-Only.

1.  **On Server B (Client 1):** Try to create a file in the read-only folder.
    ```bash
    touch /mnt/static_data/hacker_file.txt
    ```

2.  **Result:** You should receive a strictly enforced error:
    > `touch: cannot touch '/mnt/static_data/hacker_file.txt': Read-only file system`

---

## 🧹 Cleanup & Persistence (Optional)

**To make these mounts permanent:**
If you reboot the clients, the mounts will disappear. To fix this, add them to `/etc/fstab` on the clients:

```text
NFS_SERVER_IP:/srv/nfs/read_write  /mnt/team_data   nfs  defaults  0  0
```

**To clean up:**
1.  Unmount on clients: `sudo umount /mnt/team_data`
2.  Terminate instances if you are done with the lab.
