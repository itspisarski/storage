# Lab: Introduction to Amazon S3 (Object Storage)

**Objective:**
In this lab, you will transition from file-based storage (NFS) to object storage (S3). You will create a bucket, manage objects via the command line (CLI), and configure a lifecycle policy to automate data management.

---

## 📋 Prerequisites & Setup

**1. Infrastructure**
You need **1 EC2 Instance** (Amazon Linux 2 or 2023) to act as your "client" for interacting with S3 via the command line.

**2. IAM Role (Critical)**
Your EC2 instance needs permission to talk to S3.
* Go to **IAM > Roles** in the AWS Console.
* Create a new Role for **EC2**.
* Attach the policy: `AmazonS3FullAccess` (for lab purposes only; in production, be more restrictive).
* Attach this Role to your EC2 instance (Actions > Security > Modify IAM Role).

---

## Part 1: Create and Manage Buckets via CLI

While you can use the Console, using the AWS CLI is faster and essential for automation.

### 1. Create a Bucket
Bucket names must be globally unique (like a domain name). Replace `yourname` with a unique identifier.

```bash
# Set your region (e.g., us-east-1)
export AWS_DEFAULT_REGION=us-east-1

# Create the bucket (mb = Make Bucket)
aws s3 mb s3://lab-bucket-yourname-123
```

### 2. Upload Objects (Files)
Create some dummy files and upload them.

```bash
# Create local files
echo "This is a log file" > app.log
echo "This is an image" > image.jpg
echo "Important backup data" > backup.tar.gz

# Upload a single file (cp = Copy)
aws s3 cp app.log s3://lab-bucket-yourname-123/
```

### 3. Sync a Directory
Often you want to upload an entire folder at once.

```bash
# Create a folder with multiple files
mkdir my_data
touch my_data/file{1..5}.txt

# Sync the local folder to S3
aws s3 sync my_data/ s3://lab-bucket-yourname-123/my_data/
```

### 4. Verify
List the contents of your bucket.

```bash
aws s3 ls s3://lab-bucket-yourname-123 --recursive
```

---

## Part 2: Versioning and Protection

S3 isn't just a hard drive; it protects your data from accidental deletion.

### 1. Enable Versioning
This keeps a copy of every change you make to a file.

```bash
aws s3api put-bucket-versioning \
    --bucket lab-bucket-yourname-123 \
    --versioning-configuration Status=Enabled
```

### 2. Test Versioning (Overwrite a file)
Let's change the content of `app.log` and re-upload it.

```bash
echo "This is the NEW version of the log" > app.log
aws s3 cp app.log s3://lab-bucket-yourname-123/
```

### 3. Retrieve Versions
You can see both the old and new versions of the file hidden in the bucket.

```bash
aws s3api list-object-versions \
    --bucket lab-bucket-yourname-123 \
    --prefix app.log
```

---

## Part 3: Lifecycle Policies (Automation)

S3 can automatically move data to cheaper storage classes (like Glacier) or delete it after a certain time.

### 1. Create a Policy JSON
Create a file named `lifecycle.json` with the following content. This rule moves objects to **Standard-IA** (Infrequent Access) after 30 days and **Expires** (deletes) them after 365 days.

```json
{
    "Rules": [
        {
            "ID": "MoveToCheaperStorage",
            "Prefix": "",
            "Status": "Enabled",
            "Transitions": [
                {
                    "Days": 30,
                    "StorageClass": "STANDARD_IA"
                }
            ],
            "Expiration": {
                "Days": 365
            }
        }
    ]
}
```

### 2. Apply the Policy
Push this configuration to your bucket.

```bash
aws s3api put-bucket-lifecycle-configuration \
    --bucket lab-bucket-yourname-123 \
    --lifecycle-configuration file://lifecycle.json
```

### 3. Verify
Check that the policy is active.

```bash
aws s3api get-bucket-lifecycle-configuration --bucket lab-bucket-yourname-123
```

---

## Part 4: Cleanup

S3 buckets cannot be deleted if they contain objects (including hidden versions).

### 1. Empty the Bucket
This command deletes all objects and all versions (Dangerous!).

```bash
aws s3 rb s3://lab-bucket-yourname-123 --force
```
*(The `rb` stands for Remove Bucket. The `--force` flag deletes the contents first.)*

### 2. Verify Deletion
```bash
aws s3 ls
```
