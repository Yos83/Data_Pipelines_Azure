# Data Pipelines Azure
Create automated pipelines in Azure to ingest, transform and analyse payroll data
Project purpose
This project helps New York City better understand how public money is spent—especially on employee salaries and overtime. It's part of Udacity’s “Data Engineering with Microsoft Azure” program, focuses on building real-world skills in working with large-scale data systems. The solution uses tools like Azure Synapse Analytics, Data Factory, and Data Lake Gen2 to create automated data pipelines. These pipelines clean and organize raw payroll data, turning it into useful summaries that can guide budget decisions and make the city’s finances more transparent for everyone.

NYC Payroll Data Base Schema
<img width="940" height="823" alt="image" src="https://github.com/user-attachments/assets/42f95793-9ed2-40e5-bba5-30fd505c41a8" />

 
Pipeline overview
<img width="940" height="398" alt="image" src="https://github.com/user-attachments/assets/2d961f0d-54e1-4449-ac03-1f904526688d" />

 # Project Environment

For this project, you'll do your work in the Azure Portal, using several Azure resources including:
- Azure Data Lake Gen2 
- Azure SQL DB
- Azure Data Factory
- Azure Synapse Analytics

# Step 1: Prepare the Data Infrastructure
Setup Data and Resources in Azure

 1.Create the data lake and upload data

Log into your Azure account and Create an Azure Data Lake Storage Gen2 (storage account) and associated storage container resource named adlsnycpayroll-yourfirstname-lastintial.
In the Azure Data Lake Gen2 creation flow, go to Advanced tab and ensure below options are checked:

- Require secure transfer for REST API operations
- Allow enabling anonymous access on individual containers
- Enable storage account key access
- Default to Microsoft Entra authorization in the Azure portal
- Enable hierarchical namespace

Create three directories in this storage container named:
- dirpayrollfiles
- dirhistoryfiles
- dirstaging

**dirstaging** will be used by the pipelines we will create as part of the project to store staging data for integration with Azure Synapse. This will be discussed in further pages

Upload these files from the project data(opens in a new tab) to the dirpayrollfiles folder:

- EmpMaster.csv
- AgencyMaster.csv
- TitleMaster.csv
- nycpayroll_2021.csv

Upload this file (historical data) from the project data(opens in a new tab) to the dirhistoryfiles folder
- nycpayroll_2020.csv

2. Create an Azure Data Factory Resource

3. Create a SQL Database

In the Azure portal, create a SQL Database resource named db_nycpayroll.

In the creation steps, you will be required to create a SQL server, create a server with Service tier: Basic (For less demanding workloads).

