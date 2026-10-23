# 3. Creating an AMI of the Web Server

An **Amazon Machine Image (AMI)** is a snapshot-backed template of your instance. Creating one from `dgcom-web-server` lets you launch identical, ready-to-serve web servers – Apache installed, enabled and the DGcom site already deployed – in seconds.

> **Previous:** [site-hosting.md](site-hosting.md) · **Next:** [clean-up.md](clean-up.md)

---

## Prerequisites

- The instance from [launching-ec2.md](launching-ec2.md) is **running** and the site works (see [site-hosting.md](site-hosting.md)).
- `httpd` is **enabled** on boot (`sudo systemctl is-enabled httpd` → `enabled`). Otherwise new instances from the AMI won't serve the site until you start Apache manually.

---

## Step 1 – Create the image

1. In the EC2 console go to **Instances** and select `dgcom-web-server`.
2. Choose **Actions → Image and templates → Create image**.
3. Fill in:

| Field | Value |
|-------|-------|
| Image name | `dgcom-web-ami-v1` |
| Image description | `Amazon Linux 2023 + Apache (httpd) + DGcom static site` |
| No reboot | Leave **unchecked** (a reboot gives a consistent file system snapshot) |
| Instance volumes | Keep the default 8 GiB root volume |

## Step 2 – Add tags

Still on the **Create image** page, in the **Tags** section:

1. Select **Tag image and snapshots together** (so the backing EBS snapshot gets the same tags).
2. Click **Add new tag** and add:

| Key | Value |
|-----|-------|
| `Name` | `dgcom-web-ami-v1` |
| `Environment` | `dev` |

3. Click **Create image**.

> The instance will reboot briefly. The website will be unavailable for a minute.

## Step 3 – Wait for the AMI to become available

1. Go to **Images → AMIs** (left menu). Make sure the filter is **Owned by me**.
2. Your AMI shows **Status = `pending`**. Wait until it becomes **`available`** (usually 2–10 minutes).
3. Go to **Elastic Block Store → Snapshots** – you'll see the snapshot backing the AMI, carrying the same tags.

## Step 4 – Verify / add tags after creation

If you forgot the tags, or want to change them:

1. **Images → AMIs** → select `dgcom-web-ami-v1`.
2. Open the **Tags** tab → **Manage tags**.
3. Make sure `Name = dgcom-web-ami-v1` and `Environment = dev` exist → **Save**.

> The **Name** column in the AMI list shows the `Name` *tag*; the **AMI name** column shows the image name you typed in Step 1. They are different fields – set both.

---

## Step 5 – Test the AMI by launching an instance from it

This proves the image works.

1. **Images → AMIs** → select `dgcom-web-ami-v1` → **Launch instance from AMI**.
2. Name: `dgcom-web-from-ami`
3. Instance type: `t2.micro` / `t3.micro`
4. Key pair: `dgcom-key`
5. Network settings → **Select existing security group** → `dgcom-web-sg`
6. **Launch instance**, wait for `2/2 checks passed`.
7. Open `http://<NEW_PUBLIC_IP>` – the DGcom site should load **without any extra setup**.

---

## Optional – Do the same with the AWS CLI

```bash
REGION=us-east-1
INSTANCE_ID=i-xxxxxxxxxxxxxxxxx   # dgcom-web-server

AMI_ID=$(aws ec2 create-image --region $REGION \
  --instance-id $INSTANCE_ID \
  --name "dgcom-web-ami-v1" \
  --description "Amazon Linux 2023 + Apache (httpd) + DGcom static site" \
  --tag-specifications \
    'ResourceType=image,Tags=[{Key=Name,Value=dgcom-web-ami-v1},{Key=Environment,Value=dev}]' \
    'ResourceType=snapshot,Tags=[{Key=Name,Value=dgcom-web-ami-v1},{Key=Environment,Value=dev}]' \
  --query 'ImageId' --output text)

echo $AMI_ID
aws ec2 wait image-available --region $REGION --image-ids $AMI_ID

# Confirm the tags
aws ec2 describe-images --region $REGION --image-ids $AMI_ID \
  --query 'Images[0].{Name:Name,State:State,Tags:Tags}'

# Add or change tags later
aws ec2 create-tags --region $REGION --resources $AMI_ID \
  --tags Key=Name,Value=dgcom-web-ami-v1 Key=Environment,Value=dev
```

---

## ✅ Check your progress

Share your answers and screenshots with your instructor/team.

1. What is your **AMI ID** (`ami-...`) and what is its **Status**?
2. Screenshot the AMI's **Tags** tab showing `Name` and `Environment`.
3. Did the instance launched from the AMI (`dgcom-web-from-ami`) serve the DGcom site immediately? Paste the output of `curl -I http://<NEW_PUBLIC_IP>`.
4. What is the **Snapshot ID** behind your AMI? (Hint: **Images → AMIs → Storage** tab, or **Elastic Block Store → Snapshots**.)
5. **Think about it:** Why do we enable `httpd` *before* creating the AMI? What would happen if we didn't?
6. **Think about it:** Does an AMI cost anything while it isn't being used? (Hint: look at the snapshot.)
