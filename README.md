# AWS金融系システムを想定したインフラ基盤・監視・AI活用環境の設計構築

## 概要

金融系システムを想定したAWS基盤を個人学習として構築しています。

AWS基盤の設計・構築、ネットワーク、監視、障害対応、運用設計、TerraformによるIaC、AI/RAG活用までを段階的に検証します。

---

## 目的

- AWSネットワーク設計を理解する
- AWS基盤の設計・構築を経験する
- VPC PeeringによるVPC間通信を理解する
- AWS Systems ManagerによるEC2管理を経験する
- Datadogによる監視を経験する
- 障害対応・トラブルシューティングを経験する
- TerraformによるIaCを経験する
- AI/RAGをAWS基盤へ組み込む

---

## 使用技術

| カテゴリ | 技術 |
|---|---|
| Cloud | AWS |
| Network | VPC / Subnet / Route Table / Internet Gateway |
| VPC間接続 | VPC Peering |
| Compute | Amazon EC2 / Amazon Linux 2023 |
| Web | Apache HTTP Server |
| Security | Security Group / IAM |
| Management | AWS Systems Manager Session Manager |
| IaC | Terraform |
| Monitoring | Datadog（予定） |
| Load Balancer | ALB（予定） |
| Database | RDS PostgreSQL（予定） |
| Storage | S3（予定） |
| AI | AI / RAG（予定） |

---

# ネットワーク構成

AWS東京リージョン（ap-northeast-1）に金融系システムを想定したVPCを構築しました。

## ネットワーク構成図

[![AWS Financial AI Platform Network Architecture](diagrams/network-architecture.png)](diagrams/network-architecture.png)

```text
financial-ai-vpc
10.0.0.0/16

├── Public Subnet
│   └── financial-web-server
│       ├── Amazon Linux 2023
│       ├── Apache HTTP Server
│       └── AWS Systems Manager
│
└── Private Subnet
    └── 将来のバックエンド配置を想定
```text
financial-ai-vpc
10.0.0.0/16

├── Public Subnet
│   └── financial-web-server
│       ├── Amazon Linux 2023
│       ├── Apache HTTP Server
│       └── AWS Systems Manager
│
└── Private Subnet
    └── 将来のバックエンド配置を想定
```

EC2にはApache HTTP Serverを構築し、Webサーバーとして動作確認を実施しています。

EC2の管理にはAWS Systems Manager Session Managerを使用し、SSH（TCP 22）をインターネットへ公開しない運用を検証しています。

---

# VPC Peering構成

異なるVPC間でPrivate IPによる通信を行うため、検証用VPCを追加しました。

```text
financial-ai-vpc                     peering-lab-vpc
10.0.0.0/16                          10.10.0.0/16

┌─────────────────────┐             ┌─────────────────────┐
│ financial-web-server│             │ peering-lab-ec2     │
│                     │             │                     │
│ 10.0.1.124          │◀──Peering──▶│ 10.10.1.x           │
└─────────────────────┘             └─────────────────────┘
```

VPC Peeringを作成し、双方のRoute Tableに相手VPCへのルートを設定しました。

### financial-ai-vpc側

```text
Destination: 10.10.0.0/16
Target: VPC Peering
```

### peering-lab-vpc側

```text
Destination: 10.0.0.0/16
Target: VPC Peering
```

Security Groupでは、VPC間通信に必要なICMPおよびHTTP（TCP 80）を許可しました。

---

# VPC間疎通テスト

## ICMP（ping）

`peering-lab-ec2` から `financial-web-server` のPrivate IPへ疎通確認を実施しました。

```bash
ping -c 4 10.0.1.124
```

結果：

```text
4 packets transmitted, 4 received, 0% packet loss
```

VPC Peeringを経由したPrivate IP通信に成功しました。

逆方向についても疎通確認を実施し、双方向通信が可能であることを確認しました。

```text
peering-lab-ec2
10.10.1.x
      │
      │ Private IP
      ▼
  VPC Peering
      │
      ▼
financial-web-server
10.0.1.124

ICMP 双方向通信：OK
```

---

# VPC間HTTP通信テスト

ICMPだけでなく、実際のアプリケーション通信についても確認しました。

`peering-lab-ec2` から `financial-web-server` のApacheへアクセスしました。

```bash
curl http://10.0.1.124
```

結果：

```html
<h1>Hello World!</h1><p>My AWS Financial AI Platform</p>
```

VPC Peeringを経由したHTTP（TCP 80）通信に成功しました。

これにより、

```text
peering-lab-ec2
      │
      │ HTTP :80
      ▼
  VPC Peering
      │
      ▼
financial-web-server
      │
      ▼
Apache HTTP Server
      │
      ▼
