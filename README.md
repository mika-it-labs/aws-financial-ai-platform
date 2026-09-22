# 「AWS金融系システムを想定したインフラ基盤・監視・AI活用環境の設計構築」

## 概要

金融系システムを想定したAWS基盤を個人学習として構築する。

AWS基盤の設計・構築、監視、障害対応、
運用設計、AI/RAG活用までを段階的に検証する。

## 目的

- AWSネットワーク設計を理解する
- AWS基盤構築を経験する
- Datadogによる監視を経験する
- 障害対応を経験する
- TerraformによるIaCを経験する
- AI/RAGをAWS基盤へ組み込む

## 使用予定技術

- AWS
- VPC
- EC2
- ALB
- RDS PostgreSQL
- S3
- IAM
- Datadog
- Terraform
- AI / RAG

## なぜEC2をPrivate Subnetに置くのか？

答え：

**外部から直接アクセスさせる必要をなくし、攻撃対象領域を減らすため。**

## なぜALBをPublic側に置く？

答え：

**インターネットからのアクセスを受け、バックエンドのEC2へ転送するため。**


## ネットワーク構成と動作確認

東京リージョンにVPC（10.0.0.0/16）を構築し、パブリックサブネットとプライベートサブネットを作成した。

パブリックサブネットにEC2（t3.micro）を配置し、Apache HTTP Serverを構築した。過去にはEC2のパブリックIPv4アドレスを使用し、ローカルPCのブラウザからHTTPアクセスに成功した。

現在、EC2にはパブリックIPv4アドレスおよびElastic IPが関連付いておらず、ローカルPCからの直接HTTPアクセスはできない。Apacheの現在の稼働状態は未確認である。

EC2への管理アクセスにはAWS Systems Manager Session Managerを使用した実績があり、Security GroupではSSH（TCP 22）を開放していない。

Terraformでは既存構成をコード化し、`terraform validate` による構文検証を実施した。既存AWSリソースのTerraform Stateへの取り込みや、`terraform apply` による変更は実施していない。
