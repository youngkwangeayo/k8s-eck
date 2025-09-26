

## 프로비저닝  

### AWS Cluster 베포
```bash
# 클러스터 베포
eksctl create cluster -f aws/eks-cluster.yaml
# 클러스터 검증.  # eksctl create cluster -f aws/eks-cluster.yaml --dry-run
# 만약 실패시 # eksctl delete cluster --name elasticsearch-cluster --region ap-northeast-2 --wait
```
### 클러스터 노드 오토스케일링 베포
```bash
kubectl apply -f aws/cluster-autoscaler.yaml
```
### 사용할 스토리지 명세
```bash
kubectl apply -f aws/storage-class.yaml
```

### 클러스태내부에 ECK 베포   Resource정의/오퍼레이터 설치
```bash
kubectl apply -f https://download.elastic.co/downloads/eck/2.10.0/crds.yaml
kubectl apply -f https://download.elastic.co/downloads/eck/2.10.0/operator.yaml 
```

### 클러스터 내부 infra Object 베포
```bash
kubectl apply -f manifests/namespace.yaml 
```

### LoadBalancer 베포
```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm upgrade --install aws-load-balancer-controller eks/aws-load-balancer-controller \
    -n kube-system \
    --set clusterName=elasticsearch-cluster \
    --set serviceAccount.create=false \
    --set serviceAccount.name=aws-load-balancer-controller

# 또는
./aws/aws-load-balancer-controller.sh
```

### Object 및 ECK? 클러스터 베포
```bash
kubectl apply -f manifests/
```

### elastic & kibana 비번확인
```bash
# ID elastic
# PASS. gAGC8Y44IT10sg7J740LEqT1
kubectl -n elastic-system get secret elasticsearch-es-elastic-user \
  -o go-template='{{index .data "elastic" | base64decode}}{{"\n"}}

```



------------------------------------------------

## 기타 내용 


## 필수패키지 

### k8s kubectl 설치 (쿠버네티스 관리용)
```bash
```
### AWS awscli 설치 (aws 베포용)
```bash
```
### helm helm 설치 (쿠버네티스 패키지 용)
```bash
```
### AWS eksctl 설치 (EKS 베포용)
```bash
brew tap aws/tap
brew install aws/tap/eksctl
```


### elastic & kibana 비번확인
```bash
# ID elastic
# PASS. gAGC8Y44IT10sg7J740LEqT1
kubectl -n elastic-system get secret elasticsearch-es-elastic-user \
  -o go-template='{{index .data "elastic" | base64decode}}{{"\n"}}

```

