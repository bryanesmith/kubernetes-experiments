# Access AWS S3 and Secrets Manager from EKS using IRSA and Secrets Store CSI Driver

## Goals

1. Demonstrate access to S3 using IRSA from EKS 
2. Mount secrets from Secrets Manager in an EKS pod using [AWS Secrets Manager Secret Store CSI Driver](https://github.com/aws/secrets-store-csi-driver-provider-aws)

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
    1. Configure AWS credentials (e.g., ~/.aws/credentials) for AWS user with sufficient privileges (e.g., admin)
    1. Install aws (CLI)
    1. Install eksctl
    1. Install Helm
    1. Set up some environment variables:
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

1. Create an S3 bucket:
    ```
    aws s3 mb s3://"$S3BUCKETNAME" --region "$REGION"
    ```

1. Create an EKS cluster
    1. Create cluster (can take 15-20 min):
        ```
        time eksctl create cluster -f k8s/my-cluster.yaml
        ```
    1. Validate cluster exists with 3 nodes: 
        ```
        kubectl get nodes
        ```
    1. Create an IAM OIDC Provider for the cluster:
        ```
        eksctl utils associate-iam-oidc-provider \
        --cluster "$CLUSTERNAME" \
        --region "$REGION" \
        --approve
        ```

1. Install Secrets Store CSI Driver on EKS cluster:
    1. Install Secret Store CSI Driver
        ```
        helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
        helm install -n kube-system csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver
        ```
    1. Wait for the Secrets Store CSI Driver to finish deploying:
        ```
        kubectl --namespace=kube-system get pods -l "app=secrets-store-csi-driver" --watch
        ```
    1. Install the AWS Provider for Secret Store CSI Driver
        ```
        helm repo add aws-secrets-manager https://aws.github.io/secrets-store-csi-driver-provider-aws
        helm install -n kube-system secrets-provider-aws aws-secrets-manager/secrets-store-csi-driver-provider-aws
        ```

1. Create a policy that has access to S3 and Secrets Manager
    ```
    POLICY_ARN=$(aws --region "$REGION" --query Policy.Arn --output text iam create-policy --policy-name "$POLICYNAME" --policy-document file://./aws/policy.json)
    ```

    Confirm the policy was created:
    ```
    aws iam list-policies --scope Local
    ```

1. Create a Kubernetes service account 
    ```
    eksctl create iamserviceaccount --name "$SERVICEACCTNAME" \
    --region="$REGION" \
    --cluster "$CLUSTERNAME" \
    --attach-policy-arn "$POLICY_ARN" \
    --approve --override-existing-serviceaccounts 
    ```

1. Create a secrets provider class:
    ```
    kubectl apply -f ./k8s/my-secret-provider-class.yaml
    ```

1. Create a pod
    ```
    kubectl apply -f ./k8s/my-deployment.yaml
    ```

1. Validate Secrets Manager and S3 access from the pod:
    1. Exec into the pod
        ```
        kubectl exec -it $(kubectl get pods | awk '/my-deployment/{print $1}' | head -1) -- /bin/bash
        ```

    1. Validate the secrets file:
        ```
        cat /mnt/secrets-store/mysecret
        ```

    1. Install AWS CLI:
        ```
        /usr/bin/apt-get update && /usr/bin/apt-get install -y awscli
        ```

    1. Put a file in S3, and verify it exists:
        ```
        S3BUCKETNAME=<your bucket name>
        echo foo > foo.txt
        aws s3 cp foo.txt "s3://${S3BUCKETNAME}/"
        aws s3 cp "s3://${S3BUCKETNAME}/foo.txt" foo2.txt
        ls
        cat foo.txt
        exit
        ```

1. Clean up
    1. Clean up IAM role and policy generated for IRSA:
        ```
        eksctl delete iamserviceaccount --name "$SERVICEACCTNAME" \
        --region="$REGION" \
        --cluster "$CLUSTERNAME"
        ```

    1. Delete secrets from Secrets Manager
        ```
        aws secretsmanager delete-secret --secret-id "$SECRETNAME"
        ```

    1. Delete S3 bucket:
        ```
        aws s3 rb s3://"$S3BUCKETNAME" --region "$REGION"
        ```

    1. Delete EKS cluster
        ```
        time eksctl delete cluster -f k8s/my-cluster.yaml --disable-nodegroup-eviction --wait
        ```

    1. Delete IAM policy 
        ```
        aws iam delete-policy --policy-arn "$POLICY_ARN"
        ```
