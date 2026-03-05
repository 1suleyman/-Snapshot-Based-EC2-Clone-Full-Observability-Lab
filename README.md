# ☁️ Snapshot-Based EC2 Clone + Full Observability Lab

In this lab, I learned how to **clone an EC2 environment using snapshots and AMIs**, configure **CloudWatch observability**, and stream **Docker container logs into CloudWatch Logs**.

This lab simulates a **real DevOps troubleshooting workflow** where engineers must:

* Clone infrastructure
* Inspect disk snapshots
* Understand filesystem behavior
* Implement observability
* Build dashboards for monitoring

---

# 📋 Lab Overview

## Goal

Create a **clone of an EC2 environment**, configure **system observability**, and stream **Docker logs into CloudWatch**.

The lab also demonstrates the difference between:

* **EBS Snapshots (storage clones)**
* **AMI images (full machine clones)**

---

## Learning Outcomes

By the end of this lab I was able to:

* Launch and configure an EC2 instance
* Install and run Docker containers
* Generate application logs
* Create **EBS snapshots**
* Clone storage volumes from snapshots
* Understand **XFS duplicate UUID issues**
* Create **true EC2 clones using AMIs**
* Install and configure the **CloudWatch Agent**
* Send **Docker container logs to CloudWatch Logs**
* Build a **CloudWatch monitoring dashboard**

---

# 🏗 Architecture

```
             +-----------------------+
             |   CloudWatch          |
             |-----------------------|
             | Metrics (CWAgent)     |
             | Logs (/aws/ec2/docker)|
             | Dashboard             |
             +----------▲------------+
                        |
                        |
              CloudWatch Agent
                        |
                        |
+-------------------------------------------+
| EC2 Clone Instance                        |
|-------------------------------------------|
| Docker Engine                             |
|   └─ log-generator container              |
|       └─ writes logs                      |
|                                           |
| /var/lib/docker/containers/*-json.log     |
+-------------------------------------------+
```

---

# 🛠 Step-by-Step Journey

---

# Step 1: Launch the Original EC2 Instance

Create the **baseline instance** that will later be cloned.

**Configuration**

| Setting       | Value                 |
| ------------- | --------------------- |
| Instance Name | `OGEC2`               |
| Instance Type | `t3.medium`           |
| Key Pair      | `labec2.pem`          |
| Monitoring    | Default               |
| IAM Role      | CloudWatch Agent role |

---

# Step 2: Create IAM Role for CloudWatch Agent

The CloudWatch agent needs permissions to send metrics and logs.

### Create Role

Trusted Entity:

```
AWS Service → EC2
```

Attach policy:

```
CloudWatchAgentServerPolicy
```

Role Name:

```
ec2-cloudwatch-agent-role
```

Description:

```
Allow EC2 instances to send metrics and logs to CloudWatch
```

Attach this role to the EC2 instance.

---

# Step 3: Connect to the Instance

Navigate to the key location and secure the private key.

```bash
cd Downloads
chmod 400 labec2.pem
```

SSH into the instance:

```bash
ssh -i "labec2.pem" ec2-user@<public-ip>
```

---

# Step 4: Install Docker

Update packages:

```bash
sudo yum update -y
```

Install Docker:

```bash
sudo dnf install docker -y
```

Verify installation:

```bash
docker --version
```

Start Docker service:

```bash
sudo systemctl start docker
sudo systemctl status docker
```

---

# Step 5: Test Docker

Run a test container:

```bash
sudo docker run hello-world
```

Verify containers:

```bash
sudo docker ps -a
```

This confirms Docker is installed and functioning correctly.

---

# Step 6: Create a Log-Generating Container

Run a container that continuously generates logs.

```bash
sudo docker run -d \
--name log-generator \
busybox \
sh -c "while true; do echo 'test log entry'; sleep 2; done"
```

Explanation:

| Component    | Purpose                     |
| ------------ | --------------------------- |
| `-d`         | Run container in background |
| `--name`     | Assign container name       |
| `busybox`    | Lightweight Linux image     |
| `while true` | Infinite log loop           |
| `sleep 2`    | Prevents log flooding       |

---

### Verify Logs

```bash
sudo docker logs log-generator
```

---

# Step 7: Prepare Instance for Snapshot

Stop the container:

```bash
sudo docker stop log-generator
```

Stop the EC2 instance.

This ensures a **clean filesystem state** before snapshot creation.

---

# Step 8: Create an EBS Snapshot

Navigate to:

```
EC2 → Volumes
```

Select the root volume and create snapshot.

Snapshot name:

```
lab-pre-clone-baseline
```

Snapshots create a **point-in-time disk copy**.

---

# Step 9: Attempt Snapshot Clone (Storage Clone)

Create a new volume from the snapshot and attach it to a new EC2 instance.

Attach volume as:

```
/dev/sdb
```

---

# Step 10: Mount Snapshot Volume

Check available disks:

```bash
lsblk
```

Create mount point:

```bash
sudo mkdir -p /data
```

Mount disk:

```bash
sudo mount /dev/nvme1n1p1 /data
```

---

# Step 11: Resolve XFS Duplicate UUID Error

Snapshot cloning duplicates the filesystem UUID.

Linux prevents mounting duplicate XFS filesystems.

Error example:

```
Filesystem has duplicate UUID
```

---

### Temporary Fix

```bash
sudo mount -t xfs -o nouuid /dev/nvme1n1p1 /data
```

---

### Permanent Fix (Regenerate UUID)

Unmount disk:

```bash
sudo umount /data
```

Generate new UUID:

```bash
sudo xfs_admin -U generate /dev/nvme1n1p1
```

Mount again:

```bash
sudo mount -t xfs /dev/nvme1n1p1 /data
```

Verify:

```bash
df -hT
```

---

