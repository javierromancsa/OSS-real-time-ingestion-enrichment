# Azure Data Explorer
A fast and highly scalable data exploration service for log and telemetry data. A fully managed data analytics service for near real-time analysis on large volumes(TBs) of data streaming from applications, websites, IoT devices, and more. You can quickly identify patterns, anomalies, and trends in your data. Explore new questions and get answers in minutes. Run as many queries as you need, thanks to the optimized cost structure.

## What makes Azure Data Explorer unique?
- Scales quickly to terabytes of data, in minutes, allowing rapid iterations of data exploration to discover relevant insights.
- Offers an innovative query language, optimized for high-performance data analytics.
- Supports analysis of high volumes of heterogeneous data (structured and unstructured).
- Provides the ability to build and deploy exactly what you need by combining with other services to supply an encompassing, powerful, and interactive data analytics solution.

## Deploying Azure Explorer and Creating DB with target tables
```
az group create --name <resource-group-name> --location <resource-group-location> #use this command when you need to create a new resource group for your deployment
az group deployment create --resource-group <my-resource-group> --template-uri https://raw.githubusercontent.com/Azure/azure-quickstart-templates/master/101-kusto-vnet/azuredeploy.json --parameters-uri https://raw.githubusercontent.com//azuredeploy.parameters.json
```
