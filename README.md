# Dashboard-Monitoring-Snowflake
Step-by-step to create a Dasboard in Snowsight to monitor usage

Admin Dashboard Sample
The below queries aim to assist in setting up an Administrator Dashboard on Snowflake to drill in even deeper into what is going on in your Snowflake environment. See below for a sample completed output of what an Admin Dashboard would look like


Admin Dashboard Credit Usage Details

Admin Dashboard: Warehouse Usage and Query Details

Admin Dashboard: Metrics on Credits Billed, Average Query Execution by User, and Cloud Services details

Admin Dashboard: Data Storage Usage, Total Rows Loaded, and Logins by clients and users
Setting up an Admin Dashboard on Snowflake
In the below section I will go over each of the setup steps to configure the dashboard tile by tile. The below queries and setup are provided to assist when creating an Admin Dashboard using Snowsight. Prior to utilizing, please be sure you are either running with the ACCOUNTADMIN privilege or the role has the ability to use the ACCOUNT_USAGE schema.

For instructions on how to Visualize Data in Snowsight utilize the following (here). Each query below is its own tile on the dashboard with the screenshots showcasing how to configure the dashboard.

Note: Each tile can be placed and resized to your liking. By double-clicking the title of each worksheet each tile can also be renamed.

Credits Used

select
    sum(credits_used)
from
    account_usage.metering_history
where
    start_time = :daterange;

Credits Used Visualization Tile Setup
Total Number of Jobs Executed

select
    count(*) as number_of_jobs
from
    account_usage.query_history
where
    start_time >= date_trunc(month, current_date);

Total # Jobs Executed Tile Setup
Current Storage

select
    avg(storage_bytes + stage_bytes + failsafe_bytes) / power(1024, 4) as billable_tb
from
    account_usage.storage_usage
where
    USAGE_DATE = current_date() -1;

Current Storage (TB) Tile Setup
Credit Usage by Warehouse

select
    warehouse_name,
    sum(credits_used) as total_credits_used
from
    account_usage.warehouse_metering_history
where
    start_time = :daterange
group by
    1
order by
    2 desc;

Credit Usage by Warehouse
Credit Usage Overtime

select
    start_time::date as usage_date,
    warehouse_name,
    sum(credits_used) as total_credits_used
from
    account_usage.warehouse_metering_history
where
    start_time = :daterange
group by
    1,
    2
order by
    2,
    1;

Credit Usage Overtime Tile Setup
Warehouse Usage Greater than 7 Day Average

SELECT
    WAREHOUSE_NAME,
    DATE(START_TIME) AS DATE,
    SUM(CREDITS_USED) AS CREDITS_USED,
    AVG(SUM(CREDITS_USED)) OVER (
        PARTITION BY WAREHOUSE_NAME
        ORDER BY
            DATE ROWS 7 PRECEDING
    ) AS CREDITS_USED_7_DAY_AVG,
    (TO_NUMERIC(SUM(CREDITS_USED)/CREDITS_USED_7_DAY_AVG*100,10,2)-100)::STRING || '%' AS VARIANCE_TO_7_DAY_AVERAGE
FROM
    "SNOWFLAKE"."ACCOUNT_USAGE"."WAREHOUSE_METERING_HISTORY"
WHERE
    START_TIME = :daterange
GROUP BY
    DATE,
    WAREHOUSE_NAME
ORDER BY
    DATE DESC;

Credit Usage Greater than a 7 Day Average Tile Setup
Execution Time by Query Type (Avg Seconds)

select
    query_type,
    warehouse_size,
    avg(execution_time) / 1000 as average_execution_time
from
    account_usage.query_history
where
    start_time = :daterange
group by
    1,
    2
order by
    3 desc;

Execution Time by Query Type Tile Setup
Top 25 Longest Queries

select
    query_id,
    query_text,
    (execution_time / 60000) as exec_time
from
    account_usage.query_history
where
    execution_status = 'SUCCESS'
order by
    execution_time desc
limit
    25

Top 25 Longest Queries Tile Setup
Total Execution Time by Repeated Queries

