---
title: Get started with the Azure SDK for Java
description: Get started with basic use of the Azure SDK for Java with your own Azure subscription.
keywords: Azure, Java, SDK, API ,authenticate, get-started
author: rloutlaw
ms.author: routlaw
manager: douge
ms.date: 04/16/2017
ms.topic: get-started-article
ms.prod: azure
ms.technology: azure
ms.devlang: java
ms.service: multiple
ms.assetid: b1e10b79-f75e-4605-aecd-eed64873e2d3
---

# Get started with the Azure SDK for Java

## Prerequisites

- An Azure account. If you don't have one , [get a free trial](https://azure.microsoft.com/free/)
- [Java 8](http://www.oracle.com/technetwork/java/javase/downloads/index.html)
- [Maven 3](http://maven.apache.org/download.cgi)
- [Azure CLI 2.0](https://docs.microsoft.com/en-us/cli/azure/install-az-cli2)

This get started guide uses the Maven build tool to build and run Java source code, but other build tools such as Gradle or SBT work fine with the Java SDK for Azure. 

## Set up authentication

Grant your application read and create permissions to your Azure subscription using a service principal. Service principals provide scoped access to resources in your account, and the Java SDK for Azure is designed around logging in with a service principal instead of a username and password.

[Create a service principal using the Azure CLI 2.0](/cli/azure/create-an-azure-service-principal-azure-cli), and make sure to capture the output:

```json
{
  "appId": "a487e0c1-82af-47d9-9a0b-af184eb87646d",
  "displayName": "JavaSDKTest",
  "name": "http://JavaSDKTest",
  "password": password,
  "tenant": "XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX"
}
```

Next, copy the following into a text file on your system:

```text
# sample management library properties file
subscription=########-####-####-####-############
client=########-####-####-####-############
key=XXXXXXXXXXXXXXXX
tenant=########-####-####-####-############
managementURI=https\://management.core.windows.net/
baseURL=https\://management.azure.com/
authURL=https\://login.windows.net/
graphURL=https\://graph.windows.net/
```

Replace the top four values with the following:

- subscription: use the *id* value from `az account show` in the Azure CLI 2.0.
- client: use the *appId* value from the output taken from a service principal output.
- key: use the *password* value from the service principal output .
- tenant: use the *tenant* value from the service principal output.

Save this file in a secure location on your system where your code can read it. Set an environment variable `AZURE_AUTH_LOCATION` with the full path to the authentication file in your shell.    

```bash
export AZURE_AUTH_LOCATION=/Users/raisa/azureauth.properties
```

## Create a Maven project and import the SDK dependency

Create a new Maven project from the command line in a new directory on your system:

```
mkdir java-sdk-test
cd java-sdk-test
mvn archetype:generate -DgroupId=com.fabrikam -DartifactId=testSDKApp -DarchetypeArtifactId=maven-archetype-quickstart -DinteractiveMode=false
```

This creates a basic Maven project under the `testSDKApp` folder. Add the the following entry into the project `pom.xml` to import the Java SDK for Azure.

```XML
<dependency>
    <groupId>com.microsoft.azure</groupId>
    <artifactId>azure</artifactId>
    <version>1.0.0-beta5</version>
</dependency>
```

Add a `build` entry under the top-level `project` element to use the [maven-exec-plugin](http://www.mojohaus.org/exec-maven-plugin/) to run the sample:

```XML
<build>
    <plugins>
        <plugin>
            <groupId>org.codehaus.mojo</groupId>
            <artifactId>exec-maven-plugin</artifactId>
            <configuration>
                <mainClass>com.fabrikam.testSDKApp.AzureSDKApp</mainClass>
            </configuration>
        </plugin>
    </plugins>
</build>
 ```


## Create a Linux virtual machine

Create a new file named AzureSDKApp.java in the project's `src/main/java` directory. Paste in the following code, then set real values for `userName` and `password`. 

This code performs the following steps:

1. Authenticates with Azure using the service principal information in `AZURE_AUTH_LOCATION`
2. Creates a new Ubuntu Linux VM in Azure with name `testLinuxVM` in a new Azure resource group `sampleResourceGroup` running in the US East region.

```java
package com.fabrikam.testSDKApp;

import com.microsoft.azure.management.Azure;
import com.microsoft.azure.management.compute.VirtualMachine;
import com.microsoft.azure.management.compute.KnownLinuxVirtualMachineImage;
import com.microsoft.azure.management.compute.VirtualMachineSizeTypes;
import com.microsoft.rest.LogLevel;
import com.microsoft.azure.management.resources.fluentcore.arm.Region;

import java.io.File;

public class AzureSDKApp {

    public static void main(String[] args) {

        final String userName = "YOUR_VM_USERNAME";
        final String password = "YOUR_VM_PASSWORD";

        try {

            // use the properties file with the service principal information to authenticate
            // change the name of the environment variable if you used a different name in the previous step
            final File credFile = new File(System.getenv("AZURE_AUTH_LOCATION"));    
            Azure azure = Azure.configure()
                    .withLogLevel(LogLevel.BASIC)
                    .authenticate(credFile)
                    .withDefaultSubscription();
           
            // create a Ubuntu virtual machine in a new resource group 
            VirtualMachine linuxVM = azure.virtualMachines().define("testLinuxVM")
                    .withRegion(Region.US_EAST)
                    .withNewResourceGroup("sampleResourceGroup")
                    .withNewPrimaryNetwork("10.0.0.0/24")
                    .withPrimaryPrivateIpAddressDynamic()
                    .withoutPrimaryPublicIpAddress()
                    .withPopularLinuxImage(KnownLinuxVirtualMachineImage.UBUNTU_SERVER_16_04_LTS)
                    .withRootUsername(userName)
                    .withRootPassword(password)
                    .withUnmanagedDisks()
                    .withSize(VirtualMachineSizeTypes.STANDARD_D3_V2)
                    .create();   

        } catch (Exception e) {
            System.out.println(e.getMessage());
            e.printStackTrace();
        }
    }
}
```

Run the sample from the command line:

```
mvn compile exec:java
```

You'll see some REST requests and responses in the console as the SDK makes the underlying calls to the Azure REST API to configure the virtual machine and its resources. When the program finishes, verify the virtual machine in your subscription with the Azure CLI 2.0:

```azurecli
az vm list --resource-group sampleResourceGroup
```

Once you've verified that the code worked, delete the VM from the CLI

```azurecli
az vm delete --resource-group sampleResourceGroup --name testLinuxVM
```

## Deploy a web app from a GitHub repo

Replace the code in AzureSDKApp.java with the following class. Update the `appName` variable to a unique value before running the code. This code deploys an code from the `master` branch in a GitHub repo into a new Azure App Service webapp running in a free pricing tier plan.

```java
package com.fabrikam.testSDKApp;

import com.microsoft.azure.management.Azure;
import com.microsoft.azure.management.resources.fluentcore.utils.SdkContext;
import com.microsoft.azure.management.appservice.WebApp;
import com.microsoft.rest.LogLevel;
import com.microsoft.azure.management.resources.fluentcore.arm.Region;

import java.io.File;

public class AzureSDKApp {

    public static void main(String[] args) {
        try {

            final File credFile = new File(System.getenv("AZURE_AUTH_LOCATION"));
            final String appName = "YOUR_APP_NAME";
            
            Azure azure = Azure.configure()
                    .withLogLevel(LogLevel.BASIC)
                    .authenticate(credFile)
                    .withDefaultSubscription();

            WebApp app = azure.webApps().define(appName)
                    .withNewResourceGroup("sampleResourceGroup")
                    .withNewAppServicePlan("testAppServicePlan")
                    .withRegion(Region.US_WEST2)
                    .withFreePricingTier()
                    .defineSourceControl()
                    .withPublicGitRepository(
                       "https://github.com/Azure-Samples/app-service-web-dotnet-get-started")
                    .withBranch("master")
                    .attach()
                    .create();
            
           

        } catch (Exception e) {
            System.out.println(e.getMessage());
            e.printStackTrace();
        }
    }
}
```

Run the code as before using Maven:

```
mvn clean compile exec:java
```

Open up a browser to the application using the Azure CLI:

```azurecli
az appservice web browse --resource-group sampleResourceGroup --name YOUR_APP_NAME
```

## Explore sample code

To learn more about how to use the Azure SDK for Java to manage resources and automate tasks, see our sample code for [virtual machines](java-sdk-azure-virtual-machine-samples.md), [web apps](java-sdk-azure-web-apps-samples.md), [resource groups](java-sdk-azure-resource-groups-samples.md), and [SQL database](java-sdk-azure-sql-databases.md).

## Reference and release notes

A complete [SDK reference](java-sdk-azure-reference.md) is available. Review the [release notes](java-sdk-azure-release-notes.md) to learn about new features, updates and changes to the SDK.

## Get help and give feedback

Post questions to the community on [Stack Overflow](http://stackoverflow.com/questions/tagged/azure+java) and open issues against the SDK on the [project GitHub](https://github.com/Azure/azure-sdk-for-java).