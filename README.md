# Deploy Django Application Deployment on AWS ECS Fargate Using GitHub Actions for CI/CD

[section:overview]

### Overview:

This project shows the complete end-to-end process of deploying a Django application on AWS ECS Fargate using a fully automated infrastructure pipeline powered by Terraform, Docker, and GitHub Actions. The project uses a secure, cost-efficient architecture with private subnets, Application Load Balancer, and VPC Interface Endpoints for ECR.

[section:architecture overview]

### Architecture Overview:
**Digram:**

<img width="1355" height="830" alt="download" src="https://github.com/user-attachments/assets/30a8f1af-edfb-4cfe-8fbe-435781ef20dc" />

**Components:**
- ECS
- ECR
- Endpoints
- VPCs
- ALB
  
[section:tech stack]
### Tech Stack:

- Django: Web framework for the backend application
- Docker: Containerization
- Terraform: Infrastructure as Code (IaC)
- AWS ECS (Fargate): Container orchestration and deployment
- Amazon RDS: PostgreSQL database
- GitHub Actions: CI/CD pipeline automation

[section:project structure]
### Project Structure
- `main.tf`
- `variables`
- `vpc.tf`
- `security.tf`
- `ecs.tf`
- `ecr.tf`
- `endpoints.tf`
- `alb.tf`
- `iam.tf`
- `output.tf`
- `dockerfile`
- `requirments.txt`

[section:implementing steps]
### Implementing Steps

##### VPC and Subnets

A VPC with 2 public and 2 private subnets spreads resources across availability zones.
- Public subnets for ALB
- Private subnets for ECS Fargate
We use `cidrsubnet()` for programmatic subnet allocation.

```
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

}


resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  count                   = 2
  cidr_block              = cidrsubnet(aws_vpc.main.cidr_block, 8, count.index + 100)
  map_public_ip_on_launch = true
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  tags                    = { Name = "public-subnet-${count.index}" }

}

resource "aws_subnet" "private" {
  vpc_id            = aws_vpc.main.id
  count             = 2
  cidr_block        = cidrsubnet(aws_vpc.main.cidr_block, 8, count.index)
  availability_zone = data.aws_availability_zones.available.names[count.index]
  tags              = { Name = "private-subnet-${count.index}" }

}

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id

}

resource "aws_route_table" "public_rtb" {
  vpc_id = aws_vpc.main.id

}

resource "aws_route_table" "private_rtb" {
  vpc_id = aws_vpc.main.id

}

resource "aws_route" "internet_access" {
  route_table_id         = aws_route_table.public_rtb.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.igw.id

}

resource "aws_route_table_association" "public" {
  count          = 2
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public_rtb.id
}

resource "aws_route_table_association" "private" {
  count          = 2
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private_rtb.id
}


data "aws_availability_zones" "available" {}
```

##### Internet Gateway & Route Tables

Only public subnets receive a route to the internet for the ALB. Private subnets remain isolated.

##### VPC Endpoints
Since tasks run without a NAT, we create VPC Interface Endpoints:
- `ecr.api` for authentication
  ```
  resource "aws_vpc_endpoint" "ecr_api" {
  vpc_id             = aws_vpc.main.id
  service_name       = "com.amazonaws.${var.aws_region}.ecr.api"
  vpc_endpoint_type  = "Interface"
  subnet_ids         = aws_subnet.private[*].id
  security_group_ids = [aws_security_group.endpoints.id]
  private_dns_enabled = true
   }
  ```
  
- `ecr.dkr` for image layer pulls
```
resource "aws_vpc_endpoint" "ecr_dkr" {
  vpc_id             = aws_vpc.main.id
  service_name       = "com.amazonaws.${var.aws_region}.ecr.dkr"
  vpc_endpoint_type  = "Interface"
  subnet_ids         = aws_subnet.private[*].id
  security_group_ids = [aws_security_group.endpoints.id]
  private_dns_enabled = true

}
```

- `logs` for CloudWatch logs streaming
```
resource "aws_vpc_endpoint" "ecr_logs" {
  vpc_id             = aws_vpc.main.id
  service_name       = "com.amazonaws.${var.aws_region}.logs"
  vpc_endpoint_type  = "Interface"
  subnet_ids         = aws_subnet.private[*].id
  security_group_ids = [aws_security_group.endpoints.id]
  private_dns_enabled = true

}
```
- And Gateway Endpoint:
   - s3 for pip installs, AWS SDK access
```
resource "aws_vpc_endpoint" "s3" {
  vpc_id            = aws_vpc.main.id
  service_name      = "com.amazonaws.${var.aws_region}.s3"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = [aws_route_table.private_rtb.id]

}
```

Each interface endpoint gets its own ENI and security group.

##### Security Groups

We define three security groups:

