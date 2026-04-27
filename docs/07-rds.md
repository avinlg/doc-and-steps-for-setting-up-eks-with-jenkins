# 7. RDS Database (Optional)

Create a shared MySQL RDS instance across two infra subnets. This includes creating a second infra subnet (RDS requires subnets in at least two AZs), a dedicated security group, a subnet group, Secrets Manager secrets, and the RDS instance itself.

## Variables

```bash
source ./eks_env.sh
```

## 7.1 Create Second Infra Subnet

RDS requires subnets in at least two different AZs. Determine which AZ the existing infra subnet is in and create a new one in a different AZ:

```bash
EXISTING_INFRA_AZ=$(aws ec2 describe-subnets \
  --region $REGION \
  --subnet-ids $SUBNET_INFRA \
  --query 'Subnets[0].AvailabilityZone' \
  --output text)

# Pick a different AZ
if [ "$EXISTING_INFRA_AZ" = "${REGION}a" ]; then
  NEW_INFRA_AZ="${REGION}b"
else
  NEW_INFRA_AZ="${REGION}a"
fi

echo "Existing: $EXISTING_INFRA_AZ → New: $NEW_INFRA_AZ"
```

```bash
NEW_INFRA_SUBNET_ID=$(aws ec2 create-subnet \
  --region $REGION \
  --vpc-id $VPC_ID \
  --cidr-block 10.10.40.0/24 \
  --availability-zone $NEW_INFRA_AZ \
  --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=${PREFIX}-private-infra-2}]" \
  --query 'Subnet.SubnetId' \
  --output text)

echo "NEW_INFRA_SUBNET_ID=$NEW_INFRA_SUBNET_ID"
```

Associate it with the existing private route table:

```bash
PRIVATE_RT_ID=$(aws ec2 describe-route-tables \
  --region $REGION \
  --filters "Name=association.subnet-id,Values=${SUBNET_INFRA}" \
  --query 'RouteTables[0].RouteTableId' \
  --output text)

aws ec2 associate-route-table \
  --region $REGION \
  --subnet-id $NEW_INFRA_SUBNET_ID \
  --route-table-id $PRIVATE_RT_ID
```

## 7.2 RDS Security Group

Allow MySQL access from EKS nodes and Jenkins:

```bash
SHARED_RDS_SG_ID=$(aws ec2 create-security-group \
  --region $REGION \
  --group-name ${PREFIX}-shared-rds-sg \
  --description "Shared RDS SG for ${PREFIX}" \
  --vpc-id $VPC_ID \
  --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=${PREFIX}-shared-rds-sg}]" \
  --query 'GroupId' \
  --output text)

echo "SHARED_RDS_SG_ID=$SHARED_RDS_SG_ID"
```

```bash
aws ec2 authorize-security-group-ingress \
  --region $REGION \
  --group-id $SHARED_RDS_SG_ID \
  --ip-permissions "IpProtocol=tcp,FromPort=3306,ToPort=3306,UserIdGroupPairs=[{GroupId=${NODE_SG_ID},Description='EKS nodes to shared RDS'}]"

aws ec2 authorize-security-group-ingress \
  --region $REGION \
  --group-id $SHARED_RDS_SG_ID \
  --ip-permissions "IpProtocol=tcp,FromPort=3306,ToPort=3306,UserIdGroupPairs=[{GroupId=${JENKINS_SG_ID},Description='Jenkins EC2 to shared RDS'}]"
```

> **Note:** You may also need to allow the EKS cluster security group (the auto-created one) if pods connect to RDS through the cluster SG rather than the node SG.

## 7.3 DB Subnet Group

```bash
export SHARED_RDS_SUBNET_GROUP=${PREFIX}-shared-rds-subnet-group

aws rds create-db-subnet-group \
  --region $REGION \
  --db-subnet-group-name $SHARED_RDS_SUBNET_GROUP \
  --db-subnet-group-description "Shared RDS subnet group for ${PREFIX}" \
  --subnet-ids $SUBNET_INFRA $NEW_INFRA_SUBNET_ID \
  --tags Key=Name,Value=${SHARED_RDS_SUBNET_GROUP} Key=Project,Value=${PREFIX}
```

## 7.4 Secrets Manager

Generate and store passwords:

