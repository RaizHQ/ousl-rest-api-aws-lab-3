# OUSL Student API — EC2 Restart and Docker Deployment Guide

This guide explains how to bring the Spring Boot Student API back online after stopping and starting the EC2 instance, and how to deploy the container again from scratch if needed.

## Deployment details

| Item | Value |
| --- | --- |
| AWS Region | `ap-southeast-2` |
| EC2 instance | `ousl-docker-host-lab-3` |
| EC2 login user | `ec2-user` |
| ECR repository | `495533465913.dkr.ecr.ap-southeast-2.amazonaws.com/ousl-student-api-lab-3` |
| Previously working image tag | `087bb0a9d4f01de6c80c3d874a3acbf97685df00` |
| Docker container name | `ousl-student-api-lab-3` |
| Application port | `8080` |
| RDS endpoint | `ousl-postgres-db-lab3.cv6gg40sgio3.ap-southeast-2.rds.amazonaws.com` |
| RDS port | `5432` |
| Database name | `ousl_db` |
| Database user | `postgres` |

> **Important:** The image tag above is the one that was confirmed to start successfully **when `DB_HOST` and `DB_USER` were provided**. If GitHub Actions has since built an updated image with the RDS endpoint embedded in `application.yml`, use the new image tag instead. Do not store the RDS password in this README or commit it to GitHub.

## A. Normal procedure after stopping and starting EC2

### 1. Start the EC2 instance

1. Open **AWS Console → EC2 → Instances**.
2. Select `ousl-docker-host-lab-3`.
3. Choose **Instance state → Start instance**.
4. Wait until the instance is **Running** and its status checks pass.
5. Check the **Public IPv4 address**. It may change after a stop/start unless an Elastic IP is attached. Update your Postman URL accordingly.

### 2. Connect to EC2

Use the **Connect** button in the EC2 console (EC2 Instance Connect), or SSH using your existing key and the instance's current public IP:

```bash
ssh -i /path/to/your-key.pem ec2-user@YOUR_CURRENT_EC2_PUBLIC_IP
```

All following commands must run **on EC2** (the prompt should resemble `[ec2-user@ip-10-0-0-154 ~]$`), not in AWS CloudShell.

### 3. Check Docker and the existing container

```bash
sudo systemctl status docker --no-pager
sudo docker ps -a
```

If Docker is not running:

```bash
sudo systemctl enable --now docker
```

The container was created with `--restart unless-stopped`, so it should normally start automatically when Docker starts after an EC2 reboot, provided it was not manually stopped beforehand.

Check the status:

```bash
sudo docker ps
sudo docker logs --tail 60 ousl-student-api-lab-3
```

- If `ousl-student-api-lab-3` shows **Up**, go to **Section C** to test the API.
- If it shows **Exited** (and was manually stopped), run:

  ```bash
  sudo docker start ousl-student-api-lab-3
  ```

- If it shows **Restarting**, inspect the logs:

  ```bash
  sudo docker logs --tail 100 ousl-student-api-lab-3
  ```

- If there is **no container**, follow **Section B**.

> You do **not** need to re-enter `DB_PASS` when restarting the *same existing container*: Docker retains its original environment configuration. You do need to supply it when creating a **new** container.

## B. Deploy the container again from scratch

Use this section only if the container is missing, needs replacement, or you are deploying a newer image.

### 1. Confirm the EC2 IAM role

```bash
aws sts get-caller-identity
```

The instance should have the `EC2-ECR-ReadOnly-Lab3` IAM role (or another role with permission to pull from ECR). Do not configure permanent AWS access keys on EC2.

### 2. Authenticate Docker with ECR

```bash
aws ecr get-login-password --region ap-southeast-2 \
  | sudo docker login \
    --username AWS \
    --password-stdin \
    495533465913.dkr.ecr.ap-southeast-2.amazonaws.com
```

Expected: `Login Succeeded`.

### 3. Pull the image

The tag below is the previously working image. Replace it if you have a newer successful GitHub Actions build.

```bash
IMAGE='495533465913.dkr.ecr.ap-southeast-2.amazonaws.com/ousl-student-api-lab-3:087bb0a9d4f01de6c80c3d874a3acbf97685df00'
sudo docker pull "$IMAGE"
```

### 4. Remove the old container, if one exists

**Only do this when you intentionally want to replace the container.**

```bash
sudo docker rm -f ousl-student-api-lab-3
```

If it says the container does not exist, you can continue.

### 5. Enter the database password without putting it in shell history

```bash
read -rsp 'Enter RDS password: ' DB_PASS
echo
export DB_PASS
```

### 6. Create and start the container

**Option 1 — Previously confirmed working image:** The old image requires the RDS hostname at runtime. This is the command that successfully started Spring Boot and connected to PostgreSQL in the EC2 logs:

