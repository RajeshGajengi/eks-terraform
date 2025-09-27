# AWS EKS Cluster with Managed Node Group using Terraform

## 📖 About the Project

This project demonstrates Infrastructure as Code (IaC) by provisioning an Amazon EKS (Elastic Kubernetes Service) cluster along with a managed node group using Terraform.
The goal is to showcase how to automate Kubernetes cluster creation on AWS, with a focus on scalability, repeatability, and DevOps best practices.

## 🚀 Features
- Automated EKS Cluster Provisioning – fully managed Kubernetes control plane on AWS.
- IAM Roles and Policies – created for both the EKS control plane and worker nodes.
- Networking Setup – cluster deployed inside the default VPC subnets.
- EKS Managed Node Group – includes one worker node for running containerized workloads.
- (Optional) CI/CD Integration – can be extended with Jenkins or other tools for automated provisioning pipelines.

## 📌 Prerequisites
To run this project successfully, the following tools and access are required:
- An **AWS Account** with administrative permissions (IAM, EC2, EKS, VPC).
- **AWS CLI** installed and configured (`aws configure`) with appropriate credentials.
- **Terraform** (v1.0 or later) for Infrastructure as Code deployment.
- **kubectl** for managing and validating workloads on the EKS cluster.
- (Optional) **Jenkins** or another CI/CD tool if you want to automate the provisioning pipeline.


## main.tf file
```hcl
#############################################
# Provider Configuration
#############################################
provider "aws" {
  region = "ap-south-1" # Using Mumbai region
}

#############################################
# EKS Cluster IAM Role
#############################################
resource "aws_iam_role" "cluster_role" {
  name = "eks-cluster-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = [
          "sts:AssumeRole",
          "sts:TagSession"
        ]
        Effect = "Allow"
        Principal = {
          Service = "eks.amazonaws.com" # EKS will assume this role
        }
      },
    ]
  })
}

# Attach the required policy to the cluster role
resource "aws_iam_role_policy_attachment" "cluster_AmazonEKSClusterPolicy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
  role       = aws_iam_role.cluster_role.name
}

#############################################
# Networking (using default VPC & subnets)
#############################################

# Fetch default VPC
data "aws_vpc" "default_vpc" {
  default = true
}

# Fetch all subnets inside the default VPC
data "aws_subnets" "default_subnets" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.default_vpc.id]
  }
}

#############################################
# EKS Cluster Definition
#############################################
resource "aws_eks_cluster" "mycluster" {
  name     = "mycluster"
  role_arn = aws_iam_role.cluster_role.arn
  # version = "1.31"   # Optional: specify EKS version

  vpc_config {
    subnet_ids = data.aws_subnets.default_subnets.ids
  }

  depends_on = [
    aws_iam_role_policy_attachment.cluster_AmazonEKSClusterPolicy,
  ]
}

#############################################
# EKS Node Group IAM Role
#############################################
resource "aws_iam_role" "nodegroup_role" {
  name = "eks-node-group"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "ec2.amazonaws.com" # EC2 nodes (workers) assume this role
      }
    }]
  })
}

# Attach required policies for node group role
resource "aws_iam_role_policy_attachment" "nodegroup_AmazonEKSWorkerNodePolicy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy"
  role       = aws_iam_role.nodegroup_role.name
}

resource "aws_iam_role_policy_attachment" "nodegroup_AmazonEKS_CNI_Policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy"
  role       = aws_iam_role.nodegroup_role.name
}

resource "aws_iam_role_policy_attachment" "nodegroup_AmazonEC2ContainerRegistryReadOnly" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"
  role       = aws_iam_role.nodegroup_role.name
}

#############################################
# EKS Node Group
#############################################
resource "aws_eks_node_group" "node_group" {
  cluster_name    = aws_eks_cluster.mycluster.name
  node_group_name = "mynode"
  node_role_arn   = aws_iam_role.nodegroup_role.arn
  subnet_ids      = data.aws_subnets.default_subnets.ids

  scaling_config {
    desired_size = 1
    max_size     = 1
    min_size     = 1
  }

  # Ensure IAM Role permissions exist before node group creation
  depends_on = [
    aws_iam_role_policy_attachment.nodegroup_AmazonEKSWorkerNodePolicy,
    aws_iam_role_policy_attachment.nodegroup_AmazonEKS_CNI_Policy,
    aws_iam_role_policy_attachment.nodegroup_AmazonEC2ContainerRegistryReadOnly,
  ]
}

```

## Steps to Deploy
### 1. Clone the Repository
```bash
git clone https://github.com/RajeshGajengi/eks-terraform.git
cd eks-terraform
```
### 2. Initialize Terraform
```bash
terraform init
```
This downloads all required provider plugins and sets up the working directory.
### 3. Validate Configuration
```bash
terraform validate
```
`terraform validate` ensures the configuration is syntactically correct.

