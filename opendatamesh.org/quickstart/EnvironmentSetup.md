Open Data Mesh Demo
==============

Table of Contents
=================

* [Open Data Mesh Demo](#open-data-mesh-demo)
* [Table of Contents](#table-of-contents)
* [Overview](#overview)
* [Requirements](#requirements)
* [Environment Setup](#environment-setup)
   * [Create Azure DevOps Pipelines](#create-azure-devops-pipelines)
   * [Register Service Connections](#register-service-connections)
   * [Create Group Variables](#create-group-variables)
* [ODM Platform Deployment](#odm-platform-deployment)
   * [Run ODM Infrastructure Pipeline](#run-odm-infrastructure-pipeline)
   * [Register SSH service connection](#register-ssh-service-connection)
   * [Run ODM Application Pipelines](#run-odm-application-pipelines)
* [Who do I talk to?](#who-do-i-talk-to)

<!-- Created by https://github.com/ekalinin/github-markdown-toc -->


# Overview
This guide outlines the steps required to deploy the Open Data Mesh platform in the Azure environment.

# Requirements
 
 * Azure Subscription
 * Access to Azure DevOps
   * [Azure DevOps Terraform extension](https://marketplace.visualstudio.com/items?itemName=JasonBJohnson.azure-pipelines-tasks-terraform) installed
 * Terraform installed
 * Azure CLI installed
 * cURL or Postman installed

# Environment Setup
## Create Azure DevOps Pipelines
To deploy the ODM Platform, you'll need to set up specific Azure DevOps pipelines:

1. Access Azure DevOps.
2. If not already present, create a new project.
3. Go to the Repository section (found in the left menu) and click on "Import repository". Select the Git protocol and enter the following link: 
    * https://github.com/opendatamesh-initiative/odm-platform.git
4. Repeat step 3 for the following repositories:
    * https://github.com/opendatamesh-initiative/odm-platform-up-services-executor-azuredevops.git
    * https://github.com/Giandom/odm-demo-dp-airlines.git
5. In the Pipelines section (still in the left menu), click on **Create Pipeline**.
    * The first pipeline to create is for releasing the infrastructure on which the ODM Platform will rely. This pipeline will provision a VM (Standard_A1_v2, 1CPU / 2GB RAM).
        * In the repository connection section, choose Azure Repo and select the **odm-demo-dp-airlines** repository.
        * In the Configure section, select the Existing Azure Pipelines YAML file option. Then choose $master as the branch and **odm-platform-infra/azure-pipelines.yml** as the YAML file.
        * Save the pipeline by selecting Save from the menu near the "Run" button (the pipeline will be saved but not executed).
        * Change the name of the pipeline to **odm-platform-infrastructure**.
    * The second pipeline is for deploying ODM within the infrastructure created by the first pipeline.
        * In the repository connection section, choose Azure Repo and select the imported odm-platform repository.
        * In the Configure section, select the Existing Azure Pipelines YAML file option. Then choose **$main** as the branch and pass the following path: **/azure-pipelines.yml**.
        * Save the pipeline without executing it.
        * Change the name of the pipeline to **odm-platform-application**.
    * The third pipeline is for deploying the ODM Azure DevOps executor, a component that will interface with your Azure DevOps.
        * In the repository connection section, choose Azure Repo and select the imported **odm-platform-up-services-executor-azuredevops** repository.
        * In the Configure section, select the Existing Azure Pipelines YAML file option. Then choose **$main** as the branch and pass the following path: **/azure-pipelines.yml**.
        * Save the pipeline without executing it.
        * Change the name of the pipeline to **odm-platform-executor-azdevops**.
    * The fourth pipeline will provision the infrastructure for the demo (provisioning a VM and a MySQL DB).
        * In the repository connection section, choose Azure Repo and select the **odm-demo-dp-airlines** repository.
        * In the Configure section, select the Existing Azure Pipelines YAML file option. Then choose **$master** as the branch and pass the following path: **infrastructure/azure-pipelines-infra.yml**.
        * Save the pipeline without executing it.
        * Change the name of the pipeline to odm-demo-infrastructure.
    * The fifth and final pipeline is for deploying the application that will expose the REST API.
        * In the repository connection section, choose Azure Repo and select the **odm-demo-dp-airlines** repository.
        * In the Configure section, select the Existing Azure Pipelines YAML file option. Then choose **$master** as the branch and pass the following path: **application/airlinedemo/azure-pipelines-app.yml**.
        * Save the pipeline without executing it.
        * Change the name of the pipeline to **odm-demo-application**.

## Register Service Connections
1. Create an application in your Azure AD to manage the pipelines programmatically:
    * Follow the instructions in [this document](https://github.com/opendatamesh-initiative/odm-platform-up-services-executor-azuredevops), in the section: **Azure Environment**.
2. Return to Azure DevOps and create an Azure Resource Manager service connection.
    * Go to **Project Settings** (found at the bottom left of Azure DevOps).
    * In the left menu, click on **Service connections** under the Pipelines section.
    * Click on **New service connection**, in the top right, and select **Azure Resource Manager** as the type and **Workload Identity Federation** as the authentication method. Enter the following configurations:
        * Scope level: Choose **Subscription**
        * Subscription: Select the name of your Azure subscription.
        * Resource Group: Select an existing resource group in Azure where resources will be provisioned.
        * Service connection name: Enter  **ODMServiceConnection**
        * Check the option: Grant access permission to all pipelines.

## Create Group Variables
1. Create the following variable groups in the Library section of Azure DevOps:
   * Create a variable group and name it: **ODM-Platform**
      * Add the following variables: 
        * GITHUB_USERNAME: Necessary for reading dependencies generated by the odm-platform project (even though the project is public, authentication is required).
        * GITHUB_PASSWORD: Enter a personal password or app password for GitHub and make it private (click on the lock icon on the right).
        * AZURE_ODM_APP_CLIENT_ID: The client ID of the application created in step 6.
        * AZURE_ODM_APP_CLIENT_SECRET: Enter the client secret of the application created in step 6 and make it private (click on the lock icon on the right).
        * AZURE_TENANT_ID: Your Azure tenant ID.
      * Click on **Save**.
      * Click on **Pipeline permissions**, then on the three dots, and click on **Open access**.
   * Create a second group of variables and name it: **Azure-Config**.
      * Add the following variables:
        * backendServiceArm: The name of the service connection created in step 7 of this section.
        * backendAzureRmResourceGroupName: The name of an existing resource group in Azure where resources will be provisioned.
        * backendAzureRmStorageAccountName: The name of the storage account to be used for saving the Terraform state (if you don't have one, you can create it by following [this guide](https://learn.microsoft.com/en-us/azure/storage/common/storage-account-create?tabs=azure-portal))
        * backendAzureRmContainerName: The name of the container to be used for saving the Terraform state (if you have just created the storage account, follow [this guide](https://learn.microsoft.com/en-us/azure/storage/blobs/blob-containers-portal))
      * Click on **Save**.
      * Click on **Pipeline permissions**, then on the three dots, and click on **Open access**.
2. Modify the **config.json** file in the root of this repository to fill in the necessary configurations:
    * **tenant_id**: Your Azure tenant ID.
    * **region**: The region in which your resources will be deployed (e.g., "Germany West Central").
    * **subscription_id**: The ID of your Azure subscription.
    * **resource_group_name**: The name of the Azure resource group in which you want to deploy resources.


# ODM Platform Deployment
With the environment prepared, we can deploy the components of the ODM Platform.

## Run ODM Infrastructure Pipeline
Launch the **odm-platform-infrastructure** pipeline on Azure DevOps. 
Once the execution is complete, view the details of the **Terraform Apply/Apply Terraform Plan** step and copy the IP found at the bottom of the execution log: **vm-public-endpoint**.

## Register SSH service connection
Create an SSH service connection:
   1. Go to the **Project Settings** page, located at the bottom left of Azure DevOps.
   2. In the left menu, click on **Service connections** under the **Pipelines** section.
   3. Click on **New service connection**, in the top right, and select SSH as the type. Enter the following values:
      * Host name: Enter the IP you copied earlier.
      * Port number: Leave the default, 22.
      * Username: odm
      * Password: 0p3nD@t@M3sh
      * Service connection name: odm-platform.
      * Check the option: Grant access permission to all pipelines.
   4. Save. 

## Run ODM Application Pipelines
Return to the pipelines page. Now run the **odm-platform-application** pipeline.
At the end of the execution, run the **odm-platform-executor-azdevops** pipeline.
If there were no errors, you should be able to reach the following endpoints, replacing the [IP] placeholder with the hostname used in step 3:

* ODM Platform Registry Module: http://[IP]:8001/api/v1/pp/registry/swagger-ui/index.html
* ODM DevOps Module: http://[IP]:8002/api/v1/pp/devops/swagger-ui/index.html


Congratulations 🎉 You have successfully deployed the Open Data Mesh Platform!

# Who do I talk to?

* Giandomenico Avelluto
* Quantyca S.p.A
