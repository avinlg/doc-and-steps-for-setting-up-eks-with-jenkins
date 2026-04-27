# 6. ECR Repositories (Optional)

Create ECR repositories for container images with scanning, immutable tags, and lifecycle policies.

## Variables

```bash
source ./eks_env.sh
export ECR_REGISTRY="${ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com"
```

## 6.1 Create Repositories

Each repository is created with immutable tags and scan-on-push enabled:

```bash
aws ecr create-repository \
  --region $REGION \
  --repository-name <repo-name> \
  --image-scanning-configuration scanOnPush=true \
  --image-tag-mutability IMMUTABLE \
  --encryption-configuration encryptionType=AES256
```

Repeat for each application image needed. For example:

```bash
aws ecr create-repository --region $REGION --repository-name ${PREFIX}-dyo \
  --image-scanning-configuration scanOnPush=true --image-tag-mutability IMMUTABLE

aws ecr create-repository --region $REGION --repository-name ${PREFIX}-laravel-php \
  --image-scanning-configuration scanOnPush=true --image-tag-mutability IMMUTABLE

aws ecr create-repository --region $REGION --repository-name ${PREFIX}-laravel-nginx \
  --image-scanning-configuration scanOnPush=true --image-tag-mutability IMMUTABLE
```

## 6.2 Lifecycle Policy

Keep only the last 30 tagged images to control storage costs:

```bash
cat > ecr-lifecycle.json <<'EOF'
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Keep last 30 images",
      "selection": {
        "tagStatus": "tagged",
        "tagPatternList": ["*"],
        "countType": "imageCountMoreThan",
        "countNumber": 30
      },
      "action": {
        "type": "expire"
      }
    }
  ]
}
EOF
```

Apply to each repository:

```bash
aws ecr put-lifecycle-policy \
  --region $REGION \
  --repository-name <repo-name> \
  --lifecycle-policy-text file://ecr-lifecycle.json
```

## Verify

```bash
aws ecr describe-repositories \
  --region $REGION \
  --repository-names <repo-name> \
  --query 'repositories[*].{name:repositoryName,uri:repositoryUri,mutability:imageTagMutability,scanOnPush:imageScanningConfiguration.scanOnPush}' \
  --output table
```

## Save Variables

```bash
cat >> eks_env.sh <<EOF

# Step 6: ECR
export ECR_REGISTRY=$ECR_REGISTRY
EOF
echo "Variables saved to eks_env.sh"
```
