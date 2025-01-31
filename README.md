Heroku Integration - Extending Apex, Flow and Agentforce - Java
===============================================================

> [!IMPORTANT]
> For use with the Heroku Integration and Heroku Eventing pilots only

This sample demonstrates importing a Heroku application into an org to enable Apex, Flow, and Agentforce to call out to Heroku. For Apex, both synchronous and asynchronous invocation are demonstrated, along with securely elevating Salesforce permissions for processing that requires additional object or field access.

<img src="images/overview.jpg" width="80%" alt="Flow">

The scenario used in this sample illustrates a basis for performing complex compute calculations over Salesforce **Opportunity** data and storing the result back in Salesforce as a **Quote**. Calculating Quote information from Opportunities can become quite intensive, especially when large multinational businesses have complex rules that impact pricing related to region, products, and discount thresholds. It's also possible that such code already exists, and there is a desire to reuse it within a Salesforce context.

Requirements
------------
- Heroku login
- Heroku Integration Pilot enabled
- Heroku CLI installed
- Heroku Integration Pilot CLI installed
- Salesforce CLI installed
- Login information for one or more Scratch, Development or Sandbox orgs

## Local Development and Testing

Code invoked from Salesforce requires specific HTTP headers to connect back to the invoking Salesforce org. Using the `invoke.sh` script supplied with this sample, it is possible to simulate requests from Salesforce with the correct headers, enabling you to develop and test locally before deploying to test from Apex, Flow, or Agentforce. This sample leverages the `sf` CLI to allow the `invoke.sh` script to access org authentication details. Run the following commands to locally authenticate, build and run the sample:

```
sf org login web --alias my-org
mvn clean install
mvn spring-boot:run
```

In a new terminal window run the following command substituting `006am000006pS6P` for a valid **Opportunity Id** record from your Salesforce org, ensuring you identify an **Opportunity** that also has related **Product** line items.

```
./bin/invoke.sh my-org '{"opportunityId": "006am000006pS6P"}'
```

You should see the following output:

```
Response from server:
{"quoteId":"0Q0am000000nRLdCAM"}
```

You can now also view the **Quote** by refreshing the **Opportunity** page within Salesforce.

## Deploying and Testing from Apex and Flow

To test from Apex, Flow and other tools within your Salesforce org you must deploy the code and import it into your org. The following commands create a Heroku application and configure the Heroku Integration add-on. This add-on and associated buildpack allows secure authenticated access from within your code and visibility of your code from Apex, Flow and Agentforce. After this configuration, code is not accessible from the public internet, only from within an authorized Salesforce org.

```
heroku create
git push heroku main
```

Next install and configure the Heroku Integration add-on:

```
heroku addons:create heroku-integration
heroku buildpacks:add https://github.com/heroku/heroku-buildpack-heroku-integration-service-mesh
heroku salesforce:connect my-org --store-as-run-as-user
heroku salesforce:import api-docs.yaml --org-name my-org --client-name GenerateQuote
```

Trigger an application rebuild to install the Heroku Integration buildpack

```
git commit --allow-empty -m "empty commit"
git push heroku main
```

Once imported grant permissions to users to invoke your code using the following `sf` command:

```
sf org assign permset --name GenerateQuote -o my-org
```

Deploy the Heroku application and confirm it has started.

```
git push heroku main
heroku logs
```

Navigate to your orgs **Setup** menu and search for **Heroku** then click **Apps** to confirm your application has been imported.

### Invoking from Apex

Now that you have imported your Heroku application. The following shows an Apex code fragment the demonstrates how to invoke your code in an synchronous manner (waits for response). Make sure to change the **Opportunity Id** `006am000006pS6P` below to a valid **Opportunity** from your org (see above).

```
echo \
"ExternalService.GenerateQuote service = new ExternalService.GenerateQuote();" \
"ExternalService.GenerateQuote.generateQuote_Request request = new ExternalService.GenerateQuote.generateQuote_Request();" \
"ExternalService.GenerateQuote_QuoteGenerationRequest body = new ExternalService.GenerateQuote_QuoteGenerationRequest();" \
"body.opportunityId = '006am000006pS6P';" \
"request.body = body;" \
"System.debug('Quote Id: ' + service.generateQuote(request).Code200.quoteId);" \
| sf apex run -o my-org
```

Inspect the debug log output sent to to the console and you should see the generated Quote ID output as follows:

