# 【EKS】新CICDモジュール ACL適用手順書
EKSテナントに対して、ACLを取込み・適用する手順を記載致します。

## 目次
- [【EKS】新CICDモジュール ACL適用手順書](#eks新cicdモジュール-acl適用手順書)
  - [目次](#目次)
  - [前提条件](#前提条件)
  - [事前作業](#事前作業)
  - [事前作業(Helm Chart/HybridFiles 用意)](#事前作業helm-charthybridfiles-用意)
  - [直前作業](#直前作業)
  - [デフォルトACL取込み](#デフォルトacl取込み)
    - [eks.tfにACLを定義する](#ekstfにaclを定義する)
    - [importを行う](#importを行う)
  - [ACL適用](#acl適用)
  - [事後確認](#事後確認)


---
## 前提条件
---
- GCP Project IDに対して必要な権限が付与されたSAのjson keyが用意されている事  
- 必要となるクラウド(GCP/Azure/AWS)のクレデンシャル情報を把握している事
- Dockerコンテナが動作する作業環境が用意されている事
- 新CICDモジュールのcontainerイメージがローカルに存在する事

---
## 事前作業
---
| company | clusterName | ProjectId | region | VM IP | 補足 |
| --- | --- | --- | --- | --- | --- | 
| OCN Service Stg JapanWest |apg-e7eec884-5b70 |ncom-apigw-axross-g02-o22-2504 |ap-northeast-3 | 20.194.177.0 | |
| OCN Service Prd JapanWest | apg-b7474510-c9b2 | ncom-apigw-axross-g02-o24-0354 | ap-northeast-3 | 20.78.26.155 | |
| OCN Service Stg JapanEast | apg-78fe6ee4-ebca | ncom-apigw-axross-g02-o21-9447 | ap-northeast-1 | 4.241.6.42 | |
| OCN Service Prd JapanEast | apg-e49bff8f-eba1 | ncom-apigw-axross-g02-o23-2836 | ap-northeast-1 | 20.194.181.94 | |

- 対象vmにテナント資材を用意する
  - terraform template一式の配置(tfvars,stage1/\*,stage2/\*)  
    対象テナントのterraform templateファイル一式をgitより配置する  
    以下の様に配置される事  
    ```
    pwd$ls
    /tmp/<clusterName>
    stage1 stage2 terraform.tfvars
    ```  

  - Service Account Keyを配置する  
    以下の様な形になるようにGCPのSA keyを配置する
    ```
    pwd & ls
    /tmp/<clusterName>
    stage1 stage2 terraform.tfvars
    cicd-test-sa.json
    ```  

  - hybrid-files(overrides/certs/service-accounts)一式を、作業ディレクトリ配下に配置する  
    以下の様な形で配置される事
    ```
    pwd & ls
    /tmp/<clusterName>
    stage1 stage2 terraform.tfvars
    cicd-test-sa.json 
    hybrid-files

    ls hybrid-files
    certs overrides accounts
    ```  

- cicdモジュール利用準備  
  -  Azureへログイン
     ```
     az login --service-principal --username <azureClientId> --tenant <AZURE_TENANT_ID> --password <azureClientSecret>
     ```
  -  ACRへログイン
     ```
     az acr login -n cicdmodule
     ```
  -  ACRからdockerをPullする
     ```
     docker pull cicdmodule.azurecr.io/cicd-module:release-202410
     ```

  - alias設定  
    cicdmコマンドを使用するため、以下コマンドでaliasを設定する事
    ```
    alias cicdm='sudo docker run --rm -it -e TF_TOKEN_app_terraform_io=$TF_TOKEN -v `pwd`:/go/work -w /go/work cicdmodule.azurecr.io/cicd-module:release-202410'
    ```

- Terraform Cloudの再initializeを行う
  - stage1のinitializeを行う
    ```
    cicdm terraform -chdir=stage1 init -upgrade
    ```
    「Terraform Cloud has been successfully initialized!」と出力される事
  - stage2のinitializeを行う
    ```
    cicdm terraform -chdir=stage2 init -upgrade
    ```
    「Terraform Cloud has been successfully initialized!」と出力される事  

- 新CICD用VMの環境セットアップ  
  - 以下コマンドがインストールされている事  
    - gcloud  
    - az  
    - kubectl  
    - git  
    - screen  
    - helm  
  - 対象クラスタへログイン後、以下コマンドにてK8Sのリソース参照が出来る事
    ```
    kubectl get pod -n apigee
    ```

<bcr>

---
## 事前作業(Helm Chart/HybridFiles 用意)
---

- 作業用DIRに移動する
  ```
  cd /tmp/<clusterName>
  export CICDMODULE_HOME=$PWD
  ```
- Hybrid-filesを配置する
  ```
  cp -vip $CICDMODULE_HOME/hybrid-files/accounts/*cassandra.json apigee-datastore/accounts/
  cp -vip $CICDMODULE_HOME/hybrid-files/accounts/*runtime.json      apigee-env/accounts/
  cp -vip $CICDMODULE_HOME/hybrid-files/accounts/*synchronizer.json apigee-env/accounts/
  cp -vip $CICDMODULE_HOME/hybrid-files/accounts/*udca.json         apigee-env/accounts/
  cp -vip $CICDMODULE_HOME/hybrid-files/accounts/*mart.json    apigee-org/accounts/
  cp -vip $CICDMODULE_HOME/hybrid-files/accounts/*udca.json    apigee-org/accounts/
  cp -vip $CICDMODULE_HOME/hybrid-files/accounts/*watcher.json apigee-org/accounts/
  cp -vip $CICDMODULE_HOME/hybrid-files/accounts/*logger.json  apigee-telemetry/accounts/
  cp -vip $CICDMODULE_HOME/hybrid-files/accounts/*metrics.json apigee-telemetry/accounts/
  cp -vip $CICDMODULE_HOME/hybrid-files/certs/* apigee-virtualhost/certs/
  cp -vip $CICDMODULE_HOME/apigee-hybrid/helm-charts/*.yaml ./
  ```
---
## 直前作業
---
| company | clusterName | ProjectId | region | VM IP | 補足 |
| --- | --- | --- | --- | --- | --- | 
| OCN Service Stg JapanWest |apg-e7eec884-5b70 |ncom-apigw-axross-g02-o22-2504 |ap-northeast-3 | 20.194.177.0 | |
| OCN Service Prd JapanWest | apg-b7474510-c9b2 | ncom-apigw-axross-g02-o24-0354 | ap-northeast-3 | 20.78.26.155 | |
| OCN Service Stg JapanEast | apg-78fe6ee4-ebca | ncom-apigw-axross-g02-o21-9447 | ap-northeast-1 | 4.241.6.42 | |
| OCN Service Prd JapanEast | apg-e49bff8f-eba1 | ncom-apigw-axross-g02-o23-2836 | ap-northeast-1 | 20.194.181.94 | |

- screenコマンドでセッションをアタッチ  
  - 構成ファイル一式が配置された作業ディレクトリに移動する 
    ```
    screen
    ```
    ※[Ctrl]+[a] [d] でデタッチできる  

    もし急にソフトウェアが落ちてしまった場合、以下コマンドでscreen セッションに再度アタッチする
    ```
    screen -r
    ```  

- 作業ディレクトリ移動
  - 構成ファイル一式が配置された作業ディレクトリに移動する  
    ※listの[clusterName]が対象
    ```
    cd /tmp/<clusterName>
    ```

- cicdモジュール利用準備
  - alias設定  
    cicdmコマンドを使用するため、以下コマンドでaliasを設定する事
    ```
    alias cicdm='sudo docker run --rm -it -e TF_TOKEN_app_terraform_io=$TF_TOKEN -v `pwd`:/go/work -w /go/work cicdmodule.azurecr.io/cicd-module:release-202410'
    ```
  - TF_TOKENの設定
    ```
    export TF_TOKEN="tftoken"
    ```
  - stage1 plan  
    Terraform使えるかの正常確認としてstage1のplanを実行する。  
    No Chagesが出力される事を確認する事。
    ```
    cicdm terraform -chdir=stage1 plan -var-file="../terraform.tfvars"
  - stage2 plan  
    Terraform使えるかの正常確認としてstage2のplanを実行する。  
    No Chagesが出力される事を確認する事。
    ```
    cicdm terraform -chdir=stage2 plan -var-file="../terraform.tfvars"
    ```  
- 接続先確認
  - cicdモジュール  
    ./terraform.tfvarsの内容を確認し、対象テナントの構成ファイルであるか確認する  
    ```
    cat ./terraform.tfvars
    gcp_project_name         = "listの[ProjectId]と比較し、対象テナントのGcpIdである事"
    tenant_eks_name          = "listの[clusterName]と比較し、対象テナントのCluster名である事"
    ```
---
## デフォルトACL取込み
--- 
既存のACLをTerraformにimportする

◆確認用list
| company | clusterName | ProjectId | region | VM IP | 補足 |
| --- | --- | --- | --- | --- | --- | 
| OCN Service Stg JapanWest |apg-e7eec884-5b70 |ncom-apigw-axross-g02-o22-2504 |ap-northeast-3 | 20.194.177.0 | |
| OCN Service Prd JapanWest | apg-b7474510-c9b2 | ncom-apigw-axross-g02-o24-0354 | ap-northeast-3 | 20.78.26.155 | |
| OCN Service Stg JapanEast | apg-78fe6ee4-ebca | ncom-apigw-axross-g02-o21-9447 | ap-northeast-1 | 4.241.6.42 | |
| OCN Service Prd JapanEast | apg-e49bff8f-eba1 | ncom-apigw-axross-g02-o23-2836 | ap-northeast-1 | 20.194.181.94 | |

### eks.tfにACLを定義する

以下を`stage1/eks.tf`に定義する。  
※最下部で良い
```
resource "aws_default_network_acl" "cluster_acl" {
  default_network_acl_id = aws_vpc.cluster_vpc.default_network_acl_id
  subnet_ids = [
    aws_subnet.public_subnet_1.id,
    aws_subnet.public_subnet_2.id,
    aws_subnet.public_subnet_3.id,
    aws_subnet.private_subnet_1.id,
    aws_subnet.private_subnet_2.id,
    aws_subnet.private_subnet_3.id
  ]

  egress {
     action          = "allow"
     cidr_block      = "0.0.0.0/0"
     from_port       = 0
     icmp_code       = 0
     icmp_type        = 0
     protocol        = "-1"
     rule_no         = 100
     to_port         = 0
  }
  ingress {
     action          = "allow"
     cidr_block      = "0.0.0.0/0"
     from_port       = 0
     icmp_code       = 0
     icmp_type        = 0
     protocol        = "-1"
     rule_no         = 100
     to_port         = 0
  }
  ingress {
    action           = "deny"
    cidr_block       = "20.48.18.209/32"
    from_port        = 0
    icmp_code        = 0
    icmp_type        = 0
    protocol         = "-1"
    rule_no          = 1
    to_port          = 0
  }
  ingress {
    action           = "deny"
    cidr_block       = "20.48.41.226/32"
    from_port        = 0
    icmp_code        = 0
    icmp_type        = 0
    protocol         = "-1"
    rule_no          = 2
    to_port          = 0
  }
  ingress {
    action           = "deny"
    cidr_block       = "20.78.142.129/32"
    from_port        = 0
    icmp_code        = 0
    icmp_type        = 0
    protocol         = "-1"
    rule_no          = 3
    to_port          = 0
  }
  ingress {
    action           = "deny"
    cidr_block       = "20.89.244.233/32"
    from_port        = 0
    icmp_code        = 0
    icmp_type        = 0
    protocol         = "-1"
    rule_no          = 4
    to_port          = 0
  }
  ingress {
    action           = "deny"
    cidr_block       = "20.104.211.17/32"
    from_port        = 0
    icmp_code        = 0
    icmp_type        = 0
    protocol         = "-1"
    rule_no          = 5
    to_port          = 0
  }
  ingress {
    action           = "deny"
    cidr_block       = "20.151.230.239/32"
    from_port        = 0
    icmp_code        = 0
    icmp_type        = 0
    protocol         = "-1"
    rule_no          = 6
    to_port          = 0
  }
  ingress {
    action           = "deny"
    cidr_block       = "20.222.104.90/32"
    from_port        = 0
    icmp_code        = 0
    icmp_type        = 0
    protocol         = "-1"
    rule_no          = 7
    to_port          = 0
  }
  ingress {
    action           = "deny"
    cidr_block       = "20.243.24.205/32"
    from_port        = 0
    icmp_code        = 0
    icmp_type        = 0
    protocol         = "-1"
    rule_no          = 8
    to_port          = 0
  }
  ingress {
    action           = "deny"
    cidr_block       = "20.243.163.24/32"
    from_port        = 0
    icmp_code        = 0
    icmp_type        = 0
    protocol         = "-1"
    rule_no          = 9
    to_port          = 0
  }
  ingress {
    action           = "deny"
    cidr_block       = "40.74.64.226/32"
    from_port        = 0
    icmp_code        = 0
    icmp_type        = 0
    protocol         = "-1"
    rule_no          = 10
    to_port          = 0
  }
  ingress {
    action           = "deny"
    cidr_block       = "45.153.129.75/32"
    from_port        = 0
    icmp_code        = 0
    icmp_type        = 0
    protocol         = "-1"
    rule_no          = 11
    to_port          = 0
  }
  ingress {
    action           = "deny"
    cidr_block       = "52.156.24.42/32"
    from_port        = 0
    icmp_code        = 0
    icmp_type        = 0
    protocol         = "-1"
    rule_no          = 12
    to_port          = 0
  }
  ingress {
    action           = "deny"
    cidr_block       = "107.150.112.193/32"
    from_port        = 0
    icmp_code        = 0
    icmp_type        = 0
    protocol         = "-1"
    rule_no          = 13
    to_port          = 0
  }
  ingress {
    action           = "deny"
    cidr_block       = "152.32.138.156/32"
    from_port        = 0
    icmp_code        = 0
    icmp_type        = 0
    protocol         = "-1"
    rule_no          = 14
    to_port          = 0
  }
}
```

### importを行う

- OCN Service Stg JapanWestの場合
  ```
  cicdm terraform -chdir=stage1 import -var-file=../terraform.tfvars aws_default_network_acl.cluster_acl acl-0a85f9a6785533fd1
  ```

- OCN Service Prd JapanWestの場合
  ```
  cicdm terraform -chdir=stage1 import -var-file=../terraform.tfvars aws_default_network_acl.cluster_acl acl-0aafa23f6e087ee42
  ```

- OCN Service Stg JapanEastの場合
  ```
  cicdm terraform -chdir=stage1 import -var-file=../terraform.tfvars aws_default_network_acl.cluster_acl acl-0a6b3b229ac4c8ac3
  ```

- OCN Service Prd JapanEastの場合
  ```
  cicdm terraform -chdir=stage1 import -var-file=../terraform.tfvars aws_default_network_acl.cluster_acl acl-01f6b0e36a38cf731
  ```

⇒[Import successful!]が出力される事

## ACL適用
- stage1をApplyし、ACLを適用する
  ```
  cicdm terraform -chdir=stage1 apply -var-file="../terraform.tfvars"
  ```
  ```
  出力例：
    # aws_default_network_acl.cluster_acl will be updated in-place
    ~ resource "aws_default_network_acl" "cluster_acl" {
          id                     = "acl-0e5744c3c2e5653f2"
          tags                   = {}

        - ingress {
            - action          = "allow" -> null
            - cidr_block      = "0.0.0.0/0" -> null
            - from_port       = 0 -> null
            - icmp_code       = 0 -> null
            - icmp_type       = 0 -> null
            - protocol        = "-1" -> null
            - rule_no         = 100 -> null
            - to_port         = 0 -> null
              # (1 unchanged attribute hidden)
          }
        + ingress {
            + action     = "allow"
            + cidr_block = "0.0.0.0/0"
            + from_port  = 0
            + icmp_code  = 0
            + icmp_type  = 0
            + protocol   = "-1"
            + rule_no    = 100
            + to_port    = 0
          }
                      ~~~    skip    ~~~
          }
        + ingress {
            + action          = "deny"
            + cidr_block      = "52.156.24.42/32"
            + from_port       = 0
            + icmp_code       = 0
            + icmp_type       = 0
            + protocol        = "-1"
            + rule_no         = 12
            + to_port         = 0
              # (1 unchanged attribute hidden)
          }
      }
  ```
  ※既存のall allowのRuleが削除されていても、同等のRuleが追加になっていれば問題無い。  
  ⇒[Apply Complate!]が出力される事

## 事後確認
- OCN Service Stg JapanWestの場合
  ```
  aws ec2 describe-network-acls --output text --network-acl-ids acl-0a85f9a6785533fd1
  ```

- OCN Service Prd JapanWestの場合
  ```
  aws ec2 describe-network-acls --output text --network-acl-ids acl-0aafa23f6e087ee42
  ```

- OCN Service Stg JapanEastの場合
  ```
  aws ec2 describe-network-acls --output text --network-acl-ids acl-0a6b3b229ac4c8ac3
  ```

- OCN Service Prd JapanEastの場合
  ```
  aws ec2 describe-network-acls --output text --network-acl-ids acl-01f6b0e36a38cf731
  ```

⇒想定通りのENTRIESが出力されること。(RuleID:1~14)
```
ENTRIES 20.48.18.209/32 False   -1      deny    1
ENTRIES 20.48.41.226/32 False   -1      deny    2
ENTRIES 20.78.142.129/32        False   -1      deny    3
ENTRIES 20.89.244.233/32        False   -1      deny    4
ENTRIES 20.104.211.17/32        False   -1      deny    5
ENTRIES 20.151.230.239/32       False   -1      deny    6
ENTRIES 20.222.104.90/32        False   -1      deny    7
ENTRIES 20.243.24.205/32        False   -1      deny    8
ENTRIES 20.243.163.24/32        False   -1      deny    9
ENTRIES 40.74.64.226/32 False   -1      deny    10
ENTRIES 45.153.129.75/32        False   -1      deny    11
ENTRIES 52.156.24.42/32 False   -1      deny    12
ENTRIES 107.150.112.193/32      False   -1      deny    13
ENTRIES 152.32.138.156/32       False   -1      deny    14
```

## Call確認
- 以下コマンドでヘルスチェック先をコールし、問題なく疎通ができることを確認
- OCN Service Stg JapanWestの場合
  ```
  curl -H "User-Agent: Datadog/Synthetics" https://osaka-stg-ocngw.apigwx.com/v1/healthcheck
  ```

- OCN Service Prd JapanWestの場合
  ```
  curl -H "User-Agent: Datadog/Synthetics" https://osaka-pro-ocngw.apigwx.com/v1/healthcheck
  ```

- OCN Service Stg JapanEastの場合
  ```
  curl -H "User-Agent: Datadog/Synthetics" https://tokyo-stg-ocngw.apigwx.com/v1/healthcheck
  ```

- OCN Service Prd JapanEastの場合
  ```
  curl -H "User-Agent: Datadog/Synthetics" https://tokyo-pro-ocngw.apigwx.com/v1/healthcheck
  ```
  ※直し⇒[200 OK]が出力される事

---
## ACL切り戻し手順
---
本切り戻し手順では、追記したIPブロックリストを無効化した後、ACL適用を行うことで切り戻しを行う

- 追記したIPブロックリストを無効化するため、以下のようにstage1/eks.tfに定義する。

```
resource "aws_default_network_acl" "cluster_acl" {
  default_network_acl_id = aws_vpc.cluster_vpc.default_network_acl_id
  subnet_ids = [
    aws_subnet.public_subnet_1.id,
    aws_subnet.public_subnet_2.id,
    aws_subnet.public_subnet_3.id,
    aws_subnet.private_subnet_1.id,
    aws_subnet.private_subnet_2.id,
    aws_subnet.private_subnet_3.id
  ]

  egress {
     action          = "allow"
     cidr_block      = "0.0.0.0/0"
     from_port       = 0
     icmp_code       = 0
     icmp_type        = 0
     protocol        = "-1"
     rule_no         = 100
     to_port         = 0
  }
  ingress {
     action          = "allow"
     cidr_block      = "0.0.0.0/0"
     from_port       = 0
     icmp_code       = 0
     icmp_type        = 0
     protocol        = "-1"
     rule_no         = 100
     to_port         = 0
  }
#  ingress {
#    action           = "deny"
#    cidr_block       = "20.48.18.209/32"
#    from_port        = 0
#    icmp_code        = 0
#    icmp_type        = 0
#    protocol         = "-1"
#    rule_no          = 1
#    to_port          = 0
#  }
#  ingress {
#    action           = "deny"
#    cidr_block       = "20.48.41.226/32"
#    from_port        = 0
#    icmp_code        = 0
#    icmp_type        = 0
#    protocol         = "-1"
#    rule_no          = 2
#    to_port          = 0
#  }
#  ingress {
#    action           = "deny"
#    cidr_block       = "20.78.142.129/32"
#    from_port        = 0
#    icmp_code        = 0
#    icmp_type        = 0
#    protocol         = "-1"
#    rule_no          = 3
#    to_port          = 0
#  }
#  ingress {
#    action           = "deny"
#    cidr_block       = "20.89.244.233/32"
#    from_port        = 0
#    icmp_code        = 0
#    icmp_type        = 0
#    protocol         = "-1"
#    rule_no          = 4
#    to_port          = 0
#  }
#  ingress {
#    action           = "deny"
#    cidr_block       = "20.104.211.17/32"
#    from_port        = 0
#    icmp_code        = 0
#    icmp_type        = 0
#    protocol         = "-1"
#    rule_no          = 5
#    to_port          = 0
#  }
#  ingress {
#    action           = "deny"
#    cidr_block       = "20.151.230.239/32"
#    from_port        = 0
#    icmp_code        = 0
#    icmp_type        = 0
#    protocol         = "-1"
#    rule_no          = 6
#    to_port          = 0
#  }
#  ingress {
#    action           = "deny"
#    cidr_block       = "20.222.104.90/32"
#    from_port        = 0
#    icmp_code        = 0
#    icmp_type        = 0
#    protocol         = "-1"
#    rule_no          = 7
#    to_port          = 0
#  }
#  ingress {
#    action           = "deny"
#    cidr_block       = "20.243.24.205/32"
#    from_port        = 0
#    icmp_code        = 0
#    icmp_type        = 0
#    protocol         = "-1"
#    rule_no          = 8
#    to_port          = 0
#  }
#  ingress {
#    action           = "deny"
#    cidr_block       = "20.243.163.24/32"
#    from_port        = 0
#    icmp_code        = 0
#    icmp_type        = 0
#    protocol         = "-1"
#    rule_no          = 9
#    to_port          = 0
#  }
#  ingress {
#    action           = "deny"
#    cidr_block       = "40.74.64.226/32"
#    from_port        = 0
#    icmp_code        = 0
#    icmp_type        = 0
#    protocol         = "-1"
#    rule_no          = 10
#    to_port          = 0
#  }
#  ingress {
#    action           = "deny"
#    cidr_block       = "45.153.129.75/32"
#    from_port        = 0
#    icmp_code        = 0
#    icmp_type        = 0
#    protocol         = "-1"
#    rule_no          = 11
#    to_port          = 0
#  }
#  ingress {
#    action           = "deny"
#    cidr_block       = "52.156.24.42/32"
#    from_port        = 0
#    icmp_code        = 0
#    icmp_type        = 0
#    protocol         = "-1"
#    rule_no          = 12
#    to_port          = 0
#  }
#  ingress {
#    action           = "deny"
#    cidr_block       = "107.150.112.193/32"
#    from_port        = 0
#    icmp_code        = 0
#    icmp_type        = 0
#    protocol         = "-1"
#    rule_no          = 13
#    to_port          = 0
#  }
#  ingress {
#    action           = "deny"
#    cidr_block       = "152.32.138.156/32"
#    from_port        = 0
#    icmp_code        = 0
#    icmp_type        = 0
#    protocol         = "-1"
#    rule_no          = 14
#    to_port          = 0
#  }
}

```

## ACL適用
- stage1をApplyし、ACLを適用する
  ```
  cicdm terraform -chdir=stage1 apply -var-file="../terraform.tfvars"
  ```
  ⇒[Apply Complate!]が出力される事
  
## 動作確認
-以下コマンドでヘルスチェック先をコールし、問題なく疎通ができることを確認
- OCN Service Stg JapanWestの場合
  ```
  curl -H "User-Agent: Datadog/Synthetics" https://osaka-stg-ocngw.apigwx.com/v1/healthcheck
  ```

- OCN Service Prd JapanWestの場合
  ```
  curl -H "User-Agent: Datadog/Synthetics" https://osaka-pro-ocngw.apigwx.com/v1/healthcheck
  ```

- OCN Service Stg JapanEastの場合
  ```
  curl -H "User-Agent: Datadog/Synthetics" https://tokyo-stg-ocngw.apigwx.com/v1/healthcheck
  ```

- OCN Service Prd JapanEastの場合
  ```
  curl -H "User-Agent: Datadog/Synthetics" https://tokyo-pro-ocngw.apigwx.com/v1/healthcheck
  ```
  ※直し⇒[200 OK]が出力される事
