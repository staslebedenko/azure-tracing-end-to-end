# Azure custom tracing end-to-end.
Azure tracing snippets request => API Management => Functions => Service Bus => Logic App
Use this flow to win a blame game after another issue :).

# APIM part

You need to add policiy below and register Azure functions as backend in API Management.
Policy with generate GUID for tracing and add it to the header that function code will access later.

```xml  
<policies>
    <inbound>
        <base />
        <set-backend-service id="apim-generated-policy" backend-id="azure-function" />
        <set-header name="custCorrelationId" exists-action="override">
            <value>@(Guid.NewGuid().ToString())</value>
        </set-header>
    </inbound>
    <backend>
        <base />
    </backend>
    <outbound>
        <base />
    </outbound>
    <on-error>
        <base />
    </on-error>
</policies>
```

# Azure function
In Azure function you need to read request header, log it to application insights and then either add it to the object field, or if you experinting just add correlation id directly.

```csharp  
 public static class FunctionTraces
    {
        [FunctionName("apimtosbus")]
        public static async Task<IActionResult> RunAsync(
            [HttpTrigger(AuthorizationLevel.Anonymous, "get", "post", Route = null)] HttpRequest req,
            [ServiceBus("tracing", Connection = "ServiceBusConnection")] IAsyncCollector<string> serviceBusMessages,
            ILogger log)
        {
            string name = req.Query["name"];
            string custCorrelationId = StringValues.IsNullOrEmpty(req.Headers["custCorrelationId"]) ? "no_correlationId" : req.Headers["custCorrelationId"];

            var message = new { custCorrelationId };
            string jsonMessage = JsonConvert.SerializeObject(message);

            await serviceBusMessages.AddAsync(jsonMessage);

            log.LogInformation($"Function processed data with custCorrelationId {custCorrelationId}");

            return custCorrelationId != null
                   ? (ActionResult)new OkObjectResult($"Request with {custCorrelationId} is processed")
                   : new BadRequestObjectResult("Operation failed, please try again later");
        }
    }
```

# Service bus
Service bus is configured to use Queue named "tracing" as you can see in Azure Function output trigger above.
So all messages will be put to Service Bus Queue and polled by Logic App from there.

# Logic App
Besides the selection of a proper Azure Logic App trigger for Azure Service Bus Queue you need to process message body and get correlation Id from it with help of the following line below.
 "clientTrackingId": "@{json(base64ToString(triggerBody()?['ContentData'])).custCorrelationId}"

```json  
{
    "definition": {
        "$schema": "https://schema.management.azure.com/providers/Microsoft.Logic/schemas/2016-06-01/workflowdefinition.json#",
        "actions": {
            "Compose": {
                "inputs": "@json(base64ToString(triggerBody()?['ContentData'])).custCorrelationId",
                "runAfter": {},
                "trackedProperties": {},
                "type": "Compose"
            }
        },
        "contentVersion": "1.0.0.0",
        "outputs": {},
        "triggers": {
            "When_a_message_is_received_in_a_queue_(auto-complete)": {
                "correlation": {
                    "clientTrackingId": "@{json(base64ToString(triggerBody()?['ContentData'])).custCorrelationId}"
                },
                "inputs": {
                    "host": {
                        "connection": {
                            "referenceName": "servicebus"
                        }
                    },
                    "method": "get",
                    "path": "/@{encodeURIComponent(encodeURIComponent('tracing'))}/messages/head",
                    "queries": {
                        "queueType": "Main"
                    }
                },
                "recurrence": {
                    "frequency": "Minute",
                    "interval": 5
                },
                "type": "ApiConnection"
            }
        }
    },
    "kind": "Stateful"
}
```

# App insight and Log Analytics
Results of this custom tracing flow can be access via standard UI of Application Insights transaction search or Log Analytics workspace.
If you will try to find correlation Id of failed Logic App execution, you can do so by using RunId of logic app in custom query.
```kusto  
LogicAppWorkflowRuntime
| where RunId == "324328437439499383UC99324"
```  

