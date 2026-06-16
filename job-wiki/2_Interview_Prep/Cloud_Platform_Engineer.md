# 🎯 AVP Cloud Platform Engineer — Complete 2-Day Study Guide
> Barclays Interview Prep | All Topics From Scratch
> Format for every topic: Simple Analogy → Technical → Code → Interview Answer

---

## 📌 YOUR 2-DAY PLAN

```
DAY 1 — Infrastructure & Cloud
  Morning:   Topic 1 — Linux
  Afternoon: Topic 2 — Python (AI/ML focus)
  Evening:   Topic 3 — AWS Deep Dive

DAY 2 — AI, Orchestration & Platform
  Morning:   Topic 4 — Kubernetes + OpenShift
  Afternoon: Topic 5 — LangChain + LangGraph + Agentic AI
  Evening:   Topic 6 — Terraform + CI/CD + GitOps
             Topic 7 — ML & Fine-Tuning (LoRA, QLoRA)
             Topic 8 — Security & Observability
             Topic 9 — Quick Reference Cheatsheet
```

---

# 🐧 TOPIC 1 — LINUX

## 🍎 The Simple Analogy

Imagine your computer is a **big house**.

- **Linux** is the house's foundation, plumbing, and electricity — everything that makes the house work
- **Your application** (Node.js server, Python script) is furniture living inside the house
- **You** are the house manager — you need to know how to fix pipes, check electricity, and manage rooms
- When something breaks at 3 AM, you open the **basement (terminal)** and fix it with commands

As an AVP at Barclays, you won't build Linux — but when a production server slows down or crashes, you need to **diagnose and fix it fast**.

---

## 🔧 What You Must Know

### File System — The House Layout

```
/           → Root (the whole house)
/home       → Where users live (your room)
/etc        → Configuration files (house manual)
/var/log    → Log files (diary of everything that happened)
/tmp        → Temporary files (sticky notes — gone on restart)
/usr        → Programs and tools (your toolbox)
/proc       → Running processes info (live status board)
/etc/hosts  → Maps hostnames to IPs (your address book)
```

### File Permissions — Who Can Enter Which Room

```bash
# See permissions
ls -la /etc/nginx/nginx.conf
# Output: -rw-r--r-- 1 root root 1234 Jun 08 nginx.conf
#          ↑ ↑↑↑ ↑↑↑ ↑↑↑
#          | |   |   |
#          | |   |   others can only READ
#          | |   group can only READ
#          | owner can READ and WRITE
#          file type (- = file, d = directory)

# Change permissions
chmod 755 script.sh    # owner: rwx, group: r-x, others: r-x
chmod +x script.sh     # add execute permission for everyone
chown ubuntu:ubuntu file.txt  # change owner:group

# Analogy: 755 means owner has master key, others have guest key
```

### Process Management — Managing Workers in the House

```bash
# See all running processes
ps aux
ps aux | grep python    # find python processes
ps aux | grep node      # find node processes

# Real-time process monitor (like Task Manager)
top                     # basic
htop                    # prettier (install with apt-get install htop)

# Kill a process
kill 1234               # graceful stop (send SIGTERM — "please stop")
kill -9 1234            # force kill (send SIGKILL — "stop NOW")
pkill python            # kill all processes named python

# Check what's using a port
lsof -i :3000           # what's on port 3000?
ss -tulpn | grep 3000   # alternative

# Analogy: kill is like asking a worker to stop, kill -9 is pulling the power plug
```

### Networking — The House's Phone System

```bash
# Check network connections
netstat -tulpn          # all listening ports
ss -tulpn               # modern alternative (faster)

# Test connectivity
ping google.com         # is the internet working?
curl -I https://api.company.com  # test HTTP endpoint
curl -v https://api.company.com  # verbose — see headers

# DNS lookup
dig api.company.com     # what IP does this hostname resolve to?
nslookup api.company.com  # alternative

# Download files
wget https://example.com/file.zip
curl -O https://example.com/file.zip

# Network interface info
ip addr show            # see all IPs on this machine
ifconfig                # older alternative

# Analogy: ping is like calling someone to see if they pick up
#          dig is like asking "what's the address of this person?"
```

### Log Management — Reading the House Diary

```bash
# View logs
tail -f /var/log/syslog           # watch live (like watching diary being written)
tail -100 /var/log/app.log        # last 100 lines
cat /var/log/app.log | grep ERROR # find errors
journalctl -u nginx               # logs for nginx service
journalctl -f                     # follow all system logs
journalctl --since "1 hour ago"   # last hour of logs

# Search in logs
grep "ERROR" /var/log/app.log
grep -i "error" /var/log/app.log  # case insensitive
grep -n "error" file.log          # show line numbers
grep -A 5 "ERROR" file.log        # show 5 lines AFTER each match
grep -B 3 "ERROR" file.log        # show 3 lines BEFORE each match

# Count occurrences
grep -c "ERROR" /var/log/app.log  # how many errors?

# Analogy: grep is like a search engine for your log files
```

### Bash Scripting — Automating House Tasks

```bash
#!/bin/bash
# This line MUST be first — tells system this is a bash script

# Variables
APP_NAME="attendance-api"
PORT=3000
LOG_FILE="/var/log/${APP_NAME}.log"

# If statement
if [ $PORT -eq 3000 ]; then
    echo "Running on default port"
elif [ $PORT -eq 8080 ]; then
    echo "Running on alternative port"
else
    echo "Running on custom port: $PORT"
fi

# Loop
SERVICES=("nginx" "node" "redis")
for SERVICE in "${SERVICES[@]}"; do
    if systemctl is-active --quiet $SERVICE; then
        echo "$SERVICE is running ✅"
    else
        echo "$SERVICE is DOWN ❌"
        systemctl start $SERVICE
    fi
done

# Function
check_port() {
    local port=$1
    if ss -tulpn | grep ":$port" > /dev/null; then
        echo "Port $port is in use"
        return 0
    else
        echo "Port $port is free"
        return 1
    fi
}

check_port 3000
check_port 8080

# Error handling
set -e  # exit on any error
set -u  # exit if variable undefined
set -o pipefail  # exit if any pipe command fails

# Read file line by line
while IFS= read -r line; do
    echo "Processing: $line"
done < servers.txt

# Command substitution
CURRENT_DATE=$(date +%Y-%m-%d)
DISK_USAGE=$(df -h / | awk 'NR==2 {print $5}')
echo "Disk usage on $CURRENT_DATE: $DISK_USAGE"
```

### Performance Troubleshooting — Diagnosing a Sick House

```bash
# CPU usage
top -bn1 | grep "Cpu(s)"     # one-time CPU snapshot
vmstat 1 5                    # system stats every 1 sec, 5 times

# Memory usage
free -h                       # human readable memory usage
# Output shows: total, used, free, available
# "available" is what matters — not "free"

# Disk usage
df -h                         # disk space per partition
du -sh /var/log/*             # size of each log directory
du -sh /home/*                # size of each user's home

# I/O performance
iostat -x 1                   # disk I/O stats
iotop                         # which process is hammering disk

# Find large files (useful when disk is full)
find / -size +100M -type f 2>/dev/null | sort -rh | head -20

# System calls (advanced debugging)
strace -p 1234                # trace system calls of process 1234
strace python script.py       # trace from start

# Network performance
iperf3 -s                     # start server
iperf3 -c server_ip           # run bandwidth test
```

---

## 💬 Interview Answer

**Q: "We have a production server that's responding slowly. Walk me through how you'd diagnose it."**

> *"I'd follow a systematic top-down approach. First — check if it's the server itself: top to see CPU and memory, free -h for memory pressure, df -h to rule out full disk. If CPU is high, ps aux sorts by CPU to find the culprit process. If memory is exhausted — check for memory leaks, consider adding swap or scaling horizontally.*
>
> *Next — check the application layer: tail -f the application logs looking for errors, timeouts, or unusual patterns. grep for ERROR and WARN in the last hour.*
>
> *Then — check the network: ss -tulpn to verify services are listening, netstat for connection states — too many TIME_WAIT or CLOSE_WAIT often indicates connection pool exhaustion.*
>
> *For disk I/O: iostat -x to see if disk is saturated, iotop to find which process is causing it.*
>
> *Finally — check external dependencies: are DB connections timing out? Is a downstream API slow? I'd check logs for timeout errors and use curl to test response times directly.*
>
> *The key is eliminating layers systematically — CPU, memory, disk, network, application, dependencies — rather than guessing."*

---

# 🐍 TOPIC 2 — PYTHON (AI/ML Focus)

## 🍎 The Simple Analogy

Python is like a **Swiss Army knife**.
- JavaScript (your main tool) is like a hammer — great for one thing (web)
- Python is a Swiss Army knife — it has a blade (web), scissors (data), screwdriver (automation), and a file (AI/ML)
- For AI work, Python is the **only real choice** — all AI frameworks (TensorFlow, PyTorch, Hugging Face) are Python-first
- You need to be comfortable enough to write Python services, not just scripts

