# MLflow on EC2 + S3 — Setup Guide

> **TL;DR:** Run MLflow Tracking Server on an EC2 instance backed by PostgreSQL (metadata)
> and S3 (artifacts). Every developer and CI job points `MLFLOW_TRACKING_URI` at the
> EC2 public IP. One shared source of truth for all runs.

---

## 1. Architecture

```mermaid
flowchart TB
    subgraph Local["💻 Developer / CI"]
        CODE["train.py\nMLFLOW_TRACKING_URI=http://ec2-ip:5000"]
    end

    subgraph EC2["🖥️ EC2 Instance (mlflow-server)"]
        SERVER["mlflow server\n--host 0.0.0.0 --port 5000"]
        PG["PostgreSQL\n(run metadata, params, metrics)"]
        SERVER <--> PG
    end

    subgraph AWS["☁️ AWS"]
        S3["S3 Bucket\nmlflow-artifacts-<account-id>\n(model files, plots, artifacts)"]
        IAM["IAM Role\nattached to EC2\nS3 read/write"]
    end

    CODE -- "HTTP log_param / log_metric" --> SERVER
    SERVER -- "artifact upload/download" --> S3
    EC2 -. "assumes" .-> IAM
    IAM -. "grants" .-> S3
```

**Why PostgreSQL + S3 instead of SQLite + local disk?**

| SQLite + local disk | PostgreSQL + S3 |
|---|---|
| Data lost if EC2 terminates | Persistent, survives instance replacement |
| No concurrent writes | Multiple workers safe |
| Artifacts not shareable | Any machine can `dvc pull` / serve models |
| No backup story | S3 versioning + RDS snapshots |

---

## 2. Prerequisites

- AWS account with IAM permissions to create EC2, S3, IAM roles
- Local machine: `aws` CLI configured (`aws configure`)
- SSH key pair created in EC2 console (or via CLI)

---

## 3. Step 1 — Create S3 Bucket

```bash
# pick a globally unique name
BUCKET_NAME="mlflow-artifacts-$(aws sts get-caller-identity --query Account --output text)"
REGION="us-east-1"

# create bucket
aws s3api create-bucket \
  --bucket "$BUCKET_NAME" \
  --region "$REGION"

# enable versioning (recover overwritten artifacts)
aws s3api put-bucket-versioning \
  --bucket "$BUCKET_NAME" \
  --versioning-configuration Status=Enabled

# block all public access
aws s3api put-public-access-block \
  --bucket "$BUCKET_NAME" \
  --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

echo "Bucket: s3://$BUCKET_NAME"
```

---

## 4. Step 2 — Create IAM Role for EC2

The EC2 instance needs permission to read/write S3. Use an IAM role — no hardcoded keys.

```bash
# trust policy: allow EC2 to assume this role
cat > ec2-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "ec2.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
EOF

# create the role
aws iam create-role \
  --role-name mlflow-ec2-role \
  --assume-role-policy-document file://ec2-trust-policy.json

# s3 permissions policy
cat > mlflow-s3-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "s3:GetObject",
      "s3:PutObject",
      "s3:DeleteObject",
      "s3:ListBucket"
    ],
    "Resource": [
      "arn:aws:s3:::$BUCKET_NAME",
      "arn:aws:s3:::$BUCKET_NAME/*"
    ]
  }]
}
EOF

# attach inline policy
aws iam put-role-policy \
  --role-name mlflow-ec2-role \
  --policy-name mlflow-s3-access \
  --policy-document file://mlflow-s3-policy.json

# create instance profile and attach role
aws iam create-instance-profile --instance-profile-name mlflow-ec2-profile
aws iam add-role-to-instance-profile \
  --instance-profile-name mlflow-ec2-profile \
  --role-name mlflow-ec2-role
```

---

## 5. Step 3 — Launch EC2 Instance

```bash
# get latest Amazon Linux 2023 AMI
AMI_ID=$(aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=al2023-ami-2023*-x86_64" \
            "Name=state,Values=available" \
  --query "sort_by(Images, &CreationDate)[-1].ImageId" \
  --output text)

echo "AMI: $AMI_ID"

# launch instance
INSTANCE_ID=$(aws ec2 run-instances \
  --image-id "$AMI_ID" \
  --instance-type t3.small \
  --key-name YOUR_KEY_PAIR_NAME \
  --iam-instance-profile Name=mlflow-ec2-profile \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=mlflow-server}]' \
  --query "Instances[0].InstanceId" \
  --output text)

echo "Instance: $INSTANCE_ID"

# wait until running
aws ec2 wait instance-running --instance-ids "$INSTANCE_ID"

# get public IP
PUBLIC_IP=$(aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query "Reservations[0].Instances[0].PublicIpAddress" \
  --output text)

echo "Public IP: $PUBLIC_IP"
```

