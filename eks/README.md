# 🚀 AWS EKS デモ環境構築ガイド

<div align="center">
  
  [![AWS EKS](https://img.shields.io/badge/AWS_EKS-Latest-FF9900?style=for-the-badge&logo=amazon-aws&logoColor=white)](https://aws.amazon.com/eks/)
  [![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.30-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)](https://kubernetes.io/)
  [![eksctl](https://img.shields.io/badge/eksctl-Latest-48C9B0?style=for-the-badge&logo=kubernetes&logoColor=white)](https://eksctl.io/)
  
  **プロダクション環境でKubernetesを体験しよう！** 🎯
  
</div>

---

## 📁 ディレクトリ構造

```
eks-demo/
├── 📘 README.md              # このファイル（全手順を含む）
├── 📂 01-preparation/        # 事前準備用（デモ前日に実行）
│   ├── infrastructure/       # VPC構築用CloudFormation
│   ├── cluster-config-with-vpc.yaml
│   └── create-cluster-with-vpc.sh
├── 📂 02-demo-day/          # デモ当日用のYAMLファイル
│   ├── configmap.yaml       # 環境変数設定
│   ├── deployment.yaml      # アプリケーションデプロイ
│   ├── service.yaml         # ClusterIPサービス
│   └── ingress.yaml         # ALB Ingress設定
└── 📂 99-reference/         # 参考資料（通常は使用しない）
```

---

## 📋 前提条件

### 🛠 必要なツール

<details>
<summary>📍 <b>Mac環境の方はこちら</b></summary>

#### 1️⃣ AWS CLIのインストール
```bash
# 🍺 Homebrewでインストール
brew install awscli

# ✅ バージョン確認
aws --version
```

#### 2️⃣ eksctlのインストール
```bash
# 🍺 eksctlをインストール
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl

# ✅ バージョン確認
eksctl version
```

#### 3️⃣ kubectlのインストール
```bash
# 🍺 kubectlをインストール  
brew install kubectl

# ✅ バージョン確認
kubectl version --client
```

</details>

### 🔑 AWS設定

```bash
# AWS認証情報の設定
aws configure

# 以下を入力:
# AWS Access Key ID: [アクセスキー]
# AWS Secret Access Key: [シークレットキー]
# Default region name: ap-northeast-1
# Default output format: json
```

#### 動作確認 ✨

```bash
# 📊 アカウント情報の確認
aws sts get-caller-identity

# 🌏 リージョンの確認
aws configure get region
```

---

## 🏗️ Step 1: VPCインフラ構築（事前準備）

### 📅 実施タイミング
<table>
<tr>
<td width="60%">

```bash
# 🚀 事前準備ディレクトリへ移動
cd 01-preparation/infrastructure

# 📦 CloudFormationでVPC作成
aws cloudformation create-stack \
  --stack-name k8s-demo-vpc-stack \
  --template-body file://vpc-stack.yaml \
  --region ap-northeast-1
```

</td>
</tr>
</table>

#### スタック作成の確認 ⏳

```bash
# 📊 作成状況の確認（CREATE_COMPLETEになるまで待機）
aws cloudformation describe-stacks \
  --stack-name k8s-demo-vpc-stack \
  --region ap-northeast-1 \
  --query 'Stacks[0].StackStatus' \
  --output text

# 📝 VPC情報を設定ファイルに保存（次のステップで使用）
cat > 01-preparation/infrastructure/infrastructure-config.sh << EOF
#!/bin/bash
export VPC_ID=$(aws cloudformation describe-stacks \
  --stack-name k8s-demo-vpc-stack \
  --query 'Stacks[0].Outputs[?OutputKey==`VPCId`].OutputValue' \
  --output text)
export PRIVATE_SUBNET_1=$(aws cloudformation describe-stacks \
  --stack-name k8s-demo-vpc-stack \
  --query 'Stacks[0].Outputs[?OutputKey==`PrivateSubnet1Id`].OutputValue' \
  --output text)
export PRIVATE_SUBNET_2=$(aws cloudformation describe-stacks \
  --stack-name k8s-demo-vpc-stack \
  --query 'Stacks[0].Outputs[?OutputKey==`PrivateSubnet2Id`].OutputValue' \
  --output text)
export REGION="ap-northeast-1"
EOF

# 実行権限を付与
chmod +x 01-preparation/infrastructure/infrastructure-config.sh
```

<details>
<summary>🔍 <b>作成されるリソース詳細</b></summary>

| リソース | 説明 | 
|:--------:|:-----|
| 🌐 **VPC** | 10.192.0.0/16 のプライベートネットワーク |
| 🔧 **Subnets** | Public×2、Private×2 (マルチAZ) |
| 🚪 **NAT Gateway** | Private SubnetからのInternet接続用 |
| 🏷️ **EIP** | NAT Gateway用の固定IP（必須） |
| 🛣️ **Route Tables** | 適切なルーティング設定 |

</details>

---

## ☸️ Step 2: EKSクラスタ作成とALB Controller設定
#### 事前準備済みのVPCを使用した作成

```bash
# 📝 Step1で作成したVPC情報を取得
source 01-preparation/infrastructure/infrastructure-config.sh

# 🔍 VPC IDとSubnet IDを確認
echo "VPC ID: $VPC_ID"
echo "Private Subnet 1: $PRIVATE_SUBNET_1"
echo "Private Subnet 2: $PRIVATE_SUBNET_2"
```

<table>
<tr>
<td>

```yaml
# 01-preparation/cluster-config-with-vpc.yaml を参照
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: k8s-demo-cluster
  region: ap-northeast-1
  version: "1.30"

iam:
  withOIDC: true

vpc:
  id: "vpc-xxxxx"  # infrastructure-config.shから取得
  subnets:
    private:
      ap-northeast-1a: { id: subnet-xxxxx }
      ap-northeast-1c: { id: subnet-xxxxx }

managedNodeGroups:
  - name: demo-nodes
    instanceType: t3.small
    desiredCapacity: 2
    minSize: 1
    maxSize: 3
    volumeSize: 20
    volumeType: gp3
```

</td>
</tr>
</table>

```bash
# 🚀 クラスタ作成（約20分）
eksctl create cluster -f 01-preparation/cluster-config-with-vpc.yaml

# ✅ 作成確認
eksctl get cluster --region ap-northeast-1
```

#### 接続確認 🔌

```bash
# 📝 kubeconfigの更新
aws eks update-kubeconfig \
  --region ap-northeast-1 \
  --name k8s-demo-cluster

# 🖥️ ノード確認
kubectl get nodes
```

#### AWS Load Balancer Controllerのインストール 🎮

> 💡 **AWS Load Balancer Controller とは**  
> Kubernetes Ingress リソースを AWS Application Load Balancer (ALB) として自動作成・管理するコントローラーです。

##### 1️⃣ IAMポリシーの作成

```bash
# IAMポリシーのダウンロード
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.7.0/docs/install/iam_policy.json

# ポリシーの作成
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json

# 作成確認（ポリシーARNが表示される）
aws iam list-policies --query 'Policies[?PolicyName==`AWSLoadBalancerControllerIAMPolicy`].Arn' --output text
```

##### 2️⃣ IAMサービスアカウントの作成

```bash
# アカウントIDを取得
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
echo "Account ID: $ACCOUNT_ID"

# サービスアカウントの作成（OIDCプロバイダーとの連携）
eksctl create iamserviceaccount \
  --cluster=k8s-demo-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::${ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve \
  --region=ap-northeast-1

# 作成確認
kubectl get serviceaccount aws-load-balancer-controller -n kube-system
```

##### 3️⃣ Helmのインストール（未インストールの場合）

```bash
# Homebrewでインストール
brew install helm

# バージョン確認
helm version --short
```

##### 4️⃣ AWS Load Balancer Controllerのデプロイ

```bash
# Helmリポジトリの追加
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks

# VPC IDを取得（Step1で作成したVPC）
source 01-preparation/infrastructure/infrastructure-config.sh
echo "VPC ID: $VPC_ID"

# コントローラーのインストール
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=k8s-demo-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=ap-northeast-1 \
  --set vpcId=${VPC_ID}
```

##### 5️⃣ インストール確認

```bash
# Deploymentの確認（READYが2/2になることを確認）
kubectl get deployment -n kube-system aws-load-balancer-controller

# Podの確認（Runningになることを確認）
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller

# ログ確認（エラーがないことを確認）
kubectl logs -n kube-system deployment/aws-load-balancer-controller --tail=20
```

<details>
<summary>⚠️ <b>トラブルシューティング</b></summary>

**Podが起動しない場合**
```bash
# イベント確認
kubectl get events -n kube-system --sort-by='.lastTimestamp' | grep load-balancer

# サービスアカウントの確認
kubectl describe serviceaccount aws-load-balancer-controller -n kube-system
```

**IAMロールの問題**
```bash
# OIDCプロバイダーの確認
aws eks describe-cluster --name k8s-demo-cluster --query "cluster.identity.oidc.issuer" --output text
```

</details>

> 📝 **重要な注意事項**  
> - クラスタ作成時に `--with-oidc` オプションが必要（OIDC プロバイダー）
> - VPCのサブネットには適切なタグが必要（eksctlで作成した場合は自動設定済み）
>   - Public Subnet: `kubernetes.io/role/elb=1`
>   - Private Subnet: `kubernetes.io/role/internal-elb=1`

<details>
<summary>🔍 <b>期待される出力</b></summary>

```
NAME                                              STATUS   ROLES    AGE   VERSION
ip-10-192-20-xxx.ap-northeast-1.compute.internal Ready    <none>   5m    v1.30.x
ip-10-192-21-xxx.ap-northeast-1.compute.internal Ready    <none>   5m    v1.30.x
```

</details>

---

## 📦 Step 3: アプリケーションのデプロイ

### 🔹 ConfigMap作成（環境変数）

```bash
# 🎯 ConfigMapを作成
kubectl apply -f 02-demo-day/configmap.yaml
```

<details>
<summary>📝 <b>configmap.yaml の内容</b></summary>

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: demo-config
data:
  MESSAGE: "Hello from AWS EKS! ConfigMap is working!"
  ENVIRONMENT: "production"
  VERSION: "1.0.0"
```

</details>

#### 作成確認

```bash
# 📊 ConfigMapの確認
kubectl get configmap demo-config
kubectl describe configmap demo-config
```

### 🔹 Deployment作成（3レプリカ）

```bash
# 🎯 Deploymentを作成
kubectl apply -f 02-demo-day/deployment.yaml
```

<details>
<summary>📝 <b>deployment.yaml の内容</b></summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: eks-demo-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: eks-demo
  template:
    metadata:
      labels:
        app: eks-demo
    spec:
      containers:
      - name: echo-server
        image: hashicorp/http-echo:0.2.3
        args:
          - "-text=Pod $(HOSTNAME) | Message: $(MESSAGE) | Environment: $(ENVIRONMENT) | Version: $(VERSION)"
          - "-listen=:8080"
        envFrom:
        - configMapRef:
            name: demo-config
        resources:
          limits:
            memory: "128Mi"
            cpu: "100m"
```

</details>

#### Pod起動の確認 🚦

```bash
# 📊 Podの状態確認
kubectl get pods -l app=eks-demo

# 👀 リアルタイム監視（Ctrl+Cで終了）
kubectl get pods -l app=eks-demo -w
```

### 🔹 Service作成（ClusterIP）

```bash
# 🌐 Serviceを作成（内部通信用）
kubectl apply -f 02-demo-day/service.yaml
```

<details>
<summary>📝 <b>service.yaml の内容</b></summary>

```yaml
apiVersion: v1
kind: Service
metadata:
  name: eks-demo-service
spec:
  type: ClusterIP  # ALB経由でアクセスするため
  selector:
    app: eks-demo
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

</details>

### 🔹 Ingress作成（ALB）

```bash
# 🌐 Ingressを作成（ALBが自動作成される）
kubectl apply -f 02-demo-day/ingress.yaml
```

<details>
<summary>📝 <b>ingress.yaml の内容</b></summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: eks-demo-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: eks-demo-service
            port:
              number: 80
```

</details>

---

## 🌐 Step 4: アプリケーションへのアクセス

### 🔌 **AWS ALB（Application Load Balancer）の自動作成**

<div style="background: linear-gradient(135deg, #667eea 0%, #764ba2 100%); padding: 15px; border-radius: 10px; color: white;">

⚡ **重要な概念**  
Ingress リソースを作成すると、AWS Load Balancer Controller が自動的に Application Load Balancer (ALB) を作成します。  
ALBは L7 ロードバランサーで、パスベースのルーティングやHTTPS終端などの高度な機能を提供します。

</div>

#### ALB URLの取得

```bash
# 📡 Ingressの状態確認（ALB作成に2-3分かかります）
kubectl get ingress eks-demo-ingress --watch
```

<details>
<summary>🔍 <b>期待される出力</b></summary>

```
NAME              CLASS    HOSTS   ADDRESS                                          PORTS   AGE
eks-demo-ingress  <none>   *       k8s-default-eksdemo-xxxxx.ap-northeast-1.elb.amazonaws.com   80      2m
```

</details>

#### ブラウザでアクセス

```bash
# 🌏 ALB URLを取得
ALB_URL=$(kubectl get ingress eks-demo-ingress \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

echo "http://$ALB_URL"

# 🔍 curlでテスト
curl http://$ALB_URL
```

#### ALBの詳細確認（AWSコンソール）

```bash
# ALBの一覧表示
aws elbv2 describe-load-balancers \
  --query "LoadBalancers[?contains(LoadBalancerName, 'k8s-default')].{Name:LoadBalancerName, DNS:DNSName, State:State.Code}" \
  --output table

# ターゲットグループの確認
aws elbv2 describe-target-groups \
  --query "TargetGroups[?contains(TargetGroupName, 'k8s-default')].{Name:TargetGroupName, Protocol:Protocol, Port:Port}" \
  --output table
```

> 📝 **期待される出力**  
> `Pod eks-demo-app-xxxxx-abc | Message: Hello from AWS EKS! ConfigMap is working! | Environment: production | Version: 1.0.0`

---

## ⚖️ Step 5: 負荷分散の確認

### 現在のPod一覧を確認

```bash
# 📊 3つのPodが起動していることを確認
kubectl get pods -l app=eks-demo -o wide
```

### 負荷分散テスト（手動）

```bash
# 🔄 10回連続でアクセスして、異なるPodにリクエストが分散されることを確認
for i in {1..10}; do 
  echo "Request $i:"
  curl -s http://$ALB_URL | grep "Pod"
  sleep 1
done
```

<details>
<summary>🔍 <b>期待される結果</b></summary>

```
Request 1: Pod eks-demo-app-xxxxx-abc | Message: Hello from AWS EKS!...
Request 2: Pod eks-demo-app-xxxxx-def | Message: Hello from AWS EKS!...
Request 3: Pod eks-demo-app-xxxxx-ghi | Message: Hello from AWS EKS!...
Request 4: Pod eks-demo-app-xxxxx-abc | Message: Hello from AWS EKS!...
...
```

3つの異なるPod名が表示され、リクエストが分散されていることが確認できます。

</details>

### リアルタイムモニタリング

```bash
# 👀 別ターミナルでPodの状態を監視
watch -n 2 kubectl get pods -l app=eks-demo

# 🔄 連続アクセステスト
while true; do 
  curl -s http://$ALB_URL | grep "Pod"
  sleep 0.5
done
```

---

## 🧹 Step 6: クリーンアップ

### 📝 削除の順序が重要！

<div style="background: #FFF3CD; padding: 15px; border-left: 5px solid #FFC107; margin: 20px 0;">

⚠️ **重要な注意点**  
Ingressリソースを削除せずにクラスタを削除すると、  
ALBが残ってコストが発生し続ける可能性があります。

</div>

### 1️⃣ Kubernetesリソースの削除

```bash
# 🗑️ Ingressを最初に削除（重要！ALBが削除される）
kubectl delete ingress eks-demo-ingress

# ✅ ALBが削除されたことを確認（2-3分待つ）
aws elbv2 describe-load-balancers \
  --query "LoadBalancers[?contains(LoadBalancerName, 'k8s-default')]" \
  --region ap-northeast-1

# 🗑️ その他のリソース削除
kubectl delete service eks-demo-service
kubectl delete deployment eks-demo-app
kubectl delete configmap demo-config
```

### 2️⃣ EKSクラスタの削除

```bash
# 🗑️ クラスタと関連リソースを削除（10-15分）
eksctl delete cluster \
  --name=k8s-demo-cluster \
  --region=ap-northeast-1 \
  --wait

# ✅ 削除確認
eksctl get cluster --region ap-northeast-1
```

### 3️⃣ VPCインフラの削除（オプション）

```bash
# 🗑️ VPCとすべての関連リソースを削除
aws cloudformation delete-stack \
  --stack-name k8s-demo-vpc-stack \
  --region ap-northeast-1

# ✅ 削除確認
aws cloudformation describe-stacks \
  --stack-name k8s-demo-vpc-stack \
  --region ap-northeast-1
```

---

## 🔧 トラブルシューティング

### ⚠️ よくあるエラーと対処法

<details>
<summary>🔴 <b>IAM権限不足</b></summary>

**エラー**: `UnauthorizedOperation`

**対処法**:
```bash
# 現在のIAMユーザー/ロールを確認
aws sts get-caller-identity

# 必要な権限:
# - EKS, EC2, VPC, IAM, CloudFormation への権限
```

</details>

<details>
<summary>🟡 <b>Podが起動しない（ImagePullBackOff）</b></summary>

**症状**: Pod STATUSが `ImagePullBackOff`

**対処法**:
```bash
# Pod の詳細確認
kubectl describe pod <pod-name>

# イベント確認
kubectl get events --sort-by='.lastTimestamp'

# ネットワーク確認（NAT Gatewayが正常か）
```

</details>

<details>
<summary>🟠 <b>IngressのALBが作成されない</b></summary>

**症状**: Ingress の ADDRESS が空のまま

**対処法**:
```bash
# Ingress の詳細確認
kubectl describe ingress eks-demo-ingress

# AWS Load Balancer Controller のログ確認
kubectl logs -n kube-system deployment/aws-load-balancer-controller

# サブネットにタグが付いているか確認
# kubernetes.io/role/elb=1 (Public Subnet)
# kubernetes.io/role/internal-elb=1 (Private Subnet)
```

</details>

<details>
<summary>🔵 <b>kubectlが接続できない</b></summary>

**エラー**: `Unable to connect to the server`

**対処法**:
```bash
# kubeconfigを再生成
aws eks update-kubeconfig \
  --region ap-northeast-1 \
  --name k8s-demo-cluster

# 現在のコンテキスト確認
kubectl config current-context
```

</details>

---

## 📋 コマンドリファレンス

### 🎮 基本操作

<table>
<tr>
<td width="50%">

**☸️ クラスタ管理**
```bash
# クラスタ一覧
eksctl get cluster --region ap-northeast-1

# ノードグループ確認
eksctl get nodegroup \
  --cluster=k8s-demo-cluster \
  --region ap-northeast-1

# ノード数変更
eksctl scale nodegroup \
  --cluster=k8s-demo-cluster \
  --nodes=3 \
  --name=demo-nodes
```

</td>
<td width="50%">

**📊 リソース確認**
```bash
# 全リソース表示
kubectl get all

# Pod詳細表示
kubectl get pods -o wide

# ログ確認
kubectl logs <pod-name>

# Pod内でコマンド実行
kubectl exec -it <pod-name> -- /bin/sh
```

</td>
</tr>
</table>

### 🔍 デバッグ用コマンド

```bash
# 📝 イベント確認
kubectl get events --sort-by='.lastTimestamp'

# 💻 リソース使用状況
kubectl top nodes
kubectl top pods

# 🌐 エンドポイント確認
kubectl get endpoints

# 🔧 AWS リソース確認
aws eks describe-cluster \
  --name k8s-demo-cluster \
  --region ap-northeast-1
```

---
