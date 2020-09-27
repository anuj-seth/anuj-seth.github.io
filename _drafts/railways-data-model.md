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
The Indian Railways runs more than 12000 trains between 8000 stations covering a distance of 120000 Kms and carried 8 billion people in 2019. 9000 trains run each day moving people and freight to all corners of the country.  
Modeling the railways timetable is an interesting exercise as the train routes have to be modeled as a hierarchical or graph structure which is traditionally considered a weak spot of relational databases. Our dataset is large enough to punish a bad design and challenge a good design.   
The [dataset](https://data.gov.in/resources/indian-railways-time-table-trains-available-reservation-01112017) I use is the 2017 railway timetable and all the code (including the dataset) can be found [here](https://github.com/anuj-seth/railways-data-model). If you want to follow along then you must have a Postgres database up and running.  
&nbsp;  
The dataset contains one row for each station along a train's with other information like the arrival and departure time of the train and the distance travelled.  
Let's load this csv to a table so that we can take a better look at the data.  
```
create table staging_trains
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
create temp table all_stations as
select station_code, station_name from staging_trains
union
select source_station, source_station_name from staging_trains
union
select destination_station, destination_station_name from staging_trains;

create temp table all_stations_ranked as
select station_code, station_name, rank() over (partition by station_code order by length(station_name) desc) as r
from all_stations;

create table stations as
select station_code, station_name from all_stations_ranked where r = 1;

```
*NOTE* I choose not to introduce a surrogate key in the ***stations*** table for aesthetic reasons.  
With that out of the way, we can now get down to the job of splitting the original data into two tables - one for each train
```
create table trains
(train_no integer primary key,
 train_name text,
 source_station text,
 source_station_name text,
 destination_station text,
 destination_station_name text);

insert into trains (train_no, train_name, source_station, source_station_name, destination_station, destination_station_name)
select distinct train_no, train_name, source_station, source_station_name, destination_station, destination_station_name from staging_trains;
```
```
create table train_stations
(train_no integer,
 seq smallint,
 station_code text,
 station_name text,
 arrival_time time without time zone,
 departure_time time without time zone,
 distance_from_origin smallint,
 constraint train_station_seq primary key (train_no, seq));

insert into train_stations (train_no, seq, station_code, station_name, arrival_time, departure_time, distance_from_origin)
select train_no, seq, station_code, station_name, arrival_time, departure_time, distance from staging_trains;
```
