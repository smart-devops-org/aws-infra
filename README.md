# 1. eks 클러스터용 vpc 및 서브넷 생성(인프라 구성)

- 템플릿 생성
- [Amazon EKS 클러스터에 대해 VPC 생성
  ](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/creating-a-vpc.html)

```
aws cloudformation create-stack \
 --region ap-northeast-2 \
 --stack-name smart-cluster-vpc \
 --template-url https://s3.us-west-2.amazonaws.com/amazon-eks/cloudformation/2020-10-29/amazon-eks-vpc-private-subnets.yaml

```

# 2. 클러스터 iam 역할 생성 및 정책 연결

- [Amazon EKS cluster IAM role
  ](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/creating-a-vpc.html)

- 역할 생성 생성

```
aws iam create-role \
  --role-name smart-ClusterRole \
  --assume-role-policy-document file://policy/cluster-trust-policy.json
```

- 필요한 eks 관리형 iam 정책 역할에 연결

```
# EKS 워커 노드가 EKS 클러스터와 통신하는 데 필요한 권한을 부여
aws iam attach-role-policy \
  --role-name smart-ClusterRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy

#  EKS 클러스터의 CNI (Container Network Interface) 플러그인이 필요로 하는 권한을 부여합니다. CNI 플러그인은 파드 간 네트워킹을 가능
aws iam attach-role-policy \
  --role-name smart-ClusterRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy

# Amazon ECR (Elastic Container Registry)의 이미지를 읽을 수 있는 권한을 부여합니다. 이는 컨테이너 이미지를 pull하는 데 필요
aws iam attach-role-policy \
  --role-name smart-ClusterRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly

aws iam attach-role-policy \
  --role-name smart-ClusterRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
```

# 3. eks 클러스터 생성

```
aws eks create-cluster \
  --region ap-northeast-2 \
  --name smart-cluster \
  --role-arn arn:aws:iam::221370546661:role/smart-ClusterRole \
  --resources-vpc-config subnetIds=subnet-04c1fa565da32dbaf,subnet-054ad0183286e717b,subnet-0b3861508471e6c1e,subnet-094b2fc905022e004,securityGroupIds=sg-032a2e66db2413eb0
```

## 노드 그룹 생성

- 공부용으로 spot 인스턴스로 사용하였음

```
aws eks create-nodegroup \
  --cluster-name smart-cluster \
  --nodegroup-name smart-cluster-nodegroup \
  --node-role arn:aws:iam::221370546661:role/smart-ClusterRole \
  --subnets subnet-04c1fa565da32dbaf subnet-054ad0183286e717b subnet-0b3861508471e6c1e subnet-094b2fc905022e004 \
  --region ap-northeast-2 \
  --capacity-type SPOT
```

```
# max 개수 수정
eksctl scale nodegroup --cluster=smart-cluster --name=smart-cluster-nodegroup --nodes-max=4
# 노드 그룹 노드 추가
eksctl scale nodegroup --cluster=smart-cluster --name=smart-cluster-nodegroup --nodes=3

# 노드 그룹 노드 삭제
eksctl delete nodegroup --cluster=smart-cluster --name=smart-cluster-nodegroup --region=ap-northeast-2

# 클러스터 삭제
eksctl delete cluster smart-cluster
```

### kubeconfig 파일 업데이트

```
aws eks --region ap-northeast-2 update-kubeconfig --name smart-cluster
```

- IAM OIDC 자격증명 공급자 생성
  - 쿠버네티스 서비스 계정에 IAM 역할을 사용하려면 OIDC 공급자가 필요

```
eksctl utils associate-iam-oidc-provider --cluster smart-cluster --approve
```

# 4. external-dns

## 네임 스페이스 생성

```
k create ns external-dns
```

## external-dns 가 route53을 제어할 수 있도록 정책 생성

```
aws iam create-policy --policy-name AllowExternalDNSUpdates --policy-document file://aws/iam/external-dns-route53-policy.json
```

## iam service account 생성

