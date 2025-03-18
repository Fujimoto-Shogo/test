# 【AKSテナント】新CICDモジュール AKS LTS適用手順書手順書
CICD Moduleを利用した、AKSをLTSに対応させるための手順について記述する。

## 目次
- [【AKSテナント】新CICDモジュール AKS LTS適用手順書手順書](#aksテナント新cicdモジュール-aks-lts適用手順書手順書)
  - [目次](#目次)
  - [前提条件](#前提条件)
  - [事前作業](#事前作業)
  - [直前作業](#直前作業)
  - [LTS適用](#lts適用)
  - [LTS適用切り戻し手順書](#lts適用切り戻し手順書)


---
## 前提条件
---
- GCP Project IDに対して必要な権限が付与されたSAのjson keyが用意されている事  
- 必要となるクラウド(GCP/Azure)のクレデンシャル情報を把握している事
- Dockerコンテナが動作する作業環境が用意されている事
- 新CICDモジュールのcontainerイメージがローカルに存在する事
- AKSのversionが`1.27以上`となっている事

---
## 事前作業
---
| company | clusterName | ProjectId | region | VM IP | 補足 |
| --- | --- | --- | --- | --- | --- |
| SmartEducation | apg-096bdef8-c965 | ncom-apigw-across-g01-o20-1176 | japaneast | 20.63.131.167 | |
| SmartCityStg | apg-8363620c-607d | ncom-apigw-across-g01-o19-9571 | japaneast | 20.210.37.173 | |
| SmartEducation | apg-8e7953c5-a1c7 | ncom-apigw-axross-g02-o19-4710 | japaneast | 20.194.203.155 | |
| SmartCityPrd | apg-af773d8f-bc5a | ncom-apigw-across-g01-o21-9574 | japaneast | 20.27.40.179 | |
| NTTPC Stg|apg-6ef84802-05bc|ncom-apigw-across-g01-o23-8409|japaneast|4.189.95.40||
| NTTPC Prd|apg-d7bceb8d-4f3b|ncom-apigw-across-g01-o25-0693|japaneast|4.190.20.162||

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
  * `stage1/stage2`の`main.tf`を編集する  
    `terraform.required_providers`のazurerm.versionを`3.97.1`に編集する
    ```
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.97.1"
    }
    ```
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
  * azコマンドで対象のクラスタが確認できる事  
    ```
    az aks list -o table
    ```

- CICDモジュールの環境準備
  - stage1,stage2に対して、Planが実行でき "No Chages" が出力されている事  
  - コメントアウト実行/解除の為の以下アドオンがVSCodeにインストールされている事  
    - Azure Terraform
    - HashiCorp Terraform

---
## 直前作業 
--- 
| company | clusterName | ProjectId | region | VM IP | 補足 |
| --- | --- | --- | --- | --- | --- |
| SmartEducation | apg-096bdef8-c965 | ncom-apigw-across-g01-o20-1176 | japaneast | 20.63.131.167 | |
| SmartCityStg | apg-8363620c-607d | ncom-apigw-across-g01-o19-9571 | japaneast | 20.210.37.173 | |
| SmartEducation | apg-8e7953c5-a1c7 | ncom-apigw-axross-g02-o19-4710 | japaneast | 20.194.203.155 | |
| SmartCityPrd | apg-af773d8f-bc5a | ncom-apigw-across-g01-o21-9574 | japaneast | 20.27.40.179 | |
| NTTPC Stg|apg-6ef84802-05bc|ncom-apigw-across-g01-o23-8409|japaneast|4.189.95.40||
| NTTPC Prd|apg-d7bceb8d-4f3b|ncom-apigw-across-g01-o25-0693|japaneast|4.190.20.162||

* screenコマンドでセッションをアタッチ  
  * 構成ファイル一式が配置された作業ディレクトリに移動する 
    ```
    screen
    ```
    ※[Ctrl]+[a] [d] でデタッチできる  

    もし急にソフトウェアが落ちてしまった場合、以下コマンドでscreen セッションに再度アタッチする
    ```
    screen -r
    ```  

* 作業ディレクトリ移動
  * 構成ファイル一式が配置された作業ディレクトリに移動する  
    ※listの[clusterName]が対象
    ```
    cd /tmp/<clusterName>
    ```

* cicdモジュール利用準備
  * alias設定  
    cicdmコマンドを使用するため、以下コマンドでaliasを設定する事
    ```
    alias cicdm='sudo docker run --rm -it -e TF_TOKEN_app_terraform_io=$TF_TOKEN -v `pwd`:/go/work -w /go/work cicdmodule.azurecr.io/cicd-module:release-202403'
    ```
  * TF_TOKENの設定
    ```
    export TF_TOKEN="tftoken"
    ```
  * stage1 plan  
    Terraform使えるかの正常確認としてstage1のplanを実行する。  
    No Chagesが出力される事を確認する事。
    ```
    cicdm terraform -chdir=stage1 plan -var-file="../terraform.tfvars"
  * stage2 plan  
    Terraform使えるかの正常確認としてstage2のplanを実行する。  
    No Chagesが出力される事を確認する事。
    ```
    cicdm terraform -chdir=stage2 plan -var-file="../terraform.tfvars"
    ```  
* 接続先確認
  * kubectl  
    以下コマンドを実行し、対象のClusterに接続されているか確認する事  
    ※Listの[clusterName]を確認する
    ```
    kubectl config current-context
    ```
  * cicdモジュール  
    ./terraform.tfvarsの内容を確認し、対象テナントの構成ファイルであるか確認する  
    ```
    cat ./terraform.tfvars
    gcp_project_name         = "listの[ProjectId]と比較し、対象テナントのGcpIdである事"
    tenant_aks_name          = "listの[clusterName]と比較し、対象テナントのCluster名である事"
    ```
  * azコマンド
    ```
    az account show -o table
    ```
    対象のtenant idに接続されている事

<br>

---
## LTS適用
--- 
個別テナントのLTS適用手順ついて記述する。  
以下、個別テナントの構成フォルダに移動し作業を実施する。

◆確認用list
| company | clusterName | ProjectId | region | VM IP | 補足 |
| --- | --- | --- | --- | --- | --- |
| SmartEducation | apg-096bdef8-c965 | ncom-apigw-across-g01-o20-1176 | japaneast | 20.63.131.167 | |
| SmartCityStg | apg-8363620c-607d | ncom-apigw-across-g01-o19-9571 | japaneast | 20.210.37.173 | |
| SmartEducation | apg-8e7953c5-a1c7 | ncom-apigw-axross-g02-o19-4710 | japaneast | 20.194.203.155 | |
| SmartCityPrd | apg-af773d8f-bc5a | ncom-apigw-across-g01-o21-9574 | japaneast | 20.27.40.179 | |
| NTTPC Stg|apg-6ef84802-05bc|ncom-apigw-across-g01-o23-8409|japaneast|4.189.95.40||
| NTTPC Prd|apg-d7bceb8d-4f3b|ncom-apigw-across-g01-o25-0693|japaneast|4.190.20.162||

* stage1/aks.tf編集  
  stage1/aks.tfの`azurerm_kubernetes_cluster.cluster`に`sku_tier/support_plan`を追加する  
  ```
  name                = var.tenant_aks_name
  location            = var.tenant_aks_region
  sku_tier            = "Premium"
  support_plan        = "AKSLongTermSupport"
  ```

* LTS適用  
  AKSに対し、LTSプランを適用する
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
        ~ sku_tier                            = "Free" -> "Premium"
        ~ support_plan                        = "KubernetesOfficial" -> "AKSLongTermSupport"
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
  * LTS適用確認  
    下記コマンドを実行し、LTSが適用されたか確認する  
    ```
    az aks show --name <clusterName> -g <ResourceGroup> --query supportPlan
    ```
    ⇒ `AKSLongTermSupport`が出力されること
    ```
    az aks show --name <clusterName> -g <ResourceGroup> --query sku.tier
    ```
    ⇒ `Premium`が出力されること

以上でLTS適用手順完了。

--- 
## LTS適用切り戻し手順書
--- 

設定を再更新することで、設定の切り戻しを行う。

◆確認用list
| company | clusterName | ProjectId | region | VM IP | 補足 |
| --- | --- | --- | --- | --- | --- |
| SmartEducation | apg-096bdef8-c965 | ncom-apigw-across-g01-o20-1176 | japaneast | 20.63.131.167 | |
| SmartCityStg | apg-8363620c-607d | ncom-apigw-across-g01-o19-9571 | japaneast | 20.210.37.173 | |
| SmartEducation | apg-8e7953c5-a1c7 | ncom-apigw-axross-g02-o19-4710 | japaneast | 20.194.203.155 | |
| SmartCityPrd | apg-af773d8f-bc5a | ncom-apigw-across-g01-o21-9574 | japaneast | 20.27.40.179 | |
| NTTPC Stg|apg-6ef84802-05bc|ncom-apigw-across-g01-o23-8409|japaneast|4.189.95.40||
| NTTPC Prd|apg-d7bceb8d-4f3b|ncom-apigw-across-g01-o25-0693|japaneast|4.190.20.162||
* stage1/aks.tf編集  
  stage1/aks.tfの`azurerm_kubernetes_cluster.cluster`の`sku_tier/support_plan`をコメントアウトする  
  ```
  name                = var.tenant_aks_name
  location            = var.tenant_aks_region
  # sku_tier            = "Premium"
  # support_plan        = "AKSLongTermSupport"
  ```

* 設定切り戻し  
  AKSに対し、設定したパラメータを切り戻す
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
        ~ sku_tier                            = "Premium" -> "Free"   ★Free Tierに切り戻し
        ~ support_plan                        = "AKSLongTermSupport" -> "KubernetesOfficial"  ★KubernetesOfficial Planに切り戻し
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
    ⇒ `Free`が出力されること

以上で設定切り戻し手順完了。
