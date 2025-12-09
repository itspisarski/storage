# Lab: Implementing iSCSI Block Storage (Linux-to-Linux)

## Overview
In this lab, you will simulate a Storage Area Network (SAN) entirely in the cloud. You will configure one Linux server as the **Target** (storage array) and a second Linux server as the **Initiator** (client). You will create a raw block device on the Target and mount it over the network to the Initiator.

## Prerequisites
1.  **Target Instance:** Linux EC2 (Amazon Linux 2023 or Ubuntu).
2.  **Initiator Instance:** A **second** Linux EC2 instance.
3.  **Network:** The **Target's Security Group** must allow **Custom TCP** traffic on port **3260** from the Initiator's IP.

---

## Phase 1: Storage Preparation (Target Setup)
*Since standard EC2 instances only have a root drive, we must create and attach a secondary "physical" disk to the Target instance.*

1.  **Identify Availability Zone:**
    * Go to the **EC2 Dashboard** -> **Instances**.
    * Select your **Target** instance.
    * Note the **Availability Zone** (e.g., `us-east-1a`).

2.  **Create the Volume:**
    * In the left menu, click **Volumes** (under Elastic Block Store).
    * Click **Create volume**.
    * **Size:** Enter `5` (5 GiB).
    * **Availability Zone:** **Critical:** Select the *exact same zone* as your Target instance.
    * Click **Create volume**.

3.  **Attach the Volume:**
    * Right-click the new volume (Status: "Available") and select **Attach volume**.
    * **Instance:** Select your **Target** Linux instance.
    * **Device name:** Keep default (e.g., `/dev/sdf`).
    * Click **Attach volume**.

4.  **Verify:**
    * SSH into the **Target** instance.
    * Run `lsblk`.
    * **Success Check:** You should see a secondary disk (e.g., `nvme1n1` or `xvdf`) approx 5GB in size.

---

## Phase 2: Configure the iSCSI Target
*Perform these steps on the **Target** instance.*

1.  **Install Target Tools:**
    ```bash
    # Amazon Linux 2023:
    sudo yum install targetcli -y
    sudo systemctl enable --now target

    # Ubuntu:
    sudo apt update && sudo apt install targetcli-fb -y
    sudo systemctl enable --now target
    ```

2.  **Enter Admin Shell:**
    ```bash
    sudo targetcli
    ```

3.  **Create Backstore (The Disk):**
    *Replace `/dev/nvme1n1` with your specific device name from Phase 1.*
    ```bash
    /> cd /backstores/block
    /backstores/block> create lun0 /dev/nvme1n1
    ```

4.  **Create Target IQN (The Identity):**
    ```bash
    /backstores/block> cd /iscsi
    /iscsi> create iqn.2025-03.lab:storage.target01
    ```

5.  **Create Portal (The Network Listener):**
    *Note: If `0.0.0.0 3260` already exists, skip the delete step.*
    ```bash
    /iscsi> cd iqn.2025-03.lab:storage.target01/tpg1/portals
    .../portals> delete 0.0.0.0 3260
    .../portals> create 0.0.0.0 3260
    ```

6.  **Map the LUN:**
    ```bash
    .../portals> cd ../luns
    .../luns> create /backstores/block/lun0
    ```

7.  **Configure Access (Demo Mode):**
    *Allow connection without password (CHAP) for this lab.*
    ```bash
    .../luns> cd ../
    .../tpg1> set attribute authentication=0
    .../tpg1> set attribute demo_mode_write_protect=0
    .../tpg1> set attribute generate_node_acls=1
    .../tpg1> set attribute cache_dynamic_acls=1
    ```

8.  **Save and Exit:**
    ```bash
    .../tpg1> cd /
    /> saveconfig
    /> exit
    ```

---

## Phase 3: Configure the iSCSI Initiator
*Perform these steps on the **Initiator** (the second Linux instance).*

1.  **Install Initiator Tools:**
    ```bash
    # Amazon Linux 2023:
    sudo dnf install iscsi-initiator-utils -y

    # Ubuntu:
    sudo apt update && sudo apt install open-iscsi -y
    ```

2.  **Start Service:**
    ```bash
    sudo systemctl enable --now iscsid
    ```

3.  **Discover the Target:**
    *Replace `<TARGET_PRIVATE_IP>` with the internal IP of your Target instance.*
    ```bash
    sudo iscsiadm -m discovery -t sendtargets -p <TARGET_PRIVATE_IP>
    ```
    * **Success:** It prints the IQN: `iqn.2025-03.lab:storage.target01`.

4.  **Log In (Connect):**
    ```bash
    sudo iscsiadm -m node -T iqn.2025-03.lab:storage.target01 -p <TARGET_PRIVATE_IP> -l
    ```
    * **Success:** You will see "Login to [target: ...] successful."

---

## Phase 4: Mount and Verify
*Perform these steps on the **Initiator**.*

1.  **Find the New Disk:**
    ```bash
    lsblk
    ```
    * You should see a new disk (often `sda`, `sdb`, or `dm-0`) with the 5GB size.

2.  **Format the Disk:**
    *Create a file system on the raw block device. Replace `/dev/sda` with your new disk name.*
    ```bash
    sudo mkfs -t xfs -f -K /dev/sda
    ```
> This step might take few minutes to run (~2)
3.  **Mount the Storage:**
    ```bash
    sudo mkdir /data
    sudo mount /dev/sda /data
    ```

4.  **Test Write Access:**
    *Verify you can write data to the remote drive.*
    ```bash
    sudo touch /data/lab_success.txt
    ls -l /data
    ```
    * **Concept Check:** You are executing file commands on the Initiator, but the physical data is being written to the EBS volume attached to the Target instance!

---

## Phase 5: Cleanup
1.  **Unmount (Initiator):** `sudo umount /mnt/san_storage`
2.  **Logout (Initiator):** `sudo iscsiadm -m node -u`
3.  **Terminate:** Delete both EC2 instances and the extra 5GB volume to prevent AWS charges.