```
07:56:11.212 (3213672014)|USER_DEBUG|[1]|DEBUG|Quote Id: 0Q0am000000nRS5CAM
```

### Invoking from Flow

Salesforce Flow is the primary no code automation tool within Salesforce orgs and is regularly used to automate business tasks related to Accounts, Opportunities, and more. Flow builders, like Apex developers, can also benefit from additional compute power provided through your Heroku applications. There is no additional work required to enable Flow Builder to see your code-simply search for it using the Action element, as shown below.

<img src="images/flow.jpg" width="50%" alt="Flow">

To test invoking your code from Flow, deploy a simple Flow using the command below:

```
sf project deploy start --metadata Flow -o my-org
```

Open the Flow called **Generate Quote Test** and click the **Debug** button entering a valid **Opportunity Id**. Once the Flow completes inspect the contents of the output variable and you should see the now familiar **Quote Id** present. 

<img src="images/flowdebug.jpg" width="50%" alt="Flow Debug">

The Flow used here is a very basic Autolaunched Flow, however you can use this approach in other Flow types such as Screen Flows as well - allowing you to build user experiences around your Heroku applications with Flow for example.

## Permissions and Permission Elevation (Optional Advanced Topic)

Authenticated connections passed to your code are created with the user identity of the user that directly or indirectly causes a given Apex, Flow, or Agentforce operation to invoke your code. As such, in contrast to Apex, your code always accesses org data in the more secure User mode (only the permissions granted to the user), not System mode (access to all data). This favors the more secure coding convention of the principle of least privilege. If your code needs to access or manipulate information not visible to the user, you must use the approach described here to elevate permissions.

For this part of the sample we expand the code logic to detect a discount override field that only sales leaders would have access to. This field completely overrides the discount calculations for some opportunity product lines. Since by design not all users should have access to this field, in order to allow only the code to access this field it is necessary to configure elevated permissions. Run the command below to deploy a new custom field on the `OpportunityLineItem` called `DiscountOverride__c`. 

```
sf project deploy start --metadata CustomObject:OpportunityLineItem -o my-org
```

First let's understand the behavior without elevated permissions when the code tries to access this field with only the permissions of the user. Create a `.env` file in the root of the project with the following contents and then rerun the application locally using `mvn spring-boot:run` as described earlier.

```
ENABLE_DISCOUNT_OVERRIDES=True
```

Use the `invoke.sh` script to attempt to generate a new **Quote**. The request will fail with a `503` and you will see in the console output the following error:

```
No such column 'DiscountOverride__c' on entity 'OpportunityLineItem'. If you are attempting to use a custom field, be sure to append the '__c' after the custom field name. Please reference your WSDL or the describe call for the appropriate names.'
```

Replicate this situation with your deployed code, by enabling discount overrides using `heroku config:set ENABLE_DISCOUNT_OVERRIDES=True`. Retry with the Apex invocation example above while monitoring the Heroku logs using `heroku logs --tail`. Once again in the Heroku debug logs you will see the above error. This is because your user does not have access to the `DiscountOverride__c` field - as per our scenario, this is a valid use case.

In order to elevate just the permissions of your code to access this field and not that of the user, an additional **Permission Set** is needed. This Permission Set contains only additional permissions the code needs - in this case access to the `DiscountOverride__c` field. The following command deploys a permission set called `GenerateQuoteAuthorization` that contains this permission. Note this permission set is defined as requiring session activation to be active, this temporary activation is handled by the Heorku Integration add-on for you. However it must still be assigned to a user and must following the naming convention of `[yourapplication]Authorization` for the **Heroku Integration** add-on to detect it.

```
sf project deploy start --metadata PermissionSet -o my-org
sf org assign permset --name GenerateQuoteAuthorization -o my-org
```

Now that this has been deployed, the **Heroku Integration** add-on automatically detects it and adds it to the users own permissions while executing the code - hence permissions are now elevated. To test this rerun the code using Apex invocation examples above and you should now find that a **Quote** has been successfully created as evident from the console output per the example below.

```
07:56:11.212 (3213672014)|USER_DEBUG|[1]|DEBUG|Quote Id: 0Q0am000000nRS5CAM
```

When running developing and testing locally the `invoke.sh` can take a third argument to emulate the above deployed behavior. Also note that at time of writing, during the Pilot, **Flow** and **Agentforce** invocation with elevated permissions is not supported and returns an error.

