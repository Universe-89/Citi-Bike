# Citi Bike - 


The goals of the project are:
* develop a data pipeline that will help to organize data processing in a batch manner (on a monthly basis);
* build analytical dashboard that will make it easy to discern the trends and digest the insights.

## Dataset used in the project
The data of Citi Bike Trip Histories could be found [here](https://s3.amazonaws.com/tripdata/index.html) in a compressed format.
It contains information about bikes sharing in different regions of New York.

The dataset includes the following columns:

* bikeid
* birth_year
* gender
* usertype
* tripduration
* start_station_id
* end_station_id
* start_station_name
* end_station_name
* start_station_latitude
* end_station_latitude
* start_station_longitude
* end_station_longitude
* starttime
* stoptime


User Type (Customer = 24-hour pass or 3-day pass user; Subscriber = Annual Member)
Gender (0=unknown; 1=male; 2=female)


## Problem description
The project is related to Citi Bike trips. Where do Citi Bikers ride? When do they ride? How far do they go? 
Which stations are most popular? What days of the week are most rides taken on? The provided data will help you discover 
the answers to these questions and more.

## Technologies
We are going to use the following technologies for this project:

* Cloud: GCP
    * Data Lake (DL): GCS
    * Data Warehouse (DWH): BigQuery
* Infrastructure as code (IaC): Terraform
* Workflow orchestration: Airflow
* Transforming data: DBT
* Data Visualization: Google Data Studio

## Project architecture
The end-to-end data pipeline includes the next steps:
* downloading, processing and uploading of the initial dataset to a DL;
* moving the data from the lake to a DWH;
* transforming the data in the DWH and preparing it for the dashboard;
* dashboard creating.

You can find the detailed information on the diagram below:
![pipeline](https://user-images.githubusercontent.com/93099817/166298371-ee9cda12-4332-4906-9d17-454cc36c4e39.png)



## Tutorial
This tutorial contains the instructions you need to follow to reproduce the project results.

### 1. Pre-requisites
Make sure you have the following pre-installed components: 
* [GCP account](https://cloud.google.com/)

* [Terraform](https://www.terraform.io/downloads)

* [Docker](https://docs.docker.com/get-docker/)

### 2. Google Cloud Platform
To set up GCP, please follow the steps below:
1. If you don't have a GCP account, please create a free trial.
2. Setup new project and write down your Project ID.
3. Configure service account to get access to this project and download auth-keys (.json). Please check the service 
account has all the permissions below:
   * Viewer
   * Storage Admin
   * Storage Object Admin
   * BigQuery Admin 
   
   (if you have any trouble with permissions when you are running the airflow dag, just add these permissions aswell)
   * BigQuery Data Editor
   * BigQuery Data Owner
   * BigQuery Data Viewer
   * BigQuery Job User
   * BigQuery User
4. Download [SDK](https://cloud.google.com/sdk) for local setup.
 
5. Set environment variable to point to your downloaded auth-keys:
```bash
export GOOGLE_APPLICATION_CREDENTIALS="<path/to/your/service-account-authkeys>.json"

# Refresh token/session, and verify authentication
gcloud auth application-default login
```
6. Enable the following options under the APIs and services section:
   * [Identity and Access Management (IAM) API](https://console.cloud.google.com/apis/library/iam.googleapis.com)
   * [IAM service account credentials API](https://console.cloud.google.com/apis/library/iamcredentials.googleapis.com)
   * [Compute Engine API](https://console.developers.google.com/apis/api/compute.googleapis.com) (if you are going to use VM instance)

### 3. Terraform
We will use Terraform to build and manage GCP infrastructure. Terraform configuration files are located in the [separate folder](terraform),
There are 3 configuration files: 
* [terraform-version](terraform/terraform-version) - contains information about the installed version of Terraform, i used 1.1.4 ;
* [variables.tf](terraform/variables.tf) - contains variables to make your configuration more dynamic and flexible;
* [main.tf](terraform/main.tf) - is a key configuration file consisting of several sections.

Now you can use the steps below to generate resources inside the GCP:
1. Move to the [terraform folder](terraform) using bash command `cd`.
2. Run `terraform init` command to initialize the configuration.
3. Use `terraform plan` to match previews local changes against a remote state.
4. Apply changes to the cloud with `terraform apply` command.

> Note: In steps 3 and 4 Terraform may ask you to specify the Project ID. Please use the ID that you noted down 
earlier at the project setup stage.
> 
If you would like to remove your stack from the Cloud, use the `terraform destroy` command. 

### 4. Airflow
The next steps provide you with the instructions of running Apache Airflow, which will allow you to run the entire 
orchestration, taking into account that you have already set up a GCP account.

You can run Airflow locally using docker-compose. Before running it, please make sure you have at least 5 GB of free RAM.
Alternatively, you can launch Airflow on a virtual machine in GCP

#### Setup
Go to the [airflow](airflow) subdirectory: here you can find the [Dockerfile](airflow/Dockerfile) and the lightweight version
of the [docker-compose.yaml](airflow/docker-compose.yaml) file that are required to run Airflow. 

The lightweight version of docker-compose file contains the minimum required set of components to run data pipelines. 
The only things you need to specify before launching it are your Project ID (`GCP_PROJECT_ID`) and Cloud Storage name (`GCP_GCS_BUCKET`)
in the [docker-compose.yaml](airflow/docker-compose.yaml). Please specify these variables according to your actual GCP setup.

You can easily run Airflow using the following commands:
* `docker-compose build` to build the image (takes ~15 mins for the first-time);
* `docker-compose up airflow-init` to initialize the Airflow scheduler, DB and other stuff;
* `docker-compose up` to kick up the all the services from the container.

Now you can launch Airflow UI and run the DAGs.
> Note: If you want to stop Airflow, please type `docker-compose down` command in your terminal.


#### Running DAGs
Open the [http://localhost:8080/](http://localhost:8080/) address in your browser and login using `airflow` username
and `airflow` password.

On the DAGs View page you can find three dags:
* `data_ingestion_to_gcs_dag` for downloading data from the source, unpacking and converting it to parquet format and 
finally uploading it to the Cloud Storage.
* `gcs_to_bq_dag` to subsequently create an external and then optimized table in BigQuery from the data stored in GCS.
* `data_transform_dag` to prepare data for analytics.

The first dag is scheduled to run every month, while the second one should be triggered manually. Therefore, you need
to activate the `data_ingestion_to_gcs_dag` dag first and wait for it to finish uploading data to GCS. And only after 
that manually run the `gcs_to_bq_dag` dag to create tables in DWH. Finally, you can trigger `data_transform_dag`. 

### 5. DBT
We are going to use [dbt](https://www.getdbt.com/) for data transformation in DWH and further analytics dashboard development.

First you will need to create a dbt Cloud account (if you don't already have one) using [this link](https://www.getdbt.com/signup/) 
and connect to your BigQuery by following [these instructions](https://docs.getdbt.com/docs/dbt-cloud/cloud-configuring-dbt-cloud/cloud-setting-up-bigquery-oauth).



### 6.Google Data Studio
When the production models are ready, you can start building a dashboard.

The [dashboard](https://datastudio.google.com/s/u5AyaHHljbo) is built using Google Data Studio.

And the final dashboard includes the following diagrams:
* Gender distribution among the trips
* Total trips count
* Total trips duration
* User type Record count and Trip duration
* User type distribution
* User type Record Count VS User type Trip duration

### 8.MY REPORT
from The data of Citi Bike Trip Histories from 2018 to 2020.
![Screenshot (1001)](https://user-images.githubusercontent.com/93099817/166313537-ac36c41d-6dfa-4bfe-a946-d9f2635ed9b5.png)




