# **Microsoft Graph Data Connect Lab Guide**

## 2 Integrating Notebooks and Deltas

### **Introduction**

Well done! You have now you have successfully completed a full pull of your sites dataset. Go and download that .json file from your storage account. Pretty cool right?

So that's all well and good but quite a lot of effort for site report... To really make use of the power of MDGC we need to add further datasets and handle deltas pulls

> [!NOTE]  Deltas
> Deltas are great way to reduce the MDGC cost and keep your dataset up-to-date!

### **Prerequisites**

Before you begin, you’ll need:

1. Completed lab 1 😎

### **Step 1: Parameterise the Pipeline

It's good practise to parametrise the pipeline. This enable you to quickly change config without needing to go and edit settings all over the pipeline. 

1. Click on the canvas. In the bottom window  > Parameter tab > New. Create the following parameters:

|Name|Type|Default value|
|---|---|---|
|StartDate|String|2023-11-30T00:00:00Z|
|EndDate|String|2023-11-30T00:00:00Z|
|StorageAccountName|String|mgdclab|
|StorageContainerName|String|sites-perms-hacklab|
|RetainForHistroic|Bool|true|

Nice, we will use these parameters throughout the pipeline to configure various settings.

>[!IMPORTANT] 
> Use the StorageAccountName and container that match your enviroment.
### **Step 2: Parametrise the Copy Sites Activity

There are quite a few things to change here, but trust me it will be worth it.

#### 2.1 Update the Source

1. On the source tab, go to the Start time you configured in the previous lab and click on the value. You should see some text pop up "Add dynamic content". Click this.
2. In the flyout menu you should see the parameters you just created. Click on StartDate. This will add the dynamic content to the expression builder
3. OK!
4. Do the same for End time. But use the EndDate parameter.
#### 2.2 Update the Sink

1. On the Sink tab, click on the Open button. This will open a new tab in your canvas for the HackLabSitesTarget dataset. Switch to the Parameters tab.
2. Create the following parameters. Leave default value empty

|Name|Type|Default value|
|---|---|---|
|StartDate|String||
|EndDate|String||
|RunId|String||
|StorageContainerName|String||

3. Go back to the container tab. We will build the Filepath. Click the File system box > Add dynamic content. Choose StorageContainerName
4. Now the directory, this one is fun. Copy the below and add it the the expression builder box and click OK!

```d
@concat('raw/sites/',formatDateTime(dataset().EndDate, 'yyyy'),'/', formatDateTime(dataset().EndDate, 'MM'),'/',formatDateTime(dataset().EndDate, 'dd'),'/',dataset().RunId)
```

This expression will end up generating a path in your storage account as bellow

```
https://mgdcag.blob.core.windows.net/sites-perms-hacklab/raw/sites/2023/11/26/fd3acea4-d6a6-4b8b-b347-10d42a9039f3
```

> [!NOTE] 
> We use EndDate rather than StartDate. This is to support deltas. For a full pull, StartDate and EndDate will be the same. So the path  would be accurate, for a delta pull, StartDate and EndDate are different but the data is up-to-the EndDate. So it makes sense to add the data to a folder with the EndDate

#### 2.3 Populate the Sink Parameters

Now that our sink has parameters we need to populate the values with the parameters from the pipeline. The sink will essentially inherit the parameters from the pipeline.

> [!INFO] 
> Why can't we use the pipline parameters in the Sink? Well the sink dataset it's its own object within Synapse Studio. It can be reused in other Piplines.. But don't worry about this right now

1. Go back to the canvas, Copy Sites > Sink. Fill out the dataset properties.
2. Same as before, click on the StartDate value box > Add dynamic content. Choose StartDate from the parameters list. Click OK
3. Repeat for EndDate and StroageContainerName
4. RunId is a little different, this is the RunId of the Pipeline. Add the following in the permission builder for RunId and click OK.
```d
@pipeline().RunId
```

#### 2.4 Test the new config

Before we continue It's worth testing our new parametrised configuration. To do this, we must first publish our pipeline changes.  

1. Click Publish all, you should see a flyout menu with the changes. Give me a shout if you see any errors here.
2. Now click Add Trigger > Trigger now. A flyout menu will appear. Enter in some values. Make sure these values are reflective of your environment and click OK

> [!NOTE] 
> You might want to create a new container in the storage account for the Sites + Permissions pipeline that we are building out in this lab

3. Monitor the status of the activity - Go and get a brew as this can take about 15 mins ☕

### **Step 3: Add Copy Permissions Activity

