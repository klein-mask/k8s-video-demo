# 🚀 Minikube デモ環境の使い方

<div align="center">
  
  [![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.30-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)](https://kubernetes.io/)
  [![Minikube](https://img.shields.io/badge/Minikube-Latest-FF7043?style=for-the-badge&logo=kubernetes&logoColor=white)](https://minikube.sigs.k8s.io/)
  [![Docker](https://img.shields.io/badge/Docker-Required-2496ED?style=for-the-badge&logo=docker&logoColor=white)](https://www.docker.com/)
  
  **ローカルでKubernetesを体験しよう！** 🎯
  
</div>

---

## 🛠 環境構築

### 📋 必要なもの
> 🐳 **Docker**（インストール済みの前提で進めます）  
> ☸️ **minikube**  
> 🎮 **kubectl**  

### 🔧 各インストール手順

<details>
<summary>📍 <b>Mac環境の方はこちら</b></summary>

#### 1️⃣ Minikubeのインストール
```bash
# 🍺 Homebrewでインストール
brew install minikube

# ✅ バージョン確認
minikube version
```

#### 2️⃣ kubectlのインストール
```bash
# 🍺 kubectlをインストール
brew install kubectl

# ✅ バージョン確認
kubectl version --client
```

</details>

<details>
<summary>📍 <b>Windows環境の方はこちら</b></summary>

> 🔗 [Windowsの方向けminikube](https://minikube.sigs.k8s.io/docs/start/?arch=%2Fwindows%2Fx86-64%2Fstable%2F.exe+download)  
> 🔗 [Windowsの方向けkubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/)

</details>

---

## 📚 デモの内容

### 🎬 Step 1: Minikubeクラスタの起動

<table>
<tr>
<td>

```bash
# 🚀 推奨設定で起動
minikube start --memory=2048 --cpus=2
```

</td>
<td>

💡 **ポイント**  
メモリ2GB、CPU2コアを割り当て

</td>
</tr>
</table>

#### 起動後の確認 ✨

```bash
# 📊 状態確認
minikube status
```

```bash
# 🖥️ ノード確認
kubectl get nodes
```

---

### 📦 Step 2: アプリケーションのデプロイ

#### 🔹 Deployment作成（nginx）

```bash
# 🎯 Deploymentを作成
kubectl apply -f deployment.yaml
```

<details>
<summary>🔍 <b>作成後の確認コマンド</b></summary>

```bash
# Deploymentの確認
kubectl get deployments

# Podの確認
kubectl get pods
```

</details>

#### 🔹 Service作成（NodePort）

```bash
# 🌐 Serviceを作成
kubectl apply -f service.yaml
```

<details>
<summary>🔍 <b>作成後の確認コマンド</b></summary>

```bash
kubectl get services
```

</details>

---

### 🌐 Step 3: アプリケーションへのアクセス

#### 🔌 **Minikubeのサービス公開の仕組み**

<div style="background: linear-gradient(135deg, #667eea 0%, #764ba2 100%); padding: 15px; border-radius: 10px; color: white;">

⚡ **重要な概念**  
MinikubeはローカルVM内で動作するため、通常のNodePort（30080番）に直接アクセスできません。  
そこで、Minikubeが提供する特別な機能を使用します。

</div>

#### URLを確認
```bash
# 📡 URLのみ表示
minikube service demo-service --url
```

> 🔄 **このコマンドの動作**
> - MinikubeがローカルPCの空いているポートを自動的に選択
> - そのポートからVM内のNodePort（30080）へのトンネルを作成
> - 例: `http://127.0.0.1:54321` → VM内の `NodePort:30080` へ転送

#### ブラウザで開く
```bash
# 🌏 ブラウザで自動的に開く
minikube service demo-service
```

---

### ⚖️ Step 4: スケーリングの確認

#### 現在のPod数を確認
```bash
# 📊 現在の状態
kubectl get pods -l app=demo
```

#### スケールアウトを実行
```bash
# 📈 3レプリカにスケールアウト
kubectl scale deployment demo-app --replicas=3
```

#### リアルタイムで変化を観察
```bash
# 👀 リアルタイム監視（Ctrl+Cで終了）
kubectl get pods -w
```

---

### 🧹 Step 5: クリーンアップ

#### 作成したリソースを順番に削除

```bash
# 🗑️ Service削除
kubectl delete -f service.yaml
```

```bash
# 🗑️ Deployment削除（Podも自動的に削除される）
kubectl delete -f deployment.yaml
```

#### 削除の確認
```bash
# ✅ 全リソースが削除されたことを確認
kubectl get all -l app=demo
```

---

## ✅ 動作確認ポイント

### 📊 各ステップでの確認項目

| ステップ | 確認コマンド | 期待される状態 |
|:--------:|:------------|:--------------|
| 🚀 クラスタ起動 | `minikube status` | ✅ host: Running, kubelet: Running |
| 📦 Deployment作成 | `kubectl get deployments` | ✅ READY 1/1 |
| 🔄 Pod起動 | `kubectl get pods` | ✅ STATUS: Running |
| 🌐 Service作成 | `kubectl get endpoints demo-service` | ✅ ENDPOINTS にIPが表示 |
| 📈 スケール後 | `kubectl get pods -l app=demo` | ✅ 指定数のPodがRunning |

---

## 📋 よく使うコマンド集

### 🎮 基本操作

<table>
<tr>
<td width="50%">

**🏗️ クラスタ管理**
```bash
minikube start              # 起動
minikube stop               # 停止
minikube delete             # 削除
minikube dashboard          # ダッシュボード起動
```

</td>
<td width="50%">

**📊 リソース確認**
```bash
kubectl get all             # 全リソース表示
kubectl get pods -o wide    # Pod詳細表示
kubectl describe pod <name> # Pod詳細情報
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

# 🌐 ネットワーク確認
kubectl get endpoints
kubectl describe service <name>

# 🔧 Minikube関連
minikube ssh               # ノードにSSH接続
minikube logs              # Minikubeログ確認
```

---

## 🔧 トラブルシューティング

### ⚠️ よくあるエラーと対処法

<details>
<summary>🔴 <b>Docker Desktopが起動していない</b></summary>

**エラー**: `Docker daemon is not running`

**対処法**:
```bash
# Docker Desktopを起動
open -a Docker

# 起動を待ってから再試行
sleep 30
minikube start
```

</details>

<details>
<summary>🟡 <b>Podが起動しない（ImagePullBackOff）</b></summary>

**症状**: Pod STATUSが `ImagePullBackOff`

**対処法**:
```bash
# 詳細確認
kubectl describe pod <pod-name>

# ネットワーク確認
ping hub.docker.com

# プロキシ環境の場合
minikube start --docker-env HTTP_PROXY=$HTTP_PROXY
```

</details>

<details>
<summary>🟠 <b>メモリ不足</b></summary>

**エラー**: `Insufficient memory`

**対処法**:
```bash
# Minikube再作成（メモリ増加）
minikube delete
minikube start --memory=4096 --cpus=2

# Docker Desktop設定確認
# Preferences > Resources > Memory を4GB以上に
```

</details>

---

## 📝 補足説明

### 🌐 MinikubeのService公開方法

### 🔄 実際のKubernetesクラスタとの違い

| 環境 | 特徴 |
|:----:|:-----|
| 🏢 **本番環境** | NodePortは全ノードの指定ポート（30000-32767）で直接アクセス可能 |
| 💻 **Minikube** | `minikube service`コマンドでトンネリングが必要 |
