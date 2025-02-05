# SQLDW Query Performance Tuning

### Table of Contents
Summary<br/>
Pre-requisites: What you need to get started<br/>
Part 1: Queries running slowly<br/>
Part 2: Joins<br/>
Part 3: Troubleshooting<br/>
Part 4: Query performance improvements<br/>
Part 5: Query analysis

## Summary
Over the course of this lab you will look for inefficiencies that are causing sub-optimal performance.<br/>
The tables in this lab use the TPCH database.<br/>
In each of the examples, there will be two versions of same query. One that is running a poor performing scenario (denoted as slow) and one that is running a more optimal scenario (denoted fast). Your goal should be to study the slow scenario using the DMV queries and SSMS object explorer to decide what is wrong with the slow scenario. Once you feel you understand the issue in the ‘slow’ scenario, connect to the ‘fast’ version to verify your findings. 

### SQL DW Resources
These articles to help you solve the scenarios presented in this lab.<br/>
•	Query investigation: https://azure.microsoft.com/en-us/documentation/articles/sql-data-warehouse-managemonitor/<br/>
•	Best Practices: https://azure.microsoft.com/en-us/documentation/articles/sql-data-warehouse-best-practices/

## Pre-requisites: What you need to get started

This lab requires you to load a new data source to the Azure Data Warehouse server created in the previous labs. Please follow below steps to load the sample dataset to your server.
The login that you use for running this script should have “Create Login” permission on your server!
This script will create multiple versions of customer, orders, lineitem, part, partsupp, supplier, nation and region tables. These tables will be used during your lab.
You will also edit the PowerShell script and add your server and database names. This will be used during exercises.

1.	Open a PowerShell window in your virtual machine.<br/>
2.	Change directory to Query Performance Tuning lab content folder with given command:<br/>
``
cd C:\LabContent\Analytics-Airlift-master\Day 1\07.SQLDW - Query tuning lab\Prep
``
3.	Change directory to Prep sub folder.<br/>
4.	Run PrepLab.ps1 script with you Azure Data Warehouse details.<br/>
    <img src="images/sql1.jpg"/><br/>
5.  Run the following command for changing the PowerShell execution policies for Windows computers.<br/>
``
Set-ExecutionPolicy -Scope Process -ExecutionPolicy ByPass
``
    <img src="images/sql2.jpg"/><br/>

6.	Open your **Query Performance Tuning Lab** content folder.<br/>
7.	Change directory to Lab sub folder.<br/>
8.	Edit “RunExercise.ps1” script.<br/>
9.	Replace **<your_server>** with your **server** name. (Without database.windows.net)<br/>
    <img src="images/sql01.jpg"/><br/>
10.	Replace **<your_database>** with your **database** name.<br/>
    <img src="images/sql02.jpg"/><br/>
    
## Part 1: Queries running slowly
Your user comes to you and says “My query is running slow. I’m not sure why, because I’m not selecting very many rows. Can you help me figure out why?

1.	Open a PowerShell window.<br/>
2.	Change directory to **Query Performance Tuning lab** content folder.<br/>
3.	Change directory to Lab sub folder.<br/>
4.	Run **RunExercise.ps1** script with following parameters. This will execute a query on your server and show the result<br/>

``
.\RunExercise.ps1 -Name Exercise1 -Type Slow
``
    <img src="images/sql4.jpg"/><br/>
    
5.	Open Query editor of SQL Data Warehouse in Azure Portal.<br/>
    <img src="images/sql3.jpg"/><br/>
    
6.	Check the query execution details with using DMVs.<br/>
```
SELECT * FROM sys.dm_pdw_exec_requests
WHERE [Label] like 'Exercise1 | Slow%' 
ORDER BY submit_time DESC
```
7.	You can use the labels to search for your specific query. Powershell window shows the “Label” that was used during query execution.<br/>
    <img src="images/sql5.jpg"/><br/>
8.	Look for most recent execution of Exercise 1 query (“Running” or “Completed”)<br/>
9.	Once you’ve identified the problematic query ID for this scenario, take a deeper look into it by using dm_pdw_request_steps:<br/>

