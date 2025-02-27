---
title: Make your project compatible with Azure Developer CLI (preview)
description: How to convert an app to an Azure developer enabled template.
author: hhunter-ms
ms.author: hannahhunter
ms.date: 08/11/2022
ms.service: azure-dev-cli
ms.topic: how-to
ms.custom: devx-track-azdevcli
zone_pivot_group_filename: developer/azure-developer-cli/azd-zone-pivot-groups.json
zone_pivot_groups: make-azure-developer-cli-compatible-set
---

# Make your project compatible with Azure Developer CLI (preview)

Azure Developer CLI (`azd`) enables developers to create apps from [templates](./azd-templates.md) stored in GitHub repositories. Microsoft provides [several templates](overview.md?branch=pr-en-us-3070&tabs=nodejs#azure-developer-cli-templates) to get you started. In this article, you learn how to enable your own project as a template.

## Understand the `azd` architecture

The following diagram gives a quick overview of the process to create an `azd` template:

:::image type="content" source="media/make-azd-compatible/workflow.png" alt-text="Diagram of Azure Developer CLI template workflow.":::

All `azd` templates have the same file structure, based on `azd` conventions. The following hierarchy shows the directory structure you'll build in this tutorial. For the complete directory structure, refer to the [azd conventions](#azd-conventions) section.

```txt
├── .github                    [ Configure GitHub workflow ]
├── infra                      [ Creates and configures Azure resources ]
│   ├── main.bicep             [ Main infrastructure file ]
│   ├── main.parameters.json   [ Parameters file ]
│   └── resources.bicep        [ Resources file ]
└── azure.yaml                 [ Describes the app and type of Azure resources]
```

## Create the project and source directories

::: zone pivot="azd-create"

1. Create a new directory for your project and change into it. This directory is your `project directory`.

1. Add your app source code either:

   - To the root of your project directory, or
   - In a subdirectory `src`, or `source directory`, which can be the same as your project directory.

::: zone-end

::: zone pivot="azd-convert"

1. Run the following command to clone the [Python Flask web app](/azure/app-service/quickstart-python?tabs=flask%2Cwindows%2Cazure-portal%2Cterminal-bash%2Cvscode-deploy%2Cdeploy-instructions-azportal%2Cdeploy-instructions-zip-azcli):

    ```bash
    git clone https://github.com/Azure-Samples/msdocs-python-flask-webapp-quickstart
    ```

   Optionally, to see the sample working, follow the instructions to [run the app locally](/azure/app-service/quickstart-python?tabs=flask%2Cwindows%2Cazure-portal%2Cterminal-bash%2Cvscode-deploy%2Cdeploy-instructions-azportal%2Cdeploy-instructions-zip-azcli#1---sample-app).

1. Change the current directory to `msdocs-python-flask-webapp-quickstart`.

::: zone-end

## Initialize a new environment

1. Run the following command to initialize the project. 

    ```bash
    azd init
    ```

1. Select **Empty Template** from the list of project templates.

1. Supply/select the appropriate values for your environment.

   | Parameter | Description |
   | --------- | ----------- |
   | `Environment Name` | Prefix for the resource group that will be created to hold all Azure resources. [What is an Environment Name in `azd`?](./faq.yml#what-is-an-environment-name) |
   | `Azure Location`   | The Azure location where your resources will be deployed. |
   | `Azure Subscription` | The Azure Subscription where your resources will be deployed. |

    **Key points**
    After you run `azd init`:

    - A directory called `.azure` is created.
    - Within the `.azure` directory, a directory is created: `<environment_name>`.
    - Within the `\.azure\<your environment_name>` directory, a file named `.env` is created.
    - The `.env` file contains information such as the values you supplied: 
      - Environment name
      - Location
      - Azure subscription
    - A file named `azure.yaml` is created in the root of your project.

## Add Bicep files

To provision the Azure resources, Bicep files need to be created within a directory called `infra`. In this section, you'll see how to perform this step by provisioning Azure App Service resources.

As this sample provisions App Service resources, you need:

- An Azure App Service Plan
- An Azure App Service running on Linux

For samples, refer to [sample Azure App Service Bicep files](/azure/app-service/samples-bicep). However, you can use the information in this section with [any supported host](./overview.md#supported-azure-compute-services-host).

1. Create a directory named `infra` in your project directory and change into it.

1. Create a new file named `main.parameters.json`. Insert the environment variables (found in the `.env` file in your project's `.azure/<environment_name>` directory). For example:

    ```json
    {
        "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
        "contentVersion": "1.0.0.0",
        "parameters": {
            "name": {
            "value": "${AZURE_ENV_NAME}"
            },
            "location": {
            "value": "${AZURE_LOCATION}"
            },
            "principalId": {
            "value": "${AZURE_PRINCIPAL_ID}"
            }
        }
    }
    ```

1. Create a file named `main.bicep` as the main entry point. Declare the parameters you included in `main.parameters.json`. 

   For more information, see [Parameters in Bicep](/azure/azure-resource-manager/bicep/parameters). You can also refer to the `main.bicep` of an [Azure Developer CLI template](./azd-templates.md) (such as [todo-nodejs-mongo template](https://github.com/Azure-Samples/todo-nodejs-mongo/blob/main/infra/main.bicep)) and remove the outputs you don't need. 

   For example:

    ```bicep
    targetScope = 'subscription'

    @minLength(1)
    @maxLength(50)
    @description('Name of the the environment which is used to generate a short unique hash used in all resources.')
    param name string

    @minLength(1)
    @description('Primary location for all resources')
    param location string

    @description('Id of the user or app to assign app roles')
    param principalId string = ''

    resource resourceGroup 'Microsoft.Resources/resourceGroups@2020-06-01' = {
        name: 'rg-${name}'
        location: location
        tags: tags
    }

    var resourceToken = toLower(uniqueString(subscription().id, name))
    var tags = {
        'azd-env-name': name
    }

    module resources './resources.bicep' = {
        name: 'resources-${resourceToken}'
        scope: resourceGroup
        params: {
            location: location
            principalId: principalId
            resourceToken: resourceToken
            tags: tags
        }
    }

    output APP_WEB_BASE_URL string = resources.outputs.WEB_URI
    output AZURE_LOCATION string = location
    
    ```
  
    In this sample, a unique string is generated based on subscription ID and used as a resource token. This token is appended to the name of all Azure resources created by azd. `azd` uses tags to identify resources so you can modify the names based on your organization's naming convention.

1. Create a file named `resources.bicep`.

1. Declare the following parameters:
    
    ```bicep
    param location string
    param principalId string = ''
    param resourceToken string
    param tags object
    param sku string = 'S1' 
    param linuxFxVersion string = 'PYTHON|3.8'
    ```
    
1. Add the following code, to add a tag named azd-service-name with the value of web to your Azure resource. `azd` uses this tag to determine what resource to deploy your application to. The value should match the name of your service as defined in [azure.yaml](#update-azureyaml).

    ```bicep
    tags: union(tags, {
      'azd-service-name': 'web'
      })
    ```

1. Add the following code for zip deployment.

    ```bicep
    resource appSettings 'config' = {
      name: 'appsettings'
      properties: {
        'SCM_DO_BUILD_DURING_DEPLOYMENT': 'true'
        }
      }
    ```

1. The following code represents a complete `resources.bicep` file that creates an Azure App Service for hosting a Python web app:

    ```bicep
    param location string
    param principalId string = ''
    param resourceToken string
    param tags object
    param sku string = 'S1' 
    param linuxFxVersion string = 'PYTHON|3.8'
    
    resource appServicePlan 'Microsoft.Web/serverfarms@2020-06-01' = {
      name: 'plan-${resourceToken}'
      location: location
      tags: tags
      sku: {
        name: sku
      }
      kind: 'linux'
      properties: {
      reserved: true
      }
    }
    
    resource web 'Microsoft.Web/sites@2020-06-01' = {
      name: 'app-web-${resourceToken}'
      location: location
      tags: union(tags, {
        'azd-service-name': 'web'
        })
      kind: 'app'
      properties: {
        serverFarmId: appServicePlan.id
        siteConfig: {
        linuxFxVersion: linuxFxVersion
        }
      }
    
      resource appSettings 'config' = {
        name: 'appsettings'
        properties: {
          'SCM_DO_BUILD_DURING_DEPLOYMENT': 'true'
          }
        }
      }
    
      output WEB_URI string = 'https://${web.properties.defaultHostName}'
    ```

1. Run the following command to provision the Azure resources.

    ```bash
    azd provision
    ```
   
    **Key points:**
    - After you run `azd provision`, the Azure resources are created under the resource group `rg-<environment_name>`.
    - The web end point is added to `.env` file in the project's `.azure/<environment_name>` directory.

## Update azure.yaml

To deploy the app, `azd` needs to know more about your app. Specify the app's source code location, the app type, and the Azure service that will be hosting your app in the `azure.yaml` file.

1. Edit `azure.yaml` by adding the following lines:

    ```yml
    name: msdocs-python-flask-webapp-quickstart
    services:
      web:
        project: .
        language: py
        host: appservice
    ```

    | Value | Description |
    | ----- | ----------- |
    | `name` | Root element. Required. Name of the app. |
    | `services` | Root element. Required. Definition of services that is part of the app. |
    | `web` | Required. Name of the service. Can be any name, for example, api, web. This name needs to be the same as the `azd-service-name` value you specified earlier. |
    | `project` | Required. Path to the service source code directory. Use **src/web** if your source code is found under /src/web. |
    | `language` | Service implementation language. `py` for Python. If not specified, .NET will be assumed. |
    | `host` | Type of Azure resource used for service implementation. "appservice" for Azure App Service. If not required, appservice is assumed. |

    For full details, refer to [azure.yaml.json](https://github.com/Azure/azure-dev/blob/main/schemas/v1.0/azure.yaml.json/).

1. Run the following command to deploy the app to Azure:

    ```bash
    azd deploy
    ```

    **Key points**
    After running `azd deploy`, the service **web** is deployed to the app service you previously provisioned.

1. Use your browser to open the end point to test your app.

Your project is now compatible with Azure Developer CLI and can be used as a template!

> [!NOTE]
> You can run `azd up` to perform both `azd provision` and `azd deploy` in a single step. If you wish to create a new environment, run `azd env new`.

## Configure a DevOps pipeline

1. Within your project directory, create a directory named `.github`.

1. Within the `.github` directory, create a directory named `workflows`.

1. Copy the **azure-dev.yml** file from any [azd template](./azd-templates.md) (for example, [todo-nodejs-mongo template](https://github.com/Azure-Samples/todo-nodejs-mongo/blob/main/.github/workflows/azure-dev.yml)) and paste as new file into the `.github/workflows` directory.

1. Run the following command to push updates to the repository. The GitHub Actions workflow is triggered because of the update.

    ```bash
    azd pipeline config    
    ```

1. Using your browser, go to the GitHub repository for your project.

1. Select **Actions** to see the workflow running.

::: zone pivot="azd-convert"

> [!NOTE]
> `.venv` should be added to the `.gitignore` file

::: zone-end

## Clean up resources

When you no longer need the resources created in this article, run the following command:

``` bash
azd down
```

## `azd` conventions

The following hierarchy shows the complete directory structure of an `azd` template.

```txt
├── .devcontainer              [ For DevContainer ]
├── .github                    [ Configure GitHub workflow ]
├── .vscode                    [ VS Code workspace ]
├── assets                     [ Assets used by README.MD ]
├── infra                      [ Creates and configures Azure resources ]
│   ├── main.bicep             [ Main infrastructure file ]
│   ├── main.parameters.json   [ Parameters file ]
│   └── resources.bicep        [ Resources file ]
├── src                        [ Contains directories for the app code ]
└── azure.yaml                 [ Describes the app and type of Azure resources]
```

## See also

- [Create Bicep files with Visual Studio Code](/azure/azure-resource-manager/bicep/quickstart-create-bicep-use-visual-studio-code?tabs=CLI) for an introduction to working with Bicep files.
- [Bicep Samples](/samples/browse/?languages=bicep)
- [How to decompile Azure Resource Manager templates (ARM templates) to Bicep](/azure/azure-resource-manager/bicep/decompile?tabs=azure-cli)

## Next steps

> [!div class="nextstepaction"]
> [Azure Developer CLI FAQ](./faq.yml)