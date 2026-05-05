# 2. Security Groups

Create security groups for the ALB, Jenkins EC2, EKS worker nodes, and EKS control plane.

## Variables

```bash
source ./eks_env.sh
```

## 2.1 ALB Security Group

Allows HTTP/HTTPS inbound from the internet:

```bash
ALB_SG_ID=$(aws ec2 create-security-group \
  --region $REGION \
  --group-name ${PREFIX}-alb-sg \
  --description "ALB security group for ${PREFIX}" \
  --vpc-id $VPC_ID \
  --query 'GroupId' \
  --output text)

echo "ALB_SG_ID=$ALB_SG_ID"
```

```bash
aws ec2 authorize-security-group-ingress \
  --region $REGION \
  --group-id $ALB_SG_ID \
  --ip-permissions '[
    {"IpProtocol":"tcp","FromPort":80,"ToPort":80,"IpRanges":[{"CidrIp":"0.0.0.0/0","Description":"HTTP from internet"}]},
    {"IpProtocol":"tcp","FromPort":443,"ToPort":443,"IpRanges":[{"CidrIp":"0.0.0.0/0","Description":"HTTPS from internet"}]}
  ]'
```

## 2.2 Jenkins Security Group

Allows port 8080 inbound from the ALB only:

```bash
JENKINS_SG_ID=$(aws ec2 create-security-group \
  --region $REGION \
  --group-name ${PREFIX}-jenkins-sg \
  --description "Jenkins EC2 security group for ${PREFIX}" \
  --vpc-id $VPC_ID \
  --query 'GroupId' \
  --output text)

echo "JENKINS_SG_ID=$JENKINS_SG_ID"
```

```bash
aws ec2 authorize-security-group-ingress \
  --region $REGION \
  --group-id $JENKINS_SG_ID \
  --ip-permissions "[
    {
      \"IpProtocol\":\"tcp\",
      \"FromPort\":8080,
      \"ToPort\":8080,
      \"UserIdGroupPairs\":[
        {\"GroupId\":\"$ALB_SG_ID\",\"Description\":\"Jenkins UI from ALB\"}
      ]
    }
  ]"
```

> **Note:** There is no separately-created "node" security group. EKS auto-creates a **cluster security group** when the cluster is created and attaches it to both the control plane ENIs and the worker node ENIs. ALB→node, RDS-from-node, and similar rules are added to that auto-created SG in [04-eks-cluster.md](04-eks-cluster.md) once its ID is known.

## 2.3 EKS Control Plane Security Group

A dedicated SG attached to the EKS cluster that allows inbound HTTPS from Jenkins:

```bash
CONTROL_PLANE_SG_ID=$(aws ec2 create-security-group \
  --region $REGION \
  --group-name ${PREFIX}-eks-control-plane-sg \
  --description "EKS control plane SG for ${PREFIX}-eks" \
  --vpc-id $VPC_ID \
  --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=${PREFIX}-eks-control-plane-sg}]" \
  --query 'GroupId' \
  --output text)

echo "CONTROL_PLANE_SG_ID=$CONTROL_PLANE_SG_ID"
```

```bash
aws ec2 authorize-security-group-ingress \
  --region $REGION \
  --group-id $CONTROL_PLANE_SG_ID \
  --ip-permissions "IpProtocol=tcp,FromPort=443,ToPort=443,UserIdGroupPairs=[{GroupId=${JENKINS_SG_ID},Description='Jenkins to EKS control plane'}]"
```

> **Note:** Communication between the EKS-managed cluster SG (attached to nodes) and the control plane is handled by EKS automatically; no rule for that is needed here.

## Summary

| Security Group | Inbound Rules |
|---|---|
| `<PREFIX>-alb-sg` | 80/443 from `0.0.0.0/0` |
| `<PREFIX>-jenkins-sg` | 8080 from ALB SG |
| `<PREFIX>-eks-control-plane-sg` | 443 from Jenkins SG |
| EKS-managed cluster SG (created in step 4) | 80/443 from ALB SG (added in step 4) |

## Save Variables

```bash
cat >> eks_env.sh <<EOF

# Step 2: Security Groups
export ALB_SG_ID=$ALB_SG_ID
export JENKINS_SG_ID=$JENKINS_SG_ID
export CONTROL_PLANE_SG_ID=$CONTROL_PLANE_SG_ID
EOF
echo "Variables saved to eks_env.sh"
```
