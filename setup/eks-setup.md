# EKS Setup

## Create Cluster

```bash
ACCOUNT=$(aws sts get-caller-identity --query Account --output text)
NAME=ssor-eks
REGION=us-east-1

eksctl create cluster \
--name $NAME \
--region $REGION \
--fargate
```

## Create OIDC IAM Provider for Cluster

This is a prerequisite of the next step.

```bash
eksctl utils associate-iam-oidc-provider --cluster $NAME --approve
```

## Deploy AWS Load Balancer Controller

Download an IAM policy for the AWS Load Balancer Controller that allows it to make calls to AWS APIs on your behalf.

```bash
curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.2.0/docs/install/iam_policy.json
```

Create an IAM policy using the policy downloaded in the previous step.

```bash
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

Create an IAM role and annotate the Kubernetes service account that's named aws-load-balancer-controller in the kube-system namespace for the AWS Load Balancer Controller.

```bash
eksctl create iamserviceaccount \
  --cluster=$NAME \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::$ACCOUNT:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --approve
```

Install cert-manager to inject certificate configuration into the webhooks.

```bash
kubectl apply \
    --validate=false \
    -f https://github.com/jetstack/cert-manager/releases/download/v1.1.1/cert-manager.yaml
```

Install the controller.

```bash
kubectl apply -f v2_2_0_full.yaml
```

## Configure kubeconfig

```bash
aws eks --region $REGION update-kubeconfig --name $NAME
```
