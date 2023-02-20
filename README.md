
[![Build Status](https://dev.azure.com/thoanvo/Ensuring%20Quality%20Releases%20Project/_apis/build/status/thoanvo.devops-training-project03%20(1)?branchName=main)](https://dev.azure.com/thoanvo/Ensuring%20Quality%20Releases%20Project/_build/latest?definitionId=3&branchName=main)


# Table of Contents - Ensuring Quality Releases

- **[Overview](#Overview)**
- **[Dependencies](#Dependencies)**
- **[Azure Resources](#Azure-Resources)**
- **[Installation Configuration Steps](#Installation-Configuration-Steps)**
- **[Monitoring And Logging Result](#Monitoring-And-Logging-Result)**
- **[Clean Up](#Clean-Up)**

## Overview

This project demonstrates how to ensure quality releases using Azure cloud through the implementation of automated testing, performance monitoring and logging using Azure DevOps, Terraform, Apache JMeter, Selenium and Postman.

* To use  a variety of industry leading tools, especially Microsoft Azure, to create disposable test environments and run a variety of automated tests with the click of a button.


![intro](screenshots/intro.png)

## Dependencies
| Dependency | Link |
| ------ | ------ |
| Terraform | https://www.terraform.io/downloads.html |
| JMeter |  https://jmeter.apache.org/download_jmeter.cgi|
| Postman | https://www.postman.com/downloads/ |
| Python | https://www.python.org/downloads/ |
| Selenium | https://sites.google.com/a/chromium.org/chromedriver/getting-started |
| Azure DevOps | https://azure.microsoft.com/en-us/services/devops/ |

## Azure Resources
 - Azure account  
 - Azure Storage account (resource)
 - Azure Log Workspace (resource)
 - Terraform Service principle (resource)
 - Azure CLI (resource)

## Installation Configuration Steps
### Terraform in Azure
1. Clone source repo
2. Open a Terminal in VS Code and connect to your Azure Account and get the Subscription ID

```bash
az login 
az account list
```

3. Configure storage account to Store Terraform state
Direct to `terraform/environments/test`

  Execute the script `configure-tfstate-storage-account.sh` to configure account storage, container storage :

  ```bash
  ./configure-tfstate-storage-account.sh
  ```

  * Take notes of **storage_account_name**, **container_name**, **access_key** . They are will be used in **main.tf** terrafrom files

  ```bash
    backend "azurerm" {
    storage_account_name = "sontv"
    container_name       = "sontv"
    key                  = "terraform.tfstate"
    access_key           = <access_key_value>
    }
  ```
 
  ![azurerm-backend](./screenshots/maintf.png)

### Create a ssh-key for Terraform
On your terminal create a SSH key and also perform a keyscan of your github to get the known hosts.

  ```bash
  ssh-keygen -t rsa
  cat ~/.ssh/id_rsa.pub
  ```

### Azure DevOps
1. Login to Azure DevOPs and perform the following settings before to execute the Pipeline. 

2. Install this Extensions :

  * Terraform (https://marketplace.visualstudio.com/items?itemName=charleszipp.azure-pipelines-tasks-terraform)

3. Create a Project into your Organization

4. Create the Service Connection in Project Settings > Pipelines > Service Connection

    ![img](screenshots/service_connection.png) <br/>
 
5. Add into Pipelines --> Library --> Secure files these 2 files:
  * The private secure file : **id_rsa key**
  * The terraform tfvars file : **terraform.tfvars**

    ![library-secure-files](./screenshots/secure-file.png)

6. Modify the following lines on azure-pipelines.yaml before to update your own repo:
    * Get your "Known Hosts Entry" is the displayed third value that doesn't begin with # in the GitBash results:<br/>
    
    ```bash
    ssh-keyscan github.com
    ```
    * Take note value in highlight below to fill **knownHostsEntry**

    ![azurerm-backend](screenshots/ssh-keyscan.png)

    | # #  | parameter | description |
    | ------ | ------ | ------ |
    | 1| knownHostsEntry |  the knownHost of your ssh-keyscan github |
    | 2 | sshPublicKey |  your public ssh key (from id_rsa.pub) |
    | 3 | storageAccountName | Value from Configure storage account to Store Terraform state |

7. Create groups of variables
  Create groups of variables that you can share across multiple pipelines. 
  * Choose "+Variable groups" > Add name groups of variables : ssh-config > Add **knownHostsEntry**, **sshPublicKey**, **StorageAccountName** and value this in Variables > Select type to secret > Save <br/>
  ![screen shot](screenshots/variables.png) <br/>  

8. Create a New Pipeline in your Azure DevOPs Project
  - Located at GitHub
  - Select your Repository
  - Existing Azure Pipelines YAML file
  - Choosing **azure-pipelines.yaml** file

    8.1. Tab Pipelines -> Create Pipeline -> Where is your code? Choose Github (Yaml) -> Select Repo -> Configure your pipeline
    : Choose "Existing Azure Pipelines yaml file" > Continue > Run <br/>
      ![img](screenshots/config-pipeline.png) <br/>

      ![img](screenshots/run-pipeline2.png) <br/>

    8.2. Apcept permission for Azure Resources Create with terraform <br/>
      ![img](screenshots/terra-permit.png) <br/>

    8.3. Go to Azure pipeline -> Environments -> you can see Environments name is "TEST" -> Choose and select "Add resource" -> choose "Virtual machines" > Select "Linux" and Choose icon "Copy command ..." > Close <br>
    Something similar to </br>
      ![img1](screenshots/new-env1.png) </br>

      ![img2](screenshots/new-env2.png) </br>
    

    8.5. SSH into the VM created using the Public IP -> Enter command you just copy above step -> Run it -> Success if you see result like this 
    ![img3](screenshots/env-conn.png) <br>

    Get at the end a result like:

    ![img5](screenshots/environment.png)

    8.6. Back to pipeline and re-run


9. Wait the Pipeline is going to execute on the following Stages:

    Azure Resources Create --> Build --> Deploy App --> Test

    ![img](./screenshots/pipeline-stages.png)

### Configure Logging for the VM in the Azure Portal
1. Create a Log Analytics workspace. It will be created on the same RG used by terraform

    ![img](./screenshots/log-analytic.png)

2. Set up email alerts in the App Service:
 
 * Log into Azure portal and go to the AppService that you have created.
 * On the left-hand side, under Monitoring, click Alerts, then New Alert Rule.
 * Verify the resource is correct, then, click “Add a Condition” and choose Http 404
 * Then, set the Threshold value of 1. Then click Done
 * After that, create an action group and name it myActionGroup, short name mag.
 * Then, add “Send Email” for the Action Name, and choose Email/SMS/Push/Voice for the action type, and enter your email. Click OK
 * Name the alert rule Http 404 errors are greater than 1, and leave the severity at 3, then click “Create”
  Wait ten minutes for the alert to take effect. If you then visit the URL of the app service and try to go to a non-existent page more than once it should trigger the email alert.

3. Log Analytics
  * Go to the `App service* Diagnostic Settings > + Add Diagnostic Setting`. Tick `AppServiceHTTPLogs` and Send to Log Analytics Workspace created on step above and  `Save`. 

  * Go back to the `App service > App Service Logs `. Turn on `Detailed Error Messages` and `Failed Request Tracing` > `Save`. 
  * Restart the app service.

4. Set up log analytics workspace properly to get logs:

  * Go to Virtual Machines and Connect the VM created on Terraform to the Workspace ( Connect). Just wait that shows `Connected`.

    * Set up custom logging , in the log analytics workspace go to Advanced Settings > Data > Custom Logs > Add > Choose File. Select the file selenium.log > Next > Next. Put in the following paths as type Linux:

    /var/log/selenium/selenium.log

    Give it a name ( `selenium_logs_CL`) and click Done. Tick the box Apply below configuration to my linux machines.

  * Go to the App Service web page and navigate on the links and also generate 404 not found , example:

    ```html
    https://sontv-app-project3-appservice.azurewebsites.net

    https://sontv-app-project3-appservice.azurewebsites.net/asaf  ( click this many times so alert will be raised too)
    ```

    ![img](./screenshots/raise_alert.png)

  * After some minutes (about 15 minutes) , check the email configured since an alert message will be received. and also check the Log Analytics Logs , so you can get visualize the logs and analyze with more detail.

  ![img](screenshots/mail-alert.png)


## Monitoring And Logging Result
Configure Azure Log Analytics to consume and aggregate custom application events in order to discover root causes of operational faults, and subsequently address them.

### Environment Creation & Deployment
  #### The pipeline build results page
  ![Pipeline-Result](screenshots/pipeline-stages.png)

  #### Terraform Init
  ![Terraform](screenshots/terra-init.png)

  #### Terraform Validate
  ![Terraform](screenshots/terra-validate.png)

  #### Terraform Plan
  ![Terraform](screenshots/terra-plan.png) <br>

  
  #### Terraform Apply
  ![Terraform](screenshots/terra-apply.png) <br>
  <br>
### Automated Testing

  #### FakeRestAPI
  ![DeployWebApp](screenshots/deploy-webapp.png)

  ![FakeRestAPI](screenshots/fakeapi-app.png)

  #### JMeter log output

  ![JMeterLogOutput](screenshots/stress-run.png)

  ![JMeterLogOutput](screenshots/end-run.png)

  #### JMeter Stess/Endurance Test Results                                                                   
 ![JMeterLogOutput](screenshots/stress-test.png)

  ![JMeterLogOutput](screenshots/endur-test.png)

  #### Selenium
  ![Selenium test](screenshots/selenium.png)

  #### Regression Tests
  ![Regression test](screenshots/reg-test-az.png)

  ![Regression test](screenshots/reg-test-result.png)

  ![Regression test](screenshots/test-runs.png)

  #### Validation Tests
  ![Validation test](screenshots/validation-test.png)

  ![Validation test](screenshots/validate-test-result.png)

  ![Validation test](screenshots/test-runs.png)

  #### The artifact is downloaded from the Azure DevOps and available under the /projectartifactsrequirements folder.



### Monitoring & Observability

  #### Alert Rule:
  ![Alert Rule](screenshots/alert-rule.png)

  ![The Triggered Alert](screenshots/alert.png)


  #### Triggered Alert:
  ![Triggered Email Alert](screenshots/alert-report.png)

  ![The Graphs 404 Alert](screenshots/mail-alert.png)

  #### Logs from Azure Log Analytics
    
    Go to Log Analytics Workspace , to run the  following queries:

    ```bash
    AppServiceHTTPLogs
    | where TimeGenerated < ago(2h)
      and ScStatus == '404'
    ```

  ![Log Analytics](screenshots/app-log.png)
    
## Clean Up

* On Az DevOps Pipeline , give approval on the notification to resume with the Destroy Terraform Stage.


