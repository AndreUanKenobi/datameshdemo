# Data Mesh Demo, Google Cloud UKI Developer Day 2022

## Prep work

- All GCS files should be avialable here `gs://devday2022/'
- A BigTable instance `timeseries` should be spun up and ready
- modify the cloud big table config file: `nano .cbtrc`
- add/update/check these two properties are set:
```
project = andreuankenobi-342014
instance = timeseries
```
- Create BigTable table and column family
```
cbt createtable timeseries families="cell_data"
```
- Import data into BigTable
```
cbt import timeseries bigtable_import.csv column-family=cell_data
```
- Verify the dataload is ok
```
cbt read timeseries
```


## Section 1: BigTable federation
- Read the content of BigTable 
```
cbt read timeseries
```
- To federate BT to BigQuery, we'll have to specify a JSON file `tabledef.json`:
```
{
    "sourceFormat": "BIGTABLE",
    "sourceUris": [
     https://googleapis.com/bigtable/projects/andreuankenobi-342014/instances/timeseries/tables/timeseries
    ],
    "bigtableOptions": {
        "readRowkeyAsString": "true",
        "columnFamilies" : [
            {
                "familyId": "cell_data",
                "onlyReadLatest": "true",
                "columns": [
              {
                  "qualifierString": "attr1",
                  "type": "STRING"
              },
              {
                  "qualifierString": "attr2",
                  "type": "STRING"
              }
              ,
                  {
                  "qualifierString": "attr3",
                  "type": "STRING"
              }
              ,
                  {
                  "qualifierString": "attr4",
                  "type": "STRING"
              }
              ,
              {
                  "qualifierString": "attr5",
                  "type": "STRING"
              }
              ,
              {
                  "qualifierString": "attr6",
                  "type": "STRING"
              }
              ,
              {
                  "qualifierString": "attr7",
                  "type": "STRING"
              }
              ,
              {
                  "qualifierString": "attr8",
                  "type": "STRING"
              }
              ,
              {
                  "qualifierString": "attr9",
                  "type": "STRING"
              }
              ,
              {
                  "qualifierString": "attr10",
                  "type": "STRING"
              }
              ,
               {
                  "qualifierString": "country_code",
                  "type": "STRING"
              },
               {
                  "qualifierString": "cust_identifier",
                  "type": "STRING"
              },
              {
                  "qualifierString": "tstamp",
                  "type": "STRING"
              }
          ]

            }
        ]
    }
    
}
```
- to federate the table in BigQuery, run the following command: 
```
bq mk --external_table_definition=tabledef.json ukidev2021.telemetry_federated
```
- Showcase the table schema - NESTED
- Create a polished query on the newly federated table:
```
SELECT timestamp(cell_data.tstamp.cell.value) ts,
cast(cell_data.attr1.cell.value as NUMERIC) attr1,
cast(cell_data.attr2.cell.value as NUMERIC) attr2,
cast(cell_data.attr3.cell.value as NUMERIC) attr3,
cast(cell_data.attr4.cell.value as NUMERIC) attr4,
cast(cell_data.attr5.cell.value as NUMERIC) attr5,
cast(cell_data.attr6.cell.value as NUMERIC) attr6,
cast(cell_data.attr7.cell.value as NUMERIC) attr7,
cast(cell_data.attr8.cell.value as NUMERIC) attr8,
cast(cell_data.attr9.cell.value as NUMERIC) attr9,
cast(cell_data.attr10.cell.value as NUMERIC) attr10,
cell_data.country_code.cell.value country_code
FROM `andreuankenobi-342014.ukidev2021.telemetry_federated`
```
- Create an authorized view called `telemetry` under the dataset `reporting` using the above query.
- Create an hourly aggregated, normalized view of this data running the following query:
```
SELECT
  TIMESTAMP_TRUNC(ts, HOUR) ts,
  SUM(attr1) attr1,
  SUM(attr2) attr2,
  SUM(attr3) attr3,
  SUM(attr4) attr4,
  SUM(attr5) attr5,
  SUM(attr6) attr6,
  SUM(attr7) attr7,
  SUM(attr8) attr8,
  SUM(attr9) attr9,
  SUM(attr10) attr10
FROM
  `andreuankenobi-342014.reporting.telemetry`
GROUP BY
  TIMESTAMP_TRUNC(ts, HOUR)
