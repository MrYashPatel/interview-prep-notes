Great question—ECS Exec is the safest, fastest way to “SSH into a container” without ever touching the host. Here’s how to use it end-to-end in production and what to do when the task keeps crashing.

What ECS Exec is (and why)

What: An SSM-backed interactive shell (or command runner) directly into a running container in ECS (Fargate or EC2).

Why: No SSH keys, no open inbound ports, auditable via CloudTrail/SSM, works in private subnets.

0) One-time prerequisites (checklist)

Local

AWS CLI v2 (aws --version)

Session Manager Plugin (session-manager-plugin --version)

ECS Service/Task

Enable Exec on the service/task (details below).

IAM roles:

Task execution role needs SSM Messages permissions.

Your user/role that runs the command needs ecs:ExecuteCommand + SSM session perms.

Networking (if in private subnets): VPC endpoints for ssm, ssmmessages, and ec2messages (and logs if using CW Logs).

Logging: (recommended) Ship container logs to CloudWatch.

1) Enable Exec on your service
Console (quick)

ECS → Your Cluster → Services → your-service → Update → Enable execute command → Save/Deploy.

CLI
aws ecs update-service \
  --cluster my-cluster \
  --service my-service \
  --enable-execute-command


Optional (cluster-level config with KMS encryption of sessions):

aws ecs put-cluster-configuration \
  --cluster my-cluster \
  --execute-command-configuration "kmsKeyId=arn:aws:kms:...:key/...,logging=OVERRIDE,logConfiguration={cloudWatchLogGroupName=/ecs/exec,cloudWatchEncryptionEnabled=true}"

2) IAM you need (minimal examples)
A) Task execution role (attached to task definition)

Grants the container channels to SSM:

{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "ssmmessages:CreateControlChannel",
      "ssmmessages:CreateDataChannel",
      "ssmmessages:OpenControlChannel",
      "ssmmessages:OpenDataChannel"
    ],
    "Resource": "*"
  }]
}

B) Your human role/user (the person running execute-command)
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowECSExec",
    "Effect": "Allow",
    "Action": [
      "ecs:ExecuteCommand",
      "ssm:StartSession",
      "ssm:ResumeSession",
      "ssm:TerminateSession",
      "ssm:DescribeSessions",
      "ssm:GetConnectionStatus"
    ],
    "Resource": "*"
  }]
}


Tighten with conditions (recommended):

Limit ecs:cluster, ecs:task, and ecs:container-name via resource ARNs or condition keys.

Require MFA for production, etc.

3) Run it: attach to a running task

Find a task ID:

aws ecs list-tasks --cluster my-cluster --service-name my-service


Attach shell (try bash then sh):

aws ecs execute-command \
  --cluster my-cluster \
  --task arn:aws:ecs:region:acct:task/xyz \
  --container api \
  --interactive \
  --command "/bin/bash"


If bash not present:

... --command "/bin/sh"


Non-interactive (run a one-off command and return):

aws ecs execute-command \
  --cluster my-cluster \
  --task <task-arn> \
  --container api \
  --command "printenv | sort"

4) What to check once inside (crash triage playbook)

Environment: printenv (wrong DB URLs, missing secrets).

Filesystem & image: ls -al, check config files, build outputs.

App health: ps aux, lsof -i :PORT, curl -s localhost:PORT/health.

DNS/Network: nc -vz <host> <port> or curl -v to dependencies.

Permissions: file ownership, read/write paths, whoami.

Time/zone: date (JWT/STS drift issues).

Container logs: Prefer CloudWatch; from inside: tail -n 200 /proc/1/fd/1.

5) “It keeps crashing, I can’t exec!” (the key scenario)

You cannot exec into a stopped task. Use one of these:

A) Temporary command override to keep it alive

Run a one-off task with an override that sleeps:

