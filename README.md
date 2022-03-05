# Data Mesh Demo, Google Cloud UKI Developer Day 2022

## Step 1: load/verify files availability in a GCS bucket
Avro files for federated source stored under [gs://devday2022/clean-data](gs://devday2022/clean-data)
Consumer data files for native source stored under [gs://data_files_bq/demo-complete](gs://data_files_bq/demo-complete)

or 

available via [Google Drive](https://drive.google.com/drive/folders/12jt0rnwlYqknP30ALAk9PFHbXZCRxLcE?usp=sharing)


## Step 2: Import table using the Data Transfer Service
- Create dataset `devday2022`
- We will import the public table `bigquery-public-data.github_repos.sample_contents`
- If not enabled, enable the BigQuery Data Transfer API when prompted to do so
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

## Step 4: Import consumer data
- Create native table `card`, pointed to the GCS bucket with the .avro files (`gs://devday2022/clean-data`)
  -  make sure to use the file "card"
  -  file format is CSV
- Test the native table:
```
SELECT * FROM `<yourproject>.devday2022.card` LIMIT 10
```
