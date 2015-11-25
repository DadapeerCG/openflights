# Titan 1.0 Data Migration

<img src="https://raw.githubusercontent.com/dkuppitz/openflights/master/docs/images/gremlin-plane.png" align="left" width="300">The release of [Titan 1.0](http://thinkaurelius.github.io/titan/) brought with it many important features, but it also came with the need to do a major code and data migration given the upgrade to [Apache TinkerPop 3](http://tinkerpop.apache.org) and the finalized changes to the schema used to store the graph data in Cassandra, HBase, etc. As a result, upgrading from older versions of Titan to this new version has its challenges.

This document and the associated code in this repository are designed to help with the data migration aspect of upgrading to Titan 1.0.  It provides a model for data migration by using the [OpenFlights](https://github.com/jpatokal/openflights) data set and is aimed at users looking to upgrade from Titan 0.5.4 to Titan 1.0.0. It presents methods for dealing with the hardest aspects of data migration in this context and it is meant to offer guidance for those migrating graph data of any size.

## Approach

This repository contains all the code required to simulate a data migration from Titan 0.5.4 to Titan 1.0.0. It provides Groovy scripts and Java code that together will generate the Titan 0.5.4 instance from the OpenFlights data set and then migrate it to Titan 1.0.0.  

## Prerequisites

* Java 1.8.0_40+ (required by TinkerPop 3.x)
* Maven 3.x
* Titan 0.5.4 and Titan 1.0.0
* Cassandra as [compatible](http://s3.thinkaurelius.com/docs/titan/1.0.0/version-compat.html) with the Titan versions above (while the tutorial focuses on Cassandra, there should be no reason that these steps will not also work for other Titan backends)
* Hadoop 2
* `git clone` this repository set an environment variable to the location of where it was cloned `export OPENFLIGHTS_HOME="/projects/openflights"`)`

## Assumptions

This tutorial assumes that the reader is knowledgeable with respect to Titan 0.5.x and has some basic knowledge of Titan 1.x and TinkerPop 3.x. 

# The Tutorial

For those wishing to get right to the abbreviated steps required to run this tutorial, they can be found at the [bottom](#for-the-impatient).

## The Schema

OpenFlights is an open data set containing [airport, airline and route data](http://openflights.org/data.html). While the dataset is not large, it bears sufficient complexity so as to make for a good model for a real-world data migration. Specifically, it allows for the modelling of [multi-properties](http://s3.thinkaurelius.com/docs/titan/1.0.0/schema.html#property-cardinality) and mixed indices containing [Geoshape](http://s3.thinkaurelius.com/docs/titan/1.0.0/search-predicates.html#_geoshape_data_type) data. 

## Loading Titan 0.5.4

Before the migration process can be started, the Titan 0.5.4 database needs to be loaded wth data.  The first step in the process is to download the OpenFlights source data which is in CSV format.  The [data/download.sh](https://github.com/dkuppitz/openflights/blob/master/data/download.sh) shell script will help with that:

```text
$ data/download.sh
--2015-11-25 07:20:28--  https://raw.githubusercontent.com/jpatokal/openflights/master/data/airports.dat
Resolving raw.githubusercontent.com (raw.githubusercontent.com)... 23.235.46.133
Connecting to raw.githubusercontent.com (raw.githubusercontent.com)|23.235.46.133|:443... connected.
HTTP request sent, awaiting response... 200 OK
...
Downloaded: 3 files, 3.4M in 1.2s (2.92 MB/s)
$ ls data
airlines.dat  airports.dat  download.sh  routes.dat
```

Note that Windows users can manually download these files to the `data` directory as the above script is meant for execution on Linux compatible systems.

The next step is to use the Titan 0.5.4 Gremlin Console to run the [scripts/load-openflights-tp2.groovy](https://github.com/dkuppitz/openflights/blob/master/scripts/load-openflights-tp2.groovy) script. This script will initialize the schema and parse the data into Titan. This script configures the Titan instance to load to through the [conf/openflights-tp2.properties](https://github.com/dkuppitz/openflights/blob/master/conf/openflights-tp2.properties).  This file defaults to local Cassandra usage, but could changed to support a remote Cassandra cluster, HBase or other Titan backend.

Start the Titan 0.5.4 Gremlin Console and execute the script:

```text
`gremlin> . ${OPENFLIGHTS_HOME}/scripts/load-openflights-tp2.groovy`
```

Depending on what the reader hopes to learn from this tutorial, there are two interesting points to possibly consider in this script:

* The `location` property on the `airport` vertex is a `Geoshape` data type.
* The `equipment` property on the `route` vertex is a `SET` multi-property.

With that script executed, Titan now has the graph generated, but that script did something else at the very end.  It dumped that data into HDFS using Titan's `HadoopGraph` as shown in the final lines of the script:

```groovy
g = HadoopGraph.open(PROJECT_DIR + "/conf/hadoop/openflights-tp2.properties")
g._()
```

The contents of [conf/hadoop/openflights-tp2.properties](https://github.com/dkuppitz/openflights/blob/master/conf/hadoop/openflights-tp2.properties) shows that it uses the `TitanCassandraInputFormat` to read the data from Cassandra and the `GraphSONOutputFormat` to write it to the file system. In this case, Hadoop's local job runner will write the output to the local file system. In a production environment, given a large scale graph migration, it would be preferred to have a Hadoop cluster with HDFS up and running for this output. As a side note, if this process is being executed against a different Titan backend (i.e. not Cassandra), then the input format should be changed to match the backend being used.

Now that the OpenFlights graph data is available on the filesystem as GraphSON, it becomes possible to read that format into Titan 1.0.0.

## Migrating to Titan 1.0.0

# For the Impatient

0. Clone the project and set an environment variable that holds the path to the projects (e.g. `export OPENFLIGHTS_HOME="/projects/openflights"`)
1. Download data files (run ${OPENFLIGHTS_HOME}/data/download.sh)
2. In a Titan 0.5.4 Gremlin console type (note that you cannot use `${OPENFLIGHTS_HOME}` for the console's load command, hence you'll have to replace it with the full path):

     `gremlin> . ${OPENFLIGHTS_HOME}/scripts/load-openflights-tp2.groovy`

3. Copy the output files (from the output/ directory in HDFS) into a Hadoop2 HDFS directory called openflights
4. upload the script ${OPENFLIGHTS_HOME}/scripts/openflights-script-input.groovy into Hadoop2's HDFS home directory
5. compile the project:

     `$ mvn clean install -DskipTests`

6. Copy the jar file ${OPENFLIGHTS_HOME}/target/openflights-1.0-SNAPSHOT.jar into ${TITAN_1_0_HOME}/ext/
5. In a Titan 1.0.0 Gremlin console:

     `gremlin> :load ${OPENFLIGHTS_HOME}/scripts/load-openflights-tp3.groovy`