---

## 🔧 Python Fundamentals You Need

### Type Hints — Making Python Safer

```python
# Without type hints (like JavaScript without TypeScript)
def process_employee(employee, include_salary):
    return employee["name"]

# With type hints (like TypeScript — much better)
from typing import Optional, List, Dict, Any
from pydantic import BaseModel

def process_employee(
    employee: Dict[str, Any],
    include_salary: bool = False
) -> str:
    return employee["name"]

# Even better — use Pydantic models (like Zod in TypeScript)
class Employee(BaseModel):
    id: int
    name: str
    email: str
    salary: Optional[float] = None
    department: str
    is_active: bool = True

# Pydantic validates automatically
emp = Employee(id=1, name="Abhishek", email="abhi@company.com", department="Tech")
print(emp.name)  # "Abhishek"
print(emp.model_dump())  # convert to dict
print(emp.model_dump_json())  # convert to JSON string

# If you pass wrong type — Pydantic raises ValidationError
emp2 = Employee(id="not-a-number", name=123)  # raises error
```

### Async Python — Like Node.js but Different

```python
# In Node.js you know: async/await, Promises
# Python async works similarly but has some differences

import asyncio
import aiohttp  # async HTTP client (like axios for Python)

# Basic async function
async def fetch_employee(session: aiohttp.ClientSession, emp_id: int) -> dict:
    async with session.get(f"https://api.company.com/employees/{emp_id}") as response:
        return await response.json()

# Run multiple requests in parallel (like Promise.all in JS)
async def fetch_all_employees(emp_ids: list[int]) -> list[dict]:
    async with aiohttp.ClientSession() as session:
        # Create all tasks
        tasks = [fetch_employee(session, emp_id) for emp_id in emp_ids]
        # Run all in parallel
        results = await asyncio.gather(*tasks)
        return results

# FastAPI — Python's Express.js (most common for AI services)
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

app = FastAPI()

class CheckInRequest(BaseModel):
    employee_id: str
    location_id: str
    timestamp: str

class CheckInResponse(BaseModel):
    job_id: str
    status: str
    message: str

@app.post("/api/attendance/checkin", response_model=CheckInResponse)
async def checkin(request: CheckInRequest):
    if not request.employee_id:
        raise HTTPException(status_code=400, detail="Employee ID required")

    job_id = await process_checkin(request)
    return CheckInResponse(
        job_id=job_id,
        status="queued",
        message="Check-in processing"
    )

@app.get("/health")
async def health():
    return {"status": "ok"}

# Run: uvicorn main:app --reload
```

### Decorators — Wrapping Functions

```python
# Analogy: A decorator is like a security guard at a door
# Before you enter the function (room), the guard does something
# After you leave, the guard might do something else

import time
import functools
import logging

# Simple decorator — logs every function call
def log_function_call(func):
    @functools.wraps(func)  # preserves original function name
    def wrapper(*args, **kwargs):
        logging.info(f"Calling {func.__name__} with args={args}")
        start_time = time.time()

        result = func(*args, **kwargs)  # call the actual function

        duration = time.time() - start_time
        logging.info(f"{func.__name__} completed in {duration:.3f}s")
        return result
    return wrapper

# Decorator with parameters
def retry(max_attempts: int = 3, delay: float = 1.0):
    def decorator(func):
        @functools.wraps(func)
        async def wrapper(*args, **kwargs):
            for attempt in range(max_attempts):
                try:
                    return await func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_attempts - 1:
                        raise  # last attempt — raise the error
                    logging.warning(f"Attempt {attempt+1} failed: {e}. Retrying...")
                    await asyncio.sleep(delay * (attempt + 1))  # exponential backoff
        return wrapper
    return decorator

# Use decorators
@log_function_call
def process_attendance(employee_id: str) -> dict:
    return {"status": "processed"}

@retry(max_attempts=3, delay=1.0)
async def call_llm_api(prompt: str) -> str:
    async with aiohttp.ClientSession() as session:
        async with session.post("https://api.anthropic.com/v1/messages",
                               json={"prompt": prompt}) as resp:
            data = await resp.json()
            return data["content"][0]["text"]
```

### Context Managers — Proper Resource Cleanup

```python
# Analogy: Like checking out a library book
# You MUST return it — even if you forget, the library (context manager) ensures it's returned

# You know this pattern in Node.js:
# const connection = await db.connect()
# try { ... } finally { connection.close() }

# Python's equivalent — cleaner:
with open("employees.csv", "r") as file:
    content = file.read()
# File automatically closed even if error occurs

# Database connection
import psycopg2
from contextlib import contextmanager

@contextmanager
def get_db_connection():
    conn = psycopg2.connect("postgresql://localhost/attendance")
    try:
        yield conn  # provide connection to the with block
        conn.commit()  # commit if no error
    except Exception:
        conn.rollback()  # rollback on error
        raise
    finally:
        conn.close()  # ALWAYS close

# Usage
with get_db_connection() as conn:
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM employees WHERE org_id = %s", ("ORG-001",))
    employees = cursor.fetchall()
# Connection automatically closed and committed/rolled back
```

### Generators — Memory-Efficient Processing

```python
# Analogy: Instead of buying ALL the groceries at once (memory expensive),
# you buy one item at a time from the store (memory efficient)

# Process 1 million attendance records without loading all into memory
def read_attendance_records(file_path: str):
    """Generator — yields one record at a time"""
    with open(file_path, "r") as f:
        for line in f:  # reads one line at a time
            record = line.strip().split(",")
            yield {
                "employee_id": record[0],
                "timestamp": record[1],
                "event_type": record[2]
            }
    # Memory used: ONE record at a time, not 1 million

# Usage
for record in read_attendance_records("attendance.csv"):
    process_record(record)  # process one at a time
    # Python's garbage collector frees each record after processing

# Generator expression (like list comprehension but lazy)
active_employees = (emp for emp in employees if emp["is_active"])
# NOT evaluated yet — only when you iterate
```

### AWS SDK (Boto3) — Talking to AWS from Python

```python
import boto3
import json
from botocore.exceptions import ClientError

# Create AWS clients
s3_client = boto3.client('s3', region_name='ap-south-1')
dynamodb = boto3.resource('dynamodb', region_name='ap-south-1')
secrets_client = boto3.client('secretsmanager', region_name='ap-south-1')
lambda_client = boto3.client('lambda', region_name='ap-south-1')

# S3 — Upload and download files
def upload_attendance_report(file_path: str, bucket: str, key: str) -> str:
    try:
        s3_client.upload_file(file_path, bucket, key)
        # Generate pre-signed URL for download
        url = s3_client.generate_presigned_url(
            'get_object',
            Params={'Bucket': bucket, 'Key': key},
            ExpiresIn=3600  # 1 hour
        )
        return url
    except ClientError as e:
        print(f"Error uploading to S3: {e}")
        raise

# Get secret from AWS Secrets Manager
def get_secret(secret_name: str) -> dict:
    try:
        response = secrets_client.get_secret_value(SecretId=secret_name)
        return json.loads(response['SecretString'])
    except ClientError as e:
        if e.response['Error']['Code'] == 'ResourceNotFoundException':
            raise ValueError(f"Secret {secret_name} not found")
        raise

# Invoke Lambda function
def trigger_payroll_processing(org_id: str, period: str) -> dict:
    payload = {"org_id": org_id, "period": period}
    response = lambda_client.invoke(
        FunctionName='payroll-processor',
        InvocationType='Event',  # async — don't wait for response
        Payload=json.dumps(payload)
    )
    return {"status": "triggered", "status_code": response['StatusCode']}

# List S3 objects with pagination
def list_all_reports(bucket: str, prefix: str) -> list:
    paginator = s3_client.get_paginator('list_objects_v2')
    all_objects = []
    for page in paginator.paginate(Bucket=bucket, Prefix=prefix):
        all_objects.extend(page.get('Contents', []))
    return all_objects
```

---

## 💬 Interview Answer

**Q: "Walk me through how you'd build a Python service to process LLM requests with proper error handling and async support."**

> *"I'd use FastAPI for the service layer — it's async-native and has built-in Pydantic validation. The request model validates inputs automatically, so invalid data never reaches the business logic. For LLM calls, I'd use aiohttp for async HTTP to avoid blocking the event loop. I'd wrap LLM calls in a retry decorator with exponential backoff — LLM APIs are occasionally slow or rate-limited. Context managers ensure database connections and HTTP sessions are always properly closed. For large datasets I'd use generators to process records one at a time without loading everything into memory. Type hints throughout make the code maintainable and catch errors at development time rather than production."*

---

# ☁️ TOPIC 3 — AWS DEEP DIVE

## 🍎 The Simple Analogy

