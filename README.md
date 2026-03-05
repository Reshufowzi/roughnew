```bash
aws eks --region us-east-1 update-kubeconfig --name mycluster
```

### Install kubectl for Linux

```bash
curl -Lo kubectl https://dl.k8s.io/release/$(curl -s -L https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
```

### 🧱 eksctl Setup

```bash
# Install eksctl
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH

curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"
curl -sL "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_checksums.txt" | grep $PLATFORM | sha256sum --check

tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz
sudo install -m 0755 /tmp/eksctl /usr/local/bin && rm /tmp/eksctl
```

### 🔐 OIDC and IAM Setup

```bash
eksctl utils associate-iam-oidc-provider \
  --region=us-east-1 \
  --cluster=mycluster \
  --approve
```

### 📥 Download the IAM policy:

[Click Me for IAM Json Policy](https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/refs/heads/main/docs/install/iam_policy.json)

Create the IAM policy in AWS as: `AWSLoadBalancerControllerIAMPolicy`

### 📥 Create service account:

```bash
eksctl create iamserviceaccount \
  --cluster mycluster \
  --region us-east-1 \
  --namespace kube-system \
  --name aws-load-balancer-controller2 \
  --attach-policy-arn arn:aws:iam::account-id:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```
### 📦 Helm Setup

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```
### 📡 Add and update repo:

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update
```
```bash
aws eks describe-cluster \
  --name mycluster \
  --region us-east-1 \
  --query "cluster.resourcesVpcConfig.vpcId" \
  --output text
```

### 🚀 Install AWS ALB Ingress Controller

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=mycluster \
  --set region=us-east-1 \
  --set vpcId=vpc-id \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller2
```

## 👥 Multi-Team Namespaces with Shared ALB

### Create Namespaces

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: frontend-application
---
apiVersion: v1
kind: Namespace
metadata:
  name: backend-application
```

### frontend-app

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-app
  namespace: frontend-application
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend-app
  template:
    metadata:
      labels:
        app: frontend-app
    spec:
      containers:
      - name: frontend-app
        image: httpd
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-app-service
  namespace: frontend-application
spec:
  selector:
    app: frontend-app
  ports:
  - port: 80
    targetPort: 8080
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: frontend-app-ingress
  namespace: frontend-application
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/group.name: shared-alb
spec:
  rules:
  - http:
      paths:
      - path: /frontend
        pathType: Prefix
        backend:
          service:
            name: frontend-app-service
            port:
              number: 80
```

### Backend-app
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-app
  namespace: backend-application 
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend-app
  template:
    metadata:
      labels:
        app: backend-app
    spec:
      containers:
      - name: backend-app
        image: nginx
        ports:
        - containerPort: 8081
---
apiVersion: v1
kind: Service
metadata:
  name: backend-app-service
  namespace: backend-application
spec:
  selector:
    app: backend-app
  ports:
  - port: 80
    targetPort: 8081
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: backend-app-ingress
  namespace: backend-application
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/group.name: shared-alb
spec:
  rules:
  - http:
      paths:
      - path: /backend
        pathType: Prefix
        backend:
          service:
            name: backend-app-service
            port:
              number: 80
```

