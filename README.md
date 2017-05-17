# dmv-queries
A list of ready Dynamic Management View (DMV) queries for finding bottlenecks in your database layer

```sql
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


-- Getting Stats on What Indexes are Used and What Indexes are Not
-- ------------------------------------------------------------------------------------------------
SELECT
    [DatabaseName] = DB_Name(db_id()),
    [TableName] = OBJECT_NAME(i.object_id),
    [IndexName] = i.name, 
    [IndexType] = i.type_desc,
    [TotalUsage] = IsNull(user_seeks, 0) + IsNull(user_scans, 0) + IsNull(user_lookups, 0),
    [UserSeeks] = IsNull(user_seeks, 0),
    [UserScans] = IsNull(user_scans, 0), 
    [UserLookups] = IsNull(user_lookups, 0),
    [UserUpdates] = IsNull(user_updates, 0)
FROM sys.indexes i 
INNER JOIN sys.objects o
    ON i.object_id = o.object_id
LEFT OUTER JOIN sys.dm_db_index_usage_stats s
    ON s.object_id = i.object_id
    AND s.index_id = i.index_id
WHERE 
    (OBJECTPROPERTY(i.object_id, 'IsMsShipped') = 0)
ORDER BY [TableName], [IndexName];
```
