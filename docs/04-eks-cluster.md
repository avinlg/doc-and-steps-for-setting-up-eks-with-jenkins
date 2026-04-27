# 4. EKS Cluster

Create the EKS cluster (private endpoint only), configure access, create a managed node group, set up the OIDC provider, and install the AWS Load Balancer Controller.

## Variables

```bash
source ./eks_env.sh
```

## 4.1 Check Available Kubernetes Version

```bash
aws eks describe-cluster-versions \
  --region $REGION \
  --default-only \
  --query 'clusterVersions[].clusterVersion' \
  --output table
```

## 4.2 Create the Cluster

The cluster is created with a **private endpoint only** (no public access) and full control plane logging enabled:

```bash
aws eks create-cluster \
  --region $REGION \
  --name $CLUSTER_NAME \
  --role-arn $EKS_CLUSTER_ROLE_ARN \
  --resources-vpc-config "subnetIds=${SUBNET_APP_A},${SUBNET_APP_B},${SUBNET_INFRA},securityGroupIds=${CONTROL_PLANE_SG_ID},endpointPrivateAccess=true,endpointPublicAccess=false" \
  --kubernetes-network-config "ipFamily=ipv4" \
  --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}' \
  --output json
```

Wait for the cluster to become active:

```bash
aws eks wait cluster-active \
  --region $REGION \
  --name $CLUSTER_NAME
```

Verify:

```bash
aws eks describe-cluster \
  --region $REGION \
  --name $CLUSTER_NAME \
  --query 'cluster.{name:name,status:status,version:version}' \
  --output table
```

## 4.3 Enable API Access Mode

Switch from `CONFIG_MAP` to `API_AND_CONFIG_MAP` for EKS access entries:

```bash
aws eks update-cluster-config \
  --region $REGION \
  --name $CLUSTER_NAME \
  --access-config authenticationMode=API_AND_CONFIG_MAP
```

Wait for the update to complete:

```bash
aws eks wait cluster-active \
  --region $REGION \
  --name $CLUSTER_NAME
```

## 4.4 Grant Jenkins Cluster Admin Access

Create an access entry for the Jenkins role and associate the cluster admin policy:

```bash
aws eks create-access-entry \
  --region $REGION \
  --cluster-name $CLUSTER_NAME \
  --principal-arn $JENKINS_ROLE_ARN \
  --type STANDARD

aws eks associate-access-policy \
  --region $REGION \
  --cluster-name $CLUSTER_NAME \
  --principal-arn $JENKINS_ROLE_ARN \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
  --access-scope type=cluster
```

Verify:

```bash
aws eks list-access-entries \
  --region $REGION \
  --cluster-name $CLUSTER_NAME \
  --output table
```

## 4.5 Verify Connectivity from Jenkins

SSM into the Jenkins instance and test kubectl access:

```bash
aws ssm start-session \
  --region $REGION \
  --target $JENKINS_INSTANCE_ID
```

Inside the session:

```bash
aws eks update-kubeconfig \
  --region $REGION \
  --name $CLUSTER_NAME

kubectl get namespaces
kubectl get --raw=/readyz
```

> **Note:** Set `REGION` and `CLUSTER_NAME` in the SSM session, or substitute the literal values.

## 4.6 Create Managed Node Group

```bash
export NODEGROUP_NAME=${PREFIX}-ng-app

aws eks create-nodegroup \
  --region $REGION \
  --cluster-name $CLUSTER_NAME \
  --nodegroup-name $NODEGROUP_NAME \
  --subnets $SUBNET_APP_A $SUBNET_APP_B \
  --node-role $EKS_NODE_ROLE_ARN \
  --scaling-config minSize=2,maxSize=4,desiredSize=2 \
  --disk-size 50 \
  --capacity-type ON_DEMAND \
  --instance-types t3.large \
  --ami-type AL2023_x86_64_STANDARD \
  --labels workload=app,environment=prod \
  --tags Name=${NODEGROUP_NAME},Project=${PREFIX},Environment=prod
```

Wait for the node group to be active:

```bash
aws eks wait nodegroup-active \
  --region $REGION \
  --cluster-name $CLUSTER_NAME \
  --nodegroup-name $NODEGROUP_NAME
```

Verify from Jenkins (via SSM):

```bash
kubectl get nodes -o wide
kubectl get pods -A
```

## 4.7 OIDC Provider

The OIDC provider is required for IAM Roles for Service Accounts (IRSA), used by the AWS Load Balancer Controller and other add-ons.

### Get the OIDC Issuer

```bash
export OIDC_ISSUER=$(aws eks describe-cluster \
  --region $REGION \
  --name $CLUSTER_NAME \
  --query "cluster.identity.oidc.issuer" \
  --output text)

export OIDC_ID=$(echo "$OIDC_ISSUER" | cut -d '/' -f 5)
export OIDC_DOMAIN=$(echo "$OIDC_ISSUER" | awk -F/ '{print $3}')

echo "OIDC_ISSUER=$OIDC_ISSUER"
echo "OIDC_ID=$OIDC_ID"
echo "OIDC_DOMAIN=$OIDC_DOMAIN"
```

