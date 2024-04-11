Data Warehousing & ETL Group Project
S23 Cohort
Developing an ETL project with SSIS
Performed by:
Christy Lazar, Leonardo Bertocco, Neelam Patkar, Tetiana Shchudla
Instructed by:
Professor Nelson Lopez
DataScienceTechInstitute, Paris Campus 05.10.2023
2
TABLE OF CONTENTS
I. INTRODUCTION ..................................................................................................................................... 3
II. ETL PROCESSING - PIPELINE DESIGN .................................................................................................... 4
STA – STAGING DATABASE ................................................................................................................... 5
ODS – OPERATIONAL DATA STORE ...................................................................................................... 9
DWH – DATA WAREHOUSE ................................................................................................................ 11
ADM - ADMIN DATABASE MANAGEMENT ......................................................................................... 12
III. CONCLUSION ..................................................................................................................................... 14
3
I. INTRODUCTION
Data must first be integrated into an IT system before it can be used and its value observed. This implies that all data from multiple sources must be standardized and unified in order for other programs to utilize them. Implementing a data warehouse is one approach to accomplish this.
In this project, we are going to design and implement a Data Warehouse with calls data analysis of a call center. For this, we will use SQL Server and SSIS.
DATA - Description for Call Center Analysis Project
In our project we proceed with an analysis of call center operations, employee performance, call types, and call charges. To facilitate this analysis, we have organized a structured dataset that encompasses CSV files and lookup data, each with its specific role and relevance in uncovering insights and optimizing call center activities.
CSV Files (Data YYYY):
These CSV files contain comprehensive call data for each respective year. The columns within these files provide critical information that forms the basis for our analysis. Column Name Description
CallTimestamp
Date & Time of call
Call Type
Type of call
EmployeeID
Employee unique ID
CallDuration
Duration of call, in seconds
WaitTime
Wait time, in seconds
CallAbandoned
Was the call abandoned by the customer ? (1 = Yes, 0 = No)
Lookup Data:
To enrich our analysis, we've included lookup data that pertains to employees and call types. These datasets are useful to understand the workforce and categorizing call scenarios.
Employees Column Name Description
EmployeeID
Employee unique ID
EmployeeName
Employee full name
4
Site
Site name where the employee is working at
ManagerName
Employee's Manager
Call Types Column Name Description
CallTypeID
Unique ID for Call Type
CallTypeLabel
Call type label
US States:
We've included a dataset containing information about U.S. states. Column Name Description
StateCD
2-letter state code
Name
Name of the state
Region
US region name (East, West, etc.)
Call Charges:
This dataset captures call charges, specifying the amount of money charged to customers per minute for each call type and year. Column Name Description
Call Type Key
Unique ID for Call Type
Call Type
Call type label
Call Charges / Min (YYYY)
The amount of money that is charged to a customer for each minute spent on the phone, for a specific year (YYYY)
II. ETL PROCESSING - PIPELINE DESIGN
Three phases will be taken to complete the pipeline:
- Loading data as-is or with minor adjustments is possible in the staging area (STA).
- The Operational Data Store area (ODS) will enable data standardization and cleaning.
The "Technical_Rejects" database will be used to store the data as technical rejections if they fail to meet the quality criteria.
Data will be arranged in one fact table linked to numerous dimensions tables in the Data WareHouse area (DWH). Records that cannot be integrated into the schema are functionally rejected and placed in the "Functional_Rejects" table. Alternately, some fictitious relations could be made.
Each file will have its own STA and ODS packages.
5
STA – STAGING DATABASE
The Staging Area is responsible for extracting raw data from the source and storing all the data. We want to accept all available data.
In all SSIS STA packages the important thing is to truncate the table when executing the package to avoid duplicates. This can be done in the Control Flow Level creating a connection between Data Flow Task with an Execute SQL Task (used with corresponding database and TRUNCATE TABLE query). In additon to this there is a Data Flow Level, in which we can specify a Flat File Source (with a selected file), an OLE DB Destination (with the same STA database and a query to create a table) etc.
Five packages are created to load the data:
1.
STA CallCharges. Loaded all call charges from excel file using the Flat file source component without avoiding loading null values using a conditional split component (we will do this step in ODS).
Control Flow for STA CallCharges:
Data Flow for STA CallCharges:
2.
STA CallData. We used a foreach loop container to iterate through the Calls Data folder provided with the csv files from different years but same headers. This video was followed to reach the results presented in the screenshots below.
6
Control Flow for STA CallData:
Data Flow for STA CallData:
3.
STA CallTypes. Simply loading all call types from the excel file using the Flat file source component.
Control Flow for STA CallType:
7
Data Flow for STA CallType:
4.
STA Employees. Simply loading all employees from the excel file using flat file source component.
Control Flow for STA Employees:
Data Flow for STA Employees:
8
5.
STA USStates. Simply loading all US states from the excel file using flat file source component.
Control Flow for STA USStates:
Data Flow for STA USStates:
9
ODS – OPERATIONAL DATA STORE
The next phase in the pipeline involves importing practical (usable) data into the Operational Data Store (ODS). This necessitates converting the data into an accessible structure, along with the imperative tasks of refining and keeping consistency in the data by cleaning and standardize them. Any data that fails to meet the "quality criteria" will be excluded as a technical rejection.
The output data must be consistent in data types and in values. Furthermore, we must guarantee that the data can be effectively employed in queries, which could require restructuring and enhancing the dataset.
1.
ODS_Calls
In this package we perform many transformations on the STA_CallData table.
STA_CallData
ODS_Calls
Using Derived Column functions, we were able to:
-
Extract from CallTimestamp: CallDate, CallTime, CallYear.
-
Extract the column SLAStatus from WaitTime.
-
Create a ChangeID identifier that we used in the following DWH step as a key.
Using Data Conversions functions, we were able to adjust all the data types of all columns as needed. We proceeded also by creating a TechnicalRejects Table in which register possible errors in the data conversions processes.
2.
ODS_Employees
STA_Employees
STA_USStates
ODS_Employees
10
Using Derived Column functions we were able to:
-
Split EmployeeName from STA_Employees into EmployeeFirstName and EmployeeLastName.
-
Split ManagerName from STA_Employees into ManagerFirstName and ManagerLastName.
-
Split EmployeeName from STA_Employees into StateCode and City.
Using Lookup Function, we were able to join USStates table in order to get also the Region column.
Using Data Conversions functions, we were able to adjust all the data types of all columns as needed. We proceeded also by creating a TechnicalRejects Table in which register possible errors in the data conversions processes.
3.
ODS_CallCharges
STA_CallCharges
STA_CallTypes
ODS_CallCharges
With Conditional Split we cut off the NULL values in STA_CallCharges.
With the Lookup function we merged the STA_CallTypes to get the CallTypeID column.
11
With Unpivot transformation we were able to get the Call Charges of different years in a single column as needed.
Using Data Conversions functions, we were able to adjust all the data types of all columns as needed and to clean the data (using TRIM function and deleting useless information such as “/ min” in CallCharges column).
We created a ChangeID identifier that we used in the following DWH step as a key.
We proceeded also by creating a TechnicalRejects Table in which we register possible errors in the data conversions processes.
DWH – DATA WAREHOUSE
At the DWH stage, three dimension tables were created, including creating technical keys in each table. DimCallCharge with CallChargeKey, Dim Employees with EmployeeKey and DimDate using SQL query with DateKey. In addition, a fact table FactCalls was created in the DWH FactCalls package. Within this package possible errors which could happen in the future were redirected and gathered in a new FunctionalRejects table in the ADM database. Examples of those tables are presented below.
DWH_DimCallCharges
12
DWH_DimDate
DWH_DimEmployees
DWH_DimEmployees
ADM - ADMIN DATABASE MANAGEMENT
As mentioned before, during the ODS and DWH stages we created two tables called TechnicalRejects and FunctionalRejects. The TechnicalRejects table was created to record any errors arising from the
13
transformations applied in the ODS stage. Whereas the FunctionalRejects table was created to record any errors from the FactCalls DWH stage.
Examples are presented below:
ADM_Functional_Rejects
ADM_TechnicalRejects
How to recreate every table?
You can find the SQL script file (.sql) in the project folder, which has to be executed in Microsoft SQL Server Management in order to recreate the tables.
The tip to run the packages
You can use Sequence Container components for running all the stages separately or one by one as shown below:
14
III. CONCLUSION
To summarize, we collected all tables in staging from Excel database. Then In ODS We carried various transformations for structuring , cleaning and filtering. We tried to move absurd data in ADM for Admins verification.
Finally we moved desired data in required pattern in DWH. We have included Sequence container to run packages in correct sequence and segregating in allocated groups.
