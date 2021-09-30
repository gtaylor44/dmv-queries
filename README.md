# dmv-queries
A list of ready Dynamic Management View (DMV) queries for finding bottlenecks in your database layer

```sql

-- Get index statistics to see which indexes are being utilised
-- ------------------------------------------------------------------------------------------------
SELECT OBJECT_NAME(IX.OBJECT_ID) Table_Name
	   ,IX.name AS Index_Name
	   ,IX.type_desc Index_Type
	   ,SUM(PS.[used_page_count]) * 8 IndexSizeKB
	   ,IXUS.user_seeks AS NumOfSeeks
	   ,IXUS.user_scans AS NumOfScans
	   ,IXUS.user_lookups AS NumOfLookups
	   ,IXUS.user_updates AS NumOfUpdates
	   ,IXUS.last_user_seek AS LastSeek
	   ,IXUS.last_user_scan AS LastScan
	   ,IXUS.last_user_lookup AS LastLookup
	   ,IXUS.last_user_update AS LastUpdate
FROM sys.indexes IX
INNER JOIN sys.dm_db_index_usage_stats IXUS ON IXUS.index_id = IX.index_id AND IXUS.OBJECT_ID = IX.OBJECT_ID
INNER JOIN sys.dm_db_partition_stats PS on PS.object_id=IX.object_id
WHERE OBJECTPROPERTY(IX.OBJECT_ID,'IsUserTable') = 1
GROUP BY OBJECT_NAME(IX.OBJECT_ID) ,IX.name ,IX.type_desc ,IXUS.user_seeks ,IXUS.user_scans ,IXUS.user_lookups,IXUS.user_updates ,IXUS.last_user_seek ,IXUS.last_user_scan ,IXUS.last_user_lookup ,IXUS.last_user_update

-- Get top total worker time queries for entire instance (Query 43) (Top Worker Time Queries)
-- ------------------------------------------------------------------------------------------------
SELECT TOP(50) DB_NAME(t.[dbid]) AS [Database Name],
REPLACE(REPLACE(LEFT(t.[text], 255), CHAR(10),''), CHAR(13),'') AS [Short Query Text],
qs.total_worker_time AS [Total Worker Time], qs.min_worker_time AS [Min Worker Time],
qs.total_worker_time/qs.execution_count AS [Avg Worker Time],
qs.max_worker_time AS [Max Worker Time],
qs.min_elapsed_time AS [Min Elapsed Time],
qs.total_elapsed_time/qs.execution_count AS [Avg Elapsed Time],
qs.max_elapsed_time AS [Max Elapsed Time],
qs.min_logical_reads AS [Min Logical Reads],
qs.total_logical_reads/qs.execution_count AS [Avg Logical Reads],
qs.max_logical_reads AS [Max Logical Reads],
qs.execution_count AS [Execution Count],
CASE WHEN CONVERT(nvarchar(max), qp.query_plan) LIKE N'%<MissingIndexes>%' THEN 1 ELSE 0 END AS [Has Missing Index],
qs.creation_time AS [Creation Time]
--,t.[text] AS [Query Text], qp.query_plan AS [Query Plan] -- uncomment out these columns if not copying results to Excel
FROM sys.dm_exec_query_stats AS qs WITH (NOLOCK)
CROSS APPLY sys.dm_exec_sql_text(plan_handle) AS t
CROSS APPLY sys.dm_exec_query_plan(plan_handle) AS qp
ORDER BY qs.total_worker_time DESC OPTION (RECOMPILE);


-- CPU time per database
-- ------------------------------------------------------------------------------------------------
WITH DB_CPU_Stats
AS
(SELECT pa.DatabaseID, DB_Name(pa.DatabaseID) AS [Database Name], SUM(qs.total_worker_time/1000) AS [CPU_Time_Ms]
FROM sys.dm_exec_query_stats AS qs WITH (NOLOCK)
CROSS APPLY (SELECT CONVERT(int, value) AS [DatabaseID]
FROM sys.dm_exec_plan_attributes(qs.plan_handle)
WHERE attribute = N'dbid') AS pa
GROUP BY DatabaseID)
SELECT ROW_NUMBER() OVER(ORDER BY [CPU_Time_Ms] DESC) AS [CPU Rank],
[Database Name], [CPU_Time_Ms] AS [CPU Time (ms)],
CAST([CPU_Time_Ms] * 1.0 / SUM([CPU_Time_Ms]) OVER() * 100.0 AS DECIMAL(5, 2)) AS [CPU Percent]
FROM DB_CPU_Stats
WHERE DatabaseID <> 32767 -- ResourceDB
ORDER BY [CPU Rank] OPTION (RECOMPILE);


-- Finding Connection to Your Database
-- ------------------------------------------------------------------------------------------------
SELECT
    database_id,    -- SQL Server 2012 and after only
    session_id,
    status,
    login_time,
    cpu_time,
    memory_usage,
    reads,
    writes,
    logical_reads,
    host_name,
    program_name,
    host_process_id,
    client_interface_name,
    login_name as database_login_name,
    last_request_start_time
FROM sys.dm_exec_sessions
WHERE is_user_process = 1
ORDER BY cpu_time DESC;


-- Count of Connections by Login Name/Process (i.e. how many connections does an app have open)
-- ------------------------------------------------------------------------------------------------
SELECT
    login_name,
    host_name,
    host_process_id,
    COUNT(1) As LoginCount
FROM sys.dm_exec_sessions
WHERE is_user_process = 1
GROUP BY
    login_name,
    host_name,
    host_process_id;
	
	
-- Finding statements running in the database right now (including if a statement is blocked by another)
-- -----------------------------------------------------------------------------------------------
SELECT
        [DatabaseName] = db_name(rq.database_id),
        s.session_id, 
        rq.status,
        [SqlStatement] = SUBSTRING (qt.text,rq.statement_start_offset/2,
            (CASE WHEN rq.statement_end_offset = -1 THEN LEN(CONVERT(NVARCHAR(MAX),
            qt.text)) * 2 ELSE rq.statement_end_offset END - rq.statement_start_offset)/2),        
        [ClientHost] = s.host_name,
        [ClientProgram] = s.program_name, 
        [ClientProcessId] = s.host_process_id, 
        [SqlLoginUser] = s.login_name,
        [DurationInSeconds] = datediff(s,rq.start_time,getdate()),
        rq.start_time,
        rq.cpu_time,
        rq.logical_reads,
        rq.writes,
        [ParentStatement] = qt.text,
        p.query_plan,
        rq.wait_type,
        [BlockingSessionId] = bs.session_id,
        [BlockingHostname] = bs.host_name,
        [BlockingProgram] = bs.program_name,
        [BlockingClientProcessId] = bs.host_process_id,
        [BlockingSql] = SUBSTRING (bt.text, brq.statement_start_offset/2,
            (CASE WHEN brq.statement_end_offset = -1 THEN LEN(CONVERT(NVARCHAR(MAX),
            bt.text)) * 2 ELSE brq.statement_end_offset END - brq.statement_start_offset)/2)
    FROM sys.dm_exec_sessions s
    INNER JOIN sys.dm_exec_requests rq
        ON s.session_id = rq.session_id
    CROSS APPLY sys.dm_exec_sql_text(rq.sql_handle) as qt
    OUTER APPLY sys.dm_exec_query_plan(rq.plan_handle) p
    LEFT OUTER JOIN sys.dm_exec_sessions bs
        ON rq.blocking_session_id = bs.session_id
    LEFT OUTER JOIN sys.dm_exec_requests brq
        ON rq.blocking_session_id = brq.session_id
    OUTER APPLY sys.dm_exec_sql_text(brq.sql_handle) as bt
    WHERE s.is_user_process =1
        AND s.session_id <> @@spid
 AND rq.database_id = DB_ID()  -- Comment out to look at all databases
    ORDER BY rq.start_time ASC;
	
	
-- Finding the most expensive statements in your database
-- ------------------------------------------------------------------------------------------------
SELECT TOP 20    
        DatabaseName = DB_NAME(CONVERT(int, epa.value)), 
        [Execution count] = qs.execution_count,
        [CpuPerExecution] = total_worker_time / qs.execution_count ,
        [TotalCPU] = total_worker_time,
        [IOPerExecution] = (total_logical_reads + total_logical_writes) / qs.execution_count ,
        [TotalIO] = (total_logical_reads + total_logical_writes) ,
        [AverageElapsedTime] = total_elapsed_time / qs.execution_count,
        [AverageTimeBlocked] = (total_elapsed_time - total_worker_time) / qs.execution_count,
     [AverageRowsReturned] = total_rows / qs.execution_count,    
     [Query Text] = SUBSTRING(qt.text,qs.statement_start_offset/2 +1, 
            (CASE WHEN qs.statement_end_offset = -1 
                THEN LEN(CONVERT(nvarchar(max), qt.text)) * 2 
                ELSE qs.statement_end_offset end - qs.statement_start_offset)
            /2),
        [Parent Query] = qt.text,
        [Execution Plan] = p.query_plan,
     [Creation Time] = qs.creation_time,
     [Last Execution Time] = qs.last_execution_time   
    FROM sys.dm_exec_query_stats qs
    CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) as qt
    OUTER APPLY sys.dm_exec_query_plan(qs.plan_handle) p
    OUTER APPLY sys.dm_exec_plan_attributes(plan_handle) AS epa
    WHERE epa.attribute = 'dbid'
        AND epa.value = db_id()
    ORDER BY [AverageElapsedTime] DESC; --Other column aliases can be used
	
	
-- Looking for Missing Indexes
-- ------------------------------------------------------------------------------------------------
SELECT     
    TableName = d.statement,
    d.equality_columns, 
    d.inequality_columns,
    d.included_columns, 
    s.user_scans,
    s.user_seeks,
    s.avg_total_user_cost,
    s.avg_user_impact,
    AverageCostSavings = ROUND(s.avg_total_user_cost * (s.avg_user_impact/100.0), 3),
    TotalCostSavings = ROUND(s.avg_total_user_cost * (s.avg_user_impact/100.0) * (s.user_seeks + s.user_scans),3)
FROM sys.dm_db_missing_index_groups g
INNER JOIN sys.dm_db_missing_index_group_stats s
    ON s.group_handle = g.index_group_handle
INNER JOIN sys.dm_db_missing_index_details d
    ON d.index_handle = g.index_handle
WHERE d.database_id = db_id()
ORDER BY TableName, TotalCostSavings DESC;
```
