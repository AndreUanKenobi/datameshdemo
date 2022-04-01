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


## Section 1: BigTable federation / Analytics Hub
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
- Last: publish this dataset into the Analytics Hub

## Section 2: GCS federation / Dataplex / Data Catalog
- Create a new table - `client` - federated from gcs under the ukidev2021 dataset
- Query the newly native table:
```
SELECT * FROM `andreuankenobi-342014.ukidev2021.client`
```
- Moving over to Data Catalog, create a new policy tag PII with Fine-grained IAM role required
- In data catalog, go to search and find our original table
- Explain Policy Tags require a schema manipulation - go to BigQuery
- Apply Policy Tag and show that won't work anymore
```
SELECT * FROM `andreuankenobi-342014.ukidev2021.client`
```
- Show that the policies are sticky: 
```
create or replace view `andreuankenobi-342014.reporting.client` as
SELECT * FROM `andreuankenobi-342014.ukidev2021.client`
```
this view can't be queried as it is
- If we use except to filter out PIIs though: 
```
create or replace view `andreuankenobi-342014.reporting.client_clean` as
SELECT * except(ssn,first_name, last_name) FROM `andreuankenobi-342014.ukidev2021.client`
```


- Create an authorized view `client_clean` on reporting dataset as: 
```
SELECT * except(ssn,first_name, last_name) FROM `andreuankenobi-342014.ukidev2021.client`
```
- Save a new table `client` in the exchange dataset as:
```
SELECT * FROM `andreuankenobi-342014.reporting.client_clean` 
```
- Create an authorized view under Exchange as: 
```
SELECT * FROM `andreuankenobi-342014.reporting.client_clean`
```



## Step 3: Mesh it up!
Reference: [Build a data mesh](https://cloud.google.com/dataplex/docs/build-a-data-mesh
