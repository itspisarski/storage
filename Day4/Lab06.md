# Lab: Manual Docker Networking on AWS EC2

## 1. Overview
In this lab, you will manually build a 2-tier application (Python Web App + Redis Database) on your AWS EC2 instance. This lab teaches you how containers communicate across a private bridge network using DNS Service Discovery.

### **Lab Objectives**
* Install and configure **Docker** on Amazon Linux.
* Create a custom **Bridge Network**.
* Run a **Redis** container detached from the host.
* Build a custom **Python** web container.
* Connect containers manually so the Web App can talk to the Database.

---

## 2. Phase 0: Installation & Setup (Amazon Linux)

Since you are using a fresh EC2 instance, you must install the Docker engine and start the service.

**1. Update and Install Docker**
Run the following commands in your SSH terminal:
```bash
# Update installed packages
sudo yum update -y

# Install Docker
sudo yum install -y docker

# Start the Docker service
sudo service docker start

# Enable Docker to start on boot
sudo systemctl enable docker
```

**2. Fix Permissions (Crucial Step)**
By default, only `root` can run Docker commands. We will add the default user (`ec2-user`) to the docker group to avoid typing `sudo` every time.

```bash
sudo usermod -aG docker ec2-user
```

**3. Activate Group Changes**
You must log out and log back in for the permission change to take effect.
* **Option A:** Disconnect your SSH session and reconnect.
* **Option B:** Run this command to refresh the group instantly:
    ```bash
    newgrp docker
    ```

**4. Verify Installation**
```bash
docker info
```
*If you see a lot of output details without "Permission denied," you are ready.*

---

## 3. Phase 1: Network Setup

By default, containers on the `default` bridge network cannot resolve each other by name (DNS). We must create a user-defined bridge.

```bash
# 1. Create a custom network
docker network create my-app-net

# 2. Verify it exists
docker network ls
```

---

## 4. Phase 2: The Database Layer

We will start a Redis database and attach it to our network. We will explicitly name it `my-redis` so we can reach it by that hostname later.

```bash
# Run Redis on our custom network
docker run -d \
  --name my-redis \
  --network my-app-net \
  redis:alpine
```

**Verification:**
Check if it is running on the correct network.
```bash
docker network inspect my-app-net
```
*Look for the "Containers" section in the output.*

---

## 5. Phase 3: The Application Layer

We will create a simple Python script that tries to connect to `my-redis`.

### Step 1: Create the Project Folder
```bash
mkdir docker-lab
cd docker-lab
```

### Step 2: Create the Application Files
We will use `cat` to create the files directly in the terminal. Copy and paste these blocks entirely.

**File 1: `app.py`**
```bash
cat <<EOF > app.py
import time
import redis
from flask import Flask

app = Flask(__name__)
# MAGIC HAPPENS HERE: We use the container name 'my-redis' as the hostname
cache = redis.Redis(host='my-redis', port=6379)

def get_hit_count():
    retries = 5
    while True:
        try:
            return cache.incr('hits')
        except redis.exceptions.ConnectionError as exc:
            if retries == 0:
                raise exc
            retries -= 1
            time.sleep(0.5)

@app.route('/')
def hello():
    count = get_hit_count()
    return 'Hello from AWS! I have been seen {} times.\n'.format(count)

if __name__ == "__main__":
    app.run(host="0.0.0.0")
EOF
```

**File 2: `Dockerfile`**
```bash
cat <<EOF > Dockerfile
# Use a lightweight Python image
FROM python:3.9-alpine

# Set working directory
WORKDIR /code

# Install dependencies directly
RUN pip install flask redis

# Copy the script
COPY app.py .

# Command to run the app
CMD ["python", "app.py"]
EOF
```

### Step 3: Build the Image
```bash
docker build -t my-python-app .
```

---

## 6. Phase 4: Connecting the Pieces

Now we run the application container. Crucially, we attach it to the **same network** (`my-app-net`) and expose port 5000.

```bash
docker run -d \
  --name my-web-app \
  --network my-app-net \
  -p 5000:5000 \
  my-python-app
```

### Verification
Since this is a remote AWS server, you have two ways to verify:

**Method A: Curl (Local Test)**
Run this inside your SSH terminal:
```bash
curl http://localhost:5000
```
*Result: "Hello from AWS! I have been seen 1 times."*

**Method B: Browser (Public Internet)**
1.  Find your EC2 instance **Public IP**.
2.  Ensure your AWS **Security Group** allows **Inbound Traffic on Port 5000**.
3.  Open `http://<YOUR_EC2_IP>:5000` in your browser.

---

## 7. Phase 5: Debugging & Network Inspection

Let's "hack" into the running container to see how Docker handles DNS inside the AWS Linux environment.

```bash
# Exec into the web app container
docker exec -it my-web-app sh

# (Inside the container) Check DNS resolution
nslookup my-redis

# (Inside the container) Ping the database by name
ping -c 2 my-redis

# Exit
exit
```

---

## 8. Cleanup

Always clean up your lab environment to free up resources on the EC2 instance.

```bash
# 1. Stop containers
docker stop my-web-app my-redis

# 2. Remove containers
docker rm my-web-app my-redis

# 3. Remove the network
docker network rm my-app-net

# 4. Remove the image
docker rmi my-python-app
```