AWS is like a **massive city of services**.
- You don't build your own power plant (servers) — you rent electricity (EC2)
- You don't build your own warehouse (storage) — you rent space (S3)
- The city has traffic police (IAM) that controls who can go where
- The city has roads (VPC/networking) that connect everything
- As an AVP, you need to know the city layout — not just one street

---

## 🔧 IAM — The City's Security System

```
Analogy: IAM is like an office building's access control system
  - Users     = individual employees with keycards
  - Groups    = departments (Engineering, Finance, HR)
  - Roles     = temporary visitor badges (for applications, not people)
  - Policies  = rules about which doors each keycard can open
```

```python
# IAM Policy — JSON document defining permissions
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",                    # Allow or Deny
            "Action": [
                "s3:GetObject",                   # can download files
                "s3:PutObject"                    # can upload files
            ],
            "Resource": "arn:aws:s3:::attendance-reports/*",  # only this bucket
            "Condition": {
                "StringEquals": {
                    "aws:RequestedRegion": "ap-south-1"  # only from India region
                }
            }
        },
        {
            "Effect": "Deny",                     # Deny overrides Allow
            "Action": "s3:DeleteObject",          # can never delete
            "Resource": "*"
        }
    ]
}
```

```bash
# AWS CLI — talk to AWS from terminal
# Configure credentials
aws configure
# Enter: Access Key, Secret Key, Region, Output format

# Use profiles for different environments
aws configure --profile barclays-dev
aws configure --profile barclays-prod

# Use a profile
aws s3 ls --profile barclays-prod

# Assume a role (switch to another role temporarily)
aws sts assume-role \
    --role-arn "arn:aws:iam::123456789:role/DevOpsRole" \
    --role-session-name "MySession"

# Use output credentials
export AWS_ACCESS_KEY_ID="..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_SESSION_TOKEN="..."

# Useful AWS CLI commands
aws s3 ls s3://my-bucket/                    # list bucket contents
aws s3 cp file.txt s3://my-bucket/file.txt   # upload file
aws s3 sync ./reports s3://my-bucket/reports # sync directory

aws ec2 describe-instances --region ap-south-1  # list EC2 instances
aws ec2 describe-instances \
    --filters "Name=instance-state-name,Values=running"  # only running

aws logs tail /aws/lambda/my-function --follow  # stream Lambda logs

aws secretsmanager get-secret-value \
    --secret-id "prod/database/password"  # get secret

# IAM — check your own permissions
aws iam get-user
aws iam list-attached-user-policies --user-name myuser
```

---

## 🔧 VPC — The City's Road System

```
Analogy: VPC is like a gated community
  - VPC            = the entire gated community (your private network)
  - Subnets        = different streets within the community
  - Public subnet  = street with direct access to main road (internet)
  - Private subnet = street accessible only from inside community
  - Internet Gateway = the main gate to enter/exit the community
  - NAT Gateway    = a secret exit — private street can go OUT but no one can come IN
  - Route Tables   = road signs showing which direction to go
  - Security Groups = locks on each house (control inbound/outbound traffic)
  - NACLs          = street-level checkpoints (control subnet traffic)
```

```
Real setup for attendance system:

VPC: 10.0.0.0/16 (65,536 addresses — your whole community)

Public Subnets (have internet access):
  10.0.1.0/24 - AZ1 Mumbai — Load Balancer lives here
  10.0.2.0/24 - AZ2 Mumbai — Load Balancer backup

Private Subnets (no direct internet):
  10.0.10.0/24 - AZ1 — Application servers (Node.js, Python)
  10.0.11.0/24 - AZ2 — Application servers backup
  10.0.20.0/24 - AZ1 — Database (RDS, ElastiCache)
  10.0.21.0/24 - AZ2 — Database backup

Flow:
  Internet → Internet Gateway → Public Subnet (Load Balancer)
  Load Balancer → Private Subnet (App servers)
  App servers → Private Subnet (Database)
  App servers → NAT Gateway → Internet (for external API calls)
```

---

## 🔧 EKS — Kubernetes on AWS

```
EKS = Elastic Kubernetes Service
Analogy: Like hiring a professional building manager
  - You provide the apartments (your containers/pods)
  - AWS manages the building infrastructure (control plane)
  - You still manage your apartments (worker nodes + your apps)
```

```bash
# Setup EKS
# Install eksctl (EKS CLI)
curl --silent --location \
    "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz" \
    | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# Create EKS cluster
eksctl create cluster \
    --name attendance-cluster \
    --region ap-south-1 \
    --nodegroup-name standard-workers \
    --node-type t3.medium \
    --nodes 3 \
    --nodes-min 2 \
    --nodes-max 10 \
    --managed  # AWS manages node updates

# Configure kubectl to use EKS
aws eks update-kubeconfig \
    --name attendance-cluster \
    --region ap-south-1

# Now kubectl works with EKS
kubectl get nodes
kubectl get pods --all-namespaces
```

---

## 🔧 SageMaker — ML Training on AWS

```
Analogy: SageMaker is like a professional kitchen
  - You bring your recipe (ML code) and ingredients (data)
  - SageMaker provides the kitchen, appliances, and cleaning crew
  - You get the finished dish (trained model) without managing equipment
```

```python
import sagemaker
from sagemaker.huggingface import HuggingFace

# Fine-tune a model on SageMaker
huggingface_estimator = HuggingFace(
    entry_point='train.py',           # your training script
    source_dir='./training_code',     # directory with your code
    instance_type='ml.g4dn.xlarge',   # GPU instance for training
    instance_count=1,
    role=sagemaker.get_execution_role(),
    transformers_version='4.26',
    pytorch_version='1.13',
    py_version='py39',
    hyperparameters={
        'model_name_or_path': 'bert-base-uncased',
        'num_train_epochs': 3,
        'per_device_train_batch_size': 16,
        'learning_rate': 5e-5,
    }
)

# Start training
huggingface_estimator.fit({
    'train': 's3://my-bucket/training-data/',
    'test': 's3://my-bucket/test-data/'
})

# Deploy trained model as endpoint
predictor = huggingface_estimator.deploy(
    initial_instance_count=1,
    instance_type='ml.m5.xlarge'
)

# Call the endpoint
result = predictor.predict({"inputs": "Classify this text: ..."})
```

---

## 💬 Interview Answer

**Q: "How would you design secure AWS infrastructure for a financial application?"**

> *"I'd design it in layers. Network layer: everything inside a VPC with public subnets only for load balancers, application and database in private subnets with no direct internet access. Private subnets use NAT Gateway for outbound calls to external APIs — nothing can reach them inbound directly. IAM layer: least privilege per service — each Lambda, ECS task, or EC2 instance gets its own IAM role with only the permissions it needs. No shared credentials, no hardcoded keys. Applications use IAM roles, not access keys. Data layer: encryption at rest using KMS for all S3 buckets, RDS, and DynamoDB. Encryption in transit — TLS 1.3 everywhere. Secrets in AWS Secrets Manager, never in environment variables or code. Audit layer: CloudTrail enabled in all regions for API audit. Config rules for compliance checking. GuardDuty for threat detection. For Barclays specifically — all of this aligns with FCA operational resilience requirements, and I'd add VPC Flow Logs for network traffic analysis and regular penetration testing cycles."*

---

# ☸️ TOPIC 4 — KUBERNETES + OPENSHIFT

## 🍎 The Simple Analogy

**Kubernetes** is like a **very smart hotel manager**.
- Your application is a guest
- Pods are hotel rooms
- The manager (K8s) decides which room each guest gets
- If a room has a problem, manager moves the guest to a new room automatically
- If more guests arrive, manager opens more rooms (scaling)
- You tell the manager what you want — not HOW to do it (declarative)

**OpenShift** is like the **same hotel but with stricter fire safety rules**.
- Same rooms (pods), same manager (Kubernetes)
- But every room has sprinklers (security), cameras (audit), and fire exits (recovery)
- The hotel is certified for hospitals (banks, financial institutions)
- Some doors have extra locks (Security Context Constraints)

---

## 🔧 Kubernetes RBAC — Who Can Do What

```
Analogy: RBAC is like job roles in a company
  - Role         = a job description (what this role is allowed to do)
  - ClusterRole  = a company-wide job description
  - RoleBinding  = hiring someone for a role (in one department/namespace)
  - ClusterRoleBinding = hiring someone company-wide
  - ServiceAccount = an employee badge for applications (not humans)
```

```yaml
# Role — what can be done in a specific namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: attendance-api-role
  namespace: attendance-prod  # only in this namespace
rules:
  - apiGroups: [""]           # "" means core API group
    resources: ["pods", "services"]
    verbs: ["get", "list", "watch"]  # read-only
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "update"]  # can also update deployments

---
# ServiceAccount — identity for our application
apiVersion: v1
kind: ServiceAccount
metadata:
  name: attendance-api
  namespace: attendance-prod

---
# RoleBinding — give the ServiceAccount the Role
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: attendance-api-binding
  namespace: attendance-prod
subjects:
  - kind: ServiceAccount
    name: attendance-api
    namespace: attendance-prod
roleRef:
  kind: Role
  name: attendance-api-role
  apiGroup: rbac.authorization.k8s.io
```