### 4. Plan the Deployment
```bash
terraform plan
```
`terraform plan` shows what resources will be created.
### 5. Apply the Configuration
```bash
terraform apply -auto-approve
```
This provisions the EKS cluster and node group on AWS.


### 6. Configure kubeconfig
After the cluster is created, update your local `kubeconfig`:
```bash
aws eks update-kubeconfig --region <your-region> --name <cluster-name>
```
### 7. Verify the Cluster 
Check that nodes are running:
```bash
kubectl get nodes
```
You should see your worker node(s) in Ready state.

## 🧹 Cleanup (Destroy the Cluster)
To avoid incurring costs, destroy the resources when no longer needed:
```bash
terraform destroy -auto-approve
```

## ⚠️ Troubleshooting – UnsupportedAvailabilityZoneException
**Issue**:
When creating a cluster in us-east-1, you may encounter the error:
```bash
Error: creating EKS Cluster (mycluster): operation error EKS: CreateCluster, https response error StatusCode: 400, RequestID: 8510702a-bd46-403c-9f61-e63993a4cd9a, UnsupportedAvailabilityZoneException: Cannot create cluster 'mycluster' because EKS does not support creating control plane instances in us-east-1e, the targeted availability zone. Retry cluster creation using control plane subnets that span at least two of these availability zones: us-east-1a, us-east-1b, us-east-1c, us-east-1d, us-east-1f. Note, post cluster creation, you can run worker nodes in separate subnets/availability zones from control plane subnets/availability zones passed during cluster creation
```
**Cause**:
EKS control plane must run in at least two supported Availability Zones in your region.
In `us-east-1`, the AZ`us-east-1e` is not supported for control planes, so AWS rejects the cluster if that subnet is included.

### ✅ Solution 1: Hardcode supported AZs (Quick Fix)
Update the aws_subnets data block to only include allowed AZs:
```hcl
data "aws_subnets" "default_subnets" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.default_vpc.id]
  }

  filter {
    name   = "availability-zone"
    values = ["us-east-1a", "us-east-1b", "us-east-1c"]
  }
}
```

### ✅ Solution 2: Dynamically Exclude Bad AZ (More Robust)
If you want a reusable config, exclude `us-east-1e` dynamically:
```hcl
# Get all subnets in the default VPC
data "aws_subnets" "default_subnets" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.default_vpc.id]
  }
}

# Filter subnets to only supported AZs
locals {
  supported_azs = ["us-east-1a", "us-east-1b", "us-east-1c", "us-east-1d", "us-east-1f"]

  filtered_subnets = [
    for subnet in data.aws_subnets.default_subnets.ids :
    subnet
    if contains(local.supported_azs, data.aws_subnet.selected[subnet].availability_zone)
  ]
}

# Map subnet IDs to their availability zones
data "aws_subnet" "selected" {
  for_each = toset(data.aws_subnets.default_subnets.ids)
  id       = each.key
}

# Update cluster block to use filtered subnets
resource "aws_eks_cluster" "mycluster" {
  name     = "mycluster"
  role_arn = aws_iam_role.cluster_role.arn

  vpc_config {
    subnet_ids = local.filtered_subnets
  }

  depends_on = [
    aws_iam_role_policy_attachment.cluster_AmazonEKSClusterPolicy,
  ]
}
```
✅ With these fixes, your cluster will only use supported AZs, and you’ll avoid the UnsupportedAvailabilityZoneException error.

## Jenkins Pipeline 
### Steps to Set Up Jenkins
Make sure you have installed Java, Jenkins and Terraform in your server.

#### 1. Install Plugins 
- AWS Credentials
- Pipeline: AWS Steps

#### 2. Add Credetials
- Dashboard → Manage Jenkins → Credentials → Global credentials (unrestricted)
- Add new credential:
  - Kind: AWS Credentials
  - ID: `aws-cred` 
  - Access Key ID: `Your-AWS-Access-Key`
  - Secret Key: `Your-Secret-Key`


`Jenkinsfile.jdp`
```
pipeline {
    agent any
    stages{
        stage('git clone'){
            steps{
                git branch: 'main', url: 'https://github.com/RajeshGajengi/eks-terraform.git'
            }
        }
        stage('terraform init'){
            steps{
                withAWS(credentials: 'aws-cred', region: 'us-east-1') {
                     sh 'terraform init'
                }
               
            }
        }
        stage('terraform plan'){
            steps{
                withAWS(credentials: 'aws-cred', region: 'us-east-1') {
                sh 'terraform plan'
                }
            }
        }
        stage('terraform apply'){
            steps{
                withAWS(credentials: 'aws-cred', region: 'us-east-1') {
                sh 'terraform apply --auto-approve'
                }
            }
        }
     }
}
```

#### Stage for Destroying the EKS Cluster
Remove all stages from the above pipeline and replace them with the stage below.
This will destroy your recently created cluster:
```
stage('terraform destroy'){
            steps{
                withAWS(credentials: 'aws-cred', region: 'us-east-1') {
                sh 'terraform destroy --auto-approve'
            }
      }
}
```
