
hudi-minio-starrpcks-superset

![Screenshot 2024-02-09 at 6 36 21 PM](https://github.com/soumilshah1995/hudi-minio-starrpcks-superset/assets/39345855/590981f8-b218-428b-a300-7e07b705843b)


# Steps 

# Step 1: Spin up stack 
```
docker-compose up --build -d
```
# step 2:  Create Bucket
```agsl

Head http://localhost:9001/login
username as admin 
password password 

create a bucket called huditest

```
# Step  3: Run python file hudi_job_hive_sync.py
```
try:
    import os
    import sys
    import uuid
    import pyspark
    import datetime
    from pyspark.sql import SparkSession
    from pyspark import SparkConf, SparkContext
    from faker import Faker
    import datetime
    from datetime import datetime
    import random
    import pandas as pd  # Import Pandas library for pretty printing

    print("Imports loaded ")

except Exception as e:
    print("error", e)

HUDI_VERSION = '0.14.0'
SPARK_VERSION = '3.4'

SUBMIT_ARGS = f"--packages org.apache.hudi:hudi-spark{SPARK_VERSION}-bundle_2.12:{HUDI_VERSION},org.apache.hadoop:hadoop-aws:3.3.2 pyspark-shell"
os.environ["PYSPARK_SUBMIT_ARGS"] = SUBMIT_ARGS
os.environ['PYSPARK_PYTHON'] = sys.executable

spark = SparkSession.builder \
    .config('spark.serializer', 'org.apache.spark.serializer.KryoSerializer') \
    .config('spark.sql.extensions', 'org.apache.spark.sql.hudi.HoodieSparkSessionExtension') \
    .config('className', 'org.apache.hudi') \
    .config('spark.sql.hive.convertMetastoreParquet', 'false') \
    .getOrCreate()

spark._jsc.hadoopConfiguration().set("fs.s3a.endpoint", "http://127.0.0.1:9000/")
spark._jsc.hadoopConfiguration().set("fs.s3a.access.key", "admin")
spark._jsc.hadoopConfiguration().set("fs.s3a.secret.key", "password")
spark._jsc.hadoopConfiguration().set("fs.s3a.path.style.access", "true")
spark._jsc.hadoopConfiguration().set("fs.s3a.connection.ssl.enabled", "false")
spark._jsc.hadoopConfiguration().set("fs.s3a.impl", "org.apache.hadoop.fs.s3a.S3AFileSystem")
spark._jsc.hadoopConfiguration().set("fs.s3a.aws.credentials.provider",
                                     "org.apache.hadoop.fs.s3a.SimpleAWSCredentialsProvider")

global faker
faker = Faker()


def get_customer_data(total_customers=2):
    customers_array = []
    for i in range(0, total_customers):
        customer_data = {
            "customer_id": str(uuid.uuid4()),
            "name": faker.name(),
            "state": faker.state(),
            "city": faker.city(),
            "email": faker.email(),
            "created_at": datetime.now().isoformat().__str__(),
            "address": faker.address(),
            "salary": faker.random_int(min=30000, max=100000)
        }
        customers_array.append(customer_data)
    return customers_array


global total_customers, order_data_sample_size
total_customers = 5000
customer_data = get_customer_data(total_customers=total_customers)
spark_df_customers = spark.createDataFrame(data=[tuple(i.values()) for i in customer_data],
                                           schema=list(customer_data[0].keys()))
spark_df_customers.show(3)


def write_to_hudi(spark_df,
                  table_name,
                  db_name,
                  method='upsert',
                  table_type='COPY_ON_WRITE',
                  recordkey='',
                  precombine='',
                  partition_fields=''
                  ):
    path = f"s3a://huditest/hudi/database={db_name}/table_name={table_name}"
    # path = f"file:///Users/soumilshah/IdeaProjects/SparkProject/DeltaStreamer/hudi/{db_name}/{table_name}"

    hudi_options = {
        'hoodie.table.name': table_name,
        'hoodie.datasource.write.table.type': table_type,
        'hoodie.datasource.write.table.name': table_name,
        'hoodie.datasource.write.operation': method,
        'hoodie.datasource.write.recordkey.field': recordkey,
        'hoodie.datasource.write.precombine.field': precombine,
        "hoodie.datasource.write.partitionpath.field": partition_fields,

        "hoodie.datasource.hive_sync.database": db_name,
        "hoodie.datasource.hive_sync.table": table_name,
        "hoodie.datasource.hive_sync.partition_fields": partition_fields,
        "hoodie.datasource.hive_sync.partition_extractor_class": "org.apache.hudi.hive.MultiPartKeysValueExtractor",
        "hoodie.datasource.hive_sync.metastore.uris": "thrift://localhost:9083",
        "hoodie.datasource.hive_sync.mode": "hms",
        "hoodie.datasource.hive_sync.enable": "true",
        "hoodie.datasource.write.hive_style_partitioning": "true"
    }

    print("\n")
    print(path)
    print("\n")

    spark_df.write.format("hudi"). \
        options(**hudi_options). \
        mode("append"). \
        save(path)


write_to_hudi(
    spark_df=spark_df_customers,
    db_name="default",
    table_name="customers",
    recordkey="customer_id",
    precombine="created_at",
    partition_fields="state"
)

```
# step 4: 
```
docker exec -it starrocks-hudi-minio-starrocks-fe-1 /bin/bash -c 'mysql -P 9030 -h 127.0.0.1 -u root --prompt="StarRocks > "'
```




# Step 5: Create External Catlog 
```
CREATE EXTERNAL CATALOG hudi_catalog_hms

PROPERTIES
(
    "type" = "hudi",
    "hive.metastore.type" = "hive",
    "hive.metastore.uris" = "thrift://hive-metastore:9083",
    "aws.s3.use_instance_profile" = "false",
    "aws.s3.access_key" = "admin",
    "aws.s3.secret_key" = "password",
    "aws.s3.region" = "us-east-1",
    "aws.s3.enable_ssl" = "false",
    "aws.s3.enable_path_style_access" = "true",
    "aws.s3.endpoint" = "http://minio:9000"
);


set catalog hudi_catalog_hms;
show databases;
use default;
show tables;
select * from customers;
```

# Step 6 :Clone Apache Superset Repo
```
git clone https://github.com/apache/superset.git
export DOCKER_DEFAULT_PLATFORM=linux/amd64
docker-compose -f docker-compose-non-dev.yml up
```
# Step 7 : Exec into Apache Superset app container and install starrocks 
```
pip install starrocks
```
# Step 8: Use Servo to create DNS Name 
```
ssh -R 9030:localhost:9030 -R 8030:localhost:8030 -R 8040:localhost:8040 abc.serveo.net
```
# Step 9 : Create a connection in Superset for starocks 
```
starrocks://root:@abc.serveo.net:9030/hudi_catalog_hms.default
```
![image](https://github.com/soumilshah1995/hudi-minio-starrpcks-superset/assets/39345855/f6074c4a-9ff3-487f-a64d-4ffdf0aae1c3)


# Step 10 : Star Querying 
![image](https://github.com/soumilshah1995/hudi-minio-starrpcks-superset/assets/39345855/8bfe7e72-35a2-4995-af54-10ef26fc1e17)


## Special Thanks to Albert  Wong 
