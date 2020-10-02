---
layout: post
title: "Railways timetable data model - Part 1"
date: 2020-09-20
categories: data-modeling
tags: data-modeling postgresql
style: |
  body {
  font-size: 15px;
  } 
---
The Indian Railways runs more than 12000 trains between 8000 stations covering a distance of 120000 Kms. It carried 8 billion people in 2019 with 9000 trains running each day, moving people and freight to all corners of the country.  
&nbsp;  
Why is modeling the train timetable an intersting exercise for a relational data modeler ?  
It's definitely not the size of the data. The Indian railways network is huge but the train timetable data, assuming 15 stops per train multiplied by 12000 trains, is at most 300K rows. That number would not cause any modern database engine to break a sweat.  
Things get interesting when you consider that all the stops along a train's route have to be kept in one table with linkage from one stop to the next one on the route. To find a train between a pair of stations a query would have to move along these linkages from one row to next until it finds the station it is looking for. In relational database terms these are called self joins - manageable if you know in advance how many self joins are needed and even then a maximum of 3 or 4 of them. With our data the station we are looking for could be the next stop or hundred stops removed from the starting station.  
Modeling this parent child or hierarchical relationship of arbitrary depth is what makes this interesting.  
&nbsp;  
The [dataset](https://data.gov.in/resources/indian-railways-time-table-trains-available-reservation-01112017) used is the 2017 railway timetable and all the code (including the dataset) can be found [here](https://github.com/anuj-seth/railways-data-model). Have a Postgres database up and running if you want to follow along by running the SQL statements in this blog post.  
&nbsp;  
The dataset contains one row for each station along a train's route with other information like the arrival and departure time and the distance travelled.  
Let's load this csv to a table so that we can take a better look at the data.  
```
CREATE TABLE staging_trains
(train_no integer,
 train_name text,
 seq smallint,
 station_code text,
 station_name text,
 arrival_time time without time zone,
 departure_time time without time zone,
 distance smallint,
 source_station text,
 source_station_name text,
 destination_station text,
 destination_station_name text);
 
\copy staging_trains from 'Train_details_22122017.csv' csv header;
```
The column ***train_no*** is a unique identifier for every train and is a good candidate to be the primary key while ***train_name*** is the corresponding user friendly name.  
The column ***seq*** gives the ordering of the stations on the train's route. It is a number starting from 1 and monotonically increasing.  
The information of each station on the route is given by ***station_code*** and ***station_name*** columns.  
***arrival_time*** and ***departure_time*** are self describing.  
The column ***distance*** is the cumulative distance travelled by the train to reach the station. So it makes sense that when ***seq*** is 1 the ***distance*** is 0.  
The columns ***source_station***/***source_station_name*** and ***destination_station***/***destination_station_name*** are the origin and final stations for the train.  
```
select * from staging_trains limit 8;

train_no |  train_name  | seq | station_code | station_name | arrival_time | departure_time | distance | source_station | source_station_name | destination_station | destination_station_name 
----------|--------------|-----|--------------|--------------|--------------|----------------|----------|----------------|---------------------|---------------------|--------------------------
      107 | SWV-MAO-VLNK |   1 | SWV          | SAWANTWADI R | 00:00:00     | 10:25:00       |        0 | SWV            | SAWANTWADI ROAD     | MAO                 | MADGOAN JN.
      107 | SWV-MAO-VLNK |   2 | THVM         | THIVIM       | 11:06:00     | 11:08:00       |       32 | SWV            | SAWANTWADI ROAD     | MAO                 | MADGOAN JN.
      107 | SWV-MAO-VLNK |   3 | KRMI         | KARMALI      | 11:28:00     | 11:30:00       |       49 | SWV            | SAWANTWADI ROAD     | MAO                 | MADGOAN JN.
      107 | SWV-MAO-VLNK |   4 | MAO          | MADGOAN JN.  | 12:10:00     | 00:00:00       |       78 | SWV            | SAWANTWADI ROAD     | MAO                 | MADGOAN JN.
      108 | VLNK-MAO-SWV |   1 | MAO          | MADGOAN JN.  | 00:00:00     | 20:30:00       |        0 | MAO            | MADGOAN JN.         | SWV                 | SAWANTWADI ROAD
      108 | VLNK-MAO-SWV |   2 | KRMI         | KARMALI      | 21:04:00     | 21:06:00       |       33 | MAO            | MADGOAN JN.         | SWV                 | SAWANTWADI ROAD
      108 | VLNK-MAO-SWV |   3 | THVM         | THIVIM       | 21:26:00     | 21:28:00       |       51 | MAO            | MADGOAN JN.         | SWV                 | SAWANTWADI ROAD
      108 | VLNK-MAO-SWV |   4 | SWV          | SAWANTWADI R | 22:25:00     | 00:00:00       |       83 | MAO            | MADGOAN JN.         | SWV                 | SAWANTWADI ROAD

```
&nbsp;  
The first thing that jumps out when looking at this data is that it combines two pieces of information in each row - the source and destination of a train and the information of an intermediate stop. We should split this information into two tables but before we do that there's another problem that needs to be addressed first.  
Duplicating data opens the door for discrepancies and we see this in the very first row above. The columns ***station_name*** and ***source_station_name*** hold slightly different values - SAWANTWADI R vs SAWANRWADI ROAD - even though both use the same station code value.
Let's fix this by extracting all the station codes and names to a new table - whenever we have a station code with different data in the name column we will take the longest string on the assumption that longer name is better.  
```
CREATE TEMP TABLE all_stations AS
SELECT station_code, station_name FROM staging_trains
UNION
SELECT source_station, source_station_name FROM staging_trains
UNION
SELECT destination_station, destination_station_name FROM staging_trains;

CREATE TEMP TABLE all_stations_ranked AS
SELECT station_code, 
       station_name, 
       rank() OVER (PARTITION BY station_code ORDER BY length(station_name) desc) AS r
FROM all_stations;

CREATE TABLE stations AS
SELECT station_code, station_name FROM all_stations_ranked
WHERE r = 1;

ALTER TABLE stations ADD PRIMARY KEY (station_code);

ALTER TABLE stations ALTER COLUMN station_name SET NOT NULL;
```
*NOTE* - I choose not to introduce a surrogate key in the ***stations*** table as all our queries will specify the station codes directly.  
With that out of the way, we can now get down to the job of splitting the original data into two tables - one for each train and the other for all the stations on a train's route.  
```
CREATE TABLE trains
(train_no integer PRIMARY KEY,
 train_name text NOT NULL,
 source_station_code text NOT NULL REFERENCES stations(station_code),
 destination_station_code text NOT NULL REFERENCES stations(station_code));

CREATE INDEX trains_source_station_code_idx ON trains(source_station_code);

CREATE INDEX trains_destination_station_code_idx ON trains(destination_station_code);

INSERT INTO trains
(train_no, train_name, source_station_code, destination_station_code)
SELECT distinct train_no, train_name, source_station, destination_station FROM staging_trains;

CREATE TABLE train_stations
(train_no integer,
 seq smallint,
 station_code text NOT NULL REFERENCES stations(station_code),
 arrival_time time without time zone NOT NULL,
 departure_time time without time zone NOT NULL,
 distance_from_origin smallint NOT NULL,
 CONSTRAINT train_stations_pk PRIMARY KEY (train_no, seq));

CREATE INDEX train_stations_station_code_idx ON train_stations(station_code);

INSERT INTO train_stations
(train_no, seq, station_code, arrival_time, departure_time, distance_from_origin)
SELECT train_no, seq, station_code, arrival_time, departure_time, distance 
FROM staging_trains;
```
The datamodel looks like this now
<p align="center">
<img src="{{ site.url }}/assets/images/railways-data-model/railways_data_model_1.jpg">
</p>
### When does a train arrive at it's origin or leave from the destination ?
Let's take another look at our ***train_stations*** table.
```
select * from train_stations limit 4;

train_no | seq | station_code | arrival_time | departure_time | distance_from_origin 
---------|-----|--------------|--------------|----------------|----------------------
     107 |   1 | SWV          | 00:00:00     | 10:25:00       |                    0
     107 |   2 | THVM         | 11:06:00     | 11:08:00       |                   32
     107 |   3 | KRMI         | 11:28:00     | 11:30:00       |                   49
     107 |   4 | MAO          | 12:10:00     | 00:00:00       |                   78

```
The ***arrival_time*** at the originating station and the ***departure_time*** from the destination are set as midnight (00:00:00). These are, obviously, dummy values as there is no valid arrival time at the origin or departure time for the terminating station.  
Choosing the value of midnight creates problems for queries which try to find trains arriving or leaving at midnight.  
```
select * from train_stations t_s where arrival_time = '00:00:00' and station_code = 'NDLS';

 train_no | seq | station_code | arrival_time | departure_time | distance_from_origin 
----------|-----|--------------|--------------|----------------|----------------------
    12310 |   1 | NDLS         | 00:00:00     | 17:15:00       |                    0
    12562 |   1 | NDLS         | 00:00:00     | 20:40:00       |                    0
    12622 |   1 | NDLS         | 00:00:00     | 22:30:00       |                    0
    22412 |   1 | NDLS         | 00:00:00     | 15:50:00       |                    0

```
The correct query would have to take the value in the ***seq*** column into account which is easy to do for originating stations as we know that ***seq*** will be 1 in that case.  
```
select * from train_stations t_s where arrival_time = '00:00:00' and station_code = 'NDLS' and seq <> 1;

 train_no | seq | station_code | arrival_time | departure_time | distance_from_origin 
----------|-----|--------------|--------------|----------------|----------------------
(0 rows)
```
To find all trains leaving at midnight we will have to filter the rows which belong to termination points or the maximum ***seq*** on a train's route.  
```
SELECT *
FROM train_stations t_s_1
WHERE departure_time = '00:00:00'
and station_code = 'NDLS'
AND (train_no, seq) NOT IN (SELECT train_no, max(seq)
                            FROM train_stations t_s_2
                            WHERE t_s_2.station_code = t_s_1.station_code
                            GROUP BY t_s_2.train_no);
                            
 train_no | seq | station_code | arrival_time | departure_time | distance_from_origin 
----------|-----|--------------|--------------|----------------|----------------------
(0 rows)

```
What if we use null as the dummy value instead of midnight ? That seems like a reasonable compromise that will also simplify our queries.  
Before we can put nulls in these columns  we have to drop some constraints.  
```
ALTER TABLE train_stations ALTER COLUMN arrival_time DROP NOT NULL;

ALTER TABLE train_stations ALTER COLUMN departure_time DROP NOT NULL;

```
And then update our data.  
```
UPDATE train_stations SET arrival_time = NULL WHERE seq = 1;

UPDATE train_stations SET departure_time = NULL
WHERE (train_no, seq) IN (SELECT train_no, max(seq) as seq
                          FROM train_stations
                          GROUP BY train_no);
```
And now our queries are simple.  

```
select min(stops), max(stops), avg(stops) from (select count(*) as stops from train_stations group by train_no) as foo;

 min | max |         avg     
-----|-----|---------------------
   2 | 118 | 16.7493700503959683
```
```
select t.*, s.station_name from trains t join stations s on s.station_code = t.source_station_code where source_station_code = destination_station_code;
 train_no |  train_name  | source_station_code | destination_station_code |     station_name     
----------|--------------|---------------------|--------------------------|----------------------
    52599 | DJ - GHUM -  | DJ                  | DJ                       | DARJEELING
    52592 | DJ - GHUM -  | DJ                  | DJ                       | DARJEELING
    52597 | DJ - GHUM -  | DJ                  | DJ                       | DARJEELING
    52598 | DJ-GHUM-DJ J | DJ                  | DJ                       | DARJEELING
    64091 | HNZM-HNZM EM | NZM                 | NZM                      | HAZRAT NIZAMUDDIN JN
    52596 | DJ-GHUM-DJ J | DJ                  | DJ                       | DARJEELING
    52593 | DJ - GHUM -  | DJ                  | DJ                       | DARJEELING
    52594 | DJ - GHUM -  | DJ                  | DJ                       | DARJEELING
      290 | PALACE ON WH | DSJ                 | DSJ                      | DELHI-SAFDAR JANG
    52591 | DJ - GHUM -  | DJ                  | DJ                       | DARJEELING
      477 | FTR TRAIN NO | SSA                 | SSA                      | SIRSA
    64089 | NZM-NZM EMU  | NZM                 | NZM                      | HAZRAT NIZAMUDDIN JN
    52595 | DJ-GHUM-DJ J | DJ                  | DJ                       | DARJEELING
    64090 | NZM-NZM EMU  | NZM                 | NZM                      | HAZRAT NIZAMUDDIN JN
    64092 | NZM-NZM EMU  | NZM                 | NZM                      | HAZRAT NIZAMUDDIN JN
```
What are the start and end times of a train ?

What is the average time a train stops at a station ?