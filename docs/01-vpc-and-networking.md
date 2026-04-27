# 1. VPC and Networking

This guide sets up the entire network foundation: VPC, subnets, internet gateway, NAT gateway, route tables, S3 gateway endpoint, and Kubernetes subnet tags.

## Variables

```bash
export REGION=us-east-1
export PREFIX=<your-prefix>       # e.g. acme-prod
export EKS_VERSION=1.35
export JENKINS_DOMAIN=<your-jenkins-domain>   # e.g. jenkins.example.com
export APP_DOMAIN=<your-app-domain>            # e.g. stage-app.example.com (optional)
```

Initialize the environment file with the base variables:

```bash
cat > eks_env.sh <<EOF
export REGION=$REGION
export PREFIX=$PREFIX
export EKS_VERSION=$EKS_VERSION
export JENKINS_DOMAIN=$JENKINS_DOMAIN
export APP_DOMAIN=$APP_DOMAIN
EOF
source ./eks_env.sh
```

## 1.1 Create VPC

```bash
VPC_ID=$(aws ec2 create-vpc \
  --region $REGION \
  --cidr-block 10.10.0.0/16 \
  --tag-specifications "ResourceType=vpc,Tags=[{Key=Name,Value=${PREFIX}-vpc}]" \
  --query 'Vpc.VpcId' \
  --output text)

echo "VPC_ID=$VPC_ID"
```

## 1.2 Create Subnets

### Public Subnets (two AZs, for ALB)

```bash
SUBNET_PUBLIC_A=$(aws ec2 create-subnet \
  --region $REGION \
  --vpc-id $VPC_ID \
  --cidr-block 10.10.1.0/24 \
  --availability-zone ${REGION}a \
  --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=${PREFIX}-public-a}]" \
  --query 'Subnet.SubnetId' \
  --output text)

echo "SUBNET_PUBLIC_A=$SUBNET_PUBLIC_A"
```

```bash
SUBNET_PUBLIC_B=$(aws ec2 create-subnet \
  --region $REGION \
  --vpc-id $VPC_ID \
  --cidr-block 10.10.2.0/24 \
  --availability-zone ${REGION}b \
  --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=${PREFIX}-public-b}]" \
  --query 'Subnet.SubnetId' \
  --output text)

echo "SUBNET_PUBLIC_B=$SUBNET_PUBLIC_B"
```

Enable auto-assign public IP on both public subnets:

```bash
aws ec2 modify-subnet-attribute \
  --subnet-id $SUBNET_PUBLIC_A \
  --map-public-ip-on-launch

aws ec2 modify-subnet-attribute \
  --subnet-id $SUBNET_PUBLIC_B \
  --map-public-ip-on-launch
```

### Private App Subnets (two AZs, for EKS worker nodes)

```bash
SUBNET_APP_A=$(aws ec2 create-subnet \
  --region $REGION \
  --vpc-id $VPC_ID \
  --cidr-block 10.10.10.0/24 \
  --availability-zone ${REGION}a \
  --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=${PREFIX}-app-a}]" \
  --query 'Subnet.SubnetId' \
  --output text)

echo "SUBNET_APP_A=$SUBNET_APP_A"
```

```bash
SUBNET_APP_B=$(aws ec2 create-subnet \
  --region $REGION \
  --vpc-id $VPC_ID \
  --cidr-block 10.10.20.0/24 \
  --availability-zone ${REGION}b \
  --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=${PREFIX}-app-b}]" \
  --query 'Subnet.SubnetId' \
  --output text)

echo "SUBNET_APP_B=$SUBNET_APP_B"
```

### Private Infra Subnet (for Jenkins EC2, RDS, etc.)

```bash
SUBNET_INFRA=$(aws ec2 create-subnet \
  --region $REGION \
  --vpc-id $VPC_ID \
  --cidr-block 10.10.30.0/24 \
  --availability-zone ${REGION}a \
  --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=${PREFIX}-infra}]" \
  --query 'Subnet.SubnetId' \
  --output text)

echo "SUBNET_INFRA=$SUBNET_INFRA"
```

## 1.3 Internet Gateway

```bash
IGW_ID=$(aws ec2 create-internet-gateway \
  --region $REGION \
  --tag-specifications "ResourceType=internet-gateway,Tags=[{Key=Name,Value=${PREFIX}-igw}]" \
  --query 'InternetGateway.InternetGatewayId' \
  --output text)

echo "IGW_ID=$IGW_ID"
```

Attach to VPC:

```bash
aws ec2 attach-internet-gateway \
  --region $REGION \
  --internet-gateway-id $IGW_ID \
  --vpc-id $VPC_ID
```

## 1.4 NAT Gateway

Allocate an Elastic IP:

```bash
EIP_ALLOC_ID=$(aws ec2 allocate-address \
  --region $REGION \
  --domain vpc \
  --query 'AllocationId' \
  --output text)

echo "EIP_ALLOC_ID=$EIP_ALLOC_ID"
```

Create the NAT Gateway in a public subnet:

```bash
NAT_GW_ID=$(aws ec2 create-nat-gateway \
  --region $REGION \
  --subnet-id $SUBNET_PUBLIC_A \
  --allocation-id $EIP_ALLOC_ID \
  --tag-specifications "ResourceType=natgateway,Tags=[{Key=Name,Value=${PREFIX}-nat-a}]" \
  --query 'NatGateway.NatGatewayId' \
  --output text)

echo "NAT_GW_ID=$NAT_GW_ID"
```

