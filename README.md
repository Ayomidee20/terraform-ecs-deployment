# Automated Container Deployment on AWS with Terraform and GitHub Actions

This repository is a learning-focused, production-style DevOps project that demonstrates a modern workflow for building and deploying a simple Node.js API to AWS using **Docker**, **GitHub Actions**, **Amazon ECR**, **Terraform**, and **ECS Fargate**, fronted by an **Application Load Balancer**.

## Project overview

- **API**: Node.js (Express) service on port `3000`
- **Datastore**: Redis
  - Local dev: Redis container via Docker Compose
  - AWS: Redis runs as a *sidecar container* in the same ECS task (keeps the service list limited; easy to learn)
- **CI/CD**: GitHub Actions runs tests, builds Docker images, pushes to ECR, and deploys to ECS
- **IaC**: Terraform provisions a VPC (public subnets), ALB, ECR, ECS Fargate service, security groups, and CloudWatch logs

## Architecture

ASCII diagram (runtime traffic path):

```
Internet
  |
  v
Application Load Balancer (HTTP)
  |
  v
ECS Fargate Service
  |
  v
Task: [Node.js API container] <-> [Redis sidecar container]
  |
  v
CloudWatch Logs
```

Key points:
- **Public**: ALB and ECS tasks are deployed into public subnets
- **Traffic**: Inbound HTTP on port `80` from the internet to the ALB, then from ALB to the ECS tasks on the application port
- **Security groups**:
  - ALB allows `80` from the internet
  - ECS tasks allow only the application port **from the ALB security group**

## API endpoints

- `GET /health`

```json
{ "status": "ok" }
```

- `GET /status`

```json
{
  "service": "devops-assessment-api",
  "uptime": 123.456,
  "timestamp": "2026-03-10T12:00:00.000Z"
}
```

- `POST /process`

Request:

```json
{ "data": "example" }
```

Response:

```json
{ "id": "…", "status": "processed" }
```

## Run locally (Docker Compose)

From the repo root:

```bash
docker compose -f docker/docker-compose.yml up --build
```

Verify:

```bash
curl -s http://localhost:3000/health
```

## Run tests

From `app/`:

```bash
npm install --package-lock-only
npm ci
npm test
```

Note: The repository intentionally does not commit a `package-lock.json` (to keep generation deterministic in CI). CI generates the lockfile then uses `npm ci`.

## CI/CD pipeline (GitHub Actions)

Workflow: [`.github/workflows/ci-cd.yml`](.github/workflows/ci-cd.yml)

On pull requests to `main`:
- Installs dependencies using `npm ci`
- Runs the Jest test suite for the Node.js API

On pushes to `main`:
- Runs the same test job as for pull requests
- Authenticates to AWS using **OIDC** (no long‑lived AWS keys in GitHub)
- Logs in to both **Amazon ECR** and **Docker Hub**
- Builds the Docker image for the `api` container
- Pushes the image to:
  - **Amazon ECR** tagged as `latest` and the commit SHA
  - **Docker Hub** tagged as `latest` and the commit SHA
- Computes the immutable ECR image URI (`<account>.dkr.ecr.<region>.amazonaws.com/<repository>:<sha>`)
- Downloads the current ECS task definition for the service
- Renders a new task definition that updates the `api` container image to the new ECR URI
- Deploys the updated task definition to the ECS service (rolling deployment, waits for stability)

This gives you **CI** (tests on every PR and push) and **CD** (automatic deploy of `main` to ECS Fargate).

### Required GitHub Secrets

- `AWS_ROLE_ARN`: IAM role for GitHub OIDC to assume (must trust your GitHub org/repo)
- `ECR_REPOSITORY`: ECR repository name (Terraform creates `${project_name}-api` by default, e.g. `devops-assessment-api`)
- `ECS_CLUSTER`: ECS cluster name (e.g. `devops-assessment-cluster`)
- `ECS_SERVICE`: ECS service name (e.g. `devops-assessment-service`)
- `ECS_TASK_FAMILY`: ECS task definition family (e.g. `devops-assessment-task`)
- `ECS_CONTAINER_NAME`: Container name inside the task definition that receives new images (e.g. `api`)
- `DOCKERHUB_USERNAME`: Docker Hub account username
- `DOCKERHUB_TOKEN`: Docker Hub access token (used for `docker login`)
- `DOCKERHUB_REPOSITORY`: Docker Hub repository name (e.g. `username/devops-assessment-api`)

## Deploy infrastructure (Terraform)

Prereqs:
- AWS account credentials available locally (for Terraform)

Steps:

```bash
cd terraform
terraform init
terraform apply
```

After apply, Terraform outputs:
- the ECR repository URL
- the ECS cluster and service names
- the ALB DNS name you can use to reach the API over HTTP

### Terraform inputs

Key variables are defined in `terraform/variables.tf`:

- `aws_region`: AWS region (default `eu-west-1`)
- `project_name`: name prefix for resources (default `devops-assessment`)
- `vpc_cidr`: VPC CIDR block (default `10.0.0.0/16`)
- `az_count`: number of availability zones / public subnets (default `2`)
- `container_port`: application port exposed by the API container (default `3000`)
- `desired_count`: desired number of ECS tasks (default `2`)
- `task_cpu` / `task_memory`: Fargate task sizing
- `image_tag`: image tag to deploy from ECR (CI uses the commit SHA; default is `latest` for local experiments)

### Deploy a new app version

1. Push to `main` to publish an ECR image (tags: `latest` and SHA)
2. Update the deployed task to use the desired image tag:
   - Option A: keep `image_tag=latest` (default) and redeploy by forcing a new deployment in ECS
   - Option B: set `-var "image_tag=<sha>"` and re-apply Terraform

## Security considerations

- **No secrets committed**: AWS credentials are obtained via OIDC in GitHub Actions; infrastructure is parameterised via Terraform variables and GitHub Secrets
- **Least privilege (assessment‑appropriate)**: ECS execution role uses the AWS managed execution policy; the task role is intentionally minimal
- **Network controls**: security groups only allow inbound traffic from the ALB to the application port; all other inbound traffic to the tasks is denied
- **Centralised logging**: application and Redis logs are sent to CloudWatch Logs
- **HTTPS / TLS (not implemented in this submission)**: enabling HTTPS on the ALB via ACM requires a verified domain and DNS control (typically Route53 validation). This was not available in the assessment environment due to domain/provider constraints. In production, I would provision an ACM certificate for the application domain, add an HTTPS listener on port `443`, and redirect HTTP (`80`) to HTTPS (`443`).

## Design decisions

- **Redis sidecar on ECS**: keeps the AWS service footprint small and easy to understand. In production, you’d typically replace this with **ElastiCache for Redis** for durability and scaling.
- **Public subnets for simplicity**: for this assessment the ALB and ECS tasks run in public subnets with security‑group‑based isolation. In production you would normally place tasks in private subnets and front them with an ALB in public subnets.

## Assessment summary

This repository demonstrates an end‑to‑end implementation of a small Node.js/Redis workload on AWS using production‑style tools while keeping the design intentionally simple for an assessment. The codebase shows how to:

- Express the infrastructure as code with Terraform (VPC, ALB, ECS Fargate, ECR, security groups, CloudWatch Logs).
- Package and run the API and its Redis dependency as containers, both locally via Docker Compose and in ECS as a task with two containers.
- Implement a CI/CD pipeline with GitHub Actions that automatically tests, builds, publishes, and deploys new versions of the API to ECS when changes are pushed to `main`.

The trade‑offs are documented in the design decisions section so that it is clear what was simplified for the exercise and what would be evolved for a production environment.