```
- Save this query as `hourly_telemetry` table in the `exchange` dataset


## Step 2: Import table using the Data Transfer Service
- Create dataset `devday2022`
- Import the public table `bigquery-public-data.github_repos.sample_contents`
- On the left-hand menu, select "Data Transfers" and create a new job
  - Source: Dataset Copy
  - Repeats: on demand
  - Destination Dataset: `devday2022`
  - Source Dataset: `github_repos`
  - Source Project `bigquery-public-data`
  - Mark "overwrite destination table", just in case... 
  - Save and run the transfer

## Step 3: Create federated table in BigQuery from multiple AVRO files
- Create federated table `federated-table`, pointed to the GCS bucket with the .avro files (`gs://devday2022/clean-data`)
- Test the query federation:
```
SELECT * FROM `<yourproject>.devday2022.federated-view` LIMIT 10
```

## Step 4: Create Authorized View - a Data Product
Time for some data engineering!

- First: Will create the first temporary table from the Data Transfer defined as: 
```
create or replace table `<yourproject>.devday2022.eng_contents`
as

SELECT
 CAST(REGEXP_EXTRACT(content, r'stackoverflow.com/questions/([0-9]+)/') AS INT64) id,
 sample_path
FROM
 `<yourproject>.devday2022.sample_contents`
WHERE
   id is not null and
 content LIKE '%stackoverflow.com/questions/%'
```
- Second: we'll create another temp table from the federated view, defined as: 
```
create or replace table `<yourproject>.devday2022.post_questions_base_table`
cluster by tags
as
 (

with a as(
select id, title, answer_count,favorite_count,view_count,score, creation_date, split(tags,'|') tags_array
from `<yourproject>.devday2022.federated-view`
 
)
select * except (tags_array) from a, unnest(tags_array) as tags
where id is not null
)

```
- Create a new dataset called `data-products` 
- Create a new query as follows: 
```
SELECT a.id,title, answer_count answers, favorite_count favs, view_count views, score, count(a.id) files, min(sample_path) sample_path
FROM `<yourproject>.devday2022.post_questions_base_table` a
join `<yourproject>.devday2022.eng_contents` b
on a.id=b.id
WHERE tags='python'
group by 1,2,3,4,5,6
```
And save it as "My First Data Product" 


## Step 5: Import consumer data
- Create native table `client`, pointed to the GCS bucket with the .avro files (`gs://devday2022/clean-data`)
  -  make sure to use the file "client"
  -  file format is CSV
- Test the native table:
```
SELECT * FROM `<yourproject>.devday2022.client` LIMIT 10
```

## Step 6: Open Data Catalog
- under search, look for SSN. Oh-oh! our newly imported table has clear SSNs! 
- looks like many other fields might be impacted, so let's create a policy tag and then scan with DLP
- create a new policy tag
  - create a new taxonomy
  - name `sensitive`
  - Tag Name `PII`
  - Click on the policy tag, show the principals

## Step 7: Scan the "Client" table with DLP
- In Data Catalog, sreach again for SSN. Find the table, and specify "scan with DLP" 
- Create a DLP scan for the newly imported table
- Notable settings: 
  - Name: `PIIScan`
  - Detection: set Confidence Thresold to "Unspecified"
  - Actions: Write to BigQuery, project `<yourproject>`, dataset `devday2022`, table `dlpscan`
  - Actions: Notify via email
  - Actions: Publish to Data Catalog
  - Schedule: none
  - Confirm job and create
 - View results in BigQuery, by running this query: 
```
SELECT  info_type.name,
 likelihood,
 COUNT(info_type.name) matches FROM `<yourproject>.devday2022.dlpscan` 
 GROUP BY
 info_type.name,
 likelihood
```
  
## Step 8: Enforce Column-level security
- In Data Catalog, scearh for tag template "Data Loss Prevention Tags"
- The "Client" table will pop up. Preview it in the preview panel to confirm SSN is there 
- Click on the table, and confirm the SSN tag has been fired
- Confirm how there's not policy tag attached to the column 
- Scroll up, and select "Open in BigQuery"
- Under Schema, click "Edit Schema" - on the bottom
- Select SSN, select "Add Policy Tag"
- The table is now secured! 

## Step 9: Mesh it up!
Reference: [Build a data mesh](https://cloud.google.com/dataplex/docs/build-a-data-mesh)

Referece user `tadej@2023528041521.altostrat.com` / `TadejRule5!@`
- Create "My first lake" 
- Create the "Raw" Data zone 
- Add Assets:
  - BigQuery tables
  - Storage Buckets (mind the region!)
  - Grant Tadej view role on Dataplex resources. 
