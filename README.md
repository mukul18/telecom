
The server side (Data Fabric) uses the Data Fabic Sandbox located at :

https://docs.ezmeral.hpe.com/datafabric-customer-managed/73/MapRContainerDevelopers/RunMapRContainerDevelopers.html?hl=sandbox

In the distrib, mapr_devsandbox_container_setup.sh contains the right port numbers
To mount your local folder associated with the project, replace - local telecom folder  - by your home project directory in the  following command:

```
docker run -d --privileged -v /tmp/maprdemo/zkdata:/opt/mapr/zkdata -v /tmp/maprdemo/pid:/opt/mapr/pid  -v /tmp/maprdemo/logs:/opt/mapr/logs  -v /tmp/maprdemo/nfs:/mapr -v <Local telecom Folder>:/home/mapr/myprojects $PORTS -e MAPR_EXTERNAL -e clusterName -e isSecure --hostname ${clusterName} ${IMAGE} > /dev/null 2>&1
```

root user password: mapr
mapr user password: mapr


Several components need to be deployed in the Data Fabric server:

- Drill
- Spark
- OpenTSDB
- Kafka
- Data Access Gateway

```
apt-get install mapr-spark mapr-spark-historyserver mapr-data-access-gateway  mapr-kafka mapr-opentsdb
/opt/mapr/server/configure.sh -R

export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/opt/mapr/lib
So, madify the script ./mapr_devsandbox_container_setup.sh to authorize the network access - port mapping for the docker container
```

export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/opt/mapr/lib

# Create Data Fabric structures:

- Public Volume R/W   name: telecom  path:  /telecom
- Public JSON table   /telecom/basestations
- Public JSON table   /telecom/calls
- Public Stream       /telecom/mystream

# Install packages

```
apt-get update
apt-get install python3-pip

pip3 install pyspark==3.3.2
pip3 install importlib-metadata
pip3 install delta_spark==2.3.0
pip3 install avro
pip3 install requests
```
# Run PySpark with Delta Lake:
Download th delta lake library

```
wget https://repo1.maven.org/maven2/io/delta/delta-core_2.12/2.3.0/delta-core_2.12-2.3.0.jar
```

Connect with mapr/mapr into the Data Fabric Container:

to generate a mapr ticket for mapr/mapr (connected with mapri):

```
maprlogin password  
```


Pyspark with Delta:

	/opt/mapr/spark/spark-3.3.2/bin/pyspark --jars ~/delta-core_2.12-2.3.0.jar \
        --conf "spark.sql.extensions=io.delta.sql.DeltaSparkSessionExtension" \
        --conf "spark.sql.catalog.spark_catalog=org.apache.spark.sql.delta.catalog.DeltaCatalog"

	If running Delta 2.3, add delta-storage-2.3.0.jar 

![Application](./pictures/screenshot_0.png?raw=true "Application")


![Architecture](./pictures/screenshot_1.png?raw=true "Architecture")

# Insert Base station details:

## Into Delta table

```
/opt/mapr/spark/spark-3.3.2/bin/pyspark --jars ~/delta-core_2.12-2.3.0.jar < ./py/baseStationDelta.py
```

### Check insertion with Drill ( https://localhost:8047  user: mapr/mapr )

```
select * from dfs.`/telecom/basestationtable` limit 10
```

## Into MaprDB (JSON table)

```
/opt/mapr/spark/spark-3.3.2/bin/pyspark --jars ~/delta-core_2.12-2.3.0.jar < ./py/baseStationDB.py
```

### Check Insertion into JSON table:

```
select * from dfs.`/telecom/basestations` limit 10
```


# Inject calls into topic (check LD_LIBRARY_PATH):

```
python3  ./py/producer.py
```

# Insert Call:

## into Delta table (Parquet files):

```
/opt/mapr/spark/spark-3.3.2/bin/pyspark --jars ./delta-core_2.12-2.3.0.jar,./spark-avro_2.12-3.3.1.0-eep-910.jar,./spark-sql-kafka-0-10_2.12-3.3.1.0-eep-910.jar  < ./py/consumerCallDelta.py
```

### Check insertion with Drill:

```
select * from dfs.`/telecom/callTable` limit 10
```

## into Mapr DB table (Json Table):

```
/opt/mapr/spark/spark-3.3.2/bin/pyspark --jars ./delta-core_2.12-2.3.0.jar,./spark-avro_2.12-3.3.1.0-eep-910.jar,./spark-sql-kafka-0-10_2.12-3.3.1.0-eep-910.jar  < ./py/consumerCallDB.py 
```


### Check insertion with Drill:
 
```
select * from dfs.`/telecom/calls` limit 10
```

## Insert into OpenTSDB

```
/opt/mapr/spark/spark-3.3.2/bin/pyspark --jars ./delta-core_2.12-2.3.0.jar,./spark-avro_2.12-3.3.1.0-eep-910.jar,./spark-sql-kafka-0-10_2.12-3.3.1.0-eep-910.jar < ./py/opentsdb.py
```


## Join Delta lake tables (Calls & base stations)

```
/opt/mapr/spark/spark-3.3.2/bin/pyspark  --jars ./delta-core_2.12-2.3.0.jar  < ./py/joinCallBaseDelta.py
```
### Check merged data with drill:

```
select *  from dfs.`/telecom/staging_1` limit 10
```




## Run web UI

The front ennd is a nodeJs application located in myApp. It require to install nodeJs/npm.
The required packages are listed in package.json

self-signed certificate issue

export NODE_TLS_REJECT_UNAUTHORIZED=0

npm start
