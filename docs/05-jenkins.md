# 5. Jenkins Server

Deploy a Jenkins server on EC2 behind an ALB with HTTPS (ACM certificate), accessible via a custom domain. Jenkins runs as a Docker container on Amazon Linux 2023.

## Variables

```bash
source ./eks_env.sh
```

## 5.1 Request ACM Certificate

```bash
CERT_ARN=$(aws acm request-certificate \
  --region $REGION \
  --domain-name $JENKINS_DOMAIN \
  --validation-method DNS \
  --query 'CertificateArn' \
  --output text)

echo "CERT_ARN=$CERT_ARN"
```

Get the DNS validation record:

```bash
aws acm describe-certificate \
  --region $REGION \
  --certificate-arn $CERT_ARN \
  --query 'Certificate.DomainValidationOptions[*].ResourceRecord'
```

**Add the CNAME record to your DNS provider**, then wait for validation:

```bash
# Poll until status is ISSUED
aws acm describe-certificate \
  --region $REGION \
  --certificate-arn $CERT_ARN \
  --query 'Certificate.Status'
```

## 5.2 Launch Jenkins EC2 Instance

### Get the Latest AL2023 AMI

```bash
AL2023_AMI=$(aws ssm get-parameter \
  --region $REGION \
  --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --query 'Parameter.Value' \
  --output text)

echo "AL2023_AMI=$AL2023_AMI"
```

### Create User Data Script

```bash
cat > jenkins-user-data.sh <<'EOF'
#!/bin/bash
set -euxo pipefail

dnf update -y
dnf install -y docker git

systemctl enable docker
systemctl start docker

mkdir -p /opt/jenkins_home
chown -R 1000:1000 /opt/jenkins_home

docker pull jenkins/jenkins:lts-jdk21

docker run -d \
  --name jenkins \
  --restart unless-stopped \
  -p 8080:8080 \
  -p 50000:50000 \
  -v /opt/jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkins/jenkins:lts-jdk21
EOF
```

### Launch the Instance

```bash
JENKINS_INSTANCE_ID=$(aws ec2 run-instances \
  --region $REGION \
  --image-id $AL2023_AMI \
  --instance-type t3.large \
  --iam-instance-profile Name=$JENKINS_PROFILE_NAME \
  --security-group-ids $JENKINS_SG_ID \
  --subnet-id $SUBNET_INFRA \
  --block-device-mappings '[{"DeviceName":"/dev/xvda","Ebs":{"VolumeSize":40,"VolumeType":"gp3","DeleteOnTermination":true}}]' \
  --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=${PREFIX}-jenkins}]" \
  --user-data file://jenkins-user-data.sh \
  --query 'Instances[0].InstanceId' \
  --output text)

echo "JENKINS_INSTANCE_ID=$JENKINS_INSTANCE_ID"
```

Wait for it to be running:

```bash
aws ec2 wait instance-running \
  --region $REGION \
  --instance-ids $JENKINS_INSTANCE_ID
```

Verify:

```bash
aws ec2 describe-instances \
  --region $REGION \
  --instance-ids $JENKINS_INSTANCE_ID \
  --query 'Reservations[0].Instances[0].{PrivateIp:PrivateIpAddress,State:State.Name}'
```

### Verify SSM Connectivity

```bash
aws ssm describe-instance-information \
  --region $REGION \
  --query "InstanceInformationList[?InstanceId=='$JENKINS_INSTANCE_ID']"
```

## 5.3 Install Tools on Jenkins Host

SSM into the instance and install kubectl and Helm:

```bash
aws ssm start-session \
  --region $REGION \
  --target $JENKINS_INSTANCE_ID
```

Inside the session:

