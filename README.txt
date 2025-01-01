# Access AWS S3 and Secrets Manager from EKS using IRSA and CSI Driver

## Goal

Setup an EKS cluster, and demonstrate access to S3 using IRSA as well as mount secrets from Secrets Manager using CSI Driver

## Technology
* aws (CLI)
* EKS
* eksctl
* helm
* kubectl
* IAM
* S3
* Secrets Manager
* Secrets Store CSI Driver

## Steps:
1. Prereqs:
    a. Configure AWS credentials (e.g., ~/.aws/credentials) for AWS user with sufficient privileges (e.g., admin)
    b. Install aws (CLI)
    c. Install eksctl
    d. Install Helm
    e. Set up some environment variables:
       ```bash
       SECRETNAME=mysecret
       CLUSTERNAME=my-cluster
       SERVICEACCTNAME=my-service-account
       POLICYNAME=my-policy-name
       REGION=us-east-1
       S3BUCKETNAME="my-bucket-$(jot -r 1 10000000 99999999)"

       echo "$S3BUCKETNAME" # copy this, you'll need it later
       ```

1. Create secrets in Secret Manager
  ```
  aws secretsmanager create-secret --name "$SECRETNAME" \
  --secret-string '{"username":"gerald.piggy", "password":"banana"}' \
  --region "$REGION"
  ```

Create an S3 bucket:
aws s3 mb s3://"$S3BUCKETNAME" --region "$REGION"

Create an EKS cluster
Create cluster (can take 15-20 min):
time eksctl create cluster -f k8s/my-cluster.yaml

 Validate cluster exists with 3 nodes: 
kubectl get nodes

Create an IAM OIDC Provider for the cluster:
eksctl utils associate-iam-oidc-provider \
  --cluster "$CLUSTERNAME" \
  --region "$REGION" \
  --approve

Install Secrets Store CSI Driver on EKS cluster:
Install Secret Store CSI Driver
helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
helm install -n kube-system csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver

Wait for the Secrets Store CSI Driver to finish deploying:
kubectl --namespace=kube-system get pods -l "app=secrets-store-csi-driver" --watch

Install the AWS Provider for Secret Store CSI Driver
helm repo add aws-secrets-manager https://aws.github.io/secrets-store-csi-driver-provider-aws
helm install -n kube-system secrets-provider-aws aws-secrets-manager/secrets-store-csi-driver-provider-aws

Create a policy that has access to S3 and Secrets Manager
POLICY_ARN=$(aws --region "$REGION" --query Policy.Arn --output text iam create-policy --policy-name "$POLICYNAME" --policy-document file://./aws/policy.json)

Confirm the policy was created:
aws iam list-policies --scope Local

Create a Kubernetes service account 
eksctl create iamserviceaccount --name "$SERVICEACCTNAME" \
--region="$REGION" \
--cluster "$CLUSTERNAME" \
--attach-policy-arn "$POLICY_ARN" \
--approve --override-existing-serviceaccounts

Create a secrets provider class:
kubectl apply -f ./k8s/my-secret-provider-class.yaml

Create a pod
kubectl apply -f ./k8s/my-deployment.yaml

Validate Secrets Manager and S3 access from the pod:
Exec into the pod, and confirm you have s3 access
kubectl exec -it $(kubectl get pods | awk '/my-deployment/{print $1}' | head -1) -- /bin/bash

Validate the secrets file:
cat /mnt/secrets-store/mysecret

Install AWS CLI:
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

Put a file in S3, and verify it exists:
S3BUCKETNAME=<your bucket name>
echo foo > foo.txt
aws s3 cp foo.txt "s3://${S3BUCKETNAME}/"
aws s3 ls "s3://${S3BUCKETNAME}/"
aws s3 cp "s3://${S3BUCKETNAME}/foo.txt" foo2.txt
ls
cat foo.txt

Clean up
Clean up IAM role and policy generated for IRSA:
eksctl delete iamserviceaccount --name "$SERVICEACCTNAME" \
--region="$REGION" \
--cluster "$CLUSTERNAME"

Delete secrets from Secrets Manager
aws secretsmanager secret-id mysecret

Delete S3 bucket:
aws s3 rb s3://"$S3BUCKETNAME" --region "$REGION"

Delete EKS cluster
time eksctl delete cluster -f k8s/my-cluster.yaml --disable-nodegroup-eviction --wait

Delete IAM policy
aws iam delete-policy "$POLICY_ARN"


