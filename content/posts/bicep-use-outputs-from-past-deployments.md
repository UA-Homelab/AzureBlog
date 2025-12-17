+++
date = '2025-12-16T11:22:53+02:00'
draft = false
title = 'Use outputs from past deployments with Bicep'
tags = ['Bicep', 'Infrastructure as Code', 'ARM']
+++

<div style="color: #5e5b5bff; border-left: 4px solid #f39c12; background: #fffbe6; padding: 16px; margin: 16px 0; border-radius: 4px;">
  <strong>Disclaimer:</strong> This is <b>NOT</b> about using output from Bicep modules with other modules, but about using the output from Bicep deployments with other, completely different Bicep deployments.
</div>

## Table of Contents

- [Why I wrote this](#why-i-wrote-this)
- [Why is this usefull?](#why-is-this-usefull)
- [How to use past deployment outputs](#how-to-use-past-deployment-outputs)
- [Example](#example)
  - [Create the log analytics workspace](#create-the-log-analytics-workspace)
  - [Use the deployments output as policy assignment parameters](#use-the-deployments-output-as-policy-assignment-parameters)
- [Conclusion](#conclusion)

## Why I wrote this

I wanted to know if I can use the output of past Bicep deployments with new templates but couldn't find any information about this, as most results only talked about using module outputs. 

This was actually the reason I decided to create this blog. I am hoping this will, at some point, help someone in the same situation.

## Why is this usefull?

You can reference all outputs from your past deployments by only knowing the deployment and output name.

In my role, I regularly deploy landing zones and workloads for a variety of customers, each with their own unique requirements. To streamline this process, I’ve developed templates for the platform landing zones like connectivity, management, identity and governance (management groups, policies and rbac). Some of the platform resources provisioned are later referenced in policies or other platform components. Example: A policy to send logs to a log analytics workspace in the management subscription will need the workspace as a parameter.

To maximize flexibility — especially since different customer teams often handle different parts of the landing zone — I maintain separate templates for each platform area, as well as for the governance elements. However, this approach introduces a challenge: resource names and configurations can and will vary from customer to customer, which would normally mean updating parameters across every template and increase human errors.

By using outputs from previous deployments, I can keep each deployment fully independent while only needing to adjust parameters for the templates that actually create the platform resources. This not only simplifies the process but also makes it easier to manage variations between customers without sacrificing modularity or maintainability.

## How to use past deployment outputs

After some frustrating time searching for a solution, I was thinking that probably the deployments (that you can see on subscripions and resource groups) are also just ARM-Resources and therefore it should be possible to define them as existing resources in templates. 

It turns out that this was the solution! You can just add deployments as existing resources to your bicep file and then access all output values, that have been defined. Here are some examples:

- Use a deployment from the same subscription you're deploying to:
  ``` bicep
  param deploymentName string = '<insert_deployment_name>'

  resource deployment string 'Microsoft.Resources/deployments@2025-04-01' existing = {
      name: deploymentName
  }
  ```
- Use a deployment from another subscripion:
  ``` bicep
  param deploymentName string = '<insert_deployment_name>'
  param subscriptionId string = '<insert_subscription_id'

  resource deployment 'Microsoft.Resources/deployments@2025-04-01' existing = {
      scope: subscription(subscriptionId)
      name: deploymentName
  }
  ```
- Use a deployment from a resource group in the same subscription your're deploying to:
  ``` bicep
  param deploymentName string = '<insert_deployment_name>'
  param resourceGroupName string = '<insert_resource_group_name'

  resource deployment 'Microsoft.Resources/deployments@2025-04-01' existing = {
      scope: resourceGroup(resourceGroupName)
      name: deploymentName
  }
  ```
- Use a deployment from a resource group in a different subscription:
  ``` bicep
  param deploymentName string = '<insert_deployment_name>'
  param subscriptionId string = '<insert_subscription_id'
  param resourceGroupName string = '<insert_resource_group_name'

  resource deployment 'Microsoft.Resources/deployments@2025-04-01' existing = {
      scope: resourceGroup(subscriptionId, resourceGroupName)
      name: deploymentName
  }
  ```

<br>
After declaring the existing deployment like that, you can access the outputs as follows:

``` bicep
var deploymentOutput = deployment.properties.outputs.<outputName>.value
```

## Example

As an example I wrote a bicep template that creates a log analytics workspace. Another template uses the output of this deployment to assign a policy that sends the subscriptions activity logs to to this workspace. 

The files can be found on my [GitHub](https://github.com/UA-Homelab/blogpost-code-examples/tree/main/01_bicep_use_outputs_of_past_deployments)

### Create the log analytics workspace

This is very straight forward. I used Azure Verified Modules (AVM) to create a log analytics workspace. The important part here is to define the outputs, that you want to use later, in the main.bicep.

``` bicep
targetScope = 'subscription'

param applicationName string = 'Monitor'
param location string

@allowed([
  'Prod'
  'Test'
  'Dev'
  'QA'
  'Sandbox'
  'Staging'
  'Demo'
])
param environment string
param index string = '001'
param logAnalyticsWorkspaceSkuName string = 'PerGB2018'

var resourceNameSuffix = toLower('${replace(applicationName, ' ', '')}-${environment}-${location}-${index}')
var logAnalyticsWorkspaceName = 'law-${resourceNameSuffix}'
var resourceGroupName = 'rg-${resourceNameSuffix}'
var tags = {
  Application: applicationName
  Environment: environment
  templateSource: 'azureblog.urdl.me'
}

resource resourceGroup 'Microsoft.Resources/resourceGroups@2021-04-01' = {
  name: resourceGroupName
  location: location
  tags: tags
}

module logAnalyticsWorkspace 'br/public:avm/res/operational-insights/workspace:0.14.2' = {
  name: 'Deploy-${logAnalyticsWorkspaceName}'
  scope: resourceGroup
  params: {
    name: logAnalyticsWorkspaceName
    location: location
    skuName: logAnalyticsWorkspaceSkuName
    tags: tags
  }
}

output logAnalyticsWorkspaceName string = logAnalyticsWorkspace.outputs.name
output logAnalyticsWorkspaceId string = logAnalyticsWorkspace.outputs.resourceId
output resourceGroupName string = resourceGroup.name
output resourceGroupId string = resourceGroup.id
output subscriptionId string = subscription().subscriptionId
```

### Use the deployments output as policy assignment parameters

Now the interesting part! We can use the deployment name, that is defined in the PowerShell command used to deploy the template, to access this deployments outputs.

``` bicep
targetScope = 'subscription'

param location string

@description('The subscription ID where the Log Analytics Workspace is deployed. If not provided, the current subscription ID will be used. Override this value, if the workspace is in a different subscription.')
param logAnalyticsWorkspaceSubscriptionId string = subscription().subscriptionId

@description('The name of the deployment that created the Log Analytics Workspace. The default value assumes the deployment name follows the pattern "Deploy-LogAnalyticsWorkspace-$(location)".')
param logAnalyticsWorkspaceDeploymentName string = 'Deploy-LogAnalyticsWorkspace-${location}'

var policyAssignmentDisplayName = 'Configure Azure Activity logs to stream to specified Log Analytics workspace'
var policyDefinitionId = '/providers/Microsoft.Authorization/policyDefinitions/2465583e-4e78-4c15-b6be-a36cbc7c8b0f'

resource existingLogAnalyticsDeployment 'Microsoft.Resources/deployments@2021-04-01' existing = {
  name: logAnalyticsWorkspaceDeploymentName
  scope: subscription(logAnalyticsWorkspaceSubscriptionId)
}

module assignPolicy 'modules/policyAssignmentSubscriptionScope.bicep' = {
  scope: subscription()
  params: {
    location: location
    name: guid(subscription().subscriptionId, policyDefinitionId, logAnalyticsWorkspaceDeploymentName, location)
    displayName: policyAssignmentDisplayName
    policyDefinitionId: policyDefinitionId
    parameters: {
      logAnalytics: {
        value: existingLogAnalyticsDeployment.properties.outputs.logAnalyticsWorkspaceId.value
      }
    }
  }
}

output policyAssignmentId string = assignPolicy.outputs.policyAssignmentId
output policyAssignmentName string = assignPolicy.outputs.policyAssignmentName
output policyAssignmentPrincipalId string = assignPolicy.outputs.policyAssignmentPrincipalId

```

## Conclusion

The ability to use the outputs of past deployments is a great feature that can come in very handy. 

This example was very simple, but if you deploy a whole connectivity subscription with Bastion, Azure Firewall and VPN-Gateway, you may need lots of parameters with following deployments that have dependencies (like route tables that need the firewall private IP, spoke VNETs that need the hubs resource id, NSGs that need the bastion subnet range, etc.). 

So instead of all those parameters, you only need two (the deployment name and subscription id) and can simply access the output values.