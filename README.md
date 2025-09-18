# 🚀 Resilient Web Server with Idle Resource Monitoring

This project builds a **resilient, auto-scaling, load-balanced web server** in AWS with monitoring for idle resources to avoid unnecessary cost.
It ensures:

* Website files are stored in **Amazon S3**.
* **EC2 Auto Scaling Group** heals itself automatically.
* **Application Load Balancer** distributes traffic.
* **Lambda + SNS + CloudWatch** detect and alert idle AWS resources.

---

##  Step 1: Create Infrastructure in `us-east-1 (N. Virginia)`

### 1. S3 Bucket (Website File Storage)

* **Region**: `us-east-1`
* **Bucket Name**: `us-project-bkt`
* **ACL**: Disable
* **Block all public access**: Disable
* **Bucket Versioning**: Disable
* **Encryption**: SSE-S3

Upload your **website file** here:

```
index.html
```

---

### 2. SNS Topic for Upload Alerts

* Create Topic → **Name**: `s3-update-alert`
* Create Subscription → **Protocol**: Email → **Endpoint**: your email → Confirm email.

**Enable Event Notification on S3 Bucket**:

* Event Name: `obj-upload`
* Event Type: `All object create events`
* Send to: `SNS Topic destination` → select `s3-update-alert`.

---

### 3. IAM Role

* **Role Name**: `full-ec2-s3-us`
* **Trusted Entity**: EC2
* **Permissions**:

  * `AmazonS3FullAccess`
  * `CloudWatchFullAccess`

---

### 4. VPC + Subnets + Security Group

* **VPC**: `project-vpc-us` → CIDR: `10.0.0.0/16`
* **Internet Gateway (IGW)**: Attach to VPC.

**Subnets**:

* `project-sn1-us` → `us-east-1a` → `10.0.1.0/24`
* `project-sn2-us` → `us-east-1b` → `10.0.2.0/24`
* `project-sn3-us` → `us-east-1c` → `10.0.3.0/24`
* `project-sn4-us` → `us-east-1d` → `10.0.4.0/24`

**Route Table**:

* `project-rt-us` → Add Route `0.0.0.0/0` → Attach IGW
* Subnet Associations → select all subnets → Associate.

**Security Group**:

* Name: `project-sg-us`
* Allow: `22 (SSH)` + `80 (HTTP)`

---

### 5. EC2 Launch Template

* **Name**: `project-template-us`
* **AMI**: Amazon Linux 2 (N. Virginia AMI)
* **Instance Type**: `t2.medium`
* **Key Pair**: `projectpem`
* **IAM Role**: `full-ec2-s3-us`
* **Security Group**: `project-sg-us`

**User Data Script (Paste in EC2 User Data section):**

```bash
#!/bin/bash
yum update -y
yum install -y httpd awscli
systemctl start httpd
systemctl enable httpd
sleep 5
aws s3 sync s3://us-project-bkt /var/www/html/ --region us-east-1
sleep 5
echo $(hostname) >> /var/www/html/index.html
systemctl restart httpd
```

---

### 6. Auto Scaling Group

* **Name**: `project-asg-us`
* **Launch Template**: `project-template-us`
* **Subnets**: All 4 created in VPC
* **AZ Distribution**: Balanced best effort

**Scaling Config**:

* Desired: `2`
* Minimum: `2`
* Maximum: `4`

---

### 7. Scaling Policy

**SNS Setup for Scaling Alerts**:

* Create Topic → Name: `scale-alert-us`
* Create Subscription → Protocol: Email → Confirm email

**Attach Scaling Policy**:

* ASG → Automatic Scaling → Dynamic Policy Scaling
* **Policy Type**: Target Tracking Scaling
* **Name**: Target tracking scaling
* **Metric Type**: Average CPU Utilization
* **Target Value**: `50%`
* **Instance Warm-up**: `300s`

**ASG Notifications**:

* Go to ASG → Notifications → Create
* Select SNS Topic: `scale-alert-us`
* Event Types: Create

---

### 8. Load Balancer

* Go to **EC2 → Load Balancers → Create Application Load Balancer**
* **Name**: `project-ALB-us`
* **Scheme**: Internet-facing
* **IP type**: IPv4
* **VPC**: `project-vpc-us`
* **Subnets**: All 4 public subnets
* **Security Group**: `project-sg-us`

**Target Group**:

* **Type**: Instance
* **Name**: `project-LB-TG-us`
* **Protocol**: HTTP, Port: `80`
* Health Check Path: `/index.html`

**Attach Target Group to ALB**:

* Listener: HTTP (80) → Forward to `project-LB-TG-us`

**Connect ALB with ASG**:

* ASG → Edit → Add ALB Target Group → Select `project-LB-TG-us`

**Verify**:

* Go to EC2 → Load Balancers → Copy DNS name → Paste in browser.

---

## Idle Resource Monitoring (Cost Guard)

### Step 1: SNS Topic

* Create Topic → Name: `cost-alerts-us`
* Create Subscription → Protocol: Email → Confirm email.

---

### Step 2: IAM Role for Lambda

