## 常规问题
需要按照github上的说明安装指定版本的环境

debug之前需要先执行``azd up``，若报错提示无权限则需要到指定的keyvalut中添加access policy

debug web时需要先start api and web

debug api只需要start web

## terraform配置
### 创建container
* 创建一个tfstate的resourcegroup
* 创建一个strogeaccount
* 创建一个container
```

#!/bin/bash

RESOURCE_GROUP_NAME=tfstate
STORAGE_ACCOUNT_NAME=tfstate$RANDOM
CONTAINER_NAME=tfstate

# Create resource group
az group create --name $RESOURCE_GROUP_NAME --location eastus

# Create storage account
az storage account create --resource-group $RESOURCE_GROUP_NAME --name $STORAGE_ACCOUNT_NAME --sku Standard_LRS --encryption-services blob

# Create blob container
az storage container create --name $CONTAINER_NAME --account-name $STORAGE_ACCOUNT_NAME

```


### 配置backend
* 创建一个accountkey
* 为其配置环境变量ARM_ACCESS_KEY
```
ACCOUNT_KEY=$(az storage account keys list --resource-group $RESOURCE_GROUP_NAME --account-name $STORAGE_ACCOUNT_NAME --query '[0].value' -o tsv)
export ARM_ACCESS_KEY=$ACCOUNT_KEY
```

创建一个.tf文件并替换配置内容
```
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~>3.0"
    }
  }
  backend "azurerm" {
      resource_group_name  = "tfstate"
      storage_account_name = "<storage_account_name>"
      container_name       = "tfstate"
      key                  = "terraform.tfstate"
  }

}

provider "azurerm" {
  features {}
}

resource "azurerm_resource_group" "state-demo-secure" {
  name     = "state-demo"
  location = "eastus"
}
```
控制台命令
```
terraform init
terraform apply
```
### 项目文件内配置
* 在infra文件夹内创建provider.conf.json文件
* 输入内容
```
{
    "storage_account_name": "${RS_STORAGE_ACCOUNT}",
    "container_name": "${RS_CONTAINER_NAME}",
    "key": "azd/azdremotetest.tfstate",
    "resource_group_name": "${RS_RESOURCE_GROUP}"
}
```
在provider.tf中添加内容
```
terraform {
  required_version = ">= 1.1.7, < 2.0.0"
  backend "azurerm" {
      resource_group_name  = "tfstate"
      storage_account_name = "<storage_account_name>"
      container_name       = "tfstate"
      key                  = "terraform.tfstate"
  }
```
在env中设置变量
```
azd env set RS_STORAGE_ACCOUNT your_storage_account_name
azd env set RS_CONTAINER_NAME your_terraform_container_name
azd env set RS_RESOURCE_GROUP your_storage_account_resource_group
```


## aks报错
若出现报错
> Version *** not supported in this region.


* 输入 ``az aks get-versions -l <location>``查看该region支持的aks版本
* 打开文件aks=managed-cluster.bicep
* 修改kubernetesVersion值为支持的版本号
