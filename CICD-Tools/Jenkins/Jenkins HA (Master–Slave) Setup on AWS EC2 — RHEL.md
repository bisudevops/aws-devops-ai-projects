# Jenkins HA (Master–Slave) Setup on AWS EC2 — RHEL

## Reality check before you build this

Open-source Jenkins has **no native active-active controller clustering**. The controller is a single Java process with an in-memory model and a filesystem-based `JENKINS_HOME` — you can't run two live controllers writing to the same state simultaneously without corruption. True active-active HA is a CloudBees (commercial) feature.

What you *can* build with OSS Jenkins, and what this guide implements, is the standard production pattern:

- **Controller**: Active-passive. One controller EC2 instance running at a time, with `JENKINS_HOME` on **EFS** (shared, persistent storage). If the active instance dies, a standby takes over the same data. This is normally automated with an **Auto Scaling Group (min=max=desired=1)** — if the instance fails, ASG launches a replacement that mounts the same EFS and comes back up with full state. That's the realistic "HA" for the controller.
- **Agents (slaves)**: Multiple EC2 RHEL instances, connected via SSH, doing the actual build work. These scale horizontally with no such limitation — this part is genuinely parallel/distributed.

If you truly need zero-downtime active-active controllers, that requires CloudBees Operations Center — happy to cover that separately if it's a hard requirement.

---

## Architecture

```
                         Route53 (jenkins.yourdomain.com)
                                    │
                              ALB (port 443/8080)
                                    │
                     ┌──────────────────────────┐
                     │   ASG (min=1,max=1)       │
                     │   Jenkins Controller       │
                     │   RHEL9 EC2, JENKINS_HOME  │
                     │   mounted from EFS         │
                     └──────────────────────────┘
                                    │
                          EFS (JENKINS_HOME, Multi-AZ)
                                    │
        ┌───────────────┬──────────────────┬───────────────┐
        │  Agent-1 (RHEL)│  Agent-2 (RHEL)  │  Agent-N (RHEL)│
        │  SSH, port 22   │  SSH, port 22    │  SSH, port 22  │
        └───────────────┴──────────────────┴───────────────┘
```

---

## Prerequisites

- AWS account, VPC with at least 2 subnets across 2 AZs (for EFS mount targets + ASG)
- IAM permissions to create EC2, EFS, ALB, ASG, Route53, Security Groups
- A RHEL 9 AMI (or RHEL 8) — Red Hat's official AMI from AWS Marketplace, or your golden AMI
- Key pair for SSH access
- Domain/hosted zone in Route53 (optional but recommended)

---

## Step 1: Create the EFS filesystem for JENKINS_HOME

```bash
aws efs create-file-system \
  --creation-token jenkins-home-efs \
  --performance-mode generalPurpose \
  --throughput-mode bursting \
  --encrypted \
  --tags Key=Name,Value=jenkins-home-efs

# Note the FileSystemId returned, e.g. fs-0123456789abcdef0
```

Create mount targets in each AZ/subnet the controller ASG can land in:

```bash
aws efs create-mount-target \
  --file-system-id fs-0123456789abcdef0 \
  --subnet-id subnet-aaaa1111 \
  --security-groups sg-efs-jenkins

aws efs create-mount-target \
  --file-system-id fs-0123456789abcdef0 \
  --subnet-id subnet-bbbb2222 \
  --security-groups sg-efs-jenkins
```

**Security group `sg-efs-jenkins`** must allow inbound NFS (port 2049) from the controller's security group only.

---

## Step 2: Security groups

| SG | Inbound rules |
|---|---|
| `sg-jenkins-controller` | 8080 (Jenkins UI) from ALB SG; 50000 (JNLP agent port) from `sg-jenkins-agent`; 22 from your bastion/admin IP |
| `sg-jenkins-agent` | 22 from `sg-jenkins-controller` (controller initiates SSH to agents) |
| `sg-efs-jenkins` | 2049 from `sg-jenkins-controller` |
| `sg-alb` | 443/80 from your allowed CIDR (office IP, VPN, or 0.0.0.0/0 if public) |

