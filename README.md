# Data Mesh Demo, 2022

## Prep work

- BigTable files [available here](https://drive.google.com/drive/folders/1QYgKLE-cEPFTEptxPj8Tv6FHTdEN9gEu?usp=sharing), in this repo, and at `gs://meshthedatabucket/bigtable`
- Client demo data [available here](https://drive.google.com/drive/folders/1QYgKLE-cEPFTEptxPj8Tv6FHTdEN9gEu?usp=sharing), in this repo, and at `gs://meshthedatabucket/consumer_data` 
- A BigTable instance `customer-telemetry` should be spun up and ready, region US-Central-1
- Import data into BigTable
    - open cloudshell
    - modify the cloud big table config file: `nano .cbtrc`
    - add/update/check these two properties are set:
        ```
        project = meshthedata
        instance = customer-telemetry
        ```
    - Create BigTable table and column family
        ```
        cbt createtable customer-telemetry families="cell_data"
        ```
    - Upload `bigtable_import.csv` and `tabledef.json` into Cloud Shell
    - Import data into BigTable
        ```
        cbt import customer-telemetry bigtable_import.csv column-family=cell_data
        ```
        the import will upload 1000+ rows
    - Verify the dataload is ok
        ```
        cbt read customer-telemetry
        ```


## Section 1: BigTable federation / Analytics Hub
- Let's roleplay we are responsible to drive data-driven insights for the Consumer division of the company. 
- Read the content of BigTable 
```
cbt read customer-telemetry
```
This table gives us information about what the customers are doing

- Go to BigQuery, and create three new datasets, US multi regional
    - `landing`
    - `reporting`
    - `exchange`   

- To federate the table in BigQuery as `customer-telemetry-federated`, run the following command: 
```
bq mk --external_table_definition=tabledef.json landing.customer-telemetry-federated
```
- The federation process requires to specify a JSON file `tabledef.json` which we previously uploaded to cloudshell - the structure is the following (mind the URI!):
```
{
    "sourceFormat": "BIGTABLE",
    "sourceUris": [
     https://googleapis.com/bigtable/projects/meshthedata/instances/customer-telemetry/tables/customer-telemetry
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
- Close Cloud Shell, we won't need it anymore! 
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
cast(cell_data.attr10.cell.value as NUMERIC) attr10
FROM `meshthedata.landing.customer-telemetry-federated`
```
- Create an authorized view called `customer-telemetry` under the dataset `reporting` using the above query.
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
  `meshthedata.reporting.customer-telemetry`
GROUP BY
  TIMESTAMP_TRUNC(ts, HOUR)
```
- Save this query as `customer-hourly-telemetry` table in the `exchange` dataset
- Last: publish this dataset into the Analytics Hub
    - Create the `Google Exchange` exchange in the US region
    - Create the `Consumer Data` listing, pointing to the `exchange` dataset
    - Last - add the dataset to another project `andreuankenobi-public`, linked dataset name `consumer_data`  
- Hit save, and show how the two datasets are now in sync

### Adding an ARIMA model
```
CREATE OR REPLACE MODEL exchange.arima_model
OPTIONS
  (model_type = 'ARIMA_PLUS',
   time_series_timestamp_col = 'ts',
   time_series_data_col = 'attr1',
   auto_arima = TRUE,
   data_frequency = 'AUTO_FREQUENCY',
   decompose_time_series = TRUE
  ) AS
SELECT
  ts, attr1
FROM
  `meshthedata.exchange.customer-hourly-telemetry`
GROUP BY date
```


```
SELECT
 *
FROM
 ML.EXPLAIN_FORECAST(MODEL exchange.arima_model,
             STRUCT(30 AS horizon, 0.8 AS confidence_level))
where time_series_type = 'forecast'; 
```

```
SELECT
  *
FROM
  ML.DETECT_ANOMALIES(MODEL exchange.arima_model,
                      STRUCT(0.8 AS anomaly_prob_threshold))

                      where is_anomaly is true
                      
  ```

## Section 2: GCS federation /  Data Catalog
- Create a new native table - `client` - from GCS, under the `landing` dataset
- Preview this table and show how there's definitely PII data involved
- [Optional] showcase DLP 
    -  from the table page, click Export > Scan with DLP
    -  in "Choose input data", name the scan `ClientPIIScan`
    -  in "Configure detection", mention how custom templates and rulesets could be potentially configured, and set the likelyhood to "Unspecified"
    -  in "Actions", check "Publish to Data Catalog" and uncheck "Notify by email"
    -  Continue and save; the job should complete in a couple of minutes. Feel free to show a previous job and the findings. 
- Move over to Data Catalog, create a new policy tag taxonomy `PII`, with tags `PIIfield` and Fine-grained IAM role 
- In data catalog, search for the original table by searching "ssn" in the search page
- Explain the difference between a business tag and a policy tag (add tag), and explain how in the schema is shown how the sensitive fields have no policy tag attached
- Click on "Open with BigQuery"
- Edit the schema, and assign the Policy Tag
- Apply Policy Tag and show that won't work anymore
    - from preview
    - by running 
    ```
    SELECT * FROM `meshthedata.landing.client`
    ```
- Show that the policies are sticky: 
```
create or replace view `meshthedata.reporting.client` as
SELECT * FROM `meshthedata.landing.client`
```
this view can't be queried as it is
- If we use except to filter out PIIs though: 
```
create or replace view `meshthedata.reporting.client` as
SELECT * except(ssn,first_name, last_name) FROM `meshthedata.landing.client`
```

- Save a new view `client-profiles` in the exchange dataset as:
```
SELECT * except (gender, street, address) FROM `meshthedata.reporting.client` 
```

## Step 3: Mesh it up!
Reference: [Build a data mesh](https://cloud.google.com/dataplex/docs/build-a-data-mesh
- Create/verify the lake "Consumer Data" exists; use no metastore for the moment; creating the data lake will take a while, so get it ready before the demo!
- Create a "Consumer Raw Data" Raw zone, 
    -  Mind to us US multiregional!
    -  Add the assets: landing BigQuery Datasets, "meshthedatabucket" bucket
- Create a "Consumer Prepared Data" Raw zone, 
    -  Mind to us US multiregional!
    -  Add the assets: landing BigQuery Datasets
- Showcase Security, Discovery, Explore (briefly)
- For process, mention [this article](https://cloud.google.com/dataplex/docs/check-data-quality) for data quality checks from Dataplex.