aws ecs run-task \
  --cluster my-cluster \
  --launch-type FARGATE \
  --task-definition my-task:42 \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-abc],securityGroups=[sg-abc],assignPublicIp=ENABLED}" \
  --enable-execute-command \
  --overrides '{
    "containerOverrides": [{
      "name": "api",
      "command": ["sh","-lc","while true; do sleep 3600; done"]
    }]
  }'


Now execute-command into that sleeping container to inspect the environment/image.

B) Debug variant of the task definition

Clone your task def, change ENTRYPOINT/CMD to a sleep loop or a shell, deploy to a debug service with 0/1 task in a private env.

C) Reproduce locally

Run the same image/command with Docker and bind the same env vars/secrets to replicate.

6) Fargate vs EC2 specifics

Fargate

No hosts to manage. Exec works if service has Exec enabled + task execution role has SSM messages perms.

Ensure your private subnets can reach SSM endpoints (VPC endpoints or NAT).

EC2 launch type

Container instance must have a recent ECS Agent and SSM Agent.

The instance profile should include AmazonSSMManagedInstanceCore (or equivalent) so the agent can talk to SSM.

Keep instances patched; old agents can break Exec.

7) Security, auditing, and hygiene

Disable Exec when not needed in prod or restrict to break-glass roles.

Audit: CloudTrail logs ecs:ExecuteCommand, SSM session history records who/when.

KMS-encrypt session logs, send to CloudWatch (cluster execute-command configuration).

Least privilege on both the human role and the task execution role.

No shells in base images for high-security workloads; use non-interactive commands + sidecar debugging patterns.

8) Common errors & quick fixes
Symptom	Likely Cause	Fix
You must have permission to execute the command	Missing IAM on your user	Add ecs:ExecuteCommand + SSM session actions
ExecuteCommand failed: unable to reach SSM	No VPC endpoints/NAT	Add endpoints for ssm, ssmmessages, ec2messages
Session opens then drops	Task role lacks SSM messages	Add ssmmessages:*Channel actions to task execution role
bash: not found	Image is minimal	Use /bin/sh
Can’t exec (task not found or stopped)	Crash loop	Use command override/sleep trick above
Example: Full minimal flow (Fargate)
# 1) Enable exec on service
aws ecs update-service --cluster prod --service api --enable-execute-command

# 2) Get a running task
TASK_ARN=$(aws ecs list-tasks --cluster prod --service-name api --query 'taskArns[0]' --output text)

# 3) Exec into the container
aws ecs execute-command \
  --cluster prod \
  --task "$TASK_ARN" \
  --container api \
  --interactive \
  --command "/bin/sh"


If it’s crash-looping, run a temporary sleeping copy (see §5A), then exec into that.



---

Section 3 – Troubleshooting Playbook (Real Job Problems)
1️⃣ Tasks stuck in PENDING
Checks:

ecs:DescribeServices for events.

EC2 cluster CPU/mem capacity.

ECS agent logs /var/log/ecs/ecs-agent.log.

Task execution IAM role permissions.

2️⃣ ALB health check failing
Checks:

ALB health check path/port.

Security group allows ALB → task traffic.

App responds with 200 OK on health check.

3️⃣ High CPU cost in Fargate
Fixes:

Reduce vCPU in Task Definition.

Switch to EC2 Spot instances if possible.

4️⃣ Image pull failure
Checks:

ECR auth policy.

Image tag exists in ECR.

Network route from ECS task to ECR.

5️⃣ Logs missing in CloudWatch
Checks:

ECS log driver set to awslogs.

IAM permissions: logs:CreateLogStream, logs:PutLogEvents.

Section 4 – ECS Cost Optimization Cheatsheet
EC2 launch type → Use Spot + On-Demand mix with ASG.

Fargate → Right-size vCPU/memory, stop idle services.

Use CloudWatch alarms to detect underutilization.

Use Capacity Providers for automatic Spot instance usage.

Turn off unused ECS services in non-prod at night with Lambda.