We now need to add another dataset. This will be the permissions dataset. We've already covered how to do this for sites, so instructions here will be light. But feel free to go back and use as a reference.

1. Pull in a new Copy data activity
2. Rename to Copy Permissions
3. Add a new source dataset (365). Call this `PermissionsSource`. Reuse the Linked service from the previous lab.
4. Select the `BasicDataSet_v0.SharePointPermissions_v1` table and click OK
5. Wire up the Date filter, remember to use the parameters for Start and End time.
6. Import the schema
7. Add a new Sink dataset (ADLS2). Call this `HackLabPermissionsTarget`. Reuse the Linked service from the previous lab. Leave the file path blank for now. OK!
8. Open your newly created sink dataset and add the same four parameters as we did last time
9. Go back to the Connection tab and configure the file path using dynamic content. For the directory i'll give you that again. Same as sites, but in a `permissions` sub folder
```d
@concat('raw/permissions/',formatDateTime(dataset().EndDate, 'yyyy'),'/', formatDateTime(dataset().EndDate, 'MM'),'/',formatDateTime(dataset().EndDate, 'dd'),'/',dataset().RunId)
```

10. Populate the sink dataset property values with the pipeline parameters, same as before. (the sink tab on the copy data activity)
11. Finally publish all those pipeline changes!

> [!NOTE] 
> You can test the pipeline again here. The folder structure we created will support multiple runs per day.

### **Step 4: Add a Synapse Notebook

Now we are very much wading into the Data and AI realms. Before we create a Notebook we need to provision some commute to run in. This will be a spark pool.
#### 4.1 Create a Spark pool

1. Go to your Synapse Workspace resource in the Azure Portal
2. Use the top ribbon to create a new Apache Spark pool
3. In the new spark pool dialogue, give it a name and reduce the node size to Small. Let try and protect those credits
4. Review + Create > Create!

#### 4.2 Add a Notebook Activity
We need a way of reading in our data from MGDC and performing some data engineering. This will include adding additional data / calculation and handling deltas. This is all achieved through a notebook

1. Pull in a Synapse Notebook activity into the pipeline
2. Give it a name `SitesPermsNotebook`
3. Configure the Notebook to run on Success of the two Copy data activities. Click on the tick and drag the arrow to the notebook. Repeat

![[Pasted image 20231129234615.png]]

4. On the Settings and the following base parameters. Remember dynamic content 😎

|Name|Type|Default value|
|---|---|---|
|windowStartTime|String|@pipeline().parameters.StartDate|
|windowEndTime|String|@pipeline().parameters.EndDate|
|runId|String|@pipeline().RunId|
|storageAccountName|String|@pipeline().parameters.StorageAccountName|
|storageContainerName|String|@pipeline().parameters.StorageContainerName|
|retainForHistoricTrending|Bool|@pipeline().parameters.RetainForHistoric|

5. Select the sparkpool you just created and the executor size

#### 4.2 Create or Import a Notebook

You have a choice, take the red pill or the blue. Go for blue if you are short on time. 

* Download .ipynb file from here and import in to synapse workspace 🔹
* Click new and write the scala notebook from scratch.  🔺

##### 4.2.1 Import and wire up 🔹

To import the notebook go to the develop tab (far left rail). This will open up the list of notebooks (this will likely be empty). 

1. Click on the Plus > Import
2. Navigate to the notebook you just downloaded.
3. Now go back to your pipeline. Click on the Notebook activity > Settings Tab
4. On the Notebook dropdown select the notebook you just imported.
5. Job done. Publish everything

Skip ahead to Step 6

##### 4.2.2 Create the Notebook 🔺

Nice, this is where the real fun (and value) begins. This is going get it's own section. Move on to Step 5

### **Step 5: Developing the Notebook**

*Todo*
### **Step 6: Running the Pipeline**

We are now going to perform a full pull and we will perform a delta.

#### 6.1 Full Pull

We first need to perform a full. This is to get a full copy of the sites and permissions dataset in the correct location in the storage account

1. Click on 'Add Trigger' > ‘Trigger Now’ to run your pipeline.
2. Go to monitor and view your progress!

> [!Remember] 
> StartDate and EndDate need to be the same to indicate you want a full pull

#### 6.2 Delta Pull (optional)

We now need to perform a delta. This is going to test the code we've written to handle the delta. The outer join to overwrite the existing data with the changes and add anything new

1. Click on 'Add Trigger' > ‘Trigger Now’ to run your pipeline.
2. Go to monitor and view your progress!

Awesome. Data pulled. Now onto visualisation 