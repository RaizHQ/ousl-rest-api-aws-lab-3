# Restart and Verify the OUSL Student API on EC2

This guide follows the successful command sequence used on **Amazon Linux 2023 EC2** to launch the Dockerized Spring Boot Student REST API and verify its connection to **Amazon RDS PostgreSQL**.

## Prerequisites

- Connect to the EC2 instance (`ousl-docker-host-lab-3`) using SSH.
- Ensure the EC2 instance is running, Docker is installed and running, and the RDS PostgreSQL instance is available.
- The Docker image must already be present on EC2. If not, use the ECR login and pull commands in **Optional: Pull the image again** below.
- Have the RDS database password ready. Do not paste it into the command line or commit it to Git.

> **Note:** This procedure creates a container if one does not already exist. If an existing container is running, do not run `docker run` again with the same name. Use `sudo docker ps -a` to check first.

## Step 1 — Check existing Docker containers

```bash
sudo docker ps -a
```

In the recorded session, the list was empty, so a new container was created. If `ousl-student-api-lab-3` already exists but is stopped, start it instead:

```bash
sudo docker start ousl-student-api-lab-3
```

Then skip to Step 5. With `--restart unless-stopped`, a previously running container normally starts automatically when Docker starts after an EC2 reboot/start.

## Step 2 — Enter the RDS password securely

```bash
read -rsp "Enter RDS password: " DB_PASS
echo
export DB_PASS
```

The password is not echoed to the terminal. The environment variable is used only when creating the container. Note that Docker environment variables are still visible to users with sufficient Docker privileges.

## Step 3 — Create and run the Docker container

```bash
sudo docker run -d \
  --name ousl-student-api-lab-3 \
  --restart unless-stopped \
  -p 8080:8080 \
  -e DB_HOST=ousl-postgres-db-lab3.cv6gg40sgio3.ap-southeast-2.rds.amazonaws.com \
  -e DB_USER=postgres \
  -e DB_PASS \
  495533465913.dkr.ecr.ap-southeast-2.amazonaws.com/ousl-student-api-lab-3:087bb0a9d4f01de6c80c3d874a3acbf97685df00
```

This matches the image and configuration from the successful deployment. If you build a newer image, replace the tag accordingly. `DB_HOST` is explicitly provided because this image required the RDS hostname at runtime.

## Step 4 — Clear the temporary shell variable

```bash
unset DB_PASS
```

## Step 5 — Confirm the container is running

```bash
sudo docker ps -a
```

Expected: container `ousl-student-api-lab-3` has status `Up`, with port mapping `0.0.0.0:8080->8080/tcp`.

## Step 6 — Inspect Spring Boot startup logs

```bash
sudo docker logs --tail 60 ousl-student-api-lab-3
```

Confirm that the logs show:

```text
HikariPool-1 - Added connection
HikariPool-1 - Start completed.
Tomcat started on port 8080 (http)
Started StudentApplication
```

These messages confirm successful application startup and PostgreSQL connectivity.

## Step 7 — Check the Actuator health endpoint

```bash
curl http://localhost:8080/actuator/health
```

The successful response contained overall `"status":"UP"` and database component `"status":"UP"`. The exact JSON also includes disk space, liveness, readiness, and other health details.

## Step 8 — Test the Student REST API

```bash
curl http://localhost:8080/api/students
```

Example response observed during the lab:

```json
[
  {"name":"Dhilhara","course":"Neural Networks","address":"Colombo","id":1},
  {"name":"Adhil","course":"AI","address":"Kandy","id":2},
  {"name":"Adhil","course":"AI","address":"Kandy","id":3}
]
```

The actual response depends on the records currently stored in RDS.

## Optional — Pull the image again if it is missing

The EC2 instance needs an IAM role with ECR pull permissions (the lab used `EC2-ECR-ReadOnly-Lab3`). Verify the role:

```bash
aws sts get-caller-identity
```

Log in to ECR:

```bash
aws ecr get-login-password --region ap-southeast-2 \
  | sudo docker login \
    --username AWS \
    --password-stdin \
    495533465913.dkr.ecr.ap-southeast-2.amazonaws.com
```

Pull the image:

```bash
sudo docker pull \
  495533465913.dkr.ecr.ap-southeast-2.amazonaws.com/ousl-student-api-lab-3:087bb0a9d4f01de6c80c3d874a3acbf97685df00
```

Then return to Step 2.

## Troubleshooting

- **Container already exists:** Run `sudo docker ps -a`. Use `sudo docker start ousl-student-api-lab-3` for a stopped container; only remove and recreate it if you need to change its configuration.
- **Container keeps restarting:** Check `sudo docker logs --tail 100 ousl-student-api-lab-3`. Confirm the RDS instance is available and the hostname/password are correct.
- **Cannot connect to the API:** Test `curl http://localhost:8080/actuator/health` on EC2 first. For external access, confirm EC2's current public IP and the inbound TCP 8080 security-group rule.
- **After stopping and starting EC2:** First check `sudo docker ps -a`. If the container is already `Up`, skip container creation and go directly to the health and API tests.

**Security reminder:** The original lab security group allowed SSH from `0.0.0.0/0`. Restrict port 22 to your own trusted IP whenever possible. For a production deployment, use AWS Secrets Manager or another managed secret mechanism rather than long-lived database passwords in container environment variables.
