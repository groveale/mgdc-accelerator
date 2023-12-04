# **Microsoft Graph Data Connect Lab Guide**

## 1 Getting Started

### **Introduction**

Microsoft Graph Data Connect (MGDC) allows developers to extract data from Microsoft 365 for use in other applications. This guide first guide will walk you through the process of setting up MGDC and running your first data extraction.

### **Prerequisites**

Before you begin, you’ll need:

1. An Office 365 tenant with administrative access.
2. An Azure subscription linked to the above tenant. 
3. Access to Azure Synapse Studio

### **Step 1: Enable MGDC in M365 tenant**

First step, we need to enable MGDC in our tenant. 

1. Navigate to the Admin Centre portal.
2. Open Settings > Org Settings > Services Tab
3. Scroll down and look for Microsoft Graph Data Connect. Or click [here]([Settings - Microsoft 365 admin center](https://admin.microsoft.com/#/Settings/Services/:/Settings/L1/O365DataPlan))
4. Check "Turn on Microsoft Graph Data Connect on..." Also enable SPO and OneDrive Datasets. This is what we will be working with in the lab
5. Click Save!

Hopefully you've already done this, but if not - Do it now and we will set you up an account in either mine or Pete's environment so you can continue with todays lab. 
### **Step 2: Set up the Apps**

We will need to create an app registration in Entra. This app registration will be tied to an MGDC app (created in the Azure subscription). This is how the billing and authorization / permissions work.

#### 2.1 Create an App Registration in Entra

1. Navigate to the Azure Portal
2. Click on Microsoft Entra Id
3. Click Add > App Registration. Give this a name. Something like MGDC hack lab. If you are in mine or Pete's tenant then also include your name in the name so you can find the right one :D
4. Click Register!

> [!NOTE]  
> We will need to come back and generate a secret but let's not do that yet.

#### 2.2 Create a Resource Group and Storage Account
We need a resource group for the MGDC app and a storage account. We will also grant the App Id above a role on our storage account so that it has permission to copy the MGDC data to it.

Resource Group
1. Azure Portal > Resource Groups
2. Create +
3. Give it a name and region. Choose the same region as your M365 tenant
4. Create! 

> [!IMPORTANT]  
> The region must match your tenant otherwise MGDC will fail to copy the data. The error message won't indicate that you are in the wrong region. But if the copy is taking longer than 30 minutes you can be pretty confident you are in the wrong region

Storage Account
1. Azure Portal > Storage Accounts
2. Create + 
3. Choose the resoruce group just created
4. Give it a name and let it inherit the same region
5. Leave everything else default and go to the Advanced tab
6. Check Enable hierarchical namespace. Leave everything else
7. Review > Create!

Storage Account Access Policy
1. From the newly created storage account go to IAM blade
2. Click + Add > Role Assignment on the IAM menu
3. Search for "Storage Blob Data Contributor".
4. Select the role and click Next
5. Click on add members and search your app reg you created in Step 2.1
6. Select the app and click Review + assign
7. You should see an Azure notification for the role being successfully assigned

Create a Container
1. Now click on the Container blade
2. In the container menu slick + Container and give it the name sites

> [!Info]  
> This is where are raw data from MGDC will end up!
#### 2.3 Create an MGDC App in Azure Subscription

1. Go back to the Azure Portal
2. Search for "MGDC". You should see "Microsoft Graph Data Connect" listed as a service in your search results
3. Click Add. 
4. Fill out the instance details: 
	**App Id:** Select the App Id you just created.
	**Description:** Chat's and Hacks lab
	**Publish Type:** Single-Tenant
	**Compute Type:** Azure Synapse
	**Activity Type:** Copy Activity
5. Fill out the project details:
	Select your Sub, if using my tenant go for "Microsoft Azure Sponsorship 2"
	**Resource Group**: Use the one you just created.
	**Destination Type:** Azure Storage Account
	**Storage Account:** Use the one you just created.
	**Storage Account Uri**: Select the uri with .dfs from the dropdown
6. Click Next: Datasets!
7. Datasets, select the following two datasets. For columns choose All
	BasicDataSet_v0.SharePointSites_v1
	BasicDataSet_v0.SharePointPermissions_v1
8. Review + Create!

> [!NOTE]  
> You can update the datasets that your app will request access to at any time. It's worth having a browse of the datasets to see if there is anything you think could be useful!

Below slide indicates the data currently avalible and the data we can expect to see in the future. Copilot 👀

![[Pasted image 20231128171430.png]]
### **Step 3: Approve the MGDC app in Admin Centre**

1. Navigate back to M365 Admin Centre
2. Open Settings > Org Settings > Security & privacy Tab
3. Click MGDC apps or use this [link](Settings - Microsoft 365 admin center]https://admin.microsoft.com/#/Settings/MGDCAdminCenter)
4. You should see your app listed here with status "Pending Approval"
5. Click the app and follow the approval workflow

> [!Oops]  
> You will need to login with another admin account to approve the app! If you are in mine or Pete's tenant then ask us to go in and approve for you!
### **Step 4: Creating a Synapse Pipeline

#### 4.1 The Pipeline

1. Navigate to Azure Synapse Studio in the Azure portal.
	If you don't have Synapse workspace then please create one in your tenant
2. Click on new > Pipeline
	This will open the Pipeline editor / canvas
3. Give your Pipeline a name on the right has pane.
4. Drag and drop the ‘Copy Data’ activity onto the canvas from the activities toolbar to the left of the canvas (Move and transform)
5. Give this a name in the bottom pane: Go for  "Copy Sites"

#### 4.2 The Data Source
Now we must configure the "Source", this is the MGDC dataset that we want to extract from the tenant.

1. Click on the Source Tab in this bottom pane. If you don't see the bottom pane click on the Copy sites activity in the canvas.
2. Under the source dataset drop down click New. This will open window from right
3. Search for 365 in the data store box. Click the M365 tile
4. Give this a name, something like "HackLabSitesSource". 
5. In the Linked service drop down click "+ New"
    **Name**: Microsoft365HackLab
    **Service Principal Id**: app id from earlier (Entra app registration not MGDC app)
    **Service Principal Key**: Create a secret for the app reg and enter it here
7. Test Connection - Hopefully this works, if not give me a shout!
8. Click create. You will be taken back to the data set source menu.

> [!Note]  
> You will only need to create this "Linked Service" once. Once created it can be reused for other datasets and other pipelines!

9. Select BasicDataSet_v0.SharePointSites_v1 from the dropdown
10. Click OK. You will be taken back to the canvas.

#### 4.3. Configure the source data set
Now that we have chosen our source dataset we need to set our date filter and import the dataset schema. The date filter is essentially when we are requesting the snapshot. 

Set the date filter
1. On the source tab, scroll down to Date filter.
2. Select SnapshotDate from the column name dropdown.
3. Select a Start date
4. Select the same date as an end date

> [!Note]  
> The start date must be at least 2 days ago. Choosing a start date and end of the same day instructs MGDC to perform  full pull. Choosing different dates is a delta pull. More on this later 😎

Import the Schema
1. Click the import schema button
2. You are done!

> [!Note]  
> This is where you can remove columns from the dataset if needed. Leave all for now
#### 4.4 The Data Sink (Target)
Now we must configure the "Sink", this is where the copy data tool will export our MGDC data to. Click on the 

1. Click on the Sink Tab in this bottom pane. If you don't see the bottom pane click on the Copy sites activity in the canvas.
2. Under the sink dataset drop down click New. This will open window from right
3. Click the Azure Data Lake Storage Gen2 tile
4. Click Binary. Give the sink a name, something like "HackLabSitesTarget"
5. In the Linked service drop down click "+ New"
    **Name**: ServicePrincipalHackCon
    **Authentication type**: Service Principal
    **Account selection method**: From Azure Sub
    **Storage Account**: From earlier
    **Authentication**: 
		**Tenant**: Your tenant Id
		**Service Principal Id**: App reg from earlier
		**Cred Type:** Key
		**Service Principal Key**: You can use the same secret, but if you haven't saved that. Good on you, go and get another!
6. Test Connection - Hopefully this works, if not give me a shout!
7. Click create. You will be taken back to the data set sink menu.

> [!Again]  
> You will only need to create this "Sink Service" once. Once created it can be reused for other sinks and other pipelines!

8. Use the browse button, You should see a single container "sites". This was created earlier. Select this and click OK
9. Finally click the last OK, you will be taken back to the canvas.

Now we have the source and sink configured and linked in our copy data tool it's time to finally run the pipeline.
### **Step 5: Running the Pipeline**

1. Click on ‘Validate’ to check your pipeline for errors. If there are error. Let me know and I will help you figure them out.
2. If no errors are found, click on ‘Publish’ to save your pipeline.
3. Click on 'Add Trigger' > ‘Trigger Now’ to run your pipeline.
4. Go to monitor and view your progress!

### **Conclusion**

You’ve now successfully set up MGDC and run your first data extraction! This first lab has been overly verbose to make sure you have everything in place for the next lab exercise where we  will be turning this sample into a more production ready solution!

Happy hacking! 😊