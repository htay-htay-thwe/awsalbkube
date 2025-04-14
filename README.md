# 📦 AWS EKS Setup Guide: Load Balancer & EBS Storage

This guide helps you configure **AWS Load Balancer Controller** and **EBS CSI Driver** for your EKS cluster using `eksctl` and `Helm`.

---

## ✅ ELB (Elastic Load Balancer) Setup

### Step 1: Create IAM Role using eksctl

Download IAM Policy:

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json
```

Create the IAM Policy:

```bash
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
```

Create IAM Service Account:

```bash
eksctl create iamserviceaccount \
  --cluster=alb-demo-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --region us-east-2 \
  --approve
```

### Step 2: Install AWS Load Balancer Controller

Add Helm Repo:

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update
```

Install the Controller:

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=alb-demo-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set vpcId=vpc-xxxxxxxxxxxxxxxxx \
  --set image.repository=xxxxxxxxxx.dkr.ecr.us-east-2.amazonaws.com/amazon/aws-load-balancer-controller
```

Install CRDs (if not already installed):

```bash
wget https://raw.githubusercontent.com/aws/eks-charts/master/stable/aws-load-balancer-controller/crds/crds.yaml
kubectl apply -f crds.yaml
```

Verify Installation:

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

---

## 💾 EBS CSI Driver Setup

### Step 1: Create IAM Policy for EBS CSI Driver

Save the following to `ebs-csi-policy.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:AttachVolume",
        "ec2:DetachVolume",
        "ec2:CreateVolume",
        "ec2:DeleteVolume",
        "ec2:ModifyVolume",
        "ec2:DescribeVolumes",
        "ec2:DescribeVolumeAttribute",
        "ec2:DescribeVolumeStatus",
        "ec2:DescribeInstances"
      ],
      "Resource": "*"
    }
  ]
}
```

Create the IAM Policy:

```bash
aws iam create-policy \
  --policy-name AmazonEKS_EBS_CSI_Driver_Policy \
  --policy-document file://ebs-csi-policy.json
```

### Step 2: Create IAM Role & Service Account

```bash
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster <cluster-name> \
  --attach-policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/AmazonEKS_EBS_CSI_Driver_Policy \
  --approve \
  --override-existing-serviceaccounts
```

Verify:

```bash
kubectl get serviceaccount ebs-csi-controller-sa -n kube-system
```

### Step 3: Install EBS CSI Driver

```bash
helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
helm repo update

helm install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver \
  --namespace kube-system \
  --set controller.serviceAccount.create=false \
  --set controller.serviceAccount.name=ebs-csi-controller-sa
```

### Step 4: Create StorageClass

Save the following to `ebs-sc.yaml`:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  fsType: ext4
  encrypted: "true"
  kmsKeyId: "arn:aws:kms:REGION:ACCOUNT_ID:key/KEY_ID"
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

Apply:

```bash
kubectl apply -f ebs-sc.yaml
```

### Step 5: Create PVC

Save to `ebs-pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: ebs-sc
```

Apply:

```bash
kubectl apply -f ebs-pvc.yaml
```

### Step 6: Create Pod with PVC

Save to `app-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
    - name: app-container
      image: nginx
      volumeMounts:
        - mountPath: "/data"
          name: ebs-volume
  volumes:
    - name: ebs-volume
      persistentVolumeClaim:
        claimName: ebs-pvc
```

Apply:

```bash
kubectl apply -f app-pod.yaml
```

### Step 7: Verify Everything

```bash
kubectl get pvc
kubectl describe pod app-pod
```

Go to AWS Console → Elastic Block Store → Volumes to check the volume.
