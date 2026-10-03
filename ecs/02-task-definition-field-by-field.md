# Module 02 — The Task Definition, Field by Field

← [All tutorials](../README.md) · **ECS tutorial**, module 2 of 6

Most ECS problems trace back to the **task definition**: wrong sizing, the wrong role, a secret that can't be read, a missing log setting. This module takes one realistic task definition for the `shop-api` service and explains **every field**.

---

## 1. A complete task definition

`shop-api` is an HTTP API on port 8080. It reads uploads from S3, gets its database password from Secrets Manager, and ships traces through a small collector container running beside it (a **sidecar**).

```json
{
  "family": "shop-api",
  "requiresCompatibilities": ["FARGATE"],
  "networkMode": "awsvpc",
  "cpu": "512",
  "memory": "1024",
  "runtimePlatform": { "cpuArchitecture": "ARM64", "operatingSystemFamily": "LINUX" },
  "executionRoleArn": "arn:aws:iam::111122223333:role/shop-api-execution",
  "taskRoleArn": "arn:aws:iam::111122223333:role/shop-api-task",
  "containerDefinitions": [
    {
      "name": "api",
      "image": "111122223333.dkr.ecr.eu-central-1.amazonaws.com/shop-api:1.4.2",
      "essential": true,
      "portMappings": [{ "containerPort": 8080, "protocol": "tcp", "name": "http" }],
      "environment": [
        { "name": "APP_ENV",   "value": "prod" },
        { "name": "S3_BUCKET", "value": "shop-uploads" }
      ],
      "secrets": [
        { "name": "DB_PASSWORD",
          "valueFrom": "arn:aws:secretsmanager:eu-central-1:111122223333:secret:shop/db-AbCdEf:password::" }
      ],
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"],
        "interval": 15, "timeout": 5, "retries": 3, "startPeriod": 30
      },
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/shop-api",
          "awslogs-region": "eu-central-1",
          "awslogs-stream-prefix": "api",
          "mode": "non-blocking"
        }
      },
      "dependsOn": [{ "containerName": "otel-collector", "condition": "START" }],
      "stopTimeout": 30,
      "user": "1000",
      "readonlyRootFilesystem": true,
      "mountPoints": [{ "sourceVolume": "scratch", "containerPath": "/tmp" }],
      "linuxParameters": { "initProcessEnabled": true }
    },
    {
      "name": "otel-collector",
      "image": "public.ecr.aws/aws-observability/aws-otel-collector:latest",
      "essential": false,
      "memoryReservation": 128
    }
  ],
  "volumes": [{ "name": "scratch" }]
}
```

---

## 2. Task-level fields

### `"family": "shop-api"`
The task definition's name. Every time you register a changed version, ECS creates a new **revision**: `shop-api:1`, `shop-api:2`, and so on. Revisions are **immutable**. To change anything, you register a new revision and point the service at it (Module 05).

### `"requiresCompatibilities": ["FARGATE"]`
Which kind of capacity this definition is valid for: `FARGATE`, `EC2`, `MANAGED_INSTANCES`, or `EXTERNAL` (your own servers). ECS validates the rest of the definition against it. Fargate, for example, requires `awsvpc` networking and task-level CPU and memory. You can list several.

### `"networkMode": "awsvpc"`
How containers get networking. With **`awsvpc`**, every task gets **its own network interface and private IP** in your subnet, with **its own security groups**. That's the only mode on Fargate, and the recommended one everywhere. Module 03 covers the alternatives (`bridge`, `host`) used on EC2.

### `"cpu": "512"` and `"memory": "1024"`
The task's total size: **512 CPU units = 0.5 vCPU** (1024 units = 1 vCPU), and **1024 MiB** of memory. On Fargate these are **required**, they decide the price, and they must be one of these combinations:

| CPU | Allowed memory |
|---|---|
| 256 (0.25 vCPU) | 512 MiB, 1 GB, 2 GB |
| 512 (0.5 vCPU) | 1–4 GB |
| 1024 (1 vCPU) | 2–8 GB |
| 2048 (2 vCPU) | 4–16 GB |
| 4096 (4 vCPU) | 8–30 GB |
| 8192 (8 vCPU) | 16–60 GB, in 4 GB steps |
| 16384 (16 vCPU) | 32–120 GB, in 8 GB steps |

Start small, watch the real usage (Module 06), then adjust.

