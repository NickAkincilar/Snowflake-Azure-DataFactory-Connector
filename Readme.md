# ( *** Retired ***) Snowflake Azure Data Factory ADF Connector (with Dynamic Credentials & SQL) #


<br><br>
## **This version is no longer being updated.**

## **Please use the new & improved version 3 instead**


## ***Use this link instead :*** <a href="https://github.com/NickAkincilar/Azure-DataFactory-ADF-Snowflake-Connector">Snowflake ADF connector Version 3</a>

<br><br><br>
<hr>
<br><br><br>

***01/21/2020 - Added multi SQL statement support within a single call. Simply add additional SQL queries in the "query": parameter sepearated by ; character***

***12/01/2019 - Updated the code run the Snowflake Query Asynch in order to avoid ADF Http request time-outs when queries run longer than 230 secs. Downside is function will no longer wait for query to finish hence will always return a positive result.***

This is a solution to trigger SQL commands in Snowflake from Azure Data Factory using a custom Azure Fucntion. Azure function uses Snowflake .Net connector to make a connection to Snowflake and trigger SQL commands.

Typical usage would be to place this at the end of a data pipeline and issue a copy command from Snowflake once Data Factory generates data files in an Azure blob storage.

<img src="https://raw.githubusercontent.com/NickAkincilar/Snowflake-Azure-DataFactory-Connector/master/images/Snowflake%20Azure%20Data%20Factory%20diagram.png" alt="drawing" width="900"/>

Function is essentially a rest endpoint which accepts a POST request which needs to contain the following JSON payload in the body of the request. JSON payload is a combination of SF connection parameters plus the SQL command that needs to be executed.

Execution Logic is follows:
1. Create an ADF data pipe line to create necessary data files on a Azure blob stage
2. ADF creates data file(s) in the Azure blob storage
3. Upon successfull execution of step 2, ADF makes a call to Snowflake_ADF Azure function with a JSON data  (Configure JSON body with connection + proper copy command to execute )
4. Snowflake_ADF Function makes a .Net connection to Snowflake using the connection parameteres from JSON payload and executes the embedded SQL command.
5. Snowflake executes the COPY from @STAGE command to ingest the newly created files by ADF.

<hr>

Connection attributes & the SQL command can be inserted in to JSON two different ways.

- **Using dynamic ADF variables.** Credentials & other attributes can be dynamically fetched from **Azure Key Vault** Secret keys where they are securely stored.  (_Preferred_)

      		    {
      		    "account":"your_SF_account",
      		    "host":"",
      		    "userid":"@{variables('v_user')}",
      		    "pw":"@{variables('v_pw')}",
      		    "database":"db_name",
      		    "schema":"PUBLIC",
      		    "role":"Your_ROLE",
      		    "warehouse":"Warehouse_Name",
      		    "query":"Copy into SomeTable from @YourStage;"
      		    }

	<img src="https://raw.githubusercontent.com/NickAkincilar/Snowflake-Azure-DataFactory-Connector/master/images/Credentials_From_KeyVault.png" alt="drawing" width="500"/>

You can use web requests to fetch Azure Vault Secrets & set it as variable values to be passed to this custom function as user credentials.
<hr>

- **As Static Values** Credentials can also be stored as part of the JSON input of the Azure function properties within the ADF Pipeline

				{
				"account":"your_SF_account",
				"host":"",
				"userid":"YourUserName",
				"pw":"YourPassword",
				"database":"db_name",
				"schema":"PUBLIC",
				"role":"Your_ROLE",
				"warehouse":"Warehouse_Name",
				"query":"Copy into SomeTable from @YourStage;"
				}


	<img src="https://raw.githubusercontent.com/NickAkincilar/Snowflake-Azure-DataFactory-Connector/master/images/Credentials_Static.png" alt="drawing" width="500"/>

# Setup (Part 1) - Create Snowflake Azure Function 

  

1. Create a new Azure Function

  

<img  src="https://raw.githubusercontent.com/NickAkincilar/Snowflake-Azure-DataFactory-Connector/master/images/Screenshot00001.png"  alt="drawing"  width="300"/>

  

2. Set the BASICS as following

	1.  **Resource Group** = Create a new one or use an existing one

	2.  **Function App Name** = Provide a unique name for your function

	3.  **Runtime Stack** : .NET Core

  

		<img  src="https://raw.githubusercontent.com/NickAkincilar/Snowflake-Azure-DataFactory-Connector/master/images/Screenshot00002.png"  alt="drawing"  width="500"/>

  

3. Set the HOSTING properties as:

	- Storage Account = Create New or use existing

	- Operating System = Windows

	- Plan Type = You Choice

  

		<img  src="https://raw.githubusercontent.com/NickAkincilar/Snowflake-Azure-DataFactory-Connector/master/images/Screenshot00003.png"  alt="drawing"  width="500"/>

4. Following the following steps &amp; click CREATE to finish the initial phase

5. It will take few minutes to deploy it.


	<img  src="https://raw.githubusercontent.com/NickAkincilar/Snowflake-Azure-DataFactory-Connector/master/images/Screenshot00004.png"  alt="drawing"  width="500"/>

6. Once finished, click on GO TO RESOURCE

  

	<img  src="https://raw.githubusercontent.com/NickAkincilar/Snowflake-Azure-DataFactory-Connector/master/images/Screenshot00005.png"  alt="drawing"  width="500"/>

  

7. Click on NEW FUNCTION


	<img  src="https://raw.githubusercontent.com/NickAkincilar/Snowflake-Azure-DataFactory-Connector/master/images/Screenshot00006.png"  alt="drawing"  width="500"/>

