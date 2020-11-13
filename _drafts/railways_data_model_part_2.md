---
layout: post
title: "Railways route graph - Part 2 - Recursion and More"
date: 2020-11-03
categories: postgresql
tags: PostgreSQL data-modeling
style: |
  body {
  font-size: 15px;
  } 
---
In [part 1](https://anuj-seth.github.io/postgresql/2020/10/07/railways_data_model_part_1.html) we created a data model to hold the data for our trains and this is what it looked like in the end.  
&nbsp;  
<p align="center">
<img src="{{ site.url }}/assets/images/railways-data-model/railways_data_model_1.jpg">
</p>
&nbsp;  
What kind of questions can we ask of our data ?  
Can I plan my trip using this data ?  
Yes, but we need to learn some things before we can do that.  
_NOTE_
&nbsp;  
### Where can two trains take me ?
Starting from Warangal (station code WL) station what places can I reach if I change trains only once ?  
Why Warangal you ask ?  
I have always liked the name for some reason, it has natural beauty and also has important historical structures dating from the 13th century. More importantly, there are only 2 trains departing from Warangal station so I am hoping that the places I can finally reach will not be huge and I will be able to fit the whole output here.  
I am only interested in the final destinations and not intermediate stops so I can join the ***trains*** table with itself.  
```sql
SELECT distinct train_2.destination_station_code
FROM trains train_1
JOIN trains train_2 ON train_2.source_station_code = train_1.destination_station_code
WHERE train_1.source_station_code = 'WL';

 destination_station_code 
--------------------------
 PBN
 NZM
 AWB
 TVC
 LPI
 KOP
 GR
 JP
 MAS
 TDU
 PAU
 NDLS
 VSKP
 AII
 CSMT
 QLN
 BJP
 FM
 NS
 SKZR
 GTL
 WL
 PUNE
 RXL
 HWH
 KZJ
(26 rows)

```
What if I wanted to change 2 trains or 3 ? Each train change would require one more self-join.  
```sql
SELECT count(distinct train_3.destination_station_code)
FROM trains train_1
JOIN trains train_2 ON train_2.source_station_code = train_1.destination_station_code
JOIN trains train_3 ON train_3.source_station_code = train_2.destination_station_code
WHERE train_1.source_station_code = 'WL';
 count 
-------
   269
```
This strategy cannot scale.  
If you remember the tree or graph traversal algorithms you wrote in college, they always involved recursion and that is what we will use to compute arbitrary number of train changes.  
Recursive queries use the ***WITH*** common table expression and consist of a non-recursive part that is run to select the initial rows from the database.  
To illustrate this I use the ***RECURSIVE*** but my query does not have any recursion.  
```sql
WITH RECURSIVE all_hops AS
(SELECT train_no, train_name, source_station_code, destination_station_code, 1 as depth
 FROM trains
 WHERE source_station_code = 'WL')
SELECT * FROM all_hops;

train_no | train_name | source_station_code | destination_station_code | depth 
----------|------------|---------------------|--------------------------|-------
    67265 | PUSHPULL   | WL                  | HYB                      |     1
    67267 | PUSHPULL   | WL                  | HYB                      |     1
```
This first set of rows is placed in an intermediate table that is used to join with the rows in the subsequent pass.  
Notice the ***depth*** column which is used to control how deep we want to look into the graph and also acts as a marker for the depth at which a particular row was found.  
Recursion is added to the query by using a ***UNION*** to join the recursive and the non-recursive parts.  
Notice how we reference the ***all_hops*** CTE in the recursive part of the query to refer to rows selected in the previous pass. Strictly speaking, this is iteration not recursion but the SQL standards committee decided on that name so we will stick with it.   
```sql
WITH RECURSIVE all_hops AS
(SELECT train_no, train_name, source_station_code, destination_station_code, 1 as depth
 FROM trains
 WHERE source_station_code = 'WL'
 UNION
 SELECT t.train_no, t.train_name, t.source_station_code, t.destination_station_code, hops.depth + 1 as depth
 FROM trains t
 JOIN all_hops hops ON t.source_station_code = hops.destination_station_code
 WHERE hops.depth < 2)
SELECT distinct destination_station_code FROM all_hops;

destination_station_code 
--------------------------
 JP
 PBN
 NS
 SKZR
 GTL
 WL
 MAS
 TDU
 PAU
 CSMT
 NZM
 AWB
 NDLS
 QLN
 TVC
 VSKP
 BJP
 PUNE
 FM
 RXL
 LPI
 KOP
 HYB
 AII
 GR
 HWH
 KZJ
(27 rows)
```
If you have been paying attention, you will notice that the recursive query returned 27 rows while the earlier self-join solution had returned 26 rows. That is because we have selected ***destination_station_code*** at both depths 1 and 2.  
I leave it as an exercise for you to select only the rows at depth 2 and also count how many places you can reach by changing 2 trains.
&nbsp;  
### Find all trains between two stations
If you are travelling from my home town Jalandhar to Amritsar, famous for the Golden Temple and delicious food, what trains can you take ?  
Note that we only look for a single train that passes through Jalandhar and Amritsar, we don't hop trains.  
JUC and ASR are the station codes for Jalandhar and Amritsar, respectively.  
```sql
WITH RECURSIVE all_stations_list AS
(SELECT train_no, seq, station_code, distance_from_origin, ARRAY[station_code] as stations, 1 as depth
 FROM train_stations
 WHERE station_code = 'JUC'
 UNION ALL
 SELECT t_s.train_no, t_s.seq, t_s.station_code, t_s.distance_from_origin, a_s_l.stations || ARRAY[t_s.station_code], a_s_l.depth + 1 as depth
 FROM train_stations t_s
 JOIN all_stations_list a_s_l ON a_s_l.train_no = t_s.train_no AND t_s.seq = a_s_l.seq  + 1)
SELECT distinct train_no FROM all_stations_list WHERE stations @> ARRAY['ASR'] ORDER BY train_no;

train_no 
----------
     1707
    11057
    12013
    .
    .
    .
    .
    22429
    22445
    54601
    64551
    74643
    74923
(51 rows)
```
The interesting thing here is the ***stations*** column which accumulates the stations each train passes through. Once ourrecursion terminates, and it will terminate since each train has only a finite number of stops and there are no loops, we can easily select those trains which passed through ASR by looking into the list of ***stations***.  
You might wonder if we need to go all the way till the end of train's journey if we find the destination station somewhere in the middle of the route. We can avoid doing that by using a where clause in the recursive query to check that ASR does not occur in the ***stations*** array.  
For trains which don't pass through Amritsar we still go till the end of their journey, for others we stop as soon as they reach Amritsar.  
```sql
WITH RECURSIVE all_stations_list AS
(SELECT train_no, seq, station_code, distance_from_origin, ARRAY[station_code] as stations, 1 as depth
 FROM train_stations
 WHERE station_code = 'JUC'
 UNION ALL
 SELECT t_s.train_no, t_s.seq, t_s.station_code, t_s.distance_from_origin, a_s_l.stations || ARRAY[t_s.station_code], a_s_l.depth + 1 as depth
 FROM train_stations t_s
 JOIN all_stations_list a_s_l ON a_s_l.train_no = t_s.train_no AND t_s.seq = a_s_l.seq  + 1
 WHERE 'ASR' != ALL(a_s_l.stations))
SELECT * FROM all_stations_list WHERE stations @> ARRAY['ASR'] ORDER BY train_no, seq;

```
&nbsp;  
### But my company banned recursion
If your co-workers are afraid of recursion then a workaround suggests itself if you look at what our recursive query is doing. It is building up a list of all the stations a train passes.  
If we pre-compute this list so that for each station on a train's route we already know what stations follow it, we can quickly find a train between two stations.  
```sql
CREATE TABLE train_following_stations
(train_no integer,
 seq smallint,
 station_code text NOT NULL REFERENCES stations(station_code),
 following_stations text ARRAY NOT NULL,
 CONSTRAINT train_following_stations_pk PRIMARY KEY (train_no, seq, station_code),
 FOREIGN KEY (train_no, seq) REFERENCES train_stations(train_no, seq));

CREATE INDEX train_following_stations_following_stations_idx ON train_following_stations USING GIN (following_stations);

INSERT INTO train_following_stations
SELECT *
FROM (SELECT train_no,
             seq,
             station_code,
             array_agg(station_code) OVER following_stations_window AS following_stations
      FROM train_stations
      WINDOW following_stations_window AS (PARTITION BY train_no ORDER BY seq ASC ROWS BETWEEN 1 following AND unbounded following)) AS foo
WHERE foo.following_stations IS NOT NULL;
```
The data in this new table looks like this.  
```sql
select * from train_following_stations order by train_no, seq limit 10;

train_no | seq | station_code |                                      following_stations                                       
----------|-----|--------------|-----------------------------------------------------------------------------------------------
      107 |   1 | SWV          | {THVM,KRMI,MAO}
      107 |   2 | THVM         | {KRMI,MAO}
      107 |   3 | KRMI         | {MAO}
      108 |   1 | MAO          | {KRMI,THVM,SWV}
      108 |   2 | KRMI         | {THVM,SWV}
      108 |   3 | THVM         | {SWV}
      128 |   1 | MAO          | {KRMI,THVM,SWV,KUDL,SNDD,KKW,VBW,RAJP,RN,SGR,CHI,KHED,MNI,ROHA,PNVL,KJT,LNL,PUNE,STR,MRJ,KOP}
      128 |   2 | KRMI         | {THVM,SWV,KUDL,SNDD,KKW,VBW,RAJP,RN,SGR,CHI,KHED,MNI,ROHA,PNVL,KJT,LNL,PUNE,STR,MRJ,KOP}
      128 |   3 | THVM         | {SWV,KUDL,SNDD,KKW,VBW,RAJP,RN,SGR,CHI,KHED,MNI,ROHA,PNVL,KJT,LNL,PUNE,STR,MRJ,KOP}
      128 |   4 | SWV          | {KUDL,SNDD,KKW,VBW,RAJP,RN,SGR,CHI,KHED,MNI,ROHA,PNVL,KJT,LNL,PUNE,STR,MRJ,KOP}

```
And the search for trains from Jalandhar to Amritsar is as simple as  
```sql
select train_no from train_following_stations where station_code = 'JUC' and following_stations @> ARRAY['ASR'];

train_no 
----------
     1707
    11057
    12013
    12029
    12031
    .
    .
    .
    .
    54601
    64551
    74643
    74923
(51 rows)
```
&nbsp;  
### Why should I learn recursion then ?
Consider a social network graph where fickle friendships hinge on a like or failure to provide one. Pre-computing the friends of friends there will not work since the graph keeps changing shape. There you will need to traverse the latest web of ~~lies~~ likes all the time.  
For situations like train routes where the data is relatively static - trains don't add or remove stops everyday - pre-computing may be the best solution for single train journeys.
&nbsp;  
### What if I want to find routes with layovers ?
Airline flight searches typically show direct as well as routes with layovers. The Indian government railways booking website does not, neither do some other popular travel booking portals. There may be valid reasons for that but as a programming problem it is interesting, hence I will tackle this in the next part.  
