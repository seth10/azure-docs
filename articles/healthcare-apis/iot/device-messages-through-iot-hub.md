---
title: Receive device messages through Azure IoT Hub - Azure Health Data Services
description: Learn how to deploy Azure IoT Hub with message routing to send device messages to the MedTech service. The tutorial uses an Azure Resource Manager template and Visual Studio Code with the Azure IoT Hub extension.
services: healthcare-apis
author: msjasteppe
ms.service: healthcare-apis
ms.subservice: iomt
ms.topic: tutorial
ms.date: 05/03/2023
ms.custom: devx-track-arm-template
ms.author: jasteppe
---

# Tutorial: Receive device messages through Azure IoT Hub

> [!NOTE] 
> [Fast Healthcare Interoperability Resources (FHIR&#174;)](https://www.hl7.org/fhir/) is an open healthcare specification.

For enhanced workflows and ease of use, you can use the MedTech service to receive messages from devices you create and manage through an IoT hub in [Azure IoT Hub](../../iot-hub/iot-concepts-and-iot-hub.md). This tutorial uses an Azure Resource Manager template (ARM template) and a **Deploy to Azure** button to deploy a MedTech service. The template deploys an IoT hub to create and manage devices, and then routes the device messages to an event hub for the MedTech service to read and process.

:::image type="content" source="media\device-messages-through-iot-hub\data-flow-diagram.png" border="false" alt-text="Diagram of the IoT device message flow through an IoT hub and event hub, and then into the MedTech service." lightbox="media\device-messages-through-iot-hub\data-flow-diagram.png":::

> [!TIP]
> To learn how the MedTech service transforms and persists device data into the FHIR service as FHIR Observations, see [Overview of the MedTech service device data processing stages](overview-of-device-data-processing-stages.md).

In this tutorial, you learn how to:

> [!div class="checklist"]
> * Open an ARM template in the Azure portal.
> * Configure the template for your deployment.
> * Create a device.
> * Send a test message.
> * Review metrics for the test message.

> [!TIP]
> To learn about ARM templates, see [What are ARM templates?](./../../azure-resource-manager/templates/overview.md)

## Prerequisites

To begin your deployment and complete the tutorial, you must have the following prerequisites:

- An active Azure subscription account. If you don't have an Azure subscription, see the [subscription decision guide](/azure/cloud-adoption-framework/decision-guides/subscriptions/).

- **Owner** or **Contributor and User Access Administrator** role assignments in the Azure subscription. For more information, see [What is Azure role-based access control (Azure RBAC)?](../../role-based-access-control/overview.md)

- The Microsoft.HealthcareApis, Microsoft.EventHub, and Microsoft.Devices resource providers registered with your Azure subscription. To learn more, see [Azure resource providers and types](../../azure-resource-manager/management/resource-providers-and-types.md).

- [Visual Studio Code](https://code.visualstudio.com/Download) installed locally.

- [Azure IoT Tools](https://marketplace.visualstudio.com/items?itemName=vsciot-vscode.azure-iot-toolkit) installed in Visual Studio Code. Azure IoT Tools is a collection of extensions that makes it easy to connect to IoT hubs, create devices, and send messages. In this tutorial, you use the Azure IoT Hub extension in Visual Studio Code to connect to your deployed IoT hub, create a device, and send a test message from the device to your IoT hub.

When you have these prerequisites, you're ready to configure the ARM template by using the **Deploy to Azure** button.

## Review the ARM template - Optional

The ARM template used to deploy the resources in this tutorial is available at [Azure Quickstart Templates](/samples/azure/azure-quickstart-templates/iotconnectors-with-iothub/) by using the *azuredeploy.json* file on [GitHub](https://github.com/azure/azure-quickstart-templates/tree/master/quickstarts/microsoft.healthcareapis/workspaces/iotconnectors-with-iothub).

## Use the Deploy to Azure button

To begin deployment in the Azure portal, select the **Deploy to Azure** button:

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FAzure%2Fazure-quickstart-templates%2Fmaster%2Fquickstarts%2Fmicrosoft.healthcareapis%2Fworkspaces%2Fiotconnectors-with-iothub%2Fazuredeploy.json)

## Configure the deployment

1. In the Azure portal, on the **Basics** tab of the Azure Quickstart Template, select or enter the following information for your deployment:

   - **Subscription**: The Azure subscription to use for the deployment.

   - **Resource group**: An existing resource group, or you can create a new resource group.

   - **Region**: The Azure region of the resource group that's used for the deployment. **Region** autofills by using the resource group region.

   - **Basename**: A value that's appended to the name of the Azure resources and services that are deployed. The examples in this tutorial use the basename *azuredocsdemo*. You can choose your own basename value.

   - **Location**: A supported Azure region for Azure Health Data Services (the value can be the same as or different from the region your resource group is in). For a list of Azure regions where Health Data Services is available, see [Products available by regions](https://azure.microsoft.com/explore/global-infrastructure/products-by-region/?products=health-data-services).

   - **Fhir Contributor Principle Id** (optional): An Azure Active Directory (Azure AD) user object ID to provide read/write permissions in the FHIR service.

     You can use this account to give access to the FHIR service to view the device messages that are generated in this tutorial. We recommend that you use your own Azure AD user object ID, so you can access the messages in the FHIR service. If you choose not to use the **Fhir Contributor Principle Id** option, clear the text box.

     To learn how to get an Azure AD user object ID, see [Find the user object ID](/partner-center/find-ids-and-domain-names#find-the-user-object-id). The user object ID that's used in this tutorial is only an example. If you use this option, use your own user object ID or the object ID of another person who you want to be able to access the FHIR service.

   - **Device mapping**: Leave the default values for this tutorial.

   - **Destination mapping**: Leave the default values for this tutorial.

   :::image type="content" source="media\device-messages-through-iot-hub\deploy-template-options.png" alt-text="Screenshot that shows deployment options for the MedTech service for Health Data Services in the Azure portal." lightbox="media\device-messages-through-iot-hub\deploy-template-options.png":::

2. To validate your configuration, select **Review + create**.

   :::image type="content" source="media\device-messages-through-iot-hub\review-and-create-button.png" alt-text="Screenshot that shows the Review + create button selected in the Azure portal.":::

3. In **Review + create**, check the template validation status. If validation is successful, the template displays **Validation Passed**. If validation fails, fix the detail that's indicated in the error message, and then select **Review + create** again.

   :::image type="content" source="media\device-messages-through-iot-hub\validation-complete.png" alt-text="Screenshot that shows the Review + create pane displaying the Validation Passed message.":::

4. After a successful validation, to begin the deployment, select **Create**.

   :::image type="content" source="media\device-messages-through-iot-hub\create-button.png" alt-text="Screenshot that shows the highlighted Create button.":::

5. In a few minutes, the Azure portal displays the message that your deployment is completed.

   :::image type="content" source="media\device-messages-through-iot-hub\deployment-complete-banner.png" alt-text="Screenshot that shows a green checkmark and the message Your deployment is complete.":::

   > [!IMPORTANT]
   > If you're going to allow access from multiple services to the event hub, it's required that each service has its own event hub consumer group.
   >
   > Consumer groups enable multiple consuming applications to have a separate view of the event stream, and to read the stream independently at their own pace and with their own offsets. For more information, see [Consumer groups](../../event-hubs/event-hubs-features.md#consumer-groups).
   >
   > Examples:
   >
   > * Two MedTech services accessing the same event hub.
   >
   > * A MedTech service and a storage writer application accessing the same event hub.

## Review deployed resources and access permissions

When deployment is completed, the following resources and access roles are created in the template deployment:

* Event Hubs namespace and event hub. In this deployment, the event hub is named *devicedata*.

  * Event hub consumer group. In this deployment, the consumer group is named *$Default*.

  * **Azure Event Hubs Data Sender** role. In this deployment, the sender role is named *devicedatasender* and can be used to provide access to the event hub using a shared access signature (SAS). To learn more about authorizing access using a SAS, see [Authorizing access to Event Hubs resources using Shared Access Signatures](../../event-hubs/authorize-access-shared-access-signature.md). The **Azure Event Hubs Data Sender** role isn't used in this tutorial.

* IoT hub with [message routing](../../iot-hub/iot-hub-devguide-messages-d2c.md) configured to send device messages to the event hub.

* [User-assigned managed identity](../../active-directory/managed-identities-azure-resources/overview.md), which provides send access from the IoT hub to the event hub. The managed identity has the **Azure Event Hubs Data Sender** role in the [Access control section (IAM)](../../role-based-access-control/overview.md) of the event hub.

* Health Data Services workspace.

* Health Data Services FHIR service.

* Health Data Services MedTech service with the [system-assigned managed identity](../../active-directory/managed-identities-azure-resources/overview.md) enabled and granted the following access roles:

  * For the event hub, the **Azure Event Hubs Data Receiver** access role is assigned in the [Access control section (IAM)](../../role-based-access-control/overview.md) of the event hub.

  * For the FHIR service, the **FHIR Data Writer** access role is assigned in the [Access control section (IAM)](../../role-based-access-control/overview.md) of the FHIR service.

* Conforming and valid MedTech service [device](overview-of-device-mapping.md) and [FHIR destination mappings](overview-of-fhir-destination-mapping.md). **Resolution type** is set to **Create**.

> [!IMPORTANT]
> In this tutorial, the ARM template configures the MedTech service to operate in **Create** mode. A Patient resource and a Device resource are created for each device that sends data to your FHIR service.
>
> To learn about the MedTech service resolution types **Create** and **Lookup**, see [Configure the Destination tab](deploy-manual-portal.md#configure-the-destination-tab).

## Create a device and send a test message

With your resources successfully deployed, you next connect to your IoT hub, create a device, and send a test message to the IoT hub. After you complete these steps, your MedTech service can:

* Read the IoT hub-routed test message from the event hub.
* Transform the test message into five FHIR Observations.
* Persist the FHIR Observations into your FHIR service.

You complete the steps by using Visual Studio Code with the Azure IoT Hub extension:

1. Open Visual Studio Code with Azure IoT Tools installed.

2. In Explorer, in **Azure IoT Hub**, select **…** and choose **Select IoT Hub**.

   :::image type="content" source="media\device-messages-through-iot-hub\select-iot-hub.png" alt-text="Screenshot of Visual Studio Code with the Azure IoT Hub extension with the deployed IoT hub selected." lightbox="media\device-messages-through-iot-hub\select-iot-hub.png":::

3. Select the Azure subscription where your IoT hub was provisioned.

4. Select your IoT hub. The name of your IoT hub is the *basename* you provided when you provisioned the resources prefixed with **ih-**. An example hub name is *ih-azuredocsdemo*.

5. In Explorer, in **Azure IoT Hub**, select **…** and choose **Create Device**. An example device name is *iot-001*.

   :::image type="content" source="media\device-messages-through-iot-hub\create-device.png" alt-text="Screenshot that shows Visual Studio Code with the Azure IoT Hub extension with Create device selected." lightbox="media\device-messages-through-iot-hub\create-device.png":::

6. To send a test message from the device to your IoT hub, right-click the device and select **Send D2C Message to IoT Hub**.

   > [!NOTE]
   > In this device-to-cloud (D2C) example, *cloud* is the IoT hub in the Azure IoT Hub that receives the device message. Azure IoT Hub supports two-way communications. To set up a cloud-to-device (C2D) scenario, select **Send C2D Message to Device Cloud**.

   :::image type="content" source="media\device-messages-through-iot-hub\select-device-to-cloud-message.png" alt-text="Screenshot that shows Visual Studio Code with the Azure IoT Hub extension and the Send D2C Message to IoT Hub option selected." lightbox="media\device-messages-through-iot-hub\select-device-to-cloud-message.png":::

7. In **Send D2C Messages**, select or enter the following values:

   * **Device(s) to send messages from**: The name of the device you created.

   * **Message(s) per device**: **1**.

   * **Interval between two messages**: **1 second(s)**.

   * **Message**: **Plain Text**.

   * **Edit**: Clear any existing text, and then paste the following JSON.

     > [!TIP]
     > You can use the **Copy** option in in the right corner of the below test message, and then paste it within the **Edit** option.

     ```json
     {
       "HeartRate": 78,
       "RespiratoryRate": 12,
       "HeartRateVariability": 30,
       "BodyTemperature": 98.6,
       "BloodPressure": {
         "Systolic": 120,
         "Diastolic": 80
       }
     }
     ```

8. To begin the process of sending a test message to your IoT hub, select **Send**.

   :::image type="content" source="media\device-messages-through-iot-hub\select-device-to-cloud-message-options.png" alt-text="Screenshot that shows Visual Studio Code with the Azure IoT Hub extension with the device message options selected." lightbox="media\device-messages-through-iot-hub\select-device-to-cloud-message-options.png":::

   After you select **Send**, it might take up to five minutes for the FHIR resources to be available in the FHIR service.

   > [!IMPORTANT]
   > To avoid device spoofing in device-to-cloud (D2C) messages, Azure IoT Hub enriches all device messages with additional properties before sending it to the MedTech service device event hub. For example: **Properties**: `iothub-creation-time-utc` and **SystemProperties**: `iothub-connection-device-id`. For more information, see [Anti-spoofing properties](../../iot-hub/iot-hub-devguide-messages-construct.md#anti-spoofing-properties) and [How to use IotJsonPathContent mappings](how-to-use-iot-jsonpath-content-mappings.md). 
   >
   > You do not want to send this example device message to your IoT hub as the enrichments will be duplicated by the IoT hub and cause an error with your MedTech service. This is only an example of how your device messages are enriched by the IoT hub. 
   >
   > Example:
   >
   > :::image type="content" source="media\device-messages-through-iot-hub\iot-hub-enriched-device-message.png" alt-text="Screenshot of an Azure IoT Hub enriched device message." lightbox="media\device-messages-through-iot-hub\iot-hub-enriched-device-message.png":::
   >
   > `patientIdExpression` is only required for MedTech services in the **Create** mode, however, if **Lookup** is being used, a Device resource with a matching Device Identifier must exist in the FHIR service. This example assumes your MedTech service is in a **Create** mode. The **Resolution type** for this tutorial set to **Create**. For more information on the **Destination properties**: **Create** and **Lookup**, see [Configure the Destination tab](deploy-manual-portal.md#configure-the-destination-tab). 

## Review metrics from the test message

Now that you have successfully sent a test message to your IoT hub, review your MedTech service metrics. You review metrics to verify that your MedTech service received, grouped, transformed, and persisted the test message to your FHIR service. To learn more, see [How to display the MedTech service monitoring tab metrics](how-to-use-monitoring-tab.md).

For your MedTech service metrics, you can see that your MedTech service completed the following steps for the test message:

* **Number of Incoming Messages**: Received the incoming test message from the device message event hub.
* **Number of Normalized Messages**: Created five normalized messages.
* **Number of Measurements**: Created five measurements.
* **Number of FHIR resources**: Created five FHIR resources that are persisted in your FHIR service.

:::image type="content" source="media\device-messages-through-iot-hub\metrics-tile-one.png" alt-text="Screenshot that shows a MedTech service metrics tile and test data metrics." lightbox="media\device-messages-through-iot-hub\metrics-tile-one.png":::

:::image type="content" source="media\device-messages-through-iot-hub\metrics-tile-two.png" alt-text="Screenshot that shows a second MedTech service metrics tile and test data metrics." lightbox="media\device-messages-through-iot-hub\metrics-tile-two.png":::

## View test data in the FHIR service

If you provided your own Azure AD user object ID as the optional value for **Fhir Contributor Principal ID** in the deployment template, you can query FHIR resources in your FHIR service.

To learn how to get an Azure AD access token and view FHIR resources in your FHIR service, see [Access by using Postman](../fhir/use-postman.md).

## Next steps

In this tutorial, you deployed an ARM template in the Azure portal, connected to your IoT hub, created a device, sent a test message, and reviewed your MedTech service metrics.

To learn about other methods for deploying the MedTech service, see

> [!div class="nextstepaction"] 
> [Choose a deployment method for the MedTech service](deploy-choose-method.md)

FHIR&#174; is a registered trademark of Health Level Seven International, registered in the U.S. Trademark Office and is used with their permission.
