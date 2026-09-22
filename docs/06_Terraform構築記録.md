# Terraform構築・学習記録

**プロジェクト：AWS Financial AI Platform**

| 項目 | 内容 |
|---|---|
| 作業期間 | 2026年9月21日〜22日 |
| 作業環境 | Windows 11 / Windows Terminal |
| Terraform | v1.16.2 |
| AWS CLI | v2系 |
| AWSリージョン | ap-northeast-1（東京） |
| AWSアカウント | 無料プラン・無料クレジット利用中 |
| 構築方針 | 既存AWS環境を変更せず、Terraformコードを先行作成 |
| 現在の到達点 | ネットワーク構成のTerraformコードを作成し、構文検証に成功 |

---

## 1. 作業目的

AWSコンソールで手動構築したインフラをTerraformでコード化し、Infrastructure as Code（IaC）の基本を習得する。

最終的には、以下を目指す。

- AWS構成をコードで管理する。
- 設計書とTerraformコードを対応させる。
- GitHubで変更履歴を管理する。
- 同じ構成を再現できる状態にする。
- 既存環境の安全性と料金を考慮して運用する。

現段階では、既存AWS環境のTerraform管理への移行は実施しない。

---

## 2. 作業開始時のAWS環境

以下はTerraform導入前に、AWSコンソールから手動構築したリソースである。

| リソース | 名称・設定 | 状態 |
|---|---|---|
| VPC | financial-ai-vpc / 10.0.0.0/16 | 手動構築済み |
| Public Subnet | financial-public-subnet-a / 10.0.1.0/24 | 手動構築済み |
| Private Subnet | financial-private-subnet-a / 10.0.11.0/24 | 手動構築済み |
| Internet Gateway | financial-igw | 手動構築済み |
| Public Route Table | financial-public-rt | 手動構築済み |
| Private Route Table | financial-private-rt | 手動構築済み |
| Security Group | financial-web-sg | 手動構築済み |
| IAM Role | financial-ec2-ssm-role | 手動構築済み |
| EC2 | financial-web-server | 手動構築済み |
| Apache | HTTP 80 / Hello World! | 表示確認済み |

HTTP通信の障害発生・復旧テストも実施し、GitHubに障害対応記録を保存済み。

**注意：これらはTerraformによって構築したリソースではない。**

---

# 3. Terraform環境構築

## 3.1 Terraformのインストール

Windows TerminalでTerraformをインストールした。

```powershell
winget install Hashicorp.Terraform
```

バージョン確認：

```powershell
terraform -version
```

実行結果：

```text
Terraform v1.16.2
on windows_amd64
```

**結果：成功**

最新版への更新通知が表示されたが、学習を継続できる状態である。

## 3.2 AWS CLIの確認

```powershell
aws --version
```

実行結果：

```text
aws-cli/2.x.x
```

**結果：AWS CLI v2系のインストールを確認**

詳細なバージョン番号は未記録。

## 3.3 AWS認証状態の確認

```powershell
aws sts get-caller-identity
```

実行結果：

```text
Unable to locate credentials
```

**結果：Windows側のAWS CLI認証は未設定**

この時点ではAWSへの認証情報が存在しないことを確認した。

---

# 4. AWS認証方式の検討

## 4.1 IAMユーザーの作成

ルートユーザーのアクセスキーを作成しない方針とした。

IAMで以下のユーザーを作成した。

| 項目 | 設定 |
|---|---|
| ユーザー名 | terraform-admin |
| 許可ポリシー | ReadOnlyAccess |
| コンソールアクセス | なし |
| アクセスキー | 未作成 |

**結果：IAMユーザー作成済み**

ユーザー名は `terraform-admin` だが、現在は管理者権限を持たない読み取り専用ユーザーである。

このユーザーによるAWS CLI認証は未設定。

## 4.2 CloudShellの確認

AWS CloudShellを開き、次のコマンドを実行した。

```bash
aws sts get-caller-identity --query Arn --output text
```

結果のARN末尾が `:root` であることを確認した。

ルートユーザーとしてTerraformを実行しない方針とし、CloudShellを閉じた。

**結果：CloudShellによるTerraform実行は中止**

## 4.3 IAM Identity Centerの検討

IAM Identity Centerの有効化画面を確認した。

画面には、AWS Organizationsの作成によって無料プランから有料プランへ移行し、無料利用枠クレジットが期限切れになる旨の警告が表示された。

