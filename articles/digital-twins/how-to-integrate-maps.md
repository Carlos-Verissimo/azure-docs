---
# Mandatory fields.
title: Integrate with Azure Maps
titleSuffix: Azure Digital Twins
description: Learn how to use Azure Functions to create a function that can use the twin graph and Azure Digital Twins notifications to update an Azure Maps indoor map.
author: baanders
ms.author: baanders # Microsoft employees only
ms.date: 02/22/2022
ms.topic: how-to
ms.service: digital-twins

# Optional fields. Don't forget to remove # if you need a field.
# ms.custom: can-be-multiple-comma-separated
ms.reviewer: baanders
# manager: MSFT-alias-of-manager-or-PM-counterpart
---

# Use Azure Digital Twins to update an Azure Maps indoor map

This article walks through the steps required to use Azure Digital Twins data to update information displayed on an *indoor map* using [Azure Maps](../azure-maps/about-azure-maps.md). Azure Digital Twins stores a graph of your IoT device relationships and routes telemetry to different endpoints, making it the perfect service for updating informational overlays on maps.

This guide will cover:

1. Configuring your Azure Digital Twins instance to send twin update events to a function in [Azure Functions](../azure-functions/functions-overview.md).
2. Creating a function to update an Azure Maps indoor maps feature stateset.
3. How to store your maps ID and feature stateset ID in the Azure Digital Twins graph.

### Prerequisites

* Follow the Azure Digital Twins in [Connect an end-to-end solution](./tutorial-end-to-end.md).
    * You'll be extending this twin with another endpoint and route. You'll also be adding another function to your function app from that tutorial. 