select
    query_text,
    (sum(execution_time) / 60000) as exec_time
from
    account_usage.query_history
where
    execution_status = 'SUCCESS'
group by
    query_text
order by
    exec_time desc
limit
    25;

Total Execution Time by Repeated Queries Tile Setup
Credits Billed by Month

select
    date_trunc('MONTH', usage_date) as Usage_Month,
    sum(CREDITS_BILLED)
from
    account_usage.metering_daily_history
group by
    Usage_Month;

Credits Billed by Month Tile Setup
Average Query Execution Time (By User)

select
    user_name,
    (avg(execution_time)) / 1000 as average_execution_time
from
    account_usage.query_history
where
    start_time = :daterange
group by
    1
order by
    2 desc;

Average Query Execution (By User) Tile Setup
GS Utilization by Query Type (Top 10)

select
    query_type,
    sum(credits_used_cloud_services) cs_credits,
    count(1) num_queries
from
    account_usage.query_history
where
    true
    and start_time >= timestampadd(day, -1, current_timestamp)
group by
    1
order by
    2 desc
limit
    10;

GS Utilization by Query Type Tile Setup
Compute and Cloud Services by Warehouse

select
    warehouse_name,
    sum(credits_used_cloud_services) credits_used_cloud_services,
    sum(credits_used_compute) credits_used_compute,
    sum(credits_used) credits_used
from
    account_usage.warehouse_metering_history
where
    true
    and start_time = :daterange
group by
    1
order by
    2 desc
limit
    10;

Compute and Cloud Service by Warehouse Tile Setup
Credit Breakdown by Day with Cloud Services Adjustment

select
    *
from
    account_usage.metering_daily_history
where
    USAGE_DATE = :daterange;

Credit Breakdown by Day w/Cloud Services Adjustment
Data Storage used Overtime

select
    date_trunc(month, usage_date) as usage_month,
    avg(storage_bytes + stage_bytes + failsafe_bytes) / power(1024, 4) as billable_tb,
    avg(storage_bytes) / power(1024, 4) as Storage_TB,
    avg(stage_bytes) / power(1024, 4) as Stage_TB,
    avg(failsafe_bytes) / power(1024, 4) as Failsafe_TB
from
    account_usage.storage_usage
group by
    1
order by
    1;

Data Storage Usage (1 Year) Tile Setup
Total Row Loaded by Day

SELECT
 TO_CHAR(TO_DATE(load_history.LAST_LOAD_TIME),
 'YYYY-MM-DD') AS "load_history.last_load_time_date",
 COALESCE(SUM(load_history.ROW_COUNT),
 0) AS "load_history.total_row_count"
FROM
 SNOWFLAKE.ACCOUNT_USAGE.LOAD_HISTORY AS load_history
WHERE
 (((load_history.LAST_LOAD_TIME) = :daterange
 AND (load_history.LAST_LOAD_TIME) = :daterange))
GROUP BY
 TO_DATE(load_history.LAST_LOAD_TIME)
ORDER BY
 1 DESC;

Total Rows Loaded Tile Setup
Logins by User

select
    user_name,
    sum(iff(is_success = 'NO', 1, 0)) as Failed,
    count(*) as Success,
    sum(iff(is_success = 'NO', 1, 0)) / nullif(count(*), 0) as login_failure_rate
from
    account_usage.login_history
where
    event_timestamp = :daterange
group by
    1
order by
    4 desc;

Logins by User Tile Setup
Logins by Client

select
    reported_client_type as Client,
    user_name,
    sum(iff(is_success = 'NO', 1, 0)) as Failed,
    count(*) as Success,
    sum(iff(is_success = 'NO', 1, 0)) / nullif(count(*), 0) as login_failure_rate
from
    account_usage.login_history
where
    event_timestamp = :daterange
group by
    1,
    2
order by
    5 desc;

Logins by Client Tile Setup
Once all the above setup is complete you are now all ready to go to utilize the Admin Dashboard on Snowflake. Once again, you can rearrange and add/remove tiles as you need. If there is anything missing that you’d like to see please don’t hesitate to put it in the comments below!
