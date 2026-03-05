Below is a **step-by-step runbook** someone else can follow to reproduce your whole lab:

**Snapshot-Based EC2 Clone + Full Observability Setup (Docker logs → CloudWatch + Dashboard)**

---

## 0) What this runbook achieves

**Goal:** Take an existing “known-good” EC2 (with Docker + containers), create a **clone EC2 from an AMI**, then add **full telemetry**:

* EC2 clone created from AMI
* Docker installed/running (if needed)
* CloudWatch Agent installed + configured for:

  * Host metrics (memory + disk, etc.)
  * Docker container log shipping to CloudWatch Logs
* CloudWatch dashboard showing CPU/mem/disk/network + Log Insights
* Cleanup steps to avoid cost

(Your transcript shows the CloudWatch Agent wizard + config choices + log group path attempts + common errors like missing IAM role and permission denied.)   

---

## 1) Preconditions / Inputs

You need:

1. **Source EC2 instance** (the “original”) you want to clone.
2. **Key pair** to SSH in (e.g. `labec2.pem`) and correct username (`ec2-user` on Amazon Linux 2023). 
3. **IAM role (instance profile)** for the new clone instance with at least:

   * `CloudWatchAgentServerPolicy` (required for agent to publish metrics/logs)
   * (Recommended) `AmazonSSMManagedInstanceCore` if you want SSM later
4. Region: your outputs show **eu-west-2** (London).

---

## 2) Create the “clean clone” using an AMI

### Step 2.1 — Create AMI from the source instance

**Console path:** EC2 → Instances → select **source instance** → **Actions → Image and templates → Create image**

* Name: `og-ec2-baseline-ami` (or similar)
* Keep volumes as-is (this captures the root volume + any attached EBS volumes)

**Expected:** AMI becomes **Available** (EC2 → AMIs).

> Why AMI (vs manually attaching snapshot volumes)?
> AMI is the “packaged server blueprint” AWS can boot directly, including block device mappings.

### Step 2.2 — Launch new EC2 from AMI

**Console path:** EC2 → AMIs → select AMI → **Launch instance from AMI**

Pick:

* Instance name: `og-ec2-clone`
* Instance type: `t3.micro` (lab)
* Key pair: your `.pem`
* Network/Security group: allow SSH from your IP
* **IMPORTANT:** Attach IAM Role (Instance profile) = role with `CloudWatchAgentServerPolicy`

**Expected:** Instance reaches **Running**, status checks **2/2 passed**.

---

## 3) SSH into the clone (verify you can access it)

### Step 3.1 — Fix file permissions on the key

```bash
chmod 400 "labec2.pem"
```

(Your Mac transcript shows this exact flow.) 

### Step 3.2 — SSH in as the correct user (Amazon Linux 2023 uses `ec2-user`)

```bash
ssh -i "labec2.pem" ec2-user@<public-dns>
```

**Expected:** You land at:

```text
[ec2-user@ip-... ~]$
```

> If you try `root@...` you’ll get blocked (“Please login as the user ec2-user rather than root”). 

---

## 4) Validate the clone actually copied your workload (Docker + containers)

### Step 4.1 — Check Docker

```bash
sudo docker ps -a
```

**Expected:** You should see previous containers (example from your run): `busybox` log generator + `hello-world`.  

### Step 4.2 — Check the container logs still exist (proves clone fidelity)

```bash
sudo docker logs log-generator
```

**Expected:** repeated lines like:

```text
Test log entry
Test log entry
...
```



---

## 5) If Docker is missing on the clone: install & enable it (Amazon Linux 2023)

Sometimes your clone may not have Docker installed (depends what was on the source and what exactly you cloned). Your transcript shows a case where `docker: command not found`, then you installed with `dnf`. 

### Step 5.1 — Install Docker

```bash
sudo dnf install docker -y
```

**Expected:** packages install successfully. 

### Step 5.2 — Start + enable Docker

```bash
sudo systemctl enable --now docker
```

### Step 5.3 — (Optional) allow ec2-user to run docker without sudo

```bash
sudo usermod -aG docker ec2-user
```

Log out and back in.

---

## 6) Create a test log source (log-generator container)

