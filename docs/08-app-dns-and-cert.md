# 8. Application DNS and Certificate (Optional)

Request an ACM certificate for your application domain (e.g. staging or production) and validate it via DNS. This certificate will be used by ALB Ingress annotations in your Kubernetes deployments.

## Variables

```bash
source ./eks_env.sh
```

## 8.1 Request Certificate

```bash
APP_CERT_ARN=$(aws acm request-certificate \
  --region $REGION \
  --domain-name $APP_DOMAIN \
  --validation-method DNS \
  --query CertificateArn \
  --output text)

echo "APP_CERT_ARN=$APP_CERT_ARN"
```

## 8.2 DNS Validation

Get the CNAME record to add:

```bash
aws acm describe-certificate \
  --region $REGION \
  --certificate-arn $APP_CERT_ARN \
  --query 'Certificate.DomainValidationOptions[].ResourceRecord' \
  --output table
```

**Add the CNAME record to your DNS provider.**

## 8.3 Wait for Issuance

```bash
# Poll until status is ISSUED
aws acm describe-certificate \
  --region $REGION \
  --certificate-arn $APP_CERT_ARN \
  --query 'Certificate.{Status:Status,Domain:DomainName}' \
  --output table
```

The certificate ARN can then be used in Kubernetes Ingress annotations:

```yaml
annotations:
  alb.ingress.kubernetes.io/certificate-arn: <APP_CERT_ARN>
```

## Save Variables

```bash
cat >> eks_env.sh <<EOF

# Step 8: App DNS and Certificate
export APP_CERT_ARN=$APP_CERT_ARN
EOF
echo "Variables saved to eks_env.sh"
```