### Instance sizing guide

| Traffic | Instance type | Notes |
|---|---|---|
| Solo / small team | `t3.small` | 2 vCPU, 2 GB RAM |
| Team of 10–20 | `t3.medium` | 2 vCPU, 4 GB RAM |
| Heavy parallel training | `t3.large` | 2 vCPU, 8 GB RAM |
| Production | `m5.large` + RDS | Dedicated DB, stable performance |

---

## 6. Step 4 — Security Group

```bash
# create security group
SG_ID=$(aws ec2 create-security-group \
  --group-name mlflow-sg \
  --description "MLflow tracking server" \
  --query "GroupId" \
  --output text)

# allow SSH from your IP only
MY_IP=$(curl -s https://checkip.amazonaws.com)
aws ec2 authorize-security-group-ingress \
  --group-id "$SG_ID" \
  --protocol tcp --port 22 --cidr "$MY_IP/32"

# allow MLflow UI/API — restrict to your office IP or VPN CIDR in production
aws ec2 authorize-security-group-ingress \
  --group-id "$SG_ID" \
  --protocol tcp --port 5000 --cidr "$MY_IP/32"

# attach to instance
aws ec2 modify-instance-attribute \
  --instance-id "$INSTANCE_ID" \
  --groups "$SG_ID"
```

> **Security note:** Never open port 5000 to `0.0.0.0/0` in production.
> MLflow has no built-in authentication. Use VPN, SSH tunnel, or put Nginx + OAuth2 proxy in front.

---

## 7. Step 5 — Install MLflow on EC2

SSH into the instance:

```bash
ssh -i ~/.ssh/YOUR_KEY.pem ec2-user@$PUBLIC_IP
```

Run all of the following on the EC2 instance:

```bash
# update and install dependencies
sudo dnf update -y
sudo dnf install -y python3.11 python3.11-pip postgresql15-server gcc

# install mlflow and drivers
pip3.11 install --user mlflow psycopg2-binary boto3

# add local bin to PATH
echo 'export PATH=$HOME/.local/bin:$PATH' >> ~/.bashrc
source ~/.bashrc

# verify
mlflow --version
```

---

## 8. Step 6 — Set Up PostgreSQL (metadata store)

```bash
# initialize and start postgres
sudo postgresql-setup --initdb
sudo systemctl enable postgresql
sudo systemctl start postgresql

# create mlflow database and user
sudo -u postgres psql << 'EOF'
CREATE USER mlflow_user WITH PASSWORD 'choose_strong_password_here';
CREATE DATABASE mlflow_db OWNER mlflow_user;
GRANT ALL PRIVILEGES ON DATABASE mlflow_db TO mlflow_user;
EOF

# allow password auth — edit pg_hba.conf
sudo sed -i 's/^local\s\+all\s\+all\s\+peer/local   all             all                                     md5/' \
  /var/lib/pgsql/data/pg_hba.conf

sudo systemctl restart postgresql

# test connection
psql -U mlflow_user -d mlflow_db -h localhost -c "SELECT version();"
```

---

## 9. Step 7 — Run MLflow Server

```bash
# set your bucket name (replace with actual bucket)
BUCKET_NAME="mlflow-artifacts-123456789012"
REGION="us-east-1"

# start server manually first to test
mlflow server \
  --backend-store-uri postgresql://mlflow_user:choose_strong_password_here@localhost/mlflow_db \
  --default-artifact-root s3://$BUCKET_NAME/mlflow/ \
  --host 0.0.0.0 \
  --port 5000 \
  --workers 2
```

Test from local machine:

```bash
curl http://$PUBLIC_IP:5000/api/2.0/mlflow/experiments/list
# should return {"experiments": [...]}
```

---

## 10. Step 8 — Run as a systemd Service

Keep MLflow running after SSH disconnect and auto-restart on crash:

```bash
# create environment file (never hardcode creds in service file)
sudo mkdir -p /etc/mlflow
sudo tee /etc/mlflow/env << EOF
MLFLOW_BACKEND_STORE_URI=postgresql://mlflow_user:choose_strong_password_here@localhost/mlflow_db
MLFLOW_DEFAULT_ARTIFACT_ROOT=s3://mlflow-artifacts-123456789012/mlflow/
AWS_DEFAULT_REGION=us-east-1
EOF
sudo chmod 600 /etc/mlflow/env

# create systemd unit
sudo tee /etc/systemd/system/mlflow.service << 'EOF'
[Unit]
Description=MLflow Tracking Server
After=network.target postgresql.service
Requires=postgresql.service

[Service]
Type=simple
User=ec2-user
EnvironmentFile=/etc/mlflow/env
ExecStart=/home/ec2-user/.local/bin/mlflow server \
  --backend-store-uri ${MLFLOW_BACKEND_STORE_URI} \
  --default-artifact-root ${MLFLOW_DEFAULT_ARTIFACT_ROOT} \
  --host 0.0.0.0 \
  --port 5000 \
  --workers 2
Restart=on-failure
RestartSec=5
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
EOF

# enable and start
sudo systemctl daemon-reload
sudo systemctl enable mlflow
sudo systemctl start mlflow

# check status
sudo systemctl status mlflow
sudo journalctl -u mlflow -f    # live logs
```