また、選択されていたマルチリージョンインスタンスにはAWS KMSの料金に関する警告も表示された。

無料プランの維持を優先し、Identity Centerの有効化を中止した。

**結果：IAM Identity Centerは未有効化**

### 認証方式に関する現在の方針

- ルートユーザーのアクセスキーは作成しない。
- 長期アクセスキーの発行は保留する。
- TerraformからAWSへの接続はまだ行わない。
- 当面はローカル環境でコード作成と構文検証を進める。

---

# 5. Terraform基本操作の練習

## 5.1 練習用フォルダー

```text
%USERPROFILE%\terraform-practice
```

`main.tf` を作成し、以下の基本要素を学習した。

- terraformブロック
- variable
- locals
- output

練習用コード：

```hcl
terraform {
  required_version = ">= 1.16.0"
}

variable "project_name" {
  type    = string
  default = "aws-financial-ai-platform"
}

locals {
  environment = "dev"
}

output "project_info" {
  value = "${var.project_name}-${local.environment}"
}
```

## 5.2 実行したコマンド

```powershell
terraform init
terraform fmt
terraform validate
terraform apply
```

`terraform apply` でエラーが発生した。

エラーメッセージの詳細は未取得のため、原因は未特定。

後続の作業では `terraform plan` による確認を案内したが、結果は未記録。

**ステータス：エラー未解決**

この練習用コードにはAWSプロバイダーとAWSリソースを定義していない。

---

# 6. AWSネットワーク構成のコード化

## 6.1 作業フォルダー

```text
%USERPROFILE%\aws-financial-ai-terraform
```

このフォルダーは、既存のAWSインフラをTerraformで表現するために作成した。

## 6.2 main.tfの作成

次のリソースをTerraformコードとして定義した。

| Terraformリソース | 内容 |
|---|---|
| aws_vpc | VPC |
| aws_subnet | Public / Private Subnet |
| aws_internet_gateway | Internet Gateway |
| aws_route_table | Public / Private Route Table |
| aws_route_table_association | サブネットとルートテーブルの関連付け |

AWSプロバイダーのリージョン：

```hcl
provider "aws" {
  region = "ap-northeast-1"
}
```

VPCの定義例：

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = "financial-ai-vpc"
  }
}
```

### コード化したネットワーク構成

```text
VPC：10.0.0.0/16
│
├── Public Subnet：10.0.1.0/24
│   └── Public Route Table
│       └── 0.0.0.0/0 → Internet Gateway
│
└── Private Subnet：10.0.11.0/24
    └── Private Route Table
        └── VPC内通信のみ
