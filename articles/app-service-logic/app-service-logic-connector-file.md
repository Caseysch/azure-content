<properties
	pageTitle="Using the File connector in Logic Apps | Microsoft Azure App Service"
	description="How to create and configure the File Connector or API app and use it in a logic app in Azure App Service"
	authors="rajeshramabathiran"
	manager="dwrede"
	editor=""
	services="app-service\logic"
	documentationCenter=""/>

<tags
	ms.service="app-service-logic"
	ms.workload="integration"
	ms.tgt_pltfrm="na"
	ms.devlang="na"
	ms.topic="article"
	ms.date="11/11/2015"
	ms.author="rajram"/>

# Get started with the File Connector and add it to your Logic App
Connect to a file system to upload, download, and more to your files on a host machine. Logic apps can trigger based on a variety of data sources and offer connectors to get and process data. You can add the File connector to your business workflow and process data as part of this workflow within a Logic App. 

The File Connector uses the Hybrid Connection Manager for hybrid connectivity to the host file system.

## Creating a File connector for your Logic App ##
To use the File connector, you need to first create an instance of the File connector API app. This can be done as follows:

1.	Open the Azure Marketplace using the + NEW option on the left side of the Azure Portal.
2.	Browse to “Marketplace > API Apps” and search for “File Connector”.
3.	Configure the File connector as follows:  
![][1]

	- **Name** - give a name for your File Connector
	- **Package Settings**
		- **Root Folder** - Specify the root folder path on your host machine. Eg. D:\FileConnectorTest
		- **Service Bus Connection String** - Provide a Service Bus Connection String. Make sure that the service bus namespace is of type Standard and NOT Basic to allow for use of Service Bus Relays.  Service Bus Relay is used to connect to the Hybrid Connection Manager.
	- **App Service plan** - select or create a App Service plan
	- **Pricing tier** - choose a pricing tier for the connector
	- **Resource group** - select or create a resource group where the connector should reside
	- **Subscription** - choose a subscription you want this connector to be created in
	- **Location** - choose the geographic location where you would like the connector to be deployed

4. Click on Create. A new File Connector will be created

## Configure Hybrid Connection Manager ##
Once the API App instance is created, browse to its dashboard.  This can be done by clicking on Browse > API Apps > select your File Connector API App.  From here the Hybrid Connection Manager needs to be configured.
For more information on configuring and trouble shooting the Hybrid Connection Manager see [Using the Hybrid Connection Manager].

## Using the File Connector in your Logic App ##
Once your API app is created, you can now use the File connector as an action for your Logic App. To do this, you need to:

1.	Create a new Logic App and choose the same resource group which has the File connector. Follow instructions to [Create a new Logic App].

2.	Open “Triggers and Actions” within the created Logic App to open the Logic Apps Designer and configure your flow.

3.	The File connector would appear in the “API Apps in this resource group” section in the gallery on the right hand side.

4.	You can drop the File Connector API app into the editor by clicking on the “File Connector”. File Connector exposes one trigger and 4 Actions:  
![][5]

6.	Each one of these exposes certain properties. The image below lists the properties for the trigger and Get File Action:  
![][6]

7. Once these are configured, the Trigger and Action can be used in your flow. Similarly, other actions can be configured as well.

> [AZURE.NOTE] The file trigger will delete the file after it is successfully read from the folder.

## File Connector REST APIs ##
To use the connector outside of a logic App, the REST APIs exposed by the connector can be leveraged. You can view this API Definitions using Browse->Api App->File Connector. Now click on the API Definition lens under the Summary Section to view all the APIs exposed by this connector:  
![][7]

Details of the APIs can be found at [File Connector API Definition].

## Do more with your Connector
Now that the connector is created, you can add it to a business workflow using a Logic App. See [What are Logic Apps?](app-service-logic-what-are-logic-apps.md).

>[AZURE.NOTE] If you want to get started with Azure Logic Apps before signing up for an Azure account, go to [Try Logic App](https://tryappservice.azure.com/?appservice=logic), where you can immediately create a short-lived starter logic app in App Service. No credit cards required; no commitments.

View the Swagger REST API reference at [Connectors and API Apps Reference](http://go.microsoft.com/fwlink/p/?LinkId=529766).

You can also review performance statistics and control security to the connector. See [Manage and Monitor your built-in API Apps and Connectors](app-service-logic-monitor-your-connectors.md).

<!-- Image reference -->
[1]: ./media/app-service-logic-connector-file/img1.PNG
[5]: ./media/app-service-logic-connector-file/img5.PNG
[6]: ./media/app-service-logic-connector-file/img6.PNG
[7]: ./media/app-service-logic-connector-file/img7.PNG

<!-- Links -->
[Create a new Logic App]: app-service-logic-create-a-logic-app.md
[File Connector API Definition]: https://msdn.microsoft.com/library/dn936296.aspx
[Using the Hybrid Connection Manager]: app-service-logic-hybrid-connection-manager.md
