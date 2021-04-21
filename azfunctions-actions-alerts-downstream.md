# Azure Functions and KEDA

## Pre-requesitics:

- Use this Java Azure Function walkthough to install the require dependencies in your VM and validate you can deploy to Azure a simple Java function.
    - https://docs.microsoft.com/en-us/azure/azure-functions/create-first-function-cli-java?tabs=bash%2Cazure-cli%2Cbrowser
    - If you get this error: ' Failed to start Worker Channel. Process fileName: %JAVA_HOME%/bin/java System.Diagnostics.Process: No such file or directory. Failed to start language worker process for runtime: (null).
        - sudo vi /usr/lib/azure-functions-core-tools-3/workers/java/worker.config.json and edit the JAVA_HOME path.
- Install .NET Core SDK version 3+ for buidling the kafka extentions
    -  sudo apt-get install dotnet-sdk-3.1
- Git clone this repo
## Using the repo to deploy functions

