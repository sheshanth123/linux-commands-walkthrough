# AWS CLI Setup and EC2 Launch Guide

This guide walks you through authenticating your local terminal with AWS, verifying your connection, and spinning up a free-tier eligible `t3.micro` EC2 instance accessible via SSH.

## 1. Authenticating with AWS CLI

Depending on your AWS environment setup, choose one of the following methods to log in:

### Method A: AWS SSO Login (Recommended for Enterprise/Standard use)
Use this if you need to configure a permanent local profile linked to AWS IAM Identity Center.
1. Run the interactive setup:
   ```bash
   aws configure sso
   ```
2. Follow the prompts to add your start URL, region, and select your target account/role.
3. Log in using your configured profile name:
   ```bash
   aws sso login --profile <your-profile-name>
   ```

### Method B: Unified Interactive Login (AWS CLI v2.32.0+)
Use this for a streamlined, one-step browser authentication (great for AWS Builder IDs).
```bash
aws login
```
*(To end this session later, run `aws logout`)*

### Method C: Static IAM Access Keys
Use this if you are using long-lived access keys instead of temporary SSO credentials.
```bash
aws configure
```
Provide your `AWS Access Key ID`, `AWS Secret Access Key`, default region (e.g., `us-east-1`), and output format (e.g., `json`) when prompted.

---

## 2. Verify Your Login Status

To confirm that your terminal is successfully authenticated and communicating with AWS, run:

```bash
aws sts get-caller-identity
```
* **Success:** Returns a JSON object with your `UserId`, `Account`, and `Arn`.
* **Failure:** Returns an error (e.g., `ExpiredToken`), indicating you need to re-authenticate.

---

## 3. Launching an SSH-Accessible t3.micro EC2 Instance

Once authenticated, run these commands sequentially to launch your free-tier instance.

### Step 3.1: Create and Secure a Key Pair
This generates the SSH key you will use to connect to the server and locks down its permissions.
```bash
aws ec2 create-key-pair --key-name MyDemoKey --query 'KeyMaterial' --output text > MyDemoKey.pem
chmod 400 MyDemoKey.pem
```

### Step 3.2: Create a Security Group
Create a virtual firewall to govern network traffic to your instance.
```bash
aws ec2 create-security-group --group-name AllowSSHGroup --description "Allow SSH inbound traffic"
```

### Step 3.3: Authorize Inbound SSH Traffic
Open Port 22. 
*Security Note: For best practices, replace `0.0.0.0/0` with your actual public IP address (e.g., `203.0.113.25/32`) so only your machine can connect.*
```bash
aws ec2 authorize-security-group-ingress --group-name AllowSSHGroup --protocol tcp --port 22 --cidr 0.0.0.0/0
```

### Step 3.4: Launch the Instance
Spin up the `t3.micro` instance using the latest Amazon Linux 2023 image.
```bash
aws ec2 run-instances \
    --image-id resolve:ssm:/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-6.1-x86_64 \
    --count 1 \
    --instance-type t3.micro \
    --key-name MyDemoKey \
    --security-groups AllowSSHGroup
```

### Step 3.5: Retrieve the Public IP Address
Wait about 30-60 seconds for the instance to boot, then run this to get its IP address:
```bash
aws ec2 describe-instances \
    --filters "Name=instance-state-name,Values=running" "Name=key-name,Values=MyDemoKey" \
    --query "Reservations[*].Instances[*].PublicIpAddress" \
    --output text
```

---

## 4. Connect to Your Instance

Using the IP address retrieved in the previous step, connect to your new Linux environment:

```bash
ssh -i "MyDemoKey.pem" ec2-user@<Public-IP-Address>
```
*(Type `yes` if prompted to accept the SSH fingerprint).*

**Important Cleanup:** When your session is over, remember to terminate your instance to avoid accidental charges if you leave it running past your monthly free tier limits:
```bash
aws ec2 terminate-instances --instance-ids <your-instance-id>
```