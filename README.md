# Azure-ServiceBus-DR

Azure CLI sample showing the process of leveraging Azure Service Bus Geo-disaster recovery

The guide script can be found here [service-bus-provisioning.sh](provisioning/service-bus-provisioning.sh)

This guid show the following:

1. Creating 2 new Service Bus namespaces in 2 different regions
2. Establish geo-pairing between the primary and secondary namespaces
3. Create various entities in the primary namespace (which will be replicated to the secondary namespace)
4. Fail over to secondary namespace
5. Create a new geo paired namespace (as the pairing to the old primary no long exists)

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