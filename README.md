# Big Data ecosystem

Hadoop tutorial inside the [Cloudera quickstart VM](https://downloads.cloudera.com/demo_vm/virtualbox/cloudera-quickstart-vm-5.13.0-0-virtualbox.zip).

## Prerequisites

#### Importing the Virtual Machine

Before working on the tutorial, we need a working Hadoop cluster.

For the following exercises, we will use the [Cloudera quickstart VM](https://downloads.cloudera.com/demo_vm/virtualbox/cloudera-quickstart-vm-5.13.0-0-virtualbox.zip).

* Download and unzip the applicance for VirtualBox.

* Import the `.ova` file into VirtualBox. **Don't start it yet** if you want to configure it.

  * You may configure the VM to use at least 6-8 Go of RAM, through the `Configuration > System` view. 

* Start the virtual machine with the green arrow.


The virtual machine should start and you should have access to a window. Do take note that the machine virtual by default is in QWERTY.

* Open a terminal and type `setxkbmap fr` to temporarily change to AZERTY.
* To [enable copy/paste](https://www.techrepublic.com/article/how-to-enable-copy-and-paste-in-virtualbox/), select `Devices > Shared Clipboard > Bidirectional`.

For the first part of the tutorial, we will interact with a [trucks geolocation dataset from the Cloudera tutorial](https://www.cloudera.com/tutorials/getting-started-with-hdp-sandbox.html). 



##  TD1 - Hadoop, MapReduce, Pig, Hive

For today, most of our interactions to the Hadoop cluster will be done through [Hue](https://docs.cloudera.com/documentation/enterprise/5-13-x/topics/hue.html).

* Open Firefox : open a terminal and type `firefox` or `Applications > System tools > Web browser`
* Open the `Hue` tab. This will be our main UI for interacting with HDFS today. 
  * The credentials are **cloudera/cloudera**.



####  1. First steps on HDFS 

* We need to download and unzip the [Geolocation data from Cloudera](https://www.cloudera.com/content/dam/www/marketing/tutorials/getting-started-with-hdp-sandbox/assets/datasets/Geolocation.zip). 
  * You can do it from Firefox, download it to a folder in the virtual machine.
  * You can also `wget https://www.cloudera.com/content/dam/www/marketing/tutorials/getting-started-with-hdp-sandbox/assets/datasets/Geolocation.zip` and `unzip Geolocation.zip`.
* In Hue, select `Browsers > Files`. 
* Create a new directory in HDFS called `data` inside HDFS from Hue.
  * By default this should be created under `hdfs:///user/cloudera/`.
* Upload `Geolocation.csv` and `trucks.csv` into your newly created `data/` folder.



#### 2. HDFS CLI

* Open a terminal. 
* You can access the `hdfs` command from the terminal. This should output the help from the command line.
* Display the version of HDFS with `hdfs version`.

The `hdfs dfs` command gives you access to all commands to interact with files in HDFS.

* List all folders inside HDFS with `hdfs dfs -ls`. 
  * If you only type `hdfs dfs -ls` it will naturally take you to `hdfs:///user/cloudera`.  You should find your geolocation files.
  * By default if you don't put an absolute HDFS path, HDFS will start from your HDFS home `hdfs:///user/cloudera`.
* Rename the `hdfs:///user/cloudera/data` folder to `hdfs:///user/cloudera/geoloc` with `hdfs dfs -mv` . 
  * Remember by default that `hdfs dfs -mv data geoloc` is equivalent to `hdfs dfs -mv hdfs:///user/cloudera/data hdfs:///user/cloudera/geoloc`.
* Create a folder in HDFS named `test`. 
* To copy files from your local machine to HDFS, there is `hdfs dfs -copyFromLocal <local_file> <path_in_HDFS>`. Try to copy  your `geolocation.csv` file from the locall VM into the `test` HDFS folder.
* Currently, the `hdfs:///tmp` folder doesn't have permissions for everyone to write in. 
  In the Hadoop ecosystem, `root` is not the superuser but `hdfs` is. So we need to be in the `hdfs` user before running set permissions. Run the following script.

```
sudo su -
su hdfs
hdfs dfs -chmod -R 777 /tmp
```



#### 3. Map Reduce 

Now that storage is over, time to compute stuff.

* Run the Pi example : `yarn jar /usr/lib/hadoop-mapreduce/hadoop-mapreduce-examples.jar pi 4 100`.  Observe the flow. You can check all logs in Firefox, in the `Hadoop > YARN Resource Manager` tab.

Now we want to run a wordcount on a file inside HDFS, let's run it on files inside `hdfs:///user/cloudera/geoloc/`. 

* Run `yarn jar /usr/lib/hadoop-mapreduce/hadoop-mapreduce-examples.jar wordcount 
  geoloc/geolocation.csv output`.
  * The command may file if `hdfs:///user/cloudera/output` already exists, in that case remove the folder with `hdfs dfs -rm -r -f output`.
* Examine the `hdfs:///user/cloudera/output` folder. You can use `hdfs dfs -ls output` and `hdfs dfs -cat output/part-r-00000`. Explain
  * Also see the logs in a new tab to the favorite `Hadoop > YARN Resource Manager` in Firefox. Explain.



#### 4. Pig

Pig is an extension to compile Pig Latin, a specific language, into MapReduce scripts.

The following can be run in two ways :

1. Interactively through the Terminal, by launching a Pig shell with `pig `and run commands one by one.
2. In Hue, you can go to the Pig Editor via `Query > Editor > Pig`. This is the preferred method if you want to run full scripts but is much longer to run than the Pig shell.

*  Run the following script/commands to load and display the first ten lines from the geolocation file :

```
geoloc = LOAD 'geoloc/geolocation.csv' USING PigStorage(',') AS (truckid,driverid,event,latitude,longitude,city,state,velocity,event_ind,idling_ind);

geoloc_limit = LIMIT geoloc 10;

DUMP geoloc_limit;
```

* Let's try to compute some stats on this file. 

```
geoloc = LOAD 'geoloc/geolocation.csv' USING PigStorage(',') AS (truckid:chararray, driverid:chararray, event:chararray, latitude:double, longitude:double, city:chararray, state:chararray, velocity:double, event_ind:long, idling_ind:long);

truck_ids = GROUP geoloc BY truckid;

result = FOREACH truck_ids GENERATE group AS truckid, COUNT(geoloc) as count;

STORE result INTO 'results';

DUMP result;
```

* Analysis :
  * Check the `results` folder stored in HDFS by the `STORE result` line. What can you say compared to the MapReduce wordcount ?
  * Also see the logs in a new tab to the favorite `Hadoop > YARN Resource Manager` in Firefox. Explain.
* Are you able to count the list of distinct cities visited per truckid, and mean velocity per truckid ?



#### 5. Hive

Like for Pig, you have the choice between the `hive` command in the Terminal, or in Hue go to the Hive Editor via `Query > Editor > Hive`.

* If using the Terminal, create an external table for the `geoloc` folder which contains geolocation.csv. Maybe you should also move the `trucks.csv` file outside of the `geoloc` folder with `hdfs dfs -mv geoloc/trucks.csv`. The following command creates a Hive table pointing to a HDFS location. You can drop it, it won't destroy the data in HDFS : 

```sql
CREATE EXTERNAL TABLE geolocation (truckid STRING, driverid STRING, event STRING, latitude DOUBLE, longitude DOUBLE, city STRING, state STRING, velocity DOUBLE, event_ind BIGINT, idling_ind BIGINT)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
LOCATION '/user/cloudera/geoloc';
```

```sql
SELECT truckid FROM geolocation LIMIT 10;
```

* Or with Hue, on the left select SQL, click on the + button, well you will have to look by yourself a little bit :)...



* Are you able to count the list of distinct cities visited per truckid, and mean velocity per truckid ?

  



## TD2 - Hue and Zeppelin

You can find the [Hue documentation](https://docs.cloudera.com/documentation/enterprise/5-13-x/topics/hue.html).

Today we will analyze some HTML files by web scraping some sites and using Hue/Zeppelin to chart.



## TD3 - HBase





## Appendix

### Cluster installation

For the more adventurous one, there are other ways to install a Hadoop cluster :

* Download the [Hortonworks](https://www.cloudera.com/downloads/hortonworks-sandbox.html) sandbox. https://archive.cloudera.com/hwx-sandbox/hdp/hdp-3.0.1/HDP_3.0.1_virtualbox_181205.ova
* Using [Ambari](https://cwiki.apache.org/confluence/display/AMBARI/Quick+Start+Guide)



#### Setup Ambari cluster through Vagrant

```
git clone https://github.com/u39kun/ambari-vagrant.git
cat ambari-vagrant/append-to-etc-hosts.txt >> C:\Windows\System32\drivers\etc\hosts
cd ambari-vagrant
cd centos6.8 # ubuntu18.4

cp ~/.vagrant.d/insecure_private_key .

vagrant up c6801 # u1801
vagrant up c6802 # u1802
vagrant up c6803 # u1803

vagrant ssh c6801 # u1801
sudo su -
wget -O /etc/yum.repos.d/ambari.repo http://public-repo-1.hortonworks.com/ambari/centos6/2.x/updates/2.6.2.0/ambari.repo
yum install ambari-server -y

# wget -O /etc/apt/sources.list.d/ambari.list http://public-repo-1.hortonworks.com/ambari/ubuntu18/2.x/updates/2.7.4.0/ambari.list
# apt-key adv --recv-keys --keyserver hkp://keyserver.ubuntu.com:80 B9733A7A07513CAD
# apt-get update
# apt-get install ambari-server -y

ambari-server setup -s
ambari-server start
```

Once Ambari Server is started, open 8080 on Virtualbox c6801 then hit http://localhost:8080 from your browser on your local computer.

Note that Ambari Server can take some time to fully come up and ready  to accept connections. Keep hitting the URL until you get the login  page. 

Once you are at the login page, login with the default username **admin** and password **admin**. 

###### Create a Cluster

Select HDP 3.1.4

On the Install Options page, use the FQDNs of the VMs. For example:

```
c6801.ambari.apache.org
c6802.ambari.apache.org
c6803.ambari.apache.org
```

Specify the the non-root SSH user **vagrant**, and upload **insecure_private_key** file that you copied earlier as the private key.

Follow the on-screen instructions to install your cluster. Just Install HDFS, YARN, Tez, Hive, Zookeeper,  Kafka, Spark, Spark2, Zeppelin, Ambari Metrics

Let defaults everywhere. Where you need to configure services put passwords everywhere.

 **vagrant suspend** and  **vagrant resume** when over.

When done testing, run **vagrant destroy -f** to purge the VMs.

**Recommendations**

If you would like to save state for a period of time and you plan to stop using your Mac during that time, if you sleep your Mac the cluster  should continue from where it left off after you wake the Mac. 

When stopping a set of VMs--if you don't need to save cluster state--it can  be helpful to stop all services first, stop ambari-server (`ambari-server stop`), and then issue a Vagrant `halt` or `suspend` command.

When restarting a cluster after halting or taking a snapshot, check Ambari server status and restart it if necessary:

```
ambari-server status   
ambari-server start
```

After logging into the Ambari Web UI, expect to see alert warnings or errors  due to timeout conditions. Check the associated messages to determine  whether they might affect your use of the virtual cluster. If so, it can be helpful to stop and restart one or more associated components. 

