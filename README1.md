Prereqs
Step 1: Create Required Azure Resources
Resource Group
Create a new Resource Group: ecommerce-etl-rg

Storage Account
Create a Storage Account: ecommercestorageacc

Enable Hierarchical Namespace = true (for ADLS Gen2)

Go to Containers → Create one called anydata, and one called raw

Create another container transformed (optional but good practice)

Data Ingestion
Step 2: Upload Raw CSV
Sample file: sales_data.csv

order_id,product,amount,date
101,TV,450.00,2024-03-01
102,Laptop,800.00,2024-03-01
103,Mobile,200.00,2024-03-02
104,Tablet,300.00,2024-03-02
Upload it into: anydata/ folder in your Storage Account container

Step 3: Set Up Azure Data Factory (ADF)
create a new data factory with same resource group then go to the data factory URL.
Go to Author → Pipelines

Create a pipeline: IngestSalesCSV

Add Copy Data activity:

Source: CSV in Blob/ADLS (linked to raw/sales_data.csv)

Sink: ADLS folder (e.g., bronze/sales_data.csv or intermediate/)

This copies raw file to your data lake for processing.

Transformation
Step 4: Create Data Flow for Transformation
Go to Author → Data Flows → + New Data Flow

Name it: TransformSalesData

Data Flow Components:
1. Source
Dataset: Delimited Text (CSV)

File path: bronze/sales_data.csv

Enable:

✅ First row as header

✅ Schema import

2. Derived Column (Convert Data Types)
Add a Derived Column transformation to cast types:

Column Name	    Expression
amount_float	toFloat(amount)
date_actual	    toDate(date, 'yyyy-MM-dd')
Resulting schema:

order_id (string)

product (string)

amount_float (float)

date_actual (date)

3. Aggregate
Add Aggregate transformation:

Group by: date_actual

Aggregate column:

Name: total_revenue

Expression: sum(amount_float)

✅ This gives total revenue per day.

4. Sink (Write Transformed Data to ADLS)
Type: Delimited Text (CSV)

Dataset points to: container = transformed

In Sink → Settings: Add file system as raw

✅ Output to single file: revenue_by_date.csv

✅ Enable header row

Folder path: summary/ (or any subfolder)

Step 6: Add to ADF Pipeline
Go back to your main pipeline.

Add a Data Flow Activity

Link it to the above data flow

Trigger → Pipeline runs → Check output in transformed folder

We can see the final file result after pipeline trigger -

date,total_revenue
2024-03-01,1250.0
2024-03-02,500.0
BELOW NOT WORKING
Load Transformed Data into Azure Synapse Analytics
This step enables querying, reporting, and visualization of your cleaned data using SQL tools like Synapse Studio, Power BI, or Excel.

Step 1: Create Azure Synapse Analytics Workspace
Go to Azure Portal → Create Resource → Synapse Analytics

Fill basic details:

Workspace name: ecommerce-synapse

Storage account: Link your existing ecommercestorageacc

File system: Choose raw or create synapse

Create the workspace

Step 2.
Create Dedicated SQL Pool Inside Synapse Workspace:

Go to Manage → SQL Pools → + New

Name it: SalesSQLPool

Performance: Choose small size (DW100c)

Wait for provisioning (1–2 mins)

Step3.
Create Table in Synapse

Open Synapse Studio → Develop → New SQL Script Connect to the SQL pool and run:

CREATE TABLE revenue_by_date (
    revenue_date DATE,
    total_revenue FLOAT
);
✅ You now have an empty table ready to accept data.

Also run the create user query to add user and password to authenticate with SQL DB in Synapse. [ Select database as master]

CREATE LOGIN rajani WITH PASSWORD = 'Bank@103';
alt text

After this run this query to create your user by selecting the user database different from master

CREATE USER rajani FOR LOGIN rajani;

EXEC sp_addrolemember 'db_datareader', 'rajani';
EXEC sp_addrolemember 'db_datawriter', 'rajani';
EXEC sp_addrolemember 'db_owner', 'rajani';
Step4
Create Linked Service in ADF for Synapse Go to ADF → Manage → Linked Services → + New

Type: Azure Synapse Analytics

Set:

Name: AzureSynapseLS

Connect to your Synapse Workspace

Auth: Use SQL auth (username + password) or MSI

Test connection ✅

alt text

Step5
Add Synapse Sink to ADF Data Flow Go back to your existing Data Flow

After the Aggregate step, click + → Sink

Choose Dataset → Azure Synapse Analytics

If needed, create a new Synapse Dataset:

Type: Azure Synapse Analytics

Linked service: AzureSynapseLS

Table: revenue_by_date

In Sink settings:

✅ Allow insert

✅ Auto map columns

error:

Job failed due to reason: com.microsoft.dataflow.broker.InvalidOperationException: Only one valid authentication should be used for AzureSynapseAnalytics2. SQLAuthentication is invalid. One of user/password is missing.
