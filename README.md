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

### Step 4: Create Authorized View - a Data Product
Time for some data engineering!

- First: Will create the first temporary table from the Data Transfer defined as: 
```
create or replace table `andreuankenobi-342014.devday2022.eng_contents`
as

SELECT
 CAST(REGEXP_EXTRACT(content, r'stackoverflow.com/questions/([0-9]+)/') AS INT64) id,
 sample_path
FROM
 `andreuankenobi-342014.devday2022.sample_contents`
WHERE
   id is not null and
 content LIKE '%stackoverflow.com/questions/%'
```
- Second: we'll create another temp table from the federated view, defined as: 
```
create or replace table `andreuankenobi-342014.devday2022.post_questions_base_table`
cluster by tags
as
 (

with a as(
select id, title, answer_count,favorite_count,view_count,score, creation_date, split(tags,'|') tags_array
from `andreuankenobi-342014.devday2022.federated-view`
 
)
select * except (tags_array) from a, unnest(tags_array) as tags
where id is not null
)

```
- Create a new dataset called `data-products` 
- Create a new query as follows: 
```
SELECT a.id,title, answer_count answers, favorite_count favs, view_count views, score, count(a.id) files, min(sample_path) sample_path
FROM `andreuankenobi-342014.devday2022.post_questions_base_table` a
join `andreuankenobi-342014.devday2022.eng_contents` b
on a.id=b.id
WHERE tags='python'
group by 1,2,3,4,5,6
```
And save it as "My First Data Product" 


## Step 4: Import consumer data
- Create native table `client`, pointed to the GCS bucket with the .avro files (`gs://devday2022/clean-data`)
  -  make sure to use the file "client"
  -  file format is CSV
- Test the native table:
```
SELECT * FROM `<yourproject>.devday2022.client` LIMIT 10
```

## Step 5: Open Data Catalog
- under search, look for SSN. Oh-oh! our newly imported table has clear SSNs! 
- looks like many other fields might be impacted, so let's create a policy tag and then scan with DLP
- create a new policy tag
  - create a new taxonomy
  - name `sensitive`
  - Tag Name `PII`
  - Click on the policy tag, show the principals

## Step 5: Scan the "Client" table with DLP
- In Data Catalog, sreach again for SSN. Find the table, and specify "scan with DLP" 
- Create a DLP scan for the newly imported table
- Notable settings: 
  - Name: `PIIScan`
  - Detection: set Confidence Thresold to "Unspecified"
  - Actions: Write to BigQuery, project `<yourproject>`, dataset `devday2022`, table `dlpcscan`
  - Actions: Notify via email
  - Actions: Publish to Data Catalog
  - Schedule: none
  - Confirm job and create
  
## Step 6: Enforce Column-level security
- In Data Catalog, scearh for tag template "Data Loss Prevention Tags"
- The "Client" table will pop up. Preview it in the preview panel to confirm SSN is there 
- Click on the table, and confirm the SSN tag has been fired
- Confirm how there's not policy tag attached to the column. 
- Scroll up, and select "Open in BigQuery"
- Under Schema, click "Edit Schema" - on the bottom
- Select SSN, select "Add Policy Tag". 
- The table is now secured! 

## Step 7: Mesh it up!
- Create "My first lake" 
- Create the "Raw" Data zone 
- Add ASSETS:
  - BigQuery tables
  - Storage Buckets 
