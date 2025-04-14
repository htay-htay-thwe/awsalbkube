# 📦 AWS EKS Setup Guide: Load Balancer & EBS Storage

This guide helps you configure **AWS Load Balancer Controller** and **EBS CSI Driver** for your EKS cluster using `eksctl` and `Helm`.

---

## ✅ ELB (Elastic Load Balancer) Setup

### Step 1: Create IAM Role using `eksctl`

#### 1. Download IAM Policy

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json
2. Create IAM Policy
bash
Copy
Edit
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
3. Create IAM Service Account
bash
Copy
Edit
eksctl create iamserviceaccount \
  --cluster=alb-demo-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --region us-east-2 \
  --approve
Step 2: Install AWS Load Balancer Controller
1. Add and Update Helm Repo
bash
Copy
Edit
helm repo add eks https://aws.github.io/eks-charts
helm repo update
2. Install Controller via Helm
bash
Copy
Edit
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=alb-demo-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set vpcId=vpc-xxxxxxxx \
  --set image.repository=xxxxxxxx.dkr.ecr.us-east-2.amazonaws.com/amazon/aws-load-balancer-controller
ℹ️ Note: If using helm upgrade, install CRDs manually:

bash
Copy
Edit
wget https://raw.githubusercontent.com/aws/eks-charts/master/stable/aws-load-balancer-controller/crds/crds.yaml
kubectl apply -f crds.yaml
3. Verify Installation
bash
Copy
Edit
kubectl get deployment -n kube-system aws-load-balancer-controller
💾 EBS CSI Driver Setup
Step 1: Create IAM Policy
Save the policy to ebs-csi-policy.json
json
Copy
Edit
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
Create IAM Policy
bash
Copy
Edit
aws iam create-policy \
  --policy-name AmazonEKS_EBS_CSI_Driver_Policy \
  --policy-document file://ebs-csi-policy.json
Step 2: Create Service Account with IAM Role
bash
Copy
Edit
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster <cluster-name> \
  --attach-policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/AmazonEKS_EBS_CSI_Driver_Policy \
  --approve \
  --override-existing-serviceaccounts
bash
Copy
Edit
kubectl get serviceaccount ebs-csi-controller-sa -n kube-system
Step 3: Install the EBS CSI Driver
bash
Copy
Edit
helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
helm repo update
bash
Copy
Edit
helm install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver \
  --namespace kube-system \
  --set controller.serviceAccount.create=false \
  --set controller.serviceAccount.name=ebs-csi-controller-sa
Step 4: Create a StorageClass
yaml
Copy
Edit
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
Step 5: Create a PersistentVolumeClaim (PVC)
yaml
Copy
Edit
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
bash
Copy
Edit
kubectl apply -f ebs-pvc.yaml
Step 6: Deploy a Pod Using the PVC
yaml
Copy
Edit
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
bash
Copy
Edit
kubectl apply -f app-pod.yaml
Step 7: Verification
bash
Copy
Edit
kubectl get pvc
kubectl describe pod app-pod
Then check in AWS Console:
Elastic Block Store → Volumes
