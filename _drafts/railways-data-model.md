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
Things get interesting when you consider that all the stops along a train's route would have to be kept in one table with linkage from one stop to the next one on the route. To find a train between a pair of stations a query would have to move along these linkages from one row to next until it finds the station it is looking for. In relational database terms these are called self joins - only manageable if you know in advance how many self joins are needed. With our data the station we are looking for could be the next stop or hundred stops removed from the starting station.  
Modeling this parent child or hierarchical relationship of arbitrary depth is what makes this interesting.  
&nbsp;  
The [dataset](https://data.gov.in/resources/indian-railways-time-table-trains-available-reservation-01112017) I use is the 2017 railway timetable and all the code (including the dataset) can be found [here](https://github.com/anuj-seth/railways-data-model). If you want to follow along then you must have a Postgres database up and running.  
&nbsp;  
The dataset contains one row for each station along a train's with other information like the arrival and departure time of the train and the distance travelled.  
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
The columns ***station_name***, ***source_station_name*** and ***destination_station_name*** should all hold the same string for a ***station_code*** but the data breaks this rule in the very first row of the output above. The columns ***station_name*** and ***source_station_name*** hold slightly different values - SAWANTWADI R vs SAWANRWADI ROAD - for the same station.
Duplicating the same information in multiple places wastes space, but more significantly it opens the door for discrepancies to creep in when the information is updated by different programs or at different times.  
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
With that out of the way, we can now get down to the job of splitting the original data into two tables - one for each train and a second table for the intermediate stops on a train's route.  
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
<p align="center">
<img src="{{ site.url }}/assets/images/railways-data-model/railways_data_model_1.jpg">
</p>
```
select min(stops), max(stops), avg(stops) from (select count(*) as stops from train_stations group by train_no) as foo;

 min | max |         avg     
-----|-----|---------------------
   2 | 118 | 16.7493700503959683
```
