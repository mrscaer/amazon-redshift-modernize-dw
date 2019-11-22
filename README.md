# re:Invent 2019 - Redshift Workshop Zero to Hero [ANT237]
In this lab you will launch a new Redshift Cluster, setup connectivity and work with 5 important considerations to get the most out of Redshift (SET_DW):

* **S**ort Key
* **E**ncoding (column compression)
* **T**able fitness (Auto Analyze and Auto Vacuum)
* **D**istribtion Key
* **W**orkload Management (WLM)



## Before You Begin
Please get the workshop account credentials from the workshop leaders.

# SECTION #1: Setup and Encoding (~30 minutes)

### Challenge #1a: Login with the provided credentials. This will automatically spin-up an environment using a CloudFormation template:

Login with the provided workshop credentials at https://dashboard.eventengine.run/login

Change to the correct region (us-west-2).

Optionally, you can look at the events completed by the CloudFormation template from the CloudFormation Console:

![GitHub Logo](/images/toronto_cloudformation_events.jpg)

And/or review the Redshift Console 'Clusters' section for the cluster to confirm that the cluster is available:

![GitHub Logo](/images/toronto_redshift_create_events.jpg)


Please proceed with the workshop once the Redshift cluster is available.

### Challenge #1b: Connect to the Redshift cluster with the built-in query editor and create a view that is used later:

Find and connect to the Query Editor (or you are welcome to use your own editor). The defaults for the workshop are:

* DB: dev
* Master Username: awsuser

Here's the SQL for the view:

```
CREATE OR REPLACE VIEW public.v_describe_table AS
SELECT DISTINCT 
       a.attnum AS column_position,
       n.nspname AS schema_name,
       c.relname AS table_name,
       a.attname AS column_name,
       pg_catalog.format_type(a.atttypid,a.atttypmod) AS data_type,
       pg_catalog.format_encoding(a.attencodingtype) AS compression_type,
       CASE WHEN a.attisdistkey IS TRUE THEN a.attisdistkey ELSE NULL END AS distkey,
       CASE WHEN a.attsortkeyord > 0 THEN a.attsortkeyord ELSE NULL END AS sortkey
FROM pg_catalog.pg_namespace n,
     pg_catalog.pg_class c,
     pg_catalog.pg_attribute a,
     pg_catalog.pg_stats stats
WHERE n.oid = c.relnamespace
AND   c.oid = a.attrelid
AND   a.attnum > 0
AND   c.relname NOT LIKE '%pkey' 
ORDER BY A.ATTNUM;
```



### Challenge #1c: Use the (simulated) Redshift Advisor and compare the impact of compressed and uncompressed columns

The Redshift Advisor is an important element of Redshift’s ease of use proposition. The Advisor will warn if there are tables without compression, which in turn leads to scanning work than would otherwise be required to execute the query.

![GitHub Logo](/images/toronto_advisor_compress_table.jpg)

Query the Redshift system view SVV_TABLE_INFO to see the list of tables on the cluster.

`   SELECT * FROM svv_table_info;`

