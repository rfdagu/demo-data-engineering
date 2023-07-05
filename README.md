# data-engineering-demo
## Introduction

This file describes the procedures taken to run a demonstration of Data Engineering and Data Analysis using AWS
resources as part of a scalable solution, namely S3 and Amazon Redshift, a scalable Data Warehouse tool.

The goal of this exercise is to conduct data analysis on a dataset of NYC Taxi Trips between 2009 and 2012.
These instructions cover the setup of the AWS environment, loading of data, queries, processing and analysis.

## Preliminary analysis
Six files were provided for the analysis: 4 json files with the trip data and 2 csv files with lookup tables
for the vendors and common typos and misspellings for the payment method.

```
$ aws s3api get-bucket-location --bucket my-bucket
```

All files were contained in AWS S3 buckets, however, the command above returned "Access Denied". Since
the command is only available for the owner of the bucket, I chose to have all the data available on a bucket of my own.

The CSV files can be quickly examined for the column headers:

```
~/Downloads$ head data-payment_lookup-csv.csv 
A,B
payment_type,payment_lookup
Cas,Cash
CAS,Cash
Cre,Credit
CRE,Credit
No ,No Charge
Dis,Dispute
Cash,Cash
CASH,Cash
```

However, there is useless data on the payment lookup table, this will be corrected when the data is loaded to the
Data Warehouse tool Amazon Redshift:

```
~/Downloads$ tail data-payment_lookup-csv.csv 
...
3269,Foo
3270,Foo
3271,Foo
```

The vendors table is also analysed and because of its small size, a thorough look showed that there was no faulty data:

```
~/Downloads$ head data-vendor_lookup-csv.csv 
vendor_id,name,address,city,state,zip,country,contact,current
CMT,"Creative Mobile Technologies, LLC",950 4th Road Suite 78,Brooklyn,NY,11210,USA,contactCMT@gmail.com,Yes
VTS,VeriFone Inc,26 Summit St.,Flushing,NY,11354,USA,admin@vtstaxi.com,Yes
DDS,"Dependable Driver Service, Inc",8554 North Homestead St.,Bronx,NY,10472,USA,9778896500,Yes
TS,Total Solutions Co,Five Boroughs Taxi Co.,Brooklyn,NY,11229,USA,mgmt@5btc.com,Yes
MT,Mega Taxi,4 East Jennings St.,Brooklyn,NY,11228,USA,contact@megataxico.com,No
```

The json files are large and therefore the "head" command is specially useful:

```
~/Downloads$ head data-sample_data-nyctaxi-trips-2009-json_corrigido.json 
{"vendor_id":"CMT","pickup_datetime":"2009-04-21T18:51:11.767205+00:00","dropoff_datetime":"2009-04-21T18:57:09.433767+00:00","passenger_count":2,"trip_distance":0.8,"pickup_longitude":-74.004114,"pickup_latitude":40.74295,"rate_code":null,"store_and_fwd_flag":null,"dropoff_longitude":-73.994712,"dropoff_latitude":40.74795,"payment_type":"Cash","fare_amount":5.4,"surcharge":0,"tip_amount":0,"tolls_amount":0,"total_amount":5.4}
{"vendor_id":"CMT","pickup_datetime":"2009-01-13T07:40:07.639754+00:00","dropoff_datetime":"2009-01-13T07:50:36.386011+00:00","passenger_count":1,"trip_distance":5.4,"pickup_longitude":-73.996506,"pickup_latitude":40.747784,"rate_code":null,"store_and_fwd_flag":null,"dropoff_longitude":-73.940449,"dropoff_latitude":40.792385,"payment_type":"Cash","fare_amount":15.4,"surcharge":0,"tip_amount":0,"tolls_amount":0,"total_amount":15.4}
...
```

## Setting up the AWS resources
### S3

We started with a Linux computer with AWS CLI installed and the account/password already set. An S3 can be created with
the command below. It is planned that Amazon Redshift will access the data from the bucket from within the default VPC,
and as such I will use my-bucket as the name.

```
$ aws s3api create-bucket --bucket my-bucket --create-bucket-configuration LocationConstraint=us-east-2
$ aws s3api put-public-access-block \
> --bucket my-bucket \
> --public-access-block-configuration "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"
```