```bash
RDS_MASTER_PASSWORD=$(openssl rand -base64 32 | tr -dc 'A-Za-z0-9' | head -c 24)
STAGE_DB_PASSWORD=$(openssl rand -base64 32 | tr -dc 'A-Za-z0-9' | head -c 24)
PROD_DB_PASSWORD=$(openssl rand -base64 32 | tr -dc 'A-Za-z0-9' | head -c 24)

aws secretsmanager create-secret \
  --region $REGION \
  --name ${PREFIX}/shared/rds/master \
  --secret-string "{\"username\":\"db_admin\",\"password\":\"${RDS_MASTER_PASSWORD}\"}"

aws secretsmanager create-secret \
  --region $REGION \
  --name ${PREFIX}/stage/laravel/db \
  --secret-string "{\"database\":\"stage_db\",\"username\":\"stage_user\",\"password\":\"${STAGE_DB_PASSWORD}\"}"

aws secretsmanager create-secret \
  --region $REGION \
  --name ${PREFIX}/prod/laravel/db \
  --secret-string "{\"database\":\"prod_db\",\"username\":\"prod_user\",\"password\":\"${PROD_DB_PASSWORD}\"}"
```

## 7.5 Create the RDS Instance

```bash
export SHARED_DB_ID=${PREFIX}-dyo-db
export SHARED_DB_CLASS=db.t4g.small

aws rds create-db-instance \
  --region $REGION \
  --db-instance-identifier $SHARED_DB_ID \
  --engine mysql \
  --db-instance-class $SHARED_DB_CLASS \
  --allocated-storage 100 \
  --storage-type gp3 \
  --master-username db_admin \
  --master-user-password "$RDS_MASTER_PASSWORD" \
  --vpc-security-group-ids $SHARED_RDS_SG_ID \
  --db-subnet-group-name $SHARED_RDS_SUBNET_GROUP \
  --no-publicly-accessible \
  --backup-retention-period 7 \
  --deletion-protection \
  --tags Key=Name,Value=${SHARED_DB_ID} Key=Project,Value=${PREFIX}
```

Wait for it to be available:

```bash
aws rds wait db-instance-available \
  --region $REGION \
  --db-instance-identifier $SHARED_DB_ID
```

Get the endpoint:

```bash
RDS_ENDPOINT=$(aws rds describe-db-instances \
  --region $REGION \
  --db-instance-identifier $SHARED_DB_ID \
  --query 'DBInstances[0].Endpoint.Address' \
  --output text)

echo "RDS_ENDPOINT=$RDS_ENDPOINT"
```

## 7.6 Initialize Databases

From the Jenkins instance (via SSM), run:

```bash
cat > /tmp/init-shared-rds.sql <<EOF
CREATE DATABASE IF NOT EXISTS stage_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE DATABASE IF NOT EXISTS prod_db  CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

CREATE USER IF NOT EXISTS 'stage_user'@'%' IDENTIFIED BY '${STAGE_DB_PASSWORD}';
CREATE USER IF NOT EXISTS 'prod_user'@'%'  IDENTIFIED BY '${PROD_DB_PASSWORD}';

GRANT ALL PRIVILEGES ON stage_db.* TO 'stage_user'@'%';
GRANT ALL PRIVILEGES ON prod_db.*  TO 'prod_user'@'%';

FLUSH PRIVILEGES;
EOF

docker run --rm -i -e MYSQL_PWD="$RDS_MASTER_PASSWORD" mysql:8 \
  mysql -h "$RDS_ENDPOINT" -P 3306 -u db_admin < /tmp/init-shared-rds.sql
```

Verify:

```bash
docker run --rm -e MYSQL_PWD="$RDS_MASTER_PASSWORD" mysql:8 \
  mysql -h "$RDS_ENDPOINT" -P 3306 -u db_admin -e "SHOW DATABASES;"
```

## Save Variables

> **Note:** Database passwords are not saved here — they are stored in Secrets Manager.

```bash
cat >> eks_env.sh <<EOF

# Step 7: RDS
export NEW_INFRA_SUBNET_ID=$NEW_INFRA_SUBNET_ID
export SHARED_RDS_SG_ID=$SHARED_RDS_SG_ID
export SHARED_RDS_SUBNET_GROUP=$SHARED_RDS_SUBNET_GROUP
export SHARED_DB_ID=$SHARED_DB_ID
export RDS_ENDPOINT=$RDS_ENDPOINT
EOF
echo "Variables saved to eks_env.sh"
```
