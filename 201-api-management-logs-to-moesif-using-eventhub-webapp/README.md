# Moesif API Analytics for Azure API Management
![Azure Public Test Date](https://azurequickstartsservice.blob.core.windows.net/badges/201-api-management-logs-to-moesif-using-eventhub-webapp/PublicLastTestDate.svg)
![Azure Public Test Result](https://azurequickstartsservice.blob.core.windows.net/badges/201-api-management-logs-to-moesif-using-eventhub-webapp/PublicDeployment.svg)

![Azure US Gov Last Test Date](https://azurequickstartsservice.blob.core.windows.net/badges/201-api-management-logs-to-moesif-using-eventhub-webapp/FairfaxLastTestDate.svg)
![Azure US Gov Last Test Result](https://azurequickstartsservice.blob.core.windows.net/badges/201-api-management-logs-to-moesif-using-eventhub-webapp/FairfaxDeployment.svg)

![Best Practice Check](https://azurequickstartsservice.blob.core.windows.net/badges/201-api-management-logs-to-moesif-using-eventhub-webapp/BestPracticeResult.svg)
![Cred Scan Check](https://azurequickstartsservice.blob.core.windows.net/badges/201-api-management-logs-to-moesif-using-eventhub-webapp/CredScanResult.svg)

[![Deploy To Azure](https://raw.githubusercontent.com/Azure/azure-quickstart-templates/master/1-CONTRIBUTION-GUIDE/images/deploytoazure.svg?sanitize=true)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FAzure%2Fazure-quickstart-templates%2Fmaster%2F201-api-management-logs-to-moesif-using-eventhub-webapp%2Fazuredeploy.json)
[![Deploy To Azure US Gov](https://raw.githubusercontent.com/Azure/azure-quickstart-templates/master/1-CONTRIBUTION-GUIDE/images/deploytoazuregov.svg?sanitize=true)](https://portal.azure.us/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FAzure%2Fazure-quickstart-templates%2Fmaster%2F201-api-management-logs-to-moesif-using-eventhub-webapp%2Fazuredeploy.json)
[![Visualize](https://raw.githubusercontent.com/Azure/azure-quickstart-templates/master/1-CONTRIBUTION-GUIDE/images/visualizebutton.svg?sanitize=true)](http://armviz.io/#/?load=https%3A%2F%2Fraw.githubusercontent.com%2FAzure%2Fazure-quickstart-templates%2Fmaster%2F201-api-management-logs-to-moesif-using-eventhub-webapp%2Fazuredeploy.json)    

Analyze API logs from Azure API Management using [Moesif API Analytics](https://www.moesif.com/?language=azure-api-management).
Does not require any modifications to existing Api Management APIs. Just make change to the `policy` XML. Now you can view rejected requests that never reach your underlying service.

The integration works by logging API calls to an Azure EventHub. An Azure WebJob reads the event hub and sends to Moesif
[More info on this integration](https://www.moesif.com/implementation/log-http-calls-from-azure-api-management?platform=azure-management).

## How to install

### 1. Start Azure Resource Deployment

Click the below button to start a Custom deployment with the Moesif Azure Resource Template.

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FMoesif%2Fazure-quickstart-templates%2Fmaster%2F201-api-management-logs-to-moesif-using-eventhub-webapp%2Fazuredeploy.json)

### 2. Configure Parameters

Within the Azure Custom Deployment, set the following properties:

![Create a Custom Deployment in Azure](https://docs.moesif.com/images/docs/integration/azure-api-management-create-custom-deployment.png)

* For the resource group, select the same resource group that your Azure APIM resides in. This ensures the APIM logger, `moesif-log-to-event-hub`, is automatically created for you. 

* Set Moesif Application Id to the one displayed after logging into your Moesif account. You can create a free one on [Moesif's website](https://www.moesif.com/?language=azure-api-management)

* Set _Existing Api Mgmt Name_ to the name of your Azure APIM instance.

Then, click _Review+create_ and create the template to start the deployment. 

> You can deploy Moesif to a different resource group or leave _Existing Api Mgmt Name_ blank, but you will need to manually create an [APIM logger](https://docs.microsoft.com/en-us/azure/api-management/api-management-log-to-eventhub-sample#policy-declaration) later. 

### 3. Add XML Policy

Within the Azure portal, navigate to your existing Azure API Management instance.
Then, add the below XML policies to all products or APIs that you want logged to Moesif. 

More info on editing APIM policies is available on the [Azure docs](https://docs.microsoft.com/en-us/azure/api-management/set-edit-policies)

```xml
<policies>
    <inbound>
        <set-variable name="message-id" value="@(Guid.NewGuid())" />
        <log-to-eventhub logger-id="moesif-log-to-event-hub" partition-id="0">@{
          var body = context.Request.Body?.As<string>(true);
          var MAX_BODY_SIZE_FOR_EH = 145000;
          var origBodyLen = (null != body) ? body.Length : 0;
          if (MAX_BODY_SIZE_FOR_EH < origBodyLen){
              body = body.Remove(MAX_BODY_SIZE_FOR_EH);
          }
          var headers = context.Request.Headers
                               .Where(h => h.Key != "Ocp-Apim-Subscription-Key")
                               .Select(h => string.Format("{0}: {1}", h.Key, String.Join(", ", h.Value).Replace("\"", "\\\"")))
                               .ToArray<string>();
          var headerString = (headers.Any()) ? string.Join(";;", headers) : string.Empty;
          var messageId = context.Variables["message-id"];
          var jwtToken = context.Request.Headers.GetValueOrDefault("Authorization","").AsJwt();
          var userId = (context.User != null && context.User.Id != null) ? context.User.Id : (jwtToken != null && jwtToken.Subject != null ? jwtToken.Subject : null);
          var companyId = "";
          var requestBody = (body != null ? System.Convert.ToBase64String(Encoding.ASCII.GetBytes(body)) : null);
          string metadata = $@"";
          var request = $@"
                    ""event_type"": ""request"",
                    ""message-id"": ""{messageId}"",
                    ""method"": ""{context.Request.Method}"",
                    ""uri"": ""{context.Request.Url}"",
                    ""user_id"": ""{userId}"",
                    ""company_id"": ""{companyId}"",
                    ""request_headers"": ""{headerString}"",
                    ""request_body"": ""{requestBody}"",
                    ""metadata"": ""{metadata}""
                ";
            return "{" + request + "}";
      }</log-to-eventhub>
    </inbound>
    <backend>
        <forward-request follow-redirects="true" />
    </backend>
    <outbound>
        <log-to-eventhub logger-id="moesif-log-to-event-hub" partition-id="1">@{
          var body = context.Response.Body?.As<string>(true);
          var MAX_BODY_SIZE_FOR_EH = 145000;
          var origBodyLen = (null != body) ? body.Length : 0;
          if (MAX_BODY_SIZE_FOR_EH < origBodyLen){
              body = body.Remove(MAX_BODY_SIZE_FOR_EH);
          }
          var headers = context.Response.Headers
                                          .Select(h => string.Format("{0}: {1}", h.Key, String.Join(", ", h.Value).Replace("\"", "\\\"")))
                                          .ToArray<string>();
          var headerString = (headers.Any()) ? string.Join(";;", headers): string.Empty;
          var messageId = context.Variables["message-id"];
          var responseBody = (body != null ? System.Convert.ToBase64String(Encoding.ASCII.GetBytes(body)) : null);
          var response = $@"
                    ""event_type"": ""response"",
                    ""orig_body_len"": ""{origBodyLen}"",
                    ""message-id"": ""{messageId}"",
                    ""status_code"": ""{context.Response.StatusCode}"",
                    ""response_headers"": ""{headerString}"",
                    ""response_body"": ""{responseBody}""
                    ";
          return "{" + response + "}";
     }</log-to-eventhub>
    </outbound>
    <on-error>
        <base />
    </on-error>
</policies>
```

## Config Options

### UserId
(optional) => string, A userId enables Moesif to attribute API requests to individual unique users so you can understand who calling your API. This can be used simultaneously with companyId to track both individual customers and the companies they're a part of.

### CompanyId
(optional) => string, If your business is B2B, this enables Moesif to attribute API requests to specific companies or organizations so you can understand which accounts are calling your API. This can be used simultaneously with userId to track both individual customers and the companies they're a part of.

### Metadata
(optional) => string, An object that allows you to add custom metadata that will be associated with the event. The metadata must be a dictionary that can be converted to JSON. For example, you may want to save a VM instance_id, a trace_id, or a tenant_id with the request.

The [ARM Template `nested/microsoft.apimanagement/service/apis/policies.json`](nested/microsoft.apimanagement/service/apis/policies.json) can be used as a reference to set policy XML thru Azure deployment. This sample template, as implemented, will overwrite existing policy.xml
That's it! API logs should start showing up after a few minutes.

## Manual creation

If you want to create the resources directly, follow [Microsoft's documentation](https://docs.microsoft.com/en-us/azure/api-management/api-management-log-to-eventhub-sample) on configuring Moesif. The example WebJob is available [on GitHub](https://github.com/Moesif/ApimEventProcessor).

## About this template:
This template performs the following tasks

- Create Azure Eventhub Namespace, an Eventhub and a policy for Send and for Listen
- If an existing Azure Api Management instance is provided, create a new logger `log-to-eventhub` named `moesif-log-to-event-hub` for use in Api Management policy xml.
- Create Storage account to store eventhub consumption checkpoints
- Create Azure App Service plan and an Azure App Service. Sets environment variables for use by Webjob
- Create a Webjob (Continuous) that listens for eventhub events (Api Management request/responses) and forwards events to Moesif.

- The Webjob pulls [Moesif/ApimEventProcessor](https://github.com/Moesif/ApimEventProcessor), builds it, and runs it using preset environment variable parameters. (At present, this webjob is not automatically run. It can be manually launched by deploying arm template [nested/microsoft.web/sites/extensions.json](nested/microsoft.web/sites/extensions.json) and providing it path to zip file [scripts/apim-2-moesif-webjob-webdeploy.zip](scripts/apim-2-moesif-webjob-webdeploy.zip))

If an existing Azure Api Management is not specified, the `log-to-eventhub` logger is not automatically created. To manually create this logger, utilize the [`nested/microsoft.apimanagement/service/loggers.json` ARM template](nested/microsoft.apimanagement/service/loggers.json)

## Troubleshooting

- It is possible that the final step of deployment `app-service-webjob-msdeploy` reports a failed deployment with error such as `conflict` or `BadRequest` or `GatewayTimeout`. Despite these initial errors reported by deployment, it is possible that the WebJob deployment within Azure App Service may succeed automatically after 5-10 minutes without further user action. At the end of this template deployment, you may view detailed logs in your App Service/Activity log.
- Ensure the `log-to-eventhub` logger is created
- Ensure the `policy` is set on Api Management Apis
- Ensure App Service configuration contains correct environment variables. View your App Service/Settings/Configuration/Application settings
- Review the logs of App Service Webjob named `azure-api-mgmt-logs-2-moesif` and ensure it is running. View your App Service/Settings/WebJobs 

`Tags: Azure API Management, API Management, EventHub, Event Hub, API Gateway, Monitoring, Analytics, Observability, Logs, Logging, API Monitoring, API Analytics, API Logs, API Logging, Moesif, Kong, Tyk, Envoy, WebApp, WebJob, App`

## Updating the integration

If you need to update [Moesif/ApimEventProcessor](https://github.com/Moesif/ApimEventProcessor) and don't want to redeploy the entire template, you can follow these steps:

Before starting, make sure you fork the repo [ApimEventProcessor](https://github.com/Moesif/ApimEventProcessor), so it's in your GitHub account. 

1. Log into your Azure Portal and navigate to the resource group holding your Moesif resources. 

2. Select the WebApp and then click the Deployment Center panel on the left side. 
   
3. This will open the deployment panel as shown below, you'll want to click on GitHub.

![Redeploy Webjob GitHub](https://docs.moesif.com/images/docs/integration/azure-api-management-redeploy-github.png)
   
4. Click on _App Service build service_ (via Kudu) deployment

![Redeploy Webjob Kudu](https://docs.moesif.com/images/docs/integration/azure-api-management-redeploy-kudu.png)

5.  Select the repo you forked earlier and finish the walkthrough. 

Deployment may take a few minutes. 

> Double check your XML policy if there are any changes. 
