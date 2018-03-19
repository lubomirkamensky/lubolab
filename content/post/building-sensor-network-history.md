+++
date = "2018-03-14T03:57:28+02:00"
draft = false
title = "Building Sensor Network: History layer using InfluxData"

tags = [ "Personal DWH", "InfluxData"]
+++

### History Layer
There are 5 access data layers introduced in my elder post [Building Sensor Network: with BigClown](/building-sensor-network-bc/): 
Current, History, Integration, Machine Learning, Reporting. And we have already utilized the first one and the last one in the post [IoT Doorbell: with BigClown](/iot-doorbell-bc/). Now it is the time to look into the History layer.  

### Time series database
Storing, maintaining and accessing historical data is always expensive, so it is worth spending some time on finding the most effective way how to cope with that. 

A classical relational database is a powerful tool for organizing any kind of relational data. In my first post, I described how to build both detail and aggregate layer using classical SQL database, see [Weather Data Mart](/weather-data-mart/). 

Meteorological data is a perfect example of Time series data. It is data provided by some kind of sensors during a specific period of time. If we remove the time dimension, the data is pretty small, perhaps only sensor type, quantity name, and unit of measure. For one weather station just a few records. What makes the data big is really just the time dimension, considering measurement every second it makes 31.5 million records just for one quantity in 10 years. 

Yes, it is a quite specific type of data so using classical SQL database may not be the most effective way of processing them. A specialized tool can be better here. And of course, it is.  

### InfluxData
Time series database like InfluxData focuses just on one type of data and it is extremely effective in processing it. Of course, this extra effectivity comes with other big limitations. It definitely cannot replace SQL database in general but that's not the goal. And as the technology is still young, even some expected features are still missing. For example, I would expect the support of calendar month and year, but so far only symmetric time units like seconds, minutes, hours and days are supported.

But let's look into InfluxDb more closely.

### Data Model
This is how the database is defined in the InfluxDB
```
CREATE DATABASE "landing" WITH DURATION 30d REPLICATION 1 SHARD DURATION 1d NAME "one_month" 
```

The purpose of this database is to store the data exactly as they come from the sensors. DURATION specifies how long the data is supposed to be stored in the database. Then it is automatically removed.  SHARD DURATION specifies the minimal data block which can be removed. So to summarize,  we keep data here for 30 days and everyday is one day of the history removed. This set of rules is called the Retention policy and we give it a name "one_month". There can be more of them within one database.

There are no tables like in SQL databases in the InfluxData. There is data structure called Measurement. In fact, you can store very different data in one Measurement structure, even with distinct primary keys but the concept of primary key is anyway never used in the world of time series databases.

<b>I decided to create two Measurements in the database, additive and nonadditive, grouping quantities with the same way of aggregation.
Additive quantities are aggregated using SUM, Nonadditive using AVG, MIN, MAX.</b>

Withing the Measurement we define some sort of indexed dimensions called tags and the facts called fields. Both are defined as a key, value pairs.

<b>Measurements in Landing database have only one field named value. 
And I use four tags: quantity, subject, sensor, UOM.</b>

<div class="c_alert c_alert-success"><i class="fa fa-info-circle"></i>
The tags and fields aren't predefined for the Measurement, this definition is part of any loaded data. So there can be data using various tags within one Measurement. And I mean both sets of tag values and tag keys can different.
</div>

Well, coming from the SQL database world this can be a bit hard to grasp. Measurement is not really like a physical table or logical entity in relational databases.

<div class="c_alert c_alert-success"><i class="fa fa-exclamation-triangle"></i> 
In fact, the essential component of SQL data modeling, the concept of a table (or entity) doesn't exist in the Time series database universe at all.
</div>

But the instance of the entity (or record of the table, if you wish) plays the main role. It is called the Data point here and is identified by Measurement, timestamp, and set of tag key, values pairs. 

Collection of data points, sharing Measurement and set of tag key, values pairs, is called a Series. Series represents the history of a data point.  If you think the Series is something like table or entity, then you are wrong again. Imagine you have a Customer table in SQL database storing the history of all your customers, then the Series is the history of just one particular customer, not all of them. Really the concept of table or entity is not used in the world of Time series databases.

The case of Customer table is also a good example why the time series databases cannot be used for any kind of historical data. While for few sensors we have millions of measurements,  the situation with Customers can be quite the opposite. We can have millions of customers and collect new data for them just once in a month. 

