# Griffin AWS Console Verification Guide

A step-by-step walkthrough to verify all provisioned resources, roles, and permissions for the `griffin-batch-test` CloudFormation stack in the AWS Console.

> **Region:** Always confirm you are in **Asia Pacific (Sydney) `ap-southeast-2`** (top-right dropdown) before checking any resource.

---

## 1. CloudFormation Stack (Overview)

1. Go to **AWS Console** → search "CloudFormation" → open it
2. Make sure region is **Asia Pacific (Sydney) ap-southeast-2** (top-right dropdown)
3. Click **Stacks** → find `griffin-batch-test`
4. Check **Status** = `CREATE_COMPLETE`
5. Click the stack → tabs to review:
   - **Events** — full deployment history, any failures
   - **Resources** — every resource CFN created (with links)
   - **Outputs** — exported values (URLs, etc.)
   - **Parameters** — what values were used (VPC, Subnet, CIDR, etc.)
   - **Template** — the actual YAML that was deployed

---

## 2. EC2 Instance

1. From the CFN **Resources** tab → click the `GriffinInstance` Physical ID link
   *(Or: EC2 → Instances → search `griffin`)*
2. Check:
   - **State**: `running`
   - **Instance type**: matches your template (e.g., `t3.medium`)
   - **Public IP**: `3.25.244.218`
   - **AMI ID**: ECS-optimized Amazon Linux 2
   - **Key pair**: `griffin-test-key`
   - **IAM Role**: click the role link (should be `GriffinEcsInstanceRole` or similar)
3. Click **Storage** tab → confirm root volume exists
4. Click **Security** tab → shows the attached security group and IAM role

---

## 3. Security Group

1. From the EC2 instance → **Security** tab → click the Security Group link
   *(Or: EC2 → Security Groups → search `griffin`)*
2. Check **Inbound rules**:
   - Port `38080` (Griffin UI) — CIDR should be your IP `/32`
   - Port `38088` (YARN UI)
   - Port `39200` (Elasticsearch)
   - Port `22` (SSH) — if present
3. Check **Outbound rules**: should allow all outbound (`0.0.0.0/0`)
4. Note the Security Group ID (`sg-0a45e1e82b2b4076d`)

---

## 4. IAM Role & Permissions

1. Go to **IAM** → **Roles** → search `griffin` or `Griffin`
2. Open the `GriffinEcsInstanceRole`:
   - **Trust relationships** tab → should trust `ec2.amazonaws.com`
   - **Permissions** tab → should show:
     - `AmazonEC2ContainerServiceforEC2Role` (ECS agent permissions)
     - `AmazonSSMManagedInstanceCore` (SSM access — allows console-based shell access)
3. Click each policy → **JSON** tab to see the exact permissions granted

---

## 5. ECS Cluster

1. Go to **ECS** → **Clusters** → click `griffin-batch-test` (or similar name)
2. Check:
   - **Status**: `ACTIVE`
   - **Registered container instances**: `1`
   - **Running tasks**: `1`
3. Click **Tasks** tab → find the running task → click it
4. In the task detail:
   - **Last status**: `RUNNING`
   - **Containers** section → expand each:
     - `griffin` — status `RUNNING`, exit code absent (still running)
     - `es` (Elasticsearch) — status `RUNNING`
   - **Network bindings** — shows host port → container port mappings

---

## 6. ECS Task Definition

1. ECS → **Task Definitions** → find `griffin-batch` (or similar)
2. Click the latest revision → review:
   - **Network mode**: `bridge`
   - **Container definitions**: `griffin` and `es`
   - CPU/memory allocations
   - Port mappings for each container
   - Environment variables

---

## 7. ECS Service

1. ECS → Clusters → your cluster → **Services** tab
2. Find the Griffin service → check:
   - **Desired count** vs **Running count** (should match)
   - **Deployments** tab — shows rollout history
   - **Events** tab — useful for troubleshooting past failures

---

## 8. CloudWatch Logs

1. Go to **CloudWatch** → **Log groups** → search `/griffin` or `griffin`
2. Find the log group (e.g., `/griffin/batch`)
3. Click it → see **Log streams** (one per container run)
4. Open a stream → see real-time stdout/stderr from Griffin and Elasticsearch containers

---

## 9. EC2 Key Pair

1. EC2 → **Key Pairs** (under Network & Security)
2. Find `griffin-test-key` — confirm it exists
3. Note: the `.pem` file is at `~/.ssh/griffin-test-key.pem` on your local machine

---

## 10. VPC & Subnet

1. Go to **VPC** → **Your VPCs** → find `vpc-09de7c3a5eca00c56`
2. Check it's the **Default VPC**
3. VPC → **Subnets** → find `subnet-0f0224219b67fe5bb`
4. Confirm:
   - Availability zone: `ap-southeast-2a`
   - **Auto-assign public IPv4**: enabled (required for public access)
   - Route table has an Internet Gateway route (`0.0.0.0/0 → igw-xxx`)

---

## Quick Summary Checklist

| Resource | Where to Check | Expected State |
|---|---|---|
| CFN Stack | CloudFormation → Stacks | `CREATE_COMPLETE` |
| EC2 Instance | EC2 → Instances | `running` |
| Security Group | EC2 → Security Groups | Inbound rules on ports 38080/38088/39200 |
| IAM Role | IAM → Roles → `GriffinEcsInstanceRole` | 2 managed policies attached |
| ECS Cluster | ECS → Clusters | `ACTIVE`, 1 instance registered |
| ECS Task | ECS → Clusters → Tasks | Both containers `RUNNING` |
| CloudWatch Logs | CloudWatch → Log groups | Log streams present |
| Key Pair | EC2 → Key Pairs | `griffin-test-key` exists |

---

## Live Endpoints (griffin-batch-test)

| Service | URL |
|---|---|
| Griffin UI | http://3.25.244.218:38080 |
| YARN UI | http://3.25.244.218:38088 |
| Elasticsearch | http://3.25.244.218:39200 |

> **Note:** The security group only allows access from a specific IP (`/32`). If you cannot reach these URLs, your outbound IP may have changed. Run `curl -s https://checkip.amazonaws.com` to get your current IP, then update the security group inbound rules accordingly.
