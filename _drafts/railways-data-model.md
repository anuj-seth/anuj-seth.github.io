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
The [dataset](https://data.gov.in/resources/indian-railways-time-table-trains-available-reservation-01112017) I use is the 2017 railway timetable and all the code (including the dataset) can be found [here](https://github.com/anuj-seth/railways-data-model). If you want to follow along then you must have Postgres running.  
&nbsp;  
The dataset contains one row for each station along a train's route and the sequence of stations on the route is given by another column containing a numeric value starting from 1 and monotonically increasing.  

|train_no |  train_name  | seq | station_code | station_name | arrival_time | departure_time | distance | source_station | source_station_name | destination_station | destination_station_name 
----------|--------------|-----|--------------|--------------|--------------|----------------|----------|----------------|---------------------|---------------------|--------------------------|
|      107 | SWV-MAO-VLNK |   1 | SWV          | SAWANTWADI R | 00:00:00     | 10:25:00       |        0 | SWV            | SAWANTWADI ROAD     | MAO                 | MADGOAN JN.|
|      107 | SWV-MAO-VLNK |   2 | THVM         | THIVIM       | 11:06:00     | 11:08:00       |       32 | SWV            | SAWANTWADI ROAD     | MAO                 | MADGOAN JN.|
|      107 | SWV-MAO-VLNK |   3 | KRMI         | KARMALI      | 11:28:00     | 11:30:00       |       49 | SWV            | SAWANTWADI ROAD     | MAO                 | MADGOAN JN.|
|      107 | SWV-MAO-VLNK |   4 | MAO          | MADGOAN JN.  | 12:10:00     | 00:00:00       |       78 | SWV            | SAWANTWADI ROAD     | MAO                 | MADGOAN JN.|
|      108 | VLNK-MAO-SWV |   1 | MAO          | MADGOAN JN.  | 00:00:00     | 20:30:00       |        0 | MAO            | MADGOAN JN.         | SWV                 | SAWANTWADI ROAD|
|      108 | VLNK-MAO-SWV |   2 | KRMI         | KARMALI      | 21:04:00     | 21:06:00       |       33 | MAO            | MADGOAN JN.         | SWV                 | SAWANTWADI ROAD|
|      108 | VLNK-MAO-SWV |   3 | THVM         | THIVIM       | 21:26:00     | 21:28:00       |       51 | MAO            | MADGOAN JN.         | SWV                 | SAWANTWADI ROAD|
|      108 | VLNK-MAO-SWV |   4 | SWV          | SAWANTWADI R | 22:25:00     | 00:00:00       |       83 | MAO            | MADGOAN JN.         | SWV                 | SAWANTWADI ROAD|

The column *train_no* is a unique identifier for every train and is a good candidate to be the primary key while *train_name* is the corresponding user friendly name.  
The column *seq* gives the ordering of the stations on the train's route.  
The information of each station on the route is given by *station_code* and *station_name* columns.  
*arrival_time* and *departure_time* are self describing.  
The column *distance* is the cumulative distance travelled by the train to reach the station. So it makes sense that when *seq* is 1 then the *distance* is 0.  
The columns *source_station*/*source_station_name* and *destination_station*/*destination_station_name* are the origin and final stations for the train.  
&nbsp;  
This table structure is combining two pieces of information in each row - the source and destination of a train and information about an intermediate stop - and should be split into two tables.

|day_number | one_letter_abbreviation | two_letter_abbreviation | three_letter_abbreviation | full_name |
|------------|-------------------------|-------------------------|---------------------------|-----------|
|          0 | S                       | Su                      | Sun                       | Sunday|
|          2 | T                       | Tu                      | Tue                       | Tuesday|
|          3 | W                       | We                      | Wed                       | Wednesday|
|          4 | T                       | Th                      | Thu                       | Thursday|
|          5 | F                       | Fr                      | Fri                       | Friday|
|          6 | S                       | Sa                      | Sat                       | Saturday|
