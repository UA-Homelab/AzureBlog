+++
date = '2025-08-02T11:22:53+02:00'
draft = false
title = 'Use outputs from past deployments with Bicep'
tags = ['Bicep', 'Infrastructure as Code', 'ARM']

+++

<div style="color: #5e5b5bff; border-left: 4px solid #f39c12; background: #fffbe6; padding: 16px; margin: 16px 0; border-radius: 4px;">
  <strong>Disclaimer:</strong> This is <b>NOT</b> about using output from Bicep modules with other modules, but about using the output from Bicep deployments with completely independent Bicep deployments.
</div>

## Why I wrote this

I wanted know how to use the output of past Bicep deployments with new templates but couldn't find any information about this, as most results only talked about using module outputs. 

This was actually what gave me the idea to create this blog. I am hoping this will, at some point, help someone in the same situation.

## Why is this usefull?

You can reference all outputs from your past deployments by only knowing the deployment and output name.

In my role, I regularly deploy landing zones for a variety of customers, each with their own unique requirements. To streamline this process, I’ve developed templates for key areas like connectivity, management, identity and governance (management groups, policies and rbac). Some of the platform resources provisioned are later referenced in policies or other platform components. Example: A policy to send logs to a log analytics workspace in the management subscription will need the workspace as a parameter.

To maximize flexibility—especially since different customer teams often handle different parts of the landing zone—I maintain separate templates for each platform area, as well as for the governance elements. However, this approach introduces a challenge: resource names and configurations can vary from customer to customer, which would normally mean updating parameters across every template and increase human errors.

By leveraging outputs from previous deployments, I can keep each deployment fully independent, while only needing to adjust parameters for the templates that actually create those resources. This not only simplifies the process but also makes it much easier to manage variations between customers without sacrificing modularity or maintainability.

## How to use past deployment outputs

After some frustrating time searching for a solution, I was thinking that probably the deployments (that you can see on subscripions and resource groups) are also just ARM-Resources and therefore it should be possible to define them as existing resources in templates. 

It turns out that this was the solution! You can just add deployments as existing resources to your bicep file and then access all output values, that have been defined. Here is an example:

``` bicep

var deploymentName = '<insert_deployment_name>'

resource deployment 'Microsoft.Resources/deployments@2025-04-01' existing = {
    name: deploymentName
}

``` 