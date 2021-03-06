# Challenge01 - Azure API for FHIR

## Scenario
Your team has now learned a little about the FHIR standard. Our first challenge focuses on you as a software developer on the data interoperability team. **Any Healthcare’s** strategic direction is to build new solutions on public cloud resources wherever possible.

The first task has landed on you: In order to learn a bit about the capabilities of Azure’s API for FHIR API, you will set up a development instance. This development instance is designed to highlight the key features, by building a demo-like environment. Once you are done with the setup you will verify the service is functioning by loading some synthetic data. The data is generated using Synthea which allows you to mimic EMR/EHR data. You will then use the dashboard app and run some basic queries to ensure the core functionality is working. 

## Reference Architecture
<center><img src="../images/challenge01-architecture.png" width="550"></center>

## To complete this challenge successfully, you will perform the following tasks.

* **Provision Azure API for FHIR API demo environment**. Given the limited time, we'll provide a set of scripts to accomplish this. For step by step instructions, check the appendix.
* **Load Synthetic data**. You can generate the data using Synthea or use a staged dataset that we'll provide.
* **Validate data load**. You can use the dashboard application to validate the data or the provided APIs by using Postman.

## Before you start

Make sure you have completed the pre-work covered in the previous challenge: [Challenge00 - Pre-requisites: Technical and knowledge requirements for completing the Challenges](../Challenge00-Prerequistes/ReadMe.md).

* **Azure Subscription**: You will need permissions to perform CRUD operations in the Subscription.

* **Create Secondary AD tenant**: Azure API for FHIR needs to be deployed in Active Directory tenants for Data control plane authorization and Resource control plane for resources. Most of the companies lock down Active Directory App Registration for security where you can't publish App, register roles or grant permissions. So, you will create local Active Directory which is free. This will 
   * Go to Portal, navigate to Azure Active Directory. Click "Create a tenant". Give it a name for example: "{yourname}fhirad" for Organization name and Initial domain name and click Create. This will be referred to as **Secondary AD** in this page for clarity. 

* **PowerShell modules**: Primary script deployer is going to be PowerShell. Using either the Azure PowerShell or Windows PowerShell and make sure you are running it as administrator. 
   * Get PowerShell module version: Make sure your version is 5.1.19041.1. If not, install this version.

   ```powershell
   $PSVersionTable.PSVersion
   ```  
   * Get Azure PowerShell module versions: If your results show Az version 4.1.0 and AzureAd version 2.0.2, then proceed to login step. If not, get the right versions.

   ```powershell
   Get-InstalledModule -Name Az
   Get-InstalledModule -Name AzureAd
   ```  

   * Uninstall and Re-install PowerShell modules: Uninstall Az and AzureAd modules and install the right version needed.
   ```powershell
   Uninstall-Module -Name Az
   Uninstall-Module -Name AzureAD
   ```  

   ```powershell
   Install-Module -Name Az -RequiredVersion 4.1.0 -Force -AllowClobber -SkipPublisherCheck
   Install-Module AzureAD -RequiredVersion 2.0.2.4
   ```  

   * Close PowerShell and open a new session.

   * Login to your Azure account where you want to deploy resources and authenticate. This will be called **Primary AD** in this page for clarity.
   ```powershell
   Login-AzAccount
   ```

   If you are seeing errors, or you don't see the subscription in your **Primary AD** you want to deploy, you might have wrong Azure Context. Clear, Set and verify Az Context.
   ```powershell
   Clear-AzContext
   ```
   ```powershell
   Set-AzContext -TenantId **{YourPrimaryADTenantID}**
   ```
   ```powershell
   Get-AzContext
   ```

   * Connect to **Secondary AD** and authenticate
   ```powershell
   Connect-AzureAd -TenantDomain **{yourname}fhirad.onmicrosoft.com**
   ``` 

   * You will get security exception error if you haven't set the execution policy below. This is because the repo you will clone in the next step is a public repo, and the PowerShell is not signed and so you might not have access to run the script. So, set the execution policy and type A, to run unsigned Powershell scripts.
   ```powershell
   Set-ExecutionPolicy -Scope Process -ExecutionPolicy ByPass
   ```

