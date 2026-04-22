author: yarodmi
id: vhol-data-vault
categories: snowflake-site:taxonomy/solution-center/certification/quickstart, snowflake-site:taxonomy/product/data-engineering
language: en
summary: Building a Real-time Data Vault in Snowflake 
environments: web
status: Published 
feedback link: https://github.com/Snowflake-Labs/sfguides/issues

# Building a Real-Time Data Vault in Snowflake
<!-- ------------------------ -->
## Continuous Data

Today, with the ever-increasing availability and volume of data from many types of sources such as IoT, mobile devices, and weblogs, there is a growing need, and yes, demand, to go from batch load processes to streaming continuous real-time (RT) data. Businesses change at a rapid rate, becoming more competitive all the time. Those that can harness the value of their data faster to drive better decision making and better business outcomes will prevail.

One of the benefits of using the Data Vault system is that it was designed from inception not only to accept data loaded using traditional batch mode (which was the prevailing mode in the early 2000s when [Dan Linstedt](https://datavaultalliance.com/#about) introduced Data Vault) but also to easily accept data loading continuously in real or near-realtime (NRT). In the early 2000s, that was a nice-to-have aspect of the approach and meant the methodology was effectively future-proofed from that perspective. Still, few database systems had the capacity to support that kind of requirement. Today, RT or at least NRT loading is almost becoming a mandatory requirement for modern data platforms. Granted, not all loads or use cases need to be NRT, but most forward-thinking organizations need to onboard data for analytics in an NRT manner.

Those who have been using the Data Vault approach don’t need to change much other than figure out how to engineer their data pipeline to serve up data to the Data Vault in NRT. The data models don’t need to change; the reporting views don’t need to change; even the loading patterns don’t need to change. For those that aren’t using Data Vault already, if they have real-time loading requirements, this architecture and method might be worth considering.

### Data Vault on Snowflake

There have been numerous [blog posts](/blog/tips-for-optimizing-the-data-vault-architecture-on-snowflake/), user groups, and webinars over the years, discussing the best practices and customer success stories around implementing Data Vaults on Snowflake.  So the question now is how do you build a Data Vault on Snowflake that has real-time or near real-time continuous data streaming into it.

Luckily, streaming data is one of the [use-cases](/cloud-data-platform/) that Snowflake was built to support, so we have many features to help us achieve this goal. **This guide is an extended version of the [article](https://datavaultalliance.com/news/building-a-real-time-data-vault-in-snowflake/) posted on Data Vault Alliance website, now including practical steps to build an example of real-time Data Vault feed on Snowflake. Join us on simple-to-follow steps to see it in action.**

### Prerequisites
- A Snowflake account, set up using the [Defensible Analytics using Data Vault and Snowflake](https://www.snowflake.com/en/developers/guides/defensible-analytics-using-data-vault-and-snowflake/) guide. If you don't use this guide to set up your account, you'll need to alter the references to roles, databases, schemas, warehouses, and other Snowflake objects to work with your account.
- Familiarity with [Snowflake key concepts and architecture](https://docs.snowflake.com/en/user-guide/intro-key-concepts)
- Familiarity with [Data Vault methodology and architecture](https://datavaultalliance.com/#resources)

### What You’ll Learn
- How to use Data Vault modeling on Snowflake
- How to build basic objects and write ELT code for them
- How to leverage [Snowpipe](https://docs.snowflake.com/en/user-guide/data-load-snowpipe-intro.html) and [Continuous Data Pipelines](https://docs.snowflake.com/en/user-guide/data-pipelines.html) to automate data processing
- How to apply data virtualization to accelerate data access

### What You’ll Need 
- A Snowflake account -- we recommend starting with a [trial](https://trial.snowflake.com/) account

### What You’ll Build 
- Data Vault models on Snowflake, based on sample dataset 
- Data pipelines, leveraging streams, tasks and Snowpipe


<!-- ------------------------ -->
## Reference Architecture and Environment Setup

This is the target architecture. 

![Multi-Tier Data Vault Architecture](assets/img1.png)  

In the [Defensible Analytics using Data Vault and Snowflake](https://www.snowflake.com/en/developers/guides/defensible-analytics-using-data-vault-and-snowflake/) guide, we implement components of this architecture, which we'll use in this guide. If you haven't already executed the steps in that guide in your Snowflake account, you'll need to do that now, or change the examples given in this guide to accommodate your design.


<!-- ------------------------ -->
## Data Pipelines: Design

![Data Pipeline Architecture](assets/img2.png)  

Snowflake supports multiple options for engineering data pipelines. In this post we are going to show one of the most efficient ways to implement incremental NRT integration leveraging Snowflake [Continuous Data Pipelines](https://docs.snowflake.com/en/user-guide/data-pipelines.html).  Let's take a look at the architecture diagram above to understand how it works. 

Snowflake has a special [stream](https://docs.snowflake.com/en/user-guide/streams.html) object that tracks all data changes on a table (inserts, updates, and deletes). This process is 100% automatic and unlike traditional databases will never impact the speed of data loading. The change log from a stream is automatically ‘consumed’ once there is a successfully completed DML operation using the stream object as a source. 

So, loading new data into a staging table, would immediately be reflected in a stream showing the [‘delta’](https://docs.snowflake.com/en/user-guide/streams.html#data-flow) that requires processing.

The second component we are going to use is [tasks](https://docs.snowflake.com/en/user-guide/tasks-intro.html). It is a Snowflake managed data processing unit that will wake up on a defined interval (e.g., every 1-2 min), check if there is any data in the associated stream and if so, will run SQL to push it to the Raw Data Vault objects. Tasks could be arranged in a [tree-like dependency graph](https://docs.snowflake.com/en/user-guide/tasks-intro.html#simple-tree-of-tasks), executing child tasks the moment the predecessor finished its part. 

Last but not least, following Data Vault 2.1 best practices for NRT data integration (to load data in parallel) we are going to use Snowflake’s [multi-table insert (MTI)](https://docs.snowflake.com/en/sql-reference/sql/insert-multi-table.html) inside tasks to populate multiple Raw Data Vault objects by a single DML command. (Alternatively you can create multiple streams & tasks from the same table in stage in order to populate each data vault object by its own asynchronous flow.) 

Next step, you assign tasks to one or many virtual warehouses. This means you always have enough [compute power](https://docs.snowflake.com/en/user-guide/warehouses-overview.html#warehouse-size) (XS to 6XL) to deal with any size workload, whilst the [multi-cluster virtual warehouse](https://docs.snowflake.com/en/user-guide/warehouses-multicluster.html#multi-cluster-warehouses) option will automatically scale-out and load balance all the tasks as you introduce more hubs, links and satellites to your vault. 

Talking about tasks, Snowflake just introduced another fantastic capability - serverless tasks. This enables you to rely on compute resources managed by Snowflake instead of user-managed virtual warehouses. These compute resources are automatically resized and scaled up and down by Snowflake as required by each workload. This feature will be out of scope for this guide, but serverless compute model could reduce compute costs, in some cases significantly, allowing you to process more data, faster with even less management. 

As your raw vault is updated, streams can then be used to propagate those changes to Business Vault objects (such as derived Sats, PITS, or Bridges, if needed) in the next layer. This setup can be repeated to move data through all the layers in small increments very quickly and efficiently. All the way until it is ready to be accessed by data consumers (if materialization of the data is required for performance). 

Following this approach will result in a hands-off production data pipeline that feeds your Data Vault architecture.

<!-- ------------------------ -->
## Sample Data & Staging Area

Every Snowflake account provides access to [sample data sets](https://docs.snowflake.com/en/user-guide/sample-data.html). You can find corresponding schemas in SNOWFLAKE_SAMPLE_DATA database in your object explorer.
For this guide we are going to use a subset of objects from [TPC-H](https://docs.snowflake.com/en/user-guide/sample-data-tpch.html) set, representing **customers** and their **orders**. We also going to take some reference data about **nations** and **regions**. 

| Dataset | Description | Source | Load Scenario | Mechanism |
|---------|-------------|--------|---------------|-----------|
| Nation | Static ref data | snowflake.sample_data.tpch_sf10.nation | one-off CTAS | SQL |
| Region | Static ref data | snowflake.sample_data.tpch_sf10.region | one-off CTAS | SQL |
| Customer | Customer data | snowflake.sample_data.tpch_sf10.customer | incremental JSON files | Snowpipe |
| Orders | Orders data | snowflake.sample_data.tpch_sf10.orders | incremental CSV files | Snowpipe |

### Static Reference Data

Let's start with the static reference data. If you've just completed the [Defensible Analytics using Data Vault and Snowflake](https://www.snowflake.com/en/developers/guides/defensible-analytics-using-data-vault-and-snowflake/) guide, create a new SQL file in Workspaces named DVRealTime.sql.

> **Data Vault audit columns:** Every entity in this guide carries two mandatory audit columns. `ldts` (*Load Date Time Stamp*) is the immutable timestamp of when the row was loaded into the vault — set once on arrival and never changed. `rsrc` (*Record Source*) identifies the originating system or feed. Together they ensure every record is fully traceable: you can always answer *when* did this arrive and *where* did it come from, satisfying both auditability and compliance requirements.

```sql
--------------------------------------------------------------------
-- setting up staging area
--------------------------------------------------------------------

USE ROLE DEV_LZ_INGEST;
USE WAREHOUSE DEV_INGEST_WH;
USE SCHEMA DEV_LZ.TPCH;

CREATE OR REPLACE TABLE stg_nation 
AS 
SELECT src.*
     , CURRENT_TIMESTAMP()          ldts 
     , 'Static Reference Data'      rsrc 
  FROM snowflake_sample_data.tpch_sf10.nation src;

CREATE OR REPLACE TABLE stg_region
AS 
SELECT src.*
     , CURRENT_TIMESTAMP()          ldts 
     , 'Static Reference Data'      rsrc 
  FROM snowflake_sample_data.tpch_sf10.region src;
```

2. Next, let's create staging tables for our data loading. This syntax should be very familiar with anyone working with databases before. It is ANSI SQL compliant DDL, with probably one key exception - for stg_customer we are going to load the full payload of JSON into raw_json column. For this, Snowflake has a special data type [VARIANT](https://docs.snowflake.com/en/sql-reference/data-types-semistructured.html). 

As we load data we also going to add some technical metadata, like load data timestamp, row number in a file. 

```sql
CREATE OR REPLACE TABLE stg_customer
(
  raw_json                VARIANT
, filename                STRING   NOT NULL
, file_row_seq            NUMBER   NOT NULL
, ldts                    STRING   NOT NULL
, rsrc                    STRING   NOT NULL
);

CREATE OR REPLACE TABLE stg_orders
(
  o_orderkey              NUMBER
, o_custkey               NUMBER  
, o_orderstatus           STRING
, o_totalprice            NUMBER  
, o_orderdate             DATE
, o_orderpriority         STRING
, o_clerk                 STRING
, o_shippriority          NUMBER
, o_comment               STRING
, filename                STRING   NOT NULL
, file_row_seq            NUMBER   NOT NULL
, ldts                    STRING   NOT NULL
, rsrc                    STRING   NOT NULL
);
```

3. Tables we just created are going to be used by Snowpipe to drip-feed the data as it is lands in the stage. 
In order to easily detect and incrementally process the new portion of data we are going to create [streams](https://docs.snowflake.com/en/user-guide/streams.html) on these staging tables:

```sql
USE ROLE SALESMKT_ENGINEER;
USE WAREHOUSE ENGINEERING_WH;

CREATE OR REPLACE STREAM DEV_DV.SALESMKT.stg_customer_strm ON TABLE DEV_LZ.TPCH.stg_customer;

USE ROLE CUSTSERV_ENGINEER;

CREATE OR REPLACE STREAM DEV_DV.CUSTSERV.stg_orders_strm ON TABLE DEV_LZ.TPCH.stg_orders;
```

4. Next we are going to produce some sample data. And for the sake of simplicity we are going to take a bit of a shortcut here. 
We are going to generate data by unloading subset of data from our TPCH sample dataset into files and then use Snowpipe to load it back into our Data Vault lab, simulating the streaming feed. 
Let's start by creating two stages for each data class type (orders, customers data). In real-life scenarios these could be internal or external stages as well as these feeds could be sourced via Kafka connector. The world is your oyster. 

```sql
USE ROLE DEV_LZ_INGEST;
USE WAREHOUSE DEV_INGEST_WH;
USE SCHEMA DEV_LZ.TPCH;

CREATE OR REPLACE STAGE customer_data FILE_FORMAT = (TYPE = JSON);
CREATE OR REPLACE STAGE orders_data   FILE_FORMAT = (TYPE = CSV) ;
```

5. Generate and unload sample data. There are couple of things going on. 
First, we are using [object_construct](https://docs.snowflake.com/en/sql-reference/functions/object_construct.html) as a quick way to create a object/document from all columns and subset of rows for customer data and offload it into customer_data stage. Orders data would be extracted into compressed CSV files. There are many additonal options in [COPY INTO stage](https://docs.snowflake.com/en/sql-reference/sql/copy-into-location.html) construct that would fit most requirements, but in this case we are using INCLUDE_QUERY_ID to make it easier to generate new incremental files as we are going to run these commands over and over again, without a need to deal with file overriding. 

```sql

COPY INTO @customer_data 
FROM
(SELECT object_construct(*)
  FROM snowflake_sample_data.tpch_sf10.customer limit 10
) 
INCLUDE_QUERY_ID=TRUE;

COPY INTO @orders_data 
FROM
(SELECT *
  FROM snowflake_sample_data.tpch_sf10.orders limit 1000
) 
INCLUDE_QUERY_ID=TRUE;
```

You can now run the following to validate that the data is now stored in files:
```sql
list @customer_data;
SELECT METADATA$FILENAME,$1 FROM @customer_data; 
```
![staged data](assets/img10.png)

6. Next, we are going to setup Snowpipe to load data from files in a stage into staging tables. 
In this guide, for better transparency we are going to trigger Snowpipe explicitly to scan for new files, but in real projects you will likely going to enable AUTO_INGEST, connecting it with your cloud storage events (like AWS SNS) and process new files automatically. 

```sql
CREATE OR REPLACE PIPE stg_orders_pp 
AS 
COPY INTO stg_orders 
FROM
(
SELECT $1,$2,$3,$4,$5,$6,$7,$8,$9 
     , metadata$filename
     , metadata$file_row_number
     , CURRENT_TIMESTAMP()
     , 'Orders System'
  FROM @orders_data
);

CREATE OR REPLACE PIPE stg_customer_pp 
--AUTO_INGEST = TRUE
--aws_sns_topic = 'arn:aws:sns:mybucketdetails'
AS 
COPY INTO stg_customer
FROM 
(
SELECT $1
     , metadata$filename
     , metadata$file_row_number
     , CURRENT_TIMESTAMP()
     , 'Customers System'
  FROM @customer_data
);

ALTER PIPE stg_customer_pp REFRESH;

ALTER PIPE stg_orders_pp   REFRESH;
```

Once this done, you should be able to see data is appearing in the target tables and the stream on these tables.
As you would notice, number of rows in a stream is exactly the same as in the base table. This is because we didn't process/consumed the delta of that stream yet. Stay tuned! 

```sql
SELECT 'DEV_LZ.TPCH.stg_customer', count(1) FROM DEV_LZ.TPCH.stg_customer
UNION ALL
SELECT 'DEV_LZ.TPCH.stg_orders', count(1) FROM DEV_LZ.TPCH.stg_orders
UNION ALL
SELECT 'DEV_DV.CUSTSERV.stg_orders_strm', count(1) FROM DEV_DV.CUSTSERV.stg_orders_strm
UNION ALL
SELECT 'DEV_DV.SALESMKT.stg_customer_strm', count(1) FROM DEV_DV.SALESMKT.stg_customer_strm
;
```
![staged data](assets/img11.png)

7. Finally, now that we established the basics and new data is knocking at our door (stream), let's see how we can derive some of the business keys for the Data Vault entites we are going to model. In this example, we will model it as a view on top of the stream that should allow us to perform data parsing (raw_json -> columns) and business_key, hash_diff derivation on the fly.
Another thing to notice here is the use of SHA1_BINARY as hasing function. There are many articles on choosing between MD5/SHA1(2)/other hash functions, so we won't focus on this. For this lab, we are going to use fairly common SHA1 and its BINARY version from Snowflake arsenal of functions that use less bytes to encode value than STRING. 

```sql
USE ROLE SALESMKT_ENGINEER;
USE WAREHOUSE ENGINEERING_WH;
USE SCHEMA DEV_DV.SALESMKT;

CREATE OR REPLACE VIEW stg_customer_strm_outbound AS 
SELECT src.*
     , raw_json:C_CUSTKEY::NUMBER           c_custkey
     , raw_json:C_NAME::STRING              c_name
     , raw_json:C_ADDRESS::STRING           c_address
     , raw_json:C_NATIONKEY::NUMBER         C_nationcode
     , raw_json:C_PHONE::STRING             c_phone
     , raw_json:C_ACCTBAL::NUMBER           c_acctbal
     , raw_json:C_MKTSEGMENT::STRING        c_mktsegment
     , raw_json:C_COMMENT::STRING           c_comment     
--------------------------------------------------------------------
-- derived business key
--------------------------------------------------------------------
     , SHA1_BINARY(UPPER(TRIM(c_custkey)))  sha1_hub_customer     
     , SHA1_BINARY(UPPER(ARRAY_TO_STRING(ARRAY_CONSTRUCT( 
                                              NVL(TRIM(c_name)       ,'-1')
                                            , NVL(TRIM(c_address)    ,'-1')              
                                            , NVL(TRIM(c_nationcode) ,'-1')                 
                                            , NVL(TRIM(c_phone)      ,'-1')            
                                            , NVL(TRIM(c_acctbal)    ,'-1')               
                                            , NVL(TRIM(c_mktsegment) ,'-1')                 
                                            , NVL(TRIM(c_comment)    ,'-1')               
                                            ), '^')))  AS customer_hash_diff
  FROM stg_customer_strm src
;

USE ROLE CUSTSERV_ENGINEER;
USE SCHEMA DEV_DV.CUSTSERV;

CREATE OR REPLACE VIEW stg_order_strm_outbound AS 
SELECT src.*
--------------------------------------------------------------------
-- derived business key
--------------------------------------------------------------------
     , SHA1_BINARY(UPPER(TRIM(o_orderkey)))             sha1_hub_order
     , SHA1_BINARY(UPPER(TRIM(o_custkey)))              sha1_hub_customer  
     , SHA1_BINARY(UPPER(ARRAY_TO_STRING(ARRAY_CONSTRUCT( NVL(TRIM(o_orderkey)       ,'-1')
                                                        , NVL(TRIM(o_custkey)        ,'-1')
                                                        ), '^')))  AS sha1_lnk_customer_order             
     , SHA1_BINARY(UPPER(ARRAY_TO_STRING(ARRAY_CONSTRUCT( NVL(TRIM(o_orderstatus)    , '-1')         
                                                        , NVL(TRIM(o_totalprice)     , '-1')        
                                                        , NVL(TRIM(o_orderdate)      , '-1')       
                                                        , NVL(TRIM(o_orderpriority)  , '-1')           
                                                        , NVL(TRIM(o_clerk)          , '-1')    
                                                        , NVL(TRIM(o_shippriority)   , '-1')          
                                                        , NVL(TRIM(o_comment)        , '-1')      
                                                        ), '^')))  AS order_hash_diff     
  FROM DEV_DV.CUSTSERV.stg_orders_strm src
;
```
Finally let's query these views to validate the results:
![staged data](assets/img12.png)

Well done! We build our staging/inbound pipeline, ready to accommodate streaming data and derived business keys that we are going to use in our Raw Data Vault. Let's move on to the next step!


<!-- ------------------------ -->
## Build: Raw Data Vault

In this section, we will start building structures and pipelines for **Raw Data Vault** area. 

Here is the ER model of the objects we are going to deploy using the script below:
![staged data](assets/img16.png)


1. We'll start by deploying DDL for the HUBs, LINKs and SATellite tables. As you can imagine, this guide has no chance to go in the detail on data vault modelling process. This is something we usually highly recommend to establish by working with experts & partners from Data Vault Alliance.  



```sql
--------------------------------------------------------------------
-- setting up RDV
--------------------------------------------------------------------

USE ROLE SALESMKT_ENGINEER;
USE WAREHOUSE ENGINEERING_WH;
USE SCHEMA DEV_DV.SALESMKT;

-- Sales & Marketing domain (DEV_DV.SALESMKT): hub, satellite, and reference data

CREATE OR REPLACE TABLE hub_customer 
( 
  sha1_hub_customer       BINARY    NOT NULL   
, c_custkey               NUMBER    NOT NULL                                                                                 
, ldts                    TIMESTAMP NOT NULL
, rsrc                    STRING    NOT NULL
, CONSTRAINT pk_hub_customer        PRIMARY KEY(sha1_hub_customer)
);                                     

-- sat

CREATE OR REPLACE TABLE sat_customer 
( 
  sha1_hub_customer      BINARY    NOT NULL   
, ldts                   TIMESTAMP NOT NULL
, c_name                 STRING
, c_address              STRING
, c_phone                STRING 
, c_acctbal              NUMBER
, c_mktsegment           STRING    
, c_comment              STRING
, nationcode             NUMBER
, hash_diff              BINARY    NOT NULL
, rsrc                   STRING    NOT NULL  
, CONSTRAINT pk_sat_customer       PRIMARY KEY(sha1_hub_customer, ldts)
, CONSTRAINT fk_sat_customer       FOREIGN KEY(sha1_hub_customer) REFERENCES hub_customer
);                                     

-- ref data

CREATE OR REPLACE TABLE ref_region
( 
  regioncode            NUMBER 
, ldts                  TIMESTAMP
, rsrc                  STRING NOT NULL
, r_name                STRING
, r_comment             STRING
, CONSTRAINT PK_REF_REGION PRIMARY KEY (REGIONCODE)                                                                             
)
AS 
SELECT r_regionkey
     , ldts
     , rsrc
     , r_name
     , r_comment
  FROM DEV_LZ.TPCH.stg_region;

CREATE OR REPLACE TABLE ref_nation 
( 
  nationcode            NUMBER 
, regioncode            NUMBER 
, ldts                  TIMESTAMP
, rsrc                  STRING NOT NULL
, n_name                STRING
, n_comment             STRING
, CONSTRAINT pk_ref_nation PRIMARY KEY (nationcode)                                                                             
, CONSTRAINT fk_ref_region FOREIGN KEY (regioncode) REFERENCES ref_region(regioncode)  
)
AS 
SELECT n_nationkey
     , n_regionkey
     , ldts
     , rsrc
     , n_name
     , n_comment
  FROM DEV_LZ.TPCH.stg_nation;           

--------------------------------------------------------------------
-- Customer Service domain (DEV_DV.CUSTSERV): hub, satellite, and link
--------------------------------------------------------------------

USE ROLE CUSTSERV_ENGINEER;
USE SCHEMA DEV_DV.CUSTSERV;

CREATE OR REPLACE TABLE hub_order 
( 
  sha1_hub_order          BINARY    NOT NULL   
, o_orderkey              NUMBER    NOT NULL                                                                                 
, ldts                    TIMESTAMP NOT NULL
, rsrc                    STRING    NOT NULL
, CONSTRAINT pk_hub_order           PRIMARY KEY(sha1_hub_order)
);                                     

CREATE OR REPLACE TABLE sat_order 
( 
  sha1_hub_order         BINARY    NOT NULL   
, ldts                   TIMESTAMP NOT NULL
, o_orderstatus          STRING   
, o_totalprice           NUMBER
, o_orderdate            DATE
, o_orderpriority        STRING
, o_clerk                STRING    
, o_shippriority         NUMBER
, o_comment              STRING
, hash_diff              BINARY    NOT NULL
, rsrc                   STRING    NOT NULL   
, CONSTRAINT pk_sat_order PRIMARY KEY(sha1_hub_order, ldts)
, CONSTRAINT fk_sat_order FOREIGN KEY(sha1_hub_order) REFERENCES hub_order
);   

CREATE OR REPLACE TABLE lnk_customer_order
(
  sha1_lnk_customer_order BINARY     NOT NULL   
, sha1_hub_customer       BINARY 
, sha1_hub_order          BINARY 
, ldts                    TIMESTAMP  NOT NULL
, rsrc                    STRING     NOT NULL  
, CONSTRAINT pk_lnk_customer_order  PRIMARY KEY(sha1_lnk_customer_order)
, CONSTRAINT fk1_lnk_customer_order FOREIGN KEY(sha1_hub_customer) REFERENCES DEV_DV.SALESMKT.hub_customer
, CONSTRAINT fk2_lnk_customer_order FOREIGN KEY(sha1_hub_order)    REFERENCES hub_order
);
```

2. Now we have source data waiting in our staging streams & views, we have target RDV tables. 
Let's connect the dots. We are going to create tasks, one per each stream so whenever there is new records coming in a stream, that delta will be incrementally propagated to all dependent RDV models in one go. To achieve that, we are going to use multi-table insert functionality as described in design section before. As you can see, tasks can be set up to run on a pre-defined frequency (every 1 minute in our example) and use dedicated virtual warehouse as a compute power (in our guide we are going to use same warehouse for all tasks, though this could be as granular as needed). Also, before waking up a compute resource, tasks are going to check that there is data in a corresponding stream to process. Again, you are paying only for the compute when you actually use it.  

```sql
USE ROLE SALESMKT_ENGINEER;
USE SCHEMA DEV_DV.SALESMKT;

CREATE OR REPLACE TASK customer_strm_tsk
  WAREHOUSE = DEV_XFORM_WH
  SCHEDULE = '1 minute'
WHEN
  SYSTEM$STREAM_HAS_DATA('DEV_DV.SALESMKT.STG_CUSTOMER_STRM')
AS 
INSERT ALL
WHEN (SELECT COUNT(1) FROM hub_customer tgt WHERE tgt.sha1_hub_customer = src_sha1_hub_customer) = 0
THEN INTO hub_customer  
( sha1_hub_customer
, c_custkey
, ldts
, rsrc
)  
VALUES 
( src_sha1_hub_customer
, src_c_custkey
, src_ldts
, src_rsrc
)  
WHEN (SELECT COUNT(1) FROM sat_customer tgt WHERE tgt.sha1_hub_customer = src_sha1_hub_customer AND tgt.hash_diff = src_customer_hash_diff) = 0
THEN INTO sat_customer  
(
  sha1_hub_customer  
, ldts              
, c_name            
, c_address         
, c_phone           
, c_acctbal         
, c_mktsegment      
, c_comment         
, nationcode        
, hash_diff         
, rsrc              
)  
VALUES 
(
  src_sha1_hub_customer  
, src_ldts              
, src_c_name            
, src_c_address         
, src_c_phone           
, src_c_acctbal         
, src_c_mktsegment      
, src_c_comment         
, src_nationcode        
, src_customer_hash_diff         
, src_rsrc              
)
SELECT sha1_hub_customer   src_sha1_hub_customer
     , c_custkey           src_c_custkey
     , c_name              src_c_name
     , c_address           src_c_address
     , c_nationcode        src_nationcode
     , c_phone             src_c_phone
     , c_acctbal           src_c_acctbal
     , c_mktsegment        src_c_mktsegment
     , c_comment           src_c_comment    
     , customer_hash_diff  src_customer_hash_diff
     , ldts                src_ldts
     , rsrc                src_rsrc
  FROM stg_customer_strm_outbound src
;

USE ROLE CUSTSERV_ENGINEER;
USE SCHEMA DEV_DV.CUSTSERV;

CREATE OR REPLACE TASK order_strm_tsk
  WAREHOUSE = DEV_XFORM_WH
  SCHEDULE = '1 minute'
WHEN
  SYSTEM$STREAM_HAS_DATA('DEV_DV.CUSTSERV.STG_ORDERS_STRM')
AS 
INSERT ALL
WHEN (SELECT COUNT(1) FROM hub_order tgt WHERE tgt.sha1_hub_order = src_sha1_hub_order) = 0
THEN INTO hub_order  
( sha1_hub_order
, o_orderkey
, ldts
, rsrc
)  
VALUES 
( src_sha1_hub_order
, src_o_orderkey
, src_ldts
, src_rsrc
)  
WHEN (SELECT COUNT(1) FROM sat_order tgt WHERE tgt.sha1_hub_order = src_sha1_hub_order AND tgt.hash_diff = src_order_hash_diff) = 0
THEN INTO sat_order  
(
  sha1_hub_order  
, ldts              
, o_orderstatus  
, o_totalprice   
, o_orderdate    
, o_orderpriority
, o_clerk        
, o_shippriority 
, o_comment              
, hash_diff         
, rsrc              
)  
VALUES 
(
  src_sha1_hub_order  
, src_ldts              
, src_o_orderstatus  
, src_o_totalprice   
, src_o_orderdate    
, src_o_orderpriority
, src_o_clerk        
, src_o_shippriority 
, src_o_comment      
, src_order_hash_diff         
, src_rsrc              
)
WHEN (SELECT COUNT(1) FROM lnk_customer_order tgt WHERE tgt.sha1_lnk_customer_order = src_sha1_lnk_customer_order) = 0
THEN INTO lnk_customer_order  
(
  sha1_lnk_customer_order  
, sha1_hub_customer              
, sha1_hub_order  
, ldts
, rsrc              
)  
VALUES 
(
  src_sha1_lnk_customer_order
, src_sha1_hub_customer
, src_sha1_hub_order  
, src_ldts              
, src_rsrc              
)
SELECT sha1_hub_order          src_sha1_hub_order
     , sha1_lnk_customer_order src_sha1_lnk_customer_order
     , sha1_hub_customer       src_sha1_hub_customer
     , o_orderkey              src_o_orderkey
     , o_orderstatus           src_o_orderstatus  
     , o_totalprice            src_o_totalprice   
     , o_orderdate             src_o_orderdate    
     , o_orderpriority         src_o_orderpriority
     , o_clerk                 src_o_clerk        
     , o_shippriority          src_o_shippriority 
     , o_comment               src_o_comment      
     , order_hash_diff         src_order_hash_diff
     , ldts                    src_ldts
     , rsrc                    src_rsrc
  FROM stg_order_strm_outbound src;    

ALTER TASK DEV_DV.SALESMKT.customer_strm_tsk RESUME;  
ALTER TASK DEV_DV.CUSTSERV.order_strm_tsk    RESUME;  

```

2. Once tasks are created and RESUMED (by default, they are initially suspended) let's have a look on the task execution history to see how the process will start. 

```sql
SELECT *
  FROM table(information_schema.task_history())
  ORDER BY scheduled_time DESC;
```
Notice how after successfull execution, next two tasks run were automatically SKIPPED as there were nothing in the stream and there nothing to do. 

![staged data](assets/img13.png)

3. We can also check content and stats of the objects involved. Please notice that views on streams in our staging area are no longer returning any rows. This is because that delta of changes was consumed by a successfully completed DML transaction (in our case, embedded in tasks). This way you don't need to spend any time implementing incremental detection/processing logic on the application side. 

```sql
SELECT 'hub_customer', count(1) FROM DEV_DV.SALESMKT.hub_customer
UNION ALL
SELECT 'hub_order', count(1) FROM DEV_DV.CUSTSERV.hub_order
UNION ALL
SELECT 'sat_customer', count(1) FROM DEV_DV.SALESMKT.sat_customer
UNION ALL
SELECT 'sat_order', count(1) FROM DEV_DV.CUSTSERV.sat_order
UNION ALL
SELECT 'lnk_customer_order', count(1) FROM DEV_DV.CUSTSERV.lnk_customer_order
UNION ALL
SELECT 'stg_customer_strm_outbound', count(1) FROM DEV_DV.SALESMKT.stg_customer_strm_outbound
UNION ALL
SELECT 'stg_order_strm_outbound', count(1) FROM DEV_DV.CUSTSERV.stg_order_strm_outbound;
```
![staged data](assets/img14.png)

Great. We now have data in our **Raw Data Vault** core structures. Let's move on and talk about the concept of virtualization for building your near-real time Data Vault solution.

<!-- ------------------------ -->
## Views for Agile Reporting

One of the great benefits of having the compute power from Snowflake is that now it is totally possible to have most of your business vault and information marts in a Data Vault architecture be built exclusively from views. There are numerous customers using this approach in production today. There is no longer a need to have the argument that there are “too many joins” or that the response won’t be fast enough. The elasticity of the Snowflake virtual warehouses combined with our dynamic optimization engine have solved that problem. (For more details, see this [post](/blog/tips-for-optimizing-the-data-vault-architecture-on-snowflake-part-3/))

If you really want to deliver data to the business users and data scientists in NRT, in our opinion using views is the only option. Once you have the streaming loads built to feed your Data Vault, the fastest way to make that data visible downstream will be views. Using views allows you to deliver the data faster by eliminating any latency that would be incurred by having additional ELT processes between the Data Vault and the data consumers downstream.

All the business logic, alignment, and formatting of the data can be in the view code. That means fewer moving parts to debug, and reduces the storage needed as well.


![Data Vault Virtualization Architecture](assets/img3.png)  

Looking at the diagram above you will see an example of how virtualization could fit in the architecture. Here, solid lines are representing physical tables and dotted lines - views. You incrementally ingest data into **Raw Data Vault** and all downstream transformations are applied as views. From a data consumer perspective when working with a virtualized information mart, the query always shows everything known by your data vault, right up to the point the query was submitted. 

With Snowflake you have the ability to provide as much compute as required, on-demand,  without a risk of causing performance impact on any surrounding processes and pay only for what you use. This makes materialization of transformations in layers like **Business Data Vault** and **Information delivery** an option rather than a must-have. Instead of “optimizing upfront” you can now make this decision based on the usage pattern characteristics, such as frequency of use, type of queries, latency requirements, readiness of the requirements etc. 


Many modern data engineering automation frameworks are already actively supporting virtualization of logic. Several tools offer a low-code or configuration-like ability to switch between materializing an object as a view or a physical table, automatically generating all required DDL & DML. This could be applied on specific objects, layers or/and be environment specific. So even if you start with a view, you can easily refactor to use a table if user requirements evolve.

As said before, virtualization is not only a way to improve time-to-value and provide near real time access to the data, given the scalability and workload isolation of Snowflake, virtualization also is a design technique that could make your Data Vault excel: minimizing cost-of-change, accelerating the time-to-delivery and becoming an extremely agile, future proof solution for ever growing business needs. 

<!-- ------------------------ -->
## Build: Business Data Vault

As a quick example of using views for transformations we just discussed, here is how enrichment of customer descriptive data could happen in Business Data Vault, connecting data received from source with some reference data.

1. Let's create a view that will perform these additional derivations on the fly. Assuming non-functional capabilities are satisfying our requirements, deploying (and re-deploying a new version) transformations in this way is super easy. 

```SQL
USE ROLE SALESMKT_ENGINEER;
USE WAREHOUSE ENGINEERING_WH;
USE SCHEMA DEV_DV.SALESMKT;

CREATE OR REPLACE VIEW sat_customer_bv
AS
SELECT rsc.sha1_hub_customer  
     , rsc.ldts                   
     , rsc.c_name                 
     , rsc.c_address              
     , rsc.c_phone                 
     , rsc.c_acctbal              
     , rsc.c_mktsegment               
     , rsc.c_comment              
     , rsc.nationcode             
     , rsc.rsrc 
     -- derived 
     , rrn.n_name                    nation_name
     , rrr.r_name                    region_name
  FROM sat_customer          rsc
  LEFT OUTER JOIN ref_nation rrn
    ON (rsc.nationcode = rrn.nationcode)
  LEFT OUTER JOIN ref_region rrr
    ON (rrn.regioncode = rrr.regioncode)
;
```
2. Now,let's imagine we have a heavier transformation to perform that it would make more sense to materialize it as a table. It could be more data volume, could be more complex logic, PITs, bridges or even an object that will be used frequently and by many users. For this case, let's first build a new business satellite that for illustration purposes will be deriving additional classification/tiering for orders based on the conditional logic. 

```sql
USE ROLE CUSTSERV_ENGINEER;
USE WAREHOUSE ENGINEERING_WH;
USE SCHEMA DEV_DV.CUSTSERV;

CREATE OR REPLACE TABLE sat_order_bv
( 
  sha1_hub_order         BINARY    NOT NULL   
, ldts                   TIMESTAMP NOT NULL
, o_orderstatus          STRING   
, o_totalprice           NUMBER
, o_orderdate            DATE
, o_orderpriority        STRING
, o_clerk                STRING    
, o_shippriority         NUMBER
, o_comment              STRING  
, hash_diff              BINARY    NOT NULL
, rsrc                   STRING    NOT NULL   
-- additional attributes
, order_priority_bucket  STRING
, CONSTRAINT pk_sat_order PRIMARY KEY(sha1_hub_order, ldts)
, CONSTRAINT fk_sat_order FOREIGN KEY(sha1_hub_order) REFERENCES hub_order
)
AS 
SELECT sha1_hub_order 
     , ldts           
     , o_orderstatus  
     , o_totalprice   
     , o_orderdate    
     , o_orderpriority
     , o_clerk        
     , o_shippriority 
     , o_comment      
     , hash_diff      
     , rsrc 
     -- derived additional attributes
     , CASE WHEN o_orderpriority IN ('2-HIGH', '1-URGENT')             AND o_totalprice >= 200000 THEN 'Tier-1'
            WHEN o_orderpriority IN ('3-MEDIUM', '2-HIGH', '1-URGENT') AND o_totalprice BETWEEN 150000 AND 200000 THEN 'Tier-2'  
            ELSE 'Tier-3'
       END order_priority_bucket
  FROM sat_order;
```

3. What we are going to do from processing/orchestration perspective is extending our order processing pipeline so that when the task populates **DEV_DV.CUSTSERV.sat_order** this will generate a new stream of changes and these changes are going to be propagated by a dependent task to **DEV_DV.CUSTSERV.sat_order_bv**.
This is super easy to do as tasks in Snowflake can be not only schedule-based but also start automatically once the parent task is completed.

```sql
USE ROLE CUSTSERV_ENGINEER;
USE WAREHOUSE ENGINEERING_WH;
USE SCHEMA DEV_DV.CUSTSERV;

CREATE OR REPLACE STREAM sat_order_strm ON TABLE sat_order;

ALTER TASK order_strm_tsk SUSPEND;

CREATE OR REPLACE TASK hub_order_strm_sat_order_bv_tsk
  WAREHOUSE = DEV_XFORM_WH
  AFTER order_strm_tsk
AS 
INSERT INTO sat_order_bv
SELECT   
  sha1_hub_order 
, ldts           
, o_orderstatus  
, o_totalprice   
, o_orderdate    
, o_orderpriority
, o_clerk        
, o_shippriority 
, o_comment      
, hash_diff      
, rsrc 
-- derived additional attributes
, CASE WHEN o_orderpriority IN ('2-HIGH', '1-URGENT')             AND o_totalprice >= 200000 THEN 'Tier-1'
       WHEN o_orderpriority IN ('3-MEDIUM', '2-HIGH', '1-URGENT') AND o_totalprice BETWEEN 150000 AND 200000 THEN 'Tier-2'  
       ELSE 'Tier-3'
  END order_priority_bucket
FROM sat_order_strm;

ALTER TASK hub_order_strm_sat_order_bv_tsk RESUME;
ALTER TASK order_strm_tsk RESUME;
```

4. Now, let's go back to our staging area to process another slice of data to test the task. 

```sql
USE ROLE DEV_LZ_INGEST;
USE WAREHOUSE DEV_INGEST_WH;
USE SCHEMA DEV_LZ.TPCH;

COPY INTO @orders_data 
FROM
(SELECT *
  FROM snowflake_sample_data.tpch_sf10.orders limit 1000
) 
INCLUDE_QUERY_ID=TRUE;

ALTER PIPE stg_orders_pp   REFRESH;
```

5. Data is now automatically flowing through all the layers via asynchronous tasks. With the results you can validate: 

```sql
SELECT 'DEV_LZ.TPCH.stg_orders', count(1) FROM DEV_LZ.TPCH.stg_orders
UNION ALl
SELECT 'DEV_DV.CUSTSERV.stg_orders_strm', count(1) FROM DEV_DV.CUSTSERV.stg_orders_strm
UNION ALl
SELECT 'DEV_DV.CUSTSERV.sat_order', count(1) FROM DEV_DV.CUSTSERV.sat_order
UNION ALl
SELECT 'DEV_DV.CUSTSERV.sat_order_strm', count(1) FROM DEV_DV.CUSTSERV.sat_order_strm
UNION ALL
SELECT 'DEV_DV.CUSTSERV.sat_order_bv', count(1) FROM DEV_DV.CUSTSERV.sat_order_bv;
```

![staged data](assets/img15.png)

Great. Hope this example illustrated few ways of managing **Business Data Vault** objects in our pipeline.
Let's finally move into the **Information Delivery** layer. 

<!-- ------------------------ -->
## Build: Information Delivery

When it comes to Information Delivery area we are not changing the meaning of data, but we may change format to simplify users to access and work with the data products/output interfaces. Different consumers may have different needs and preferences, some would prefer star/snowflake dimensional schemas, some would adhere to use flattened objects or even transform data into JSON/parquet objects. 

1. First things we would like to add to simplify working with satellites is creating views that shows latest version for each key. 

```sql
--------------------------------------------------------------------
-- Sales & Marketing curr views (DEV_DV.SALESMKT)
--------------------------------------------------------------------

USE ROLE SALESMKT_ENGINEER;
USE WAREHOUSE ENGINEERING_WH;
USE SCHEMA DEV_DV.SALESMKT;

CREATE VIEW sat_customer_curr_vw
AS
SELECT  *
FROM    sat_customer
QUALIFY LEAD(ldts) OVER (PARTITION BY sha1_hub_customer ORDER BY ldts) IS NULL;

CREATE VIEW sat_customer_bv_curr_vw
AS
SELECT  *
FROM    sat_customer_bv
QUALIFY LEAD(ldts) OVER (PARTITION BY sha1_hub_customer ORDER BY ldts) IS NULL;

--------------------------------------------------------------------
-- Customer Service curr views (DEV_DV.CUSTSERV)
--------------------------------------------------------------------

USE ROLE CUSTSERV_ENGINEER;
USE SCHEMA DEV_DV.CUSTSERV;

CREATE OR REPLACE VIEW sat_order_curr_vw
AS
SELECT  *
FROM    sat_order
QUALIFY LEAD(ldts) OVER (PARTITION BY sha1_hub_order ORDER BY ldts) IS NULL;

CREATE VIEW sat_order_bv_curr_vw
AS
SELECT  *
FROM    sat_order_bv
QUALIFY LEAD(ldts) OVER (PARTITION BY sha1_hub_order ORDER BY ldts) IS NULL;
```

2. Let's create a simple dimensional structure. Again, we will keep it virtual(as views) to start with, but you already know that depending on access characteristics required any of these could be selectively materialized. 

```sql
USE ROLE SALESMKT_ENGINEER;
USE WAREHOUSE ENGINEERING_WH;
USE SCHEMA DEV_DW.SALESMKT;

-- DIM TYPE 1
CREATE OR REPLACE VIEW dim1_customer 
AS 
SELECT hub.sha1_hub_customer                      AS dim_customer_key
     , sat.ldts                                   AS effective_dts
     , hub.c_custkey                              AS customer_id
     , sat.rsrc                                   AS record_source
     , sat.*     
  FROM DEV_DV.SALESMKT.hub_customer                       hub
     , DEV_DV.SALESMKT.sat_customer_bv_curr_vw            sat
 WHERE hub.sha1_hub_customer                      = sat.sha1_hub_customer;

USE ROLE CUSTSERV_ENGINEER;
USE SCHEMA DEV_DW.CUSTSERV;

-- DIM TYPE 1
CREATE OR REPLACE VIEW dim1_order
AS 
SELECT hub.sha1_hub_order                         AS dim_order_key
     , sat.ldts                                   AS effective_dts
     , hub.o_orderkey                             AS order_id
     , sat.rsrc                                   AS record_source
     , sat.*
  FROM DEV_DV.CUSTSERV.hub_order                          hub
     , DEV_DV.CUSTSERV.sat_order_bv_curr_vw               sat
 WHERE hub.sha1_hub_order                         = sat.sha1_hub_order;

-- FACT table

CREATE OR REPLACE VIEW fct_customer_order
AS
SELECT lnk.ldts                                   AS effective_dts     
     , lnk.rsrc                                   AS record_source
     , lnk.sha1_hub_customer                      AS dim_customer_key
     , lnk.sha1_hub_order                         AS dim_order_key
-- this is a factless fact, but here you can add any measures, calculated or derived
  FROM DEV_DV.CUSTSERV.lnk_customer_order                 lnk;  
```

3. All good so far? Now lets try to query **fct_customer_order** and at least in my case this view was not returning any rows.
Why? If you remember, when we were unloading sample data, we took a subset of random orders and a subset of random customers. Which in my case didn't have any overlap, therefore doing the inner join with dim1_order was resulting in all rows being eliminated from the resultset. 
Thankfully we are using Data Vault and all need to do is go and load full customer dataset. Just think about it, there is no need to reprocess any links or fact tables simply because customer/reference feed was incomplete. I am sure for those of you who were using different architectures for data engineering and warehousing have painful experience when such situations occur. So, lets go and fix it:

```sql
USE ROLE DEV_LZ_INGEST;
USE WAREHOUSE DEV_INGEST_WH;
USE SCHEMA DEV_LZ.TPCH;

COPY INTO @customer_data 
FROM
(SELECT object_construct(*)
  FROM snowflake_sample_data.tpch_sf10.customer
  -- removed LIMIT 10
) 
INCLUDE_QUERY_ID=TRUE;

ALTER PIPE stg_customer_pp   REFRESH;
```

All you need to do now is just wait a few seconds whilst our continuous data pipeline will automatically propagate new customer data into Raw Data Vault.
Quick check for the records count in customer dimension now shows that there are 1.5Mn records: 

```sql
USE SCHEMA DEV_DW.SALESMKT;

SELECT COUNT(1) FROM dim1_customer;


COUNT(1)
------
1,500,000
```

4. Finally lets wear user's hat and run a query to break down orders by nation,region and order_priority_bucket (all attributes we derived in **Business Data Vault**). As we are using Snowsight, why not quickly creating a chart from this result set to better understand the data. For this simply click on the 'Chart' section on the bottom pane and put attributes/measures as it is snown on the screenshot below. 

```sql
USE SCHEMA DEV_DW.CUSTSERV;

SELECT dc.nation_name
     , dc.region_name
     , do.order_priority_bucket
     , COUNT(1)                                   cnt_orders
  FROM fct_customer_order                         fct
     , DEV_DW.SALESMKT.dim1_customer              dc
     , dim1_order                                 do
 WHERE fct.dim_customer_key                       = dc.dim_customer_key
   AND fct.dim_order_key                          = do.dim_order_key
GROUP BY 1,2,3;
```
![staged data](assets/img17.png)

Voila! This concludes our journey for this guide. Hope you enjoyed it and lets summarise key points in the next section.

<!-- ------------------------ -->
## Conclusion

Simplicity of engineering, openness, scalable performance, enterprise-grade governance enabled by the core of the Snowflake platform are now allowing teams to focus on what matters most for the business and build truly agile, collaborative data environments. Teams can now connect data from all parts of the landscape, until there are no stones left unturned. They are even tapping into new datasets via live access to the Snowflake Marketplace. The Snowflake Data Cloud combined with a Data Vault 2.1 approach is allowing teams to democratize access to all their data assets at any scale. We can now easily derive more and more value through insights and intelligence, day after day, bringing businesses to the next level of being truly data-driven.  

Delivering more usable data faster is no longer an option for today’s business environment. Using the Snowflake platform, combined with the Data Vault 2.1 architecture it is now possible to build a world class analytics platform that delivers data for all users in near real-time. 

### What we've covered
- unloading and loading back data using COPY and Snowpipe
- engineering data pipelines using virtualization, streams and tasks
- building multi-layer Data Vault environment on Snowflake: 

![Multi-Layer Data Vault Environment](assets/img19.png)  

### Call to action
- seeing is believing. Try it! 
- we made examples limited in size, but feel free to scale the data volumes and virtual warehouse size to see scalability in action
- tap into numerous communities of practice for Data Engineering on Snowflake and Data Vault in particular
- talk to us about modernizing your data landscape! Whether it is Data Vault or not you have on your mind, we have the top expertise and product to meet your demand
- feedback is super welcome! 
- Enjoy your journey! 
