![AWS](https://imgur.com/wLMcRHS.jpg)

# Django on AWS ECS (EC2) + ECR

A minimal "Hello World" Django application, containerized and deployed on AWS using a self-managed, EC2-backed ECS cluster behind an Application Load Balancer, with images stored in a private ECR repository. The app is a single stateless view, so there's no database tier — no RDS, no Postgres, no MySQL.

## Architecture

The VPC (`10.0.0.0/16`) spans two Availability Zones, `us-east-1a` and `us-east-1b`. Each AZ has a public subnet and a private subnet. The public subnets hold the Application Load Balancer and the NAT Gateway; the private subnets hold the EC2 instances that ECS schedules containers onto. Traffic flow: internet → Internet Gateway → ALB → ECS service → container. Outbound traffic from the private instances (registering with ECS, pulling images from ECR, shipping logs to CloudWatch) goes out through the NAT Gateway.

## Tech stack

Django (single view, no models/database), Docker (multi-stage build, non-root user, gunicorn as the WSGI server), Amazon ECR (private image registry), Amazon ECS with the EC2 launch type (an Auto Scaling Group of two instances, `bridge` network mode, dynamic host-port mapping so multiple containers can share one instance), an Application Load Balancer, and CloudWatch Logs for container output.

## Containerizing the app

![Docker](https://imgur.com/raGErLx.png)

The Dockerfile uses a two-stage build: a `builder` stage installs dependencies into a virtual environment, and a slim `production` stage copies just that environment plus the app code, running everything as a non-root `django` user. The container exposes port 8000 and starts via gunicorn (`hello_world_django_app.wsgi:application`), with a `HEALTHCHECK` hitting `/hello/` — the only route the app actually defines.

## Networking and security groups

| Security group | Inbound rule | Source | Why |
|---|---|---|---|
| `alb-sg` | TCP 80 | `0.0.0.0/0` | The only point the public internet is allowed to enter |
| `ecs-sg` | TCP 32768–65535 | `alb-sg` | Dynamic host ports (since each EC2 instance can run multiple containers via ECS's dynamic port mapping), reachable only from the load balancer's security group, not from the open internet |

Two IAM roles back the compute layer: `ecsInstanceRole` lets each EC2 instance register itself with the ECS cluster, and `ecsTaskExecutionRole` lets ECS pull the container image from ECR and ship its logs to CloudWatch on the task's behalf.

## Deployment pipeline

1. Build the Docker image locally and push it to a private ECR repository.
2. Stand up the VPC, subnets, Internet Gateway, NAT Gateway, route tables, and security groups.
3. Create the ECS cluster, launch template (ECS-optimized AMI), and Auto Scaling Group, so two EC2 instances register as ECS container instances.
4. Register a task definition referencing the ECR image, with dynamic port mapping and `awslogs` logging configured.
5. Create the Application Load Balancer, target group, and listener.
6. Create the ECS service, tying the task definition to the load balancer so ECS automatically registers each running task's dynamic port with the target group.

## Verified deployment

![Hello World output](<img width="959" height="539" alt="Screenshot 2026-06-17 064622" src="https://github.com/user-attachments/assets/cc245ccc-9fd8-45ac-be8e-df981fe5e99c" />
)

Hitting the ALB's DNS name at `/hello/` returns the expected response, confirming the full path works end to end: Internet Gateway → ALB → ECS service → container → Django.

## Acknowledgements

The base Django app and original project template are adapted from [Harshhaa's DevOps-Project-04](https://github.com/NotHarshhaa). This README documents the architecture as actually implemented here, which differs from the original template's production-considerations section in one key way: no RDS or database layer was added, since the app has no functionality that requires one.