```
SELECT * FROM sys.dm_pdw_request_steps
WHERE request_id = 'QID####'
ORDER BY step_index
```

10.	After running these queries, come up with a hypothesis about why the operation may be taking a long time.  What are the longest running operations?  What might they tell you about how the workflow is structured?<br/>
    <img src="images/sql6.jpg"/><br/>
```
Hint: Moving data will typically be one of the most expensive operations within SQL DW.  How is this query moving data and how could it move it more effectively?  See the Appendix for more info about data movement types.<br/>
Hint: Why is this query moving data?  What is the source of the moved data? 
Hint: If you’re still stuck, look at your tables in the object explorer (or sys.tables) – what’s different about the table the user is querying?  Can you find it where you expect in the object explorer tree?  Why not?  What type of table is this table?<br/>
```
11.	After you have a firm understanding of this slow scenario, run the same query with Fast optionThis will execute a query on your server and show the result.<br/>
``
.\RunExercise.ps1 -Name Exercise1 -Type Fast
``
    <img src="images/sql7.jpg"/><br/>
12.	Run the following querys and check the results<br/>
```
SELECT * FROM sys.dm_pdw_exec_requests
WHERE [Label] like 'Exercise1 | Fast%' 
ORDER BY submit_time DESC

SELECT * FROM sys.dm_pdw_request_steps
WHERE request_id = 'QID####'
ORDER BY step_index
```
13.	Compare the results.<br/>

    <img src="images/sql8.jpg"/><br/>
    <img src="images/sql9.jpg"/><br/>

### Discussion
After steps 1-5, you should have taken a look at the query that was being executed and think about what kind of plan you may expect to see.<br/><br/>
At step 10, you should have been able to find your query in exec_requests using the label and determine: Request_ID, total elapsed time, resource class, and first 4,000 characters of the command text. Looking at the command you can see that you are running a select count_big(*) against the 'dbo.lineitem_0' table. You may want to look in object explorer or sys.tables at this point to see a few general things: what type of table is it? If it's distributed, what is the distribution key/column? Are there multi column statistics? Have they been updated? If it's a view, what are the underlying tables? These are all general things you will want to know for most query performance issues.<br/><br/>
At step 11, you want to pick out the long-running step based on total_elapsed_time, which in this case was a HadoopRoundRobinOperation. If you look up this movement, it is querying data in an external table and storing the results in PDW in a round robin table. Also notice that the row count is large at about 60 million rows. This is because when copying data from an external table we copy a full version of the table into SQLDW - the predicate is not used to trim this data down yet.<br/><br/>
At step 14, you should be able to use exec_requests in the same way you did before to get the query text and see that we are now querying table 'dbo.lineitem_3'. You should look at sys.tables or object explorer in SSMS/SSDT to see the differences between this table and the previous one. You should notice that lineitem_3 is a distributed table whereas Lineitem_2 was an external table. The new plan in request_steps no longer has the Hadoop shuffle because the data is already in SQLDW.<br/><br/>
This should illustrate that when you are going to be using external tables repeatedly, it is much more efficient if you first import the table(s) into SQLDW then run the queries on the local tables. Otherwise, every query that touches the external table will have to import the table into tempdb as part of the plan before being able to execute the query.

## Part 2: Joins
Now that you’ve got the hang of things, let’s try the same process on the next exercise. 
Again, your user comes to you with questions, complaining that they are joining two of their most important tables together, and SQL DW just isn’t performing as well as they had expected. 

1.	Open a PowerShell window.<br/>
2.	Change directory to Query Performance Tuning lab content folder.<br/>
3.	Change directory to Lab sub folder.<br/>
4.	Run “RunExercise.ps1” script with following parameters<br/>
``
\RunExercise.ps1 -Name Exercise2 -Type Slow
``
    <img src="images/sql10.jpg"/><br/>
5.	Open Query editor of SQL Data Warehouse in Azure Portal.<br/>
6.	Check the query execution details with using DMVs.<br/>
    <img src="images/sql03.jpg"/><br/>
