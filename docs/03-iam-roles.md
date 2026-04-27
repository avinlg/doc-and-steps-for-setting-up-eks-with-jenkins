# 3. IAM Roles

Create IAM roles for the EKS cluster, worker nodes, and Jenkins EC2 instance.

## Variables

```bash
source ./eks_env.sh
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

export EKS_CLUSTER_ROLE_NAME=${PREFIX}-eks-cluster-role
export EKS_NODE_ROLE_NAME=${PREFIX}-node-role
export JENKINS_ROLE_NAME=${PREFIX}-jenkins-role
```

## 3.1 EKS Cluster Role

Create the trust policy file:

```bash
cat > eks-cluster-trust-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "eks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF
```

Create the role and attach the managed policy:

```bash
aws iam create-role \
  --role-name $EKS_CLUSTER_ROLE_NAME \
  --assume-role-policy-document file://eks-cluster-trust-policy.json

aws iam attach-role-policy \
  --role-name $EKS_CLUSTER_ROLE_NAME \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy
```

```bash
EKS_CLUSTER_ROLE_ARN=$(aws iam get-role \
  --role-name $EKS_CLUSTER_ROLE_NAME \
  --query 'Role.Arn' \
  --output text)

echo "EKS_CLUSTER_ROLE_ARN=$EKS_CLUSTER_ROLE_ARN"
```

## 3.2 EKS Node Role

Create the trust policy file:

```bash
cat > eks-node-trust-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF
```

Create the role and attach managed policies:

```bash
aws iam create-role \
  --role-name $EKS_NODE_ROLE_NAME \
  --assume-role-policy-document file://eks-node-trust-policy.json

aws iam attach-role-policy \
  --role-name $EKS_NODE_ROLE_NAME \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy

aws iam attach-role-policy \
  --role-name $EKS_NODE_ROLE_NAME \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryPullOnly

aws iam attach-role-policy \
  --role-name $EKS_NODE_ROLE_NAME \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
```

```bash
EKS_NODE_ROLE_ARN=$(aws iam get-role \
  --role-name $EKS_NODE_ROLE_NAME \
  --query 'Role.Arn' \
  --output text)

echo "EKS_NODE_ROLE_ARN=$EKS_NODE_ROLE_ARN"
```

## 3.3 Jenkins Role

### Create Role

```bash
cat > jenkins-ec2-trust-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF
```

```bash
aws iam create-role \
  --role-name $JENKINS_ROLE_NAME \
  --assume-role-policy-document file://jenkins-ec2-trust-policy.json
```

### Attach Managed Policies

```bash
aws iam attach-role-policy \
  --role-name $JENKINS_ROLE_NAME \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

aws iam attach-role-policy \
  --role-name $JENKINS_ROLE_NAME \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryPowerUser
```

### Create Custom Jenkins Policy

This policy grants EKS describe access, Secrets Manager read access (scoped), and KMS decrypt for secrets:

```bash
cat > ${PREFIX}-jenkins-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DescribeEKS",
      "Effect": "Allow",
      "Action": [
        "eks:DescribeCluster",
        "eks:ListClusters",
        "eks:DescribeNodegroup",
        "eks:ListNodegroups",
        "eks:DescribeAddon",
        "eks:ListAddons"
      ],
      "Resource": "*"
    },
    {
      "Sid": "ReadSpecificSecrets",
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": [
        "arn:aws:secretsmanager:${REGION}:${ACCOUNT_ID}:secret:${PREFIX}/mongodb*",
        "arn:aws:secretsmanager:${REGION}:${ACCOUNT_ID}:secret:${PREFIX}/rds*",
        "arn:aws:secretsmanager:${REGION}:${ACCOUNT_ID}:secret:${PREFIX}/api*",
        "arn:aws:secretsmanager:${REGION}:${ACCOUNT_ID}:secret:${PREFIX}/jenkins*"
      ]
    },
    {
      "Sid": "DecryptSecretsIfCustomerManagedKMSUsed",
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "kms:ViaService": "secretsmanager.${REGION}.amazonaws.com"
        }
      }
    }
  ]
}
EOF
```

```bash
aws iam create-policy \
  --policy-name ${PREFIX}-jenkins-policy \
  --policy-document file://${PREFIX}-jenkins-policy.json
```

```bash
JENKINS_CUSTOM_POLICY_ARN=$(aws iam list-policies \
  --scope Local \
  --query "Policies[?PolicyName=='${PREFIX}-jenkins-policy'].Arn | [0]" \
  --output text)

echo "JENKINS_CUSTOM_POLICY_ARN=$JENKINS_CUSTOM_POLICY_ARN"

aws iam attach-role-policy \
  --role-name $JENKINS_ROLE_NAME \
  --policy-arn $JENKINS_CUSTOM_POLICY_ARN
```

### Create Instance Profile

```bash
aws iam create-instance-profile \
  --instance-profile-name ${PREFIX}-jenkins-instance-profile

aws iam add-role-to-instance-profile \
  --instance-profile-name ${PREFIX}-jenkins-instance-profile \
  --role-name $JENKINS_ROLE_NAME
```

```bash
JENKINS_ROLE_ARN=$(aws iam get-role \
  --role-name $JENKINS_ROLE_NAME \
  --query 'Role.Arn' \
  --output text)

echo "JENKINS_ROLE_ARN=$JENKINS_ROLE_ARN"
```

## Verify

```bash
aws iam list-attached-role-policies --role-name $EKS_CLUSTER_ROLE_NAME
aws iam list-attached-role-policies --role-name $EKS_NODE_ROLE_NAME
aws iam list-attached-role-policies --role-name $JENKINS_ROLE_NAME
```

## Summary

| Role | Attached Policies |
|---|---|
| `<PREFIX>-eks-cluster-role` | `AmazonEKSClusterPolicy` |
| `<PREFIX>-node-role` | `AmazonEKSWorkerNodePolicy`, `AmazonEC2ContainerRegistryPullOnly`, `AmazonEKS_CNI_Policy` |
| `<PREFIX>-jenkins-role` | `AmazonSSMManagedInstanceCore`, `AmazonEC2ContainerRegistryPowerUser`, `<PREFIX>-jenkins-policy` (custom) |

## Save Variables

```bash
cat >> eks_env.sh <<EOF

# Step 3: IAM Roles
export ACCOUNT_ID=$ACCOUNT_ID
export EKS_CLUSTER_ROLE_NAME=$EKS_CLUSTER_ROLE_NAME
export EKS_NODE_ROLE_NAME=$EKS_NODE_ROLE_NAME
export EKS_CLUSTER_ROLE_ARN=$EKS_CLUSTER_ROLE_ARN
export EKS_NODE_ROLE_ARN=$EKS_NODE_ROLE_ARN
export JENKINS_ROLE_NAME=$JENKINS_ROLE_NAME
export JENKINS_ROLE_ARN=$JENKINS_ROLE_ARN
export JENKINS_CUSTOM_POLICY_ARN=$JENKINS_CUSTOM_POLICY_ARN
export JENKINS_PROFILE_NAME=${PREFIX}-jenkins-instance-profile
EOF
echo "Variables saved to eks_env.sh"
```
