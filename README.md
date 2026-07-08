# swanpyaetun/swan_retail-store-sample-app

# Deploying Microservices Application to EKS with GitHub Actions and Argo CD

![](swan_docs/swan_images/architecture_diagram.png)

- Tools used: GitHub Actions, EKS, Karpenter, Argo CD, Kyverno
- Deploy microservices application to EKS with GitHub Actions and Argo CD
- Set up 5 GitHub Actions CI/CD pipelines for microservices, which use reusable workflow to build and push Docker images to private ECR repositories
- Secure GitHub Actions authentication to AWS by using short-lived OIDC tokens with automatic expiration, instead of storing long-lived IAM user credentials in GitHub
- Use Karpenter to autoscale EKS nodes
- Prioritize spot capacity type with fallback option to on-demand in Karpenter to optimize cost
- Remove EKS nodes that are idle or underutilized, and replace with cheaper node to optimize cost
- Deny all ingress and egress traffic in the namespace with "default-deny" network policy
- Secure the application by creating least privilege network policies
- Create internet-facing ALB for Kubernetes ingress with AWS Load Balancer Controller
- Use ACM certificate in ingress to enable https
- Redirect http to https in ingress
- Create DNS records in Route 53 public hosted zone with External DNS
- Enable Argo CD to automatically synchronize with git
- Monitor ECR for new container image tags, and update the container image tags in the git repository with Argo CD Image Updater
- Add PSA labels to namespaces with Kyverno mutating policy to enforce baseline Pod Security Standard

## Table of Contents

- [1. Prerequisites](#1-see-prerequisites)
- [2. Technical Details](#2-see-technical-details)
- [3. Instructions](#3-instructions)
- [4. Additional Information](#4-additional-information)

## 1. See [Prerequisites](swan_docs/swan_docs/swan_prerequisites.md)

## 2. See [Technical Details](swan_docs/swan_docs/swan_technical_details.md)

## 3. Instructions

Run "Provision AWS Infrastructure using Terraform" pipeline in [https://github.com/swanpyaetun/swan_eks-infrastructure-for-retail-store-sample-app](https://github.com/swanpyaetun/swan_eks-infrastructure-for-retail-store-sample-app) to create EKS infrastructure.
<br><br>

Run CI/CD pipelines for microservices to build and push Docker images to private ECR repositories.<br>
CI/CD pipelines for microservices can be triggered in 2 ways:
1. The CI/CD pipelines run when a direct push is made to the main branch.
2. The CI/CD pipelines run when a user manually triggers them.

To view ECR basic scanning results, in AWS Management Console, go to ap-southeast-1 region -> Elastic Container Registry -> Private registry -> Repositories. Choose a repository that has container image that you want to view ECR basic scanning result for. Choose an image that you want to view ECR basic scanning result for. Under "Scanning and vulnerabilities", you will see ECR basic scanning result for that image.
<br><br>

```bash
aws eks update-kubeconfig --region ap-southeast-1 --name swan_production_eks_cluster --role-arn arn:aws:iam::655355946217:role/swan_production_eks_cluster-swan_eks_cluster_admin_iam_role
```
This command updates ~/.kube/config so that "swan_production_eks_cluster" EKS cluster can be accessed using kubectl, assuming "swan_production_eks_cluster-swan_eks_cluster_admin_iam_role" IAM role.
<br><br>

```bash
cd ~/Desktop/
git clone git@github.com:swanpyaetun/swan_retail-store-sample-app.git
```
Go to ~/Desktop/ and clone the [https://github.com/swanpyaetun/swan_retail-store-sample-app](https://github.com/swanpyaetun/swan_retail-store-sample-app) repository.
<br><br>

```bash
kubectl apply -f ~/Desktop/swan_retail-store-sample-app/swan_kubernetes/swan_karpenter/
```
This command creates "default" ec2nodeclass and "default" nodepool.
<br><br>

Go to "Settings" -> Developer settings -> GitHub Apps. Select "swan-argocd-image-updater" GitHub App. Go to "General". You will see "githubAppID".<br>
Go to "Settings" -> Integrations -> Applications -> Installed GitHub Apps. Select "swan-argocd-image-updater" GitHub App by clicking "Configure". Look at the URL. The number behind https://github.com/settings/installations/ is "githubAppInstallationID".<br>
Go to "Settings" -> Developer settings -> GitHub Apps. Select "swan-argocd-image-updater" GitHub App. Go to "General" -> Private keys. Click "Generate a private key". The private key will be downloaded to your work station.<br>
```bash
kubectl -n argocd create secret generic git-creds \
  --from-literal=githubAppID=applicationid \
  --from-literal=githubAppInstallationID=installationid \
  --from-literal=githubAppPrivateKey='-----BEGIN RSA PRIVATE KEY-----PRIVATEKEYDATA-----END RSA PRIVATE KEY-----'
```
This command creates "git-creds" secret in "argocd" namespace.
<br><br>

OPTIONAL (Accessing Argo CD ui):
```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d
```
This command will generate "argocd-initial-admin-secret". Copy the secret.
```bash
kubectl port-forward service/argocd-server 8080:80 -n argocd
```
This command makes Argo CD ui accessible at http://localhost:8080.<br>
To access Argo CD ui, go to http://localhost:8080. Enter "admin" in Username field and secret that you copied in Password field. Click "SIGN IN".
<br><br>

```bash
kubectl apply -f ~/Desktop/swan_retail-store-sample-app/swan_kubernetes/swan_argocd/root-app.yaml
```
This command creates "root" application, which will then create child resources.
<br><br>

Go to www.swanpyaetun.com to access the application.
<br><br>

```bash
kubectl delete -f ~/Desktop/swan_retail-store-sample-app/swan_kubernetes/swan_argocd/root-app.yaml
```
This command deletes Argo CD resources.

After Argo CD resources are deleted, wait for 1 minute. After 1 minute, Karpenter will terminate empty nodes. Check in AWS Management Console or use "kubectl get node" to make sure only system EKS node group nodes are left.
<br><br>

Run "Terraform Destroy" pipeline in [https://github.com/swanpyaetun/swan_eks-infrastructure-for-retail-store-sample-app](https://github.com/swanpyaetun/swan_eks-infrastructure-for-retail-store-sample-app) to destroy EKS infrastructure.

## 4. Additional Information

Terraform code for EKS infrastructure, and GitHub Actions CI/CD pipelines for Terraform: [https://github.com/swanpyaetun/swan_eks-infrastructure-for-retail-store-sample-app](https://github.com/swanpyaetun/swan_eks-infrastructure-for-retail-store-sample-app)

This repository is a fork of [https://github.com/aws-containers/retail-store-sample-app](https://github.com/aws-containers/retail-store-sample-app).