```bash
# kubectl
cd /tmp
curl -LO "https://s3.us-west-2.amazonaws.com/amazon-eks/1.35.2/2026-02-27/bin/linux/amd64/kubectl"
curl -LO "https://s3.us-west-2.amazonaws.com/amazon-eks/1.35.2/2026-02-27/bin/linux/amd64/kubectl.sha256"
sha256sum -c kubectl.sha256
chmod +x kubectl
sudo mv kubectl /usr/local/bin/kubectl
kubectl version --client

# Helm
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

## 5.4 Create Target Group

```bash
JENKINS_TG_ARN=$(aws elbv2 create-target-group \
  --region $REGION \
  --name ${PREFIX}-jenkins-tg \
  --protocol HTTP \
  --port 8080 \
  --vpc-id $VPC_ID \
  --target-type instance \
  --health-check-protocol HTTP \
  --health-check-port traffic-port \
  --health-check-path /login \
  --matcher HttpCode=200-399 \
  --query 'TargetGroups[0].TargetGroupArn' \
  --output text)

echo "JENKINS_TG_ARN=$JENKINS_TG_ARN"
```

Register the Jenkins instance:

```bash
aws elbv2 register-targets \
  --region $REGION \
  --target-group-arn $JENKINS_TG_ARN \
  --targets Id=$JENKINS_INSTANCE_ID,Port=8080
```

## 5.5 Create Application Load Balancer

```bash
JENKINS_ALB_ARN=$(aws elbv2 create-load-balancer \
  --region $REGION \
  --name ${PREFIX}-jenkins-alb \
  --subnets $SUBNET_PUBLIC_A $SUBNET_PUBLIC_B \
  --security-groups $ALB_SG_ID \
  --scheme internet-facing \
  --type application \
  --ip-address-type ipv4 \
  --query 'LoadBalancers[0].LoadBalancerArn' \
  --output text)

echo "JENKINS_ALB_ARN=$JENKINS_ALB_ARN"
```

Get the ALB DNS name:

```bash
JENKINS_ALB_DNS=$(aws elbv2 describe-load-balancers \
  --region $REGION \
  --load-balancer-arns $JENKINS_ALB_ARN \
  --query 'LoadBalancers[0].DNSName' \
  --output text)

echo "JENKINS_ALB_DNS=$JENKINS_ALB_DNS"
```

## 5.6 Create Listeners

### HTTP → HTTPS Redirect

```bash
cat > redirect.json <<'EOF'
[
  {
    "Type": "redirect",
    "RedirectConfig": {
      "Protocol": "HTTPS",
      "Port": "443",
      "StatusCode": "HTTP_301"
    }
  }
]
EOF

aws elbv2 create-listener \
  --region $REGION \
  --load-balancer-arn $JENKINS_ALB_ARN \
  --protocol HTTP \
  --port 80 \
  --default-actions file://redirect.json
```

### HTTPS Listener (TLS 1.3)

```bash
aws elbv2 create-listener \
  --region $REGION \
  --load-balancer-arn $JENKINS_ALB_ARN \
  --protocol HTTPS \
  --port 443 \
  --certificates CertificateArn=$CERT_ARN \
  --ssl-policy ELBSecurityPolicy-TLS13-1-2-2021-06 \
  --default-actions Type=forward,TargetGroupArn=$JENKINS_TG_ARN
```

## 5.7 DNS Record

**Add a CNAME record** in your DNS provider pointing your Jenkins domain to the ALB DNS name:

```
CNAME  <JENKINS_DOMAIN>  →  <JENKINS_ALB_DNS>
```

## Verify

Check target health:

```bash
aws elbv2 describe-target-health \
  --region $REGION \
  --target-group-arn $JENKINS_TG_ARN
```

Test HTTPS access:

```bash
curl -I https://${JENKINS_DOMAIN}/login
```

Get the initial admin password (via SSM session):

```bash
sudo cat /opt/jenkins_home/secrets/initialAdminPassword
```

## Save Variables

```bash
cat >> eks_env.sh <<EOF

# Step 5: Jenkins
export CERT_ARN=$CERT_ARN
export AL2023_AMI=$AL2023_AMI
export JENKINS_INSTANCE_ID=$JENKINS_INSTANCE_ID
export JENKINS_TG_ARN=$JENKINS_TG_ARN
export JENKINS_ALB_ARN=$JENKINS_ALB_ARN
export JENKINS_ALB_DNS=$JENKINS_ALB_DNS
EOF
echo "Variables saved to eks_env.sh"
```