7.	You can use the labels to search for your specific query. Powershell window shows the “Label” that was used during query execution.<br/>
8.	Look for most recent execution of Exercise 2 query (“Running” or “Completed”)<br/>
    <img src="images/sql11.jpg"/><br/>
9.	Once you’ve identified the problematic query ID for this scenario, take a deeper look into it by using dm_pdw_request_steps:<br/><br/>
Some steps of the DSQL plan are mostly overhead and can generally be ignored for purposes of optimizing the plan.<br/>
These steps include the RandomIDOperation and the creation of the temporary tables for DMS.<br/
It can often help to add additional predicates to the above query to remove some of the overhead steps thus allowing you to focus on the heavy lifting operations. AND operation_type NOT IN ('RandomIDOperation')  AND command NOT LIKE 'CREATE %'  AND command NOT LIKE 'DROP %'<br/><br/>
    <img src="images/sql04.jpg"/><br/>    
10.	Check the steps and determine which one(s) might be the problematic steps.<br/>
    <img src="images/sql12.jpg"/><br/>
11.	Run the same query with Fast option.<br/>

``
.\RunExercise.ps1 -Name Exercise2 -Type Fast”
``
    <img src="images/sql13.jpg"/><br/>
12.	Compare the 2 query execution plans and determine what would be the reason for query slowness.<br/>
    <img src="images/sql14.jpg"/><br/>
    <img src="images/sql15.jpg"/><br/>
``
Hint: Look at the tables that are being joined with the query.  Take a look at the table distribution types in the SSMS object explorer.  The icon for each table tells you if the table is hash distributed or round robin distributed?  What occurs when two round robin tables are joined?
``

### Discussion:
You should have followed the same workflow as you did in exercise 1 up until step 9. In Step 9 we introduce some helpful additions to the query to filter out some of the SQL OnOperations that are not helpful for troubleshooting. This makes request_steps easier to read.<br/><br/>
At step 10, you can see that we are performing 5 broadcast moves and 1 shuffle move. Most of the time was spent in the shuffle move, but the large rowcount on the first broadcast is a point of interest, because remember we generally do not want to be broadcasting large tables.<br/><br/>
At step 12, you are comparing the fast plan to the slow plan. You can see in the fast plan that we now have 4 broadcast moves (instead of 5) and 1 shuffle. The table that is no longer being broadcasted is that large table we noticed in step 10. We can get the tables being queries from exec_requests, then look at sys.tables or object explorer to see what kind of tables they are. You will see that the fast version has all hash distributed tables, while the slow version has round robin tables.<br/><br/>
In general, you want large fact tables to be distributed tables. In this query both the orders and lineitem tables are large fact tables. If we are joining them together then it is best if they are distribution-compatible, which means distributed on the same key. This way each distribution has just a small slice of data to work with. In the fast version, both of these tables are distributed on orderkey. Round robin tables are never distribution-compatible, so the slow plan has to perform some sort of movement, like a broadcast, to make them distribution compatible before performing the join. The fast version shuffle will be faster because of the smaller input data volume.

## Part 3: Troubleshooting Nuke)
Again, your user comes to you with questions, saying “I’ve just loaded my data and my queries are running slow than on SQL Server! What am I missing here?”

1.	Open a PowerShell window.<br/>
2.	Change directory to Query Performance Tuning lab content folder.<br/>
3.	Change directory to Lab sub folder.<br/>
4.Run “RunExercise.ps1” script with following parameters<br/>
``
.\RunExercise.ps1 -Name Exercise3 -Type Slow
``
    <img src="images/sql16.jpg"/><br/>
5.	Open Query editor of SQL Data Warehouse in Azure Portal.<br/>
6.	Check the query execution details with using DMVs.<br/>
    <img src="images/sql05.jpg"/><br/>
7.	You can use the labels to search for your specific query. Powershell window shows the “Label” that was used during query execution.<br/> 
8.	Look for most recent execution of Exercise 3 query (“Running” or “Completed”)<br/>
    <img src="images/sql17.jpg"/><br/>
9.	Once you’ve identified the problematic query ID for this scenario, take a deeper look into it by using dm_pdw_request_steps:<br/>
    <img src="images/sql06.jpg"/><br/>
