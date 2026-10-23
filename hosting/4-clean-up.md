# 4. Cleaning Up the Environment

Delete everything created in this lab so you don't keep paying for it. Running instances, EBS volumes and AMI snapshots all incur charges (some beyond the Free tier).

> **Previous:** [create-ami.md](create-ami.md)

> ⚠️ **These actions are permanent.** Double-check every resource name/ID before deleting, and only delete resources **you** created for this lab.

---

## What to delete (in this order)

| # | Resource | Name / identifier | Created in |
|---|----------|-------------------|------------|
| 1 | EC2 instances | `dgcom-web-server`, `dgcom-web-from-ami` | [launching-ec2.md](launching-ec2.md), [create-ami.md](create-ami.md) |
| 2 | AMI | `dgcom-web-ami-v1` | [create-ami.md](create-ami.md) |
| 3 | EBS snapshot (behind the AMI) | tagged `Name = dgcom-web-ami-v1` | [create-ami.md](create-ami.md) |
| 4 | Security group | `dgcom-web-sg` | [launching-ec2.md](launching-ec2.md) |
| 5 | Key pair | `dgcom-key` (+ local `dgcom-key.pem`) | [launching-ec2.md](launching-ec2.md) |

The order matters: a security group can't be deleted while an instance uses it, and a snapshot can't be deleted while an AMI is registered from it.

---

## Step 1 – Terminate the instances

1. **EC2 → Instances**, select `dgcom-web-server` **and** `dgcom-web-from-ami`.
2. **Instance state → Terminate (delete) instance → Terminate**.
3. Wait until **Instance state = `Terminated`**.

> The root EBS volumes are deleted automatically on termination (default *Delete on termination = Yes*). Check **Elastic Block Store → Volumes** to confirm no `available` 8 GiB volumes are left behind.

## Step 2 – Deregister the AMI

1. **Images → AMIs** (filter: **Owned by me**), select `dgcom-web-ami-v1`.
2. Note the **Snapshot ID** on the **Storage** tab – you'll need it in Step 3.
3. **Actions → Deregister AMI**.
   - If shown, tick **Delete associated snapshots** to do Step 3 automatically.
4. Confirm with **Deregister AMI**.

## Step 3 – Delete the snapshot

Deregistering an AMI does **not** always remove its snapshot, and the snapshot is what you pay for.

1. **Elastic Block Store → Snapshots** (filter: **Owned by me**).
2. Select the snapshot from Step 2 (tagged `Name = dgcom-web-ami-v1`).
3. **Actions → Delete snapshot → Delete**.

## Step 4 – Delete the security group

1. **Network & Security → Security Groups**, select `dgcom-web-sg`.
2. **Actions → Delete security groups → Delete**.

> If you get *"has a dependent object"*, an instance or network interface is still using it – make sure Step 1 finished (`Terminated`).
> **Do not** delete the `default` security group.

## Step 5 – Delete the key pair

1. **Network & Security → Key Pairs**, select `dgcom-key` → **Actions → Delete**.
2. Delete the local `dgcom-key.pem` file from your machine.

---

## Optional – Do the same with the AWS CLI

```bash
REGION=us-east-1

# 1. Terminate instances
INSTANCE_IDS=$(aws ec2 describe-instances --region $REGION \
  --filters "Name=tag:Name,Values=dgcom-web-server,dgcom-web-from-ami" \
            "Name=instance-state-name,Values=pending,running,stopping,stopped" \
  --query 'Reservations[].Instances[].InstanceId' --output text)
echo "Terminating: $INSTANCE_IDS"
aws ec2 terminate-instances --region $REGION --instance-ids $INSTANCE_IDS
aws ec2 wait instance-terminated --region $REGION --instance-ids $INSTANCE_IDS

# 2. Deregister AMI (capture its snapshot first)
AMI_ID=$(aws ec2 describe-images --region $REGION --owners self \
  --filters "Name=name,Values=dgcom-web-ami-v1" \
  --query 'Images[0].ImageId' --output text)
SNAP_ID=$(aws ec2 describe-images --region $REGION --image-ids $AMI_ID \
  --query 'Images[0].BlockDeviceMappings[0].Ebs.SnapshotId' --output text)
echo "AMI: $AMI_ID  Snapshot: $SNAP_ID"
aws ec2 deregister-image --region $REGION --image-id $AMI_ID

# 3. Delete snapshot
aws ec2 delete-snapshot --region $REGION --snapshot-id $SNAP_ID

# 4. Delete security group
aws ec2 delete-security-group --region $REGION --group-name dgcom-web-sg

# 5. Delete key pair
aws ec2 delete-key-pair --region $REGION --key-name dgcom-key
rm -f dgcom-key.pem
```

---

## Final verification

Confirm each of these is empty / gone (in the **same region** you used):

- [ ] **Instances** – no `dgcom-*` instances other than `Terminated` (they disappear after ~1 hour)
- [ ] **AMIs** (Owned by me) – no `dgcom-web-ami-v1`
- [ ] **Snapshots** (Owned by me) – no snapshot tagged `dgcom-web-ami-v1`
- [ ] **Volumes** – no leftover `available` volumes from this lab
- [ ] **Security Groups** – no `dgcom-web-sg`
- [ ] **Key Pairs** – no `dgcom-key`

---

## ✅ Check your progress

Share your answers and screenshots with your instructor/team.

1. Screenshot the **Instances** list showing both lab instances as `Terminated`.
2. Screenshot **AMIs** and **Snapshots** (filter *Owned by me*) showing no DGcom resources.
3. What error would you get if you tried to delete `dgcom-web-sg` **before** terminating the instances, and why?
4. **Think about it:** Why isn't deregistering the AMI enough to stop all storage charges?
5. **Reflection:** In one or two sentences, how would you use an AMI to quickly recover or scale this website in the future?