---

## Step 3: Launch the controller EC2 instance (RHEL)

Launch a RHEL 9 instance (e.g. `t3.large` — Jenkins controller is memory-hungry once you have plugins and running jobs) with an **instance profile** that has permission to mount EFS (`elasticfilesystem:ClientMount`, `ClientWrite`).

SSH in and run:

```bash
sudo dnf update -y

# Java 17 (required for Jenkins 2.463+)
sudo dnf install -y java-17-openjdk java-17-openjdk-devel

# Add Jenkins repo
sudo wget -O /etc/yum.repos.d/jenkins.repo \
  https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.repo.key

sudo dnf install -y jenkins

# Don't start yet — we need to redirect JENKINS_HOME to EFS first
sudo systemctl stop jenkins 2>/dev/null || true
```

---

## Step 4: Mount EFS as JENKINS_HOME

```bash
sudo dnf install -y amazon-efs-utils

sudo mkdir -p /var/lib/jenkins

# Mount via efs-utils with the EFS mount helper (uses TLS)
sudo mount -t efs -o tls fs-0123456789abcdef0:/ /var/lib/jenkins

# Persist across reboots
echo "fs-0123456789abcdef0:/ /var/lib/jenkins efs _netdev,tls 0 0" | \
  sudo tee -a /etc/fstab

# Fix ownership — jenkins user/group created by the rpm install
sudo chown -R jenkins:jenkins /var/lib/jenkins
```

Verify:

```bash
df -h | grep jenkins
# should show fs-0123456789abcdef0:/  on /var/lib/jenkins
```

---

## Step 5: SELinux and firewall (RHEL specifics)

```bash
# Open the Jenkins port
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --permanent --add-port=50000/tcp   # agent JNLP port
sudo firewall-cmd --reload

# If SELinux is enforcing, allow Jenkins to bind and access EFS mount
sudo setsebool -P httpd_can_network_connect 1
sudo semanage fcontext -a -t var_lib_t "/var/lib/jenkins(/.*)?" 2>/dev/null
sudo restorecon -Rv /var/lib/jenkins
```

If you hit SELinux denials, check `sudo ausearch -m avc -ts recent` and address with `audit2allow` rather than disabling SELinux outright.

---

## Step 6: Start Jenkins and complete setup wizard

```bash
sudo systemctl enable --now jenkins
sudo systemctl status jenkins

# Initial admin password
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Browse to `http://<controller-ip>:8080`, paste the password, install suggested plugins, create your admin user.

---

## Step 7: Put the controller behind an ALB + ASG for self-healing

