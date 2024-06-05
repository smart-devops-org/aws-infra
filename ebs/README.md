# EBS를 사용한 EKS 스토리지

## 1. IAM 정책 생성

- 서비스 -> IAM -> 정책 만들기 -> Amazon_EBS_CSI_Driver
  - 이름: Amazon_EBS_CSI_Driver
  - 설명: EC2 인스턴스가 Elastic Block Store에 액세스하기 위한 정책

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:AttachVolume",
        "ec2:CreateSnapshot",
        "ec2:CreateTags",
        "ec2:CreateVolume",
        "ec2:DeleteSnapshot",
        "ec2:DeleteTags",
        "ec2:DeleteVolume",
        "ec2:DescribeInstances",
        "ec2:DescribeSnapshots",
        "ec2:DescribeTags",
        "ec2:DescribeVolumes",
        "ec2:DetachVolume"
      ],
      "Resource": "*"
    }
  ]
}
```

## 2. 이 정책을 사용하여 IAM 역할 작업자 노드를 가져오고 해당 역할에 연결

```
# Get Worker node IAM Role ARN
kubectl -n kube-system describe configmap aws-auth
```

- 서비스 -> IAM -> 역할
- eksctl-eksdemo1-nodegroup 역할 검색
- 권한 -> 정책 연결
- Amazon_EBS_CSI_Driver 추가

## 3. Amazon EBS CSI 드라이버 배포

- Amazon EBS CSI 드라이버 배포

```
# Deploy EBS CSI Driver
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=master"

# Verify ebs-csi pods running
kubectl get pods -n kube-system
```