My AWS Financial AI Platform
```

というVPC間アプリケーション通信を確認できました。

---

# AWS Systems Manager

EC2への管理アクセスにはAWS Systems Manager Session Managerを使用しています。

EC2には以下のIAMロールを設定しました。

```text
financial-ec2-ssm-role
```

IAMロールにはAWS管理ポリシーを設定しています。

```text
AmazonSSMManagedInstanceCore
```

IAMロールの信頼関係ではEC2からのAssumeRoleを許可しています。

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

Session Managerを利用することで、SSHポートをインターネットへ公開せずEC2を管理する構成を検証しました。

---

# トラブルシューティング

## 1. VPC CIDR重複

当初接続を検討した既存VPC同士が、どちらも以下のCIDRを使用していました。

```text
10.0.0.0/16
```

CIDRが重複しているため、そのままVPC Peeringを構成できませんでした。

そこで検証用として、

```text
peering-lab-vpc
10.10.0.0/16
```

を新規作成しました。

### 学習ポイント

複数VPCを将来的に接続する可能性がある場合は、設計段階から重複しないCIDRを割り当てる必要があります。

例：

```text
financial-ai-vpc    10.0.0.0/16
peering-lab-vpc     10.10.0.0/16
future-vpc          10.20.0.0/16
```

---

## 2. Session Manager接続問題

新規EC2でSession Managerが、

```text
is not connected
```

となる問題が発生しました。

以下を確認しながら原因を切り分けました。

- IAMロール
- AmazonSSMManagedInstanceCore
- IAMロールの信頼関係
- SSM Agent
- Public IPv4
- Internet Gateway
- Route Table
- Systems Managerへの通信経路

IAM、ネットワーク、SSM Agentを個別に確認することで、Session Manager接続に必要な構成要素を理解しました。

---

## 3. ping成功・HTTP通信失敗

VPC Peering構築後、ICMPによる疎通確認には成功しました。

```text
ping → OK
```

一方、最初のHTTPアクセスでは、

```text
curl → Failed to connect
```

となりました。

Apache HTTP Serverの稼働状態およびSecurity GroupのTCP 80設定を確認した後、再度アクセスしました。

```bash
curl http://10.0.1.124
```

最終的に、

```html
<h1>Hello World!</h1><p>My AWS Financial AI Platform</p>
```

を取得しました。

### 学習ポイント

ICMPによる疎通確認が成功していても、HTTPなどのアプリケーションレベルの通信が正常とは限りません。

ネットワーク障害の切り分けでは、

```text
Route Table
    ↓
Security Group
    ↓
OS / Service
    ↓
Application
```

のようにレイヤーを分けて確認することが重要だと学びました。

---

# Terraform

AWS構成の一部をTerraformでコード化しました。

現在コード化している主なリソース：

```text
VPC
Subnet
Internet Gateway
Route Table
Security Group
IAM Role
IAM Instance Profile
EC2
```

Terraformの構文確認として、

```bash
terraform validate
```

を実施し、正常に完了しています。

既存AWSリソースへの意図しない変更を防ぐため、現時点では既存リソースのTerraform Stateへのimportおよび `terraform apply` は実施していません。

---

# なぜEC2をPrivate Subnetに置くのか？

外部からEC2へ直接アクセスさせる必要をなくし、攻撃対象領域を減らすためです。

最終構成では、インターネットから直接EC2へアクセスさせるのではなく、ALBなどを公開エンドポイントとして使用し、バックエンドEC2をPrivate Subnetへ配置する構成を想定しています。

---

# なぜALBをPublic側に置くのか？

インターネットからのHTTP/HTTPSアクセスをALBで受け、Private Subnet側のバックエンドEC2へ転送するためです。

```text
Internet
   │
   ▼
Public ALB
   │
   ▼
Private EC2
   │
   ▼
Application
```

---

# 現在の進捗

| 項目 | 状態 |
|---|---|
| VPC / Subnet構築 | ✅ 完了 |
| EC2構築 | ✅ 完了 |
| Apache HTTP Server | ✅ 完了 |
| IAM | ✅ 完了 |
| Systems Manager Session Manager | ✅ 完了 |
| VPC Peering | ✅ 完了 |
| 双方向Private IP疎通 | ✅ 完了 |
| VPC間HTTP通信 | ✅ 完了 |
| Terraformコード化 | ✅ 完了 |
| terraform validate | ✅ 完了 |
| Datadog監視 | ⏳ 未実施 |
| ALB | ⏳ 未実施 |
| RDS PostgreSQL | ⏳ 未実施 |
| AI / RAG | ⏳ 未実施 |

---

# 今後の予定

次のステップとしてDatadogを導入し、EC2のCPU使用率などの監視を実装します。

その後、以下の順番で環境を拡張する予定です。

```text
AWS基盤構築
    ↓
VPC Peering
    ↓
Datadog監視
    ↓
障害検知・障害対応
    ↓
ALB
    ↓
RDS PostgreSQL
    ↓
AI / RAG
```

最終的には、AWSインフラの設計・構築だけでなく、監視・障害対応・IaC・AI活用まで含めた金融系システムを想定したクラウド基盤として完成させることを目標としています。