All files are copied to S3 with the commands below. It is recommended to use the AWS CLI rather than the website
interface for larger files.

```
$ aws s3 cp data-vendor_lookup-csv.csv s3://my-bucket/
$ aws s3 cp data-payment_lookup-csv.csv s3://my-bucket/
...
```

One extra file will be uploaded to S3, called `jsonpaths.json`, this file is required by Amazon Redshift to map the
keys of the json data to column names for the relational database. There is an 'auto' option when copying data from S3
to Redshift, but it did not work well in my experience.

`jsonpaths.json` is written as follows and is also saved in this code repository:

```
{

"jsonpaths": [

"$['vendor_id']",
"$['pickup_datetime']",
"$['dropoff_datetime']",
"$['passenger_count']",
"$['trip_distance']",
"$['pickup_longitude']",
"$['pickup_latitude']",
"$['rate_code']",
"$['store_and_fwd_flag']",
"$['dropoff_longitude']",
"$['dropoff_latitude']",
"$['payment_type']",
"$['fare_amount']",
"$['surcharge']",
"$['tip_amount']",
"$['tolls_amount']",
"$['total_amount']"

]

}
```

### Setting up IAM Role for permissions for Amazon Redshift

IAM roles are used in AWS to associate permissions for resources to access and use other resources. In our case, an IAM
role is necessary for the Redshift cluster to access the S3 buckets and load the data.

For simplicity and because this resource deals with security credentials, I preferred to access the AWS Dashboard and
manually create the IAM role through IAM -> Roles -> Create Role. The name of the IAM role is "myRedshiftRole", and the
only permission is `AmazonS3ReadOnlyAccess`. The process will create the Role ARN, which is necessary when copying from
S3.

### Creating an Amazon Redshift cluster

The processing of data will be conducted by a cluster of Amazon Redshift nodes, a scalable solution for Data Warehousing.
The CLI command to create a cluster is in the format below. However, the actual cluster was created on the AWS website
because the node type and number (1x dc2.large) were selected to keep within the limits of the Free Tier.

Note that the ARN of the IAM Role created previously is associated to the cluster to allow access to the S3 buckets.

```
$ aws redshift create-cluster --node-type dc2.large --number-of-nodes 1 --master-username awsuser \
> --master-user-password <password> --cluster-identifier mycluster --publicly-accessible --db-name dev \
> --availability-zone us-east-2 --port 5439 --iam-roles <ARN of IAM Role>
```

The Redshift cluster should be ready for the queries, that can be run from the Query Editor on the AWS website, or from
an SQL client. The first queries to load the data from S3 will be run from the Query Editor. The later queries for the
data analysis of the NYC trip data will be run from an SQL client implemented by the Python library "sqlalchemy".

One last configuration to allow the connection from the external Python script is to allow outside connections in the
cluster's security group, by adding an inbound rule.

On the AWS website: Redshift -> mycluster -> Properties -> Security Group -> Inbound Rules -> Add rule for:
`Type: All traffic, Source: My IP.`

This security rule may be too lax for a production application, however, it is enough for this demonstration. Also, the
AWS resources will be instantiated for a quick series of queries for data analysis and later deleted to avoid ongoing
costs.

### Copying the data from S3

All the provided data will be loaded into Redshift as relational tables. The **Preliminary Analysis** section covered
the discovery of the table headers for both the CSV and JSON files, the latter resulting in the `jsonpaths.json` file.

The tables **vendors**, **payment** and **trips** were created with the queries below, directly in the AWS webpage
Query Editor:

```
create table vendors(
  vendor_id varchar(50),
  name varchar(50),
  address varchar(50),
  city varchar(50),
  state varchar(50),
  zip varchar(50),
  country varchar(50),
  contact varchar(50),
  current varchar(50)
);

create table payment(
  payment_type varchar(20),
  payment_lookup varchar(20)
);

create table trips(
  vendor_id varchar(30),
  pickup_datetime TIMESTAMP,
  dropoff_datetime TIMESTAMP,
  passenger_count INTEGER,
  trip_distance FLOAT,
  pickup_longitude FLOAT,
  pickup_latitude FLOAT,
  rate_code VARCHAR(20),
  store_and_fwd_flag VARCHAR(30),
  dropoff_longitude FLOAT,
  dropoff_latitude FLOAT,
  payment_type VARCHAR(30),
  fare_amount FLOAT,
  surcharge FLOAT,
  tip_amount FLOAT,
  tolls_amount FLOAT,
  total_amount FLOAT
);
```