The description of the view is at [https://docs.aws.amazon.com/redshift/latest/dg/r\_SVV\_TABLE\_INFO.html](https://docs.aws.amazon.com/redshift/latest/dg/r_SVV_TABLE_INFO.html)

Examine the public.orders table to see some attributes that support efficient usage:

`   SELECT * FROM public.v_describe_table WHERE schema_name = 'public' AND table_name = 'orders';`

Now, let’s look at the impact of compression on the number of blocks written to disk:

Start by creating and populating an uncompressed version of the orders table:

```
CREATE TABLE public.uncompressed_orders
(
o_orderkey BIGINT ENCODE RAW,
o_custkey BIGINT ENCODE RAW,
o_orderstatus CHARACTER VARYING(1) ENCODE RAW,
o_totalprice NUMERIC(18,4) ENCODE RAW,
o_orderdate DATE ENCODE RAW,
o_orderpriority CHARACTER VARYING(15) ENCODE RAW,
o_clerk CHARACTER VARYING(15) ENCODE RAW,
o_shippriority INTEGER ENCODE RAW,
o_comment CHARACTER VARYING(79) ENCODE RAW
)
DISTSTYLE KEY DISTKEY (o_orderkey)
SORTKEY (o_orderdate);
```

`INSERT INTO public.uncompressed_orders SELECT * FROM public.orders;`

`ANALYZE public.uncompressed_orders;`

Getting the table IDs from SVV\_TABLE\_INFO (where we can actually look at the ‘size’ column to see the number of 1MB blocks, we can further examine this at the slice level with a query against STV_BLOCKLIST:

```
SELECT tbl,
       slice,
       COUNT(blocknum) AS num_blocks
FROM stv_blocklist
WHERE tbl IN (100485,100392) /* <-- Your table IDs go here. */
GROUP BY 1,2
ORDER BY 1,2
;
```

Or you can join to SVV\_TABLE\_INFO and use the table names:

```
SELECT b.tbl,
       b.slice,
       COUNT(b.blocknum) AS num_blocks
FROM stv_blocklist b, svv_table_info t
WHERE b.tbl = t.table_id 
and t."table" IN ('orders','uncompressed_orders') 
and t.schema = 'public' 
GROUP BY 1,2
ORDER BY 1,2
;
```


### Bonus Challenge #1d: Used a stored procedure to analyze a single 'best' column

Redshift has the ability to run stored procedures, try the stored proc "MinAnalyze" to quickly collect stats on public.uncompressed columns. How is this different from ANALYZE PREDICATE COLUMNS? The procedure is in the Redshift Engineer's GitHub repository: [https://github.com/awslabs/amazon-redshift-utils/blob/master/src/StoredProcedures/sp_analyze_minimal.sql](https://github.com/awslabs/amazon-redshift-utils/blob/master/src/StoredProcedures/sp_analyze_minimal.sql)



# Section #2: Distribution (~30 minutes)

### Challenge #2a: Use the (simulated) Redshift Advisor to identify a table with row skew.
The Redshift Advisor is an important element of Redshift’s ease of use proposition. Looking at the example Advisor alert, build a skewed table, understand the magnitude of the tuning opportunity, then quickly resolve the issue.


![GitHub Logo](/images/vegas_advisor_skewed_table.jpg)


### Challenge #2b: Deliberately create and populate a table with high row skew to see how we can analyze it.



Load another version of the orders table, deliberately fouling the distribution:

```
CREATE TABLE public.bad_orders
(
o_orderkey BIGINT,
o_custkey BIGINT,
o_orderstatus CHARACTER VARYING(1),
o_totalprice NUMERIC(18,4),
o_orderdate DATE,
o_orderpriority CHARACTER VARYING(15),
o_clerk CHARACTER VARYING(15),
o_shippriority INTEGER,
o_comment CHARACTER VARYING(79)
)
DISTSTYLE KEY DISTKEY (o_orderkey)
SORTKEY (o_orderdate);
```

```
ALTER TABLE public.bad_orders ALTER DISTKEY o_shippriority;
```

![GitHub Logo](/images/toronto_iam.jpg)

Get the IAM role from the Redshift Console, in the cluster details, under ‘See IAM Roles’. 

Note that there’s no need to specify compression for each column, Redshift will apply column-level compression heuristics automatically. 

```
COPY bad_orders 
FROM 's3://redshift-immersionday-labs/data/orders/orders.tbl.' 
iam_role 'arn:aws:iam::999999999999:role/RedshiftImmersionRole' /* <-- Your role goes here. */
region 'us-west-2' 
lzop delimiter '|' ;
```

### Challenge #2c: Try a few ways to see the skew information. 
Use the SVV\_TABLE\_INFO to see the skew:
```
   SELECT * FROM svv_table_info;
```

To understand the impact of the skew, look at the block level on the disk for the cluster.

```
SELECT s.node,
s.slice,
COUNT(NVL (b.blocknum,0)) AS num_blocks
FROM stv_slices s
LEFT OUTER JOIN (SELECT slice, blocknum FROM stv_blocklist
WHERE tbl = 100459 /* <--   Your table ID goes into the next line: */
) b
ON (b.slice = s.slice)
GROUP BY s.node, s.slice
ORDER BY s.node, s.slice
;
```

### Challenge #2d: Change the distribution key on the table with an ALTER TABLE command:

Changing the table distribution can be done with the ALTER TABLE command:

```
  ALTER TABLE public.bad_orders ALTER DISTKEY o_orderkey;
```

### Bonus Challenge #2e: Working with Redshift AUTO distribution. 

Launched in Q1 2019, Redshift can now pick the table distibution for you, and even update the distribution style as the number of rows in the table increases. You can test this by creating a table and running a few COPY commands:

Create a test table:
```
CREATE TABLE IF NOT EXISTS public.supplier_test (
  s_suppkey bigint 
 ,s_name character varying(25) 
 ,s_address character varying(40) 
 ,s_nationkey bigint 
 ,s_phone character varying(15) 
 ,s_acctbal numeric(18,4) 
 ,s_comment character varying(101) 
) 
DISTSTYLE AUTO /* <--- This is actually optional, as it's the default for the CREATE TABLE command. */
;
```

Load the table with data from S3 using the COPY command:

```
COPY supplier_test FROM 's3://redshift-managed-loads-datasets-us-east-1/dataset=tpch/size=1GB/table=supplier/supplier.manifest'
iam_role 'arn:aws:iam::999999999999:role/RedshiftImmersionRole' /* <-- Your role goes here. Scroll up to Challenge #2b:
 to find the role. */
region 'us-east-1' gzip delimiter '|' COMPUPDATE OFF MANIFEST;
```

Now, query the SVV_TABLE_INFO view and note the column 'diststyle'.

Run the COPY command again, followed by the SVV_TABLE_INFO query. After a few loops, did you see the distribution style change automatically on the table?

Note: The hueristic being used by Redshift takes into account several considerations (such as cluster size and number of blocks); so it's not possible to document the rules that would correct across the entire Redshift fleet. The Redshift Advisor now makes recommendations for distribution key changes based on actual query patterns:

![GitHub Logo](/images/toronto_advisor_dist_key.jpg)

# Section #3: Sort Key Columns (~30 minutes)

### Introduction:
One of the tools for querying very large tables quickly is to implement a sort key, which can be several columns. Redshift will physically reorder the data to minimize the amount of scanning that would otherwise be required. You use the Redshift sort key in a similar way as you would use partition elimination in other databases.

Inherent to Redshift is a min and max value for each 1MB block for each column for each table. This in-memory structure is called the zone map. By identifying a sort key set of the columns, Redshift can amplify the effectiveness of the zone map. 

### Challenge #3a: Quantify the query benefit of Redshift automatically using the zone map on a table:
Study the decrease in the amount of data scanned between the two queries, nothing that column ‘o\_totalprice’ isn’t part of the sort key. The decrease is attributed to Redshift leveraging the zone map and only looking at blocks where the ‘o_totalprice’ for a particular block is between the min and max according to the zone map. No user action is ever required with respect to the zone map, it’s an embedded and automatic feature of the service.

```
SELECT MIN(o_totalprice) FROM public.orders WHERE o_totalprice < 812;
```

```

SELECT *
FROM stl_scan
WHERE query = pg_last_query_id()
AND   perm_table_name NOT LIKE 'Internal Worktable%'
ORDER BY slice, segment, step
;
```

```
SELECT MIN(o_totalprice) FROM public.orders WHERE o_totalprice > 500000;
```

```
SELECT *
FROM stl_scan
WHERE query = pg_last_query_id()
AND   perm_table_name NOT LIKE 'Internal Worktable%'
ORDER BY slice, segment, step
;
```

### Challenge #3b: Working with sort keys to improve query performance: 
For the purpose of analysis, let’s build two versions of the public.orders table and compare the number of rows needed to be scanned with and without a sort key column.

Create and populate the version of the table with the sort key defined.

```
CREATE TABLE public.orders_sort 
(
  o_orderkey        BIGINT,
  o_custkey         BIGINT,
  o_orderstatus     CHARACTER VARYING(1),
  o_totalprice      NUMERIC(18,4),
  o_orderdate       DATE SORTKEY,
  o_orderpriority   CHARACTER VARYING(15),
  o_clerk           CHARACTER VARYING(15),
  o_shippriority    INTEGER,
  o_comment         CHARACTER VARYING(79)
);

INSERT INTO public.orders_sort SELECT * FROM public.orders;
ANALYZE public.orders_sort;
VACUUM public.orders_sort;
```

Now, a version without the sort key, but holding everything else constant.

```
CREATE TABLE public.orders_no_sort 
(
  o_orderkey        BIGINT,
  o_custkey         BIGINT,
  o_orderstatus     CHARACTER VARYING(1),
  o_totalprice      NUMERIC(18,4),
  o_orderdate       DATE,
  o_orderpriority   CHARACTER VARYING(15),
  o_clerk           CHARACTER VARYING(15),
  o_shippriority    INTEGER,
  o_comment         CHARACTER VARYING(79)
);

INSERT INTO public.orders_no_sort SELECT * FROM public.orders;
ANALYZE public.orders_no_sort;
VACUUM public.orders_no_sort;
```

Now, test the performance of both tables on (otherwise) the same SQL. Get the query ID for later analysis:

```
SELECT MIN(o_custkey),
       MAX(o_custkey)
FROM (SELECT o_custkey,
             SUM(o_totalprice)
      FROM orders_sort
      WHERE o_orderdate BETWEEN '1992-07-05' AND '1992-07-07'
      GROUP BY o_custkey
      )
;

SELECT pg_last_query_id();

SELECT MIN(o_custkey),
       MAX(o_custkey)
FROM (SELECT o_custkey,
             SUM(o_totalprice)
      FROM orders_no_sort
      WHERE o_orderdate BETWEEN '1992-07-05' AND '1992-07-07'
      GROUP BY o_custkey
     )
;

SELECT pg_last_query_id();
```

Then, look at the number of rows processed in the columns 'rows_pre_filter' and 'rows_pre_user_filter' for your two queries

```
SELECT query,
       slice,
       rows_pre_filter,
       rows_pre_user_filter,
       perm_table_name,
       ROWS,
       bytes
FROM stl_scan
WHERE query IN ({your comma-delimited query IDs go here.})
ORDER BY 1,2
;

```




# Section #4: Workload Management (WLM) (~30 minutes)
### Introduction:
Redshift Workload Management (WLM) is largely an automated process where the cluster administrator defines categories of workload defined by either the user’s group affiliation or a declared query group, and assigns a priority (such as Low, Normal, High) so Redshift knows how much of the system to allocate per bucket. If the administrator chooses to do so, they can further define the number of simultaneous jobs per category and the percent of the cluster’s memory reserved per job. Most workloads should consider the first option (Auto WLM with Priorities) before experimenting with a manual WLM setting.

The Redshift Advisor can warn on underutilized categories of workload. For example, if the College Intern group was given their own WLM queue, but they’ve all returned to school, the queue would be unused. This is primarily a concern for the manual WLM setting described in the second scenario above.

![GitHub Logo](/images/toronto_advisor_wlm.jpg)


### Challenge #4a: Setting-up with Auto WLM with Priorities:
On the Redshift Console, choose ‘Workload management’ from the left nav, then create a new Parameter Group.
Confirm that the parameter is already in Auto WLM with Priorities (or switch it to Auto WLM with Priorities).
Create at least two queues, their names and priorities (Lowest, Low, Normal, etc.) are up to you. 
Use the Query Group option to identify jobs that should run in each queue. Note the Default queue for all other jobs.
Save your changes to the Parameter Group, then update the cluster to use the new Parameter Group. Note this will require a bounce of the cluster (this is only required when the number of WLM queues is changing).

### Challenge #4b: Working with Auto WLM with Priorities:
From the Query editor, write a couple of SQL statements (at least one per queue you defined), then set a session variable to assign each query to one of the query groups, such as:

```
SET query_group TO 'dashboard';
or
SET query_group TO 'college_interns';
```

Until you change the query_group session variable (or RESET it), those jobs will route to the correct queue. Verify with the following SQL statement:

```
SELECT 
   wq.query,
   wq.service_class,
   wq.exec_start_time,
   ROUND((DATEDIFF(mseconds, exec_start_time, exec_end_time) / 1000.0),2) AS run_secs,
   wq.query_priority,
   LISTAGG(TRIM(wcc.condition),'|') AS condition,
   SUBSTRING(qt.text,0,100)
FROM stl_wlm_query wq,
   stv_wlm_classification_config wcc,
   stl_querytext qt
WHERE wq.query = qt.query
   AND wq.service_class = wcc.action_service_class
   AND wcc.id > 6
GROUP BY 1,2,3,4,5,7
ORDER BY wq.exec_start_time DESC
LIMIT 25;
```

Try switching to other query_groups and noting how WLM routes the jobs.

### Challenge #4c: Using database User Group with Auto WLM with Priorities:
Another popular way to route jobs through Redshift Workload Management is to assign entire user groups to WLM queues. To test this, you’ll need to create a handful of users, create some number of groups, and update your Parameter Group to use the user group.

Make any users and groups that you want, below is included as an example:

```
CREATE USER alex WITH password '123_2112_Ruser' NOCREATEUSER;
CREATE USER neil WITH password '123_mp_Ruser' NOCREATEUSER;
CREATE USER geddy WITH password '123_ca_Ruser' NOCREATEUSER;
```

```
CREATE GROUP group_rush WITH USER alex, neil, geddy;
```

```
GRANT USAGE ON SCHEMA public TO GROUP group_rush;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO GROUP group_rush;
```

Note: Since you are logged-in as an administrator, the SESSION_AUTHORIZATION is available, such as:

```
SET SESSION AUTHORIZATION 'neil';
SET SESSION AUTHORIZATION default;
```


More at [https://docs.aws.amazon.com/redshift/latest/dg/r\_SET\_SESSION_AUTHORIZATION.html ](https://docs.aws.amazon.com/redshift/latest/dg/r_SET_SESSION_AUTHORIZATION.html )


Test by using some of the users/groups you created above, and evaluate with the WLM system table query from Challenge #4b. 

* Why wasn’t a bounce of the cluster required this time?
* What’s the relationship between the User and Query groups? Is access to the WLM an ‘AND’ or an ‘OR’ relationship between the two groups?
* What would be a scenario where this relationship could be especially useful?

### Bonus Challenge #4d: Test the wildcard option for either User or Query Groups.
Implement a solution using the wildcard option. How can this simplify the rules (and number of queues)?

## Bonus
* How to use snapshot to create a QA or DEV cluster quickly?
* How can you use Amazon QuickSight to crawl and then visualize your Redshift data?

# Before You Leave
When you are done with the workshop, feel free to delete the stack from the CloudFormation Console.

![GitHub Logo](/images/toronto_advisor_stack_delete.jpg)