# Step 12: Discover Snapshot vs Machine Clone

Docker was **missing on the new instance**.

Reason:

Snapshots clone **storage**, not the **machine configuration**.

EC2 consists of:

| Layer   | Description        |
| ------- | ------------------ |
| Compute | CPU and RAM        |
| Storage | EBS volumes        |
| OS      | Installed software |

Docker existed only on the original **root disk**, not the new instance.

---

# Step 13: Create True Machine Clone with AMI

Create AMI from the original instance.

```
EC2 → Actions → Image → Create Image
```

Name:

```
lab-pre-clone-baseline
```

An AMI includes:

* root disk snapshot
* additional volumes
* boot configuration
* installed software

---

# Step 14: Launch EC2 from AMI

Launch new instance:

```
OGEC2-clone
```

Same configuration:

| Setting       | Value                  |
| ------------- | ---------------------- |
| Instance Type | t3.medium              |
| Key Pair      | labec2                 |
| AMI           | lab-pre-clone-baseline |

---

# Step 15: Verify Clone

SSH into new instance.

Check Docker containers:

```bash
sudo docker ps -a
```

Expected output:

```
hello-world
log-generator
```

Check logs:

```bash
sudo docker logs log-generator
```

This confirms the **machine clone worked correctly**.

---

# Step 16: Install CloudWatch Agent

Install agent:

```bash
sudo yum install amazon-cloudwatch-agent -y
```

---

# Step 17: Run Configuration Wizard

Launch wizard:

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard
```

Key configuration:

| Option       | Value      |
| ------------ | ---------- |
| OS           | Linux      |
| Environment  | EC2        |
| Metrics      | Basic      |
| Resolution   | 60 seconds |
| Collect logs | Yes        |

Docker log path:

```
/var/lib/docker/containers/*/*.log
```

Log group:

```
/aws/ec2/docker-containers
```

---

# Step 18: Start CloudWatch Agent

Run agent:

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
-a fetch-config \
-m ec2 \
-c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json \
-s
```

Verify:

```bash
sudo systemctl status amazon-cloudwatch-agent
```

---

# Step 19: Create CloudWatch Dashboard

Navigate to:

```
CloudWatch → Dashboards → Create Dashboard
```

Name:

```
lab-clone-dashboard
```

Add widgets:

| Metric          |
| --------------- |
| CPU Utilization |
| Memory Used %   |
| Disk Used %     |
| Network In      |
| Network Out     |

<img width="2834" height="344" alt="Pasted Graphic" src="https://github.com/user-attachments/assets/50e1699a-6fcd-4cd6-9f25-b15297ca79d0" />

---

# Step 20: Create Log Insights Widget

Use query:

```
fields @timestamp, @message
| sort @timestamp desc
| limit 300
```

Log group:

```
/aws/ec2/docker-containers
```

---

# Step 21: Fix Log Permission Issue

CloudWatch agent couldn't read Docker logs.

Fix by editing config:

```bash
sudo vi /opt/aws/amazon-cloudwatch-agent/bin/config.json
```

Change:

```
run_as_user: root
```

Reload configuration.

---

# Step 22: Generate Logs

Restart container:

```bash
sudo docker start log-generator
```

Verify logs appear in CloudWatch dashboard.

<img width="2840" height="1378" alt="Pasted Graphic 1" src="https://github.com/user-attachments/assets/8d5faaaa-b46d-4c21-9a04-5f62b7e0ddc7" />

---

# Step 23: Create Observability Baseline Snapshot

Stop container:

```bash
sudo docker stop log-generator
```

Stop instance and create snapshot.

<img width="794" height="134" alt="image" src="https://github.com/user-attachments/assets/4416a010-595a-4296-b5bb-e8b5bbd1e6bb" />

This acts as a **rollback checkpoint**.

---

# Step 24: Clean Up Resources

Delete:

* EC2 instances
* Snapshots
* AMIs
* CloudWatch dashboards
* EBS volumes

This prevents unnecessary AWS costs.

---

# ✅ Key Commands Summary

| Task               | Command                                       |
| ------------------ | --------------------------------------------- |
| Install Docker     | `sudo dnf install docker -y`                  |
| Start Docker       | `sudo systemctl start docker`                 |
| Run container      | `sudo docker run hello-world`                 |
| Generate logs      | BusyBox loop container                        |
| Create mount point | `sudo mkdir /data`                            |
| Mount volume       | `sudo mount /dev/nvme1n1p1 /data`             |
| Fix XFS UUID       | `sudo xfs_admin -U generate`                  |
| Install CW Agent   | `sudo yum install amazon-cloudwatch-agent -y` |
| Start agent        | `amazon-cloudwatch-agent-ctl`                 |
| View containers    | `sudo docker ps -a`                           |

---

# 💡 Notes / Tips

### Snapshot vs AMI

| Snapshot         | AMI                 |
| ---------------- | ------------------- |
| Copies storage   | Copies full machine |
| Disk clone       | Instance template   |
| Used for backups | Used for scaling    |

---

### Why XFS Blocks Duplicate UUIDs

Snapshots copy disks **bit-for-bit**, including filesystem metadata.

XFS blocks duplicates to prevent:

* disk corruption
* mount confusion
* data loss

---

### Why Engineers Mount Snapshots

Mounting snapshots allows:

* forensic debugging
* configuration recovery
* container log inspection
* disaster recovery analysis

Without booting the system.

---

# ✅ References

AWS Documentation:

EC2 Snapshots
[https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EBSSnapshots.html](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EBSSnapshots.html)

CloudWatch Agent
[https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Install-CloudWatch-Agent.html](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Install-CloudWatch-Agent.html)

Docker Logs
[https://docs.docker.com/config/containers/logging/](https://docs.docker.com/config/containers/logging/)