Wait for it to become available:

```bash
aws ec2 wait nat-gateway-available \
  --region $REGION \
  --nat-gateway-ids $NAT_GW_ID
```

## 1.5 Route Tables

### Public Route Table

```bash
RT_PUBLIC_ID=$(aws ec2 create-route-table \
  --region $REGION \
  --vpc-id $VPC_ID \
  --tag-specifications "ResourceType=route-table,Tags=[{Key=Name,Value=${PREFIX}-public-rt}]" \
  --query 'RouteTable.RouteTableId' \
  --output text)

echo "RT_PUBLIC_ID=$RT_PUBLIC_ID"
```

Add internet route and associate public subnets:

```bash
aws ec2 create-route \
  --region $REGION \
  --route-table-id $RT_PUBLIC_ID \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id $IGW_ID

aws ec2 associate-route-table \
  --region $REGION \
  --subnet-id $SUBNET_PUBLIC_A \
  --route-table-id $RT_PUBLIC_ID

aws ec2 associate-route-table \
  --region $REGION \
  --subnet-id $SUBNET_PUBLIC_B \
  --route-table-id $RT_PUBLIC_ID
```

### Private Route Table

```bash
RT_PRIVATE_ID=$(aws ec2 create-route-table \
  --region $REGION \
  --vpc-id $VPC_ID \
  --tag-specifications "ResourceType=route-table,Tags=[{Key=Name,Value=${PREFIX}-private-rt}]" \
  --query 'RouteTable.RouteTableId' \
  --output text)

echo "RT_PRIVATE_ID=$RT_PRIVATE_ID"
```

Add NAT route and associate private subnets:

```bash
aws ec2 create-route \
  --region $REGION \
  --route-table-id $RT_PRIVATE_ID \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id $NAT_GW_ID

aws ec2 associate-route-table \
  --region $REGION \
  --subnet-id $SUBNET_APP_A \
  --route-table-id $RT_PRIVATE_ID

aws ec2 associate-route-table \
  --region $REGION \
  --subnet-id $SUBNET_APP_B \
  --route-table-id $RT_PRIVATE_ID

aws ec2 associate-route-table \
  --region $REGION \
  --subnet-id $SUBNET_INFRA \
  --route-table-id $RT_PRIVATE_ID
```

## 1.6 S3 Gateway Endpoint

Enables private S3 access (ECR image layers, etc.) without traversing the NAT gateway:

```bash
aws ec2 create-vpc-endpoint \
  --region $REGION \
  --vpc-id $VPC_ID \
  --service-name com.amazonaws.${REGION}.s3 \
  --vpc-endpoint-type Gateway \
  --route-table-ids $RT_PRIVATE_ID \
  --tag-specifications "ResourceType=vpc-endpoint,Tags=[{Key=Name,Value=${PREFIX}-s3-endpoint}]"
```

## 1.7 Kubernetes Subnet Tags

Tag public subnets for ELB discovery and all EKS-related subnets for cluster ownership:

```bash
export CLUSTER_NAME=${PREFIX}-eks

# Public subnets: allow ALB creation
aws ec2 create-tags \
  --region $REGION \
  --resources $SUBNET_PUBLIC_A $SUBNET_PUBLIC_B \
  --tags Key=kubernetes.io/role/elb,Value=1

# All subnets used by the cluster: shared ownership
aws ec2 create-tags \
  --region $REGION \
  --resources $SUBNET_PUBLIC_A $SUBNET_PUBLIC_B $SUBNET_APP_A $SUBNET_APP_B \
  --tags Key=kubernetes.io/cluster/${CLUSTER_NAME},Value=shared
```

## Verify

```bash
aws ec2 describe-route-tables \
  --region $REGION \
  --filters Name=vpc-id,Values=$VPC_ID \
  --query 'RouteTables[*].{Name:Tags[?Key==`Name`].Value|[0],RouteTableId:RouteTableId,Routes:Routes[*].[DestinationCidrBlock,GatewayId,NatGatewayId],Associations:Associations[*].SubnetId}' \
  --output table
```

Expected layout:

| Route Table | Subnets | Default Route |
|---|---|---|
| `<PREFIX>-public-rt` | public-a, public-b | → Internet Gateway |
| `<PREFIX>-private-rt` | app-a, app-b, infra | → NAT Gateway |

## Save Variables

```bash
cat >> eks_env.sh <<EOF

# Step 1: VPC and Networking
export VPC_ID=$VPC_ID
export SUBNET_PUBLIC_A=$SUBNET_PUBLIC_A
export SUBNET_PUBLIC_B=$SUBNET_PUBLIC_B
export SUBNET_APP_A=$SUBNET_APP_A
export SUBNET_APP_B=$SUBNET_APP_B
export SUBNET_INFRA=$SUBNET_INFRA
export IGW_ID=$IGW_ID
export EIP_ALLOC_ID=$EIP_ALLOC_ID
export NAT_GW_ID=$NAT_GW_ID
export RT_PUBLIC_ID=$RT_PUBLIC_ID
export RT_PRIVATE_ID=$RT_PRIVATE_ID
export CLUSTER_NAME=$CLUSTER_NAME
EOF
echo "Variables saved to eks_env.sh"
```