### Get the Root CA Thumbprint

```bash
rm -f /tmp/oidc-cert-*.pem

openssl s_client \
  -servername "$OIDC_DOMAIN" \
  -showcerts \
  -connect "${OIDC_DOMAIN}:443" </dev/null 2>/dev/null \
| awk '/BEGIN CERTIFICATE/{i++} {print > ("/tmp/oidc-cert-" i ".pem")}'

export ROOT_CERT=$(ls -1 /tmp/oidc-cert-*.pem | tail -n1)

export THUMBPRINT=$(openssl x509 -in "$ROOT_CERT" -fingerprint -sha1 -noout \
  | cut -d= -f2 | tr -d ':' | tr 'A-F' 'a-f')

echo "THUMBPRINT=$THUMBPRINT"
```

### Create the OIDC Provider

```bash
aws iam create-open-id-connect-provider \
  --url "$OIDC_ISSUER" \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list "$THUMBPRINT"
```

Verify:

```bash
aws iam list-open-id-connect-providers | grep "$OIDC_ID"
```

## 4.8 AWS Load Balancer Controller

### Create IAM Policy

Download the latest policy document and create it:

```bash
curl -Lo iam-policy-aws-load-balancer-controller.json \
  https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.14.1/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name ${PREFIX}-AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam-policy-aws-load-balancer-controller.json
```

```bash
export LBC_POLICY_ARN=$(aws iam list-policies \
  --scope Local \
  --query "Policies[?PolicyName=='${PREFIX}-AWSLoadBalancerControllerIAMPolicy'].Arn | [0]" \
  --output text)

echo "LBC_POLICY_ARN=$LBC_POLICY_ARN"
```

### Create IAM Role with IRSA Trust

```bash
export OIDC_PROVIDER_HOSTPATH=$(echo "$OIDC_ISSUER" | sed 's#^https://##')
export LBC_ROLE_NAME=${PREFIX}-AmazonEKSLoadBalancerControllerRole

cat > load-balancer-role-trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::${ACCOUNT_ID}:oidc-provider/${OIDC_PROVIDER_HOSTPATH}"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "${OIDC_PROVIDER_HOSTPATH}:aud": "sts.amazonaws.com",
          "${OIDC_PROVIDER_HOSTPATH}:sub": "system:serviceaccount:kube-system:aws-load-balancer-controller"
        }
      }
    }
  ]
}
EOF
```

```bash
aws iam create-role \
  --role-name $LBC_ROLE_NAME \
  --assume-role-policy-document file://load-balancer-role-trust-policy.json

aws iam attach-role-policy \
  --role-name $LBC_ROLE_NAME \
  --policy-arn $LBC_POLICY_ARN
```

### Deploy the Service Account and Controller

SSM into the Jenkins instance and run:

```bash
cat > /tmp/aws-load-balancer-controller-sa.yaml <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: aws-load-balancer-controller
  namespace: kube-system
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::${ACCOUNT_ID}:role/${LBC_ROLE_NAME}
EOF

kubectl apply -f /tmp/aws-load-balancer-controller-sa.yaml
```

Install the controller via Helm:

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=$CLUSTER_NAME \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=$REGION \
  --set vpcId=$VPC_ID
```

> **Note:** If running inside an SSM session, set `CLUSTER_NAME`, `REGION`, and `VPC_ID` first, or substitute the literal values directly.

Verify:

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
```

## Verify End-to-End

Deploy a smoke test to confirm the ALB Ingress Controller works:

```bash
cat > /tmp/smoke-test.yaml <<'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: smoke
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo
  namespace: smoke
spec:
  replicas: 1
  selector:
    matchLabels:
      app: echo
  template:
    metadata:
      labels:
        app: echo
    spec:
      containers:
        - name: echo
          image: nginx:alpine
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: echo
  namespace: smoke
spec:
  selector:
    app: echo
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: echo
  namespace: smoke
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/healthcheck-path: /
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: echo
                port:
                  number: 80
EOF

kubectl apply -f /tmp/smoke-test.yaml
```

Wait for the ALB DNS and test:

```bash
kubectl get ingress -n smoke
curl -I http://<ALB-DNS-from-output>
```

Clean up after verifying:

```bash
kubectl delete namespace smoke
```

## Save Variables

```bash
cat >> eks_env.sh <<EOF

# Step 4: EKS Cluster
export NODEGROUP_NAME=$NODEGROUP_NAME
export OIDC_ISSUER=$OIDC_ISSUER
export OIDC_ID=$OIDC_ID
export OIDC_DOMAIN=$OIDC_DOMAIN
export OIDC_PROVIDER_HOSTPATH=$OIDC_PROVIDER_HOSTPATH
export LBC_POLICY_ARN=$LBC_POLICY_ARN
export LBC_ROLE_NAME=$LBC_ROLE_NAME
EOF
echo "Variables saved to eks_env.sh"
```