* Follow the Azure Maps in [Use Azure Maps Creator to create indoor maps](../azure-maps/tutorial-creator-indoor-maps.md) to create an Azure Maps indoor map with a *feature stateset*.
    * [Feature statesets](../azure-maps/creator-indoor-maps.md#feature-statesets) are collections of dynamic properties (states) assigned to dataset features such as rooms or equipment. In the Azure Maps tutorial above, the feature stateset stores room status that you'll be displaying on a map.
    * You'll need your feature **stateset ID** and Azure Maps **subscription key**.

### Topology

The image below illustrates where the indoor maps integration elements in this tutorial fit into a larger, end-to-end Azure Digital Twins scenario.

:::image type="content" source="media/how-to-integrate-maps/maps-integration-topology.png" alt-text="Diagram of Azure services in an end-to-end scenario, highlighting the Indoor Maps Integration piece." lightbox="media/how-to-integrate-maps/maps-integration-topology.png":::

## Create a function to update a map when twins update

First, you'll create a route in Azure Digital Twins to forward all twin update events to an Event Grid topic. Then, you'll use a function to read those update messages and update a feature stateset in Azure Maps. 

## Create a route and filter to twin update notifications

Azure Digital Twins instances can emit twin update events whenever a twin's state is updated. The Azure Digital Twins [Connect an end-to-end solution](./tutorial-end-to-end.md) linked above walks through a scenario where a thermometer is used to update a temperature attribute attached to a room's twin. You'll be extending that solution by subscribing to update notifications for twins, and using that information to update your maps.

This pattern reads from the room twin directly, rather than the IoT device, which gives you the flexibility to change the underlying data source for temperature without needing to update your mapping logic. For example, you can add multiple thermometers or set this room to share a thermometer with another room, all without needing to update your map logic.

1. Create an Event Grid topic, which will receive events from your Azure Digital Twins instance.
    ```azurecli-interactive
    az eventgrid topic create --resource-group <your-resource-group-name> --name <your-topic-name> --location <region>
    ```

2. Create an endpoint to link your Event Grid topic to Azure Digital Twins.
    ```azurecli-interactive
    az dt endpoint create eventgrid --endpoint-name <Event-Grid-endpoint-name> --eventgrid-resource-group <Event-Grid-resource-group-name> --eventgrid-topic <your-Event-Grid-topic-name> --dt-name <your-Azure-Digital-Twins-instance-name>
    ```

3. Create a route in Azure Digital Twins to send twin update events to your endpoint.

    >[!NOTE]
    >There is currently a known issue in Cloud Shell affecting these command groups: `az dt route`, `az dt model`, `az dt twin`.
    >
    >To resolve, either run `az login` in Cloud Shell prior to running the command, or use the [local CLI](/cli/azure/install-azure-cli) instead of Cloud Shell. For more detail on this, see [Troubleshoot known issues](troubleshoot-known-issues.md#400-client-error-bad-request-in-cloud-shell).

    ```azurecli-interactive
    az dt route create --dt-name <your-Azure-Digital-Twins-instance-name> --endpoint-name <Event-Grid-endpoint-name> --route-name <my-route> --filter "type = 'Microsoft.DigitalTwins.Twin.Update'"
    ```

## Create a function to update maps

You're going to create an Event Grid-triggered function inside your function app from the end-to-end tutorial ([Connect an end-to-end solution](./tutorial-end-to-end.md)). This function will unpack those notifications and send updates to an Azure Maps feature stateset to update the temperature of one room.

See the following document for reference info: [Azure Event Grid trigger for Azure Functions](../azure-functions/functions-bindings-event-grid-trigger.md).

Replace the function code with the following code. It will filter out only updates to space twins, read the updated temperature, and send that information to Azure Maps.

:::code language="csharp" source="~/digital-twins-docs-samples/sdks/csharp/updateMaps.cs":::

You'll need to set two environment variables in your function app. One is your [Azure Maps primary subscription key](../azure-maps/quick-demo-map-app.md#get-the-primary-key-for-your-account), and one is your [Azure Maps stateset ID](../azure-maps/tutorial-creator-feature-stateset.md).

```azurecli-interactive
az functionapp config appsettings set --name <your-function-app-name> --resource-group <your-resource-group> --settings "subscription-key=<your-Azure-Maps-primary-subscription-key>"
az functionapp config appsettings set --name <your-function-app-name>  --resource-group <your-resource-group> --settings "statesetID=<your-Azure-Maps-stateset-ID>"
```

### View live updates on your map

To see live-updating temperature, follow the steps below:

1. Begin sending simulated IoT data by running the *DeviceSimulator* project from the Azure Digital Twins [Connect an end-to-end solution](tutorial-end-to-end.md). The instructions for this process are in the [Configure and run the simulation](././tutorial-end-to-end.md#configure-and-run-the-simulation) section.
2. Use [the Azure Maps Indoor module](../azure-maps/how-to-use-indoor-module.md) to render your indoor maps created in Azure Maps Creator.
    1. Copy the HTML from the [Example: Use the Indoor Maps Module](../azure-maps/how-to-use-indoor-module.md#example-use-the-indoor-maps-module) section of the indoor maps in [Use the Azure Maps Indoor Maps module](../azure-maps/how-to-use-indoor-module.md) to a local file.
    1. Replace the **subscription key**, **tilesetId**, and **statesetID**  in the local HTML file with your values.
    1. Open that file in your browser.

Both samples send temperature in a compatible range, so you should see the color of room 121 update on the map about every 30 seconds.

:::image type="content" source="media/how-to-integrate-maps/maps-temperature-update.png" alt-text="Screenshot of an office map showing room 121 colored orange.":::

## Store your maps information in Azure Digital Twins

Now that you have a hardcoded solution to updating your maps information, you can use the Azure Digital Twins graph to store all of the information necessary for updating your indoor map. This information would include the stateset ID, maps subscription ID, and feature ID of each map and location respectively. 

A solution for this specific example would involve updating each top-level space to have a stateset ID and maps subscription ID attribute, and updating each room to have a feature ID. You would need to set these values once when initializing the twin graph, then query those values for each twin update event.

Depending on the configuration of your topology, storing these three attributes at different levels correlating to the granularity of your map will be possible.

## Next steps

To read more about managing, upgrading, and retrieving information from the twins graph, see the following references:

* [Manage digital twins](./how-to-manage-twin.md)
* [Query the twin graph](./how-to-query-graph.md)