8. Use IN-PORTAL option

  

	<img  src="https://raw.githubusercontent.com/NickAkincilar/Snowflake-Azure-DataFactory-Connector/master/images/Screenshot00007.png"  alt="drawing"  width="500"/>

9. Use WEBHOOK + API

  

	<img  src="https://raw.githubusercontent.com/NickAkincilar/Snowflake-Azure-DataFactory-Connector/master/images/Screenshot00008.png"  alt="drawing"  width="500"/>

  

10. It will automatically create a function called HTTPTrigger1 which you need to delete

	- Click on FUNCTIONS on the left

	- Click on Trash can Icon to delete for HTTPTrigger1

  

		<img  src="https://raw.githubusercontent.com/NickAkincilar/Snowflake-Azure-DataFactory-Connector/master/images/Screenshot00009.png"  alt="drawing"  width="500"/>

11. Click on NEW FUNCTION to add your own

	<img  src="https://raw.githubusercontent.com/NickAkincilar/Snowflake-Azure-DataFactory-Connector/master/images/Screenshot00010.png"  alt="drawing"  width="500"/>

  

12. Choose HTTP Trigger

  

	<img  src="https://raw.githubusercontent.com/NickAkincilar/Snowflake-Azure-DataFactory-Connector/master/images/Screenshot00011.png"  alt="drawing"  width="500"/>

13. Give it a unique name for the call

	1. Authorization level determines who can call this function?

		-  **Anonymous** = REST endpoint is open to public (still need to provide proper credentials for snowflake)

		-  **Function** = This requires sender to provide a unique API security key to make a connection o this REST end point

  

			<img  src="https://raw.githubusercontent.com/NickAkincilar/Snowflake-Azure-DataFactory-Connector/master/images/Screenshot00012.png"  alt="drawing"  width="400"/>

14. Override the code the in RUN.CSX.

15. Copy &amp; Paste code from RUN.CSX file on [from this project](https://github.com/NickAkincilar/Snowflake-Azure-DataFactory-Connector/blob/master/ADF_RestCallSnowflake/run.csx) &amp; SAVE.

16. Copy necessary supporting files &amp; driver

	1. Click on the FUNCTION APP name

	2. Click PLATFORM FEATURES tab

	3. Click Advanced tools(Kudu)

		- Choose - Debug Console

		- Pick - POWERSHELL

  

			<img  src="https://raw.githubusercontent.com/NickAkincilar/Snowflake-Azure-DataFactory-Connector/master/images/Screenshot00013.png"  alt="drawing"  width="500"/>

  

17. Navigate to site following path = \wwwroot\Your\_Function\_Name\

  

	<img  src="https://raw.githubusercontent.com/NickAkincilar/Snowflake-Azure-DataFactory-Connector/master/images/Screenshot00014.png"  alt="drawing"  width="500"/>

18. [Download](https://github.com/NickAkincilar/Snowflake-Azure-DataFactory-Connector/archive/master.zip) the repository from this project &amp; unzip.

19. Drag &amp; drop the BIN folder in to the files area under your function folder

	<img  src="https://raw.githubusercontent.com/NickAkincilar/Snowflake-Azure-DataFactory-Connector/master/images/Screenshot00015.png"  alt="drawing"  width="800"/>

20. If you choose, Function Auth Level as. FUNCTION then you need to get the API call key to call this function

21. Go back to Function area

	1. Click on **Function name**

	2. Click on **Manage**

	3. Click on **COPY** for the default function key &amp; **store it in a text editor.**

  

		<img  src="https://raw.githubusercontent.com/NickAkincilar/Snowflake-Azure-DataFactory-Connector/master/images/Screenshot00016.png"  alt="drawing"  width="500"/>

  


# Setup (Part 2) - USE THE FUNCTION IN AZURE DATA FACTORY PIPELINE
  

1. Add Azure Function on to canvas

2. Give it a name
	- **Check Secure Input & Output** to hide connection info from being logged. <br/><br/>

  


	<img  src="https://raw.githubusercontent.com/NickAkincilar/Snowflake-Azure-DataFactory-Connector/master/images/Screenshot00017.png"  alt="drawing"  width="400"/>

  

3. Switch to SETTINGS

	-  **+ NEW** to add a new AZURE FUNCTION LINKED SERVICE

		- Set All Required Properties

		- Function Key = **Key value from step 21** (_if FUNCTION mode is used for security_)

  

			<img  src="https://raw.githubusercontent.com/NickAkincilar/Snowflake-Azure-DataFactory-Connector/master/images/Screenshot00018.png"  alt="drawing"  width="500"/>

		- Function Name = **Function name from Step #13**

		- Method = **POST**

  

			<img  src="https://raw.githubusercontent.com/NickAkincilar/Snowflake-Azure-DataFactory-Connector/master/images/Screenshot00019.png"  alt="drawing"  width="500"/>

		- Set BODY to following JSON format with Snowflake connection & SQL command attributes


			    {  
			    "account":"your_SF_account", 
			    "host":"",
			    "userid":"YourUserName",
			    "pw":"YourPassword",
			    "database":"db_name",
			    "schema":"PUBLIC",
			    "role":"Your_ROLE",
			    "warehouse":"Warehouse_Name",
			    "query":"Copy into SomeTable from @YourStage;" 
			    }

  

4. **DEBUG** on the PIPELINE to test.

  

If the data pipeline was able to successfully create output files, it will trigger the Azure Function and that would connect to Snowflake and Execute the SQL command using the attributes of the JSON post data.
<img  src="https://raw.githubusercontent.com/NickAkincilar/Snowflake-Azure-DataFactory-Connector/master/images/Screenshot00020.png"  alt="drawing"  width="800"/>
