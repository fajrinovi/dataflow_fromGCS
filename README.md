# Dataflow

## Tech Stack
1. Cloud Composer
2. Google Cloud Storage
3. BigQuery
4. Dataflow

## Setup
### Composer Environment
Composer is a managed Airflow environment provided by GCP (Google Cloud Platform). It simplifies Airflow management, allowing you to devote more time to writing DAGs (Directed Acyclic Graphs). While it's not mandatory to use Composer for converting data from Google Cloud Storage (GCS) to BigQuery, it can be beneficial if you want to utilize a DAG for the task. 

Here are the steps to create an environment in Composer:
1. Activate Composer API if you haven't. You can refer to this link on how to: https://cloud.google.com/endpoints/docs/openapi/enable-api#console
2. Open https://console.cloud.google.com/
3. On Navigation Menu, go to Big Data > Composer. Wait until Composer page shown
4. Click **CREATE** button to create new Composer environment and choose Composer 1![create new Composer environment](https://github.com/fajrinovi/dataflow_fromGCS/assets/57751463/b81da5c8-1381-4c81-865b-e17fba981794)
5. Fill some required fields there. This projects use this machine config:
   - Name: cloud-dataflow
   - Location: asia-east2
   - Image Version: composer-1.20.12-airflow-2.3.4
   - Node count: 3
   - Zone: asia-east2-a
   - Machine Type: n1-standard-1
   - Disk size (GB): 30 GB
   - Service account: choose your Compute Engine service account.
   - Cloud SQL machine type: db-n1-standard-2
   - Web server machine type: composer-n1-webserver2
6.  Fill the Environment Variable with
ENVIRONMENT: production
7. Then, click the **CREATE** button
8. You will need to wait a few minutes (15-25 minutes) for the environment to be successfully created. After the environment has been created successfully you will see like this:![3  composer done](https://github.com/fajrinovi/dataflow_fromGCS/assets/57751463/46759cac-cdc0-4e26-8c50-2a4ed9fedd84)
notes: errors may occur, you can try to change the location

### Airflow webserver on Composer
After the environment successfully created from the previous steps. Now its time to access the airflow web page. Here are the steps:

1. Click the Airflow webserver link in Composer Environments list. This will open airflow web page that same as we opened http://localhost:8080 if we using Local Airflow.
2. Create `json` file on your project that stores Airflow variables. Lets call it `variables.json`. The minimum key-value of its file must contains this:
   ```js
    {
      "PROJECT_ID": "",
      "GCS_TEMP_LOCATION": "",
      "BUCKET_NAME": "",
      "ALL_KEYWORDS_BQ_OUTPUT_TABLE": "",
      "GCS_STG_LOCATION": "",
      "MOST_SEARCHED_KEYWORDS_BQ_OUTPUT_TABLE": "",
      "EVENTS_BQ_TABLE": ""
    }
   ```
   - `PROJECT_ID` : your GCP project ID (e.g: fellowship10)
   - `GCS_TEMP_LOCATION`: temp location to store data before loaded to BigQuery. The format is `gs://` followed by your bucket and object name (e.g: gs://bucket-dataflow10/tmp)
   - `BUCKET_NAME` : name of your GCS bucket (e.g: bucket-dataflow10)
   - `ALL_KEYWORDS_BQ_OUTPUT_TABLE` : output table in BigQuery to store keyword search data. The value format is `dataset_id.table_id` (e.g: dataflow_citizen.citizen)
   - `GCS_STG_LOCATION`: staging location to store data before loaded to BigQuery. The format is `gs://` followed by your bucket and object name (e.g: gs://bucket-dataflow10/srg)
   - `MOST_SEARCHED_KEYWORDS_BQ_OUTPUT_TABLE`: output table in BigQuery to store most searched keyword data. The value format is `dataset_id.table_id` (e.g: dataflow_citizen.citizen)
   - `EVENTS_BQ_TABLE`: output table in BigQuery to store event data from reverse engineering result. The value format is `dataset_id.table_id` (e.g: fellowship10.dataflow_citizen.citizen)
3. Go to Admin > Variables
4. Choose File then click Import Variables
5. After the variable has been created successfully you will see like this:![variable airflow](https://github.com/fajrinovi/dataflow_fromGCS/assets/57751463/80b8e1d7-0b29-43a5-a9a4-9150397bd087)

### BigQuery
1. On your GCP console, go to BigQuery. You can find it on More Products > Analytics > BigQuery, or simply search it at the search bar.
2. Then create dataset

   ![4  Create dataset](https://github.com/fajrinovi/dataflow_fromGCS/assets/57751463/70df9b45-9a7b-4c27-a012-237cb60b1e21)
3. Fill the Dataset field such as:
  - Data set ID (example: dataflow_citizen)
  - Data location. Note: make sure the dataset location is the same as the bucket location!
![create dataset](https://github.com/fajrinovi/dataflow_fromGCS/assets/57751463/8f9db4dc-d53b-4480-9e62-8ab2fd40a44f)

4. Click **CREATE DATASET**
5. Ensure that your dataset has been created

   ![dataset info](https://github.com/fajrinovi/dataflow_fromGCS/assets/57751463/831f74aa-4a8a-4d5f-9d82-c156bf016a72)
6. Create table
  
   ![Screenshot 2023-07-10 234414](https://github.com/fajrinovi/dataflow_fromGCS/assets/57751463/fe478f16-9fb3-4b3d-866b-de35cf9e3df3)
9. Fill the Table field such as:
- table name (eg: citizen)
- schema (can be skipped)

### Google Cloud Storage
1. Back to your GCP console, choose Cloud Storage. You can find it on **Storage > Cloud Storage**
2. Click **CREATE BUCKET** button. Then fill some fields such as:
   - Name your bucket (example: bucket-dataflow10)
   - Choose where to store your data
     - I would suggest to choose **Region** option because it offers lowest latency and single region. But if you want to get high availability you may consider to choose other location type options.
     - Then choose nearest location from you
   - Leave default for the rest of fields.
3. Click **CREATE**
4. Your bucket will be created and showed on GCS Browser

   ![create bucker-3](https://github.com/fajrinovi/dataflow_fromGCS/assets/57751463/b20f47a9-f3e9-4e06-9ab6-a705050081f3)
5. Upload files to bucket:
- javascript file and json that you can find in the dataflow-function folder.
- citizen.txt that you can find in the data folder.
6. Click **CREAT FOLDER** button. Then fill name of folder:
  - Name your folder (e.g: `tmp_exe/` dan `tmp_proc/`)
7. Clikc **CREATE**
8. So the result will be like this:

![bucker](https://github.com/fajrinovi/dataflow_fromGCS/assets/57751463/a8a57443-15da-4bb5-8677-2206d33ae1a0)

### Dataflow
1. On your GCP console, go to Dataflow. You can find it on More Products > Analytics > Dataflow, or simply search it at the search bar.
2. Make sure that you have enabled API.
3. Click **CREATE JOB FROM TEMPLATE**
4. Fill the field such as:
   - Jobname (eg: bq_dataflow)
   - Regional endpoint: asia-southeast2
   - Dataflow template: Process Data in Bulk (batch) > Text Files on Cloud Storage to BigQuery
   - Javascript path: open your javascript file in the bucket, and copy the gsutil URL.

     ![bucket js](https://github.com/fajrinovi/dataflow_fromGCS/assets/57751463/6225828a-0532-470e-9c7d-267249c5a6a2)

   - Json path: browse your json file.
   - Javascript UDF name: the function to call from your JavaScript file (in this project, the function to call javascript is "transform")
   - Bigquery output table (browse your output table, in this project the table is "citizen")
   - Temporary Bigquery directory (browse your temporary folder, in this project the table is `tmp_proc/`)
   - Temporary location (browse your temporary folder, in this project the table is `tmp_exe/`)

     ![create job-2](https://github.com/fajrinovi/dataflow_fromGCS/assets/57751463/b4ee978c-f7e6-48f4-bee5-e6111999efdc)

5. Click **RUN JOB**
6. This is the result of Dataflow Pipeline:
   ![job run](https://github.com/fajrinovi/dataflow_fromGCS/assets/57751463/75fd1154-fe66-485c-af49-4fc61ff4b31d)

7. Check the table output in BigQuery to ensure the data has been loaded.
   ![output in bq](https://github.com/fajrinovi/dataflow_fromGCS/assets/57751463/6657e130-644d-4fc7-bfea-78e47ddb97ec)






     