```

NAT Gatewayは定義していない。

プライベートサブネットとルートテーブルの関連付けはコードに含めたが、既存環境との完全な一致は未確認。

---

# 7. Terraformコードの検証

ネットワーク構成のコードを作成した後、次のコマンドを実行した。

```powershell
terraform fmt
terraform init -backend=false
terraform validate
```

実行結果：

```text
Success! The configuration is valid.
```

**結果：構文検証成功**

この結果から、作成したTerraform構成が構文検証を通過したことを確認した。

ただし、以下は確認していない。

- AWS認証の成功
- 実際のAWS環境との設定一致
- AWS上でのリソース作成
- 既存リソースのTerraform管理への取り込み
- AWS料金への影響

`terraform validate` の成功だけでは、AWSへの適用が成功するとは限らない。

---

# 8. Security Group・IAMのコード化

次の段階として、`compute.tf` の作成を開始した。

## 8.1 Security Group

コードには以下の設定を定義した。

| 項目 | 設定 |
|---|---|
| セキュリティグループ名 | financial-web-sg |
| HTTP | TCP 80 / 0.0.0.0/0 |
| SSH | 許可ルールなし |
| アウトバウンド | IPv4全通信許可 |

## 8.2 IAM・SSM

以下を定義した。

- EC2用IAMロール
- EC2がIAMロールを引き受けるための信頼ポリシー
- AmazonSSMManagedInstanceCoreのアタッチ
- IAM Instance Profile

Instance Profile名はコード上の設計値であり、既存AWS環境との照合は未完了。

## 8.3 EC2

EC2のコード化に必要な実設定を確認する段階まで進んだ。

確認対象：

- AMI ID
- インスタンスタイプ
- ルートEBS容量
- パブリックIPv4の設定
- 既存のInstance Profile

これらの実測値はまだ記録されていない。

**ステータス：EC2のリソース定義は未完成**

また、`compute.tf` の保存完了および追加後の `terraform validate` 成功については、まだ結果報告を受けていないため未確認とする。

---

# 9. AWS環境への影響

今回のTerraform編では、AWS用Terraformコードに対して次の操作を実施していない。

```text
terraform apply
terraform import
terraform destroy
```

既存のVPCやEC2をTerraformで変更した実績はない。

一方、認証準備としてAWS上にIAMユーザー `terraform-admin` を作成している。

そのため、今回の作業全体について「AWSへの変更ゼロ」とは記載しない。

**正確な状態：既存のVPC・EC2はTerraformから変更していない。IAMユーザーは新規作成済み。**

---

# 10. セキュリティ・料金面の判断

今回の作業では、次の方針を採用した。

- ルートユーザーのアクセスキーを作成しない。
- Terraform用ユーザーに管理者権限を付与しない。
- AWS認証情報をTerraformコードへ直接記載しない。
- IAM Identity Centerの有効化による料金プラン変更を避ける。
- NAT Gatewayなどの新規有料リソースを作成しない。
- 既存AWSリソースをTerraformで操作しない。

Terraformのローカル構文検証を中心に進めることで、追加のAWSインフラを作成せずに学習を継続した。

---

# 11. 現在の進捗一覧

| 作業 | 状態 |
|---|---|
| Terraformインストール | 完了 |
| AWS CLIインストール確認 | 完了 |
| AWS CLI認証設定 | 未完了 |
| 読み取り専用IAMユーザー作成 | 完了 |
| IAM Identity Center導入 | 中止 |
| Terraform基本操作練習 | applyエラー未解決 |
| VPCコード化 | 完了 |
| Subnetコード化 | 完了 |
| Internet Gatewayコード化 | 完了 |
| Route Tableコード化 | 完了 |
| ネットワーク構成のvalidate | 成功 |
| Security Groupコード化 | コード提示済み・保存未確認 |
| IAM / SSMコード化 | コード提示済み・保存未確認 |
| EC2コード化 | 未完成 |
| 既存AWS設定との照合 | 未実施 |
| terraform import | 未実施 |
| AWS用コードのterraform apply | 未実施 |
| GitHubへのTerraformコード保存 | 未確認 |

---

# 12. 次回の作業計画

## 優先度1：コードの完成と検証

1. `compute.tf` の保存状況を確認する。
2. `terraform fmt` を実行する。
3. `terraform validate` を実行する。
4. EC2の実際の設定値を確認する。
5. EC2のTerraformコードを完成させる。

## 優先度2：既存AWS環境との照合

以下を確認する。

- VPC ID
- Subnet IDとAZ
- Route Tableの関連付け
- Security Groupの実際のルール
- IAM RoleとInstance Profile
- EC2のAMI・インスタンスタイプ・EBS

既存リソースとTerraformコードの差異を整理する。

## 優先度3：Terraform管理方式の決定

既存環境をTerraformに取り込むか、別の検証環境を新規作成するかを検討する。

無料プランと残りのクレジット、既存リソースへの影響を確認してから決定する。

## 優先度4：GitHub管理

TerraformコードをGitHubへ保存する。

ただし、以下は公開しない。

```text
*.tfstate
*.tfstate.*
.terraform/
*.tfvars
*.tfplan
```

`.terraform.lock.hcl` は原則としてバージョン管理する。

AWS認証情報、パスワード、秘密鍵なども公開しない。

---

# 13. 今回の学習成果

今回の作業を通じて、以下を学習した。

1. TerraformのWindows環境構築
2. TerraformとAWS CLIの役割の違い
3. AWS認証方式とルートユーザー利用のリスク
4. AWS無料プランへの影響を考慮した構築判断
5. Terraformの基本構文
6. VPC・サブネット・ルートテーブルのコード化
7. `terraform init`・`fmt`・`validate` の基本操作
8. 構文検証とAWSへの実際の適用の違い
9. 既存リソースを管理する際の注意点

現段階の成果は、**AWSネットワーク構成のTerraformコード作成と構文検証の成功**である。

TerraformによるAWSリソースのデプロイや、既存環境のIaC管理は今後の課題とする。

---

# 14. 更新履歴

| 日付 | 内容 |
|---|---|
| 2026-09-22 | Terraform環境構築、認証方式の検討、ネットワークコード化、構文検証、今後の課題を記録 |
