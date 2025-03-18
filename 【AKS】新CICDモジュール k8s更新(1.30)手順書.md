# 【AKS】新CICDモジュール k8s更新(1.30)手順書
AKSテナントにおける、CICD Moduleを利用したk8s 1.30の更新手順について記述する。

##  目次
- [【AKS】新CICDモジュール k8s更新(1.30)手順書](#aks新cicdモジュール-k8s更新130手順書)
  - [目次](#目次)
  - [前提条件](#前提条件)
  - [事前作業](#事前作業)
  - [直前作業](#直前作業)
  - [AKS VersionUp (1.30)](#aks新cicdモジュール-k8s更新130手順)
  - [LTS取り消し手順](#LTS取り消し手順)

---
## 前提条件
---

- GCP Project IDに対して必要な権限が付与されたSAのjson keyが用意されている事  
- 必要となるクラウド(GCP/Azure)のクレデンシャル情報を把握している事
- Dockerコンテナが動作する作業環境が用意されている事
- 新CICDモジュールのcontainerイメージがローカルに存在する事
- CICDモジュールが、k8s 1.30対応 version になっている事
- apigee hybrid がk8s対象 version となっている事
  - AKS1.30 対応apigee hybrid versionは[1.13.x]以上

---
## 事前作業
---
| company | clusterName | ProjectId | region | VM IP | 補足 |
| --- | --- | --- | --- | --- | --- |
| SmartEducation | apg-096bdef8-c965 | ncom-apigw-across-g01-o20-1176 | japaneast | 20.18.110.227 | |
| SmartEducation | apg-8e7953c5-a1c7 | ncom-apigw-axross-g02-o19-4710 | japaneast | 74.226.234.49 | |

* 対象vmにテナント資材を用意する
  * terraform template一式の配置(tfvars,stage1/\*,stage2/\*)  
    対象テナントのterraform templateファイル一式をgitより配置する   
    以下の様に配置される事
    ```
    pwd$ls
    /tmp/<clusterName>
    stage1 stage2 terraform.tfvars
    ```  

  * Service Account Keyを配置する  
    以下の様な形になるようにGCPのSA keyを配置する
    ```
    pwd & ls
    /tmp/<clusterName>
    stage1 stage2 terraform.tfvars
    cicd-test-sa.json
    ```  

  * hybrid-files(overrides/certs/service-accounts)一式を、作業ディレクトリ配下に配置する  
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

* cicdモジュール利用準備  
  *  Azureへログイン
     ```
     az login --service-principal --username <azureClientId> --tenant <AZURE_TENANT_ID> --password <azureClientSecret>
     ```
  *  ACRへログイン
     ```
     az acr login -n cicdmodule
     ```
  *  ACRからdockerをPullする
     ```
     docker pull cicdmodule.azurecr.io/cicd-module:release-202403
     ```

  * alias設定  
    cicdmコマンドを使用するため、以下コマンドでaliasを設定する事
    ```
    alias cicdm='sudo docker run --rm -it -e TF_TOKEN_app_terraform_io=$TF_TOKEN -v `pwd`:/go/work -w /go/work cicdmodule.azurecr.io/cicd-module:release-202403'
    ```

* Terraform Cloudの再initializeを行う
  * stage1のinitializeを行う
    ```
    cicdm terraform -chdir=stage1 init -upgrade
    ```
    「Terraform Cloud has been successfully initialized!」と出力される事
  * stage2のinitializeを行う
    ```
    cicdm terraform -chdir=stage2 init -upgrade
    ```
    「Terraform Cloud has been successfully initialized!」と出力される事  

* 作業環境セットアップ  
  * 以下コマンドがインストールされている事  
    * gcloud  
    * az
    * kubectl  
    * git  
    * screen
  * 対象クラスタへログイン後、以下コマンドにてK8Sのリソース参照が出来る事
    ```
    Kubectl get pod -n apigee
    ```
---
## 直前作業 
--- 
| company | clusterName | ProjectId | region | VM IP | 補足 |
| --- | --- | --- | --- | --- | --- |
| SmartEducation | apg-096bdef8-c965 | ncom-apigw-across-g01-o20-1176 | japaneast | 20.18.110.227 | |
| SmartEducation | apg-8e7953c5-a1c7 | ncom-apigw-axross-g02-o19-4710 | japaneast | 74.226.234.49 | |


- 対象クラスタのDatadog監視をMuteにする
  - datadogのWebUI上から、対象テナントのMonitors監視をMuteにする  
    [Monitors] -> [Manage Monitors]から、対象テナントの監視を選択し、画面右上からMuteを選択

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
    export CICDMODULE_HOME=$PWD
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
  - kubectl  
    以下コマンドを実行し、対象のClusterに接続されているか確認する事  
    ※Listの[clusterName]を確認する
    ```
    kubectl config current-context
    ```
  - cicdモジュール  
    ./terraform.tfvarsの内容を確認し、対象テナントの構成ファイルであるか確認する  
    ```
    cat ./terraform.tfvars
    gcp_project_name         = "listの[ProjectId]と比較し、対象テナントのGcpIdである事"
    tenant_aks_name          = "listの[clusterName]と比較し、対象テナントのCluster名である事"
    ```
---
## AKS VersionUp (1.30)
---
◆確認用list
| company | clusterName | ProjectId | region | VM IP | 補足 |
| --- | --- | --- | --- | --- | --- |
| SmartEducation | apg-096bdef8-c965 | ncom-apigw-across-g01-o20-1176 | japaneast | 20.63.131.167 | |
| SmartEducation | apg-8e7953c5-a1c7 | ncom-apigw-axross-g02-o19-4710 | japaneast | 20.194.203.155 | |


* terraform.tfvars編集  
  terraform.tfvarsファイル内の`tenant_aks_version`の値を1.27.9から1.30.9へ書き換える。
  ```
  tenant_aks_version = "1.27.9"
  ↓
  tenant_aks_version = "1.30.9"
  ```
  ※1.30.9が利用可能Versionか確認しておく
  ```
  az aks get-upgrades -g <clusterName>-rg -n <clusterName> -o table
  ```
* k8s更新(1.27→1.30)   
 　個別テナントのk8sバージョンを更新する
  * terraform apply  
    stage1に対し、terraform applyを実行する  
    ※実行時にk8sのバージョンのみが更新の対象となってることを確認する。
    ```bash
    cicdm terraform -chdir=stage1 apply -var-file="../terraform.tfvars"
    ```
    
    ```
    出力例：
    # azurerm_kubernetes_cluster.cluster will be updated in-place
    ~ resource "azurerm_kubernetes_cluster" "cluster" {
          id                                  = "/subscriptions/xx
        ~ kubernetes_version                  = "1.27" -> "1.30"
          name                                = "xxxx"


        ~ default_node_pool {
              name                         = "runtime"
            ~ orchestrator_version         = "1.27" -> "1.30"
          }
    # azurerm_kubernetes_cluster_node_pool.cluster_data_node will be updated in-place
    ~ resource "azurerm_kubernetes_cluster_node_pool" "cluster_data_node" {
          id                      = "/subscriptions/xx"
          name                    = "data"
        ~ orchestrator_version    = "1.27" -> "1.30"
      }
    ```
    ⇒[Apply Complate]が出力される事
    
* 更新後確認(1.27→1.30)
  * k8s version更新確認  
    下記コマンドを実行し、KubernetesVersionが```1.30.9```になっている事を確認。
    
    ```
    az aks nodepool get-upgrades --resource-group aks-test20240610-resources-group --cluster-name aks-test20240610 --nodepool-name runtime --output table The behavior of this command has been altered by the following extension: aks-preview
    ```
    
    ```
    az aks nodepool get-upgrades --resource-group aks-test20240610-resources-group --cluster-name aks-test20240610 --nodepool-name data --output table The behavior of this command has been altered by the following extension: aks-preview
    ```
    
　　下記コマンドを実行し、K8versionが```1.30```になっている事を確認。
   ```
  az aks nodepool list --resource-group aks-test20240610-resources-group --cluster-name aks-test20240610 --query "[].{Name:name,k8version:orchestratorVersion}" --output table
   ```

  下記コマンドを実行し、LTSが外れていないことを確認。  
  ※```"AKSLongTermSupport"```になっていることを確認　　
  
  ```
  az aks show --name aks-test20240610 -g aks-test20240610-resources-group --query supportPlan
　The behavior of this command has been altered by the following extension: aks-preview
　```

　※```"Premium"```になっていることを確認

　```
　az aks show --name aks-test20240610 -g aks-test20240610-resources-group --query sku.tier
　The behavior of this command has been altered by the following extension: aks-preview
　```

  * k8s ノードversion更新確認
    下記コマンドを実施し、VERSIONが```1.30.9```になっている事を確認。
    ```
    kubectl get nodes
    ```
  * apigee hybrid k8sリソース確認
    下記コマンドを実行し、各種k8sリソース稼働状況に問題が無い事  
    ※STATUS : Completed or Runningであること  
    ※READY :  Pod内のすべてのコンテナが立ち上がっている事 (1/1,2/2 等)
    ```
    kubectl get all -n apigee
    ```
  * ポッドの動作確認
    下記コマンドを実行し、k8sのポッドが正常に動作している事  
    ※STATUS : Completed or Runningであること  
    ※READY :  Pod内のすべてのコンテナが立ち上がっている事 (1/1,2/2 等)
    ```
    kubectl get pods -n apigee
    ```

* 対象クラスタのDatadog監視をUnmuteにする
  * datadogのWebUI上から、対象テナントのMonitors監視を Unmuteする
    [Monitors] -> [Manage Monitors]から、対象テナントの監視を選択し、画面右上からunmuteを選択

* screenコマンドでセッションを解除する
  [Ctrl]+[a] [d] でデタッチを実行する。  
  ※切り戻しが発生した場合は、切り戻し後にscreenセッションを解除する。


##  LTS取り消し手順
--- 
個別テナントのLTS取り消し手順ついて記述する。

◆確認用list
| company | clusterName | ProjectId | region | VM IP | 補足 |
| --- | --- | --- | --- | --- | --- |
| SmartEducation | apg-096bdef8-c965 | ncom-apigw-across-g01-o20-1176 | japaneast | 20.63.131.167 | |
| SmartEducation | apg-8e7953c5-a1c7 | ncom-apigw-axross-g02-o19-4710 | japaneast | 20.194.203.155 | |

* stage1/aks.tf編集  
  stage1/aks.tfの`azurerm_kubernetes_cluster.cluster`の`sku_tier`を`standard`に`support_plan`をコメントアウトする  
  ```
  name                = var.tenant_aks_name
  location            = var.tenant_aks_region
  sku_tier            = "standard"
  # support_plan        = "AKSLongTermSupport"
  ```

* 設定適用 
  AKSに対し、設定したパラメータを適用
  * terraform apply  
    stage1に対し、terraform applyを実行する  
    ※実行時に変更箇所(`azurerm_kubernetes_cluster.cluster`の`sku_tier/support_plan`)が更新の対象となってることを確認する。
    ```bash
    cicdm terraform -chdir=stage1 apply -var-file="../terraform.tfvars"
    ```
    ```
    # azurerm_kubernetes_cluster.cluster will be updated in-place
    ~ resource "azurerm_kubernetes_cluster" "cluster" {
          id                                  = "/subscriptions/******/resourceGroups/aks-test194-resources-group/providers/Microsoft.ContainerService/       managedClusters/clusterName"
          name                                = "clusterName"
        ~ sku_tier                            = "Premium" -> "standard"   ★standardになっていることを確認
        ~ support_plan                        = "AKSLongTermSupport" -> "KubernetesOfficial"  ★KubernetesOfficial Planになっていることを確認
    ```
    ⇒[Apply Complate]の出力を確認する事。 

* 更新後確認
  * apigee hybrid k8sリソース確認  
    下記コマンドを実行し、各種k8sリソース稼働状況に問題が無い事  
    ※STATUS : Completed or Runningであること  
    ※READY :  Pod内のすべてのコンテナが立ち上がっている事 (1/1,2/2 等)
    ```
    kubectl get all -n apigee
    ```
  * 設定切り戻し確認  
    下記コマンドを実行し、設定が切り戻ったか確認する  
    ```
    az aks show --name <clusterName> -g <ResourceGroup> --query supportPlan
    ```
    ⇒ `KubernetesOfficial`が出力されること
    ```
    az aks show --name <clusterName> -g <ResourceGroup> --query sku.tier
    ```
    ⇒ `standard`が出力されること
