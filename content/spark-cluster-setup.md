This guide is for setting up `Spark 2.2.0` on AWS EC2.

* Spark 2.2.0 requires Java 8.X


# [Setting Up the Spark Cluster]
# [Spark Master]
# Download and install Spark

https://spark.apache.org/downloads.html

Then unpack and move the directory to your desired location.


In the ec2-user bash profile (/home/ec2-user/.bashrc), add the following:

```
export SPARK_HOME=<absolute path the installation of spark>

if [ -z $PATH ]
then
    export PATH=<$SPARK_HOME/python>:<$SPARK_HOME/python/lib>
else
    export PATH=$PATH:<$SPARK_HOME/python>:<$SPARK_HOME/python/lib>
fi

export AWS_ACCESS_KEY_ID=<your access key id>
export AWS_SECRET_ACCESS_KEY=<your secret access key>
```

where you replace all `<>` entries with actual values.
We also recommend you create `spark-defaults.conf` and `hive-site.xml` configuration
files in the `$SPARK_HOME/conf/` directory.

You will also need to add specify .jar files in order to use Derby (to create
DataFrames) and to read/write from S3.


# For Derby
You can use .jar files in the lib dir in your Derby installation (see section
for Derby installation).

# For S3
You can download the sdk .jar from:

https://aws.amazon.com/sdk-for-java/


Once you've created `spark-defaults.conf`, add:

spark.driver.extraClassPath      <path to dir with jars>/*:<path to derby installation>/lib/*
spark.executor.extraClassPath    <path to dir with jars>/*:<path to derby installation>/lib/*

if you've placed all .jar files in a single directory, otherwise, add as many entries
as needed.

* You can add explicit file paths for each .jar, but the config file supports `*`
expansion and will include all .jar files in the specified directory (though not
recursively).


Create `hive-site.xml` which should look like:

* This is sets Spark to use an external rather than embedded Derby server
when it creates DataFrames. If you do not have a `hive-site.xml` file,
Spark will default to running an embedded Derby server when DataFrames
are created.

```
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
    <property>
        <name>javax.jdo.option.ConnectionURL</name>
        <value>jdbc:derby://<derby network server host>:1527/metastore_db;create=true</value>
        <description>JDBC connect string for a JDBC metastore</description>
    </property>

    <property>
        <name>javax.jdo.option.ConnectionDriverName</name>
        <value>org.apache.derby.jdbc.ClientDriver</value>
        <description>Driver class name for a JDBC metastore</description>
    </property>
</configuration>
```

* Replace `<derby network server host>` with the appropriate value (`localhost`)
for the an instance running on the same server as the driver server (see below setup).

* Additional configuration for Google Cloud Storage can also be added to the
`hive-site.xml` file.


# Start up

We use the `rc.local` script that runs on startup as an entry point (to load ec2-user's
profile script, and then execute a Python script as ec2-user starting and orphaning
subprocesses as needed)

[1] Get private IP address of current server (if the current server is to be the master)
[2] Start master process



[Spark Slave]
# Download and install Spark

# Start up

[1] Get the private IP Address of the master server
[2] Get the processor count of the current slave server
[3] Start the process with the specified master IP and core count


# [AWS EC2 Security Groups]
The respective security group(s) for Master and Slave servers need to have access on
all ports to each other (worker processes on slaves will connect to the master
server on a random high number port).

* This is simplest if your entire cluster is in a single region (i.e. 'us-west-2'), as
multi-region clusters will not only suffer from increased latency, but also will require
you to generate ingress rules based on public IP addresses, rather than security groups.
The start up processes will also need to change (from using private IP addresses to
public).


# [Driver Server Config]
If you are using a server that is in your cluster, you will also have install and
configure Spark on that server. Oddly enough, paths to .jar dependencies on the 
cluster machines are actually specified by configuration in the the driver server
(the server actually running the client code for your Spark application).

It is also recommended to install Derby and have it running in network server mode
on your driver server (this is used to store metadata about DataFrames in Spark,
which is managed by the driver application). If Spark is not configured to use an
external Derby Network Server, it will by default start an embedded instance.
This will, however, prevent any other process on a server from being able to use
DataFrames while application code is running.



# [Installing Derby]
# Download and install Derby 10.14.1.1

https://db.apache.org/derby/releases/release-10.14.1.0.cgi

Then unpack and move the directory to your desired location.


# Add Java policy to allow Derby to bind to port 1527
In your `$JAVA_HOME\jre/lib/security/java.policy` (where `$JAVA_HOME is the location of your Java installation),
add:

grant {
        permission java.net.SocketPermission "<host>:1527", "listen";
};

where <host> is replaced with the appropriate value (localhost if you intend to
run Derby Network Server on the same server as the Spark driver application).


* The Derby Network Server needs to have started before the driver server can create
DataFrames. The driver server will also need to have access to the Derby Network
Server (no additional configuration needed if the Derby Network Server is running
locally, which we recommend). Overhead for running Derby Network Server is trivial.