---

## 🔧 Network Policies — Traffic Rules Between Pods

```
Analogy: Network Policies are like office security doors
  Default: All pods can talk to all other pods (dangerous!)
  With Network Policy: Only specific pods can talk to specific pods

Example: Database pod should ONLY accept connections from API pods
         Not from monitoring pods, not from other services
```

```yaml
# Deny ALL traffic by default in namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: attendance-prod
spec:
  podSelector: {}  # applies to ALL pods in namespace
  policyTypes:
    - Ingress
    - Egress

---
# Allow: API pods can receive traffic from Load Balancer only
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-ingress
  namespace: attendance-prod
spec:
  podSelector:
    matchLabels:
      app: attendance-api  # applies to API pods
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: ingress-nginx  # only from ingress namespace
      ports:
        - protocol: TCP
          port: 3000

---
# Allow: Database accepts ONLY from API pods
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-db-from-api
  namespace: attendance-prod
spec:
  podSelector:
    matchLabels:
      app: postgres  # applies to DB pods
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: attendance-api  # only API pods
      ports:
        - port: 5432
```

---

## 🔧 Resource Requests and Limits — Fair Resource Allocation

```
Analogy: Resource requests/limits are like booking a hotel room
  - Request = minimum guaranteed room size (you're always given at least this)
  - Limit   = maximum room size (you can never take more than this)
  
Without limits: One greedy app can use ALL the server's CPU/memory
                and starve other apps (noisy neighbour problem)
With limits:    Every app gets its fair share, no one can hog resources
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: attendance-api
spec:
  template:
    spec:
      containers:
        - name: attendance-api
          image: attendance-api:v1.2.3
          resources:
            requests:           # Kubernetes GUARANTEES this much
              memory: "256Mi"   # 256 megabytes RAM guaranteed
              cpu: "250m"       # 0.25 CPU cores guaranteed (250 millicores)
            limits:             # Kubernetes NEVER allows more than this
              memory: "512Mi"   # 512 MB max — exceed this = OOM kill
              cpu: "500m"       # 0.5 CPU max — exceed this = throttled

# For AI/LLM service — needs more resources
        - name: llm-service
          image: llm-service:v2.0.0
          resources:
            requests:
              memory: "2Gi"     # 2 GB RAM needed for model
              cpu: "1000m"      # 1 full CPU core
              nvidia.com/gpu: 1 # request 1 GPU!
            limits:
              memory: "4Gi"
              cpu: "2000m"
              nvidia.com/gpu: 1
```

---

## 🔧 OpenShift Specifics — What's Different

```
Kubernetes vs OpenShift Key Differences:

Feature              Kubernetes          OpenShift
─────────────────────────────────────────────────────
Ingress              Ingress resource    Route resource
Security             PodSecurityPolicy   SecurityContextConstraints (SCC)
CLI                  kubectl             oc (superset of kubectl)
Image Registry       External (ECR,GCR)  Built-in registry
CI/CD                External (Jenkins)  Built-in Tekton
User Management      Basic               Full OAuth + LDAP
Namespace            Namespace           Project (namespace + policies)
```

```bash
# OpenShift CLI (oc) — same as kubectl but more features
oc login https://openshift.barclays.com --token=xxx
oc project attendance-prod      # switch to project (like kubectl namespace)
oc get pods                     # same as kubectl get pods
oc get routes                   # OpenShift-specific: see exposed URLs

# Create a route (OpenShift's Ingress)
oc expose service attendance-api
oc get route attendance-api     # see the URL

# Security Context Constraints — OpenShift's strict security
oc get scc                      # list all SCCs
# Common SCCs:
# restricted     → most secure, no root, no host access (default)
# anyuid         → can run as any user ID
# privileged     → can do almost anything (avoid this!)

# Check which SCC a pod is using
oc describe pod my-pod | grep scc

# Add SCC to service account (if your app needs it)
oc adm policy add-scc-to-user anyuid -z my-service-account
```

---

## 💬 Interview Answer

**Q: "What is the difference between Kubernetes and OpenShift, and why would Barclays use OpenShift?"**

> *"Both OpenShift and Kubernetes are container orchestration platforms — OpenShift is actually built ON Kubernetes, it's Red Hat's enterprise distribution. The core concepts are identical: pods, deployments, services, ConfigMaps, secrets. The differences are in what's added on top. OpenShift adds Security Context Constraints which are stricter than Kubernetes' Pod Security Policies — by default no container can run as root, and you need explicit permission to override that. This is critical for a bank where security is regulatory, not optional. OpenShift also has a built-in container registry, built-in CI/CD via Tekton, and integrated OAuth and LDAP for user management — so you're not piecing together separate tools. Routes replace Ingress with HAProxy-based load balancing. For Barclays, the security defaults and enterprise support from Red Hat make OpenShift the right choice — it's auditable, hardened, and supported with SLAs that a financial institution requires."*

---

# 🦜 TOPIC 5 — LANGCHAIN + LANGGRAPH + AGENTIC AI

## 🍎 The Simple Analogy

**LangChain** is like a **Lego set for building AI applications**.
- Each Lego piece is a component: LLM, memory, tools, retrievers
- LangChain gives you pre-built pieces and ways to connect them
- You don't build the AI model — you use existing models (Claude, GPT) and connect them to data and tools

**LangGraph** is like a **traffic management system**.
- Your agents are cars
- LangGraph defines the roads (graph), intersections (nodes), and traffic rules (edges)
- It decides: "After this agent finishes, which agent goes next?"
- It can loop back, branch, pause for human approval

**Agentic AI** means the AI **takes actions**, not just answers questions.
- Non-agentic: "What is the weather?" → AI answers
- Agentic: "Book me a hotel for next week" → AI searches, compares, books, confirms

---

## 🔧 LangChain — Core Concepts

```python
# Install
# pip install langchain langchain-anthropic langchain-community

from langchain_anthropic import ChatAnthropic
from langchain_core.messages import HumanMessage, SystemMessage
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

# 1. Basic LLM call
llm = ChatAnthropic(model="claude-3-5-sonnet-20241022")

messages = [
    SystemMessage(content="You are a helpful HR assistant"),
    HumanMessage(content="What is the leave policy for sick leave?")
]

response = llm.invoke(messages)
print(response.content)

# 2. Prompt Templates — reusable prompts with variables
prompt = ChatPromptTemplate.from_messages([
    ("system", "You are an HR assistant for {company_name}"),
    ("human", "Employee {employee_name} is asking: {question}")
])

# LCEL — LangChain Expression Language (pipe syntax)
chain = prompt | llm | StrOutputParser()

result = chain.invoke({
    "company_name": "Barclays",
    "employee_name": "Abhishek",
    "question": "How many sick days do I have left?"
})
print(result)

# 3. Memory — remember conversation history
from langchain.memory import ConversationBufferMemory
from langchain.chains import ConversationChain

memory = ConversationBufferMemory()
conversation = ConversationChain(llm=llm, memory=memory)

# First message
response1 = conversation.predict(input="My name is Abhishek")
# Second message — AI remembers the name
response2 = conversation.predict(input="What is my name?")
# AI responds: "Your name is Abhishek"
```

### RAG with LangChain — Connect AI to Your Documents

```python
from langchain_community.document_loaders import PyPDFLoader, TextLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_anthropic import AnthropicEmbeddings
from langchain_community.vectorstores import Chroma
from langchain.chains import RetrievalQA

# Step 1: Load documents
loader = PyPDFLoader("hr_policy.pdf")
documents = loader.load()

# Step 2: Split into chunks
splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,      # 1000 characters per chunk
    chunk_overlap=200,    # 200 characters overlap between chunks
    separators=["\n\n", "\n", " "]  # split at paragraphs first, then lines
)
chunks = splitter.split_documents(documents)

# Step 3: Create embeddings and store in vector database
embeddings = AnthropicEmbeddings()
vectorstore = Chroma.from_documents(
    documents=chunks,
    embedding=embeddings,
    persist_directory="./hr_vectorstore"
)

# Step 4: Create retrieval chain
retriever = vectorstore.as_retriever(
    search_type="mmr",    # Maximum Marginal Relevance — diverse results
    search_kwargs={"k": 5}  # retrieve top 5 chunks
)

qa_chain = RetrievalQA.from_chain_type(
    llm=llm,
    chain_type="stuff",   # stuff all chunks into prompt
    retriever=retriever,
    return_source_documents=True
)

# Query
result = qa_chain.invoke("What is Barclays' sick leave policy?")
print(result["result"])
print("Sources:", [doc.metadata for doc in result["source_documents"]])
```