> [!NOTE]
> Your developer user needs permissions to assign session based permission sets required by the `invoke.sh` script. Before running the above command assign this permission using `sf org assign permset --name GenerateQuoteAuthorization -o my-org`. You need only run it once. 

```
./bin/invoke.sh my-org '{"opportunityId": "006am000006pS6P"}' GenerateQuoteAuthorization
```

The above `invoke.sh` now outputs additional information confirming the elevation:

```
Activating session-based permission set: GenerateQuoteAuthorization...
Session-based permission set activated. Activation ID: 5Paam00000O6m16CAB
Response from server:
{"quoteId":"0Q0am000000nSZRCA2"}
Deactivating session-based permission set: GenerateQuoteAuthorization...
Session-based permission set deactivated successfully.
```

## Invoking from Agentforce

Consult the more in-depth Heroku and Agentforce Tutorial [here](https://github.com/heroku-examples/heroku-agentforce-tutorial/tree/heroku-integration-pilot). 

## Technical Information
- The pricing engine logic is implemented in the `PricingEngineSevice.java` source file, under the `/src` directory. It demonstrates how to query and insert records into Salesforce.
- The `api-docs.yaml` file is not automatically refreshed. If you change the request or response of your code this needs to be updated before importing into Salesforce. To do this, when developing locally navigate to `http://localhost:8080/v3/api-docs.yaml` to download the latest generated version. It is feasible to automate this process with a shell script that wraps the `heroku salesforce:import` command to avoid maintaining this file.
- At present this Java sample does not support asynchronous invocation, please consult the Node version for this.
- The [Salesforce WSC Java](https://github.com/forcedotcom/wsc) client is used to simplify API communications with the org, this is initialized in the Spring Boot filter, `SalesforceClientContextFilter.java` by decoding information within the `x-client-context` HTTP header. The code accesses this via `PartnerConnection connection = (PartnerConnection) httpServletRequest.getAttribute("salesforcePartnerConnection");`. The HTTP header `x-client-context` will be documented more fully in the future, please consult this sample code in the meantime.
- Source code for configuration/metadata deployed to Salesforce can be found in the `/src-org` directory.
- Requests in this type of Heroku application are being managed by a web process that implements a strict timeout as described [here](https://devcenter.heroku.com/articles/request-timeout) you will see errors in the Apex debug logs only. If you are hitting this limit consult the [Scaling Batch Jobs with Heroku - Java](https://github.com/heroku-examples/heroku-integration-pattern-org-job-java) sample.
- Per the **Heroku Integration** add-on documentation and steps above, the service mesh buildpack must be installed to enable authenticated connections to be intercepted and be passed through to your code. To configure this the `Procfile` and Java Spring boot, `server.port` configuration has been updated (see `application.properties`).

## Other Samples

| Sample | What it covers? |
| ------ | --------------- |
| [Salesforce API Access - Java](https://github.com/heroku-examples/heroku-integration-pattern-api-access-java) | This sample application showcases how to extend a Heroku web application by integrating it with Salesforce APIs, enabling seamless data exchange and automation across multiple connected Salesforce orgs. It also includes a demonstration of the Salesforce Bulk API, which is optimized for handling large data volumes efficiently. |
| [Extending Apex, Flow and Agentforce - Java](https://github.com/heroku-examples/heroku-integration-pattern-org-action-java) | This sample demonstrates importing a Heroku application into an org to enable Apex, Flow, and Agentforce to call out to Heroku. For Apex, both synchronous and asynchronous invocation are demonstrated, along with securely elevating Salesforce permissions for processing that requires additional object or field access. |
| [Scaling Batch Jobs with Heroku - Java](https://github.com/heroku-examples/heroku-integration-pattern-org-job-java) | This sample seamlessly delegates the processing of large amounts of data with significant compute requirements to Heroku Worker processes. It also demonstrates the use of the Unit of Work aspect of the SDK (JavaScript only for the pilot) for easier utilization of the Salesforce Composite APIs. |
| [Using Eventing to drive Automation and Communication](https://github.com/heroku-examples/heroku-integration-pattern-eventing-java) | This sample extends the batch job sample by adding the ability to use eventing to start the work and notify users once it completes using Custom Notifications. These notifications are sent to the user's desktop or mobile device running Salesforce Mobile. Flow is used in this sample to demonstrate how processing can be handed off to low-code tools such as Flow. |
