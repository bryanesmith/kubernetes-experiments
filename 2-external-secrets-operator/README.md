# EKS, Secrets Manager, and External Secrets Operator

## Goals

* Demonstrate syncing secrets from AWS Secrets Manager to Kubernetes Secrets using [External Secrets Operator](https://external-secrets.io/)

## Technology


## Steps

1. Prereqs:
    1. Configure AWS credentials (e.g., `~/.aws/credentials`) for AWS user with sufficient privileges (e.g., admin)
    1. Install `aws` (CLI)
    1. Install `eksctl`
    1. Install Helm
    1. Set up some environment variables:
        ```bash
        REGION=us-east-1
        SECRETNAME=eso-demo-secret
        CLUSTERNAME=eso-demo-cluster
        USERNAME=eso-demo-user
        POLICYNAME=eso-demo-policy
        ```

1. Create secrets in Secret Manager
    ```bash
    aws secretsmanager create-secret --name "$SECRETNAME" \
    --secret-string '{"username":"sam", "password":"howell6"}' \
    --region "$REGION"
    ```

1. Create an EKS cluster
    ```bash
    time eksctl create cluster -f k8s/cluster.yaml

    kubectl get nodes # validate 3 nodes
    ```

1. Install [External Secrets Operator](https://external-secrets.io/latest/)
    ```bash
    helm repo add external-secrets https://charts.external-secrets.io

    helm repo ls # verify repo added

    helm install external-secrets \
        external-secrets/external-secrets \
        -n external-secrets \
        --create-namespace 
    
    helm ls -n external-secrets # verify external secrets chart installed
    ```

1. Create IAM user for External Secrets Operator:
    ```bash
    # create policy
    POLICY_ARN=$(aws --region "$REGION" --query Policy.Arn --output text iam create-policy --policy-name "$POLICYNAME" --policy-document file://./aws/policy.json)

    # create user
    aws iam create-user --user-name "$USERNAME"

    # attach policy to user
    aws iam attach-user-policy --user-name "$USERNAME" --policy-arn arn:aws:iam::aws:policy/"$POLICYNAME"
    ```

1. Fetch access key for IAM user:
    ```bash
    aws iam create-access-key --user-name "$USERNAME" # you'll need this in next step
    ```

1. Create Kubernetes secret containing IAM user access key:
    ```bash
    # use values from previous step
    kubectl create secret generic eso-demo-secret-store-access-key --from-literal=access-key=<access-key> --from-literal=secret-access-key=<secret-access-key>

    kubectl get secret # validate eso-demo-secret-store-access-key exists
    ```

1. Create the External Secrets Operator `SecretStore`:
    ```bash
    kubectl apply -f k8s/secret-store.yaml

    kubectl get SecretStore # validate eso-demo-secret-store exists
    ```

1. To fetch the secret, create a `ExternalSecret` and validate the Kubernetes secret was created:
    ```bash
    kubectl apply -f k8s/external-secret.yaml

    # validate eso-demo-exteral-secret exists, dont proceed until status is `SecretSynced`
    kubectl get ExternalSecret 

    # validate the secret was created
    kubectl get secret 
    kubectl get secret eso-demo-secret -o jsonpath="{.data.username}" | base64 --decode
    kubectl get secret eso-demo-secret -o jsonpath="{.data.password}" | base64 --decode
    ```

1. Let's validate ESO will resync changed passwords:
    ```bash
    # modify secret in AWS Secrets Manager
    aws secretsmanager create-secret --name "$SECRETNAME" \
    --secret-string '{"username":"jayden", "password":"daniels5"}' \
    --region "$REGION"

    # wait a minute
    sleep 75

    # ensure secret was updated
    kubectl get secret eso-demo-secret -o jsonpath="{.data.username}" | base64 --decode
    kubectl get secret eso-demo-secret -o jsonpath="{.data.password}" | base64 --decode
    ```

1. Clean up

    1. Delete secrets from Secrets Manager
        ```bash
        aws secretsmanager secret-id "$SECRETNAME"
        ```

    1. Delete EKS cluster
        ```bash
        time eksctl delete cluster -f k8s/cluster.yaml --disable-nodegroup-eviction --wait
        ```
    
    1. Delete IAM user and policy
        ```bash
        # detach policy from user
        aws iam detach-user-policy --user-name "$USERNAME" --policy-arn arn:aws:iam::aws:policy/"$POLICYNAME"

        # delete user
        aws iam delete-user --user-name "$USERNAME"

        # delete policy
        aws iam delete-policy --policy-arn "$POLICY_ARN"
        ```