### Tools — Give AI the Ability to Act

```python
from langchain.tools import tool
from langchain_anthropic import ChatAnthropic
from langchain.agents import create_tool_calling_agent, AgentExecutor
from langchain_core.prompts import ChatPromptTemplate

# Define tools (functions the AI can call)
@tool
def get_employee_attendance(employee_id: str, date: str) -> str:
    """Get attendance record for an employee on a specific date.
    Use this when asked about an employee's attendance or check-in status.

    Args:
        employee_id: The employee's ID (e.g., EMP-123)
        date: Date in YYYY-MM-DD format
    """
    # In real app: query your database
    return f"Employee {employee_id} checked in at 09:15 on {date}"

@tool
def apply_leave(employee_id: str, leave_type: str, from_date: str, to_date: str) -> str:
    """Apply for leave on behalf of an employee.

    Args:
        employee_id: The employee's ID
        leave_type: Type of leave (sick/casual/earned)
        from_date: Leave start date (YYYY-MM-DD)
        to_date: Leave end date (YYYY-MM-DD)
    """
    # In real app: submit to leave management system
    return f"Leave application submitted for {employee_id}: {leave_type} from {from_date} to {to_date}"

@tool
def get_leave_balance(employee_id: str) -> str:
    """Get current leave balance for an employee."""
    return f"Employee {employee_id} has: Sick: 8 days, Casual: 5 days, Earned: 12 days"

# Create agent with tools
tools = [get_employee_attendance, apply_leave, get_leave_balance]
llm_with_tools = ChatAnthropic(model="claude-3-5-sonnet-20241022").bind_tools(tools)

prompt = ChatPromptTemplate.from_messages([
    ("system", "You are an HR assistant. Use tools to help employees."),
    ("human", "{input}"),
    ("placeholder", "{agent_scratchpad}")  # where agent puts its thinking
])

agent = create_tool_calling_agent(llm_with_tools, tools, prompt)
agent_executor = AgentExecutor(agent=agent, tools=tools, verbose=True)

# Agent decides which tools to use automatically
result = agent_executor.invoke({
    "input": "Check if EMP-123 came to office today (2026-06-09) and apply sick leave for tomorrow"
})
print(result["output"])
# Agent calls get_employee_attendance, then apply_leave — automatically
```

---

## 🔧 LangGraph — Stateful Multi-Agent Workflows

```python
# LangGraph is the modern way to build complex agent workflows
# Think of it as a state machine — like a flowchart for AI

from langgraph.graph import StateGraph, END
from typing import TypedDict, Annotated
import operator

# Define the state — what information flows through the graph
class AttendanceWorkflowState(TypedDict):
    employee_id: str
    query: str
    attendance_data: str
    leave_balance: str
    final_response: str
    needs_escalation: bool

# Define nodes — each node is a function that processes state
def check_attendance_node(state: AttendanceWorkflowState):
    """Node 1: Check attendance"""
    # Call attendance service
    attendance = f"EMP {state['employee_id']} checked in at 09:15"
    return {"attendance_data": attendance}

def check_leave_balance_node(state: AttendanceWorkflowState):
    """Node 2: Check leave balance"""
    balance = "Sick: 8, Casual: 5, Earned: 12"
    return {"leave_balance": balance}

def generate_response_node(state: AttendanceWorkflowState):
    """Node 3: Generate final response using LLM"""
    llm = ChatAnthropic(model="claude-3-5-sonnet-20241022")
    response = llm.invoke(f"""
    Employee query: {state['query']}
    Attendance: {state['attendance_data']}
    Leave Balance: {state['leave_balance']}

    Provide a helpful response to the employee.
    """)
    return {"final_response": response.content}

def check_escalation_needed(state: AttendanceWorkflowState):
    """Conditional edge — should we escalate to HR?"""
    if "dispute" in state['query'].lower() or "wrong" in state['query'].lower():
        return "escalate"  # go to escalation node
    return "respond"       # go to response node

def escalate_to_hr_node(state: AttendanceWorkflowState):
    """Node 4: Escalate to HR (if needed)"""
    return {
        "final_response": f"This has been escalated to HR for employee {state['employee_id']}",
        "needs_escalation": True
    }

# Build the graph
workflow = StateGraph(AttendanceWorkflowState)

# Add nodes
workflow.add_node("check_attendance", check_attendance_node)
workflow.add_node("check_leave", check_leave_balance_node)
workflow.add_node("generate_response", generate_response_node)
workflow.add_node("escalate", escalate_to_hr_node)

# Add edges (connections between nodes)
workflow.set_entry_point("check_attendance")     # start here
workflow.add_edge("check_attendance", "check_leave")  # then check leave

# Conditional edge — go different places based on condition
workflow.add_conditional_edges(
    "check_leave",                    # from this node
    check_escalation_needed,          # run this function to decide
    {
        "escalate": "escalate",       # if returns "escalate" → go to escalate node
        "respond": "generate_response" # if returns "respond" → go to response node
    }
)

workflow.add_edge("generate_response", END)  # end after response
workflow.add_edge("escalate", END)           # end after escalation

# Compile and run
app = workflow.compile()

result = app.invoke({
    "employee_id": "EMP-123",
    "query": "I think my attendance is wrong for yesterday",
    "attendance_data": "",
    "leave_balance": "",
    "final_response": "",
    "needs_escalation": False
})

print(result["final_response"])
```

---

## 🔧 Multi-Agent Patterns with LangGraph

```python
# Pattern: Supervisor + Specialist Agents
# Like a manager who delegates to team members

from langchain_anthropic import ChatAnthropic
from langgraph.graph import StateGraph, END
from typing import TypedDict, List

class MultiAgentState(TypedDict):
    user_request: str
    selected_agent: str
    research_result: str
    analysis_result: str
    final_answer: str
    messages: List[str]

# Supervisor — decides which agent to use
def supervisor_node(state: MultiAgentState):
    llm = ChatAnthropic(model="claude-3-5-sonnet-20241022")
    decision = llm.invoke(f"""
    User request: {state['user_request']}

    Available agents:
    - research_agent: For finding information from documents
    - analysis_agent: For analyzing data and trends
    - answer_agent: For providing final answer

    Which agent should handle this? Reply with ONLY the agent name.
    """)
    return {"selected_agent": decision.content.strip()}

# Route based on supervisor's decision
def route_to_agent(state: MultiAgentState):
    return state["selected_agent"]

# Specialist agents
def research_agent_node(state: MultiAgentState):
    # This agent searches documents/knowledge base
    result = f"Found relevant information about: {state['user_request']}"
    return {"research_result": result, "messages": ["Research complete"]}

def analysis_agent_node(state: MultiAgentState):
    # This agent analyzes data
    result = f"Analysis of {state['user_request']}: trends show X"
    return {"analysis_result": result, "messages": ["Analysis complete"]}

def answer_agent_node(state: MultiAgentState):
    # Final agent that synthesizes everything
    llm = ChatAnthropic(model="claude-3-5-sonnet-20241022")
    context = f"""
    Research: {state.get('research_result', 'N/A')}
    Analysis: {state.get('analysis_result', 'N/A')}
    Request: {state['user_request']}
    """
    answer = llm.invoke(f"Based on this context, answer the request:\n{context}")
    return {"final_answer": answer.content}

# Build multi-agent graph
graph = StateGraph(MultiAgentState)
graph.add_node("supervisor", supervisor_node)
graph.add_node("research_agent", research_agent_node)
graph.add_node("analysis_agent", analysis_agent_node)
graph.add_node("answer_agent", answer_agent_node)

graph.set_entry_point("supervisor")
graph.add_conditional_edges(
    "supervisor",
    route_to_agent,
    {
        "research_agent": "research_agent",
        "analysis_agent": "analysis_agent",
        "answer_agent": "answer_agent"
    }
)
# All agents flow to answer_agent for final synthesis
graph.add_edge("research_agent", "answer_agent")
graph.add_edge("analysis_agent", "answer_agent")
graph.add_edge("answer_agent", END)

multi_agent_app = graph.compile()
```

---

## 💬 Interview Answer

**Q: "What is LangGraph and why would you use it over regular LangChain agents?"**

> *"LangGraph is a framework for building stateful, graph-based agent workflows. Think of it as a state machine for AI — you define nodes (processing steps), edges (connections between steps), and conditional edges (branching logic). Regular LangChain agents use a simple ReAct loop — think, act, observe, repeat — which works for simple tasks but breaks down for complex multi-step workflows. LangGraph gives you explicit control over the flow: you can branch based on conditions, loop back when needed, add human-in-the-loop approval steps, and checkpoint state so workflows can be paused and resumed. In our Agent Builder Platform I've built something similar — but LangGraph is now the standard way to express these patterns. For a use case like Barclays' economic crime detection, you might have a supervisor agent that routes to a research agent for gathering evidence, an analysis agent for pattern detection, and a compliance agent for generating reports — LangGraph makes this explicit, testable, and observable, which is critical in a regulated environment."*

