# Gobblin-PNDA

This is the Gobblin job for pulling data from Kafka topic to HDFS Datasets. Should be done at `cdh-edge` on PNDA cluster with `root` user.

## Quick Start

### Compile Source Code

```bash
mvn install
```

Then you will get `target/gobblin-pnda-1.0-SNAPSHOT.jar`.

### Job Configuration

All the `Jobs` should be claimed in `src/main/java/resources/job-conf/` by `*.pull` or `*.job` files.

In the config file:

- `source.schema` is the Avro schema of kafka topic.
- `kite.writer.dataset.uri` is the dataset URI where the data will be stored. (See **#Kite Dataset**)
- `topic.whitelist` assign the topic to be listened.

### Kite Dataset

Create the dataset in HDFS using Kite-CLI to store kafka data, then set the Hive dataset mapping from HDFS as well:

```bash
kite-dataset create dataset:hdfs://10.0.1.63:8020/user/pnda/PNDA_datasets/datasets/kafka/depa_raw --partition-by partition.json --schema sensorRecord.avsc
kite-dataset create depa_raw --location hdfs://10.0.1.63:8020/user/pnda/PNDA_datasets/datasets/kafka/depa_raw
```

### Crontab Job

```bash
#List the existing cron jobs:
crontab -l

#Edit cron jobs:
crontab -e
```

crete gobblin job executed every 5 minutes:

```bash
*/5 * * * * /sbin/start gobblin
```

scheduler using [Quartz](http://www.quartz-scheduler.org/documentation/quartz-2.2.x/examples/Example3.html).

Then edit the `/etc/init/gobblin.conf`:

```bash
description     "Linked-in Gobblin MRv2 application: PNDA pull"

task

umask 022
setuid gobblin

env JAVA_HOME="/usr/lib/jvm/java-8-oracle/"
env HADOOP_BIN_DIR="/opt/cloudera/parcels/CDH/bin"

chdir /home/gobblin/gobblin/gobblin-dist

script
    bash ./bin/gobblin-mapreduce.sh --conf /home/ubuntu/opt/gobblin-pnda/src/main/resources/job-conf/depa-hdfs.pull --workdir "/user/gobblin/work" --jars $(ls lib/*.jar | grep -v -E '(hive-exec|hadoop)' | tr '\n' ',')/home/ubuntu/opt/gobblin-pnda/target/gobblin-pnda-1.0-SNAPSHOT.jar
    hive -e "MSCK REPAIR TABLE depa_raw;"
end scrip
```

- `--conf` should assign the file in **#Job Configuaration**
- `--jars` should add the file in **#Compile Source Code**
- `hive -e` means to execute query in command line, this is for updating the partition from HDFS to Hive.

Now the MapReduce job will run as scheduled to pull data from kafka to HDFS and map to Hive Table.