- ALB SG for Allows HTTP from internet
```
resource "aws_security_group" "alb" {
  name   = "alb-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

- ECS Tasks SG for Allows inbound only from ALB SG
```
resource "aws_security_group" "ecs" {
  name        = "${var.app_name}-sg"
  description = "Allow HTTP traffic"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 8000
    to_port     = 8000
    protocol    = "tcp"
    security_groups = [aws_security_group.alb.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  
}
```


- VPC Endpoints SG for Allowing inbound HTTPS from ECS SG
```
resource "aws_security_group" "endpoints" {
  name   = "vpc-endpoints-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port       = 443
    to_port         = 443
    protocol        = "tcp"
    security_groups = [aws_security_group.ecs.id]
    description     = "Allow ECS tasks to reach VPC endpoints"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

##### IAM Roles
We configure:

- Task Execution Role that Allows ECS to pull images and write logs
```
resource "aws_iam_role" "ecs_task_execution" {
  name = "${var.app_name}-ecs-execution-role"


  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [{
      Effect = "Allow",
      Principal = {
        Service = "ecs-tasks.amazonaws.com"
      },

      Action = "sts:AssumeRole"
      

    }]

  })


}



resource "aws_iam_role_policy_attachment" "ecs_execution_attach" {
  role       = aws_iam_role.ecs_task_execution.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}
```

- Task Role For app-level permissions (optional)
```
resource "aws_iam_role_policy" "ecs_ecr_access" {
  name = "${var.app_name}-ecr-access"
  role = aws_iam_role.ecs_task_execution.id

  policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Effect = "Allow",
        Action = [
          "ecr:GetAuthorizationToken",
          "ecr:BatchCheckLayerAvailability",
          "ecr:GetDownloadUrlForLayer",
          "ecr:BatchGetImage"
        ],
        Resource = "*"
      }
    ]
  })
}
```

##### ECS Cluster and Service
We define:

- ECR Regisrty
```
resource "aws_ecr_repository" "django" {
  name = var.app_name
  force_delete = true

}
```
- ECS Cluster
```
resource "aws_ecs_cluster" "main" {
  name = "${var.app_name}-cluster"

}
```

- Task Definition using the ECR image URL
```
resource "aws_ecs_task_definition" "django" {
  family                   = "${var.app_name}-task"
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  cpu                      = "256"
  memory                   = "512"
  execution_role_arn       = aws_iam_role.ecs_task_execution.arn

  container_definitions = jsonencode([
    {
      name      = var.app_name
      image     = "${aws_ecr_repository.django.repository_url}:latest"
      essential = true
      portMappings = [{
        containerport = 8000
        hostport      = 8000
      }]
    }
  ])

}
```

- ECS Service connecting to ALB target group
```
resource "aws_ecs_service" "django" {
  name            = "${var.app_name}-service"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.django.arn
  desired_count   = 1
  launch_type     = "FARGATE"

  network_configuration {
    subnets          = aws_subnet.private[*].id
    assign_public_ip = false
    security_groups  = [aws_security_group.ecs.id]

  }

  load_balancer {
    target_group_arn = aws_lb_target_group.django.arn
    container_name   = "django-app"
    container_port   = 8000

  }

depends_on = [aws_lb_listener.http]

}
```

The container listens on port 8000.

At this stage  after building your docker image, you are able to build the infrastructre using terrform command 
[section: Docker part]

##### Dockerizing the Django Application
```
FROM python:3.11-slim

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["gunicorn", "myproject.wsgi:application", "--bind", "0.0.0.0:8000"]
```

##### Docker Compose for Local Testing
A docker-compose.yml allows you to:

- Build image locally

- Run Django with database if needed

- Validate dependencies

Local testing ensures containers run before pushing to AWS.

```
version: '3.9'

services:
  postgres_db:
    image: postgres:15  # latest stable is 15; version 17 doesn't exist (yet)
    container_name: postgres-db
    environment:
      POSTGRES_DB: ${DATABASE_NAME}
      POSTGRES_USER: ${DATABASE_USERNAME}
      POSTGRES_PASSWORD: ${DATABASE_PASSWORD}
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    env_file:
      - .env
    networks:
      - mynetwork

  django-web:
    build: .
    command: gunicorn myproject.wsgi:application --bind 0.0.0.0:8000
    container_name: django-web
    ports:
      - "8000:8000"
    depends_on:
      - postgres_db
    environment:
      POSTGRES_HOST: postgres_db
      POSTGRES_DB: ${DATABASE_NAME}
      POSTGRES_USER: ${DATABASE_USERNAME}
      POSTGRES_PASSWORD: ${DATABASE_PASSWORD}
    env_file:
      - .env
    networks:
      - mynetwork

volumes:
  postgres_data:

networks:
  mynetwork:
```

[section:github action pipeline]
### GitHub Actions CI/CD Pipeline:
GitHub Actions is a powerful automation engine built directly into GitHub. It allows you to automate workflows such as testing, building Docker images, provisioning infrastructure, and deploying applications. To begin using GitHub Actions in this project:

1- Create a `.github/workflows/` directory in your repository.

2- Inside it, create a YAML workflow file such as `deploy.yml`.

Add your AWS credentials and Terraform Cloud token under Repository → Settings → Secrets and variables → Actions.

Define jobs that run automatically whenever code is pushed to the `main` branch.

This provides a fully automated CI/CD pipeline that builds, tests, deploys, and updates your infrastructure and application without any manual intervention.

Below is the detailed implementation used in this project. The CI/CD pipeline consists of three jobs:

#### Stage 1 — Build & Test Django App
   Job: `build-test`

Runs on: ubuntu-latest

Steps:

Checkout Code
Pull repository source code into the runner.

Set Up Python Environment
Install Python 3.11.

Install Dependencies
Run `pip install -r requirements.txt`

Run Tests
Execute:
`
python manage.py test
`

this ensures the application works before deployment.

```
jobs:

  build-test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: 3.11

      - name: Install Dependencies
        run: |
          pip install -r requirements.txt

      - name: Run Tests
        run: |
          python manage.py test
```


#### Stage 2 — Deploy Infrastructure with Terraform

Job: `terraform-deploy`

Depends on: `build-test`

Steps:

1- Checkout Code

2- Configure AWS Credentials
   - Using GitHub secrets, grants Terraform access to AWS.

3- Install Terraform

4- Configure Terraform Cloud Credentials
   - `Creates ~/.terraform.d/credentials.tfrc.json`
   -  authenticates Terraform to Terraform Cloud.

5- Terraform Init
   - Downloads providers + connects to remote backend.

6- Terraform Plan
   - Shows what infra will be created.

7- Terraform Apply
   - Deploys:

      VPC

      Subnets

      ALB

      ECS cluster

      ECS service

      Security groups

      VPC endpoints

      (Important) ECR Repository

      IAM roles

❗ Important Problem:
This job runs before building and pushing the Docker image.
So ECS will try to pull :latest but no image exists yet therefor, task will fail.

```
terraform-deploy:
    needs: build-test
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4
      
      - name: Export AWS_REGION to env
        run: echo "AWS_REGION=${{ secrets.AWS_REGION }}"


      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.8.2

      - name: Configure Terraform credentials for TFC
        run: |
              mkdir -p ~/.terraform.d
              echo '{
                "credentials": {
                 "app.terraform.io": {
                  "token": "${{ secrets.TF_API_TOKEN }}"
                 }
                }
              }' > ~/.terraform.d/credentials.tfrc.json

      
      - name: Terraform Init
        run: terraform init

      - name: Terraform Plan
        run: terraform plan

      - name: Terraform Apply
        run: terraform apply -auto-approve
```


#### Stage 3 — Build & Push Docker Image to ECR
  Job: `docker-build-push`

  Depends on: `terraform-deploy`

  Steps:

   1- Checkout Code

   2- Configure AWS Credentials
      - Needed for Docker login to ECR.

   3- Login to ECR
      - Makes Docker able to push to the ECR registry.

   4- Build Docker Image
      - ` docker build -t $ECR_REPOSITORY`

   5- Tag Docker Image
      - `docker tag $ECR_REPOSITORY $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPOSITORY`
   
   6- Push Docker Image
      - Pushes :latest to your ECR repository

   7- Set ECR_REPO_URI
      - Stores repo URI into `$GITHUB_ENV`

   ```
docker-build-push:
    needs: terraform-deploy
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build, Tag, and Push Docker Image
        run: |
          docker build -t $ECR_REPOSITORY .
          docker tag $ECR_REPOSITORY $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPOSITORY

      - name: Push Docker image to Amazon ECR
        run: |
          docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPOSITORY:latest
      
      - name: Set ECR_REPO_URI
        run: echo "ECR_REPO_URI=${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPOSITORY}" >> $GITHUB_ENV
```


  - The whole code below:
   ```
      name: Django CI/CD on AWS ECS

on:
  push:
    branches: [main]

env:
  AWS_REGION: ${{ secrets.AWS_REGION }}
  AWS_ACCOUNT_ID: ${{ secrets.AWS_ACCOUNT_ID }}
  ECR_REPOSITORY: ${{ secrets.ECR_REPOSITORY }}
  IMAGE_TAG: ${{ github.sha }}


jobs:

  build-test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: 3.11

      - name: Install Dependencies
        run: |
          pip install -r requirements.txt

      - name: Run Tests
        run: |
          python manage.py test

  


  terraform-deploy:
    needs: build-test
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4
      
      - name: Export AWS_REGION to env
        run: echo "AWS_REGION=${{ secrets.AWS_REGION }}"


      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.8.2

      - name: Configure Terraform credentials for TFC
        run: |
              mkdir -p ~/.terraform.d
              echo '{
                "credentials": {
                 "app.terraform.io": {
                  "token": "${{ secrets.TF_API_TOKEN }}"
                 }
                }
              }' > ~/.terraform.d/credentials.tfrc.json

      
      - name: Terraform Init
        run: terraform init

      - name: Terraform Plan
        run: terraform plan

      - name: Terraform Apply
        run: terraform apply -auto-approve


  docker-build-push:
    needs: terraform-deploy
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build, Tag, and Push Docker Image
        run: |
          docker build -t $ECR_REPOSITORY .
          docker tag $ECR_REPOSITORY $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPOSITORY

      - name: Push Docker image to Amazon ECR
        run: |
          docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPOSITORY:latest
      
      - name: Set ECR_REPO_URI
        run: echo "ECR_REPO_URI=${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPOSITORY}" >> $GITHUB_ENV
```