---

# 🏗️ TOPIC 6 — TERRAFORM + GITOPS

## 🍎 Terraform Analogy

Terraform is like an **architect's blueprint for your cloud infrastructure**.
- Without Terraform: you click buttons in AWS console to create servers (manual, error-prone)
- With Terraform: you write a blueprint file, and Terraform builds EXACTLY what's in the blueprint
- If you delete the blueprint and run Terraform — it deletes the infrastructure
- If two people change the infrastructure — Terraform detects the difference (drift)

**GitOps** is like **using Git as the single source of truth for your infrastructure AND deployments**.
- Whatever is in Git = what should be running in production
- ArgoCD watches your Git repo and makes sure Kubernetes matches what's in Git
- If someone manually changes a pod — ArgoCD reverts it to match Git

---

## 🔧 Terraform — Advanced Concepts

```hcl
# Project structure — best practice
terraform/
  modules/
    vpc/
      main.tf        # VPC resources
      variables.tf   # input variables
      outputs.tf     # output values
    eks/
      main.tf
      variables.tf
      outputs.tf
  environments/
    dev/
      main.tf        # calls modules with dev config
      terraform.tfvars  # dev variable values
    prod/
      main.tf        # calls modules with prod config
      terraform.tfvars  # prod variable values
  backend.tf         # where to store state

# backend.tf — CRITICAL: store state remotely (not locally)
terraform {
  backend "s3" {
    bucket         = "barclays-terraform-state"
    key            = "attendance/prod/terraform.tfstate"
    region         = "ap-south-1"
    encrypt        = true                          # encrypt state file

    # DynamoDB for state locking — prevents two people running at same time
    dynamodb_table = "terraform-state-locks"
  }
}

# modules/vpc/main.tf — reusable VPC module
variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "environment" {
  description = "Environment name"
  type        = string
}

variable "tags" {
  description = "Tags to apply to all resources"
  type        = map(string)
  default     = {}
}

resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true

  tags = merge(var.tags, {
    Name        = "${var.environment}-vpc"
    Environment = var.environment
  })
}

resource "aws_subnet" "private" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 10)
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name = "${var.environment}-private-${count.index + 1}"
    "kubernetes.io/role/internal-elb" = "1"  # for EKS
  }
}

output "vpc_id" {
  value = aws_vpc.main.id
}

output "private_subnet_ids" {
  value = aws_subnet.private[*].id
}

# environments/prod/main.tf — use the module
module "vpc" {
  source = "../../modules/vpc"

  vpc_cidr    = "10.0.0.0/16"
  environment = "prod"
  tags = {
    Project    = "attendance-system"
    CostCenter = "engineering"
    ManagedBy  = "terraform"
  }
}

module "eks" {
  source = "../../modules/eks"

  cluster_name    = "attendance-prod"
  vpc_id          = module.vpc.vpc_id
  subnet_ids      = module.vpc.private_subnet_ids
  node_group_size = 3
}

# Terraform workspace — separate state per environment
# terraform workspace new dev
# terraform workspace new staging
# terraform workspace new prod
# terraform workspace select prod
# terraform plan
```

```bash
# Essential Terraform commands
terraform init          # initialise — download providers, setup backend
terraform plan          # show what WILL change (dry run — always do this first)
terraform apply         # apply changes (will ask for confirmation)
terraform apply -auto-approve  # apply without confirmation (use in CI/CD)
terraform destroy       # destroy ALL infrastructure (dangerous!)
terraform show          # show current state
terraform state list    # list all resources in state
terraform state show aws_vpc.main  # show specific resource
terraform import aws_vpc.vpc-12345 my-vpc  # import existing resource into state

# Detect drift — what changed outside of Terraform?
terraform plan          # any differences shown = drift
terraform refresh       # update state to match real infrastructure

# Common workflow
cd environments/prod
terraform init
terraform workspace select prod
terraform plan -out=tfplan.out    # save plan to file
terraform apply tfplan.out        # apply exact plan (no surprises)
```

---

## 🔧 GitOps with ArgoCD

```
Analogy: ArgoCD is like a security camera system for your Kubernetes cluster
  - It watches your Git repo (the blueprint)
  - It watches your Kubernetes cluster (the building)
  - If the building doesn't match the blueprint — it fixes it automatically
  - If someone rearranges furniture (changes K8s manually) — ArgoCD puts it back
```

```yaml
# ArgoCD Application — tells ArgoCD what to watch and where to deploy
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: attendance-api
  namespace: argocd
spec:
  project: default

  # WHERE is the source? (Git repo)
  source:
    repoURL: https://github.com/barclays/attendance-k8s-configs
    targetRevision: main          # watch this branch
    path: environments/prod/attendance-api  # in this directory

  # WHERE to deploy? (Kubernetes cluster)
  destination:
    server: https://kubernetes.default.svc  # this cluster
    namespace: attendance-prod

  # HOW to sync?
  syncPolicy:
    automated:
      prune: true      # delete resources removed from Git
      selfHeal: true   # fix manual changes automatically
    syncOptions:
      - CreateNamespace=true  # create namespace if doesn't exist
```

```bash
# ArgoCD CLI
argocd login argocd.barclays.com --username admin

# List all apps
argocd app list

# Check app status
argocd app get attendance-api

# Manual sync (if not auto-sync)
argocd app sync attendance-api

# Check sync status
argocd app wait attendance-api --sync

# Rollback to previous version
argocd app rollback attendance-api 3  # rollback to revision 3

# Diff — what's different between Git and cluster?
argocd app diff attendance-api
```

---

## 💬 Interview Answer

**Q: "What is GitOps and how does it improve deployment reliability?"**

> *"GitOps is a deployment methodology where Git is the single source of truth for both application code AND infrastructure configuration. Whatever is committed to the Git repository is what should be running in the cluster — always. ArgoCD implements this by continuously watching Git and comparing it to the live cluster state. If there's a drift — someone manually patched a deployment, a pod crashed and came back with wrong config — ArgoCD automatically reconciles back to the Git state. The benefits are significant: deployment is just a Git commit, so every change is auditable and reversible. Rollback is git revert. Access control to production is enforced through Git branch protection — not through direct cluster access. For Barclays, this maps directly to change management requirements — every production change has a trail in Git, approved through pull requests, and the cluster never drifts from the approved state."*

---

# 🧠 TOPIC 7 — ML & FINE-TUNING (LoRA, QLoRA, Quantization)

## 🍎 The Simple Analogy for the Whole ML Concept

Imagine you have a **brilliant university professor** (the base LLM — like GPT or Claude).
- The professor knows about EVERYTHING — history, science, cooking, law
- But you're a bank and need them to be an **expert in financial regulations**
- You have two options:

**Option 1 — RAG (Retrieval Augmented Generation):**
Give the professor a library card. Before answering, they look up relevant books.
- Fast and cheap
- Professor doesn't actually learn — just looks things up each time
- Best when: information changes frequently, you have lots of documents

**Option 2 — Fine-tuning:**
Send the professor back to university specifically for financial regulations.
- They actually LEARN — no need to look up books each time
- Expensive and slow to train
- Best when: you need consistent format/style, very domain-specific responses

---

## 🔧 Why LoRA? The Problem With Normal Fine-Tuning

```
Normal fine-tuning problem:
  GPT-3 has 175 BILLION parameters (numbers/weights)
  Fine-tuning = updating ALL 175 billion numbers
  Requires: 80+ GPU (expensive!) cards
  Time: days or weeks
  Cost: thousands of dollars

Analogy: Normal fine-tuning is like rewriting an entire encyclopaedia
         to add 100 new finance articles. Wasteful!
```

```
LoRA solution:
  Instead of changing ALL parameters — add small "sticky notes" to the model
  Only update the sticky notes (much smaller)
  Original encyclopaedia stays the same
  You just add a small supplement booklet

Technical: Add small matrices (rank r=8) alongside existing weight matrices
           Only these small matrices are trained
           Parameters reduced from 175B to ~1M (99.4% reduction!)
```