### `"runtimePlatform"`
The OS (`LINUX` or `WINDOWS_SERVER_…`) and CPU architecture (`X86_64` or `ARM64`). **ARM64 (AWS Graviton) is about 20% cheaper** on Fargate, if your image is built for ARM (multi-arch images work everywhere).

### `"executionRoleArn"` and `"taskRoleArn"`: two different roles

This is the most common source of confusion in ECS. (How roles work in general: [IAM tutorial, Module 02](../iam/02-how-services-use-iam.md).)

| | **Task execution role** | **Task role** |
|---|---|---|
| Used by | **ECS itself**, **before** your code starts | **Your code**, **while** it runs |
| For | Pulling the image from ECR, writing to CloudWatch Logs, reading the `secrets` listed in the definition | Whatever your app calls: S3, DynamoDB, SQS… |
| Typical permissions | The AWS managed policy `AmazonECSTaskExecutionRolePolicy`, plus `secretsmanager:GetSecretValue` (and `kms:Decrypt`) for your secrets | A least-privilege custom policy, e.g. `s3:GetObject` on `shop-uploads/*` |
| When it's wrong | The task **never starts**: `CannotPullContainerError` or `ResourceInitializationError` | The task runs, but the app gets **AccessDenied** |

Both roles trust the service principal `ecs-tasks.amazonaws.com`. Your SDK finds the task role's credentials automatically, with no configuration needed.

### `"volumes": [{ "name": "scratch" }]`
Storage the containers can mount. A volume with only a name is **task storage**: it lives as long as the task, and only its containers share it. Here it gives the read-only container a writable `/tmp`. Other options:

| Volume | Lives | Shared between tasks? | Use for |
|---|---|---|---|
| Task storage (as above) | As long as the task | No | Scratch space. Fargate gives each task 20 GB, expandable to 200 GB |
| **Amazon EFS** | Permanently | **Yes**, across tasks and AZs | Shared files (uploads, content) |
| **Amazon EBS** (attached at deployment) | Per task, can be restored from a snapshot | No | Data that needs a fast disk per task |

Keep real state in a database or S3, and treat containers as disposable. The [Storage tutorial](../storage/01-storage-basics-and-s3.md) helps you choose.

---

## 3. Container-level fields

### `"name": "api"` and `"image"`
The container's name, which other fields refer to, and its image. An ECR image URI reads `<account>.dkr.ecr.<region>.amazonaws.com/<repository>:<tag>`. **Deploy specific tags** (`1.4.2`) or digests (`@sha256:…`), never `:latest`. With `latest`, two tasks of the same revision can end up running different code.

### `"essential": true`
If an **essential** container stops, ECS stops the **whole task** (and the service replaces it). Make the main app essential. Helpers like the collector here are usually `essential: false`, so a crashing helper doesn't take the app down.

### `"portMappings"`
The port the container listens on. With `awsvpc`, it's reachable on the task's own IP at that port, and that's what the load balancer targets (Module 03). The optional `name` is used by Service Connect (Module 03).

### `"environment"` vs `"secrets"`
- **`environment`**: plain values. They're visible to anyone who can read the task definition, so **never put passwords here**.
- **`secrets`**: ECS reads the value from **Secrets Manager** or **SSM Parameter Store** (using the **execution role**) and injects it as an environment variable when the task starts. The suffix `:password::` picks the `password` key out of a JSON secret.
- Secrets are read **only at start**. After you rotate a password, **redeploy** (Module 05) so new tasks pick it up.

### `"healthCheck"`
A command Docker runs **inside** the container: `interval` seconds apart, failing after `timeout` seconds, unhealthy after `retries` failures. `startPeriod` gives the app time to boot before failures count. An unhealthy essential container gets the task replaced. The tool you call (`curl`, `wget`) must exist in the image.

### `"logConfiguration"`
Where container output (stdout/stderr) goes. The `awslogs` driver sends it to a CloudWatch Logs **group**, with one **stream** per container named `<prefix>/<container>/<task-id>`. `"mode": "non-blocking"` prevents a slow log path from freezing your app. The log group must exist, and the execution role writes to it. (For other destinations, a **FireLens** log-router sidecar can forward logs anywhere.)

### `"dependsOn"`
Start order inside the task. Conditions are `START`, `COMPLETE` (exited), `SUCCESS` (exited with 0), and `HEALTHY`. Here the API waits until the collector has started.

### `"stopTimeout": 30`
When a task stops, each container first gets **SIGTERM**, and then **SIGKILL** after this many seconds (Fargate allows up to 120). Your app should finish in-flight requests and exit within this window (Module 05).

