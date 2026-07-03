# Technical Details

## Table of Contents

- [1. AWS](#1-aws)
  - [1.1. IAM Role for GitHub Actions to authenticate to AWS](#11-iam-role-for-github-actions-to-authenticate-to-aws)
  - [1.2. Private ECR Repositories for Microservices](#12-private-ecr-repositories-for-microservices)
  - [1.3. ACM Certificate for Kubernetes Ingress](#13-acm-certificate-for-kubernetes-ingress)
- [2. GitHub Actions](#2-github-actions)
  - [2.1. .github/workflows/swan_docker.yml](#21-githubworkflowsswan_dockeryml)
  - [2.2. CI/CD pipelines for microservices](#22-cicd-pipelines-for-microservices)
- [3. GitHub](#3-github)
  - [3.1. GitHub App for Argo CD Image Updater](#31-github-app-for-argo-cd-image-updater)
- [4. Karpenter](#4-karpenter)
  - [4.1. swan_kubernetes/swan_karpenter/ec2nodeclass.yaml](#41-swan_kubernetesswan_karpenterec2nodeclassyaml)
  - [4.2. swan_kubernetes/swan_karpenter/nodepool.yaml](#42-swan_kubernetesswan_karpenternodepoolyaml)
- [5. Helm](#5-helm)
  - [5.1. swan_kubernetes/swan_helm/base/](#51-swan_kubernetesswan_helmbase)
  - [5.2. swan_kubernetes/swan_helm/swan_microservices/](#52-swan_kubernetesswan_helmswan_microservices)
- [6. Argo CD](#6-argo-cd)
  - [6.1. swan_kubernetes/swan_argocd/root-app.yaml](#61-swan_kubernetesswan_argocdroot-appyaml)
  - [6.2. swan_kubernetes/swan_argocd/swan_argocd_apps/](#62-swan_kubernetesswan_argocdswan_argocd_apps)

## 1. AWS

### 1.1. IAM Role for GitHub Actions to authenticate to AWS

IAM role is configured to trust GitHub OIDC provider for swan_retail-store-sample-app repository in swanpyaetun organization. An inline policy with ECR permissions for selected ECR repositories is attached to IAM role.

GitHub Actions authentication to AWS is secured by implementing the following practices:
1. Not storing long-lived IAM user credentials in GitHub
2. Using short-lived OIDC tokens with automatic expiration

### 1.2. Private ECR Repositories for Microservices

There are 5 private ECR repositories for microservices.

### 1.3. ACM Certificate for Kubernetes Ingress

ACM certificate is used to enable https for the application.

## 2. GitHub Actions

### 2.1. .github/workflows/swan_docker.yml

.github/workflows/swan_docker.yml is a reusable workflow for building and pushing Docker images to ECR.

swan_docker job does the following steps:
1. checkout repository
2. configure AWS credentials using OIDC
3. login to ECR
4. set up Docker in the runner
5. build and push Docker image

### 2.2. CI/CD pipelines for microservices

There are 5 CI/CD pipelines which build and push Docker images to private ECR repositories.

CI/CD pipelines for microservices can be triggered in 2 ways:
1. The CI/CD pipelines run when a direct push is made to the main branch.
2. The CI/CD pipelines run when a user manually triggers them.

In CI/CD pipelines for microservices, swan_docker job uses [./.github/workflows/swan_docker.yml reusable workflow](#21-githubworkflowsswan_dockeryml).

## 3. GitHub

### 3.1. GitHub App for Argo CD Image Updater

Argo CD Image Updater uses GitHub App to push to GitHub.

Argo CD Image Updater authentication to GitHub is secured by implementing the following practices:
1. Using GitHub App for fine grained control over permissions and repositories
2. GitHub App uses short lived tokens

## 4. Karpenter

### 4.1. swan_kubernetes/swan_karpenter/ec2nodeclass.yaml

The following command can be used to determine the alias version in a specific region.
```bash
export K8S_VERSION="1.35"
aws ssm get-parameter --name "/aws/service/eks/optimized-ami/${K8S_VERSION}/amazon-linux-2023/x86_64/standard/recommended/image_id" --query Parameter.Value | xargs aws ec2 describe-images --query 'Images[0].Name' --image-ids | sed -r 's/^.*(v[[:digit:]]+).*$/\1/'
```
<br>

Private subnets, EKS node IAM role, and default cluster security group are already created in [https://github.com/swanpyaetun/swan_eks-infrastructure-for-retail-store-sample-app](https://github.com/swanpyaetun/swan_eks-infrastructure-for-retail-store-sample-app).
<br><br>

In "default" ec2nodeclass,
1. EKS node IAM role is attached to Karpenter managed nodes.
2. Latest EKS-optimized Amazon Linux 2023 x86_64 AMI is used to create Karpenter managed nodes. 
3. Karpenter managed nodes are deployed in private subnets with tag "karpenter.sh/discovery" = "swan_production_eks_cluster". 
4. Default cluster security group is attached to Karpenter managed nodes.

### 4.2. swan_kubernetes/swan_karpenter/nodepool.yaml

In "default" nodepool, Karpenter only creates the nodes with
1. amd64 CPU architecture
2. linux OS
3. spot and on-demand capacity type
4. "c" -> compute optimized, "m" -> general purpose, "r" -> memory optimized instance category
5. instance generation greater than 2

In this nodepool, Karpenter will prioritize spot capacity type. If spot capacity is unavailable, Karpenter will fallback to on-demand. This nodepool references "default" ec2nodeclass. Karpenter managed nodes will expire after 720 hours (30 days). Karpenter can allocate up to 1000 CPU and 1000Gi memory. Karpenter consolidates nodes that are idle or underutilized. Karpenter waits 1 minute before consolidating the nodes.

Karpenter does cost-optimization by implementing the following practices:
1. Prioritizing spot capacity over on-demand
2. Consolidating nodes that are idle or underutilized
3. Replacing with cheaper node

## 5. Helm

Helm is used to package Kubernetes manifests into Helm charts.

### 5.1. swan_kubernetes/swan_helm/base/

In "base" Helm chart, "default-deny" network policy, and "allow-dns-access" network policy are created. "default-deny" network policy denies all ingress and egress traffic in the namespace. "allow-dns-access" network policy allows the pods in the namespace to access coredns pods.

Security in namespace "otel-demo" is achieved by implementing the following practices:
1. "default-deny" network policy denies all ingress and egress traffic in the namespace
2. Creating least privilege network policies

### 5.2. swan_kubernetes/swan_helm/swan_microservices/

swan_kubernetes/swan_helm/swan_microservices/ contains 10 Helm charts.

In "ui" Helm chart, there is an ingress called "ui". AWS Load Balancer Controller in EKS will create internet-facing ALB. "ui" ingress uses ip mode to route traffic directly to pod ip addresses. The ingress uses ACM certificate to enable https. The ingress is configured to redirect http to https. External DNS in EKS will create DNS records in "swanpyaetun.com" Route 53 public hosted zone.
<br><br>

![](../swan_images/traffic_flow.png)

Network policies are created according to traffic flows in the diagram.

Application is secured by implementing the following practices:
1. https is enabled by using ACM certificate in "ui" ingress
2. "ui" ingress redirecting http to https
3. Creating least privilege network policies

## 6. Argo CD

Argo CD App-of-Apps pattern is used.
<br><br>

```yaml
metadata:
  finalizers:
  - resources-finalizer.argocd.argoproj.io/foreground
```
This ensures Argo CD will delete child resources first before deleting the application itself.
<br><br>

```yaml
spec:
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
    - Validate=true
    - CreateNamespace=false
    - PrunePropagationPolicy=foreground
    - PruneLast=true
```
This enables Argo CD to automatically synchronize with git. Argo CD validates the manifests before applying. New resources are created first before old resources are pruned.

### 6.1. swan_kubernetes/swan_argocd/root-app.yaml

The "root" application deploys child resources.

### 6.2. swan_kubernetes/swan_argocd/swan_argocd_apps/

The "base" application deploys "base" Helm chart.
<br><br>

The "microservices" applicationset generates multiple Argo CD applications for each microservice Helm chart.
```yaml
spec:
  template:
    metadata:
      annotations:
        argocd.argoproj.io/depends-on: base
```
This ensures "microservices" applications are deployed only after "base" application is fully deployed.
<br><br>

Argo CD Image Updater monitors ECR for new container image tags for "microservices" applications. Argo CD Image Updater automatically updates the container image tags defined in the Helm values files in the git repository for each "microservices" application. Argo CD Image Updater uses "git-creds" secret in "argocd" namespace, to be able to push to the git repository.