```python
# LoRA in code — using Hugging Face PEFT library
from transformers import AutoModelForCausalLM, AutoTokenizer, TrainingArguments
from peft import LoraConfig, get_peft_model, TaskType
from datasets import load_dataset

# 1. Load the base model (the professor)
model_name = "meta-llama/Llama-2-7b-hf"  # 7 billion parameter model
tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModelForCausalLM.from_pretrained(
    model_name,
    device_map="auto"  # automatically use GPU if available
)

# 2. Configure LoRA (define the sticky notes)
lora_config = LoraConfig(
    r=16,               # rank — size of the small matrices (higher = more parameters but better)
    lora_alpha=32,      # scaling factor (usually 2x rank)
    target_modules=[    # which layers to add LoRA to
        "q_proj",       # query projection in attention
        "v_proj",       # value projection in attention
        "k_proj",       # key projection
        "o_proj"        # output projection
    ],
    lora_dropout=0.1,   # regularization (prevent overfitting)
    bias="none",        # don't train biases
    task_type=TaskType.CAUSAL_LM  # we're doing text generation
)

# 3. Apply LoRA to model
peft_model = get_peft_model(model, lora_config)

# See how many parameters we're actually training
peft_model.print_trainable_parameters()
# Output: trainable params: 4,194,304 || all params: 6,742,609,920 || trainable: 0.06%
# We're only training 0.06% of parameters! 99.94% frozen!

# 4. Load your training data
dataset = load_dataset("json", data_files={
    "train": "finance_training_data.jsonl",
    "test": "finance_test_data.jsonl"
})

# 5. Train
from trl import SFTTrainer  # Supervised Fine-Tuning Trainer

training_args = TrainingArguments(
    output_dir="./fine-tuned-finance-model",
    num_train_epochs=3,
    per_device_train_batch_size=4,
    gradient_accumulation_steps=4,
    learning_rate=2e-4,
    logging_steps=10,
    save_strategy="epoch"
)

trainer = SFTTrainer(
    model=peft_model,
    train_dataset=dataset["train"],
    eval_dataset=dataset["test"],
    args=training_args,
    dataset_text_field="text"
)

trainer.train()

# 6. Save only the LoRA weights (tiny file!)
peft_model.save_pretrained("./lora-finance-adapter")
# Saves only ~10MB instead of 14GB for the full model!
```

---

## 🔧 QLoRA — LoRA + Quantization

```
Quantization Analogy:
  Normal model: weights stored as precise decimals (0.73621847...)
                Like storing 10 decimal places for every measurement
  Quantized:    weights stored as rough approximations (0.75)
                Like rounding to 2 decimal places
  4-bit quant:  Even rougher (only 16 possible values per weight)

  Result: Model takes 8x less memory!
  Cost: Very small accuracy loss (usually < 1%)

QLoRA = Quantize the big model (8x smaller) + Apply LoRA on top
  You can fine-tune a 70B model on ONE GPU instead of needing 8!
```

```python
# QLoRA — Load model in 4-bit precision
import torch
from transformers import BitsAndBytesConfig

# 4-bit quantization config
quantization_config = BitsAndBytesConfig(
    load_in_4bit=True,              # load model in 4-bit
    bnb_4bit_compute_dtype=torch.bfloat16,  # compute in bfloat16
    bnb_4bit_use_double_quant=True, # nested quantization (even more compression)
    bnb_4bit_quant_type="nf4"       # NF4 quantization type (best for LLMs)
)

# Load huge model in 4-bit (fits on single GPU!)
model = AutoModelForCausalLM.from_pretrained(
    "meta-llama/Llama-2-70b-hf",  # 70 BILLION parameter model
    quantization_config=quantization_config,
    device_map="auto"
)

# Now apply LoRA on top of quantized model
# Everything else same as LoRA example above
peft_model = get_peft_model(model, lora_config)
# Fine-tune a 70B model on a single consumer GPU!
```

---

## 🔧 Other Fine-Tuning Methods

```
Method          How it works                      Best for
──────────────────────────────────────────────────────────────────
LoRA            Small matrices alongside weights   Most use cases
QLoRA           LoRA + 4-bit quantization          Limited GPU memory
Prefix Tuning   Prepend trainable tokens to input  Generation tasks
Adapter Layers  Small layers between transformers  Multi-task learning
Full Fine-tune  Update ALL weights                 When you have resources

Analogy for Prefix Tuning:
  Instead of teaching the professor (changing weights)
  You give them a special notecard to read BEFORE every question
  The notecard (prefix tokens) guides their responses
  The professor doesn't change — the notecard does
```

---

## 🔧 Model Quantization, Pruning, Optimization

```python
# Model Quantization — make model smaller for inference
from transformers import AutoModelForCausalLM
import torch

# Post-training quantization (after training is done)
from optimum.quanto import quantize, qint8

model = AutoModelForCausalLM.from_pretrained("my-fine-tuned-model")
quantize(model, weights=qint8)  # quantize weights to 8-bit

# Model Pruning — remove unimportant weights
# Analogy: Like editing a long document and removing filler words
# 30% of model weights can be zero without much accuracy loss

# ONNX Export — convert for faster inference
from optimum.exporters.onnx import main_export

main_export(
    model_name_or_path="my-fine-tuned-model",
    output="./model-onnx",
    task="text-generation"
)
# ONNX model is faster at inference — works on any hardware

# vLLM — high-throughput LLM serving
# Analogy: Instead of serving one customer at a time,
#          vLLM serves many customers in parallel using PagedAttention
from vllm import LLM, SamplingParams

llm = LLM(model="my-fine-tuned-model")
sampling_params = SamplingParams(temperature=0.7, max_tokens=100)

# Process MANY requests efficiently
prompts = ["Question 1...", "Question 2...", "Question 3..."]
outputs = llm.generate(prompts, sampling_params)
# vLLM handles batching automatically — 10-100x faster than naive serving
```

---

## 💬 Interview Answer

**Q: "Explain LoRA and QLoRA and when you would use fine-tuning vs RAG."**

> *"LoRA — Low-Rank Adaptation — solves the core problem with traditional fine-tuning: updating all billions of parameters in a large model requires enormous compute. Instead, LoRA freezes the original model weights entirely and injects small trainable matrices — low-rank decompositions — alongside the attention layers. These small matrices have maybe a million parameters versus the model's billions, so you're training less than 1% of the total parameters. The model's general knowledge is preserved, and you're just adding domain-specific behaviour through these small adapters. QLoRA extends this by first quantizing the base model to 4-bit precision — reducing its memory footprint by 8x — then applying LoRA on top. This lets you fine-tune a 70B parameter model on a single GPU that would otherwise need 8.*
>
> *As for fine-tuning versus RAG: RAG is my first choice when information changes frequently, when I need to cite specific sources, or when the knowledge base is large. Fine-tuning is the right choice when I need consistent output format, when latency matters and I can't afford retrieval overhead, or when the domain is so specific that the model needs to deeply internalize patterns — not just look them up. In practice, for Barclays I'd combine both: fine-tune for regulatory tone, format, and compliance language, then use RAG to ground responses in current policy documents."*

---

# 🔒 TOPIC 8 — SECURITY & OBSERVABILITY

## 🔧 HashiCorp Vault — Secrets Management

```
Analogy: Vault is like a bank safe (appropriate for Barclays!)
  - All your secrets (passwords, API keys, certificates) live in Vault
  - Applications request secrets at runtime — secrets never in code or config files
  - Every access is logged — audit trail of who accessed what and when
  - Secrets auto-rotate — the safe changes the combination regularly
```

```python
# Using Vault in Python
import hvac  # pip install hvac

# Connect to Vault
client = hvac.Client(
    url='https://vault.barclays.internal:8200',
    token=os.environ['VAULT_TOKEN']  # or use AppRole auth
)

# Read a secret
secret = client.secrets.kv.v2.read_secret_version(
    path='attendance/database',
    mount_point='secret'
)
db_password = secret['data']['data']['password']

# AppRole authentication (for applications — no human token)
client = hvac.Client(url='https://vault.barclays.internal:8200')
client.auth.approle.login(
    role_id=os.environ['VAULT_ROLE_ID'],
    secret_id=os.environ['VAULT_SECRET_ID']
)

# Dynamic secrets — Vault generates temporary DB credentials
# Instead of one password everyone shares:
db_creds = client.secrets.database.generate_credentials(name='attendance-db-role')
db_username = db_creds['data']['username']  # e.g., v-app-xK3mP
db_password = db_creds['data']['password']  # expires in 1 hour!
# After 1 hour these credentials expire automatically — massive security win
```

---

## 🔧 SLOs, SLIs, Error Budgets — Google SRE Methodology

```
Analogy: Running a train service
  SLI (Service Level Indicator) = the measurement
    "What percentage of trains arrived within 5 minutes of schedule?"
    Answer: 97.3% this month

  SLO (Service Level Objective) = your target
    "We commit to 95% of trains arriving within 5 minutes"
    97.3% > 95% ✅ We're meeting our SLO

  Error Budget = how much failure you're allowed
    SLO = 99.9% availability = 0.1% allowed failure
    In a month (43,200 minutes): 43.2 minutes of downtime allowed
    Used so far: 15 minutes
    Remaining error budget: 28.2 minutes

  Why this matters:
    If you've used 80% of error budget → stop deploying new features
    If error budget is healthy → deploy freely
    It's a data-driven decision, not opinion
```

