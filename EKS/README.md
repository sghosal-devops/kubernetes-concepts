# EKS Exploration Lab

This folder is for learning AWS Kubernetes using Amazon EKS.

## What you will practice
- Create an EKS cluster with managed node groups
- Deploy workloads in a dedicated namespace
- Expose apps with AWS Load Balancer Controller (ALB Ingress)
- Observe logs, scaling, and networking behavior

## Folder layout
- `cluster/`: cluster creation config
- `manifests/`: workload and ingress manifests
- `commands.md`: step-by-step command flow

## Prerequisites
- AWS account and IAM user/role with EKS permissions
- AWS CLI configured (`aws configure`)
- `kubectl`, `eksctl`, and `helm` installed

## Recommended learning flow
1. Review `cluster/eksctl-cluster.yaml`
2. Run commands from `commands.md`
3. Apply manifests in order from `manifests/`
4. Validate service and ingress endpoints

## Notes
- For ALB ingress to work, install AWS Load Balancer Controller first.
- If you use private subnets, ensure route/NAT is configured correctly.
- Replace placeholder values such as `<AWS_REGION>` and `<ACCOUNT_ID>` in docs/commands.
