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
Modeling the railways timetable is an interesting exercise as the data is large enough to punish bad design and challenging even for a good design. Of special interest is the modeling of the train routes so we can quickly find trains between given set of train stations. The train route has to be modeled as a hierarchical or graph structure which is traditionally considered a weak spot of relational databases.   
The [dataset](https://data.gov.in/resources/indian-railways-time-table-trains-available-reservation-01112017) I use is the 2017 railway timetable and all the code (including the dataset) can be found [here](https://github.com/anuj-seth/railways-data-model).  
&nbsp;  
The dataset loaded to a table looks like this  

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

|day_number | one_letter_abbreviation | two_letter_abbreviation | three_letter_abbreviation | full_name |
|------------|-------------------------|-------------------------|---------------------------|-----------|
|          0 | S                       | Su                      | Sun                       | Sunday|
|          2 | T                       | Tu                      | Tue                       | Tuesday|
|          3 | W                       | We                      | Wed                       | Wednesday|
|          4 | T                       | Th                      | Thu                       | Thursday|
|          5 | F                       | Fr                      | Fri                       | Friday|
|          6 | S                       | Sa                      | Sat                       | Saturday|
