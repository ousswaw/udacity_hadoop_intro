# About

Here there are notes and tasks code for Cloudera / Udacity [hadoop course](http://cloudera.com/content/cloudera/en/training/courses/udacity/mapreduce.html).

# Notes

## Hadoop

- Big Data
  - Volume (big volume of data)
  - Variety (data comes in many different formats)
  - Velocity (the speed data comes in with)
  - It is good if we store all the data, for example if we have mp3 records of phone calls - we can make a text transcription, store and analyze it, but if we store the original mp3 as well then we have possibilities to get more data in the future (like analyze emotions and get extra data from that)

- Intro to Hadoop and MapReduce
  - Core
    - HDFS + MR (MapReduce)
  - Hive - SQL-like language on top of MR
  - Pig - simple scripting language on top of MR
  - Impala - SQL-like language on top of HDFS, direct access to HDFS, no MR, faster than Hive / Pig
    - Impala queries are optimized to run fast.
    - Impala was developed as a way to query your data with SQL, but which directly accesses the data in HDFS rather than needing map reduce.
    - Impala is optimized for low latency queries. In other words Impala queries run very quickly, typically many times faster than Hive, while Hive is optimized for running long batch processing jobs.
  - Sqoop - takes data from a traditional relational database, such as Microsoft SQL Server. And, puts it in HDFS, as the limited files. So, it can be processed along with other data on the cluster.
  - Flume - injests data as it's generated by external systems. And, again, puts it into the cluster.
  - HBase - a real time database, built on top of HDFS.
  - Hue - graphical front end to the questor.
  - Oozie - a workflow management tool.
  - Mahout - a machine learning library.
  - CDH - Cloudera distribution including Hadoop - setup of Hadoop + tools
  - More info
    - http://www.cloudera.com/content/cloudera/en/resources/library/training/introduction-to-apache-pig.html
    - http://www.cloudera.com/content/cloudera/en/resources/library/training/introduction-to-apache-hive.html
    - http://www.cloudera.com/content/cloudera/en/resources/library/training/intorduction-hbase-todd-lipcon.html
    - http://www.cloudera.com/content/cloudera/en/resources/library/training/an-introduction-to-impala.html
    - http://mahout.apache.org/

- HDFS
  - files are split into 64MB blocks (chunks)
  - blocks are stored on DataNodes (machnies with date)
  - each block is stored 3 times on different nodes
    - if DataNode goes down the NameNode notices this and re-replicates the data to have 3 copies again
  - metadata about files is stored on the NameNode
    - if metadata is lost due to disk failure - all the data on DataNodes is lost as well (we can not re-construct it from blocks)
    - the second NameNode can be configured as a standby replica of the active NameNode
  - hadoop fs - allows to view and manipulate the file system, works with unix-like commands
    - hadoop fs -ls - list files
    - hadoop fs -put filename.txt - add a file to hadoop fs
    - hadoop fs -mv filename.txt newname.txt
    - hadoop fs -rm newname.txt
    - hadoop fs -mkdir newdir
    - hadoop fs -put filename.txt newdir
    - hadoop fs -ls newdir
    - hadoop fs -tail filename.txt - show tail of the file
    - hadoop fs -cat filename.txt - show the whole file
    - hadoop fs -get filename.txt localname.txt - copy file to the local filesystem

- MapReduce
  - MapReduce process
    - file is broken into chunks and processed in parallel
      - initial chunks are passed to mappers, mappers pre-process data into intermediate records
        - all data is in form of key, value
      - by default chunks for MapReduce are HDFS blocks (chunks the original file was split to)
    - shuffle and sort stage
      - shuffle - movement of intermediate records from mappers to reducers
      - sort - reducers organize records they got
    - reducer works then on each set of sorted records, it gets a key and a list of values for that key
      - final results are stored back to HDFS
    - to get final results sorted - we can do an extra step or use only one reducer and get sorted data from it (one reducer doesn't scale well)
      - more info about partitioning is here (how data is spread among reducers) - http://developer.yahoo.com/hadoop/tutorial/module5.html#partitioning

Example: sales data for all stores for 2012 year

```
2012-01-01 Miami Clothes 50.14
2012-01-01 NewYork Food 12.12
2012-01-02 Miami Clothes 20.14
2012-01-02 NewYork Food 32.12
....
```

And we need:

```
Miami - 1111.11 (total amount)
New York - 2222.222
```

Mappers will extract data we need to process:

```
Miami 50.14
NewYork 12.12
Miami 30.14
NewYork 32.12
```

Here store name is a key and sales value is a value we need.

Mapper code:

```python
#!/usr/bin/python

# Format of each line is:
# date\ttime\tstore name\titem description\tcost\tmethod of payment
#
# We want elements 2 (store name) and 4 (cost)
# We need to write them out to standard output, separated by a tab

import sys

for line in sys.stdin:
    data = line.strip().split("\t")
    if len(data) == 6:
        date, time, store, item, cost, payment = data
        print "{0}\t{1}".format(store, cost)
```

After the sort and shuffle phase we will get sorted data like this:

```
Miami 50.14
Miami 30.14
NewYork 12.12
NewYork 32.12
```

Reducers calculate total amount for each store (one reducer can calculate more than one store):

```python
"#!/usr/bin/python

import sys

salesTotal = 0
oldKey = None

# Loop around the data
# It will be in the format key\tval
# Where key is the store name, val is the sale amount
#
# All the sales for a particular store will be presented,
# then the key will change and we'll be dealing with the next store

for line in sys.stdin:
    data_mapped = line.strip().split("\t")
    if len(data_mapped) != 2:
        # Something has gone wrong. Skip this line.
        continue

    thisKey, thisSale = data_mapped

    if oldKey and oldKey != thisKey:
        print oldKey, "\t", salesTotal
        oldKey = thisKey;
        salesTotal = 0

    oldKey = thisKey
    salesTotal += float(thisSale)

if oldKey != None:
    print oldKey, "\t", salesTotal
```

- MapReduce code
  - primary language is Java
  - the Hadoop Streaming allows to write in any other language (for example, Python)

- Patterns
  - Filtering patterns: Sampling, Top-N lists
    - Don't change the data
    - Simple filter - a function
    - [Bloom filter](http://en.wikipedia.org/wiki/Bloom_filter), [hadoop docs](http://hadoop.apache.org/docs/r2.4.1/api/org/apache/hadoop/util/bloom/BloomFilter.html)
    - Sampling - pull some sample of original data
    - Random sampling - pull some random sample
    - Top-N - top N records
      - Each mapper generate top N list
      - Reducer finds global top N
  - Summarization patterns: Counting, Min/Max, Statistics, Index
      - Inverted index
      - Numerical summarizations - counts, min/max, frist/last, mean/median
  - Structural patterns: Combining data sets

- Books
  - [Hadoop: The Definitive Guide](http://shop.oreilly.com/product/0636920033448.do)
  - [MapReduce design patterns](http://shop.oreilly.com/product/0636920025122.do)


# Tasks

Download and setup VM, as described in udacity's documents ([one](https://docs.google.com/document/d/1v0zGBZ6EHap-Smsr3x3sGGpDW-54m82kDpPKC2M6uiY/edit) and [two](https://docs.google.com/document/d/1MZ_rNxJhR4HCU1qJ2-w7xlk2MTHVqa9lnl_uj-zRkzk/pub)).

It is convenient to have ssh access to VM (details are also in udacity's docs) - change network type to bridget adapter, then run `ifconfig` inside the VM to get an IP address.
Having the IP it is possible to ssh to the VM like this:

```bash
ssh training@192.168.1.108
password: training
```

It is possible copy results back from VM to your local machine this way:

```
scp -r training@192.168.1.108:/home/training/udacity_training .
```

Also in the VM the following hadoop resources are available via browser:

- job tracker - http://localhost:50030/jobtracker.jsp
- name node status - http://localhost:50070/dfshealth.jsp

## The `hs` command

The `hs` command is a shortcut to run mapreduce job.
Actually `hs` is an alias to `run_mapreduce` function which looks like this:

```bash
$ type map_reduce

run_mapreduce() {
  hadoop jar /usr/lib/hadoop-2.0-mapreduce/contrib/streaming/hadoop-streaming-2.0.0-mr1-cdh4.1.1.jar -mapper $1 -reducer $2 -file $1 -file $2 -input $3 -output $4
}
```

## Add the input data for tasks to HDFS

The input data is in two fils under ~/udacity_training/data, here is how to load it into HDFS:

```
$ cd udacity_training/data
# Add purchases.txt
$ hadoop fs -put purchases.txt
# Check the file is present in HDFS
$ hadoop fs -ls
# Extract access_log and put it to HDFS
$ gunzip access_log.gz
$ hadoop fs -put access_log
$ hadoop fs -ls
```

## Run example

The initial example to process purchases data is under `~/udacity_training/code`, here is how to run it:

```
$ cd ~/udacity_training/code
$ hs mapper.py reducer.py purchases.txt output
# Check the result
$ hadoop fs -ls output
$ hadoop fs -cat output/part-00000 | less
# Get the result to local fs
$ hadoop fs -get output
$ ls
# Run mapper and reducer locally, without hadoop
#  run first 50 lines through mapper:
$ head -50 ../data/purchases.txt | ./mapper.py
#  run first 50 lines through mapper, sort and run through reducer:
$ head -50 ../data/purchases.txt | ./mapper.py | sort | ./reducer.py
#  process all data
$ cat ../data/purchases.txt | ./mapper.py | sort | ./reducer.py

## Work on tasks

```bash
$ cd ~/udacity_training
# Make new dir for a task
$ mkdir code_by_category
#
# Copy sample code
$ cp code/*.py code_by_category
#
# Change example code according to the task
$ cd code_by_category/
$ vim mapper.py
$ vim reducer.py
#
# Test
$ head -50 ../data/purchases.txt | ./mapper.py
$ head -50 ../data/purchases.txt | ./mapper.py | sort | ./reducer.py
#
# Run the MR job
$ hs mapper.py reducer.py purchases.txt output_by_category
$ hadoop fs -get output_by_category
#
# Check results
$ vim output_by_category/part-00000
```

## The file pathname task

The final task in the lesson 3 is to find most popular file pathname.
For some reason my answer was not the same as correct result accepted by udacity.

I tried two different methods to parse URLs (regexp and urlparse.urlparse) and here are five most popular file paths with hit counts:

```bash
/assets/css/printstyles.css     93XXX.0
/assets/img/home-logo.png       98XXX.0
/assets/js/javascript_combined.js       106XXX.0
/assets/css/combined.css        117XXX.0
/downloadSingle.php     141XXX.0
/displaytitle.php       263XXX.0
```

The most popular is "/displaytitle.php" with more than 263000 hits.
While the accepted answer is "/assets/css/combined.css" with 117XXX count (the third most-popular file).

To get the result data sorted by hit count I used the following command (it sorts the reducer's result by second column):

```bash
cat output_access_log_filepath_count/part-00000 | sort -k2n -t$'\t' | less
```