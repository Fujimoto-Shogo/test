# 【AKS】新CICDモジュール ACL適用手順書
AKSテナントに対して、ACLを取込み・適用する手順を記載致します。

## 目次
- [【AKS】新CICDモジュール ACL適用手順書](#aks新cicdモジュール-acl適用手順書)
  - [目次](#目次)
  - [前提条件](#前提条件)
  - [事前作業](#事前作業)
  - [事前作業(Helm Chart/HybridFiles 用意)](#事前作業helm-charthybridfiles-用意)
  - [直前作業](#直前作業)
  - [デフォルトACL取込み](#デフォルトacl取込み)
    - [aks.tfにACLを定義する](#akstfにaclを定義する)
    - [importを行う](#importを行う)
  - [ACL適用](#acl適用)
  - [事後確認](#事後確認)
  - [Call確認](#Call確認)
  - [ACL切り戻し手順](#ACL切り戻し手順)
    - [aks.tfに追記した記述を無効化したACLを定義する](#akstfに追記した記述を無効化したACLを定義する)
    - [追記した記述を無効化したACL適用](#追記した記述を無効化したacl適用)
    - [動作確認](#動作確認)
    


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
| SmartEducation | apg-096bdef8-c965 | ncom-apigw-across-g01-o20-1176 | japaneast | 20.18.110.227 | |
| SmartEducation | apg-8e7953c5-a1c7 | ncom-apigw-axross-g02-o19-4710 | japaneast | 74.226.234.49 | |

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
| SmartEducation | apg-096bdef8-c965 | ncom-apigw-across-g01-o20-1176 | japaneast | 20.18.110.227 | |
| SmartEducation | apg-8e7953c5-a1c7 | ncom-apigw-axross-g02-o19-4710 | japaneast | 74.226.234.49 | |

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
    tenant_aks_name          = "listの[clusterName]と比較し、対象テナントのCluster名である事"
    ```
---
## デフォルトACL取込み
--- 
既存のACLをTerraformにimportする

◆確認用list
| company | clusterName | ProjectId | region | VM IP | 補足 |
| --- | --- | --- | --- | --- | --- |
| SmartEducation | apg-096bdef8-c965 | ncom-apigw-across-g01-o20-1176 | japaneast | 20.18.110.227 | |
| SmartEducation | apg-8e7953c5-a1c7 | ncom-apigw-axross-g02-o19-4710 | japaneast | 74.226.234.49 | |

### aks.tfにACLを定義する

以下を`stage1/aks.tf`に定義する。  
※最下部で良い
* apg-096bdef8-c965の場合
  ```
  resource "azurerm_network_security_group" "cluster_nsg" {
    name                = "aks-agentpool-29298052-nsg"
    location            = var.tenant_aks_region
    resource_group_name = "mc_apg-096bdef8-c965-rg_apg-096bdef8-c965_japaneast"

    security_rule = [
        {
        access                                     = "Deny"
        destination_address_prefix                 = "*"
        destination_address_prefixes               = []
        destination_application_security_group_ids = []
        destination_port_range                     = "*"
        destination_port_ranges                    = []
        direction                                  = "Inbound"
        name                                       = "DDoS-Protection"
        priority                                   = 100
        protocol                                   = "Tcp"
        source_address_prefix                      = ""
        source_address_prefixes                    = ["13.76.34.194/32","45.117.103.76/32","20.48.92.186/32","45.154.13.55/32","141.164.51.163/32","13.78.66.223/32","141.164.41.32/32","104.215.15.158/32","118.194.233.117/32","52.148.65.100/32","34.97.189.249/32","107.150.122.54/32","137.116.135.19/32","168.63.241.222/32","40.65.178.171/32","52.187.183.61/32","52.187.51.89/32","52.187.61.58/32","152.32.218.27/32","52.237.120.221/32","101.69.113.80/32","128.14.154.215/32","154.85.48.138/32","154.85.50.105/32","194.110.84.59/32","40.115.162.72/32","43.252.230.217/32","52.246.182.141/32","43.245.220.201/32","114.222.73.229/32","156.240.112.229/32","45.76.194.66/32","185.239.226.5/32","18.27.197.252/32","71.19.144.106/32","178.165.72.177/32","185.65.205.10/32","195.176.3.20/32","104.244.78.231/32","176.10.104.240/32","185.220.100.252/32","185.220.101.21/32","185.56.80.65/32","104.41.189.89/32","20.191.190.63/32","20.194.236.67/32","20.194.239.121/32","20.89.17.212/32","118.193.73.192/32","141.98.212.116/32","141.98.212.94/32","128.1.135.182/32","13.76.249.221/32","20.63.217.139/32","45.32.38.238/32","188.233.57.238/32","112.79.109.70/32","60.216.46.77/32","221.181.185.220/32","36.99.192.26/32","218.93.208.150/32","221.131.165.87/32","221.131.165.56/32","222.186.42.213/32","222.187.239.109/32","205.185.117.79/32","206.189.87.108/32","129.226.148.7/32","205.185.119.90/32","221.181.185.198/32","124.67.69.114/32","222.187.232.205/32","222.187.232.10/32","95.110.225.173/32","218.202.140.167/32","95.169.9.45/32","221.181.185.159/32","202.77.105.98/32","221.181.185.135/32","222.187.238.136/32","205.185.126.160/32","195.133.40.104/32","195.133.40.2/32","101.109.113.17/32","222.186.31.166/32","223.31.220.238/32","209.141.42.147/32","105.203.195.68/32","221.181.185.153/32","43.229.203.226/32","45.146.165.72/32","192.241.207.85/32","211.143.253.166/32","205.185.126.8/32","161.35.91.205/32","141.98.10.29/32","106.15.74.124/32","85.74.204.242/32","205.185.125.108/32","67.205.150.68/32","180.63.4.169/32","161.35.80.11/32","49.88.112.109/32","180.97.31.28/32","182.79.44.10/32","202.51.69.41/32","36.85.188.56/32","209.141.62.234/32","120.226.8.180/32","58.186.98.244/32","141.98.10.27/32","124.106.227.89/32","103.23.100.87/32","81.161.63.100/32","65.49.20.69/32","65.49.20.97/32","141.98.10.56/32","171.253.48.147/32","81.249.140.146/32","2.196.9.164/32","142.93.105.220/32","223.207.225.186/32","31.184.198.71/32","107.189.1.161/32","180.76.124.53/32","192.241.210.149/32","139.59.168.22/32","104.248.20.236/32","82.196.5.251/32","192.241.207.83/32","34.134.79.149/32","199.195.248.154/32","185.202.1.42/32","185.202.1.162/32","185.202.1.170/32","193.27.228.9/32","91.241.19.135/32","141.98.10.179/32","107.189.30.221/32","111.161.74.118/32","185.176.222.106/32","220.88.40.41/32","45.134.26.110/32","185.176.222.39/32","185.202.1.175/32","185.202.1.81/32","20.48.18.209/32","20.48.41.226/32","20.78.142.129/32","20.89.244.233/32","20.104.211.17/32","20.151.230.239/32","20.222.104.90/32","20.243.24.205/32","20.243.163.24/32","40.74.64.226/32","45.153.129.75/32","52.156.24.42/32","107.150.112.193/32","152.32.138.156/32"]
        source_application_security_group_ids      = []
        source_port_range                          = "*"
        source_port_ranges                         = []
        description                                = ""
      },
      {
        access                                     = "Allow"
        destination_address_prefix                 = "4.189.21.209"
        destination_address_prefixes               = []
        destination_application_security_group_ids = []
        destination_port_range                     = "15021"
        destination_port_ranges                    = []
        direction                                  = "Inbound"
        name                                       = "acdb3eeb6debe4288ae8bab4d4990ab3-TCP-15021-Internet"
        priority                                   = 500
        protocol                                   = "Tcp"
        source_address_prefix                      = "Internet"
        source_address_prefixes                    = []
        source_application_security_group_ids      = []
        source_port_range                          = "*"
        source_port_ranges                         = []
        description                                = ""
      },
      {
        access                                     = "Allow"
        destination_address_prefix                 = "4.189.21.209"
        destination_address_prefixes               = []
        destination_application_security_group_ids = []
        destination_port_range                     = "443"
        destination_port_ranges                    = []
        direction                                  = "Inbound"
        name                                       = "acdb3eeb6debe4288ae8bab4d4990ab3-TCP-443-Internet"
        priority                                   = 501
        protocol                                   = "Tcp"
        source_address_prefix                      = "Internet"
        source_address_prefixes                    = []
        source_application_security_group_ids      = []
        source_port_range                          = "*"
        source_port_ranges                         = []
        description                                = ""
      },
      {
        access                                     = "Allow"
        destination_address_prefix                 = "20.222.219.94"
        destination_address_prefixes               = []
        destination_application_security_group_ids = []
        destination_port_range                     = "15021"
        destination_port_ranges                    = []
        direction                                  = "Inbound"
        name                                       = "acccc8c5ec0a448d2a046db9d820c383-TCP-15021-Internet"
        priority                                   = 502
        protocol                                   = "Tcp"
        source_address_prefix                      = "Internet"
        source_address_prefixes                    = []
        source_application_security_group_ids      = []
        source_port_range                          = "*"
        source_port_ranges                         = []
        description                                = ""
      },
      {
        access                                     = "Allow"
        destination_address_prefix                 = "20.222.219.94"
        destination_address_prefixes               = []
        destination_application_security_group_ids = []
        destination_port_range                     = "443"
        destination_port_ranges                    = []
        direction                                  = "Inbound"
        name                                       = "acccc8c5ec0a448d2a046db9d820c383-TCP-443-Internet"
        priority                                   = 503
        protocol                                   = "Tcp"
        source_address_prefix                      = "Internet"
        source_address_prefixes                    = []
        source_application_security_group_ids      = []
        source_port_range                          = "*"
        source_port_ranges                         = []
        description                                = ""
      }
    ]
    tags                = {}
  }
  ```
* apg-8e7953c5-a1c7の場合
  ```
  resource "azurerm_network_security_group" "cluster_nsg" {
    name                = "aks-agentpool-41228824-nsg"
    location            = var.tenant_aks_region
    resource_group_name = "mc_apg-8e7953c5-a1c7-rg_apg-8e7953c5-a1c7_japaneast"

    security_rule = [
        {
        access                                     = "Deny"
        destination_address_prefix                 = "*"
        destination_address_prefixes               = []
        destination_application_security_group_ids = []
        destination_port_range                     = "*"
        destination_port_ranges                    = []
        direction                                  = "Inbound"
        name                                       = "DDoS-Protection"
        priority                                   = 100
        protocol                                   = "Tcp"
        source_address_prefix                      = ""
        source_address_prefixes                    = ["13.76.34.194/32","45.117.103.76/32","20.48.92.186/32","45.154.13.55/32","141.164.51.163/32","13.78.66.223/32","141.164.41.32/32","104.215.15.158/32","118.194.233.117/32","52.148.65.100/32","34.97.189.249/32","107.150.122.54/32","137.116.135.19/32","168.63.241.222/32","40.65.178.171/32","52.187.183.61/32","52.187.51.89/32","52.187.61.58/32","152.32.218.27/32","52.237.120.221/32","101.69.113.80/32","128.14.154.215/32","154.85.48.138/32","154.85.50.105/32","194.110.84.59/32","40.115.162.72/32","43.252.230.217/32","52.246.182.141/32","43.245.220.201/32","114.222.73.229/32","156.240.112.229/32","45.76.194.66/32","185.239.226.5/32","18.27.197.252/32","71.19.144.106/32","178.165.72.177/32","185.65.205.10/32","195.176.3.20/32","104.244.78.231/32","176.10.104.240/32","185.220.100.252/32","185.220.101.21/32","185.56.80.65/32","104.41.189.89/32","20.191.190.63/32","20.194.236.67/32","20.194.239.121/32","20.89.17.212/32","118.193.73.192/32","141.98.212.116/32","141.98.212.94/32","128.1.135.182/32","13.76.249.221/32","20.63.217.139/32","45.32.38.238/32","188.233.57.238/32","112.79.109.70/32","60.216.46.77/32","221.181.185.220/32","36.99.192.26/32","218.93.208.150/32","221.131.165.87/32","221.131.165.56/32","222.186.42.213/32","222.187.239.109/32","205.185.117.79/32","206.189.87.108/32","129.226.148.7/32","205.185.119.90/32","221.181.185.198/32","124.67.69.114/32","222.187.232.205/32","222.187.232.10/32","95.110.225.173/32","218.202.140.167/32","95.169.9.45/32","221.181.185.159/32","202.77.105.98/32","221.181.185.135/32","222.187.238.136/32","205.185.126.160/32","195.133.40.104/32","195.133.40.2/32","101.109.113.17/32","222.186.31.166/32","223.31.220.238/32","209.141.42.147/32","105.203.195.68/32","221.181.185.153/32","43.229.203.226/32","45.146.165.72/32","192.241.207.85/32","211.143.253.166/32","205.185.126.8/32","161.35.91.205/32","141.98.10.29/32","106.15.74.124/32","85.74.204.242/32","205.185.125.108/32","67.205.150.68/32","180.63.4.169/32","161.35.80.11/32","49.88.112.109/32","180.97.31.28/32","182.79.44.10/32","202.51.69.41/32","36.85.188.56/32","209.141.62.234/32","120.226.8.180/32","58.186.98.244/32","141.98.10.27/32","124.106.227.89/32","103.23.100.87/32","81.161.63.100/32","65.49.20.69/32","65.49.20.97/32","141.98.10.56/32","171.253.48.147/32","81.249.140.146/32","2.196.9.164/32","142.93.105.220/32","223.207.225.186/32","31.184.198.71/32","107.189.1.161/32","180.76.124.53/32","192.241.210.149/32","139.59.168.22/32","104.248.20.236/32","82.196.5.251/32","192.241.207.83/32","34.134.79.149/32","199.195.248.154/32","185.202.1.42/32","185.202.1.162/32","185.202.1.170/32","193.27.228.9/32","91.241.19.135/32","141.98.10.179/32","107.189.30.221/32","111.161.74.118/32","185.176.222.106/32","220.88.40.41/32","45.134.26.110/32","185.176.222.39/32","185.202.1.175/32","185.202.1.81/32","20.48.18.209/32","20.48.41.226/32","20.78.142.129/32","20.89.244.233/32","20.104.211.17/32","20.151.230.239/32","20.222.104.90/32","20.243.24.205/32","20.243.163.24/32","40.74.64.226/32","45.153.129.75/32","52.156.24.42/32","107.150.112.193/32","152.32.138.156/32"]
        source_application_security_group_ids      = []
        source_port_range                          = "*"
        source_port_ranges                         = []
        description                                = ""
      },
      {
        access                                     = "Allow"
        destination_address_prefix                 = "20.210.14.38"
        destination_address_prefixes               = []
        destination_application_security_group_ids = []
        destination_port_range                     = "15021"
        destination_port_ranges                    = []
        direction                                  = "Inbound"
        name                                       = "aee4728e66a42463c9e876c16c584dce-TCP-15021-Internet"
        priority                                   = 500
        protocol                                   = "Tcp"
        source_address_prefix                      = "Internet"
        source_address_prefixes                    = []
        source_application_security_group_ids      = []
        source_port_range                          = "*"
        source_port_ranges                         = []
        description                                = ""
      },
      {
        access                                     = "Allow"
        destination_address_prefix                 = "20.210.14.38"
        destination_address_prefixes               = []
        destination_application_security_group_ids = []
        destination_port_range                     = "443"
        destination_port_ranges                    = []
        direction                                  = "Inbound"
        name                                       = "aee4728e66a42463c9e876c16c584dce-TCP-443-Internet"
        priority                                   = 501
        protocol                                   = "Tcp"
        source_address_prefix                      = "Internet"
        source_address_prefixes                    = []
        source_application_security_group_ids      = []
        source_port_range                          = "*"
        source_port_ranges                         = []
        description                                = ""
      },
      {
        access                                     = "Allow"
        destination_address_prefix                 = "4.241.157.85"
        destination_address_prefixes               = []
        destination_application_security_group_ids = []
        destination_port_range                     = "15021"
        destination_port_ranges                    = []
        direction                                  = "Inbound"
        name                                       = "a2c76cddcdc0d4e3096dfb8db5b7cd92-TCP-15021-Internet"
        priority                                   = 502
        protocol                                   = "Tcp"
        source_address_prefix                      = "Internet"
        source_address_prefixes                    = []
        source_application_security_group_ids      = []
        source_port_range                          = "*"
        source_port_ranges                         = []
        description                                = ""
      },
      {
        access                                     = "Allow"
        destination_address_prefix                 = "4.241.157.85"
        destination_address_prefixes               = []
        destination_application_security_group_ids = []
        destination_port_range                     = "443"
        destination_port_ranges                    = []
        direction                                  = "Inbound"
        name                                       = "a2c76cddcdc0d4e3096dfb8db5b7cd92-TCP-443-Internet"
        priority                                   = 503
        protocol                                   = "Tcp"
        source_address_prefix                      = "Internet"
        source_address_prefixes                    = []
        source_application_security_group_ids      = []
        source_port_range                          = "*"
        source_port_ranges                         = []
        description                                = ""
      }
    ]
    tags                = {}
  }
  ```
### importを行う

- apg-096bdef8-c965の場合
  ```
  cicdm terraform -chdir=stage1 import -var-file=../terraform.tfvars azurerm_network_security_group.cluster_nsg /subscriptions/8427c6ec-b392-42b5-a072-181fb9ce57b5/resourceGroups/mc_apg-096bdef8-c965-rg_apg-096bdef8-c965_japaneast/providers/Microsoft.Network/networkSecurityGroups/aks-agentpool-29298052-nsg
  ```

- apg-8e7953c5-a1c7の場合
  ```
  cicdm terraform -chdir=stage1 import -var-file=../terraform.tfvars azurerm_network_security_group.cluster_nsg /subscriptions/d1d6f35d-2d56-4a9a-a42d-ec28d70a5204/resourceGroups/mc_apg-8e7953c5-a1c7-rg_apg-8e7953c5-a1c7_japaneast/providers/Microsoft.Network/networkSecurityGroups/aks-agentpool-41228824-nsg
  ```
  ⇒[Import successful!]が出力される事

## ACL適用
- stage1をApplyし、ACLを適用する
  ```
  cicdm terraform -chdir=stage1 apply -var-file="../terraform.tfvars"
  ```
  ```
  出力例：
    # azurerm_network_security_group.cluster_nsg will be updated in-place
    ~ resource "azurerm_network_security_group" "cluster_nsg" {
          id                  = "/subscriptions/4aa9fa49-3798-44ca-9cf0-edfa83f0cef0/resourceGroups/mc_qax-5f1af200-3f80-rg_qax-5f1af200-3f80_japaneast/providers/Microsoft.Network/networkSecurityGroups/aks-agentpool-64424924-nsg"
          name                = "aks-agentpool-64424924-nsg"
        ~ security_rule       = [
            + {
                + access                                     = "Deny"
                + destination_address_prefix                 = "*"
                + destination_address_prefixes               = []
                + destination_application_security_group_ids = []
                + destination_port_range                     = "*"
                + destination_port_ranges                    = []
                + direction                                  = "Inbound"
                + name                                       = "DDoS-Protection"
                + priority                                   = 100
                + protocol                                   = "Tcp"
                + source_address_prefixes                    = [
                    + "101.109.113.17/32",
                    ~~~~~~~  skip ~~~~~~
                    + "95.169.9.45/32",
                  ]
                + source_application_security_group_ids      = []
                + source_port_range                          = "*"
                + source_port_ranges                         = []
              },
          ]
          tags                = {}
      }
  ```
  ※ DDoS-Protection""以外""の変更がある場合は、tfファイル側を変更して差分を埋めること。  
  ⇒[Apply Complate!]が出力される事

## 事後確認
- apg-096bdef8-c965の場合
  ```
  az network nsg rule list -o table -g mc_apg-096bdef8-c965-rg_apg-096bdef8-c965_japaneast --nsg-name aks-agentpool-29298052-nsg
  ```

- apg-8e7953c5-a1c7の場合
  ```
  az network nsg rule list -o table -g mc_apg-8e7953c5-a1c7-rg_apg-8e7953c5-a1c7_japaneast --nsg-name aks-agentpool-41228824-nsg
  ```

⇒想定通りのENTRIESが出力されること。(RuleID:100)
```
DDoS-Protection  MC_qax-5f1af200-3f80-rg_qax-5f1af200-3f80_japaneast  100   *   141.98.10.27/32~36.99.192.26/32  None  Deny  Tcp  Inbound  443  20.78.117.40  None
```

## Call確認
以下コマンドでヘルスチェック先をコールし、問題なく疎通ができることを確認
- apg-096bdef8-c965の場合
  ```
  curl -H "User-Agent: Datadog/Synthetics" https://manabipocket.apigwx.com/v1/healthcheck
  ```

- apg-8e7953c5-a1c7の場合
  ```
  curl -H "User-Agent: Datadog/Synthetics" https://manapoke.apigwx.com/v1/healthcheck
  ```
  ⇒["healthcheck": "true"]が出力される事
  
---
## ACL切り戻し手順
---
本切り戻し手順では、追記したIPブロックリストを無効化した後、ACL適用を行うことで切り戻しを行う

### aks.tfに追記した記述を無効化したACLを定義する
追記したIPブロックリストを無効化するため、以下のようにstage1/aks.tfに定義する。

* apg-096bdef8-c965の場合
```
  resource "azurerm_network_security_group" "cluster_nsg" {
    name                = "aks-agentpool-29298052-nsg"
    location            = var.tenant_aks_region
    resource_group_name = "mc_apg-096bdef8-c965-rg_apg-096bdef8-c965_japaneast"

    security_rule = [
#     {
#       access                                     = "Deny"
#       destination_address_prefix                 = "*"
#       destination_address_prefixes               = []
#       destination_application_security_group_ids = []
#       destination_port_range                     = "*"
#       destination_port_ranges                    = []
#       direction                                  = "Inbound"
#       name                                       = "DDoS-Protection"
#       priority                                   = 100
#       protocol                                   = "Tcp"
#       source_address_prefix                      = ""
#       source_address_prefixes                    = ["13.76.34.194/32","45.117.103.76/32","20.48.92.186/32","45.154.13.55/32","141.164.51.163/32","13.78.66.223/32","141.164.41.32/32","104.215.15.158/32","118.194.233.117/32","52.148.65.100/32","34.97.189.249/32","107.150.122.54/32","137.116.135.19/32","168.63.241.222/32","40.65.178.171/32","52.187.183.61/32","52.187.51.89/32","52.187.61.58/32","152.32.218.27/32","52.237.120.221/32","101.69.113.80/32","128.14.154.215/32","154.85.48.138/32","154.85.50.105/32","194.110.84.59/32","40.115.162.72/32","43.252.230.217/32","52.246.182.141/32","43.245.220.201/32","114.222.73.229/32","156.240.112.229/32","45.76.194.66/32","185.239.226.5/32","18.27.197.252/32","71.19.144.106/32","178.165.72.177/32","185.65.205.10/32","195.176.3.20/32","104.244.78.231/32","176.10.104.240/32","185.220.100.252/32","185.220.101.21/32","185.56.80.65/32","104.41.189.89/32","20.191.190.63/32","20.194.236.67/32","20.194.239.121/32","20.89.17.212/32","118.193.73.192/32","141.98.212.116/32","141.98.212.94/32","128.1.135.182/32","13.76.249.221/32","20.63.217.139/32","45.32.38.238/32","188.233.57.238/32","112.79.109.70/32","60.216.46.77/32","221.181.185.220/32","36.99.192.26/32","218.93.208.150/32","221.131.165.87/32","221.131.165.56/32","222.186.42.213/32","222.187.239.109/32","205.185.117.79/32","206.189.87.108/32","129.226.148.7/32","205.185.119.90/32","221.181.185.198/32","124.67.69.114/32","222.187.232.205/32","222.187.232.10/32","95.110.225.173/32","218.202.140.167/32","95.169.9.45/32","221.181.185.159/32","202.77.105.98/32","221.181.185.135/32","222.187.238.136/32","205.185.126.160/32","195.133.40.104/32","195.133.40.2/32","101.109.113.17/32","222.186.31.166/32","223.31.220.238/32","209.141.42.147/32","105.203.195.68/32","221.181.185.153/32","43.229.203.226/32","45.146.165.72/32","192.241.207.85/32","211.143.253.166/32","205.185.126.8/32","161.35.91.205/32","141.98.10.29/32","106.15.74.124/32","85.74.204.242/32","205.185.125.108/32","67.205.150.68/32","180.63.4.169/32","161.35.80.11/32","49.88.112.109/32","180.97.31.28/32","182.79.44.10/32","202.51.69.41/32","36.85.188.56/32","209.141.62.234/32","120.226.8.180/32","58.186.98.244/32","141.98.10.27/32","124.106.227.89/32","103.23.100.87/32","81.161.63.100/32","65.49.20.69/32","65.49.20.97/32","141.98.10.56/32","171.253.48.147/32","81.249.140.146/32","2.196.9.164/32","142.93.105.220/32","223.207.225.186/32","31.184.198.71/32","107.189.1.161/32","180.76.124.53/32","192.241.210.149/32","139.59.168.22/32","104.248.20.236/32","82.196.5.251/32","192.241.207.83/32","34.134.79.149/32","199.195.248.154/32","185.202.1.42/32","185.202.1.162/32","185.202.1.170/32","193.27.228.9/32","91.241.19.135/32","141.98.10.179/32","107.189.30.221/32","111.161.74.118/32","185.176.222.106/32","220.88.40.41/32","45.134.26.110/32","185.176.222.39/32","185.202.1.175/32","185.202.1.81/32","20.48.18.209/32","20.48.41.226/32","20.78.142.129/32","20.89.244.233/32","20.104.211.17/32","20.151.230.239/32","20.222.104.90/32","20.243.24.205/32","20.243.163.24/32","40.74.64.226/32","45.153.129.75/32","52.156.24.42/32","107.150.112.193/32","152.32.138.156/32"]
#       source_application_security_group_ids      = []
#       source_port_range                          = "*"
#       source_port_ranges                         = []
#       description                                = ""
#     },
      {
        access                                     = "Allow"
        destination_address_prefix                 = "20.210.14.38"
        destination_address_prefixes               = []
        destination_application_security_group_ids = []
        destination_port_range                     = "15021"
        destination_port_ranges                    = []
        direction                                  = "Inbound"
        name                                       = "aee4728e66a42463c9e876c16c584dce-TCP-15021-Internet"
        priority                                   = 500
        protocol                                   = "Tcp"
        source_address_prefix                      = "Internet"
        source_address_prefixes                    = []
        source_application_security_group_ids      = []
        source_port_range                          = "*"
        source_port_ranges                         = []
        description                                = ""
      },
      {
        access                                     = "Allow"
        destination_address_prefix                 = "20.210.14.38"
        destination_address_prefixes               = []
        destination_application_security_group_ids = []
        destination_port_range                     = "443"
        destination_port_ranges                    = []
        direction                                  = "Inbound"
        name                                       = "aee4728e66a42463c9e876c16c584dce-TCP-443-Internet"
        priority                                   = 501
        protocol                                   = "Tcp"
        source_address_prefix                      = "Internet"
        source_address_prefixes                    = []
        source_application_security_group_ids      = []
        source_port_range                          = "*"
        source_port_ranges                         = []
        description                                = ""
      },
      {
        access                                     = "Allow"
        destination_address_prefix                 = "4.241.157.85"
        destination_address_prefixes               = []
        destination_application_security_group_ids = []
        destination_port_range                     = "15021"
        destination_port_ranges                    = []
        direction                                  = "Inbound"
        name                                       = "a2c76cddcdc0d4e3096dfb8db5b7cd92-TCP-15021-Internet"
        priority                                   = 502
        protocol                                   = "Tcp"
        source_address_prefix                      = "Internet"
        source_address_prefixes                    = []
        source_application_security_group_ids      = []
        source_port_range                          = "*"
        source_port_ranges                         = []
        description                                = ""
      },
      {
        access                                     = "Allow"
        destination_address_prefix                 = "4.241.157.85"
        destination_address_prefixes               = []
        destination_application_security_group_ids = []
        destination_port_range                     = "443"
        destination_port_ranges                    = []
        direction                                  = "Inbound"
        name                                       = "a2c76cddcdc0d4e3096dfb8db5b7cd92-TCP-443-Internet"
        priority                                   = 503
        protocol                                   = "Tcp"
        source_address_prefix                      = "Internet"
        source_address_prefixes                    = []
        source_application_security_group_ids      = []
        source_port_range                          = "*"
        source_port_ranges                         = []
        description                                = ""
      }
    ]
    tags                = {}
  }
  
```

* apg-8e7953c5-a1c7の場合
```
  resource "azurerm_network_security_group" "cluster_nsg" {
    name                = "aks-agentpool-41228824-nsg"
    location            = var.tenant_aks_region
    resource_group_name = "mc_apg-8e7953c5-a1c7-rg_apg-8e7953c5-a1c7_japaneast"

    security_rule = [
#     {
#       access                                     = "Deny"
#       destination_address_prefix                 = "*"
#       destination_address_prefixes               = []
#       destination_application_security_group_ids = []
#       destination_port_range                     = "*"
#       destination_port_ranges                    = []
#       direction                                  = "Inbound"
#       name                                       = "DDoS-Protection"
#       priority                                   = 100
#       protocol                                   = "Tcp"
#       source_address_prefix                      = ""
#       source_address_prefixes                    = ["13.76.34.194/32","45.117.103.76/32","20.48.92.186/32","45.154.13.55/32","141.164.51.163/32","13.78.66.223/32","141.164.41.32/32","104.215.15.158/32","118.194.233.117/32","52.148.65.100/32","34.97.189.249/32","107.150.122.54/32","137.116.135.19/32","168.63.241.222/32","40.65.178.171/32","52.187.183.61/32","52.187.51.89/32","52.187.61.58/32","152.32.218.27/32","52.237.120.221/32","101.69.113.80/32","128.14.154.215/32","154.85.48.138/32","154.85.50.105/32","194.110.84.59/32","40.115.162.72/32","43.252.230.217/32","52.246.182.141/32","43.245.220.201/32","114.222.73.229/32","156.240.112.229/32","45.76.194.66/32","185.239.226.5/32","18.27.197.252/32","71.19.144.106/32","178.165.72.177/32","185.65.205.10/32","195.176.3.20/32","104.244.78.231/32","176.10.104.240/32","185.220.100.252/32","185.220.101.21/32","185.56.80.65/32","104.41.189.89/32","20.191.190.63/32","20.194.236.67/32","20.194.239.121/32","20.89.17.212/32","118.193.73.192/32","141.98.212.116/32","141.98.212.94/32","128.1.135.182/32","13.76.249.221/32","20.63.217.139/32","45.32.38.238/32","188.233.57.238/32","112.79.109.70/32","60.216.46.77/32","221.181.185.220/32","36.99.192.26/32","218.93.208.150/32","221.131.165.87/32","221.131.165.56/32","222.186.42.213/32","222.187.239.109/32","205.185.117.79/32","206.189.87.108/32","129.226.148.7/32","205.185.119.90/32","221.181.185.198/32","124.67.69.114/32","222.187.232.205/32","222.187.232.10/32","95.110.225.173/32","218.202.140.167/32","95.169.9.45/32","221.181.185.159/32","202.77.105.98/32","221.181.185.135/32","222.187.238.136/32","205.185.126.160/32","195.133.40.104/32","195.133.40.2/32","101.109.113.17/32","222.186.31.166/32","223.31.220.238/32","209.141.42.147/32","105.203.195.68/32","221.181.185.153/32","43.229.203.226/32","45.146.165.72/32","192.241.207.85/32","211.143.253.166/32","205.185.126.8/32","161.35.91.205/32","141.98.10.29/32","106.15.74.124/32","85.74.204.242/32","205.185.125.108/32","67.205.150.68/32","180.63.4.169/32","161.35.80.11/32","49.88.112.109/32","180.97.31.28/32","182.79.44.10/32","202.51.69.41/32","36.85.188.56/32","209.141.62.234/32","120.226.8.180/32","58.186.98.244/32","141.98.10.27/32","124.106.227.89/32","103.23.100.87/32","81.161.63.100/32","65.49.20.69/32","65.49.20.97/32","141.98.10.56/32","171.253.48.147/32","81.249.140.146/32","2.196.9.164/32","142.93.105.220/32","223.207.225.186/32","31.184.198.71/32","107.189.1.161/32","180.76.124.53/32","192.241.210.149/32","139.59.168.22/32","104.248.20.236/32","82.196.5.251/32","192.241.207.83/32","34.134.79.149/32","199.195.248.154/32","185.202.1.42/32","185.202.1.162/32","185.202.1.170/32","193.27.228.9/32","91.241.19.135/32","141.98.10.179/32","107.189.30.221/32","111.161.74.118/32","185.176.222.106/32","220.88.40.41/32","45.134.26.110/32","185.176.222.39/32","185.202.1.175/32","185.202.1.81/32","20.48.18.209/32","20.48.41.226/32","20.78.142.129/32","20.89.244.233/32","20.104.211.17/32","20.151.230.239/32","20.222.104.90/32","20.243.24.205/32","20.243.163.24/32","40.74.64.226/32","45.153.129.75/32","52.156.24.42/32","107.150.112.193/32","152.32.138.156/32"]
#       source_application_security_group_ids      = []
#       source_port_range                          = "*"
#       source_port_ranges                         = []
#       description                                = ""
#     },
      {
        access                                     = "Allow"
        destination_address_prefix                 = "20.210.14.38"
        destination_address_prefixes               = []
        destination_application_security_group_ids = []
        destination_port_range                     = "15021"
        destination_port_ranges                    = []
        direction                                  = "Inbound"
        name                                       = "aee4728e66a42463c9e876c16c584dce-TCP-15021-Internet"
        priority                                   = 500
        protocol                                   = "Tcp"
        source_address_prefix                      = "Internet"
        source_address_prefixes                    = []
        source_application_security_group_ids      = []
        source_port_range                          = "*"
        source_port_ranges                         = []
        description                                = ""
      },
      {
        access                                     = "Allow"
        destination_address_prefix                 = "20.210.14.38"
        destination_address_prefixes               = []
        destination_application_security_group_ids = []
        destination_port_range                     = "443"
        destination_port_ranges                    = []
        direction                                  = "Inbound"
        name                                       = "aee4728e66a42463c9e876c16c584dce-TCP-443-Internet"
        priority                                   = 501
        protocol                                   = "Tcp"
        source_address_prefix                      = "Internet"
        source_address_prefixes                    = []
        source_application_security_group_ids      = []
        source_port_range                          = "*"
        source_port_ranges                         = []
        description                                = ""
      },
      {
        access                                     = "Allow"
        destination_address_prefix                 = "4.241.157.85"
        destination_address_prefixes               = []
        destination_application_security_group_ids = []
        destination_port_range                     = "15021"
        destination_port_ranges                    = []
        direction                                  = "Inbound"
        name                                       = "a2c76cddcdc0d4e3096dfb8db5b7cd92-TCP-15021-Internet"
        priority                                   = 502
        protocol                                   = "Tcp"
        source_address_prefix                      = "Internet"
        source_address_prefixes                    = []
        source_application_security_group_ids      = []
        source_port_range                          = "*"
        source_port_ranges                         = []
        description                                = ""
      },
      {
        access                                     = "Allow"
        destination_address_prefix                 = "4.241.157.85"
        destination_address_prefixes               = []
        destination_application_security_group_ids = []
        destination_port_range                     = "443"
        destination_port_ranges                    = []
        direction                                  = "Inbound"
        name                                       = "a2c76cddcdc0d4e3096dfb8db5b7cd92-TCP-443-Internet"
        priority                                   = 503
        protocol                                   = "Tcp"
        source_address_prefix                      = "Internet"
        source_address_prefixes                    = []
        source_application_security_group_ids      = []
        source_port_range                          = "*"
        source_port_ranges                         = []
        description                                = ""
      }
    ]
    tags                = {}
  }
  ```

## 追記した記述を無効化したACL適用
- stage1をApplyし、ACLを適用する
  ```
  cicdm terraform -chdir=stage1 apply -var-file="../terraform.tfvars"
  ```
  ⇒[Apply Complate!]が出力される事
  
## 動作確認
以下コマンドでヘルスチェック先をコールし、問題なく疎通ができることを確認
- apg-096bdef8-c965の場合
  ```
  curl -H "User-Agent: Datadog/Synthetics" https://manabipocket.apigwx.com/v1/healthcheck
  ```

- apg-8e7953c5-a1c7の場合
  ```
  curl -H "User-Agent: Datadog/Synthetics" https://manapoke.apigwx.com/v1/healthcheck
  ```
  ⇒["healthcheck": "true"]が出力される事
  
