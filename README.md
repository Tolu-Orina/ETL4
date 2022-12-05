# ETL4: LOADING DATA INTO AN AMAZON REDSHIFT CLUSTER AND RUNNING QUERIES WITH REDSHIFT SPECTRUM

### APPROACH
- Create Redshift Cluster
- Set up Redshift Spectrum: to query external tables on S3
- Load subset of read data into a local table in Redshift
- Run complex queries

A Data Lake serves as a single source of truth across multiple lines of business i.e from different
domains including, finance, healthcare, advertisement, marketing, IoT, It usually includes different variety of data ranging from 
structured, semi-structured and unstructured data. As a Data Engineer you may need to load data into an 
external data store(data warehouses or marts) to enable a specific set or subset of data of interest to a particular group of users.

Two primary purposes for Data warehouses are:
1. Provide a database with a subset of the data in the data lake, optimized for specific types of queries
(such as for a specific business function)  
2. High performant, low-latency query engine often used to power BI applications

AMAZON REDSHIFT
Amazon Redshift is a super-fast cloud-native data warehousing solution that provides high-performance, low-latency access to data stored
in the data warehouse. The Redshift architecture uses  the leader and compute nodes distributed model where each compute nodes contains 
a certain amount of compute power (CPU and memory), as well as certain amount of local storage.
Redshift stores data in a columnar data format, which is optimized for analytics, and uses compression algoriths to reduce disk lookup tine when a
query is run.

The Data and Use Case is for a Travel Agency.
Agents need to ensure that they can find the best deal for accommodation
in New York City and New Jersey City that os close to specific popular tourist attractions,
such as the Freedom Tower and the Empire State Building 

Dataset: Gotten from Inside Airbnb
New York Data: ny-listings.csv
New Jersey Data: nj-listings.csv

Copying the file to s3:
aws s3 cp ny-listings.csv s3://etl-trex-landing-zone/listings/city=new_york_city/ny-listings.csv
aws s3 cp nj-listings.csv s3://etl-trex-landing-zone/listings/city=new_jersey_city/nj-listings.csv 	

Quick S3 select query from the uploaded dataset to view first 5 rows
```
SELECT * FROM s3object s LIMIT 5;
```
![s3_select](https://user-images.githubusercontent.com/78595009/205768136-354cdcc3-9a8e-413c-9eef-dbc7666537f4.png)

Sample query on test databse

```
SELECT * FROM sales 
ORDER BY qtysold DESC
LIMIT 10
```

REDSHIFT Query

![Screenshot (309)](https://user-images.githubusercontent.com/78595009/205768018-b1840409-f9f7-451f-859a-5d66a60f3609.png)


- Create External Schema and New Database in Glue Catalog

```
CREATE EXTERNAL SCHEMA spectrum_schema
FROM data catalog
database 'accomodation'
iam_role '<my-redshift-role>'
CREATE EXTERNAL DATABASE IF NOT EXISTS;
```

- Define an external table
```
CREATE EXTERNAL TABLE spectrum_schema.listings(
 listing_id INTEGER,
 name VARCHAR(100),
 host_id INT,
 host_name VARCHAR(100),
 neighbourhood_group VARCHAR(100),
 neighbourhood VARCHAR(100),
 latitude Decimal(8,6),
 longitudes Decimal(9,6),
 room_type VARCHAR(100),
 price SMALLINT,
 minimum_nights SMALLINT,
 number_of_reviews SMALLINT,
 last_review DATE,
 reviews_per_month NUMERIC(8,2),
 calculated_host_listings_count SMALLINT,
 availability_365 SMALLINT)
partitioned by(city varchar(100))
row format delimited
fields terminated by ','
stored as textfile
location 's3://etl-trex-landing-zone/listings/'; 
```
- Add Partitions
```
alter table spectrum_schema.listings add
partition(city='jersey_city') 
location 's3://etl-trex-landing-zone/listings/
city=jersey_city/' 

alter table spectrum_schema.listings add
partition(city='new_jersey_city') 
location 's3://etl-trex-landing-zone/listings/
city=new_jersey_city/'

alter table spectrum_schema.listings add
partition(city='new_york_city') 
location 's3://etl-trex-landing-zone/listings/
city=new_york_city/'
```

- Create Schema for local Redshift Table
```
create schema if not exists accommodation_local;
```

```
CREATE TABLE dev.accommodation_local.listings(
 listing_id INTEGER,
 name VARCHAR(100),
 neighbourhood_group VARCHAR(100),
 neighbourhood VARCHAR(100),
 latitude Decimal(8,6),
 longitudes Decimal(9,6),
 room_type VARCHAR(100),
 price SMALLINT,
 minimum_nights SMALLINT,
 city VARCHAR(40))
distkey(listing_id)
sortkey(price);
```