```
eksctl create iamserviceaccount \
    --name external-dns \
    --namespace external-dns \
    --cluster smart-cluster \
    --attach-policy-arn arn:aws:iam::221370546661:policy/AllowExternalDNSUpdates \
    --approve \
    --override-existing-serviceaccounts
```

## 서비스 account 확인

```
k get sa -n external-dns
```

<img width="1060" alt="image" src="https://github.com/smart-devops-org/devops-system/assets/46982752/a72b9307-22dc-488c-8490-2e8a471924e5">

## 메니페스트 배포

```
k apply -f aws-infra/external-dns/values.yaml
```

<img width="1057" alt="image" src="https://github.com/smart-devops-org/devops-system/assets/46982752/88f7676a-7a81-41d7-9436-c9c53b23946d">
<img width="1065" alt="image" src="https://github.com/smart-devops-org/devops-system/assets/46982752/acbd45af-515f-430b-b99a-c691f4417f31">
<img width="1061" alt="image" src="https://github.com/smart-devops-org/devops-system/assets/46982752/5d402df4-4e09-4be8-9d68-a899484f63b8">
<img width="1070" alt="image" src="https://github.com/smart-devops-org/devops-system/assets/46982752/8edcb871-2dd9-46a5-8922-381427a3d4ed">

## 적용 테스트

```
k apply -f aws-infra/external-dns/test.yaml
```

```
    annotations:
        external-dns.alpha.kubernetes.io/hostname: 실제 적용 도메인
spec:
  type: LoadBalancer
```

