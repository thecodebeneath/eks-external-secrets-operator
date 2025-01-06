# K8s External Secrets Operator

Ref: https://aws.amazon.com/blogs/containers/leverage-aws-secrets-stores-from-eks-fargate-with-external-secrets-operator/

## Provison EKS Cluster
From the AWS console, use the Create Cluster option: *"Quick configuration (with EKS Auto Mode) - new"*

## Env Setup
```
export REGION=us-east-2
export ACCNT=$(aws sts get-caller-identity --query Account --output text)
export CLUSTERNAME=extravagant-lofi-dinosaur
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
   --create-namespace
```

## Create the EKS OIDC Provider
Automatically sets the OIDC Audience (aka Client ID) to "sts.amazonaws.com"
```
eksctl utils associate-iam-oidc-provider --region="$REGION" --cluster="$CLUSTERNAME" --approve
```

## Create sa, role and policy for Operator
```
Create IAM policy: ProjJeffSecretsMgrForEksOperator
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "ProjJeffSecretsMgrForEksOperator",
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
    --namespace jeff \
    --cluster "$CLUSTERNAME" \
    --role-name "ProjJeffRole" \
    --attach-policy-arn arn:aws:iam::"$ACCNT":policy/ProjJeffSecretsMgrForEksOperator \
    --approve \
    --override-existing-serviceaccounts
```

The IAM Role that gets created will have these Trusted Relationship entities attached
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::$ACCNT:oidc-provider/oidc.eks.us-east-2.amazonaws.com/id/703860C207140C2357EC4ADCC3F0D6B4"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "oidc.eks.us-east-2.amazonaws.com/id/703860C207140C2357EC4ADCC3F0D6B4:sub": "system:serviceaccount:default:projjeffsa",
                    "oidc.eks.us-east-2.amazonaws.com/id/703860C207140C2357EC4ADCC3F0D6B4:aud": "sts.amazonaws.com"
                }
            }
        }
    ]
}
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
kubectl describe secret projjeffsecret -n default

Data
====
CLUSTER_TYPE:             11 bytes
REGISTRY_USER:            11 bytes
REGISTRY_PASSWORD:        11 bytes
KC_ARGOCD_CLIENT_SECRET:  9 bytes
```

## Teardown
```
kubectl delete externalsecret projjeffexternalsecret (automatically deletes the Secret projjeffsecret)
kubectl delete secretstore projjeffsecretsstore
eksctl delete iamserviceaccount projjeffsa --cluster $CLUSTERNAME
  -- this deletes iamserviceaccount, serviceaccount and IAM role
```