# OpenAI end-to-end baseline reference implementation

This reference implementation illustrates an approach for authoring and running a chat application in a single region with Azure Machine Learning and OpenAI. It implements a secure environment for authoring a chat flow with Azure Machine Learning prompt flow and two options for deploying the flow:

- An Azure Machine Learning managed online endpoint in a managed virtual network.
  - If your application requires high availability and you favor leveraging a managed online endpoint, it is recommended to extend this architecture by deploying multiple online endpoints behind a load balancer to improve resiliency.
- A network-isolated, zone-redundant, highly available deployment in Azure App Service.

The implementation takes advantage of [Prompt flow](https://microsoft.github.io/promptflow/) in [Azure Machine Learning](https://azure.microsoft.com/products/machine-learning) to build and deploy flows that can link the following actions required by an LLM chat application:

- Creating prompts
- Querying data stores for grounding data
- Python code
- Calling Large Language Models (LLMs)

The reference implementation focuses on enterprise requirements such as:

- Network isolation
- Security
- Scalability

## Architecture

The implementation covers the following scenarios:

1. Authoring a flow - Authoring a flow using prompt flow in an Azure Machine Learning workspace
1. Deploying a flow to Azure Machine Learning (AML hosted option) - The deployment of an executable flow to an Azure Machine Learning online endpoint. The client UI that is hosted in Azure App Service accesses the deployed flow.
1. Deploying a flow to Azure App Service (Self-hosted option) - The deployment of an executable flow as a container to Azure App Service. The client UI that accesses the flow is also hosted in Azure App Service.

### Authoring a flow

![Diagram of the authoring architecture using Azure Machine Learning.](docs/media/azure-machine-learning-authoring.png)

The authoring architecture diagram illustrates how flow authors [connect to an Azure Machine Learning Workspace through a private endpoint](https://learn.microsoft.com/azure/machine-learning/how-to-configure-private-link) in a virtual network. In this case, the author connects to the virtual network through Azure Bastion and a virtual machine jumpbox. Connectivity to the virtual network is more commonly done in enterprises through ExpressRoute or virtual network peering.

The diagram further illustrates how the Machine Learning Workspace is configured for [Workspace managed virtual network isolation](https://learn.microsoft.com/azure/machine-learning/how-to-managed-network). With this configuration, a managed virtual network is created, along with managed private endpoints enabling connectivity to private resources such as the workplace Azure Storage and Azure Container Registry. You can also create user-defined connections like private endpoints to connect to resources like Azure OpenAI Service and Azure AI Search (previously named Cognitive Search).

### Deploying a flow to Azure Machine Learning managed online endpoint

![Diagram of the deploying a flow to Azure Machine Learning managed online endpoint.](docs/media/openai-chat-e2e-deployment-amlcompute.png)


The Azure Machine Learning deployment architecture diagram illustrates how a front-end web application, deployed into a [network-secured App Service](https://github.com/Azure-Samples/app-service-baseline-implementation), [connects to a managed online endpoint through a private endpoint](https://learn.microsoft.com/azure/machine-learning/how-to-configure-private-link) in a virtual network. Like the authoring flow, the diagram illustrates how the Machine Learning Workspace is configured for [Workspace managed virtual network isolation](https://learn.microsoft.com/azure/machine-learning/how-to-managed-network). The deployed flow is able to connect to required resources such as Azure OpenAI and Cognitive Search through managed private endpoints.

### Deploying a flow to Azure App Service (alternative)

![Diagram of the deploying a flow to Azure App Service.](docs/media/openai-chat-e2e-deployment-appservices.png)

The Azure App Service deployment architecture diagram illustrates how the same prompt flow can be containerized and deployed to Azure App Service alongside the same front-end web application from the prior architecture. This solution is a completely self-hosted, externalized alternative to an Azure Machine Learning managed online endpoint.

The flow is still authored in a network-isolated Azure Machine Learning workspace. To deploy in App Service in this architecture, the flows need to be containerized and pushed to the Azure Container Registry that is accessible through private endpoints to the App Service.

## Deploy

The following are prerequisites.

### Prerequisites

- Ensure you have an [Azure Account](https://azure.microsoft.com/free/)
- The deployment must be started by a user who has sufficient permissions to assign [roles](https://learn.microsoft.com/azure/role-based-access-control/built-in-roles), such as a User Access Administrator or Owner.
- Ensure you have the [Azure CLI installed](https://learn.microsoft.com/cli/azure/install-azure-cli)
- Ensure you have the [az Bicep tools installed](https://learn.microsoft.com/azure/azure-resource-manager/bicep/install)

Use the following to deploy the infrastructure.

### Deploy the infrastructure

The following steps are required to deploy the infrastructure from the command line.

1. In your shell, clone this repo and navigate to the root directory of this repository.

   ```bash
   git clone https://github.com/Azure-Samples/openai-end-to-end-baseline
   cd openai-end-to-end-baseline
   ```

1. Login and set subscription

```bash
az login
az account set --subscription xxxxx
```

1. Obtain the App gateway certificate
   Azure Application Gateway support for secure TLS using Azure Key Vault and managed identities for Azure resources. This configuration enables end-to-end encryption of the network traffic using standard TLS protocols. For production systems, you should use a publicly signed certificate backed by a public root certificate authority (CA). Here, we will use a self-signed certificate for demonstrational purposes.

   - Set a variable for the domain used in the rest of this deployment.

     ```bash
     export DOMAIN_NAME_APPSERV_BASELINE="contoso.com"
     ```

   - Generate a client-facing, self-signed TLS certificate.

     :warning: Do not use the certificate created by this script for actual deployments. The use of self-signed certificates are provided for ease of illustration purposes only. For your App Service solution, use your organization's requirements for procurement and lifetime management of TLS certificates, _even for development purposes_.

     Create the certificate that will be presented to web clients by Azure Application Gateway for your domain.

     ```bash
     openssl req -x509 -nodes -days 365 -newkey rsa:2048 -out appgw.crt -keyout appgw.key -subj "/CN=${DOMAIN_NAME_APPSERV_BASELINE}/O=Contoso" -addext "subjectAltName = DNS:${DOMAIN_NAME_APPSERV_BASELINE}" -addext "keyUsage = digitalSignature" -addext "extendedKeyUsage = serverAuth"
     openssl pkcs12 -export -out appgw.pfx -in appgw.crt -inkey appgw.key -passout pass:
     ```

   - Base64 encode the client-facing certificate.

     :bulb: No matter if you used a certificate from your organization or generated one from above, you'll need the certificate (as `.pfx`) to be Base64 encoded for proper storage in Key Vault later.

     ```bash
     export APP_GATEWAY_LISTENER_CERTIFICATE_APPSERV_BASELINE=$(cat appgw.pfx | base64 | tr -d '\n')
     echo APP_GATEWAY_LISTENER_CERTIFICATE_APPSERV_BASELINE: $APP_GATEWAY_LISTENER_CERTIFICATE_APPSERV_BASELINE
     ```

1. Update the infra-as-code/parameters file
  - Provide an admin password in jumpBoxAdminPassword parameter the  for the jump box; it must satisfy the [complexity requirements for Windows](https://learn.microsoft.com/en-us/windows/security/threat-protection/security-policy-settings/password-must-meet-complexity-requirements).
  - Provide the  base64 cert data from $APP_GATEWAY_LISTENER_CERTIFICATE_APPSERV_BASELINE variable into the appGatewayListenerCertificate parameter.

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
      "baseName": {
        "value": "REPLACE_ME"
      },
      "developmentEnvironment": {
        "value": false
      },
      "appGatewayListenerCertificate": {
        "value": "REPLACE_ME"
      },
      "jumpBoxAdminPassword": {
        "value": "REPLACE_ME"
      }
    }
}
```

1. Run the following command to create a resource group and deploy the infrastructure. Make sure:

   - The location you choose [supports availability zones (AZs)](https://learn.microsoft.com/azure/reliability/availability-zones-service-support) and the [model version](https://learn.microsoft.com/en-us/azure/ai-services/openai/concepts/models#standard-deployment-model-availability) gpt-35-turbo 0613 which is used in this deployment. Also make sure you have quota in the region for the resources being deployed particularly for App Service Plan SKU P2V2 which is used in this deployment. The following regions eastus, eastus2, francecentral, japaneast, swedencentral, switzerlandnorth, uksouth support both AZs and the model version.
   - The BASE_NAME contains only lowercase letters and is between 6 and 8 characters. Most resource names will include this text including resources like keyvault and storage account which need to be globally unique so provide this value accordingly.
   - You choose a valid resource group name.

```bash
LOCATION=REPLACE_ME
BASE_NAME=REPLACE_ME
RESOURCE_GROUP=REPLACE_ME

az group create -l $LOCATION -n $RESOURCE_GROUP

# This takes about 30 minutes to run.
az deployment group create -f ./infra-as-code/bicep/main.bicep \
  -g $RESOURCE_GROUP \
  -p @./infra-as-code/bicep/parameters.json \
  -p baseName=$BASE_NAME
```

### Create, test, and deploy a Prompt flow

1. Connect to the virtual network via Azure Bastion and the jump box (deployed as part of this solution) or through a force-tunneled VPN or virtual network peering that you manually configure. The username for the admin of the jumpbox is vmadmin and the password is what you provided in the parameters.json file.

1. Open the [Machine Learning Workspace](https://ml.azure.com/) in the jump box and choose your workspace. Ensure you have [enabled Prompt flow in your Azure Machine Learning workspace](https://learn.microsoft.com/azure/machine-learning/prompt-flow/get-started-prompt-flow?view=azureml-api-2#prerequisites-enable-prompt-flow-in-your-azure-machine-learning-workspace) by following the below steps.

1. Create a prompt flow connection to your gpt35 Azure OpenAI deployment. This will be used by the prompt flow you clone in the next step.
    1. Click on 'Prompt flow' in the left navigation in Machine Learning Studio
    1. Click on the 'Connections' tab and click 'Create' 'Azure OpenAI.'
    1. Fill out the properties:
        - Name: 'gpt35'   **Make sure you use this name.**
        - Provider: Azure OpenAI
        - Subscription Id: <Choose your subscription>
        - Azure OpenAI Account Names: <Choose the Azure OpenAI Account created in this deployment>
        - API Key: <Choose a key from 'Keys and endpoint' in your Azure OpenAI instance in the Portal>
        - API Base: <Choose the endpoint from 'Keys and endpoint' in your Azure OpenAI instance in the Portal>
        - API type: azure
        - API version: <Leave default>
1. Clone an existing prompt flow
    1. Click on 'Prompt flow' in the left navigation in Machine Learning Studio
    1. Click on the 'Flows' tab and click 'Create'
    1. Click 'Clone' under 'Chat with Wikipedia'
    1. Name it 'chat_wiki' and Press 'Clone'
    1. Set the 'Connection' and 'deployment_name' to 'gpt35'and set the max_tokens property of the deployment_name to 256, for the following steps:
        - extract_query_from_question
        - augmented_chat
    1. Save

1. Start compute session with advanced settings

   - Click Start with advanced settings
   - Select Compute instance in compute type
   - Choose the compute instance created by the Bicep and click next
   - Keep the default settings for Base Image settings and click next
   - Review your setting and click Apply and start compute session

1. Test the flow

   - Wait for the runtime to be created
   - Select the runtime in the UI
   - Click on 'Chat' on the UI
   - Enter the sample question in the textbox already or enter one from your own
   - You should get an appropriate response

### Deploy to Azure Machine Learning managed online endpoint

1. Create a deployment in the UI

   - Click on 'Deploy' in the UI
   - Choose 'Existing' Endpoint and select the one called _ept-<basename>_
   - Name the deployment ept-<basename>. **Make sure you name the deployment ept-<basename>. An App Service environment variable is set, assuming that naming convention**
   - Choose a small Virtual Machine size for testing and set the number of instances.
   - Press 'Review + Create'
   - Press 'Create'

### Publish the Chat front-end web app

The baseline architecture uses [run from zip file in App Service](https://learn.microsoft.com/azure/app-service/deploy-run-package). This approach has many benefits, including eliminating file lock conflicts when deploying.

> :bulb: Read through the next steps, but follow the guidance in the **Workaround** section.

To use run from zip, you do the following:

1. Create a [project zip package](https://learn.microsoft.com/azure/app-service/deploy-run-package#create-a-project-zip-package) which is a zip file of your project.
1. Upload that zip file to a location accessible to your website. This implementation uses private endpoints to connect to the storage account securely. The web app has a managed identity authorized to access the blob.
1. Set the environment variable `WEBSITE_RUN_FROM_PACKAGE` to the URL of the zip file.

In a production environment, you would likely use a CI/CD pipeline to:

1. Build your application
1. Create the project zip package
1. Upload the zip file to your storage account

The CI/CD pipeline would likely use a [self-hosted agent](https://learn.microsoft.com/azure/devops/pipelines/agents/agents?view=azure-devops&tabs=browser#install) that can connect to the storage account through a private endpoint to upload the zip. We have not implemented that here.

**Workaround**

We need a workaround to upload the file to the storage account because we have not implemented a CI/CD pipeline with a self-hosted agent. There are two workaround steps you need to do in order to manually upload the zip file using the portal.

1. The deployed storage account does not allow public access, so you will need to access it from a browser in the jump box.
1. You must authorize your user to upload a blob to the storage account if not already authorized.

Run the following to:

- Open the Azure portal in a browser inside the jump box.
- Navigate to the storage account and upload the zip file `./website/chatui.zip` to the existing `deploy` container
- After that navigate to the web app and restart it.

### Validate the web app

This section will help you to validate that the workload is exposed correctly and responding to HTTP requests.

#### Steps

1. Get the public IP address of the Application Gateway from the portal.

   > :book: The app team conducts a final acceptance test to be sure that traffic is flowing end-to-end as expected, so they place a request against the Azure Application Gateway endpoint.

1. Create an `A` record for DNS.

   > :bulb: You can simulate this via a local hosts file modification.  Alternatively, you can add a real DNS entry for your specific deployment's application domain name if permission to do so.

   Map the Azure Application Gateway public IP address to the application domain name. To do that, please edit your hosts file (`C:\Windows\System32\drivers\etc\hosts` or `/etc/hosts`) and add the following record to the end: `${APPGW_PUBLIC_IP} www.${DOMAIN_NAME_APPSERV_BASELINE}` (e.g. `50.140.130.120  www.contoso.com`)

1. Browse to the site (e.g. <https://www.contoso.com>).

   > :bulb: It may take up to a few minutes for the App Service to start properly. Remember to include the protocol prefix `https://` in the URL you type in your browser's address bar. A TLS warning will be present due to using a self-signed certificate. You can ignore it or import the self-signed cert (`appgw.pfx`) to your user's trusted root store.

## Deploying the flow to Azure App Service option

This is a second option for deploying the flow. With this option, you deploy the flow to Azure App Service instead of the managed online endpoint. At a high-level, you must do the following:

- Prerequisites - Ensure you have the prerequisites
- Download your flow - Download the flow from the Machine Learning Workspace
- Build the flow - Use the ```pf``` CLI to build your flow
- Build and push the image - Containerize the flow and push to your Azure Container Registry
- Publish the image to Azure App Service

### Prerequisites

The following are the requirements for building the image, pushing to ACR, and deploying to Azure App Service:

- az CLI
- Python
- Anaconda
- Promptflow pf CLI

Below are commands to create and activate a conda environment and install the promptflow tools. See [Set up your dev environment](https://microsoft.github.io/promptflow/how-to-guides/quick-start.html#set-up-your-dev-environment) for more information.

```bash
conda create --name pf python=3.11.4
conda activate pf
pip install promptflow promptflow-tools

# You will need to install the following if you build the docker image locally
pip install keyrings.alt
pip install bs4
```

### Download your flow

1. Open the Prompt flow UI in Azure Machine Learning Studio
1. Expand the 'Files' tab in the right pane of the UI
1. Click on the download icon to download the flow as a zip file

> :bulb: If you are using a jumpbox to connect to Azure Machine Learning workspace, when you download the flow, it will be downloaded to your jumpbox. You will either need to have the prerequisites installed on the jumpbox, or you will need to transfer the zip file to a system that has the prerequisites.

### Build the flow

> :bulb: This example assumes your flow has a connection to Azure OpenAI

1. Unzip the prompt flow zip file you downloaded
1. In your terminal, change the directory to the root of the unzipped flow
1. Create a folder called 'connections'
1. Create a file for each connection you created in the Prompt flow UI
    1. Make sure you name the file to match the name you gave the connection. For example, if you named your connection 'gpt35' in Prompt flow, create a file called 'gpt35.yaml' under the connections folder.
    1. Enter the following values in the file:

        ```bash
        $schema: https://azuremlschemas.azureedge.net/promptflow/latest/AzureOpenAIConnection.schema.json
        name: gpt35
        type: azure_open_ai
        api_key: "${env:OPENAICONNECTION_API_KEY}"
        api_base: "${env:OPENAICONNECTION_API_BASE}"
        api_type: "azure"
        api_version: "2023-07-01-preview"
        ```

        > :bulb: The App Service is configured with App Settings that surface as environment variables for ```OPENAICONNECTION_API_KEY``` and ```OPENAICONNECTION_API_BASE```.

1. Build the flow

    ```bash
    pf flow build --source ./ --output dist --format docker
    ```

    The following code will create a folder named 'dist' with a docker file and all the required flow files.

### Build and push the image

1. Ensure the requirements.txt in the dist/flow folder has the appropriate requirements. At the time of writing, they were as follows:

    ```bash
    promptflow[azure]
    promptflow-tools==0.1.0.b5
    python-dotenv
    bs4
    ```

1. Ensure the connections folder with the connection was created in the dist folder. If not, copy the connections folder, along with the connection file to the dist folder.

1. Make sure you have network access to your Azure Container Registry and have an RBAC role such as ACRPush that will allow you to push an image. If you are running on a local workstation, you can set ```Public network access``` to ```All networks``` or ```Selected networks``` and add your machine ip to the allowed ip list.

1. Build and push the container image

    Run the following commands from the dist folder in your terminal:

    ```azurecli
    az login

    NAME_OF_ACR="cr$BASE_NAME"
    ACR_CONTAINER_NAME="aoai"
    IMAGE_NAME="wikichatflow"
    IMAGE_TAG="1.1"
    FULL_IMAGE_NAME="$ACR_CONTAINER_NAME/$IMAGE_NAME:$IMAGE_TAG"

    az acr build -t $FULL_IMAGE_NAME -r $NAME_OF_ACR .
    ```

### Host the chat flow container image in Azure App Service

Perform the following steps to deploy the container image to Azure App Service:

1. Set the container image on the pf App Service

    ```azurecli
    PF_APP_SERVICE_NAME="app-$BASE_NAME-pf"
    ACR_IMAGE_NAME="$NAME_OF_ACR.azurecr.io/$ACR_CONTAINER_NAME/$IMAGE_NAME:$IMAGE_TAG"

    az webapp config container set --name $PF_APP_SERVICE_NAME --resource-group $RESOURCE_GROUP --docker-custom-image-name $ACR_IMAGE_NAME --docker-registry-server-url https://$NAME_OF_ACR.azurecr.io
    az webapp deployment container config --enable-cd true --name $PF_APP_SERVICE_NAME --resource-group $RESOURCE_GROUP
    ```

1. Modify the configuration setting in the App Service that has the chat UI and point it to your deployed promptflow endpoint hosted in App Service instead of the managed online endpoint.

    ```azurecli
    UI_APP_SERVICE_NAME="app-$BASE_NAME"
    ENDPOINT_URL="https://$PF_APP_SERVICE_NAME.azurewebsites.net/score"
    
    az webapp config appsettings set --name $UI_APP_SERVICE_NAME --resource-group $RESOURCE_GROUP --settings chatApiEndpoint=$ENDPOINT_URL
    az webapp restart --name $UI_APP_SERVICE_NAME --resource-group $RESOURCE_GROUP
    ```

1. Validate the client application that is now pointing at the flow deployed in a container still works

## Clean Up

After you are done exploring your deployed AppService reference implementation, you'll want to delete the created Azure resources to prevent undesired costs from accruing.

```bash
az group delete --name $RESOURCE_GROUP -y
az keyvault purge  -n kv-${BASE_NAME}
```
