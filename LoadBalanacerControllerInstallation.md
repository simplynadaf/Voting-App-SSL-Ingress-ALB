---
title: AWS Load Balancer Controller Install on AWS EKS
description: Learn to install AWS Load Balancer Controller for Ingress Implementation on AWS EKS
---
 

## Step-00: Introduction
1. Create IAM Policy and make a note of Policy ARN
2. Create IAM Role and k8s Service Account and bound them together
3. Install AWS Load Balancer Controller using HELM3 CLI
4. Understand IngressClass Concept and create a default Ingress Class 

## Step-01: Pre-requisites
### Pre-requisite-1: Create EKS Cluster and Worker Nodes (if not created)
```t
# Create Cluster (Section-01-02)
eksctl create cluster --name=eksdemo1 \
                      --region=us-east-1 \
                      --zones=us-east-1a,us-east-1b \
                      --version="1.21" \
                      --without-nodegroup 


# Get List of clusters (Section-01-02)
eksctl get cluster   

# Template (Section-01-02)
eksctl utils associate-iam-oidc-provider \
    --region region-code \
    --cluster <cluter-name> \
    --approve

# Replace with region & cluster name (Section-01-02)
eksctl utils associate-iam-oidc-provider \
    --region us-east-1 \
    --cluster eksdemo1 \
    --approve

# Create EKS NodeGroup in VPC Private Subnets (Section-07-01)
eksctl create nodegroup --cluster=eksdemo1 \
                        --region=us-east-1 \
                        --name=eksdemo1-ng-private1 \
                        --node-type=t3.medium \
                        --nodes-min=2 \
                        --nodes-max=4 \
                        --node-volume-size=20 \
                        --ssh-access \
                        --ssh-public-key=kube-demo \
                        --managed \
                        --asg-access \
                        --external-dns-access \
                        --full-ecr-access \
                        --appmesh-access \
                        --alb-ingress-access \
                        --node-private-networking       
```

## Step-02: Create IAM Policy & IAM Role
- Create IAM policy for the AWS Load Balancer Controller that allows it to make calls to AWS APIs on your behalf.
- As on today `2.3.1` is the latest Load Balancer Controller
- We will download always latest from main branch of Git Repo
- [AWS Load Balancer Controller Main Git repo](https://github.com/kubernetes-sigs/aws-load-balancer-controller)
```t

# Create IAM Policy using below link copy policy in Json format and crate policy from the Consol 
# Name of Policy AWSLoadBalancerControllerIAMPolicy

 https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json

### Make a note of Policy ARN    
# Make a note of Policy ARN as we are going to use that in next step when creating IAM Role.

Policy ARN:  arn:aws:iam::175662053988:policy/AWSLoadBalancerControllerIAMPolicy


## Create an IAM role for the AWS LoadBalancer Controller and attach the role to the Kubernetes service account 
### Create IAM Role using eksctl
```
# Verify if any existing service account

Check if service account is available or not
kubectl get sa -n kube-system
kubectl get sa aws-load-balancer-controller -n kube-system

# Install IAM OIDC provider

   oidc_id=$(aws eks describe-cluster --name eks --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)
   aws iam list-open-id-connect-providers | grep $oidc_id
   eksctl utils associate-iam-oidc-provider --cluster eks --approve

# Create IAMServiceAccount Replaced name, cluster and policy arn

eksctl create iamserviceaccount \
  --cluster=eks \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::175662053988:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --approve

### Verify using eksctl cli

# Get IAM Service Account
eksctl  get iamserviceaccount --cluster eks   

## Step-04: Install the AWS Load Balancer Controller using Helm V3 
### Step-04-01: Install Helm
- [Install Helm](https://helm.sh/docs/intro/install/) if not installed
- [Install Helm for AWS EKS](https://docs.aws.amazon.com/eks/latest/userguide/helm.html)

### Step-04-02: Install AWS Load Balancer Controller

# Add the eks-charts repository.
helm repo add eks https://aws.github.io/eks-charts

# Update your local repo to make sure that you have the most recent charts.
helm repo update

# Install the AWS Load Balancer Controller.

## Replace Cluster Name, Region Code, VPC ID, Image Repo Account ID and Region Code  
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=eks \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=vpc-05bb3d43a47b56b68 \
  --set image.repository=602401143452.dkr.ecr.us-east-1.amazonaws.com/amazon/aws-load-balancer-controller


### Step-04-03: Verify that the controller is installed and Webhook Service created
```t
# Verify that the controller is installed.
kubectl -n kube-system get deployment 
kubectl -n kube-system get deployment aws-load-balancer-controller
kubectl -n kube-system describe deployment aws-load-balancer-controller

# Verify AWS Load Balancer Controller Webhook service created
kubectl -n kube-system get svc 
kubectl -n kube-system get svc aws-load-balancer-webhook-service
kubectl -n kube-system describe svc aws-load-balancer-webhook-service


# Verify Labels in Service and Selector Labels in Deployment
kubectl -n kube-system get svc aws-load-balancer-webhook-service -o yaml
kubectl -n kube-system get deployment aws-load-balancer-controller -o yaml


### Step-04-04: Verify AWS Load Balancer Controller Logs
```t
# List Pods
kubectl get pods -n kube-system

# Review logs for AWS LB Controller POD-1
kubectl -n kube-system logs -f <POD-NAME> 
kubectl -n kube-system logs -f  aws-load-balancer-controller-86b598cbd6-5pjfk

# Review logs for AWS LB Controller POD-2
kubectl -n kube-system logs -f <POD-NAME> 
kubectl -n kube-system logs -f aws-load-balancer-controller-86b598cbd6-vqqsk
```

```t
# Uninstall AWS Load Balancer Controller
helm uninstall aws-load-balancer-controller -n kube-system 
```