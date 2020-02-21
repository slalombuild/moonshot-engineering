# slalom-aws-devday-2019

AWS DevDay is a half-day, hands-on technical event, delivered by APN Partners who have demonstrated technical proficiency and proven customer success in specialized solution areas. Deep dive into AWS-powered partner solutions with Slalom – learn how to deploy a modern data engineering and analytics solution on AWS using Snowflake, Tableau, and AWS native services. Technical experts will explain key features and use cases, share best practices, provide technical demos, and answer questions.

![alt text](https://slalom-aws-workshop-us-west-2.s3-us-west-2.amazonaws.com/content/images/awsdevday/aws-dev-day-2019.png "AWS Dev Day Architecture")

## Overview

The AWS Dev Day application demonstrates how to automate a Snowflake analytics pipeline running on Amazon Web Services. It is based on the [snowflake-on-ecs](https://github.com/SlalomBuild/snowflake-on-ecs) analytics pipeline framework from Slalom Build. Tableau Desktop is used to enable quick visualizations with your data.  The Dev Day activities are divided into the following sections:

- Getting Started
- Setting up Snowflake
- Building the Pipeline in AWS
- Running the Pipeline
- Visual Analytics with Tableau Desktop

## Getting Started (20 - 30 min.)

### Download the Code

Download the Dev Day code [here](https://github.com/SlalomBuild/snowflake-on-ecs/archive/release-0.2.tar.gz) and save to a location on your disk you can refer to later.

### Log into AWS

Log into the AWS account provided for the Dev Day. You can reach the sign-in console [here](https://aws.amazon.com/marketplace/management/signin). Use the account alias provided during the Dev Day. Once logged in, you'll retrieve the password to be used with Snowflake in a later step. Be sure to select the Oregon region in the AWS console.

1. Log into AWS using your username/password  
![alt text](images/awssetup-001.png)
2. Change your password to something you can remember
![alt text](images/awssetup-002.png)
3. Navigate to AWS Systems Manager and click 'Parameter Store'. SSM Parameter Store is used to store passwords in an encrypted format.
![alt text](images/awssetup-003.png)
4. Retrieve the `snowflake_user` password you will use during the Snowflake steps. It will be stored under `/airflow-ecs/SnowflakeCntl`
![alt text](images/awssetup-004.png)

## Setting Up Snowflake (20 - 30 min.)

### Log into Your Snowflake Account

Log into your Snowflake account with the username and password you created when you set it up.

### Setup the Snowflake Framework

Open the `setup_framework.sql` script in the Snowflake UI. Let's review the script before we run it, and discuss topics such as why we're doing this, why it needs elevated privilege to run, and what types of objects it creates in Snowflake.

1. Copy the contents of the `setup_framework.sql` script (located in the `snowflake` directory) and paste in the Snowflake query editor window.
![alt text](images/image-02.png)
2. Highlight the query text and click on the blue Run button at the top left of the query editor window to run the script. This will create several objects in the database, including a few new databases, and a new user account that we'll use next.
3. Log out of your account by selecting the drop down menu on the top right of the window.
![alt text](images/image-04.png)

### Deploy the Snowflake Database Objects

Log into Snowflake with the `snowflake_user` account and the default password specified in the script.
Open the `deploy_objects.sql` script in the Snowflake UI. We'll take a walk through the script and describe what it is doing and why. Then we'll instruct them how to run it in one shot.

1. Log back in to Snowflake with the following credentials:
    - User Name: `snowflake_user`
    - Password: `__CHANGE__`
2. You will immediately be promted to change your password. Enter the password you retrieved in Step 4 of the `AWS Setup` section and click Finish. You are now logged in to the same Snowflake account as a different user.
3. Copy the contents of the `deploy_objects.sql` script (located in the `snowflake` directory) and paste in the Snowflake query editor window.
4. Highlight the query text and click on the blue Run button at the top left of the query editor window to run the script. This will create two stage tables, a file format, and two tables that we'll be loading data from S3 into.
![alt text](images/image-08.png)
5. We're now done with the initial Snowflake framework setup. Next, we'll be provisioning infrastructure in AWS that will allow us to run Airflow jobs to load these Snowflake tables with data.

## Building the Pipeline in AWS (30 min.)

### Review Foundational Components - Instructor Led

Now we have the Snowflake components installed, how can we leverage AWS to automatically load data? We'll first run through the foundational AWS components that were deployed prior to the Dev Day. This includes building a Docker image with our pipeline application, setting up a VPC network in AWS, and pushing our Docker image to Amazon ECR (Elastic Container Registry).

### Deploy Airflow Running on ECS Fargate

The framework uses [Apache Airflow](https://airflow.apache.org/) for the workflow engine. ECS Fargate allows us to use any application built using a Docker image.

1. Log into AWS and navigate to CloudFormation.
![alt text](images/image-09.png)
2. On the main CloudFormation page, you'll see that a stack called `ecs-fargate-network` has already been created. This stack contains the core networking infrastucture, including the VPC, subnets, security groups, and ECS cluster, that we'll use as a foundation for our next deployment.
![alt text](images/image-10.png)
3. Click the Create Stack button at the top right of the page. In the "Specify template" section, select "Upload a template file". Click "Choose file" and navigate to the location where the project repository is cloned. Select the `cloudformation/private-subnet-pubilc-service.yml` file. Click Next.
![alt text](images/image-12.png)
4. On the next page, you'll see a list of parameters that we need to set. These will be injected into the CloudFormation stack creation and will enable you to connect to Airflow from your computer once the ECS task is up and running. Set the following parameters then click Next:
    - Stack name: `ecs-airflow-<your name>`
    - AllowWebCidrIp: `<your IP address>`  (Note: this will be the same value for everyone in the room)
    - Snowflake account: this can be derived from the URL of your Snowlake account. For example: if the URL for your account is `https://ms72259.us-east-1.snowflakecomputing.com/`, then the account ID is `ms72259.us-east-1`
    - SnowflakeRegion - the region of your Snowflake account (corresponds to an AWS region)
  ![alt text](images/image-13.png)
  ![alt text](images/image-14.png)
5. On the next page, leave all of the default options and click Next. Scroll to the bottom of the next page and click Create Stack.
6. You'll be routed to a page that contains details on your stack. Click on the Events tab to see the progress of the stack creation. This process will take between 10 and 20 minutes. Once it's complete, you'll see an event indicating that the creation is complete.
![alt text](images/image-17.png)
7. Now that the stack has been created, navigate to ECS and select the single cluster that is running. 
![alt text](images/image-19.png)
8. On the cluster details page, select the Tasks tab and find the task with a task definition name that corresponds to the name you gave your CloudFormation stack.
![alt text](images/image-20.png)
9. Select the task to view the task details. In the Network section, copy the Public IP address and paste it in your browser address bar. Append `:8080` to the IP address and navigate to the page. You should see the Airflow user interface.
![alt text](images/image-21.png)
![alt text](images/image-22.png)

## Running the Analytics Pipeline (20 - 30 min.)

### Code Walk Through - SQL and DAGs

RDS/ECS will take about 10 minutes to launch. We'll use this time to walk through the Snowflake SQL and the Airflow DAGs used to automate it.

1. Snowflake SQL Example
    - Open the file located at `airflow/dags/sql/copy_raw_nyc_taxi.sql.sql`
    - This COPY command loads data from S3 into a Snowflake tables.
    - Metadata attributes such as `filename` and `file_row_number` are captured automatically. 
    - We also store the create process name and timestamp.

    ```sql
    copy into nyc_taxi_raw(vendorid, tpep_pickup_datetime, tpep_dropoff_datetime, passenger_count, trip_distance, 
        pickup_longitude, pickup_latitude, ratecodeid, store_and_fwd_flag, pulocationid, dolocationid, payment_type, 
        fare_amount, extra, mta_tax, tip_amount, tolls_amount, improvement_surcharge, total_amount, src_filename, 
        src_file_row_num, create_process, create_ts    
    )
    from (
    select t.$1,t.$2,t.$3,t.$4,t.$5,t.$6,t.$7,t.$8,t.$9,t.$10,t.$11,t.$12,t.$13,t.$14,t.$15,t.$16,t.$17,t.$18,t.$19,
    metadata$filename,
    metadata$file_row_number,
    'Airflow snowflake_raw Dag',
    convert_timezone('UTC' , current_timestamp )::timestamp_ntz
    from @quickstart/nyc-taxi-data/ t
    )
    file_format= (format_name = csv);
    ```

2. Airflow DAG Example
    - Open the file `airflow/dags/snowflake_raw.py`
    - Airflow DAGs are written in Python
    - This DAG file constructs a pipeline for loading datasets from S3 into Snowflake
    - A sequential workflow is shown here for demonstration purposes. We can also run tasks in parallel.

    ```python
    from datetime import timedelta

    import airflow
    from airflow import DAG
    from airflow.contrib.operators.snowflake_operator import SnowflakeOperator

    # These args will get passed on to each operator
    # You can override them on a per-task basis during operator initialization
    default_args = {
        'owner': 'airflow',
        'depends_on_past': False,
        'start_date': airflow.utils.dates.days_ago(1),
        'email': ['admin@example.com'],
        'email_on_failure': False,
        'email_on_retry': False,
        'retries': 1,
        'retry_delay': timedelta(minutes=5),
    }

    dag = DAG(
        'snowflake_raw',
        default_args=default_args,
        description='Snowflake raw pipeline',
        schedule_interval='0 */6 * * *',
    )

    t1 = SnowflakeOperator(
        task_id='copy_raw_airline',
        sql='sql/copy_raw_airline.sql',
        snowflake_conn_id='snowflake_default',
        warehouse='load_wh',
        database='raw',
        autocommit=True,
        dag=dag)

    t2 = SnowflakeOperator(
        task_id='copy_raw_nyc_taxi',
        sql='sql/copy_raw_nyc_taxi.sql',
        snowflake_conn_id='snowflake_default',
        warehouse='load_wh',
        database='raw',
        autocommit=True,
        dag=dag)

    t1 >> t2

    ```

### Run the Pipeline

Launch Airflow on ECS Task public IP port 8080. Run the Raw pipeline to load the Raw tables. Run the Analytics pipeline to load the Analytics tables. This will take some time to load. Go back to Snowflake and see the History tab, you can see Snowflake running the jobs and loading data. When it's done, Airflow UI will report success. We know it will since we've run this a billion times. Run a quick SQL query to see the data in the tables `query_analytics.sql`. Hey that's cool but we can do so much more, with Tableau Desktop!

1. In the Airflow UI, you should see two DAGs, `snowflake_analytics` and `snowflake_raw`. Toggle the `snowflake_raw` switch to On and select the Trigger Dag button in the Links section of the DAG row. When prompted to run the DAG, click OK.
![alt text](images/image-23.png)
2. The `snowflake_raw` DAG is now running and loading data into Snowflake. Navigate back to Snowflake and click on the History button. You should see the progress of the queries that are being executed by Airflow.
![alt text](images/image-24.png)
3. After a few minutes, the DAG should be complete. Back in Snowflake, run a quick query on the `public.airline_raw` table to confirm that data was loaded successfully.
![alt text](images/image-26.png) 
![alt text](images/image-27.png)
4. In Airflow, toggle the `snowflake_analytics` switch to On and select the Trigger Dag button in the Links section of the DAG row. When prompted to run the DAG, click OK. This DAG will take data loaded into the stage tables and load it into the final destination tables that can be used for analytical queries.
![alt text](images/image-28.png)
5. Once the DAG execution is complete, navigate back to Snowflake. Copy the contents of the `query_analytics.sql` script (located in the `snowflake` directory) and paste it in the Snowflake query window. Run the query.
![alt text](images/image-31.png)

## Visual Analytics with Tableau Desktop (30 - 40 min.)

Please click [here](https://snowflake-lab.s3-us-west-2.amazonaws.com/public/docs/AWS-Slalom-Snowflake-Tableau-DevDay-TableauDesktop-08.20.2019.pdf) to download the Tableau Desktop instructions, then please follow the instructions outlined in the document.