### `"user"`, `"readonlyRootFilesystem"`, `"linuxParameters.initProcessEnabled"`
Hardening:
- Run as a **non-root user**.
- Make the container's filesystem **read-only** (writes go to mounted volumes such as `/tmp` above).
- Add a tiny **init process** that forwards signals to your app and cleans up zombie processes. It's also needed for ECS Exec (Module 06).

### `"memoryReservation": 128` (collector) and container limits
Inside the task's 1 GB, you can give containers a soft reservation (`memoryReservation`) or a **hard** limit (`memory`). A container that exceeds a hard limit is killed with exit code **137** (`OutOfMemoryError`).

---

## 4. Try it: a fuller task definition (revision 2)

Add a **task role** for the app (with the permissions ECS Exec needs, used in Module 06), a health check, an environment variable, and a stop timeout. Then register it as revision 2 of `lab-web`:

```bash
aws iam create-role --role-name lab-ecsTaskRole --assume-role-policy-document file:///tmp/ecs-tasks-trust.json >/dev/null
cat > /tmp/ecs-exec-policy.json <<'EOF'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Action":["ssmmessages:CreateControlChannel","ssmmessages:CreateDataChannel","ssmmessages:OpenControlChannel","ssmmessages:OpenDataChannel"],"Resource":"*"}]}
EOF
aws iam put-role-policy --role-name lab-ecsTaskRole --policy-name ecs-exec --policy-document file:///tmp/ecs-exec-policy.json
TASK_ROLE_ARN=$(aws iam get-role --role-name lab-ecsTaskRole --query Role.Arn --output text); save TASK_ROLE_ARN

cat > /tmp/taskdef.json <<EOF
{
  "family": "lab-web",
  "requiresCompatibilities": ["FARGATE", "EC2"],
  "networkMode": "awsvpc",
  "cpu": "256", "memory": "512",
  "runtimePlatform": { "cpuArchitecture": "X86_64", "operatingSystemFamily": "LINUX" },
  "executionRoleArn": "$EXEC_ROLE_ARN",
  "taskRoleArn": "$TASK_ROLE_ARN",
  "containerDefinitions": [{
    "name": "web",
    "image": "public.ecr.aws/docker/library/nginx:stable",
    "essential": true,
    "portMappings": [{ "containerPort": 80, "protocol": "tcp" }],
    "environment": [{ "name": "APP_ENV", "value": "lab" }],
    "healthCheck": { "command": ["CMD-SHELL", "curl -f http://localhost/ || exit 1"],
                     "interval": 15, "timeout": 5, "retries": 3, "startPeriod": 10 },
    "linuxParameters": { "initProcessEnabled": true },
    "stopTimeout": 20,
    "logConfiguration": { "logDriver": "awslogs",
      "options": { "awslogs-group": "/ecs/lab-web", "awslogs-region": "$AWS_REGION",
                   "awslogs-stream-prefix": "web", "mode": "non-blocking" } }
  }]
}
EOF
aws ecs register-task-definition --cli-input-json file:///tmp/taskdef.json --query 'taskDefinition.[family,revision]' --output text   # lab-web 2
aws ecs list-task-definitions --family-prefix lab-web --query taskDefinitionArns
aws ecs describe-task-definition --task-definition lab-web:2 \
  --query 'taskDefinition.{cpu:cpu,memory:memory,execRole:executionRoleArn,taskRole:taskRoleArn,health:containerDefinitions[0].healthCheck.command}'
```

Revision 1 still exists, unchanged. Revisions are immutable, and services choose which one to run.

---

## Check yourself

<details><summary>The task fails with "ResourceInitializationError: unable to pull secrets". Which role do you fix?</summary>The task execution role (secretsmanager:GetSecretValue / ssm:GetParameters, plus kms:Decrypt). Also check the network path to Secrets Manager (Module 03).</details>
<details><summary>The app gets AccessDenied when it calls S3. Which role?</summary>The task role.</details>
<details><summary>You rotated the database password. Do running tasks see the new value?</summary>No. Secrets are injected at start, so redeploy the service.</details>
<details><summary>A container hits its memory limit. What happens?</summary>It's killed (exit 137, OutOfMemoryError). If it's essential, the task stops and the service replaces it.</details>

---
**Previous:** [Module 01](01-what-ecs-is-and-your-first-task.md) · **Next:** [Module 03 — Services, Networking & Load Balancing](03-services-networking-and-load-balancing.md)