## Getting Started

## Task #1: Provision Azure API for FHIR demo environment.

* **Get the repo** fhir-server-samples from Git. If you don't have Git, install from link in [Challenge00](../Challenge00-Prerequistes/ReadMe.md).
   ```powershell
   git clone https://github.com/Microsoft/fhir-server-samples
   ``` 
* Navigate to the scripts directory where the Repo was downloaded to. Run the **one shot deployment.** Don't forget the **.\** before Create. Make sure to leave $true for EnableExport as it will needed in Challenge03.
   ```powershell
   cd fhir-server-samples/deploy/scripts
  .\Create-FhirServerSamplesEnvironment.ps1 -EnvironmentName <ENVIRONMENTNAME> -EnvironmentLocation eastus -UsePaaS $true -EnableExport $true
   ```
   * The **ENVIRONMENTNAME Example:fhirhack** is a value you type that will be used as the prefix for the Azure resources that the script deploys, therefore it should be globally unique, all lowercase and can't be longer than 13 characters. 
   * If EnvironmentLocation is not specified, it defaults to westus.
   * This is a PaaS, so leave it as $true.
   * When EnableExport is set to $true, bulkexport is turned on, service principle identity is turned on, storage account for export is created, access to storage account added to FHIR API through managed service identity, service principle identity is added to storage account.
   * If all goes well, the script will kickoff and will take about 10-15 minutes to complete. If the script throws an error, please check the **Help I'm Stuck!** section at the bottom of this page.
   
