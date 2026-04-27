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

## 2.3 EKS Node Security Group

Allows node-to-node traffic and HTTP/HTTPS from the ALB:

```bash
NODE_SG_ID=$(aws ec2 create-security-group \
  --region $REGION \
  --group-name ${PREFIX}-node-sg \
  --description "EKS worker nodes security group for ${PREFIX}" \
  --vpc-id $VPC_ID \
  --query 'GroupId' \
  --output text)

echo "NODE_SG_ID=$NODE_SG_ID"
```

Node-to-node (all traffic):

```bash
aws ec2 authorize-security-group-ingress \
  --region $REGION \
  --group-id $NODE_SG_ID \
  --ip-permissions "[
    {
      \"IpProtocol\":\"-1\",
      \"UserIdGroupPairs\":[
        {\"GroupId\":\"$NODE_SG_ID\",\"Description\":\"Node to node traffic\"}
      ]
    }
  ]"
```

ALB to nodes (HTTP/HTTPS):

```bash
aws ec2 authorize-security-group-ingress \
  --region $REGION \
  --group-id $NODE_SG_ID \
  --ip-permissions "[
    {
      \"IpProtocol\":\"tcp\",
      \"FromPort\":80,
      \"ToPort\":80,
      \"UserIdGroupPairs\":[
        {\"GroupId\":\"$ALB_SG_ID\",\"Description\":\"HTTP from ALB\"}
      ]
    },
    {
      \"IpProtocol\":\"tcp\",
      \"FromPort\":443,
      \"ToPort\":443,
      \"UserIdGroupPairs\":[
        {\"GroupId\":\"$ALB_SG_ID\",\"Description\":\"HTTPS from ALB\"}
      ]
    }
  ]"
```

Jenkins to node SG (for kubectl API access):

```bash
aws ec2 authorize-security-group-ingress \
  --region $REGION \
  --group-id $NODE_SG_ID \
  --ip-permissions "IpProtocol=tcp,FromPort=443,ToPort=443,UserIdGroupPairs=[{GroupId=${JENKINS_SG_ID}}]"
```

## 2.4 EKS Control Plane Security Group

A dedicated SG attached to the EKS cluster that allows inbound HTTPS from Jenkins and worker nodes:

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

aws ec2 authorize-security-group-ingress \
  --region $REGION \
  --group-id $CONTROL_PLANE_SG_ID \
  --ip-permissions "IpProtocol=tcp,FromPort=443,ToPort=443,UserIdGroupPairs=[{GroupId=${NODE_SG_ID},Description='EKS nodes to control plane'}]"
```

## Summary

| Security Group | Inbound Rules |
|---|---|
| `<PREFIX>-alb-sg` | 80/443 from `0.0.0.0/0` |
| `<PREFIX>-jenkins-sg` | 8080 from ALB SG |
| `<PREFIX>-node-sg` | All traffic from self, 80/443 from ALB SG, 443 from Jenkins SG |
| `<PREFIX>-eks-control-plane-sg` | 443 from Jenkins SG, 443 from Node SG |

## Save Variables

```bash
cat >> eks_env.sh <<EOF

# Step 2: Security Groups
export ALB_SG_ID=$ALB_SG_ID
export JENKINS_SG_ID=$JENKINS_SG_ID
export NODE_SG_ID=$NODE_SG_ID
export CONTROL_PLANE_SG_ID=$CONTROL_PLANE_SG_ID
EOF
echo "Variables saved to eks_env.sh"
```
