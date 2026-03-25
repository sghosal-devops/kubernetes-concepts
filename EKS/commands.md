# EKS Commands

## 1) Create cluster
```bash
eksctl create cluster -f EKS/cluster/eksctl-cluster.yaml
```

## 2) Verify access
```bash
kubectl get nodes
kubectl get ns
```

## 3) Install metrics server (optional but useful)
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

## 4) Install AWS Load Balancer Controller

Set variables:
```bash
export AWS_REGION=<AWS_REGION>
export CLUSTER_NAME=k8s-learning-eks
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
```

Create IAM policy:
```bash
curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.8.2/docs/install/iam_policy.json
aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json
```

Create service account with IAM role:
```bash
eksctl create iamserviceaccount \
  --cluster $CLUSTER_NAME \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --attach-policy-arn arn:aws:iam::$ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --approve
```

Install chart:
```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update
helm upgrade --install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=$CLUSTER_NAME \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=$AWS_REGION \
  --set vpcId=<VPC_ID>
```

## 5) Deploy sample app
```bash
kubectl apply -f EKS/manifests/00-namespace.yml
kubectl apply -f EKS/manifests/01-deployment.yml
kubectl apply -f EKS/manifests/02-service.yml
kubectl apply -f EKS/manifests/03-ingress-alb.yml
```

## 6) Verify
```bash
kubectl get pods -n eks-lab
kubectl get svc -n eks-lab
kubectl get ingress -n eks-lab
kubectl describe ingress eks-app-ingress -n eks-lab
```