![image](https://github.com/eloisjr/Data_Pipeline_Azure/assets/81710422/638b303e-ea98-4640-868d-a60d044a1358)

In Networking tab, allow both of the below options:

- Allow Azure services and resources to access this server
- Add current client IP address

  4. Create a Synapse Analytics workspace
  5. Create summary data external table in Synapse Analytics workspace

Define the file format, if not already, for reading/saving the data from/to a comma delimited file in blob storage.

![pipeline](https://github.com/eloisjr/Data_Pipeline_Azure/assets/81710422/dfc2e06c-c8f9-4b71-816c-379f354768b9)


Define the data source to persist the results. Use the blob storage account name as applicable to you.

Create external table that references the dirstaging directory of DataLake Gen2 storage for staging payroll summary data. (Pipeline for this will be created in later section).

![pipeline2](https://github.com/eloisjr/Data_Pipeline_Azure/assets/81710422/898442a7-b7ef-4356-a925-eaa3aca1663f)

In the code snippet above, the data will be stored in the ‘/’ directory in the blob storage in dirstaging (this was configured when creating datasource). You can change the location as you desire. Also, change the DATA_SOURCE value, as applicable to you. Note that, mydls20230413 is the Data Lake Gen 2 storage name, and mydlsfs20230413 is the name of the file system (container).

6. Create master data tables and payroll transaction tables in SQL DB
   
![image](https://github.com/eloisjr/Data_Pipeline_Azure/assets/81710422/175d8636-7d24-4636-8643-5b896b5bc418)

![image](https://github.com/eloisjr/Data_Pipeline_Azure/assets/81710422/48d1f2f8-e9bd-4369-8d38-d4511efde1ed)

![image](https://github.com/eloisjr/Data_Pipeline_Azure/assets/81710422/a19385f8-c8bb-4605-b846-601a8f497f47)

![image](https://github.com/eloisjr/Data_Pipeline_Azure/assets/81710422/639ca68f-d986-470c-8bb4-f4187955d5f5)

![image](https://github.com/eloisjr/Data_Pipeline_Azure/assets/81710422/cdcd3daf-378d-491a-9b77-cdb6600d8030)

# Step 2: Create Linked Services

1.Create a Linked Service for Azure Data Lake

In Azure Data Factory, create a linked service to the data lake that contains the data files

- From the data stores, select Azure Data Lake Gen 2
- Test the connection
- 
2.Create a Linked Service to SQL Database that has the current (2021) data

- If you get a connection error, remember to add the IP address to the firewall settings in SQL DB in the Azure Portal

# Step 3: Create Datasets in Azure Data Factory

1.Create the datasets for the 2021 Payroll file on Azure Data Lake Gen2

- Select DelimitedText
- Set the path to the nycpayroll_2021.csv in the Data Lake
- Preview the data to make sure it is correctly parsed

2. Repeat the same process to create datasets for the rest of the data files in the Data Lake

- EmpMaster.csv
- TitleMaster.csv
- AgencyMaster.csv
- Remember to publish all the datasets

3. Create the dataset for all the data tables in SQL DB

4. Create the datasets for destination (target) table in Synapse Analytics

- dataset for NYC_Payroll_Summary

# Step 4: Create Data Flows

In Azure Data Factory, create data flow to load 2020 Payroll data from Azure DataLake Gen2 storage to SQL db table created earlier

Create a new data flow:

1. Select the dataset for 2020 payroll file as the source
   
2. Click on the + icon at the bottom right of the source, from the options choose sink. A sink will get added in the dataflow
   
3. Select the sink dataset as 2020 payroll table created in SQL db.
   
Repeat the same process to add data flow to load data for each file in Azure DataLake to the corresponding SQL DB tables.

# Step 5: Data Aggregation and Parameterization

In this step, you'll extract the 2021 year data and historical data, merge, aggregate and store it in DataLake staging area which will be used by Synapse Analytics external table. The aggregation will be on Agency Name, Fiscal Year and TotalPaid.

1. Create new data flow and name it Dataflow Summary
 
2. Add source as payroll 2020 data from SQL DB
 
3. Add another source as payroll 2021 data from SQL DB
 
4. Create a new Union activity and select both payroll datasets as the source

5. Make sure to do any source to target mappings if required. This can be done by adding a Select activity before Union

6. After Union, add a Filter activity, go to Expression builder
- Create a parameter named- dataflow_param_fiscalyear and give value 2020 or 2021
- Include expression to be used for filtering: toInteger(FiscalYear) >= $dataflow_param_fiscalyear
 
7. Now, choose Derived Column after filter
- Name the column: TotalPaid
- Add the following expression: RegularGrossPaid + TotalOTPaid+TotalOtherPay
 
8 . Add an Aggregate activity to the data flow next to the TotalPaid activity
- Under Group by, select AgencyName and FiscalYear
- Set the expression to sum(TotalPaid)
  
9. Add a Sink activity after the Aggregate
- Select the sink as summary table created in SQL db
- In Settings, tick Truncate table
 
10. Add another Sink activity, this will create two sinks after the Aggregate
- Select the sink as dirstaging in Azure DataLake Gen2 storage
- In Settings, tick Clear the folder

# Step 6: Pipeline Creation

We will create a pipeline to load data from Azure DataLake Gen2 storage in SQL db for individual datasets, perform aggregations and store the summary results back into SQL db destination table and datalake staging storage directory which will be consumed by Synapse Analytics via CETAS.

1. Create a new pipeline
2. Include dataflows for Agency, Employee and Title to be parallel
3. Add dataflows for payroll 2020 and payroll 2021. These should run only after the initial 3 dataflows have completed
4. After payroll 2020 and payroll 2021 dataflows have completed, dataflow for aggregation should be started.
5. Refer to the below screenshot. Your final pipeline should look like this


![image](https://github.com/eloisjr/Data_Pipeline_Azure/assets/81710422/527c76b2-404a-4312-9378-65e45091736f)

A sample Pipeline flow

# Step 7: Trigger and Monitor Pipeline

1. Select Add trigger option from pipeline view in the toolbar
2. Choose trigger now to initiate pipeline run
3. You can go to monitor tab and check the Pipeline Runs
4. Each dataflow will have an entry in Activity runs list

# Step 8: Verify Pipeline run artifacts
1. Query data in SQL DB summary table (destination table). This is one of the sinks defined in the pipeline.
2. Check the dirstaging directory in Datalake if files got created. This is one of the sinks defined in the pipeline
3. Query data in Synapse external table that points to the dirstaging directory in Datalake.




