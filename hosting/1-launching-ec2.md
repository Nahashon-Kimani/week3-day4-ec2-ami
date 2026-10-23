# 1. Launching an Amazon Linux 2023 EC2 Instance

This guide walks you through launching an Amazon Linux 2023 (AL2023) EC2 instance that will host the **DGcom** static website. The instance's security group will allow **SSH (22)**, **HTTP (80)** and **HTTPS (443)** from anywhere (`0.0.0.0/0`).

> **Next:** [site-hosting.md](site-hosting.md) → [create-ami.md](create-ami.md) → [clean-up.md](clean-up.md)

---

## Prerequisites

- An AWS account with permission to manage EC2 (instances, security groups, key pairs, AMIs).
- A region selected in the AWS Console (top-right), e.g. `us-east-1`. Use the **same region** for every guide in this folder.
- An SSH client: Terminal (macOS/Linux), or PowerShell / Git Bash (Windows 10+ has `ssh` built in).

---

## Step 1 – Open the Launch wizard

1. Sign in to the AWS Console and open **EC2**.
2. In the left menu choose **Instances → Launch instances**.

## Step 2 – Name and tags

| Field | Value |
|-------|-------|
| Name  | `dgcom-web-server` |

Optionally click **Add additional tags** and add `Environment = dev`.

## Step 3 – Choose the AMI

1. Under **Application and OS Images (Amazon Machine Image)** choose **Quick Start → Amazon Linux**.
2. Select **Amazon Linux 2023 AMI** (labelled *Free tier eligible*).
3. Architecture: **64-bit (x86)**.

## Step 4 – Instance type

Choose **t2.micro** or **t3.micro** (Free tier eligible, depending on region).

## Step 5 – Key pair

1. Click **Create new key pair**.
2. Name: `dgcom-key`
3. Key pair type: **RSA**
4. Private key format: **.pem** (for OpenSSH)
5. Click **Create key pair** – the file `dgcom-key.pem` downloads. **Keep it safe; it cannot be downloaded again.**

## Step 6 – Network settings (security group)

1. Click **Edit** in the **Network settings** panel.
2. Leave the default VPC and subnet; make sure **Auto-assign public IP** is **Enable**.
3. Firewall (security groups): **Create security group**.
   - Security group name: `dgcom-web-sg`
   - Description: `Allow SSH, HTTP and HTTPS`
4. Configure the inbound rules:

| Type  | Protocol | Port | Source type | Source      | Description        |
|-------|----------|------|-------------|-------------|--------------------|
| SSH   | TCP      | 22   | Anywhere    | `0.0.0.0/0` | SSH access         |
| HTTP  | TCP      | 80   | Anywhere    | `0.0.0.0/0` | Web traffic        |
| HTTPS | TCP      | 443  | Anywhere    | `0.0.0.0/0` | Secure web traffic |

> ⚠️ **Security note:** Opening SSH to `0.0.0.0/0` is acceptable for a short lab, but in real environments restrict port 22 to **My IP** or use **EC2 Instance Connect / SSM Session Manager** instead.

## Step 7 – Storage

Keep the default **8 GiB gp3** root volume.

## Step 8 – Launch

1. Review the **Summary** panel on the right.
2. Click **Launch instance**.
3. Click **View all instances** and wait until:
   - **Instance state** = `Running`
   - **Status check** = `2/2 checks passed`
4. Select the instance and copy its **Public IPv4 address** (you'll need it next).

---

## Step 9 – Connect over SSH

### macOS / Linux / Git Bash

```bash
chmod 400 dgcom-key.pem
ssh -i dgcom-key.pem ec2-user@<PUBLIC_IP>
```

### Windows PowerShell

`chmod` is not available, so lock down the key file with `icacls` first:

```powershell
icacls .\dgcom-key.pem /inheritance:r
icacls .\dgcom-key.pem /grant:r "$($env:USERNAME):(R)"
ssh -i .\dgcom-key.pem ec2-user@<PUBLIC_IP>
```

Type `yes` when asked to trust the host fingerprint. You should see the Amazon Linux 2023 banner and a prompt like:

```
[ec2-user@ip-172-31-xx-xx ~]$
```

> **Tip:** The default user for Amazon Linux is **`ec2-user`** (not `ubuntu` or `root`).

Alternatively, select the instance → **Connect → EC2 Instance Connect → Connect** to get a browser-based shell.

---

## Optional – Do the same with the AWS CLI

```bash
# Variables
REGION=us-east-1
KEY_NAME=dgcom-key
SG_NAME=dgcom-web-sg

# Latest Amazon Linux 2023 AMI ID (via public SSM parameter)
AMI_ID=$(aws ssm get-parameter --region $REGION \
  --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --query 'Parameter.Value' --output text)

# Key pair
aws ec2 create-key-pair --region $REGION --key-name $KEY_NAME \
  --query 'KeyMaterial' --output text > $KEY_NAME.pem
chmod 400 $KEY_NAME.pem

# Security group in the default VPC
SG_ID=$(aws ec2 create-security-group --region $REGION \
  --group-name $SG_NAME --description "Allow SSH, HTTP and HTTPS" \
  --query 'GroupId' --output text)

for PORT in 22 80 443; do
  aws ec2 authorize-security-group-ingress --region $REGION \
    --group-id $SG_ID --protocol tcp --port $PORT --cidr 0.0.0.0/0
done

# Launch the instance
aws ec2 run-instances --region $REGION \
  --image-id $AMI_ID --instance-type t2.micro \
  --key-name $KEY_NAME --security-group-ids $SG_ID \
  --associate-public-ip-address \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=dgcom-web-server},{Key=Environment,Value=dev}]'
```

---

## ✅ Check your progress

Answer these before moving on. Share your answers (and screenshots) with your instructor/team.

1. What is the **Instance ID** and **Public IPv4 address** of your instance, and what does the **Status check** column show?
2. Paste the output of this command run **on the instance**:
   ```bash
   cat /etc/os-release | grep PRETTY_NAME
   ```
   *(Expected: `PRETTY_NAME="Amazon Linux 2023..."`)*
3. Take a screenshot of the **Inbound rules** tab of `dgcom-web-sg`. Which three ports are open, and to which source?
4. **Think about it:** Why is opening port 22 to `0.0.0.0/0` risky, and what would you use instead in production?