The data from the S3 bucket was copied with the queries below. Note that the ARN Role that is associated with the
Redshift cluster is necessary to successfully connect with S3:

```
copy vendors from 's3://my-bucket/data-vendor_lookup-csv.csv'
credentials 'aws_iam_role=<ARN IAM Role>'
ignoreheader 1
delimiter ',' region 'us-east-2'
CSV;

copy payment from 's3://my-bucket/data-payment_lookup-csv.csv'
credentials 'aws_iam_role=<ARN IAM Role>'
ignoreheader 2
delimiter ',' region 'us-east-2'
CSV;

copy trips from 's3://my-bucket/data-sample_data-nyctaxi-trips-2009-json_corrigido.json'
credentials 'aws_iam_role=<ARN IAM Role>'
TIMEFORMAT AS 'YYYY-MM-DDTHH:MI:SS'
json 's3://my-bucket/jsonpaths.json';

copy trips from 's3://my-bucket/data-sample_data-nyctaxi-trips-2010-json_corrigido.json'
credentials 'aws_iam_role=<ARN IAM Role>'
TIMEFORMAT AS 'YYYY-MM-DDTHH:MI:SS'
json 's3://my-bucket/jsonpaths.json';

copy trips from 's3://my-bucket/data-sample_data-nyctaxi-trips-2011-json_corrigido.json'
credentials 'aws_iam_role=<ARN IAM Role>'
TIMEFORMAT AS 'YYYY-MM-DDTHH:MI:SS'
json 's3://my-bucket/jsonpaths.json';

copy trips from 's3://my-bucket/data-sample_data-nyctaxi-trips-2012-json_corrigido.json'
credentials 'aws_iam_role=<ARN IAM Role>'
TIMEFORMAT AS 'YYYY-MM-DDTHH:MI:SS'
json 's3://my-bucket/jsonpaths.json';
```

The cluster is now ready and load with data for the analysis! The next queries will be run by the `query_cluster.py`
script. One last query that cleans up the 'Foo' lines in the payment table can be run still in the Query Editor:

```
delete from payment
where payment_lookup='Foo';
```

## Setting up the Python environment and running the main script

Setting up the python enviroment to run the SQL client requires few steps. The script runs on Python3.7 and the user
can use a virtual environment if preferred. The required packages can be directly installed through PIP with the
`requirements.txt` file.

```
$ sudo apt-get install python3.7
$ sudo apt-get install python3.7-dev
$ sudo apt-get install python3-pip
$ pip3 install -r requirements.txt
```

The `query_cluster.py` script uses the "sqlalchemy" library to create an SQL client. This instantiation requires
credentials to the cluster, and for security issues, those can be loaded from environment variables. It is therefore
necessary to load the database name, master user, master password, port and host url from when the cluster was created:

```
$ export AWS_USER=<aws_user>
$ export AWS_PASSWORD=<aws_password>
$ export AWS_HOST=<aws_host>
$ export AWS_PORT=<aws_port>
$ export AWS_DB=<aws_db>
```

Finally, the main query script can be run with `$ python3 query_cluster.py`. The script prints on the console the
results of the questions asked for the data analysis. It also saves those results as CSV files in the "Results" folder,
alongside the generated graphs that were asked in some of the questions.

## Results

The data analysis of the NYC Taxi Trips data is in the `Analysis.html` file and uses the results listed in the "Results"
folder.

## Deleting used AWS resources

After the compiling of results for the data analysis, the resources allocated from AWS can be deleted to avoid costs.
The created Redshift cluster can be deleted with the command below:

```
$ aws redshift delete-cluster --cluster-identifier mycluster --final-cluster-snapshot-identifier myfinalsnapshot
```

In the same way, the data stored in the S3 buckets can be deleted. Note that the bucket must be emptied first:

```
$ aws s3 rm s3://my-bucket --recursive
$ aws s3api delete-bucket --bucket my-bucket --region us-east-2
```
