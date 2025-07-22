@lab.Title

@lab.ActivityGroup(initialsurvey)

===

<initial introduction>

## Sign in to the lab virtual machine (VM)

When the lab first launches, you'll need to set up the initial lab environment.

1. [] Sign in to Windows with the following lab credentials:

    | Item | Value |
    |:---------|:---------|
    | Username | **Lab User** |
    | Password | +++@lab.VirtualMachine(Win11-Pro-Base).Password+++ |

1. [] On the lab VM, open Microsoft Edge, then go to `https://portal.azure.com`.

1. [] Sign in with the following lab credentials:

    | Item | Value |
    |:---------|:---------|
    | Username | `@lab.CloudPortalCredential(User1).Username` |
    | Password | `@lab.CloudPortalCredential(User1).Password` |

1. [] Accept or skip the initial prompts to sign in.

===

# Lab Overview

This reference implementation illustrates an approach running a chat application and an AI orchestration layer in a single region. It uses Azure AI Agent service as the orchestrator and OpenAI foundation models. This repository directly supports the [Baseline end-to-end chat reference architecture](https://learn.microsoft.com/azure/architecture/ai-ml/architecture/baseline-openai-e2e-chat) on Microsoft Learn.

Follow this implementation to deploy an agent in [Azure AI Foundry](https://learn.microsoft.com/azure/ai-foundry/) and uses Bing for grounding data. You'll be exposed to common generative AI chat application characteristics such as:

- Creating agents and agent prompts
- Querying data stores for grounding data
- Chat memory database
- Orchestration logic
- Calling language models (such as GPT models) from your agent

This implementation builds off the [basic implementation](https://github.com/Azure-Samples/openai-end-to-end-basic), and adds common production requirements such as:

- Network isolation
- Bring-your-own Azure AI Agent service dependencies (for security and BC/DR control)
- Added availability zone reliability
- Limit egress network traffic with Azure Firewall

## Architecture

The implementation covers the following scenarios:

- [Setting up Azure AI Foundry to host agents](#setting-up-azure-ai-foundry-to-host-agents)
- [Deploying an agent into Azure AI Agent service](#deploying-an-agent-into-azure-ai-agent-service)
- [Invoking the agent from .NET code hosted in an Azure Web App](#invoking-the-agent-from-net-code-hosted-in-an-azure-web-app)

### Setting up Azure AI Foundry to host agents

Azure AI Foundry hosts Azure AI Agent service as a capability. Azure AI Agent service's REST APIs are exposed as an AI Foundry private endpoint within the network, and the agents' all egress through a delegated subnet which is routed through Azure Firewall for any internet traffic. This architecture deploys the Azure AI Agent service with dependencies hosted within your own Azure subscription. As such, this architecture includes an Azure Storage account, Azure AI Search instance, and an Azure Cosmos DB account specifically for the Azure AI Agent service to manage.

### Deploying an agent into Azure AI Agent service

Agents can be created via the Azure AI Foundry portal, [Azure AI Agents SDK](https://github.com/Azure/azure-sdk-for-net/tree/main/sdk/ai/Azure.AI.Agents.Persistent), or the [REST API](https://learn.microsoft.com/rest/api/aifoundry/aiagents/). The creation and invocation of agents are a data plane operation. Since the data plane to Azure AI Foundry is private, all three of those are restricted to being executed from within a private network connected to the private endpoint of Azure AI Foundry.

Ideally agents should be source-controlled and a versioned asset. You then can deploy agents in a coordinated way with the rest of your workload's code. In this deployment guide, you'll create an agent from the jump box to simulate a deployment pipeline which could have created the agent.


If using the Azure AI Foundry portal is desired, then the web browser experience must be performed from a VM within the network or from a workstation that has VPN access to the private network and can properly resolve private DNS records.

### Invoking the agent from .NET code hosted in an Azure Web App

A chat UI application is deployed into a private Azure App Service. The UI is accessed through Application Gateway (WAF). The .NET code uses the [Azure AI Agents SDK](https://github.com/Azure/azure-sdk-for-net/tree/main/sdk/ai/Azure.AI.Agents.Persistent) to connect to the workload's agent. The endpoint for the agent is exposed exclusively through the Azure AI Foundry private endpoint.


===

## Prerequisites

> :bulb: Note that the following prereqs have been completed already as part of this lab environment. We recommend reviewing the steps for learning purposes before proceeding to Exercise 1.

- An [Azure subscription](https://azure.microsoft.com/free/)

The subscription must have all the resource providers used in this deployment [registered](https://learn.microsoft.com/azure/azure-resource-manager/management/resource-providers-and-types#register-resource-provider).

* Microsoft.AlertsManagement
* Microsoft.App
* Microsoft.Bing
* Microsoft.CognitiveServices
* Microsoft.Compute
* Microsoft.DocumentDB
* Microsoft.Insights
* Microsoft.KeyVault
* Microsoft.ManagedIdentity
* Microsoft.Network
* Microsoft.OperationalInsights
* Microsoft.Search
* Microsoft.Storage
* Microsoft.Web

The subscription must have the following quota available in the region you choose.

* Application Gateways: 1 WAF_v2 tier instance
* App Service Plans: P1v3 (AZ), 3 instances
* Azure AI Search (S - Standard): 1
* Azure Cosmos DB: 1 account
* OpenAI model: GPT-4o model deployment with 50k tokens per minute (TPM) capacity
* DDoS Protection Plans: 1
* Public IPv4 Addresses - Standard: 4
* Standard DSv3 Family vCPU: 2
* Storage Accounts: 2

Your deployment user must have the following permissions at the subscription scope.

- Ability to assign [Azure roles](https://learn.microsoft.com/azure/role-based-access-control/built-in-roles) on newly created resource groups and resources. (E.g. User Access Administrator or Owner)
- Ability to purge deleted AI services resources. (E.g. Contributor or Cognitive Services Contributor)

The [Azure CLI installed](https://learn.microsoft.com/cli/azure/install-azure-cli)

The [OpenSSL CLI](https://docs.openssl.org/3.5/man7/ossl-guide-introduction/#getting-and-installing-openssl) installed.

===

### Exercise 1. :rocket: Deploy the infrastructure

The following steps are required to deploy the infrastructure from the command line using the bicep files from the repository.

1. [] In the Azure portal, select **Cloud Shell** from the global controls at the top of the page.

    !IMAGE[xq2vpkqc.jpg](instructions300210/xq2vpkqc.jpg)

1. [] In the **Welcome to Azure Cloud Shell** pane, select **Bash**.

    !IMAGE[x07ouf7n.jpg](instructions300210/x07ouf7n.jpg)

1. [] Select the **Subscription** dropdown menu, select **@lab.CloudSubscription.Name**, then select **Apply**.

    !IMAGE[tpzm45f5.jpg](instructions300210/tpzm45f5.jpg)

1. [] In Cloud Shell, clone the repo, then navigate to the root directory of the repository.
 
    +++git clone https://github.com/mbilalamjad/openai-end-to-end-baseline+++
    
    +++cd openai-end-to-end-baseline+++

1. [] Obtain the App gateway certificate

    Azure Application Gateway includes support for secure TLS using Azure Key Vault and managed identities for Azure resources. This configuration enables end-to-end encryption of the network traffic going to the web application.

    - [] Set a variable for the domain used in the rest of this deployment.

        +++DOMAIN_NAME_APPSERV="contoso.com"+++

    - [] Generate a client-facing, self-signed TLS certificate.

        > :warning: Do not use the certificate created by this script for production deployments. 
        >
        > The use of self-signed certificates is for illustration purposes only. For your chat application traffic, use your organization's requirements for procurement and lifetime management of TLS certificates, *even for development purposes*.

        Create the certificate that will be presented to web clients by Azure Application Gateway for your domain.

        +++openssl req -x509 -nodes -days 365 -newkey rsa:2048 -out appgw.crt -keyout appgw.key -subj "/CN=${DOMAIN_NAME_APPSERV}/O=Contoso" -addext "subjectAltName = DNS:${DOMAIN_NAME_APPSERV}" -addext "keyUsage = digitalSignature" -addext "extendedKeyUsage = serverAuth"+++

        +++openssl pkcs12 -export -out appgw.pfx -in appgw.crt -inkey appgw.key -passout pass:+++


    - [] Base64-encode the client-facing certificate.

        +++APP_GATEWAY_LISTENER_CERTIFICATE_APPSERV=$(cat appgw.pfx | base64 | tr -d '\n')+++

        +++echo APP_GATEWAY_LISTENER_CERTIFICATE_APPSERV: $APP_GATEWAY_LISTENER_CERTIFICATE_APPSERV+++

        >[!knowledge] Whether you used a certificate from your organization or generated one above, you'll need the certificate (as .pfx) to be Base64 encoded for storage in Key Vault.

1. [] Set the deployment location to one that [supports availability zones](https://learn.microsoft.com/azure/reliability/availability-zones-service-support) and has available quota.

    +++LOCATION=swedencentral+++

    >[!note] This deployment has been tested in the following locations: swedencentral, eastus, eastus2, switzerlandnorth. You might be successful in other locations as well.

1. [] Set the base name value that will be used as part of the Azure resource names for the resources deployed in this solution. 

    +++BASE_NAME=@lab.LabInstance.Id+++

    >[!note] All DNS names will include this text, so it must be unique. Set between 6 to 8 numbers or lowercase characters. 

1. [] Create a resource group and deploy the infrastructure.

    +++RESOURCE_GROUP=rg-chat-baseline+++

    +++az group create -l $LOCATION -n $RESOURCE_GROUP+++

    +++PRINCIPAL_ID=$(az ad signed-in-user show --query id -o tsv)+++

    +++az deployment group create -f ./infra-as-code/bicep/main.bicep -g $RESOURCE_GROUP -p appGatewayListenerCertificate=${APP_GATEWAY_LISTENER_CERTIFICATE_APPSERV} -p baseName=${BASE_NAME} -p yourPrincipalId=${PRINCIPAL_ID}+++

    >[!note] You might run into transient deployment failure, in case that happens, rerun the last command to redeploy which will incrementally deploy resource that failed to deploy previously.

1. [] Set an admin password for the jump box. Please note that while typing, the prompt will not display anything being entered and will not show any cursor movement. 

    Enter the following, then wait a few seconds before selecting **Enter**.

    +++@lab.VirtualMachine(Win11-Pro-Base).Password+++

    !IMAGE[rjjdphu5.jpg](instructions300210/rjjdphu5.jpg)

    >[!alert] The deployment may take around **35 minutes**.

1. [] To follow the deployment in more detail, in the Azure portal, select the portal menu icon in the upper-left corner, then select **Resource groups**.

    !IMAGE[d7d8uwm4.jpg](instructions300210/d7d8uwm4.jpg)

1. [] Select the **rg-chat-baseline** resource group.

1. [] Under the **Essentials** section, you can select the link under **Deployments** to view deployments in more detail, or periodically **Refresh** the page.

    !IMAGE[kdvtu4pb.jpg](instructions300210/kdvtu4pb.jpg)

1. [] Wait until the deployment completes before proceeding.

===

### Exercise 2. Deploy an agent in the Azure AI Agent service

To test this scenario, you'll be deploying an AI agent included in this repository. The agent uses a GPT model combined with a Bing search for grounding data. Deploying an AI agent requires data plane access to Azure AI Foundry. In this architecture, a network perimeter is established, and you must interact with the Azure AI Foundry portal and its resources from within the network.

The AI agent definition would likely be deployed from your application's pipeline running from a build agent in your workload's network, or could be deployed via singleton code in your web application. In this deployment, you'll create an agent from the jump box, which most closely simulates pipeline-based creation.

1. [] In the **rg-chat-baseline** resource group's **Overview** page, find and select the **vm-jump-box** virtual machine.

    !IMAGE[4sqlerhm.jpg](instructions300210/4sqlerhm.jpg)

1. [] On the top menu bar, select **Connect**, then select **Connect via Bastion**.

    !IMAGE[evhj5ghh.jpg](instructions300210/evhj5ghh.jpg)

1. [] Enter the following credentials set during deployment, then select **Connect**.

    | Item | Value |
    |:---------|:---------|
    | Username | `vmadmin` |
    | Password | `@lab.VirtualMachine(Win11-Pro-Base).Password` |

    >[!alert] Unless otherwise stated, the following steps are all performed within the **vm-jump-box** virtual machine.

1. [] In the newly launched Bastion tab, Select **Allow** on the Edge prompt to **See text and images copied to the clipboard**. 

    !IMAGE[84wja34h.jpg](instructions300210/84wja34h.jpg)

1. [] Select **Next** and **Accept** the prompts for Windows 11 personalization.

1. [] Once connected to the VM, select the **Start menu**, then find and select +++PowerShell+++.

1. [] In PowerShell, sign in to Azure, then select your target subscription.

    +++az login+++

1. [] In the **Sign in** dialog, select **Work or school account**, then select **Continue**.

    !IMAGE[3qksutwf.jpg](instructions300210/3qksutwf.jpg)

1. [] Sign in with your lab credentials:

    | Item | Value |
    |:---------|:---------|
    | Username | +++@lab.CloudPortalCredential(User1).Username+++ |
    | Password | +++@lab.CloudPortalCredential(User1).Password+++ |

1. [] In the **Automatically sign in...** dialog, select **No, this app only**.

1. [] Select **Enter** to choose the default Azure subscription.

1. [] Set the base name to the same value it was when you deployed the resources.

    +++$BASE_NAME="@lab.LabInstance.Id"+++

1. [] In your lab VM, NOT the Bastion tab, select the **Start menu**, select **Notepad**.

    >[!alert] Due to limitations, the following code blocks of `Type Text` can't be entered directly into the Bastion tab for **vm-jump-box**.
    >
    > Instead, you'll be using Type Text in the lab VM, then copying that data from the lab VM over to **vm-jump-box**.

1. [] In Notepad, enter the following code block:

    ```powershell
    $RESOURCE_GROUP="rg-chat-baseline"
    $AI_FOUNDRY_NAME="aif${BASE_NAME}"
    $BING_CONNECTION_NAME="bingaiagent${BASE_NAME}"
    $AI_FOUNDRY_PROJECT_NAME="projchat"
    $MODEL_CONNECTION_NAME="agent-model"
    $BING_CONNECTION_ID="$(az cognitiveservices account show -n $AI_FOUNDRY_NAME -g $RESOURCE_GROUP --query 'id' --out tsv)/projects/${AI_FOUNDRY_PROJECT_NAME}/connections/${BING_CONNECTION_NAME}"
    $AI_FOUNDRY_AGENT_CREATE_URL="https://${AI_FOUNDRY_NAME}.services.ai.azure.com/api/projects/${AI_FOUNDRY_PROJECT_NAME}/assistants?api-version=2025-05-15-preview"

    echo $BING_CONNECTION_ID
    echo $MODEL_CONNECTION_NAME
    echo $AI_FOUNDRY_AGENT_CREATE_URL
    ```

1. [] Copy all the content in Notepad.

1. [] Switch to the **vm-jump-box** tab in Edge, then select **Ctrl+V** or **right-click** to paste it in the PowerShell window.

1. [] Select **Paste anyway** on the Warning dialog.

    !IMAGE[mgx97jzk.jpg](instructions300210/mgx97jzk.jpg)

1. [] Select **Enter**.

    !IMAGE[nyubqgjj.jpg](instructions300210/nyubqgjj.jpg)

    >[!note] This generates some variables to set context within your jump box.

1. [] Switch back to the lab VM's Notepad window.

    !IMAGE[5w71b8tz.jpg](instructions300210/5w71b8tz.jpg)

1. [] Replace Notepad's content with the following code block:

    ```powershell
    # Use the agent definition on disk
    Invoke-WebRequest -Uri "https://github.com/Azure-Samples/openai-end-to-end-baseline/raw/refs/heads/main/agents/chat-with-bing.json" -OutFile "chat-with-bing.json"

    # Update to match your environment
    ${c:chat-with-bing-output.json} = ${c:chat-with-bing.json} -replace 'MODEL_CONNECTION_NAME', $MODEL_CONNECTION_NAME -replace 'BING_CONNECTION_ID', $BING_CONNECTION_ID

    # Deploy the agent
    az rest -u $AI_FOUNDRY_AGENT_CREATE_URL -m "post" --resource "https://ai.azure.com" -b @chat-with-bing-output.json

    # Capture the Agent's ID
    $AGENT_ID="$(az rest -u $AI_FOUNDRY_AGENT_CREATE_URL -m 'get' --resource 'https://ai.azure.com' --query 'data[0].id' -o tsv)"

    echo $AGENT_ID
    ```

1. [] Copy the all the content in Notepad.

1. [] Switch back to the **vm-jump-box** tab in Edge, then select **Ctrl+V** or **right-click** to paste it in the PowerShell window.

1. [] Select **Paste anyway** on the Warning dialog.

    !IMAGE[mgx97jzk.jpg](instructions300210/mgx97jzk.jpg)

1. [] Select **Enter**.

    !IMAGE[u7tvhzah.jpg](instructions300210/u7tvhzah.jpg)

    >[!note] This deploys the agent and simulates deployment through your pipeline from a network-connected build agent.

===

### (Optional) Exercise 3. Test the agent from the Azure AI Foundry portal in the playground.

>[!note] This exercise is optional.

Here, you'll test your orchestration agent by invoking it directly from the Azure AI Foundry portal's playground experience. The Azure AI Foundry portal is only accessible from your private network, so you'll do this from the **vm-jump-box** virtual machine.

1. [] In the **vm-jump-box** virtual machine, open Microsoft Edge.

    !IMAGE[qqf0v2b2.jpg](instructions300210/qqf0v2b2.jpg)

1. [] Confirm or cancel choices through Edge's first-time setup prompts.

1. [] Go to +++portal.azure.com+++.

1. [] Sign in with the following lab credentials:

    | Item | Value |
    |:---------|:---------|
    | Username | +++@lab.CloudPortalCredential(User1).Username+++ |
    | Password | +++@lab.CloudPortalCredential(User1).Password+++ |

1. [] In the Azure portal, select the portal menu icon in the upper-left corner, then select **Resource groups**.

    !IMAGE[d7d8uwm4.jpg](instructions300210/d7d8uwm4.jpg)

1. [] Select the **rg-chat-baseline** resource group.

1. [] Find and select the **projchat** Azure AI Foundry project.

    !IMAGE[zve2ukj6.jpg](instructions300210/zve2ukj6.jpg)

1. [] Select **Go to Azure AI Foundry portal**.

    !IMAGE[s2m1jbm0.jpg](instructions300210/s2m1jbm0.jpg)

    >[!knowledge] Alternatively, you can find your Azure AI Foundry accounts and projects going directly to +++ai.azure.com+++, bypassing the Azure portal.

1. [] Select **Agents** in the leftmost navigation.

1. [] Select **Baseline Chatbot Agent**.

    !IMAGE[0x6edc9d.jpg](instructions300210/0x6edc9d.jpg)

1. [] Under the **Setup** section that opens, select **Try in playground**.

    !IMAGE[6x1psoo8.jpg](instructions300210/6x1psoo8.jpg)

1. [] Enter a question that would require grounding data through recent internet content, such as a notable recent event or the weather today in your location.

    +++How's the weather in Atlanta, GA?+++

1. [] Observe the response.

    !IMAGE[s7fl3i5a.jpg](instructions300210/s7fl3i5a.jpg)

===

### Exercise 4. Publish the chat front-end web app

Workloads build chat functionality into an application. Those interfaces usually call APIs which in turn call into your orchestrator. This implementation comes with such an interface. You'll deploy it to Azure App Service using its [run from package](https://learn.microsoft.com/azure/app-service/deploy-run-package) capabilities.

In a production environment, you use a CI/CD pipeline to:

- Build your web application
- Create the project zip package
- Upload the zip file to your Storage account from a compute that is in or connected to the workload's virtual network.

>[!alert] For this deployment guide, you'll continue using **vm-jump-box** to simulate part of that process.

1. [] In **vm-jump-box**, reopen PowerShell from the task bar.

    !IMAGE[1gseu390.jpg](instructions300210/1gseu390.jpg)

1. [] In PowerShell, download the web UI.

    +++Invoke-WebRequest -Uri https://github.com/Azure-Samples/openai-end-to-end-baseline/raw/refs/heads/main/website/chatui.zip -OutFile chatui.zip+++

1. [] Upload the web application to Azure Storage, where the web app will load the code from.

    +++az storage blob upload -f chatui.zip --account-name "stwebapp${BASE_NAME}" --auth-mode login -c deploy -n chatui.zip+++

    !IMAGE[uzxb0vfk.jpg](instructions300210/uzxb0vfk.jpg)

1. [] Update the app configuration to use the agent you deployed.

    +++az webapp config appsettings set -n "app-${BASE_NAME}" -g $RESOURCE_GROUP --settings AIAgentId="${AGENT_ID}"+++

1. [] Restart the web app to load the site code and its updated configuation.

    +++az webapp restart --name "app-${BASE_NAME}" --resource-group $RESOURCE_GROUP+++

    !IMAGE[hu49ppuc.jpg](instructions300210/hu49ppuc.jpg)

===

### Exercise 5. Try it out! Test the deployed application that calls into the Azure AI Agent service

This section will help you to validate that the workload is exposed correctly and responding to HTTP requests. This will validate that traffic is flowing through Application Gateway, into your Web App, and from your Web App, into the Azure AI Foundry agent API endpoint, which hosts the agent and its chat history. The agent will interface with Bing for grounding data and an OpenAI model for generative responses.

1. [] Close the tab for the **vm-jump-box** virtual machine.

1. [] In the Azure portal, select **Cloud Shell** from the global controls at the top of the page, which should relaunch a Bash terminal.

    !IMAGE[xq2vpkqc.jpg](instructions300210/xq2vpkqc.jpg)

1. [] In Cloud Shell, get the public IP address of the Application Gateway.

    +++APPGW_PUBLIC_IP=$(az network public-ip show -g rg-chat-baseline -n "pip-@lab.LabInstance.Id" --query [ipAddress] --output tsv)+++
    
1. [] Print the public IP address:

    +++echo APPGW_PUBLIC_IP: $APPGW_PUBLIC_IP+++

    !IMAGE[zpv05uy6.jpg](instructions300210/zpv05uy6.jpg)

1. [] Copy and paste the value of **APPGW_PUBLIC_IP** into the following text box:

    @lab.TextBox(publicIp)

1. [] Minimize Microsoft Edge.

1. [] On the desktop, right-click **Notepad++**, then select **Run as administrator**.

    !IMAGE[my8bbh37.jpg](instructions300210/my8bbh37.jpg)

1. [] Select **Yes** in the **User Account Control** dialog.

1. [] If prompted to update Notepad++, select **No**.

1. [] In Notepad++, select **File** in the upper-left corner, then select **Open**.

1. [] In the **Open** window, select the empty space in the address bar path field to modify the file path.

    !IMAGE[5748zda1.jpg](instructions300210/5748zda1.jpg)

1. [] Enter `C:\Windows\System32\drivers\etc`, then select **Enter**.

    !IMAGE[n340wvei.jpg](instructions300210/n340wvei.jpg)

1. [] Select **hosts**, then select **Open** in the lower-right corner of the window.

    !IMAGE[pp1lwowq.jpg](instructions300210/pp1lwowq.jpg)

    >[!note] You'll simulate the creation of an **A record** via a local **hosts** file modification.

1. [] Below the existing lines in the file, enter a new line, then enter the following:

    `@lab.Variable(publicIp) www.contoso.com`

    !IMAGE[tcoznuh8.jpg](instructions300210/tcoznuh8.jpg)

    >[!note] This uses the IP address you entered in the text box above for **APPGW_PUBLIC_IP**.

1. [] Select **File**, then select **Save**.

    !IMAGE[855yevko.jpg](instructions300210/855yevko.jpg)

1. [] In Microsoft Edge, open a new tab, then go to `https://www.contoso.com`.

1. [] At the **Your connection isn't private** warning, select **Advanced**.

    !IMAGE[8k3gbjs6.jpg](instructions300210/8k3gbjs6.jpg)

1. [] Select **Continue to www.contoso.com (unsafe)**

    !IMAGE[p5rt9l9j.jpg](instructions300210/p5rt9l9j.jpg)

    >[!alert] It may take a few minutes for the App Service to start properly.

1. [] Ask a question that involves something that would only be known if the RAG process included context from Bing, such as recent weather or events.

    `What are the current mortgage interest rates in the US?`

    !IMAGE[am4z4v3l.jpg](instructions300210/am4z4v3l.jpg)

===

@lab.ActivityGroup(completionsurvey)


##**IMPORTANT**
**REPORTING NOTE:** These labs are hosted on the Skillable platform. Completion data is collected by CSU and then exported to Success Factors every Monday. SF requires another 1-3 days to process that data. The status for this Lab will be visible in Viva and Learning Paths, etc., by Thursday of next week.
 
Be sure to click **End Lab** to get credit for completing this lab.

>[!note] Please select **Submit your responses**, then select **Next** to proceed.

=== 

Congratulations!

You've successfully completed the lab. Select **End** to mark the lab as **Complete**.
