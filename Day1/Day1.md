# Sqoop workshop

* Task 1: Extract data from an Oracle database, and load it into Hive

In order to load data from a releational database, and store it in hdfs, you can use the commandline tool sqoop.
Before sqoop can run, you need to make sure that the jdbc driver for the database is available. 
In this example, we needed to load from an oracle relational database. 
I downloaded the latest ojdbc.jar jdbc library from Oracle, uploaded nodes in the cluster, and stored it in /var/lib/sqoop/ directory
on all the nodes of the cluster.
The used command for this was:

```
sudo cp /home/ec2-user/ojdbc6.jar /var/lib/sqoop/
```

Now the jdbc driver is present, I managed to define a sqoop job that loads data in hdfs, and directly register it in Hive.
The command to use this on a small (dimensional) table was:

```
sqoop import --connect jdbc:oracle:thin:@gravity.cghfmcr8k3ia.us-west-2.rds.amazonaws.com:15210:gravity --username "gravity" --password "bootcamp" --table 'DETECTORS' --target-dir "/user/ec2-user/DETECTORS" --hive-import -m 1
```

Similar commands were created for the other dimensional tables.

---

* Task 2: Load the measure table, and manually create the hive definition

I had several issues loading the measures table. The first had to do with the source database. The Oracle database was far to busy to 
respond quickly. Also, on my cluster, there were far too many jobs running (I see a lot of Spark tasks funny enough) eating up all the
resources of the machine. That slowed down my testing significantly.

In order to get things workable, I decided to continue to work with a smaller measures table, taking a 1% sample from the source.

I used a sqoop statement to read the 1% sample from the Oracle measures table, using direct mode, and store it in hdfs.
The command used for that was:

```
sqoop import --connect jdbc:oracle:thin:@gravity.cghfmcr8k3ia.us-west-2.rds.amazonaws.com:15210:gravity --username gravity --password bootcamp --target-dir /user/ec2-user/andre/galaxy/measurements --direct --query 'select * from MEASUREMENTS SAMPLE(1) WHERE $CONDITIONS' --split-by 'detector_id' --num-mappers 4
```

The trick of this sqoop command is to parse a --query instead of a --table option. This allows me to specify the query myself.
And by adding the sample(1) to the query, the Oracle DB will give me a random sample back. 
This helped decreasing the number of resources, and completed in about 10 minutes.

When the file was stored on hdfs, I used the Hue hive interface to create a table definition for it. The sql I used for this statment was:

```
CREATE external TABLE andre.measurements (
 measurement_id STRING,
 detector_id INT,
 galaxy_id INT,
 astrophysicist_id INT,
 measurement_time BIGINT,
 amplitude_1 DECIMAL(7,4),
 amplitude_2 DECIMAL(7,4),
 amplitude_3 DECIMAL(7,4),
 mt_time BIGINT
)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ',' LINES TERMINATED BY '\n' 
STORED AS TEXTFILE LOCATION 'hdfs://ip-172-31-36-53.us-west-2.compute.internal:8020/user/ec2-user/andre/galaxy/measurements'
TBLPROPERTIES ("serialization.null.format"="null");
```

By using this method, I have control of the data type the attributes get (e,g, int or decimal). Also I can specify in which hive database
the table will be stored.

---

* Step 3: Find the gravity waves in the measures table.

Since time was up, I could only run the select statement directly on the measures table.
I used the following command to detect the gravity waves from the table:

```
select * from andre.measurements where amplitude_1 > 0.995 and amplitude_2 < 0.005 and amplitude_3 > 0.995;
```

It took about 5 minutes to return 1 reacord.
