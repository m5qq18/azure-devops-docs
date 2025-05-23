---
title: 'Tutorial: Create a multistage pipeline with Azure DevOps'
description: Build an app pipeline for development and staging.
ms.topic: tutorial 
ms.date: 05/14/2025
ms.custom: template-how-to-pattern
---

# Tutorial: Create a multistage pipeline with Azure DevOps

[!INCLUDE [version-lt-eq-azure-devops](../../includes/version-lt-eq-azure-devops.md)]

Use an Azure DevOps multistage pipeline to divide your CI/CD process into stages that represent different parts of your development cycle. A multistage pipeline gives more visibility into your deployment process and makes it easier to integrate [approvals and checks](approvals.md). 

In this article, you'll create two App Service instances and build a YAML pipeline with three stages: 

> [!div class="checklist"]
> * [Build: Build the source code and produce a package](#add-the-build-stage)
> * [Dev: Deploy your package to a development site for testing](#add-the-dev-stage)
> * [Staging: Deploy to a staging Azure App Service instance with a manual approval check](#add-the-staging-stage)

In a real-world scenario, you may have another stage for deploying to production depending on your DevOps process. 

The example code in this exercise is for a .NET web application for a pretend space game that includes a leaderboard to show high scores. You'll deploy to both development and staging instances of Azure Web App for Linux. 


## Prerequisites

| **Product** | **Requirements**   |
|---|---|
| **Azure DevOps** | - An Azure DevOps organization and project. [Create one for free](../get-started/pipelines-sign-up.md). <br>   - **Permissions:**<br>      &nbsp;&nbsp;&nbsp;&nbsp;- To grant access to all pipelines in the project: You must be a member of the [Project Administrators group](../../organizations/security/change-project-level-permissions.md).<br>      &nbsp;&nbsp;&nbsp;&nbsp;- To create service connections: You must have the *Administrator* or *Creator* role for [service connections](../library/add-resource-protection.md).<br>   - An ability to run pipelines on Microsoft-hosted agents. You can either purchase a [parallel job](../licensing/concurrent-jobs.md) or you can request a free tier.  |
| **GitHub** | - A [GitHub](https://github.com) account.|

## Fork the project

Fork the following sample repository on GitHub.

```
https://github.com/MicrosoftDocs/mslearn-tailspin-spacegame-web-deploy
```

## Create the App Service instances

Before you can deploy your pipeline, you need to first create an App Service instance to deploy to. You'll use Azure CLI to create the instance. 

1. Sign in to the [Azure portal](https://portal.azure.com).  

1. From the menu, select **Cloud Shell** and the **Bash** experience.

1. Generate a random number that makes your web app's domain name unique. The advantage of having a unique value is that your App Service instance won't have a name conflict with other learners completing this tutorial. 

    ```code
    webappsuffix=$RANDOM    
    ```

1. Open a command prompt and use a `az group create` command to create a resource group named *tailspin-space-game-rg* that contains all of your App Service instances. Update the `location` value to use your closest region. 
    
    ```azurecli
    az group create --location eastus --name tailspin-space-game-rg
    ```

1. Use the command prompt to create an App Service plan.

    ```azurecli
    az appservice plan create \
      --name tailspin-space-game-asp \
      --resource-group tailspin-space-game-rg \
      --sku B1 \
      --is-linux
    ```

1. In the command prompt, create two App Service instances, one for each instance (Dev and Staging) with the `az webapp create` command. 

    ```azurecli
    az webapp create \
      --name tailspin-space-game-web-dev-$webappsuffix \
      --resource-group tailspin-space-game-rg \
      --plan tailspin-space-game-asp \
      --runtime "DOTNET|8.0"
    
    az webapp create \
      --name tailspin-space-game-web-staging-$webappsuffix \
      --resource-group tailspin-space-game-rg \
      --plan tailspin-space-game-asp \
      --runtime "DOTNET|8.0"
    ```

1. With the command prompt, list both App Service instances to verify that they're running with the `az webapp list` command. 

    ```azurecli
    az webapp list \
      --resource-group tailspin-space-game-rg \
      --query "[].{hostName: defaultHostName, state: state}" \
      --output table
    ```

1. Copy the names of the App Service instances to use as variables in the next section. 

## Create your Azure DevOps project and variables

Set up your Azure DevOps project and a build pipeline. You'll also add variables for your development and staging instances. 

Your build pipeline:

* Includes a trigger that runs when there's a code change to branch
* Defines two variables, `buildConfiguration` and `releaseBranchName`
* Includes a stage named Build that builds the web application
* Publishes an artifact you'll use in a later stage

#### Add the Build stage 

[!INCLUDE [include](../ecosystems/includes/create-pipeline-before-template-selected.md)]

7. When the **Configure** tab appears, select **Starter pipeline**.

8. Replace the contents of *azure-pipelines.yml* with this code. 

    ```yml
    trigger:
    - '*'
    
    variables:
      buildConfiguration: 'Release'
      releaseBranchName: 'release'
    
    stages:
    - stage: 'Build'
      displayName: 'Build the web application'
      jobs: 
      - job: 'Build'
        displayName: 'Build job'
        pool:
          vmImage: 'ubuntu-22.04'
          demands:
          - npm
    
        variables:
          wwwrootDir: 'Tailspin.SpaceGame.Web/wwwroot'
          dotnetSdkVersion: '8.x'
    
        steps:
        - task: UseDotNet@2
          displayName: 'Use .NET SDK $(dotnetSdkVersion)'
          inputs:
            version: '$(dotnetSdkVersion)'
    
        - task: Npm@1
          displayName: 'Run npm install'
          inputs:
            verbose: false
    
        - script: './node_modules/.bin/node-sass $(wwwrootDir) --output $(wwwrootDir)'
          displayName: 'Compile Sass assets'
    
        - task: gulp@1
          displayName: 'Run gulp tasks'
    
        - script: 'echo "$(Build.DefinitionName), $(Build.BuildId), $(Build.BuildNumber)" > buildinfo.txt'
          displayName: 'Write build info'
          workingDirectory: $(wwwrootDir)
    
        - task: DotNetCoreCLI@2
          displayName: 'Restore project dependencies'
          inputs:
            command: 'restore'
            projects: '**/*.csproj'
    
        - task: DotNetCoreCLI@2
          displayName: 'Build the project - $(buildConfiguration)'
          inputs:
            command: 'build'
            arguments: '--no-restore --configuration $(buildConfiguration)'
            projects: '**/*.csproj'
    
        - task: DotNetCoreCLI@2
          displayName: 'Publish the project - $(buildConfiguration)'
          inputs:
            command: 'publish'
            projects: '**/*.csproj'
            publishWebProjects: false
            arguments: '--no-build --configuration $(buildConfiguration) --output $(Build.ArtifactStagingDirectory)/$(buildConfiguration)'
            zipAfterPublish: true
    
        - publish: '$(Build.ArtifactStagingDirectory)'
          artifact: drop
    ```


9. When you're ready, select **Save and run**.

#### Add instance variables

1. In Azure DevOps, go to **Pipelines** > **Library**. 

1. Select **+ Variable group**.

1. Under **Properties**, add *Release* for the variable group name.

1. Create a two variables to refer to your development and staging host names. Replace the value `1234` with the correct value for your instance. 

    
    |Variable name  |Example value  |
    |---------|---------|
    |WebAppNameDev     |    tailspin-space-game-web-dev-1234     |
    |WebAppNameStaging     |    tailspin-space-game-web-staging-1234     |
    
 
1. Select **Save** to save your variables. 

## Add the Dev stage

Next, you'll update your pipeline to promote your build to the *Dev* stage. 

1. In Azure Pipelines, go to **Pipelines** > **Pipelines**. 

1.  Select **Edit** in the contextual menu to edit your pipeline. 

    :::image type="content" source="media/mutistage-pipeline/multistage-pipeline-edit-contextual-menu.png" alt-text="Screenshot of select Edit menu item. ":::
    
1. Update *azure-pipelines.yml* to include a Dev stage. In the Dev stage, your pipeline will:

    * Run when the Build stage succeeds because of a condition
    * Download an artifact from `drop`
    * Deploy to Azure App Service with an [Azure Resource Manager service connection](../library/service-endpoints.md)

    ```yml
    trigger:
    - '*'
    
    variables:
      buildConfiguration: 'Release'
      releaseBranchName: 'release'
    
    stages:
    - stage: 'Build'
      displayName: 'Build the web application'
      jobs: 
      - job: 'Build'
        displayName: 'Build job'
        pool:
          vmImage: 'ubuntu-22.04'
          demands:
          - npm
    
        variables:
          wwwrootDir: 'Tailspin.SpaceGame.Web/wwwroot'
          dotnetSdkVersion: '8.x'
    
        steps:
        - task: UseDotNet@2
          displayName: 'Use .NET SDK $(dotnetSdkVersion)'
          inputs:
            version: '$(dotnetSdkVersion)'
    
        - task: Npm@1
          displayName: 'Run npm install'
          inputs:
            verbose: false
    
        - script: './node_modules/.bin/node-sass $(wwwrootDir) --output $(wwwrootDir)'
          displayName: 'Compile Sass assets'
    
        - task: gulp@1
          displayName: 'Run gulp tasks'
    
        - script: 'echo "$(Build.DefinitionName), $(Build.BuildId), $(Build.BuildNumber)" > buildinfo.txt'
          displayName: 'Write build info'
          workingDirectory: $(wwwrootDir)
    
        - task: DotNetCoreCLI@2
          displayName: 'Restore project dependencies'
          inputs:
            command: 'restore'
            projects: '**/*.csproj'
    
        - task: DotNetCoreCLI@2
          displayName: 'Build the project - $(buildConfiguration)'
          inputs:
            command: 'build'
            arguments: '--no-restore --configuration $(buildConfiguration)'
            projects: '**/*.csproj'
    
        - task: DotNetCoreCLI@2
          displayName: 'Publish the project - $(buildConfiguration)'
          inputs:
            command: 'publish'
            projects: '**/*.csproj'
            publishWebProjects: false
            arguments: '--no-build --configuration $(buildConfiguration) --output $(Build.ArtifactStagingDirectory)/$(buildConfiguration)'
            zipAfterPublish: true
    
        - publish: '$(Build.ArtifactStagingDirectory)'
          artifact: drop
    
    - stage: 'Dev'
      displayName: 'Deploy to the dev environment'
      dependsOn: Build
      condition:  succeeded()
      jobs:
      - deployment: Deploy
        pool:
          vmImage: 'ubuntu-22.04'
        environment: dev
        variables:
        - group: Release
        strategy:
          runOnce:
            deploy:
              steps:
              - download: current
                artifact: drop
              - task: AzureWebApp@1
                displayName: 'Azure App Service Deploy: dev website'
                inputs:
                  azureSubscription: 'your-subscription'
                  appType: 'webAppLinux'
                  appName: '$(WebAppNameDev)'
                  package: '$(Pipeline.Workspace)/drop/$(buildConfiguration)/*.zip'
    ```
    
1. Change the `AzureWebApp@1` task to use your subscription. 

    1. Select **Settings** for the task. 

        :::image type="content" source="media/mutistage-pipeline/select-settings-azurewebapptask.png" alt-text="Screenshot of settings option in YAML editor task. ":::

    1. Update the `your-subscription` value for **Azure Subscription** to use your own subscription. You may need to authorize access as part of this process. If you run into a problem authorizing your resource within the YAML editor, an alternate approach is to [create a service connection](../library/service-endpoints.md#create-a-service-connection). 
    
        :::image type="content" source="media/mutistage-pipeline/edit-your-subscription-value.png" alt-text="Screenshot of Azure subscription menu item. ":::

    1. Set the **App type** to Web App on Linux. 
    
    1. Select **Add** to update the task. 

1. Save and run your pipeline. 

## Add the Staging stage 

Last, you'll promote the Dev stage to Staging. Unlike the Dev environment, you want to have more control in the staging environment you'll add a manual approval. 


#### Create staging environment

1. From Azure Pipelines, select **Environments**.

1. Select **New environment**.

1. Create a new environment with the name *staging* and **Resource** set to *None*. 

1. On the staging environment page, select **Approvals and checks**.

    :::image type="content" source="media/mutistage-pipeline/pipeline-add-check-to-environment.png" alt-text="Screenshot of approvals and checks menu option. ":::

1. Select **Approvals**. 

1. In **Approvers**, select **Add users and groups**, and then select your account.

1. In **Instructions to approvers**, write  *Approve this change when it's ready for staging*.

1. Select **Save**. 

#### Add new stage to pipeline

You'll add new stage, `Staging` to the pipeline that includes a manual approval. 

1. Edit your pipeline file and add the `Staging` section.  

    ```yml
    trigger:
    - '*'
    
    variables:
      buildConfiguration: 'Release'
      releaseBranchName: 'release'
    
    stages:
    - stage: 'Build'
      displayName: 'Build the web application'
      jobs: 
      - job: 'Build'
        displayName: 'Build job'
        pool:
          vmImage: 'ubuntu-22.04'
          demands:
          - npm
    
        variables:
          wwwrootDir: 'Tailspin.SpaceGame.Web/wwwroot'
          dotnetSdkVersion: '8.x'
    
        steps:
        - task: UseDotNet@2
          displayName: 'Use .NET SDK $(dotnetSdkVersion)'
          inputs:
            version: '$(dotnetSdkVersion)'
    
        - task: Npm@1
          displayName: 'Run npm install'
          inputs:
            verbose: false
    
        - script: './node_modules/.bin/node-sass $(wwwrootDir) --output $(wwwrootDir)'
          displayName: 'Compile Sass assets'
    
        - task: gulp@1
          displayName: 'Run gulp tasks'
    
        - script: 'echo "$(Build.DefinitionName), $(Build.BuildId), $(Build.BuildNumber)" > buildinfo.txt'
          displayName: 'Write build info'
          workingDirectory: $(wwwrootDir)
    
        - task: DotNetCoreCLI@2
          displayName: 'Restore project dependencies'
          inputs:
            command: 'restore'
            projects: '**/*.csproj'
    
        - task: DotNetCoreCLI@2
          displayName: 'Build the project - $(buildConfiguration)'
          inputs:
            command: 'build'
            arguments: '--no-restore --configuration $(buildConfiguration)'
            projects: '**/*.csproj'
    
        - task: DotNetCoreCLI@2
          displayName: 'Publish the project - $(buildConfiguration)'
          inputs:
            command: 'publish'
            projects: '**/*.csproj'
            publishWebProjects: false
            arguments: '--no-build --configuration $(buildConfiguration) --output $(Build.ArtifactStagingDirectory)/$(buildConfiguration)'
            zipAfterPublish: true
    
        - publish: '$(Build.ArtifactStagingDirectory)'
          artifact: drop
    
    - stage: 'Dev'
      displayName: 'Deploy to the dev environment'
      dependsOn: Build
      condition:  succeeded()
      jobs:
      - deployment: Deploy
        pool:
          vmImage: 'ubuntu-22.04'
        environment: dev
        variables:
        - group: Release
        strategy:
          runOnce:
            deploy:
              steps:
              - download: current
                artifact: drop
              - task: AzureWebApp@1
                displayName: 'Azure App Service Deploy: dev website'
                inputs:
                  azureSubscription: 'your-subscription'
                  appType: 'webAppLinux'
                  appName: '$(WebAppNameDev)'
                  package: '$(Pipeline.Workspace)/drop/$(buildConfiguration)/*.zip'
    
    - stage: 'Staging'
      displayName: 'Deploy to the staging environment'
      dependsOn: Dev
      jobs:
      - deployment: Deploy
        pool:
          vmImage: 'ubuntu-22.04'
        environment: staging
        variables:
        - group: 'Release'
        strategy:
          runOnce:
            deploy:
              steps:
              - download: current
                artifact: drop
              - task: AzureWebApp@1
                displayName: 'Azure App Service Deploy: staging website'
                inputs:
                  azureSubscription: 'your-subscription'
                  appType: 'webAppLinux'
                  appName: '$(WebAppNameStaging)'
                  package: '$(Pipeline.Workspace)/drop/$(buildConfiguration)/*.zip'
    ```

1. Change the `AzureWebApp@1` task in the Staging stage to use your subscription. 

    1. Select **Settings** for the task. 

        :::image type="content" source="media/mutistage-pipeline/select-settings-azurewebapptask.png" alt-text="Screenshot of settings option in YAML editor task. ":::

    1. Update the `your-subscription` value for **Azure Subscription** to use your own subscription. You may need to authorize access as part of this process. 
    
        :::image type="content" source="media/mutistage-pipeline/edit-your-subscription-value.png" alt-text="Screenshot of Azure subscription menu item. ":::

    1. Set the **App type** to Web App on Linux. 
    
    1. Select **Add** to update the task. 

1.  Go to the pipeline run. Watch the build as it runs. When it reaches `Staging`, the pipeline waits for manual release approval. You'll also receive an email that you have a pipeline pending approval. 

    :::image type="content" source="media/mutistage-pipeline/pipeline-wait-approval.png" alt-text="Screenshot of wait for pipeline approval.":::

1. Review the approval and allow the pipeline to run. 
 
    :::image type="content" source="media/mutistage-pipeline/pipeline-check-manual-validation.png" alt-text="Screenshot of manual validation check.":::

## Clean up resources

 If you're not going to continue to use this application, delete the resource group in Azure portal and the project in Azure DevOps with the following steps:

To clean up your resource group:

1. Go to the [Azure portal](https://portal.azure.com?azure-portal=true) and sign in.
1. From the menu bar, select Cloud Shell. When prompted, select the **Bash** experience.

    :::image type="content" source="../apps/cd/azure/media/azure-portal-menu-cloud-shell.png" alt-text="A screenshot of the Azure portal showing selecting the Cloud Shell menu item. ":::

1. Run the following [az group delete](/cli/azure/group#az-group-delete) command to delete the resource group that you used, `tailspin-space-game-rg`.

    ```azurecli
    az group delete --name tailspin-space-game-rg
    ```

To delete your Azure DevOps project, including the build pipeline, see [Delete project](../../organizations/projects/delete-project.md).

## Related content

- [Key concepts for new Azure Pipelines users](../get-started/key-pipelines-concepts.md)
- [Deploy to App Service using Azure Pipelines](/azure/app-service/deploy-azure-pipelines)