<div class="c_alert c_alert-success"><i class="fa fa-exclamation-triangle"></i>
The shape of data is extremely important for Time series databases. Everything is focused on extremely fast access to the latest data and effective storage of the history. The logical approach is to keep the latest data in memory and the history on the drive. Now imagine what would happen with our Customer example, the in-memory part would just explode.
</div>

### Series in practice
But enough of the theory, it is the time to dive into the practice. We have defined database, retention policy, measurement and let's assume we have already loaded the data composed of tag and field sets.

What series do we have in the database?

```
use landing
show series
```

Here it is. Loaded data is the result of Integration of [Neurio](/building-sensor-network-neurio/), [Futura](/building-sensor-network-futura/), the rest are [BigClown sensors](/building-sensor-network-bc/), all described in linked posts. Data is loaded using [bch-mqtt2influxdb](https://github.com/bigclownlabs/bch-mqtt2influxdb) and my latest configuration looks like [this](https://github.com/lubomirkamensky/lubolab_src/blob/master/mqtt2influxdb.yml)

Each row represents one Series and it nicely shows how the series key is composed. Adding the timestamp you can get the data point identification.

```
key
---
additive,quantity=event_count,sensor=carpinus,subject=push
additive,quantity=event_count,sensor=pinus,subject=pir
nonadditive,quantity=concentration,sensor=quercus,subject=co2,uom=1ppm
nonadditive,quantity=consumption,sensor=futura,subject=total,uom=1W
nonadditive,quantity=consumption,sensor=neurio,subject=phase_A,uom=1W
nonadditive,quantity=consumption,sensor=neurio,subject=phase_B,uom=1W
nonadditive,quantity=consumption,sensor=neurio,subject=phase_C,uom=1W
nonadditive,quantity=consumption,sensor=neurio,subject=total,uom=1W
nonadditive,quantity=flow,sensor=futura,subject=air,uom=m3/h
nonadditive,quantity=humidity,sensor=futura,subject=ambient,uom=0.1%
nonadditive,quantity=humidity,sensor=futura,subject=fresh,uom=0.1%
nonadditive,quantity=humidity,sensor=futura,subject=indoor,uom=0.1%
nonadditive,quantity=humidity,sensor=futura,subject=waste,uom=0.1%
nonadditive,quantity=humidity,sensor=quercus,subject=default,uom=1%
nonadditive,quantity=humidity,sensor=salix,subject=default,uom=1%
nonadditive,quantity=humidity,sensor=ulmus,subject=default,uom=1%
nonadditive,quantity=illuminance,sensor=salix,subject=default,uom=1lux
nonadditive,quantity=illuminance,sensor=ulmus,subject=default,uom=1lux
nonadditive,quantity=pressure,sensor=quercus,subject=default,uom=1kPa
nonadditive,quantity=pressure,sensor=salix,subject=default,uom=1kPa
nonadditive,quantity=pressure,sensor=ulmus,subject=default,uom=1kPa
nonadditive,quantity=recovery,sensor=futura,subject=heat,uom=1W
nonadditive,quantity=temperature,sensor=carpinus,subject=default,uom=1°C
nonadditive,quantity=temperature,sensor=fraxinus,subject=default,uom=1°C
nonadditive,quantity=temperature,sensor=futura,subject=ambient,uom=0.1°C
nonadditive,quantity=temperature,sensor=futura,subject=fresh,uom=0.1°C
nonadditive,quantity=temperature,sensor=futura,subject=indoor,uom=0.1°C
nonadditive,quantity=temperature,sensor=futura,subject=waste,uom=0.1°C
nonadditive,quantity=temperature,sensor=pinus,subject=default,uom=1°C
nonadditive,quantity=temperature,sensor=quercus,subject=default,uom=1°C
nonadditive,quantity=temperature,sensor=salix,subject=default,uom=1°C
nonadditive,quantity=temperature,sensor=ulmus,subject=default,uom=1°C
nonadditive,quantity=voltage,sensor=carpinus,subject=default,uom=1V
nonadditive,quantity=voltage,sensor=pinus,subject=default,uom=1V
nonadditive,quantity=voltage,sensor=quercus,subject=default,uom=1V
nonadditive,quantity=voltage,sensor=salix,subject=default,uom=1V
nonadditive,quantity=voltage,sensor=ulmus,subject=default,uom=1V
nonadditive,quantity=wear_level,sensor=futura,subject=filter,uom=1%
```

All listed series use a fixed number of tags. Even for sensors where the subject tag is not really applicable, the value "default" is used. This approach will simplify the reporting in Grafana which I am going to describe in the next post.

Using this approach, despite everything I have written about the time series databases, logically I can consider all the series belonging to one supertype entity, sharing the same primary key. Well, it is hard to forget this RDBMS concept. Another way of looking into this is that all series inherits its properties from one common ancestor. 

I think such design is not only nicely symmetric but it significantly simplifies the overall data flow allowing to use the power of abstraction.

### Data History
Well, we have defined our retention policy for the landing database but are we really just going to drop the elder history and lose it? No, definitely not. The question is whether we really need the very detailed information as it comes from the sensors or we are happy with aggregated data for an elder history.

To be honest, I want to keep all details for unlimited history but not necessarily easily accessible. I require easy access to the latest detail data and to aggregated history. The detailed history can be just archived in some well-compressed way and ready to be loaded into a database if needed.

### Data Normalization
Before the aggregation itself, it is necessary to make it clear what data we receive from the sensors. The specific case is the event data where we receive data anytime  the event happens.  Then there are measurements of various quantities. Some sensors provides continuous measurement and send data anytime the measured value changes. Of course the "changes" always depends on the sensor sensitivity and required measurement precision, so in fact, there is never a true continuous measurement of any quantity. Other sensors provide measurements in some regular intervals and we have no clue what happens in between them.

Considering all this it is a good idea to somehow normalize data before we start the aggregation. Let's create a new database for that:

```
CREATE DATABASE "base" WITH DURATION 30d REPLICATION 1 SHARD DURATION 1d NAME "one_month"
```

And we will load this database using Continuous queries. There is one query for  additive measurement.

```
CREATE CONTINUOUS QUERY "additive_base" ON "landing" BEGIN SELECT COUNT("value") as value INTO "base"."one_month"."additive" FROM additive GROUP BY quantity, sensor, subject, uom, time(1s) END
```

It is defined on landing database, the most important is the GROUP BY section, defining the granularity of new data in the base database. We keep all the tag keys and define 1s as the time grain. If not specified in other parts of the query this time grain also specifies the frequency of query execution.  We use COUNT function to get value 1 for any event which will be later aggregated using SUM function. As result, we get data points only when there is an event in the landing database.


Now let's look at the processing of the nonadditive measurement. Here we create two continuous queries grouping the sensors into slow and fast groups. Neurio and Futura sensors provide measurements more frequently than the BigClown sensors.

```
CREATE CONTINUOUS QUERY "nonadditive_base_fast" ON "landing" RESAMPLE EVERY 11m FOR 11m BEGIN SELECT MEAN("value") as value INTO "base"."one_month"."nonadditive" FROM nonadditive WHERE quantity != 'voltage' AND (sensor = 'neurio' OR sensor = 'futura') GROUP BY quantity, sensor, subject, uom, time(1s) FILL(linear) END

CREATE CONTINUOUS QUERY "nonadditive_base_slow" ON "landing" RESAMPLE EVERY 10m FOR 40m BEGIN SELECT MEAN("value") as value INTO "base"."one_month"."nonadditive" FROM nonadditive WHERE quantity != 'voltage' AND sensor != 'neurio' AND sensor != 'futura' GROUP BY quantity, sensor, subject, uom, time(1s) FILL(linear) END
```

The GROUP BY section is the same as for additive measurement but there are some new sections. For the nonadditive quantities, we are going to normalize data using linear extrapolation filling the space between the measurements with derived values. All that is done using two words FILL(linear). To make it work as expected we have to force the execution interval and data scope processed using the RESAMPLE section. EVERY 10m defines the execution interval and FOR 40m defines the scope of data being recalculated. So every ten minutes the period of last 40 minutes is recalculated.  

The reason why to recalculate the past interval is that for the linear extrapolation of any period of data we need the value at both interval ends. But as we are not receiving all measurements every second, some intervals stay open and the values cannot be derived until the closing value arrives. 

### Testing
Below is a simple test checking the number of data points within a day to verify that the Linear fill was successful and there are no gaps in the timeline.

```
influx -database 'base' -precision 'rfc3339' -execute 'select count(*) from nonadditive group by quantity, sensor, subject, uom, time(24h) order by time' > test/nonadditive.txt
```

### Data Aggregation
Once the data is normalized it is easy to aggregate it. Let's create another database for that.

```
CREATE DATABASE "agg_hour" WITH DURATION INF REPLICATION 1 SHARD DURATION 1000w NAME "infinite"
```

This database is going to provide the aggregated history which is supposed to stay there forever so we define the infinite duration in the Retention policy.


Following continuous query is creating hourly aggregate data for an additive measurement using the SUM function.

```
CREATE CONTINUOUS QUERY "additive_agg_hour" ON "base" RESAMPLE FOR 2h BEGIN SELECT SUM("value") as sum_value INTO "agg_hour"."infinite"."additive" FROM additive GROUP BY quantity, sensor, subject, uom, time(1h) END
```

Next continuous query is creating hourly aggregate data for a nonadditive measurement using the MEAN, MIN, MAX, MEDIAN and LAST functions.

```
CREATE CONTINUOUS QUERY "nonadditive_agg_hour" ON "base" RESAMPLE FOR 2h BEGIN SELECT MEAN("value") as avg_value, MIN("value") as min_value, MAX("value") as max_value, MEDIAN("value") as mdn_value, LAST("value") as last_value INTO "agg_hour"."infinite"."nonadditive" FROM nonadditive GROUP BY quantity, sensor, subject, uom, time(1h) END
```


And finally the last one. Quantity voltage was excluded from the normalization in the base database as it is changing slowly and the normalization would not have sense. The hourly aggregate data is directly derived from the landing database. Another change is using previous value instead of linear extrapolation.

```
CREATE CONTINUOUS QUERY "voltage_agg_hour" ON "landing" RESAMPLE EVERY 1h FOR 3h BEGIN SELECT MEAN("value") as avg_value, MIN("value") as min_value, MAX("value") as max_value, MEDIAN("value") as mdn_value, LAST("value") as last_value INTO "agg_hour"."infinite"."nonadditive" FROM nonadditive WHERE quantity = 'voltage' GROUP BY quantity, sensor, subject, uom, time(1h) FILL(previous) END
```

### Data Archiving
I stated before that I want to keep all historical data archived in the full detail forever. As I use Google Drive for archiving my photos and files, the data archive is not going to be an exception.

To upload data to Google drive I use [rclone](https://rclone.org). Below is how to install it and start interactive configuration. 

```
sudo curl https://rclone.org/install.sh | sudo bash
sudo rclone config
```

When [rclone](https://rclone.org) is ready we can create a simple shell script which archive one day of data from landing database, store it in folder structure YYYY/YYYYMM and finally copy the missing files into Google drive using the same folder structure.

```
sudo nano influxdb_archive.sh
```

then copy and paste following content:

```
#!/bin/sh
# archives one day of influx landing database into Google Drive

yesterday=$(date --date='yesterday' +"%Y-%m-%d")
today=$(date +"%Y-%m-%d")
thisYear=$(date --date='yesterday' +"%Y")
thisMonth=$(date --date='yesterday' +"%Y%m")

# creates the source data path
mkdir -p "/data/export/googledrive/${thisYear}/${thisMonth}"

# exports and compresses one day of data from influxdb
influx_inspect export -datadir "/data/influxdb/data" -waldir "/data/influxdb/wal" -out "/data/export/googledrive/${thisYear}/${thisMonth}/landing_${yesterday}" -database landing -retention one_month -start "${yesterday}T00:00:00Z" -end "${today}T00:00:00Z" -compress

# makes sure the target data path exists
rclone mkdir myGoogleDrive:InfluxData/lubuntu/${thisYear}/${thisMonth} --config /home/luba/.config/rclone/rclone.conf

# copies the data to google drive
rclone copy /data/export/googledrive myGoogleDrive:InfluxData/lubuntu --config /home/luba/.config/rclone/rclone.conf
```

then introduce new job using crontab:

```
sudo crontab -e
```

adding following row:

```
0 5 * * *     /home/luba/influxdb_archive.sh  >> /home/luba/log_influxdb_archive.out 2>&1
```

### Restoring the archived data

Whenever there is a need to look into detail data from the archive, it can be easily imported:

```
sudo influx -import -path=landing_2018_03_08 -compressed
```