10.	Check the steps and determine which one(s) might be the problematic steps.<br/>
    <img src="images/sql18.jpg"/><br/>
11.	Run the same query with Fast option.<br/>
``
.\RunExercise.ps1 -Name Exercise3 -Type Fast
``
    <img src="images/sql19.jpg"/><br/>
12.	Compare the 2 query execution plans and determine what would be the reason for query slowness.<br/>
    <img src="images/sql21.jpg"/><br/>
    <img src="images/sql22.jpg"/><br/>
```
Hint: Look at our best practices (in order) to narrow down what issues cause queries to run slowly.<br/>
Hint: The “orders” table is one of the two largest tables and generally too big for a broadcast move.  Why did the optimizer choose to create a copy of these rows on all nodes?<br/>
Hint: If you’re still having trouble, look at the orders table in the object explorer tree to see what statistics are on the tables in this query.  You can see the SQL that is running in the DMV sys.dm_pdw_exec_requests.<br/>
```
### Discussion
You should be able to reach step 10 using the same method you did in the first 2 exercises. In step 10, you should look through the steps for the longest-running step. In this case it's the BroadcastMove that took over 30 seconds. Next you can see that the rowcount for this broadcast is 60 million rows. This is a red flag because broadcasting such a large table will be very expensive. You should have used the original query or the exec_requests DMV to get the command and see what tables you are querying.<br/><br/>
In step 12, you can see that this large broadcast is no longer there. If you compare the table types in the slow version to the tables in the fast version you will see that they are the same. However in the fast version you can see in object explorer that there are statistics on the distribution and join columns.<br/><br/>
Further, if you run the query provided to get details about the table, you will see that for lineitem_1, the CTL_row_count is 1,000, but the cmp_row_count is ~60 million. 1,000 is the default value for statistics on the control node, so this means that statistics were never manually created. The distributed plan was created assuming it could broadcast this table because there were only 1,000 rows, but in reality there were 60 million rows, which caused our long-running step.<br/><br/>
This illustrates how the absence of statistics or statistics not being up to date can affect the distributed plan.<br/>

## Part 4: Query performance improvements
Now that your user has got all of their data loaded and organized they are trying out some of their more complex queries.  Check this exercise to see if there are any improvements they can make to decrease their query runtime.

1.	Open a PowerShell window.<br/>
2.	Change directory to Query Performance Tuning lab content folder.<br/>
3.	Change directory to Lab sub folder.<br/>
4.	Run “RunExercise.ps1” script with following parameters<br/>
``
.\RunExercise.ps1 -Name Exercise4 -Type Slow
``
    <img src="images/sql23.jpg"/><br/>
5.	Open Query editor of SQL Data Warehouse in Azure Portal.<br/>
6.	Check the query execution details with using DMVs.<br/>
    <img src="images/sql34.jpg"/><br/>
7.	You can use the labels to search for your specific query. Powershell window shows the “Label” that was used during query execution<br/>
8.	Look for most recent execution of Exercise 4 query (“Running” or “Completed”)<br/>
    <img src="images/sql24.jpg"/><br/>
9.	Check the steps and determine which one(s) might be the problematic steps.<br/>
    <img src="images/sql35.jpg"/><br/>
10.	Run the same query with Fast option.<br/>
``
.\RunExercise.ps1 -Name Exercise4 -Type Fast
``
    <img src="images/sql26.jpg"/><br/>
11.	Compare the 2 query execution plans and determine what would be the reason for query slowness.<br/>
    <img src="images/sql27.jpg"/><br/>
    <img src="images/sql28.jpg"/><br/>
```
Hint: In this example, the query plan is optimal. This query could benefit if it was given more memory.  How much memory has been allocated to this query?  How can you use sys.dm_pdw_exec_requests to determine the memory grant? How can you change the memory allocation for a query?
```