- kube object 확인
  ![image](https://github.com/smart-devops-org/devops-system/assets/46982752/8b2c88b4-f94b-408c-81c1-dfc2c3edf1ec)

<img width="1184" alt="image" src="https://github.com/smart-devops-org/devops-system/assets/46982752/b46a8146-2654-4f17-a09d-e810c2eb35b9">

![image](https://github.com/smart-devops-org/devops-system/assets/46982752/22d09718-899e-4d54-b3a4-80585973c25f)

- route53 확인

<img width="761" alt="image" src="https://github.com/smart-devops-org/devops-system/assets/46982752/866d9536-ec36-4914-8314-21a80736338a">

- 로드밸런서 생성 확인

<img width="1071" alt="image" src="https://github.com/smart-devops-org/devops-system/assets/46982752/1d36b4a8-3976-415a-b688-5967a07c0a44">

- 접속 확인

![image](https://github.com/smart-devops-org/devops-system/assets/46982752/4aa75348-5366-4cfd-8527-b9e88ce6391e)

# 5. AWS load Balancer controller

![image](https://github.com/smart-devops-org/devops-system/assets/46982752/c478db23-0be1-4fa0-a471-797f4ab5ca7a)

![image](https://github.com/smart-devops-org/devops-system/assets/46982752/36fd218d-0aa9-4449-a8fa-e35fe51d47fa)

## 로드밸런서를 컨트롤 할 수 있도록 정책 생성

```
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.7.1/docs/install/iam_policy.json

aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://aws-infra/load-balancer-controller/load-balancer-controller.json
```

## iam service account 생성

```
eksctl create iamserviceaccount \
  --cluster=smart-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::221370546661:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

### 역할 확인

```
# IRSA 정보 확인
eksctl get iamserviceaccount --cluster

# Kubernetes 서비스 어카운트 확인
kubectl get serviceaccounts -n kube-system aws-load-balancer-controller -o yaml

```

## 배포

```
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=smart-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

### 배포 확인

```
# Kubernetes CRD 확인
kubectl get crd

# ingressclassparams.elbv2.k8s.aws
# targetgroupbindings.elbv2.k8s.aws

# AWS Load Balancer Controller Role 확인
kubectl describe clusterroles.rbac.authorization.k8s.io aws-load-balancer-controller-role

# AWS Load Balancer Controller 확인
kubectl get deployment -n kube-system aws-load-balancer-controller

kubectl describe deploy -n kube-system aws-load-balancer-controller
```

### 테스트

```
k apply -f aws-infra/load-balancer-controller
```

<img width="1007" alt="image" src="https://github.com/smart-devops-org/devops-system/assets/46982752/6ee3b40f-3b49-46a1-8dad-97f4675701b2">

![image](https://github.com/smart-devops-org/devops-system/assets/46982752/87a6f0a0-58e5-4cdf-8662-57a867e64446)

<img width="1127" alt="image" src="https://github.com/smart-devops-org/devops-system/assets/46982752/141eff8f-1d57-43cd-b02f-0707332b3a76">

# 6. EBS

# EBS를 사용한 EKS 스토리지

## 정책 생성

- 서비스 -> IAM -> 정책 만들기 -> Amazon_EBS_CSI_Driver
  - 이름: Amazon_EBS_CSI_Driver
  - 설명: EC2 인스턴스가 Elastic Block Store에 액세스하기 위한 정책

```
aws iam create-policy \
    --policy-name Amazon_EBS_CSI_Driver \
    --policy-document file://aws-infra/policy/Amazon_EBS_CSI_Driver.json
```

## 2. 이 정책을 사용하여 IAM 역할 작업자 노드를 가져오고 해당 역할에 연결

```
aws iam attach-role-policy \
  --role-name smart-ClusterRole \
  --policy-arn arn:aws:iam::221370546661:policy/Amazon_EBS_CSI_Driver

```

## 3. Amazon EBS CSI 드라이버 배포

- Amazon EBS CSI 드라이버 배포

```
# Deploy EBS CSI Driver
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=master"

# Verify ebs-csi pods running
kubectl get pods -n kube-system
```

### 테스트

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-mysql-pv-claim
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-sc
  resources:
    requests:
      storage: 4Gi

```

# 7. AutoScale

## Karpenter

- ASG와는 독립적으로 노드를 프로비저닝하고 관리
- ASG에 의존하지 않고 노드 관리를 보다 빠르고 세밀하게 제어 가능
  ![image](https://github.com/smart-devops-org/devops-system/assets/46982752/d0d9b34b-7cc2-4d66-a14e-569137f6c207)

## Karpenter 사용할 iam 역할 생성

```
aws iam create-role \
  --role-name KarpenterNodeRole-eks \
  --assume-role-policy-document file://aws-infra/policy/cluster-trust-policy.json
```

## 역할에 정책 연결

```
# EKS 워커 노드가 EKS 클러스터와 통신하는 데 필요한 권한을 부여
aws iam attach-role-policy \
  --role-name KarpenterNodeRole-eks \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy

#  EKS 클러스터의 CNI (Container Network Interface) 플러그인이 필요로 하는 권한을 부여합니다. CNI 플러그인은 파드 간 네트워킹을 가능
aws iam attach-role-policy \
  --role-name KarpenterNodeRole-eks \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy

# Amazon ECR (Elastic Container Registry)의 이미지를 읽을 수 있는 권한을 부여합니다. 이는 컨테이너 이미지를 pull하는 데 필요
aws iam attach-role-policy \
  --role-name KarpenterNodeRole-eks \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly

aws iam attach-role-policy \
  --role-name KarpenterNodeRole-eks \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

```

## 역할에 인스턴스 프로파일을 생성 및 추가

- 인스턴스 프로파일은 IAM 역할과 함께 사용되며, 인스턴스 프로파일은 인스턴스에 연결될 때 IAM 역할을 지정합니다. 그런 다음, 해당 역할에 연결된 정책에 따라 인스턴스는 AWS API를 호출할 수 있습니다.

```
aws iam create-instance-profile \
    --instance-profile-name "KarpenterNodeRoleProfile-eks"

aws iam add-role-to-instance-profile \
    --instance-profile-name "KarpenterNodeRoleProfile-eks" \
    --role-name "KarpenterNodeRole-eks"
```

## 컨트롤러용 IAM 역할 생성

### Karpenter 컨트롤러에서 사용할 IAM Role을 생성

```
aws iam create-role --role-name KarpenterControllerRole \
    --assume-role-policy-document file://aws-infra/policy/karpenter-controller-trust-policy.json
```

### Karpenter 컨트롤러용 IAM Policy를 생성 및 연결

```
aws iam put-role-policy --role-name KarpenterControllerRole \
    --policy-name KarpenterControllerPolicy \
    --policy-document file://aws-infra/policy/karpenter-controller-policy.json
```

### 서브넷 리소스에 태그 추가

- karpenter가 사용할 서브넷을 알수 있도록 노드 그룹 서브넷에 태그를 추가해야함
  ![image](https://github.com/smart-devops-org/devops-system/assets/46982752/b5d4e3d6-d21a-4a2b-bee2-8ec1cc1beb50)

- 추가되는 태그 정보

```
Key : karpenter.sh/discovery
Value : 현재 사용중인 자신의 EKS 클러스터 이름을 찾아 자동 입력됨
```

```
for NODEGROUP in $(aws eks list-nodegroups --cluster-name smart-cluster \
    --query 'nodegroups' --output text); do aws ec2 create-tags \
    --tags "Key=karpenter.sh/discovery,Value=smart-cluster" \
    --resources $(aws eks describe-nodegroup --cluster-name smart-cluster \
    --nodegroup-name $NODEGROUP --query 'nodegroup.subnets' --output text )
done
```

- 보안 그룹에도 태그 추가

```
NODEGROUP=$(aws eks list-nodegroups --cluster-name ${CLUSTER_NAME} \
    --query 'nodegroups[0]' --output text)LAUNCH_TEMPLATE=$(aws eks describe-nodegroup --cluster-name ${CLUSTER_NAME} \
    --nodegroup-name ${NODEGROUP} --query 'nodegroup.launchTemplate.{id:id,version:version}' \
    --output text | tr -s "\t" ",")

# If your EKS setup is configured to use only Cluster security group, then please execute -
# 클러스터에서 하나의 보안그룹만 사용한다면 아래 명령어 사용
SECURITY_GROUPS=$(aws eks describe-cluster \
    --name smart-cluster --query "cluster.resourcesVpcConfig.clusterSecurityGroupId" --output text)

# If your setup uses the security groups in the Launch template of a managed node group, then :
# 노드그룹의 시작 템플릿에 있는 보안그룹을 사용한다면 아래 명령어 사용
SECURITY_GROUPS=$(aws ec2 describe-launch-template-versions \
    --launch-template-id ${LAUNCH_TEMPLATE%,*} --versions ${LAUNCH_TEMPLATE#*,} \
    --query 'LaunchTemplateVersions[0].LaunchTemplateData.[NetworkInterfaces[0].Groups||SecurityGroupIds]' \
    --output text)

aws ec2 create-tags \
    --tags "Key=karpenter.sh/discovery,Value=smart-cluster" \
    --resources ${SECURITY_GROUPS}
```

## aws-auth ConfigMap 업데이트

- 방금 생성한 노드 IAM 역할을 사용하는 노드가 클러스터에 조인하도록 허용
- aws-auth이를 위해서는 클러스터에서 ConfigMap을 수정

<img width="1028" alt="image" src="https://github.com/smart-devops-org/devops-system/assets/46982752/f4e7d7b6-4e2b-415a-a18e-53e1d28ae122">

<img width="724" alt="image" src="https://github.com/smart-devops-org/devops-system/assets/46982752/912843d9-c37b-4a1c-87b8-2c41141de688">

## Karpenter 배포

```
# Karpenter Version 설정
export KARPENTER_VERSION=v0.29.0

# helm repo 추가
helm repo add karpenter https://charts.karpenter.sh/
helm repo update

# helm 을 이용하여 Karpenter를 설치 합니다.
# serviceaccount 는 위에서 기억해 놓으라고 했던 karpenter controller role name 입니다.
helm template karpenter oci://public.ecr.aws/karpenter/karpenter --version ${KARPENTER_VERSION} --namespace karpenter \
    --set aws.defaultInstanceProfile=KarpenterNodeRoleProfile-eks \
    --set clusterEndpoint="https://13B10C0949294FC6CDC7D4490B3B8582.yl4.ap-northeast-2.eks.amazonaws.com" \
    --set clusterName=smart-cluster \
    --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"="arn:aws:iam::221370546661:role/KarpenterControllerRole" \
    --version ${KARPENTER_VERSION} > karpenter.yaml

# 위의 명령어를 수행하면 karpenter.yaml 파일이 생성됩니다.
# 해당 파일을 수정하여 Node 선호도를 수정해야 합니다.
# Karpenter Controller 가 추후에 Karpenter 에 의해 생성된 WorkerNode 에 생성되지 않도록 해야 합니다.
# nodeAffinity 부분을 찾아서 다음과 같이 수정 합니다.
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: karpenter.sh/provisioner-name
                operator: DoesNotExist
            - matchExpressions:
              - key: eks.amazonaws.com/nodegroup
                operator: In
                values:
                - ${Karpenter에 생성되지 않은 현재 Worker NodeGroup Name}

# 네임스페이스와 CRD를 생성 후 배포 합니다.
k create ns karpenter
kubectl create -f https://raw.githubusercontent.com/aws/karpenter/v0.29.0/pkg/apis/crds/karpenter.sh_provisioners.yaml
kubectl create -f https://raw.githubusercontent.com/aws/karpenter/v0.29.0/pkg/apis/crds/karpenter.k8s.aws_awsnodetemplates.yaml
kubectl create -f https://raw.githubusercontent.com/aws/karpenter/v0.29.0/pkg/apis/crds/karpenter.sh_machines.yaml
k apply -f aws-infra/karpenter/karpenter.yaml
```

- 로그 확인

```
kubectl logs -f -n karpenter -l app.kubernetes.io/name=karpenter -c controller

```

<img width="998" alt="image" src="https://github.com/smart-devops-org/devops-system/assets/46982752/8bd9a986-01aa-4228-99d8-2b1c6ab68248">

## Scale out 테스트

## karpenter 가 생성할 수 있는 노드와 노드에서 실행될 수 있는 파드에 대한 제약 조건 설정

```
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: default
spec:
  requirements:
    - key: node.kubernetes.io/instance-type
      operator: In
      values: ["t3.medium"]
    - key: karpenter.sh/capacity-type
      operator: In
      values: ["spot"]
  limits:
    resources:
      cpu: 1000
  providerRef:
    name: default
  ttlSecondsAfterEmpty: 30
---
apiVersion: karpenter.k8s.aws/v1alpha1
kind: AWSNodeTemplate
metadata:
  name: default
spec:
  subnetSelector:
    karpenter.sh/discovery: smart-cluster
  securityGroupSelector:
    karpenter.sh/discovery: smart-cluster
  instanceProfile: KarpenterNodeRoleProfile-eks
  blockDeviceMappings:
    - deviceName: /dev/xvda
      ebs:
        volumeSize: 20Gi
        volumeType: gp2
        encrypted: true

```

## 테스트

### scale out

- k apply -f aws-infra/karpenter/deployment.yaml

<img width="1005" alt="image" src="https://github.com/smart-devops-org/devops-system/assets/46982752/d2b159e2-e2d4-4207-9885-c1ef8f628fd3">

<img width="1015" alt="image" src="https://github.com/smart-devops-org/devops-system/assets/46982752/10b28189-5600-40f7-87b1-6c288f8891a3">

<img width="996" alt="image" src="https://github.com/smart-devops-org/devops-system/assets/46982752/079d69c1-e36d-4895-9ce9-36a7f125aee9">

### scale in

- k delete -f aws-infra/karpenter/deployment.yaml

<img width="1017" alt="image" src="https://github.com/smart-devops-org/devops-system/assets/46982752/f7037478-a5d4-471d-bb8d-865767346304">

<img width="1001" alt="image" src="https://github.com/smart-devops-org/devops-system/assets/46982752/2fb61cf9-207c-46e4-ac6d-d5007d74921e">