* **Role Name**: `lambda-cost-monitor-role`
* **Trusted Entity**: Lambda
* **Policies**:

  * `AmazonEC2ReadOnlyAccess`
  * `AmazonS3ReadOnlyAccess`
  * `ElasticLoadBalancingReadOnly`
  * `AmazonSNSFullAccess`
  * `CloudWatchLogsFullAccess`

---

### Step 3: Lambda Function

* **Name**: `idle-resource-monitor`
* **Runtime**: Python 3.12
* **Execution Role**: `lambda-cost-monitor-role`

---

### Step 4: Paste Code in Lambda

(Delete default code → paste below)

```python
import boto3

# -----------------------------
# AWS clients in us-east-1
# -----------------------------
REGION = 'us-east-1'
sns = boto3.client('sns', region_name=REGION)
ec2 = boto3.client('ec2', region_name=REGION)
elbv2 = boto3.client('elbv2', region_name=REGION)
s3 = boto3.client('s3', region_name=REGION)

# -----------------------------
# Replace with your SNS Topic ARN
# -----------------------------
SNS_TOPIC_ARN = "arn:aws:sns:us-east-1:011587325718:cost-alerts"

def lambda_handler(event, context):
    report = []

    # 1. Stopped EC2 Instances
    instances = ec2.describe_instances(
        Filters=[{'Name': 'instance-state-name', 'Values': ['stopped']}]
    )
    for r in instances['Reservations']:
        for i in r['Instances']:
            report.append(f"Stopped EC2 Instance: {i['InstanceId']}")

    # 2. Unattached Elastic IPs
    addresses = ec2.describe_addresses()['Addresses']
    for addr in addresses:
        if 'InstanceId' not in addr:
            report.append(f"Unattached Elastic IP: {addr['PublicIp']}")

    # 3. NAT Gateways
    nat_gws = ec2.describe_nat_gateways(
        Filters=[{'Name':'state','Values':['available']}]
    )['NatGateways']
    for gw in nat_gws:
        report.append(f"NAT Gateway Active: {gw['NatGatewayId']} (Costly)")

    # 4. ALBs with no targets
    load_balancers = elbv2.describe_load_balancers()['LoadBalancers']
    for lb in load_balancers:
        tgs = elbv2.describe_target_groups(LoadBalancerArn=lb['LoadBalancerArn'])['TargetGroups']
        for tg in tgs:
            health = elbv2.describe_target_health(TargetGroupArn=tg['TargetGroupArn'])
            if not health['TargetHealthDescriptions']:
                report.append(f"ALB with no targets: {lb['LoadBalancerName']}")

    # 5. Empty S3 Buckets
    buckets = s3.list_buckets()['Buckets']
    for b in buckets:
        objects = s3.list_objects_v2(Bucket=b['Name'])
        if 'Contents' not in objects:
            report.append(f"Empty S3 Bucket: {b['Name']}")

    # Send SNS Alert
    if report:
        message = "\n".join(report)
        sns.publish(
            TopicArn=SNS_TOPIC_ARN,
            Subject="Idle AWS Resources Detected",
            Message=message
        )
        return {"status": "alert sent", "details": report}
    else:
        return {"status": "all clear"}
```

Click **Deploy**.

---

### Step 5: Fix Timeout Issue

Default Lambda timeout = 3 sec. Increase it:

* Lambda → Configuration → General → Edit → Timeout = `1 min`.
* Increase Memory = `256 MB`.
* Save.

---

### Step 6: Schedule Lambda with CloudWatch

* Go to EventBridge → Create Rule.
* **Rule Name**: `idle-resource-check`
* **Schedule Expression**: `rate(3 minutes)`
* **Target**: Lambda → `idle-resource-monitor`.
* Create.

 **Now Lambda runs automatically every 3 minutes.**
---

### Step 7: Final Verification

* Stop an EC2 instance or create an empty S3 bucket.
* Wait for schedule or test manually.
* You’ll get an **SNS Email Alert** with details.

---

## 🎯 Final Outcome

* Website served from **S3 + EC2 + Auto Scaling + Load Balancer**
* Infrastructure heals itself and scales.
* Lambda monitors idle resources.
* SNS sends cost-saving alerts.

---

## Badge

![AWS EC2](https://img.shields.io/badge/AWS-EC2-blue?logo=amazon-aws&logoColor=white)
![AWS S3](https://img.shields.io/badge/AWS-S3-orange?logo=amazon-aws&logoColor=white)
![AWS Auto Scaling](https://img.shields.io/badge/AWS-AutoScaling-green?logo=amazon-aws&logoColor=white)
![AWS ALB](https://img.shields.io/badge/AWS-ALB-lightgrey?logo=amazon-aws&logoColor=white)
![AWS Lambda](https://img.shields.io/badge/AWS-Lambda-purple?logo=amazon-aws&logoColor=white)
![AWS SNS](https://img.shields.io/badge/AWS-SNS-yellow?logo=amazon-aws&logoColor=white)
![AWS CloudWatch](https://img.shields.io/badge/AWS-CloudWatch-blueviolet?logo=amazon-aws&logoColor=white)