```bash
sudo docker run -d \
  --name ousl-student-api-lab-3 \
  --restart unless-stopped \
  -p 8080:8080 \
  -e DB_HOST=ousl-postgres-db-lab3.cv6gg40sgio3.ap-southeast-2.rds.amazonaws.com \
  -e DB_USER=postgres \
  -e DB_PASS \
  "$IMAGE"
```

**Option 2 — New image with the endpoint directly in `application.yml`:** Use this simpler command **only after verifying** that the newly built image contains the correct JDBC URL and reads the password from `${DB_PASS}`:

```bash
sudo docker run -d \
  --name ousl-student-api-lab-3 \
  --restart unless-stopped \
  -p 8080:8080 \
  -e DB_PASS \
  "$IMAGE"
```

Then remove the password from your current shell:

```bash
unset DB_PASS
```

**Security note:** Docker environment variables can be viewed by users with sufficient Docker/host privileges. For production, use a managed secret solution such as AWS Secrets Manager. The ECR login may also warn that its token is stored in `/root/.docker/config.json`; consider the Amazon ECR Docker credential helper for longer-term use.

## C. Verify that the application is running

### 1. Container status and startup logs

```bash
sudo docker ps -a
sudo docker logs --tail 80 ousl-student-api-lab-3
```

Healthy startup logs should contain messages similar to:

```text
HikariPool-1 - Start completed.
Tomcat started on port 8080 (http)
Started StudentApplication
```

### 2. Test locally on EC2

```bash
curl http://localhost:8080/actuator/health
curl http://localhost:8080/api/students
```

The health endpoint should return something like `{"status":"UP"}`. If the students table is empty, the second endpoint may return `[]`.

### 3. Test from Postman on your computer

Use the **current EC2 public IP** (check it again after each EC2 stop/start):

```text
GET http://YOUR_CURRENT_EC2_PUBLIC_IP:8080/api/students
POST http://YOUR_CURRENT_EC2_PUBLIC_IP:8080/api/students
```

For POST, select **Body → raw → JSON**, with header `Content-Type: application/json`:

```json
{
  "name": "Test Student",
  "course": "Software Engineering",
  "address": "Colombo"
}
```

Make sure the EC2 security group allows inbound TCP `8080` from the IP addresses that need to access the API. For a real deployment, use HTTPS through a properly configured load balancer or reverse proxy instead of exposing an unencrypted public HTTP API.

## D. Common problems

| Symptom | What to check |
| --- | --- |
| `no basic auth credentials` when pulling | Repeat the ECR login in Section B. |
| `Unable to locate credentials` | Check the EC2 IAM role with `aws sts get-caller-identity`. |
| `Connection refused` for PostgreSQL | Check the JDBC hostname; an old image may default to `localhost`. Use Option 1 in Section B or rebuild with the correct `application.yml`. |
| `password authentication failed` | Check the RDS username and password. Do not paste passwords into chat or logs. |
| `Connection timed out` | Test EC2-to-RDS connectivity and verify security groups, VPC routing, and network ACLs. |
| `curl: (7) Failed to connect` | Check `sudo docker ps -a` and `sudo docker logs`; the application may still be starting or may have crashed. |
| Postman cannot connect, but local `curl` works | Check the current EC2 public IP and inbound port `8080` security-group rule. |
| `docker: command not found` or Docker daemon unavailable | Check Docker installation and `sudo systemctl enable --now docker`. |

To test EC2-to-RDS TCP connectivity without installing `nc`:

```bash
timeout 5 bash -c 'echo > /dev/tcp/ousl-postgres-db-lab3.cv6gg40sgio3.ap-southeast-2.rds.amazonaws.com/5432' \
  && echo 'SUCCESS: RDS reachable' \
  || echo 'FAILED: RDS unreachable'
```

## E. Useful Docker commands

```bash
# Show all containers
sudo docker ps -a

# View recent application logs
sudo docker logs --tail 100 ousl-student-api-lab-3

# Follow live logs (Ctrl+C exits log view)
sudo docker logs -f ousl-student-api-lab-3

# Restart the existing container
sudo docker restart ousl-student-api-lab-3

# Stop the container manually
sudo docker stop ousl-student-api-lab-3

# Start a manually stopped container
sudo docker start ousl-student-api-lab-3
```

## F. Recommended permanent `application.yml`

For future builds, keep the RDS endpoint in your Spring Boot configuration and read the password from an environment variable:

```yaml
spring:
  datasource:
    url: jdbc:postgresql://ousl-postgres-db-lab3.cv6gg40sgio3.ap-southeast-2.rds.amazonaws.com:5432/ousl_db?sslmode=require
    username: postgres
    password: ${DB_PASS}
    driver-class-name: org.postgresql.Driver
```

Commit and push the application change, wait for GitHub Actions to push a **new** ECR image, and redeploy that new image using **Option 2** in Section B.

**Remember:** An EC2 stop/start does not normally delete Docker containers or the EC2 root EBS volume. RDS remains a separate database service, so your data should persist independently of the EC2 instance. An EC2 public IP can change after a stop/start.
