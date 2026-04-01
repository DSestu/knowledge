# AWS miscellaneous snippets

# aws-vault with MFA

aws-vault stores IAM credentials securely in your OS keychain and handles MFA token prompts automatically.

## Install

```fish
# Linux
sudo apt install aws-vault

# macOS
brew install aws-vault
```

## Add your IAM credentials

```fish
aws-vault add <vault-name>
```

It will prompt for your `AWS Access Key ID` and `AWS Secret Access Key`.

## Configure MFA in ~/.aws/config

Add your profile with the MFA device ARN:

```ini
[profile <vault-name>]
mfa_serial = arn:aws:iam::<account-id>:mfa/<iam-username>
region = <your-region>
```

You can find your MFA ARN in the AWS console under **IAM → Users → your user → Security credentials → MFA devices**.

## Execute a command with MFA

```fish
aws-vault exec <vault-name> -- aws sts get-caller-identity
```

It will prompt for your current MFA token code, then run the command with temporary credentials.

To open an authenticated shell session:

```fish
aws-vault exec <vault-name> -- fish
```

---

# Export connections from AWS (ECS) to local

## Method 1: via SSM Parameter Store (simplest)

If Airflow is configured with SSM as the secrets backend (`SystemsManagerParameterStoreBackend`), connections are stored directly in SSM — no ECS Exec needed. Fetch them from your local machine:

```fish
aws ssm get-parameters-by-path --path /airflow/connections --recursive --with-decryption
```

Then recreate them locally via `airflow connections add` or `AIRFLOW_CONN_*` env vars.

## Method 2: via ECS Exec

When your Airflow is dockerized and deployed on AWS ECS, you can use ECS Exec to run commands inside the container.

## Find your cluster and task

```fish
# List all ECS clusters
aws ecs list-clusters

# List tasks and their container names in one command
read -P "Cluster name: " cluster
aws ecs describe-tasks --cluster $cluster --tasks (aws ecs list-tasks --cluster $cluster | jq -r '.taskArns[]') --query 'tasks[].{Task:taskArn,Containers:containers[].name}'
```

The output maps each **task ARN** to its **container names**:
- **Task** = a running instance (like a pod). One task can run multiple containers.
- **Container** = a named process inside a task (e.g. `airflow-webserver`, `log-router`).

The `--task` parameter takes the full ARN (e.g. `arn:aws:ecs:eu-west-1:123456:task/my-cluster/abc123`), and `--container` takes the short name (e.g. `airflow-webserver`).

For exporting connections, target the **webserver** or **scheduler** — they are always running and have the full Airflow config. Avoid workers as they are ephemeral.

## Test that ECS Exec works

```fish
read -P "Cluster name: " cluster; read -P "Task ID: " task; read -P "Container name: " container
aws ecs execute-command --cluster $cluster --task $task --container $container --interactive --command "echo hello"
```

If it prints `hello`, ECS Exec is enabled. If you get an error about `executeCommandAgent`, enable it in Terraform and redeploy (see below).

### Enable ECS Exec via Terraform

Add to the ECS service module in Terraform and redeploy:

```hcl
enable_execute_command = true
```

The `sencrop/terraform-modules//ecs-fargate-service` module supports this variable and automatically attaches the required `ssmmessages` IAM policy to the task role. After applying, force a new deployment:

```fish
read -P "Cluster name: " cluster; read -P "Service name: " service
aws ecs update-service --cluster $cluster --service $service --force-new-deployment
```

## Export connections to S3

Since you can't `scp` directly from a Fargate container, use S3 as an intermediary:

```fish
read -P "Cluster name: " cluster; read -P "Task ID: " task; read -P "Container name: " container; read -P "S3 bucket: " bucket
aws ecs execute-command --cluster $cluster --task $task --container $container --interactive --command "airflow connections export - | aws s3 cp - s3://$bucket/connections.json"
```

## Download and import locally

```bash
aws s3 cp s3://<your-bucket>/connections.json ./connections.json
airflow connections import connections.json
```

### If local Airflow runs via docker-compose

Copy the file into the container and import from there:

```fish
docker compose cp connections.json airflow-webserver:/tmp/connections.json
docker compose exec airflow-webserver airflow connections import /tmp/connections.json
```

`airflow-webserver` is the service name as defined in your `docker-compose.yaml`.
