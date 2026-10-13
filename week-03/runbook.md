# Week 3 Runbook: Deploying PropertyLite to EC2

**Goal:** Run the PropertyLite Flask app (from Week 0) on a real EC2 instance, reachable from my laptop over the internet.

**Environment:** Ubuntu (WSL2) on Windows, AWS CLI configured with my IAM admin user, region `us-east-1`.

---

## 1. Launch the instance

1. **EC2 → Launch instance.** I named it `property-api-01` so that it's easy to find later.
2. **AMI:** Amazon Linux 2023 (Free Tier eligible).
3. **Instance type:** `t3.micro` (Free Tier).
4. **Key pair:** I created a new RSA key pair named `training-key` in `.pem` format. The browser downloads it once and it can't be re-downloaded, so I moved it straight into WSL:
   ```bash
   mkdir -p ~/.ssh
   cp /mnt/c/Users/altav/Downloads/training-key.pem ~/.ssh/
   chmod 400 ~/.ssh/training-key.pem
   ```
   I kept it inside the Linux filesystem because `chmod 400` has no effect on `/mnt/c`, and SSH refuses keys with loose permissions. I also deleted the copy in Downloads and keep the key outside my git repo.
5. **Network settings:** Auto-assign public IP = Enable. I created a new security group `property-api-sg` with two inbound rules:

   | Type | Port | Source | Why |
   |---|---|---|---|
   | SSH | 22 | My IP | Only my machine can log in |
   | Custom TCP | 8080 | Anywhere (0.0.0.0/0) | PropertyLite listens on 8080, not 80 |

6. **User data (Advanced details):** I pasted the bootstrap script, with the full contents of `app.py` between the `PYEOF` markers (not the placeholder comment from the handbook):
   ```bash
   #!/bin/bash
   yum update -y
   yum install -y python3 python3-pip
   pip3 install flask==3.0.3
   mkdir -p /opt/property-api
   cat > /opt/property-api/app.py << 'PYEOF'
   <full app.py contents from Week 0>
   PYEOF
   cat > /opt/property-api/rets_property_sample.csv << 'CSVEOF'
   L_ListingID,L_City,L_Keyword2,LM_Dec_3,LM_Int2_3,L_SystemPrice,L_Status
   R100234,Sacramento,3,2.0,1450,450000,Active
   R100235,Fresno,4,2.5,1900,395000,Active
   R100236,Sacramento,2,1.0,900,299000,Pending
   CSVEOF
   cd /opt/property-api && nohup python3 app.py > /var/log/property-api.log 2>&1 &
   ```
   User data only runs on the **first boot**, so mistakes in it can't be fixed by editing it on a running instance.

7. **Launch instance.**

## 2. Find the public IP

EC2 → Instances → `property-api-01` → Details tab → **Public IPv4 address**. Or from the CLI:

```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=property-api-01" \
  --query "Reservations[].Instances[].PublicIpAddress" \
  --output text
```

The public IP changes if the instance is stopped and started.

## 3. Connect with SSH

```bash
ssh -i ~/.ssh/training-key.pem ec2-user@<public-ip>
```

The username for Amazon Linux is `ec2-user`. The first connection asks to trust the host; I answered `yes`.

## 4. Verify PropertyLite is running

From my desktop:

```bash
curl http://<public-ip>:8080/health
curl http://<public-ip>:8080/properties/R100234
```

**Expected:**
- `/health` returns `{"status":"ok"}`
- `/properties/R100234` returns the Sacramento listing

**Screenshot:**

![Screenshot of working curl responses](https://i.postimg.cc/TYjbRcbm/curl-responses.png)

## 5. EBS snapshot (Lab 3.2)

EC2 → Volumes → select the root volume of `property-api-01` → Actions → **Create snapshot**. Wait until the status is `completed` under EC2 → Snapshots.

A new volume created from this snapshot would be an exact copy of the root disk as it was when the snapshot was taken, which is how I could restore or clone the instance's disk.

## 6. Cleanup (do this at the end of the week)

1. EC2 → Instances → `property-api-01` → Instance state → **Terminate**.
2. Confirm the root volume disappeared under EC2 → Volumes.
3. Optional: delete the snapshot if I no longer need it (snapshots cost a small amount of storage).

---

## What went wrong, and how I fixed it

| Problem | Cause | Fix |
|---|---|---|
| `curl` to port 8080 failed instantly ("could not connect") while SSH worked | The app wasn't running on the instance. An instant failure means the network path was open but nothing was listening. | Logged in, checked `/opt/property-api/app.py` and the logs, then started the app manually |
| `sudo nohup python3 app.py > /var/log/property-api.log` gave "permission denied" | The `>` redirect runs as my normal user before `sudo` takes effect, and `/var/log` isn't writable by `ec2-user` | Ran without sudo and logged to my home directory: `nohup python3 app.py > ~/property-api.log 2>&1 &` (port 8080 doesn't need root) |

**Lessons:**
- A security-group problem usually shows up as a **timeout**. A quick **"refused"** usually means nothing is listening.
- `ssh -v` output is the fastest way to see how far a connection got.
- Check the real file contents on the instance (`head /opt/property-api/app.py`) instead of assuming user data worked.
- If SSH suddenly times out later, my home IP may have changed. I'd update the SSH rule's source (Debug Lab 3.3).