* On **successful completion**, you'll have 2 resource groups and resources created with prefix as your ENVIRONMENTNAME. Explore these resources and get a feel what role they play in the FHIR demo environment. NOTE: As AppInsights is not available in all location, by default will be created in East US.

   The following resources in resource group **{ENVIRONMENTNAME}** Ex:fhirhack will be created:
   <center><img src="../images/challenge01-fhirhack-resources.png" width="550"></center>

   * Azure API for FHIR ({ENVIRONMENTNAME}) is the FHIR server
   * Key Vault ({ENVIRONMENTNAME}-ts) stores all secrets for all clients (public for single page apps/javascripts that can't hold secrets, confidential for clients that hold secrets, service for service to service) needs to talk to FHIR server.
   * App Service/Dashboard App ({ENVIRONMENTNAME}dash) used to analyze data loaded.
   * App Service Plan ({ENVIRONMENTNAME}-asp) to support the App Service/Dashboard App.
   * Function App ({ENVIRONMENTNAME}imp) is the import function that listens on the import storage account where you can drop bundles that get loaded into FHIR server.
   * App Service Plan ({ENVIRONMENTNAME}imp) to support the Function App.
   * Application Insights ({ENVIRONMENTNAME}imp) to monitor the Function App.
   * Storage Account ({ENVIRONMENTNAME}export) to store the data when exported from FHIR server.
   * Storage Account ({ENVIRONMENTNAME}impsa) is the storage account where synthetic data will be uploaded for loading to FHIR server.


   The following resources in resource group **{ENVIRONMENTNAME}-sof** will be created for SMART ON FHIR applications:
   <center><img src="../images/challenge01-fhirhacksof-resources.png" width="550"></center>

   * App Service/Dashboard App ({ENVIRONMENTNAME}growth) supports {ENVIRONMENTNAME}dash App.
   * App Service Plan ({ENVIRONMENTNAME}growth-plan) to support the growth App Service/Dashboard App.
   * App Service/Dashboard App ({ENVIRONMENTNAME}meds) supports {ENVIRONMENTNAME}dash App.
   * App Service Plan ({ENVIRONMENTNAME}meds-plan) to support the meds App Service/Dashboard App.

* Go to the **Secondard AD** in Portal. Go to App Registrations. All the 3 different clients are registered here.

## Task #2: Generate & Load synthetic data.

* ### Option 1: Generate Synthea data
   * **Setup Synthea**: 
      * This section shows how to setup and generate health records with [Synthea](https://github.com/synthetichealth/synthea). 
      * Synthea requires [Java 8 JDK](https://www.oracle.com/java/technologies/javase/javase-jdk8-downloads.html). Make sure to select the JDK and not the JRE install.
      * After successfull install of Java 8, download the [Synthea Jar File](https://github.com/synthetichealth/synthea/releases/download/master-branch-latest/synthea-with-dependencies.jar) or open command prompt and run, .jar file will be downloaded to directory you are running the command from.
      ```cmd
      curl https://syntheahealth.github.io/synthea/build/libs/synthea-with-dependencies.jar --output synthea-with-dependencies.jar
      ```
   * **Generate Data**:
      * Follow instructions below to generate your synthetic data set. Note that, we are using the Covid19 module (-m "covid19") and generating a 50 person (-p 50) sample. 50 patients and related resources will be downloaded as json files to a output sub-folder.
      ```shell
      cd {directory_you_downloaded_synthea_to}
      java -jar synthea-with-dependencies.jar -m "covid19" -p 50
      ```
      * Once the data has been generated, you can use the Azure Storage Explorer in Portal or from your desktop App to upload the data into the **fhirimport** folder in **{ENVIRONMENTNAME}impsa** storage account. 
      * Once the data is loaded into **fhirimport** folder, the Azure function {ENVIRONMENTNAME}imp will be triggered to start the process of importing the data into {ENVIRONMENTNAME} FHIR instance. For 50 users, assuming the default of 1000 RUs for the Azure CosmosDB, it will take about 5 minutes. You can go to the storage account and click Monitor to view status.

* ### Option 2: Use Staged data
   * For this option, we have already generated the sample data and loaded it into a publicly available storage account. The account URL and SAS token are included below.
      ```shell
      Account URL: https://a368608impsa.file.core.windows.net/
      SAS Token: ?sv=2019-12-12&ss=bfqt&srt=c&sp=rwdlacupx&se=2020-08-21T05:50:18Z&st=2020-08-20T21:50:18Z&spr=https&sig=hLoeY7kq3B%2FXvmJsBLboMsdMmMnv%2F2liAX3l231ux00%3D
      ```
      * Once the data has been generated, you can use the Azure Storage Explorer in Portal or from your desktop App to upload the data into the **fhirimport** folder in **{ENVIRONMENTNAME}impsa** storage account. 
      * Once the data is loaded into **fhirimport** folder, the Azure function {ENVIRONMENTNAME}imp will be triggered to start the process of importing the data into {ENVIRONMENTNAME} FHIR instance. For 50 users, assuming the default of 1000 RUs for the Azure CosmosDB, it will take about 5 minutes. You can go to the storage account and click Monitor to view status.

## Task #3: Validate data load

* ### Use the Dashboard App
    * Go to **Secondary AD** tenant. Go to Azure AD, click on Users. Part of the deployment will create an admin user {ENVIRONMENTNAME}-admin@{yournamefhirad}.onmicrosoft.com. Click on the admin user and Reset password.
    * Go to **Primary AD** tenant. Click on the App Service "{your resource prefix}dash". Copy the URL. Open Portal "InPrivate" window. Go to the App Service URL and login using the admin user above. 
    * The dashboard will show you all the patients in the system and allows you to see the patients medical details. You can click on little black **fire** symbol against each records and view fhir bundle and details.
    * You can click on resource links lik Condition, Encounters...to view those resource. 
    * Go to Patients, and click on little black **i** icon next to a patient record. You will notice 2 buttons "Growth Chart" and "Medications" SMART ON FHIR Apps.
 
* ### Use Postman to run queries
    * Download [Postman](https://www.postman.com/downloads/) if you haven't already.
    * Open Postman and import [Collection](../Postman/FHIR%20OpenHack.postman_collection.json).
    * Import [Environment](../Postman/FHIR%20OpenHack.postman_environment.json). An environment is a set of variables pre-created that will be used in requests. Click on Manage Environments (a settings wheel on the top right). Click on the environment you imported. Enter these values for Initial Value:
      * adtenantId: This is the **tenant Id of the Secondary AD** tenant
      * clientId: This is the **client Id** that is stored in **Secret** "{your resource prefix}-confidential-client-id" in "{your resource prefix}-ts" Key Vault.
      * clientSecret: This is the **client Secret** that is stored in **Secret** "{your resource prefix}-confidential-client-secret" in "{your resource prefix}-ts" Key Vault.
      * bearerToken: The value will be set when "AuthorizeGetToken SetBearer" request below is sent.
      * fhirurl: This is **https://{your resource prefix}.azurehealthcareapis.com** from Azure API for FHIR you created in Task #1 above
      * resource: This is the Audience of the Azure API for FHIR **https://{your fhir name}.azurehealthcareapis.com** you created in Task #1 above.      
   * Import [Collection](../Postman/FHIR%20OpenHack.postman_collection.json). Collection is a set of requests.
   * After you import, you will see both the Collection on the left and Environment on the top right.
      <center><img src="../images/challenge01-postman.png" width="850"></center>
   * Run Requests:
      * Open "AuthorizeGetToken SetBearer", make sure the environment you imported is selected in the drop-down in the top right. click Send. This should pass the values in the Body to AD Tenant, get the bearer token back and assign it to variable bearerToken. Shows in Body results how many seconds the token is valid before expires_in. 
      * Open "Get Metadata" and click Send. This will return the CapabilityStatement with a Status of 200 ....This request doesn't use the bearerToken.
      * Open "Get Patient" and click Send. This will return all Patients stored in your FHIR server. Not all might be returned in Postman.
      * "Get Patient Count" will return Count of Patients stored in your FHIR server.
      * "Get Patient Sort LastUpdated" will returns Patients sorted by LastUpdated date. This is the default sort.
      * "Get Patient Filter ID" will return one Patient with that ID. Change the ID to one you have loaded and analyze the results.
      * "Get Patient Filter Missing" will return data where gender is missing. Change to different column and analyze the results.
      * "Get Patient Filter Exact" will return a specific Patient with a given name. Change to different name and analyze the results.
      * "Get Patient Filter Contains" will return Patients with letters in the given name. Change to different letters and analyze the results.
      * "Get Filter Multiple ResourceTypes" will return multiple resource types in _type. Change to other resource type and analyze the results.
      * NOTE: bearerToken expires in ...so if you get Authentication errors in any requests, re-run "AuthorizeGetToken SetBearer" to set new value to bearerToken variable.


## Congratulations! You have successfully completed Challenge01! 

## Help, I'm Stuck!
Below are some common setup issues that you might run into with possible resolution. If your error/issue is not here and you need assistance, please let your coach know.

* **{ENVIRONMENTNAME} variable error**: EnvironmentName is used a prefix for naming Azure resources, you have to adhere to Azure naming guidelines. The value has to be globally unique and can't be longer than 13 characters. Here's an example of an error you might see due to a long name.
   <center><img src="../images/challenge01-errors-envname-length.png" width="850"></center>

* **PowerShell Execution Policy errors**: are another type of error that you might run into. In order to allow unsigned scripts and scripts from remote repositories, you might see a couple of different errors documented below.
   <center><img src="../images/challenge01-powershell-executionpolicy-1.png" width="850"></center>
   <center><img src="../images/challenge01-powershell-executionpolicy-2.png" width="850"></center>

   To allow PowerShell to run these scripts and resolve the errors, run the following command:
   ```powershell
   Set-ExecutionPolicy -Scope Process -ExecutionPolicy ByPass
   ```
* **Git Missing**: This challenge uses scripts from Git that are downloaded and installed. If you don't have Git installed you might see the following error or something similiar. Get [Git](https://git-scm.com/downloads) and try again.
   <center><img src="../images/challenge01-git-client-install.png" width="850"></center>


***

[Go to Challenge02 - HL7 Ingest and Convert: Ingest HL7v2 messages and convert to FHIR format](../Challenge02-HL7IngestandConvert/ReadMe.md)