If you don’t already have one running, create it:

```bash
sudo docker run -d --name log-generator busybox sh -c "while true; do echo 'Test log entry'; sleep 2; done"
```

**Expected:** container runs and generates logs.

Verify:

```bash
sudo docker ps
sudo docker logs --tail 20 log-generator
```

---

## 7) Install CloudWatch Agent on the clone

### Step 7.1 — Install

```bash
sudo yum install amazon-cloudwatch-agent -y
```

**Expected:** package installs; it creates `cwagent` user/group.  

---

## 8) Configure CloudWatch Agent (wizard)

### Step 8.1 — Run the config wizard

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard
```

Use the choices matching your working run:

**Core choices:**

* OS: `linux` (1) 
* Host type: `EC2` (1) 
* Run as user: `cwagent` (default) 
* StatsD: `no` (2) 
* CollectD: `no` (2) 
* Host metrics: `yes`
* CPU per core: (your choice; you were stepping through this)
* Append EC2 dimensions: `yes` (default) → adds InstanceId/ImageId/etc 
* Aggregate on InstanceId: `yes` (default) 
* Resolution: `60s` (4) 
* Default metrics config: `Basic` (1) 
* Satisfied with config: `yes` 

**Logs:**

* “Monitor any log files?” → `yes`
* Log file path: **IMPORTANT: use the real docker json log pattern**

  * In your instance, the real files were like:
    `/var/lib/docker/containers/<container-id>/<container-id>-json.log` 
  * So use this in the wizard:

    ```text
    /var/lib/docker/containers/*/*-json.log
    ```
  * (Your first attempt used `/var/lib/docker/containers/*/*.log`, which didn’t match the actual filenames you later found.)  
* Log group name: `/aws/ec2/docker-containers` (you used this) 
* Log group class: `STANDARD` (default)
* Log stream name: `{instance_id}` (default) 

**Retention gotcha (you hit this):**

* The wizard rejected `-1` even though it later printed it in JSON. 
* In the wizard, choose **1 (the default choice)** or a specific number of days (e.g. 30).
* If you truly want “never expire”, do retention management in the console later.

**Config file output:**

* Save to: `/opt/aws/amazon-cloudwatch-agent/bin/config.json` (as you did) 

---

## 9) Start the agent using your config

Run exactly:

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
-a fetch-config \
-m ec2 \
-c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json \
-s
```

**Expected:** “Configuration validation succeeded” and the service becomes active. 

Verify:

```bash
sudo systemctl status amazon-cloudwatch-agent
```

**Expected:** `active (running)` 

---

## 10) Mandatory AWS-side requirement: attach IAM role to the instance

If you forget the IAM role, CloudWatch Agent can’t call AWS APIs.

Your logs showed:

* `NoCredentialProviders`
* `no EC2 instance role found`
* plus an EC2 metadata error payload 

### Step 10.1 — Fix it (console)

EC2 → Instances → select clone → Actions → Security → **Modify IAM role**

Attach a role that includes:

* `CloudWatchAgentServerPolicy`

**Expected:** Those “NoCredentialProviders” errors stop after agent restart.

Restart agent:

```bash
sudo systemctl restart amazon-cloudwatch-agent
```

---

## 11) Troubleshooting: “permission denied” disk errors (overlay2)

You saw repeated errors like:

```text
[inputs.disk] ... error getting disk usage ("/var/lib/docker/overlay2/.../merged"): permission denied
```

 

### What’s happening (in plain terms)

* Your agent is running as **cwagent** (non-root). 
* Docker’s overlay filesystem paths under `/var/lib/docker/overlay2/.../merged` can be restricted by permissions/SELinux.
* The disk plugin tries to read usage stats and gets blocked.

### Two clean fixes (choose one)

**Fix A (simplest): run agent as root**

1. Edit config:

```bash
sudo vi /opt/aws/amazon-cloudwatch-agent/bin/config.json
```

Change:

```json
"run_as_user": "cwagent"
```

to:

```json
"run_as_user": "root"
```

