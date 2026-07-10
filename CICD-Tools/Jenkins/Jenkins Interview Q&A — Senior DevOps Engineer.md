# Jenkins Interview Q&A — Senior DevOps Engineer

First-person mock answers, grounded in an EC2/RHEL controller + EFS + ASG + SSH-agent setup (matching what you'd have just built/discussed). Adjust specifics if your actual environment differs before using these live.

---

## 1. Architecture & HA

**Q: Walk me through how you've architected Jenkins for high availability.**

> In my setup, the controller runs on RHEL EC2 behind an ALB, managed by an Auto Scaling Group with min/max/desired all set to 1. `JENKINS_HOME` isn't on local EBS — it's mounted from EFS over NFS with TLS. So if the controller instance fails or gets terminated, the ASG launches a replacement from a launch template, it auto-mounts the same EFS filesystem via user-data, and Jenkins comes back with every job, credential, and build history intact. Typical recovery is a few minutes, not a restore-from-backup exercise.
>
> I'm upfront that this is active-passive, not active-active — vanilla Jenkins can't run two live controllers against the same state; that's a CloudBees Operations Center feature. For build capacity and actual parallelism, that's handled by the agent fleet, which scales horizontally with no such constraint.

**Q: Why EFS instead of just an EBS volume with a snapshot policy?**

> EBS is single-AZ and attached to one instance at a time — if that AZ or instance has an issue, I'd need to detach/reattach or restore from a snapshot, which is minutes-to-tens-of-minutes of RTO and depends on snapshot recency for RPO. EFS is multi-AZ by design with mount targets in each subnet, so a replacement instance in a different AZ can mount the same filesystem immediately. The tradeoff is NFS latency versus EBS's block-level performance, which matters for very heavy `JENKINS_HOME` I/O — but for controller metadata and job configs, that's rarely the bottleneck; build workspace I/O happens on the agents, not the controller.

**Q: What's your actual RPO/RTO here, and how would you tighten it?**

> RTO is bounded by ASG launch time plus health-check grace period, plus Jenkins JVM startup — call it 2-4 minutes in practice. RPO is close to zero for anything already flushed to `JENKINS_HOME` on EFS, since there's no separate replication lag. The gap is if EFS itself has a corrupt or bad-write scenario — data loss there isn't protected by mere multi-AZ redundancy. I'd close that by enabling AWS Backup with automatic EFS backups for point-in-time recovery, and separately I'd push controller configuration into JCasC (Jenkins Configuration as Code) so config drift can be reconstructed from version-controlled YAML rather than depending solely on filesystem state.

---

## 2. Pipeline as Code

**Q: Freestyle jobs vs. Jenkinsfile/Pipeline — what's your default and why?**

> Pipeline, almost without exception, for anything beyond a one-off admin task. Jenkinsfile is version-controlled alongside the code it builds, so pipeline changes go through the same PR review as application changes, and I get history/blame on the CI logic itself. Freestyle jobs live only in Jenkins' own config XML — no code review trail, harder to reproduce, easy for someone to hand-edit through the UI and silently diverge from what's documented. The only place I'd still reach for freestyle is a genuinely simple, rarely-touched utility job where the overhead of a Jenkinsfile isn't worth it.

**Q: Declarative vs Scripted pipeline?**

> Declarative by default — it enforces structure (`stages`, `steps`, `post`), which makes pipelines more readable and easier for other engineers to modify without deep Groovy knowledge, and it has better built-in validation and restart-from-stage support. I drop into Scripted syntax inside a `script {}` block when I need real conditional logic or looping that declarative's DSL doesn't cleanly express — for example, dynamically generating parallel stages from a list of services in a monorepo.

**Q: How do you avoid duplicating pipeline logic across dozens of repos?**

> Shared Libraries. I define common steps — build-and-push-to-ECR, standard notification blocks, security scan gates — in a separate Git repo loaded via `@Library` in each Jenkinsfile. That way a fix or a new compliance step (say, adding a Trivy scan) is a one-line version bump across consuming pipelines rather than editing 40 individual Jenkinsfiles. I version the library (tags, not just `master`) so consuming teams can pin and upgrade deliberately instead of an unreviewed change breaking every pipeline at once.

---

## 3. Agents & Scaling

**Q: How are your agents provisioned and connected?**

> Static RHEL EC2 instances registered as SSH agents — the controller holds a dedicated ed25519 keypair, the public key is distributed to each agent's `jenkins` user `authorized_keys`, and nodes are defined in Jenkins with labels so pipelines target them by capability (`rhel`, `docker`, `terraform`) rather than by hostname. That label-based routing is what lets me add or remove agent capacity without touching pipeline definitions.

**Q: Static agents vs. dynamic/ephemeral agents (e.g., Kubernetes plugin) — which would you choose and when?**

> Static SSH agents are simpler to reason about and fine for steady, predictable load — no cold-start latency, and I already have a known-good RHEL image. But they're wasteful if build volume is spiky, and there's drift risk if builds leave stale state on a long-lived agent.
>
> Given I already run EKS clusters, I'd lean toward the Kubernetes plugin for agents in a scale-out scenario: each build gets a fresh pod from a defined pod template, so there's no cross-build contamination, and it scales to zero when idle, which static EC2 agents can't do without extra ASG scale-in/out logic. The tradeoff is added complexity — pod template maintenance, image build pipelines for agent images, and making sure the controller has network line-of-sight and RBAC into the cluster. For this specific EC2-based setup, I chose SSH agents because the build load is steady and I wanted to keep the moving parts minimal for the interview scenario — in a real scale-out environment I'd push toward ephemeral K8s agents.

**Q: A build is stuck queued and never picks up an agent. Walk me through your triage.**

> First, *Manage Jenkins → Nodes* — is the target agent online, or does it show a red X? If offline, I SSH to the agent directly and check `systemctl status` isn't relevant here since it's just an SSH-launched agent process, so I check disk space (`df -h`) — a full disk is the single most common cause of an agent silently failing to accept work. Then I check the label — did someone typo the label in the Jenkinsfile, or does no online agent actually carry that label? If the agent's online and labeled correctly, I check executor count on that node; if all executors are already consumed by other running jobs, the build is queued legitimately and I need more agents or more executors per agent, not a bug fix.

---

## 4. Security & Credentials

**Q: How do you manage credentials in Jenkins — anything hardcoded in Jenkinsfiles?**

> Never hardcoded. Everything goes through the Credentials plugin — SSH keys, AWS access via the AWS Credentials plugin (though I prefer instance profile/IAM roles on the controller and agents over static keys wherever possible), and secrets pulled at runtime with `withCredentials { }` so they're masked in console output and scoped to just the steps that need them. Where I already have Vault in the stack, I use the Vault plugin instead of Jenkins' own credential store as the source of truth, so secret rotation and audit logging live in one place rather than being split across Jenkins and Vault.

**Q: How would you lock down who can do what in Jenkins for a multi-team org?**

> Role-based Authorization Strategy (or Matrix-based for smaller setups) over the default "logged-in users can do anything." I'd define roles like `dev-team-a-job-runner` scoped to specific folder/project patterns rather than global admin, and keep global admin to a small SRE/platform group. Folders (via the CloudBees Folders plugin or Job DSL-managed structure) let me apply per-team credential scoping too, so team A's pipelines can't read team B's secrets even though they share a controller.

**Q: What's your approach to keeping Jenkins itself patched without breaking pipelines?**

> I don't apply plugin or core updates directly against the production controller's `JENKINS_HOME`. Given the EFS-backed setup, I can snapshot/clone the EFS filesystem, spin up a throwaway controller instance pointed at the clone, apply the update there, and run the critical pipelines against it before promoting. It's slightly more overhead than clicking "update" in the UI, but a bad plugin upgrade taking down the only controller — especially one that's supposed to be your HA-recoverable single instance — is a worse outage than the time spent validating first.

---

## 5. Scaling for a Large Org

**Q: If build volume outgrew this single controller, what's your scaling path?**

> First lever is agent capacity, not the controller — most load is build execution, not scheduling overhead, so I'd add agents (or move to ephemeral K8s agents) well before touching the controller. If the controller itself becomes the bottleneck — high job count causing UI/API slowness, heavy plugin load — the standard next step in OSS Jenkins isn't horizontal scaling of one controller, it's splitting into multiple controllers by team or domain (e.g., a controller per business unit), optionally federated under CloudBees Operations Center if centralized visibility matters. I'd avoid trying to force one controller to scale indefinitely; it fights the tool's actual architecture.

**Q: How do you think about Jenkins vs. a SaaS CI like GitHub Actions or GitLab CI in a senior-level tradeoff discussion?**

> Jenkins wins on flexibility — plugin ecosystem, arbitrary agent environments (including on-prem/kubeadm clusters like mine), and no vendor lock-in for build execution. It costs you operational burden: you own patching, HA, scaling, and plugin compatibility, which is everything covered above. SaaS CI trades that operational ownership for less control over the execution environment and potential cost-at-scale for compute minutes. In practice the decision usually comes down to whether the org already has the platform engineering capacity to run Jenkins well — if not, the "free" flexibility of Jenkins is paid for later in incident response time.

---

## Likely follow-up probes to prep for

- "Show me the actual Jenkinsfile you'd write for X" — have a real declarative pipeline example ready (checkout → build → test → security scan → deploy stage with `when { branch 'main' }`).
- "What happens to running builds when the controller ASG replaces the instance?" — be honest: in-flight builds on the old controller are lost/orphaned unless you've configured graceful drain before termination; this is a real gap worth naming rather than glossing over.
- "How do you test a Jenkinsfile change without breaking the pipeline for everyone?" — Replay feature, or a feature-branch pipeline against a duplicate/sandbox job before merging to the library.
