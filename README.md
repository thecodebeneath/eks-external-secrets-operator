# K8s External Secrets Operator

Ref: https://aws.amazon.com/blogs/containers/leverage-aws-secrets-stores-from-eks-fargate-with-external-secrets-operator/

## Provison EKS Cluster
From the AWS console, use the Create Cluster option: *"Quick configuration (with EKS Auto Mode) - new"*

## Env Setup
```
export REGION=us-east-2
export ACCNT=$(aws sts get-caller-identity --query Account --output text)
export CLUSTERNAME=floral-metal-dolphin
aws eks update-kubeconfig --region "$REGION" --name "$CLUSTERNAME"
```

## Create Secrets Manager entry
Create "Other" type secret, named: "proj/jeff/ProjApps", with four key/value pairs.
```
CLUSTER_TYPE: jeff-common
REGISTRY_USER: jeffadmin
REGISTRY_PASSWORD: jeffpassword!
KC_ARGOCD_CLIENT_SECRET: FGJHWHw==
```

## Install the External Secrets Operator
```
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets \
   external-secrets/external-secrets \
   -n external-secrets \
   --create-namespace \
   --set installCRDs=true \
   --set webhook.port=9443 
```

## Create the EKS OIDC Provider
```
eksctl utils associate-iam-oidc-provider --region="$REGION" --cluster="$CLUSTERNAME" --approve
```

## Create sa, role and policy for Operator
```
Create IAM policy: SecretsMgrForEksOperator
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "SecretsMgrForEksOperator",
            "Effect": "Allow",
            "Action": [
                "secretsmanager:PutSecretValue",
                "secretsmanager:GetSecretValue",
                "secretsmanager:DescribeSecret"
            ],
            "Resource": "arn:aws:secretsmanager:us-east-2:$ACCNT:secret:proj/jeff/*"
        }
    ]
}

eksctl create iamserviceaccount \
    --name projjeffsa \
    --namespace default \
    --cluster "$CLUSTERNAME" \
    --role-name "ProjJeffRole" \
    --attach-policy-arn arn:aws:iam::"$ACCNT":policy/SecretsMgrForEksOperator \
    --approve \
    --override-existing-serviceaccounts
```

## Create the Secret Store
Referencing the service account with correct polict permissions
```
kubectl apply -f secret-store.yaml
```

## Create the External Secret
```
kubectl apply -f external-secret.yaml
```

## Confirm Secret is created in the namespace
```
kubectl get secret projjeffsecret -n default
```

## Teardown
kubectl delete externalsecret projjeffexternalsecret (automatically deletes the Secret projjeffsecret)
kubectl delete secretstore projjeffsecretsstore
kubectl delete serviceaccount projjeffsa