2. Re-apply config + restart:

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
-a fetch-config \
-m ec2 \
-c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json \
-s
```

**Fix B (cleaner for metrics): exclude docker overlay paths**
Keep cwagent, but restrict disk “resources” to real mount points (like `/` and maybe `/data`), not overlay2 internals. That avoids the permission-denied spam.

---

## 12) Verify metrics are arriving (CloudWatch)

### Step 12.1 — Metrics

CloudWatch → Metrics → look for namespace:

* `CWAgent`

You should see things like:

* `mem_used_percent`
* `disk_used_percent`
  …and EC2 dimensions like InstanceId, InstanceType, etc. 

If you don’t see `CWAgent`:

* 99% of the time it’s **IAM role missing** or agent not running.

### Step 12.2 — Logs

CloudWatch → Logs → Log groups:

* `/aws/ec2/docker-containers` (your chosen log group) 

If no logs show up:

* confirm the file path matches actual files:

  ```bash
  CID=$(sudo docker inspect -f '{{.Id}}' log-generator)
  sudo ls -l /var/lib/docker/containers/$CID/
  ```

  You should see `*-json.log` files. 

---

## 13) Build the CloudWatch Dashboard (and **save it**)

CloudWatch → Dashboards → Create dashboard → name: `og-ec2-clone-observability`

Add widgets:

### Widget 1 — CPUUtilization (EC2 native)

Metrics → EC2 → Per-Instance Metrics → `CPUUtilization` → pick your clone instance

### Widget 2 — NetworkIn / NetworkOut (EC2 native)

Metrics → EC2 → Per-Instance Metrics → `NetworkIn`, `NetworkOut`

### Widget 3 — Memory (CWAgent)

Metrics → `CWAgent` → `mem_used_percent` → filter by your InstanceId

### Widget 4 — Disk (CWAgent)

Metrics → `CWAgent` → `disk_used_percent` → select the relevant device/path rows (don’t blindly select overlay paths if you’re troubleshooting permissions)

### Widget 5 — Logs Insights (Docker log group)

Add widget → Logs → Logs table (or line) → select log group: `/aws/ec2/docker-containers` 

Query:

```sql
fields @timestamp, @message
| sort @timestamp desc
| limit 300
```

✅ **IMPORTANT:** Click **Add to dashboard** AND **Save dashboard** (you learned this the hard way 😅).

---

## 14) Validation checklist (final “done” criteria)

On the clone EC2:

* [ ] `sudo docker ps` shows `log-generator` running
* [ ] `sudo docker logs --tail 5 log-generator` prints new “Test log entry” lines
* [ ] `sudo systemctl status amazon-cloudwatch-agent` is active (running) 
* [ ] CloudWatch Metrics shows `CWAgent` namespace with `mem_used_percent` and `disk used_percent` 
* [ ] CloudWatch Logs has `/aws/ec2/docker-containers` and streams log entries 
* [ ] Dashboard saved with CPU, network, mem, disk, and Logs Insights

---

## 15) Cleanup (avoid surprise costs)

### Step 15.1 — Terminate the clone instance

EC2 → Instances → select clone → Instance state → **Terminate**

### Step 15.2 — Delete the AMI + snapshots (if lab-only)

EC2 → AMIs → select your AMI → Actions → **Deregister AMI**

Then:
EC2 → Snapshots → delete the snapshots created by that AMI (only if you’re sure you don’t need them)

### Step 15.3 — Delete CloudWatch artifacts (optional)

* CloudWatch → Dashboards → delete dashboard
* CloudWatch → Logs → Log groups → delete `/aws/ec2/docker-containers` (only if lab-only)

---

## 16) “Common failures” quick map (so the next person doesn’t get stuck)

1. **No `CWAgent` namespace in Metrics**

* Cause: missing IAM role / CloudWatchAgentServerPolicy
* Symptom: `NoCredentialProviders` / `no EC2 instance role found` 
* Fix: attach role, restart agent

2. **Logs not found at `/var/lib/docker/containers/*/*.log`**

* Cause: wrong glob
* Fix: use `/var/lib/docker/containers/*/*-json.log` (matches real docker log files) 

3. **Disk plugin “permission denied” on overlay2**

* Cause: cwagent user can’t read overlay merged paths 
* Fix: run agent as root or exclude overlay paths
