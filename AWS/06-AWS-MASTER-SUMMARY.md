---
type: study-note
subject: AWS-06-AWS-MASTER-SUMMARY
category: devops
status: active
---

# 🧾 Master AWS CLI Cheat Sheet

```bash
# ── Identity & Auth ─────────────────────────────────────────
aws sts get-caller-identity                     # who am I?
aws configure list                              # active profile/creds

# ── EC2 ──────────────────────────────────────────────────────
aws ec2 describe-instances --filters "Name=instance-state-name,Values=running"
aws ec2 start-instances --instance-ids i-0abc123
aws ec2 create-snapshot --volume-id vol-0abc123

# ── S3 ───────────────────────────────────────────────────────
aws s3 ls s3://my-bucket/
aws s3 cp ./file.txt s3://my-bucket/path/
aws s3 sync ./dist s3://my-bucket/ --delete

# ── ECR / ECS ────────────────────────────────────────────────
aws ecr get-login-password | docker login --username AWS --password-stdin <acct>.dkr.ecr.<region>.amazonaws.com
aws ecs update-service --cluster my-cluster --service my-svc --force-new-deployment

# ── EKS ──────────────────────────────────────────────────────
aws eks update-kubeconfig --name my-cluster --region eu-west-1
eksctl create nodegroup --cluster my-cluster --managed
kubectl get nodes    # standard kubectl works once kubeconfig is updated

# ── CloudFormation ───────────────────────────────────────────
aws cloudformation deploy --template-file template.yaml --stack-name my-stack
aws cloudformation describe-stack-events --stack-name my-stack

# ── CloudWatch Logs ──────────────────────────────────────────
aws logs tail /ecs/vote --follow

# ── IAM ──────────────────────────────────────────────────────
aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names s3:GetObject
```

---

# 🎯 Final Cross-Domain Rapid-Fire

> [!question]- What's the single mental model that ties EC2, ECS, EKS, and Lambda together on the "who manages what" spectrum?
> They form a ladder of decreasing operational responsibility: EC2 (you manage OS+runtime), ECS/EKS-on-EC2 (you manage nodes, AWS/K8s manages scheduling), ECS/EKS-on-Fargate (AWS manages compute entirely, you manage container spec), Lambda (AWS manages everything except your code) — the Shared Responsibility Model sliding further toward AWS at each rung, and each rung trading flexibility for reduced ops burden.

> [!question]- A production incident: users report intermittent 502s from an ALB right after a deploy. What are the two most likely AWS-specific causes to check first?
> (1) **Target Group health checks** failing on the new targets — misconfigured health check path/threshold pulling healthy-but-not-yet-ready instances out of rotation too aggressively or not fast enough; (2) **deployment strategy timing** — if using an in-place/rolling ECS or ASG deployment without proper connection draining, old targets may be terminated before in-flight requests complete.

> [!question]- Name the AWS-native equivalent for each Kubernetes concept: ReplicaSet, PersistentVolume (RWO), PersistentVolume (RWX), Service DNS discovery, ConfigMap/Secret.
> ReplicaSet → Auto Scaling Group. PV (RWO) → EBS volume. PV (RWX) → EFS file system. Service DNS discovery → Route 53 (or Cloud Map for service discovery specifically). ConfigMap/Secret → Systems Manager Parameter Store / Secrets Manager.