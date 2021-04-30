# Azure Functions and KEDA

## Pre-requesitics:

- Use this Java Azure Function walkthough to install the require dependencies in your VM and validate you can deploy to Azure a simple Java function.
    - https://docs.microsoft.com/en-us/azure/azure-functions/create-first-function-cli-java?tabs=bash%2Cazure-cli%2Cbrowser
    - If you get this error: ' Failed to start Worker Channel. Process fileName: %JAVA_HOME%/bin/java System.Diagnostics.Process: No such file or directory. Failed to start language worker process for runtime: (null).
        - sudo vi /usr/lib/azure-functions-core-tools-3/workers/java/worker.config.json and edit the JAVA_HOME path.
- Install .NET Core SDK version 3+ for buidling the kafka extentions
    -  sudo apt-get install dotnet-sdk-3.1
- Git clone this repo https://github.com/javierromancsa/azure-functions-kafka

### Using the repo to deploy the functions:

#### This repo have the necesary setting to build and deploy a java function for kafka Trigger and kafka output which are the to bindings available for kafka. For further details about the kafka extension use this link https://github.com/Azure/azure-functions-kafka-extension

#### The code is for a non-authtenticate kafka client and there are a couple setting that needs to be modify based on your azure environment. Fisrt lets look at the pom.xml
 - functionappname has to be unique name \<functionAppName>fabrikam-kafka-functions-20210421193823036\</functionAppName>
 - resourcegroup name \<resourceGroup>java-functions-group\</resourceGroup>
 - appserviceplanname name \<appServicePlanName>kafka-function-jrs01plan919362\</appServicePlanName>
 - region function app azure region to deploy \<region>eastus2\</region>
 - function pricingTier for kafka we need a minimun of EP1 \<pricingTier>EP1\</pricingTier> [detail](https://github.com/Azure/azure-functions-kafka-extension#linux-premium-plan-configuration) 
 - the enviroment variables or application settings for setting your brokerlist/bootstrapers and topic name:
 ```
<property>
    <name>BrokerList</name>
    <value>wn2-jrs02k.qz1stakpznlepcjn2bzi2treqb.cx.internal.cloudapp.net:9092,wn1-jrs02k.qz1stakpznlepcjn2bzi2treqb.cx.internal.cloudapp.net:9092,wn0-jrs02k.qz1stakpznlepcjn2bzi2treqb.cx.internal.cloudapp.net:9092</value>
</property>
<property>
    <name>TopicName</name>
    <value>message</value>
</property>
<property>
    <name>GroupName</name>
    <value>myfunc01</value>
</property>
```
#### build and deploy to azure
- build with maven
```mvn clean package```
- login to azure
```az login```
- deploy to azure
```mvn azure-functions:deploy -e```

#### Add the Vnet integration feature to your Azure Function, so the function can use a private IP from the Vnet running AKS and HDInsight kafka.( if you're using confluent cloud this part is not need it)
- Go to your new azure function and click on the "Networking" blade. Then select the Vnet Integration configuration link. See photo below
![pic](https://github.com/javierromancsa/images/blob/main/vnet-integration.PNG)
- Click on "Add Vnet" and fill all the Vnet and subnet information. See photo below 
![pic](https://github.com/javierromancsa/images/blob/main/vnet-details.PNG)
