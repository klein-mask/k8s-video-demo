# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## プロジェクト概要

このプロジェクトは、KubernetesのデモンストレーションとチュートリアルのためのYAML設定ファイルとドキュメントを含むリポジトリです。Minikube（ローカル環境）とAWS EKS（本番環境）の両方での実装例を提供しています。

## コマンド

### Minikubeでのデモ実行

```bash
# Minikubeクラスタ起動
minikube start --memory=2048 --cpus=2

# アプリケーションデプロイ
kubectl apply -f minikube/deployment.yaml
kubectl apply -f minikube/service.yaml

# サービスへのアクセス
minikube service demo-service --url
minikube service demo-service  # ブラウザで自動的に開く

# スケーリング
kubectl scale deployment demo-app --replicas=3

# クリーンアップ
kubectl delete -f minikube/service.yaml
kubectl delete -f minikube/deployment.yaml
minikube stop
```

### AWS EKSでのデプロイ

```bash
# 事前準備：VPC作成（01-preparation/infrastructureディレクトリ）
aws cloudformation create-stack --stack-name k8s-demo-vpc-stack --template-body file://vpc-stack.yaml --region ap-northeast-1

# EKSクラスタ作成（既存VPCを利用）
eksctl create cluster -f 01-preparation/cluster-config-with-vpc.yaml

# kubeconfigの設定
aws eks update-kubeconfig --region ap-northeast-1 --name k8s-demo-cluster

# アプリケーションデプロイ（02-demo-dayディレクトリ）
kubectl apply -f 02-demo-day/configmap.yaml
kubectl apply -f 02-demo-day/deployment.yaml
kubectl apply -f 02-demo-day/service.yaml
kubectl apply -f 02-demo-day/ingress.yaml

# ALB URLの取得
kubectl get ingress eks-demo-ingress -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'

# クリーンアップ（順序が重要）
kubectl delete ingress eks-demo-ingress  # ALBを最初に削除
kubectl delete -f 02-demo-day/
eksctl delete cluster --name=k8s-demo-cluster --region=ap-northeast-1 --wait
aws cloudformation delete-stack --stack-name k8s-demo-vpc-stack --region ap-northeast-1
```

## アーキテクチャ構成

### プロジェクト構造
- **minikube/**: ローカル開発環境用のKubernetes設定
  - シンプルなnginxデプロイメント（1レプリカから開始）
  - NodePortサービスでポート30080を公開
  - Minikubeのトンネル機能を使用してアクセス

- **eks/**: AWS EKS本番環境用の設定
  - **01-preparation/**: 事前準備（VPCインフラ構築、クラスタ作成）
    - CloudFormationでVPC、サブネット、NAT Gatewayを作成
    - eksctlでマネージドノードグループを含むクラスタを作成
  - **02-demo-day/**: デモ用アプリケーション設定
    - ConfigMapで環境変数を管理
    - 3レプリカのデプロイメント（hashicorp/http-echo使用）
    - ClusterIPサービス + Ingress（ALB）構成

### 主要な設計判断

1. **VPCの事前作成**: EKSクラスタ作成前にVPCを個別に作成することで、インフラストラクチャのコスト管理を改善（NAT Gatewayが高価なため）

2. **プライベートサブネットの利用**: EKSのワーカーノードはプライベートサブネットに配置し、ALB経由でのみ公開

3. **AWS Load Balancer Controllerの使用**: Ingress作成時に自動的にALBを作成・管理するためのコントローラーを使用

4. **リソース制限の設定**: すべてのPodにCPU/メモリのrequests/limitsを設定し、リソース管理を適切に実施

## 注意事項

- EKSデプロイ時は必ずIngressリソースを先に削除してからクラスタを削除する（ALBが残ってコストが発生するのを防ぐ）
- AWS Load Balancer Controllerのインストールが必要（IAMサービスアカウント、Helmでのインストール）
- eksctlで作成したクラスタのサブネットには自動的にELB用のタグが付与される