## Discussion
In this exercise you will follow the same method to get to step 9. This time we see that the plan is a single step plan - a return operation. From a distributed plan perspective this is ideal because no data movement occurred. We simply were able to run the distributed queries on each distribution and return the results.<br/><br/>
For a query like this we cannot improve the MPP plan, so the next option to look at is resource class. You should have noticed that exec_requests shows that the query was running in smallRC. Certain queries will benefit from the larger memory allocation of larger resource classes and this usually requires testing to find the ideal balance of resource class usage for a query vs concurrency.<br/><br/>
Once you are at step 11, you should have looked at exec_requests and noticed that it was now running in LargeRC and the execution time was faster. These test queries are pretty fast running because the execution time is low, but for larger queries this can make a big difference. The default resource class is small. Remember, as you increase resource class you also decrease concurrency, so testing is required.<br/><br/>
If you want to change the resource class for a query you would use sp_addrolemember and sp_droprolemember

### Part 5: Query analysis
Now that you’ve helped your user with some of their initial issues, they’re beginning to bring some of their analysts onto the system – but some analysts are complaining that their queries are taking very long to run or don’t seem to be running at all.

1.	Open a PowerShell window.<br/>
2.	Change directory to Query Performance Tuning lab content folder.<br/>
3.	Change directory to Lab sub folder.<br/>
4.	Run “RunExercise.ps1” script with following parameters<br/>
``
“.\RunExercise.ps1 -Name Exercise5 -Type Slow”
``
    <img src="images/sql29.jpg"/><br/>
5.	This script will create a workload simulation on your server. It will create 20 background jobs which will send queries to your system.<br/>
6.	It will wait for 60 seconds for all background jobs properly starts and then will start your problematic query.<br/>
7.	Open Query editor of SQL Data Warehouse in Azure Portal.<br/>
8.	Check all the active queries.<br/>
```
SELECT * FROM sys.dm_pdw_exec_requests
WHERE status not in ('completed', 'failed', 'cancelled') AND session_id <> session_id()
ORDER BY request_id DESC;

```
9.	Can you tell what is happening?<br/>
``
Hint: What is the state of this query in sys.dm_pdw_exec_requests?  Why?
Hint: Run this query.  What does it tell you about the state of the query? 
Hint: What can be changed to ensure these small queries run? After you investigate connect to ‘demo5_fast’ to see the changes in action.
``
10.	You need to kill the background jobs before continuing.<br/>
11.	Cancel the running process on current PowerShell window.<br/>
12.	Run “.\Kill.ps1”<br/>
    <img src="images/kill1.jpg"/><br/>
13.	Make sure you close the PowerShell window.<br/>
14.	Open a PowerShell window.<br/>
15.	Change directory to Query Performance Tuning lab content folder.<br/>
16.	Change directory to Lab sub folder.<br/>
17.	Run “RunExercise.ps1” script with following parameters Run the same query with Fast option.<br/>
``
“.\RunExercise.ps1 -Name Exercise5 -Type Fast”
``
    <img src="images/sql32.jpg"/><br/>
18.	Compare the 2 query execution plans.<br/>
19.	You need to kill the background jobs before continuing.<br/>
20.	Cancel the running process on current PowerShell window.<br/>
21.	Run “.\Kill.ps1”
    <img src="images/kill.jpg"/><br/>
22.	Make sure you close the PowerShell window.

## Discussion
This exercise tries to simulate an active workload on your data warehouse. It creates 20 background sessions which sends queries constantly.<br/><br/>
When you reached to step 9 on Slow execution exercise, you will notice that your query is in the queue (“Suspended”) and does not “Running”. You can check wait stats and see that what is your query is waiting on “UserConcurrencyResourceType” which means that it is waiting for enough concurrency slots become available.<br/><br/>
When you check dm_pdw_exec_requests you will notice that this query is running on largerc resource class. In previous example we talk about using higher resource classes allow your query to have more memory resources. But this will result in more memory consumption from the overall system and resulted in less concurrent query executions. So you need to be careful about which resource classes you are using for executing your queries. Always test your queries with your actual workload.<br/><br/>
On faster version of this exercise you will notice that your queries might again queued but once there is enough concurrency slots available it will go through the execution. You will see that your query runs 3 times but at every execution it waits on the queue. You can check the queue waiting time by comparing start_time and submit_time in dm_pdw_exec_requests DMV.