```yaml
# SLO definition in Prometheus/Grafana
# Alert when error budget is burning too fast

groups:
  - name: attendance-api-slos
    rules:
      # SLI: What percentage of requests succeed?
      - record: job:attendance_api:success_rate
        expr: |
          sum(rate(http_requests_total{job="attendance-api",code!~"5.."}[5m]))
          /
          sum(rate(http_requests_total{job="attendance-api"}[5m]))

      # Alert: Error budget burning too fast (>2% error rate for 1 hour)
      - alert: ErrorBudgetBurnRateFast
        expr: |
          (1 - job:attendance_api:success_rate) > 0.02
        for: 1h
        labels:
          severity: critical
        annotations:
          summary: "Attendance API error rate {{ $value | humanizePercentage }}"
          description: "Error budget burning at 2x normal rate"
```

---

## 🔧 OpenTelemetry — Complete Observability

```python
# Complete OTel setup for Python FastAPI service
from opentelemetry import trace, metrics
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.httpx import HTTPXClientInstrumentor

# Setup tracing
trace.set_tracer_provider(TracerProvider())
trace.get_tracer_provider().add_span_processor(
    BatchSpanProcessor(OTLPSpanExporter(endpoint="http://jaeger:4317"))
)

# Auto-instrument FastAPI (all requests traced automatically)
FastAPIInstrumentor.instrument()
HTTPXClientInstrumentor.instrument()  # trace outbound HTTP calls too

# Get tracer for custom spans
tracer = trace.get_tracer(__name__)

async def process_llm_request(prompt: str, employee_id: str) -> str:
    # Custom span for business operation
    with tracer.start_as_current_span("llm.process_request") as span:
        span.set_attribute("employee.id", employee_id)
        span.set_attribute("prompt.length", len(prompt))
        span.set_attribute("llm.model", "claude-3-5-sonnet")

        try:
            result = await call_claude_api(prompt)
            span.set_attribute("response.length", len(result))
            span.set_attribute("success", True)
            return result
        except Exception as e:
            span.record_exception(e)
            span.set_status(Status(StatusCode.ERROR, str(e)))
            raise

# Custom metrics
meter = metrics.get_meter(__name__)
llm_request_counter = meter.create_counter(
    "llm_requests_total",
    description="Total LLM API requests"
)
llm_latency = meter.create_histogram(
    "llm_request_duration_seconds",
    description="LLM request latency"
)

# Record metrics
llm_request_counter.add(1, {"model": "claude", "status": "success"})
llm_latency.record(0.345, {"model": "claude"})
```

---

## 💬 Interview Answer — Security

**Q: "How would you approach secrets management for a financial application on Kubernetes?"**

> *"My approach is zero secrets in code, zero secrets in Kubernetes Secrets objects without encryption. The architecture: HashiCorp Vault as the central secrets store — all credentials, API keys, certificates live there and nowhere else. Applications authenticate to Vault using Kubernetes service accounts via Vault's Kubernetes auth method — the pod presents its service account token, Vault validates it, and returns a short-lived Vault token. From there the application fetches only the secrets it needs, scoped by Vault policy to exactly those paths. The huge advantage of Vault's dynamic secrets is for databases — instead of a shared password that lives forever, Vault generates a unique temporary credential per application instance that expires in hours. Even if credentials leak, they're useless within hours. For Barclays, this audit trail — every secret access logged with which service, which pod, when — directly satisfies FCA audit requirements. At the Kubernetes level: RBAC to restrict which service accounts can do what, Network Policies to prevent lateral movement, and image scanning in the CI/CD pipeline so vulnerable containers never reach production."*

---

# ⚡ TOPIC 9 — QUICK REFERENCE CHEATSHEET

## Your Unfair Advantage — Say This for EVERY AI Question

```
"At Concentrix I built two production AI systems:

1. Enterprise AI Gateway (LiteLLM + Langfuse + LLM Guard)
   - Single endpoint for Claude, GPT-4, Gemini, open-source models
   - Per-team rate limiting, budget alerts, cost tracking
   - Real-time safety filtering via LLM Guard
   - This is essentially what companies spend months building —
     I led this from scratch

2. No-code AI Agent Builder Platform
   - Multi-agent orchestration (agents calling agents)
   - Full RAG pipeline (ingest → chunk → embed → store → retrieve → generate)
   - Tool use / function calling with external APIs
   - Persistent memory across sessions
   - Used by enterprise clients in production

These aren't demos or prototypes — they're production systems
serving real enterprise clients."
```

---

## Critical One-Liners — Memorise These

```
LINUX:
  "Diagnose systematically: CPU (top) → Memory (free -h) →
   Disk (df -h) → Network (ss -tulpn) → Logs (grep ERROR)"

PYTHON:
  "FastAPI for async services, Pydantic for validation,
   asyncio.gather for parallel calls, decorators for cross-cutting concerns"

AWS IAM:
  "Least privilege per service, roles not users for applications,
   assume-role for cross-account, never hardcode credentials"

VPC:
  "Public subnet = Load Balancer only, Private subnet = everything else,
   NAT Gateway = private can call out, nobody can call in"

KUBERNETES RBAC:
  "Role = what's allowed, RoleBinding = who gets it,
   ServiceAccount = application's identity"

NETWORK POLICY:
  "Default deny all, then explicitly allow specific pod-to-pod paths"

OPENSHIFT vs K8S:
  "Same core, stricter security defaults (SCC), Routes instead of Ingress,
   built-in registry and Tekton CI/CD"

LANGCHAIN vs LANGGRAPH:
  "LangChain = Lego pieces for AI. LangGraph = state machine connecting the pieces.
   LangGraph for complex multi-step workflows with branching and human approval"

LORA:
  "Freeze 99%+ of model weights, train tiny matrices added alongside attention layers.
   Same capability, 100x less compute"

QLORA:
  "LoRA + 4-bit quantization = fine-tune 70B model on one GPU"

RAG vs FINE-TUNING:
  "RAG for changing information + citations.
   Fine-tuning for consistent format + deep domain patterns"

TERRAFORM STATE:
  "Remote state in S3 + DynamoDB locking = no conflicts,
   workspaces for per-environment isolation"

GITOPS:
  "Git = source of truth. ArgoCD watches Git, makes cluster match.
   Manual changes auto-reverted. Rollback = git revert"

VAULT:
  "Dynamic secrets expire automatically = leaked credentials useless in hours.
   Every access audited = FCA compliance satisfied"

SLO/ERROR BUDGET:
  "SLI = what you measure. SLO = your target. Error budget = allowed failure.
   Budget exhausted → stop deploying, fix reliability first"
```

---

## Barclays LEAD Behaviours — Weave Into Every Answer

```
L — Listen and be authentic
  "I make sure to understand the full requirement before proposing solutions"
  "When I disagreed with the team on Envoy Gateway, I first listened fully..."

E — Energise and inspire
  "I brought the team along by showing a proof of concept first..."
  "I created internal tech sessions to upskill engineers on LangGraph..."

A — Align across the enterprise
  "I collaborated with product, QA, and stakeholders throughout..."
  "The AI Gateway served all engineering teams — I designed it for reuse..."

D — Develop others
  "I mentored junior engineers through design reviews and pair programming..."
  "I raised engineering standards through structured code review guidelines..."
```

---

## Day-Before Checklist

```
TECHNICAL REVIEW (1 hour):
□ Read the one-liners above 3 times
□ Say "Your Unfair Advantage" paragraph out loud — time it (should be 90 sec)
□ Re-read LoRA explanation — can you explain it simply?
□ Re-read LangGraph state machine — understand the flow

BEHAVIOURAL PREP (30 min):
□ "Tell me about yourself" — say it out loud
□ Envoy Gateway story — STAR format, 3 minutes
□ Agent Builder Platform story — STAR format, 3 minutes
□ "Why Barclays?" — you worked there before — use that

LOGISTICS (30 min):
□ Research Barclays Economic Crime COO (the specific division for this role)
□ Prepare 3 smart questions to ask them
□ Check your camera/mic/internet for video call

THE NIGHT BEFORE:
□ Stop studying at 10 PM
□ Sleep — tired brain loses interviews that preparation wins
```

---

## 3 Smart Questions to Ask Barclays

```
1. "The role is in Group Economic Crime COO — can you tell me more about
   the specific AI/platform challenges the team is working on right now?
   I'm particularly curious how LLM-based agents are being applied to
   economic crime detection."

2. "How mature is the OpenShift/Kubernetes platform at Barclays Pune,
   and what does the team's involvement look like in day-to-day
   cluster operations versus new capability development?"

3. "What does success look like in this role at the 6-month and
   12-month marks — what would I have built or delivered?"
```

---

*Two days is enough to go from knowing to owning these concepts.*
*The depth is in your real experience. This guide is the vocabulary to express it.*
*You built a production AI Gateway and Agent Platform — nobody else in that room did.*
*Own it. 💪*