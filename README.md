# Azure-ServiceBus-DR

Azure CLI sample showing the process of leveraging Azure Service Bus Geo-disaster recovery

The guide script can be found here [service-bus-provisioning.sh](provisioning/service-bus-provisioning.sh)

This guid show the following:

1. Creating 2 new Service Bus namespaces in 2 different regions
2. Establish geo-pairing between the primary and secondary namespaces
3. Create various entities in the primary namespace (which will be replicated to the secondary namespace)
4. Fail over to secondary namespace
5. Create a new geo paired namespace (as the pairing to the old primary no long exists)

## Script

```bash

# Docs: https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-geo-dr
# Azure CLI: https://docs.microsoft.com/en-us/cli/azure/servicebus/namespace?view=azure-cli-latest
# Code based provisioning (.NET): https://github.com/Azure/azure-service-bus/tree/master/samples/DotNet/Microsoft.ServiceBus.Messaging/GeoDR

### Variables
PREFIX=sb-dev
LOCATION=westeurope
LOCATION_SECONDARY=northeurope
RG=$PREFIX-sb-rg
SB_NAMESPACE=$PREFIX-$RANDOM-ns
SB_NAMESPACE_SECONDARY=$PREFIX-$RANDOM-ns
SB_ALIAS=$PREFIX-sb-alias
SB_TOPIC=master-topic

### Sign In to Azure CLI
az login

az account set --subscription Azure_subscription_name

# Just to make sure setting performed correctly
az account show

### Create a resource groups
az group create --name $RG --location $LOCATION

# Create a Service Bus messaging namespaces with a unique name
az servicebus namespace create \
   --resource-group $RG \
   --name $SB_NAMESPACE \
   --location $LOCATION \
   --sku Premium

az servicebus namespace create \
   --resource-group $RG \
   --name $SB_NAMESPACE_SECONDARY \
   --location $LOCATION_SECONDARY \
   --sku Premium

# Create Alias aned establish pairing between primary and secondary namespaces
# Note if the partner namespace is in different resource group you need the full resource id not only name
az servicebus georecovery-alias set \
    --resource-group $RG \
    --namespace-name $SB_NAMESPACE \
    --alias $SB_ALIAS \
    --partner-namespace $SB_NAMESPACE_SECONDARY

# Show the geo recovery alias 
# Notice the role value between Primary and Secondary
# Also wait for provisioningState to be Succeeded (it takes less than a min)
az servicebus georecovery-alias show  \
    --resource-group $RG \
    --namespace-name $SB_NAMESPACE \
    --alias $SB_ALIAS

az servicebus georecovery-alias show  \
    --resource-group $RG \
    --namespace-name $SB_NAMESPACE_SECONDARY \
    --alias $SB_ALIAS

### Creating service bus entities to test the pairing

# Create a Service Bus Topic on primary namespace
az servicebus topic create --resource-group $RG \
   --namespace-name $SB_NAMESPACE \
   --name $SB_TOPIC

# Create subscription 1 to the topic
az servicebus topic subscription create \
    --resource-group $RG \
    --namespace-name $SB_NAMESPACE \
    --topic-name $SB_TOPIC \
    --name S1

# Create filter 1 - use custom properties
az servicebus topic subscription rule create \
    --resource-group $RG \
    --namespace-name $SB_NAMESPACE \
    --topic-name $SB_TOPIC \
    --subscription-name S1 \
    --name MyFilter \
    --filter-sql-expression "StoreId IN ('Store1','Store2','Store3')"

# Create filter 2 - use custom properties
az servicebus topic subscription rule create \
    --resource-group $RG \
    --namespace-name $SB_NAMESPACE \
    --topic-name $SB_TOPIC \
    --subscription-name S1 \
    --name MySecondFilter \
    --filter-sql-expression "StoreId = 'Store4'"

# Create subscription 2
az servicebus topic subscription create \
    --resource-group $RG \
    --namespace-name $SB_NAMESPACE \
    --topic-name $SB_TOPIC \
    --name S2

# Create filter 3 - use message header properties via IN list and 
# combine with custom properties.
az servicebus topic subscription rule create \
    --resource-group $RG \
    --namespace-name $SB_NAMESPACE \
    --topic-name $SB_TOPIC \
    --subscription-name S2 \
    --name MyFilter \
    --filter-sql-expression "sys.To IN ('Store5','Store6','Store7') OR StoreId = 'Store8'"

# Create subscription 3
az servicebus topic subscription create \
    --resource-group $RG \
    --namespace-name $SB_NAMESPACE \
    --topic-name $SB_TOPIC \
    --name S3

# Create filter 4 - Get everything except messages for subscription 1 and 2. 
# Also modify and add an action; in this case set the label to a specified value. 
# Assume those stores might not be part of your main store, so you only add 
# specific items to them. For that, you flag them specifically.
az servicebus topic subscription rule create \
    --resource-group $RG \
    --namespace-name $SB_NAMESPACE \
    --topic-name $SB_TOPIC \
    --subscription-name S3 \
    --name MyFilter \
    --filter-sql-expression "sys.To NOT IN ('Store1','Store2','Store3','Store4','Store5','Store6','Store7','Store8') OR StoreId NOT IN ('Store1','Store2','Store3','Store4','Store5','Store6','Store7','Store8')" \
    --action-sql-expression "SET sys.Label = 'SalesEvent'"

# Get the alias connection string
ALIAS_CONNECTION_STRING=$(az servicebus georecovery-alias authorization-rule keys list \
   --resource-group $RG \
   --alias $SB_ALIAS \
   --namespace-name  $SB_NAMESPACE \
   --name RootManageSharedAccessKey \
   --query aliasPrimaryConnectionString --output tsv)
echo $ALIAS_CONNECTION_STRING

### Failer over
# Note currently only forward semnatics failing are supported. 
# You can't pair the old primary again as it is not empty
az servicebus georecovery-alias fail-over \
    --resource-group $RG \
    --namespace-name $SB_NAMESPACE_SECONDARY \
    --alias $SB_ALIAS

# Check the status using the secondary namespace (notice role is PrimaryNotReplicating)
az servicebus georecovery-alias show  \
    --resource-group $RG \
    --namespace-name $SB_NAMESPACE_SECONDARY \
    --alias $SB_ALIAS

### What is next after fail over?

# Decommision the old primary namespace
# 1. Wait for the primary namespace to become avaialbe again, 
# then pull any messages needed and process them according to your 
# business logic (maybe push them to the new primary for processing)
# 2. Delete the old namespace (when no longer needed as you can't re-pair it)
az servicebus namespace delete \
    --resource-group $RG \
    --name $SB_NAMESPACE
    --no-wait
# --no-wait is used for the command to return without waiting for the deletion to complete

# Setup a new geo DR namespace
# 0. Update vairables :)
LOCATION=northeurope
LOCATION_SECONDARY=westeurope
SB_NAMESPACE=$SB_NAMESPACE_SECONDARY
SB_NAMESPACE_SECONDARY=$PREFIX-$RANDOM-ns

# 1. Create new secondary namespace
az servicebus namespace create \
   --resource-group $RG \
   --name $SB_NAMESPACE_SECONDARY \
   --location $LOCATION_SECONDARY \
   --sku Premium

# 2. Reset the pairing again with the new created namespace:
az servicebus georecovery-alias set \
    --resource-group $RG \
    --namespace-name $SB_NAMESPACE \
    --alias $SB_ALIAS \
    --partner-namespace $SB_NAMESPACE_SECONDARY

# Clean up
az group delete --name $RG

```

## About the project

I tried to make sure I cover all aspects and best practices while building this project, but all included architecture, code, documentation and any other artifact represent my personal opinion only. Think of it as a suggestion of how a one way things can work.

Keep in mind that this is a work-in-progress, I will continue to contribute to it when I can.

All constructive feedback is welcomed 🙏

## Support

You can always create issue, suggest an update through PR or direct message me on [Twitter](https://twitter.com/mohamedsaif101).

## Authors

|      ![Photo](res/mohamed-saif.jpg)            |
|:----------------------------------------------:|
|                 **Mohamed Saif**               |
|     [GitHub](https://github.com/mohamedsaif)   |
|  [Twitter](https://twitter.com/mohamedsaif101) |
|         [Blog](http://blog.mohamedsaif.com)    |