1. Create a **Launch Template** from the configured instance (or bake an AMI with the above steps done via user-data so any replacement instance auto-configures itself — recommended, since ASG replacements won't have your manual SSH steps).
2. Create an **ASG** with `min=1, max=1, desired=1`, using that launch template, spanning your subnets.
3. Create an **ALB** (or NLB) target group on port 8080, health check path `/login`, attach the ASG.
4. Point Route53 `jenkins.yourdomain.com` (A/Alias) at the ALB.

Example user-data script to make this self-healing (bakes steps 3–6 into instance boot):

```bash
#!/bin/bash
dnf install -y java-17-openjdk amazon-efs-utils
wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.repo.key
dnf install -y jenkins
systemctl stop jenkins
mkdir -p /var/lib/jenkins
mount -t efs -o tls fs-0123456789abcdef0:/ /var/lib/jenkins
echo "fs-0123456789abcdef0:/ /var/lib/jenkins efs _netdev,tls 0 0" >> /etc/fstab
chown -R jenkins:jenkins /var/lib/jenkins
firewall-cmd --permanent --add-port=8080/tcp
firewall-cmd --permanent --add-port=50000/tcp
firewall-cmd --reload
systemctl enable --now jenkins
```

Now: if the controller instance is terminated or fails health checks, ASG launches a replacement, it mounts the same EFS, and Jenkins comes back with **all jobs, config, credentials, and build history intact** — typically a 2–4 minute RTO. This is the practical HA model for OSS Jenkins.

---

## Step 8: Launch and prepare agent (slave) EC2 instances

Repeat for each agent (e.g. `agent-1`, `agent-2`), RHEL AMI, sized per your build workload:

```bash
sudo dnf update -y
sudo dnf install -y java-17-openjdk git

# Create jenkins user matching the controller's expectations
sudo useradd -m -d /home/jenkins -s /bin/bash jenkins

# Generate an SSH keypair ON THE CONTROLLER (not here) — see Step 9
sudo mkdir -p /home/jenkins/.ssh
sudo chmod 700 /home/jenkins/.ssh
```

---

## Step 9: Connect agents to the controller via SSH

**On the controller**, generate a dedicated key pair for agent connections:

```bash
sudo -u jenkins ssh-keygen -t ed25519 -f /var/lib/jenkins/.ssh/agent_key -N ""
sudo cat /var/lib/jenkins/.ssh/agent_key.pub
```

**On each agent**, add that public key:

```bash
echo "<paste public key here>" | sudo tee -a /home/jenkins/.ssh/authorized_keys
sudo chmod 600 /home/jenkins/.ssh/authorized_keys
sudo chown -R jenkins:jenkins /home/jenkins/.ssh
```

**In the Jenkins UI** (controller):

1. Install the plugin: *Manage Jenkins → Plugins → SSH Build Agents* (usually already in suggested set).
2. *Manage Jenkins → Credentials → System → Global credentials → Add Credentials*:
   - Kind: **SSH Username with private key**
   - Username: `jenkins`
   - Private key: paste contents of `/var/lib/jenkins/.ssh/agent_key` (private key, from controller)
3. *Manage Jenkins → Nodes → New Node*:
   - Name: `agent-1`
   - Type: Permanent Agent
   - Remote root directory: `/home/jenkins/agent`
   - Labels: `rhel linux docker` (whatever fits your job routing)
   - Launch method: **Launch agents via SSH**
   - Host: private IP of agent-1
   - Credentials: select the one created above
   - Host Key Verification Strategy: "Known hosts file" (or "Manually trusted key" for first connect)
4. Save. Jenkins will SSH in, drop `agent.jar`, and bring the node online — check *Manage Jenkins → Nodes* for a green icon.

Repeat for each agent instance.

---

## Step 10: Test failover

1. **Controller failover**: terminate the controller EC2 instance manually. Confirm the ASG launches a replacement, EFS remounts, and Jenkins comes back with the same job history at the same URL (ALB target group re-registers automatically once health checks pass).
2. **Agent failure**: stop an agent instance, confirm jobs pinned to that label queue and route to the next available agent (assuming you're not hard-pinning single jobs to a single named agent).

---

## Operational notes

- **Backups**: EFS is durable but not a backup strategy on its own — enable **EFS automatic backups** (AWS Backup) for point-in-time recovery of `JENKINS_HOME`, since a bad plugin update or accidental config wipe will otherwise be replicated instantly to the "standby."
- **Plugin updates**: test in a non-prod controller pointed at an EFS *clone*/snapshot before applying to the ASG's launch template AMI.
- **Configuration as Code**: strongly consider the **JCasC (Jenkins Configuration as Code)** plugin so controller config is defined in YAML and reproducible instead of relying purely on the EFS state — this also makes the "replace the instance" recovery path much cleaner to reason about.
- **Secrets**: don't bake credentials into user-data or the AMI. Use Jenkins' credential store (backed by the EFS-persisted `JENKINS_HOME`) or externalize to AWS Secrets Manager / your Vault setup via the Vault plugin.
- **Monitoring**: wire ALB target group health + CloudWatch alarms on the ASG, plus Jenkins' own Prometheus/OpenTelemetry metrics exporter if you want queue depth and executor utilization visibility.