---

## 11. Step 9 — Connect from Local Machine

```bash
# option A: direct connection (only if port 5000 open in SG)
export MLFLOW_TRACKING_URI=http://$PUBLIC_IP:5000

# option B: SSH tunnel (recommended — no port 5000 open in SG)
ssh -i ~/.ssh/YOUR_KEY.pem -L 5000:localhost:5000 ec2-user@$PUBLIC_IP -N &
export MLFLOW_TRACKING_URI=http://localhost:5000
```

Add to your shell profile or `.env` file:

```bash
# .env or ~/.zshrc
export MLFLOW_TRACKING_URI=http://YOUR_EC2_IP:5000
# if bucket is not public, boto3 uses your local AWS credentials for artifact download
export AWS_PROFILE=your-profile   # or set AWS_ACCESS_KEY_ID + AWS_SECRET_ACCESS_KEY
```

Test from Python:

```python
import mlflow

mlflow.set_tracking_uri("http://YOUR_EC2_IP:5000")
mlflow.set_experiment("connection-test")

with mlflow.start_run():
    mlflow.log_param("test", True)
    mlflow.log_metric("value", 1.0)

print("Connected! Check http://YOUR_EC2_IP:5000")
```

---

## 12. Step 10 — Artifact Access for All Users

Developers need AWS credentials to upload/download artifacts (they go directly to S3,
not through the MLflow server).

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant Server as MLflow Server (EC2)
    participant S3

    Dev->>Server: log_metric("f1", 0.87)
    Server->>Server: store in PostgreSQL

    Dev->>S3: mlflow.sklearn.log_model(model)
    Note over Dev,S3: artifact upload goes direct from dev → S3
    S3-->>Dev: upload confirmed

    Dev->>Server: tell server artifact is at s3://bucket/run_id/model
    Server->>Server: store artifact path in PostgreSQL
```

Each developer needs an IAM user or role with S3 access:

```bash
# create IAM policy for developers
cat > dev-mlflow-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:PutObject", "s3:ListBucket"],
    "Resource": [
      "arn:aws:s3:::$BUCKET_NAME",
      "arn:aws:s3:::$BUCKET_NAME/*"
    ]
  }]
}
EOF

aws iam create-policy \
  --policy-name mlflow-developer-s3 \
  --policy-document file://dev-mlflow-policy.json

# attach to a developer IAM user
aws iam attach-user-policy \
  --user-name rafat \
  --policy-arn arn:aws:iam::ACCOUNT_ID:policy/mlflow-developer-s3
```

---

## 13. Full Setup Verification Checklist

```bash
# on EC2 — all should pass
sudo systemctl is-active mlflow          # → active
sudo systemctl is-active postgresql      # → active
aws s3 ls s3://$BUCKET_NAME/             # → lists (or empty, no error)
curl localhost:5000/health               # → {"status": "OK"}

# from local machine
curl http://$PUBLIC_IP:5000/health       # → {"status": "OK"}
python -c "
import mlflow
mlflow.set_tracking_uri('http://$PUBLIC_IP:5000')
print(mlflow.get_tracking_uri())
exps = mlflow.search_experiments()
print(f'Experiments: {len(exps)}')
"
```

---

## 14. Teardown (Save Costs)

```bash
# stop instance (data preserved, no compute charge)
aws ec2 stop-instances --instance-ids $INSTANCE_ID

# start again later
aws ec2 start-instances --instance-ids $INSTANCE_ID

# terminate permanently (data in S3 and RDS persists)
aws ec2 terminate-instances --instance-ids $INSTANCE_ID
```

---

## 15. Key Takeaways

1. **EC2 + PostgreSQL + S3** = persistent, team-shared MLflow backend.
2. **IAM role on EC2** — no hardcoded credentials; instance inherits S3 access automatically.
3. **Developers use their own AWS creds** for artifact upload/download — traffic bypasses EC2.
4. **systemd service** — MLflow survives SSH disconnect, auto-restarts on crash.
5. **Never open port 5000 to 0.0.0.0/0** — use SSH tunnel or VPN.
6. `MLFLOW_TRACKING_URI` env var — one-line config change points any script at the server.
