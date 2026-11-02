# Projeto-AWS---Fundamentos-e-Servi-os

Descrição 

Este projeto reúne conceitos e práticas desenvolvidos durante o curso de AWS, incluindo serviços de computação, armazenamento, banco de dados, redes e segurança.

Serviços AWS Explorados

Amazon EC2
Amazon S3
Amazon VPC
Amazon RDS
Amazon DynamoDB
AWS Lambda
Amazon SNS
Amazon SQS
AWS Fargate
Elastic Load Balancing (ELB)
Auto Scaling

Arquitetura

Diagramas da solução
Imagens da infraestrutura
Fluxos de comunicação entre serviços

Estrutura do Repositório
Plain Text
/aws-fundamentos
│
├── README.md
├── docs/
│ ├── arquitetura.png
│ └── desafios.md
│
├── scripts/
│ ├── cloudformation/
│ └── terraform/
│
├── banco/
│ └── modelo.sql
│
└── imagens/

Tecnologias Utilizadas

AWS EC2
AWS S3
AWS RDS
AWS VPC
Git
GitHub

Conceitos Aplicados

Computação em nuvem
Escalabilidade horizontal
Alta disponibilidade
Redes VPC
Armazenamento em blocos, objetos e arquivos
Banco de dados relacionais e NoSQL
Mensageria com SNS e SQS
Segurança com IAM e Security Groups

Referências

Documentação AWS
Curso DIO
Laboratórios realizados durante o treinamento

👨‍💻 Autor
Lucas Eugênio Miranda Pinto




## Terraform
provider "aws" {
  region = "us-east-1"
}

# VPC
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = "dio-vpc"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
}

# Subnet Pública AZ A
resource "aws_subnet" "public_a" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "us-east-1a"
  map_public_ip_on_launch = true
}

# Subnet Pública AZ B
resource "aws_subnet" "public_b" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.2.0/24"
  availability_zone       = "us-east-1b"
  map_public_ip_on_launch = true
}

# Route Table
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
}

resource "aws_route" "internet" {
  route_table_id         = aws_route_table.public.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.igw.id
}

resource "aws_route_table_association" "a" {
  subnet_id      = aws_subnet.public_a.id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "b" {
  subnet_id      = aws_subnet.public_b.id
  route_table_id = aws_route_table.public.id
}

# Security Group
resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "SSH"
    from_port   = 22
    to_port     = 22
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

# Launch Template
resource "aws_launch_template" "web" {
  name_prefix   = "dio-template"
  image_id      = "ami-0c02fb55956c7d316"
  instance_type = "t2.micro"

  vpc_security_group_ids = [
    aws_security_group.web.id
  ]

  user_data = base64encode(<<EOF
#!/bin/bash
yum update -y
yum install httpd -y
systemctl start httpd
systemctl enable httpd
echo "<h1>Projeto AWS DIO</h1>" > /var/www/html/index.html
EOF
  )
}

# Target Group
resource "aws_lb_target_group" "tg" {
  name     = "dio-tg"
  port     = 80
  protocol = "HTTP"
  vpc_id   = aws_vpc.main.id
}

# Application Load Balancer
resource "aws_lb" "alb" {
  name               = "dio-alb"
  load_balancer_type = "application"

  security_groups = [
    aws_security_group.web.id
  ]

  subnets = [
    aws_subnet.public_a.id,
    aws_subnet.public_b.id
  ]
}

resource "aws_lb_listener" "listener" {
  load_balancer_arn = aws_lb.alb.arn

  port     = 80
  protocol = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.tg.arn
  }
}

# Auto Scaling Group
resource "aws_autoscaling_group" "asg" {
  desired_capacity = 2
  max_size         = 4
  min_size         = 2

  vpc_zone_identifier = [
    aws_subnet.public_a.id,
    aws_subnet.public_b.id
  ]

  target_group_arns = [
    aws_lb_target_group.tg.arn
  ]

  launch_template {
    id      = aws_launch_template.web.id
    version = "$Latest"
  }
}

## YAML 

AWSTemplateFormatVersion: '2010-09-09'

Resources:

  EC2Instance:
    Type: AWS::EC2::Instance

    Properties:
      InstanceType: t2.micro

      ImageId: ami-0c02fb55956c7d316

## Script deploy

#!/bin/bash

echo "Iniciando Deploy AWS"

aws cloudformation deploy \
--template-file ../cloudformation/infraestrutura.yaml \
--stack-name ProjetoDIOAWS

