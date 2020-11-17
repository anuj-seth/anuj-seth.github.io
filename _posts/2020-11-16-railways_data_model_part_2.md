---
layout: post
title: "Railways route graph - Part 2 - Recursive Queries"
date: 2020-11-16
categories: postgresql
tags: PostgreSQL data-modeling
style: |
  body {
  font-size: 15px;
  } 
---
Can I use the data model from [part 1](https://anuj-seth.github.io/postgresql/2020/10/07/railways_data_model_part_1.html) to plan travels around India ?  
&nbsp;  
<p align="center">
<img src="{{ site.url }}/assets/images/railways-data-model/railways_data_model_1.jpg">
</p>
&nbsp;  
The short answer to that question is yes. The long answer is this blog post.  
&nbsp;  
### Where can one train change from Warangal take me ?
Why did I choose Warangal for this example ? 
I searched the dataset for stations with only one or two originating trains, hoping that the number of stations one train change away would be manageable. Only two trains start from Warangal and the name has a nice ring to it.  
The ***trains*** table will serve my purpose as it has the origin and destination stations for all trains and I only care for final destinations, not intermediate stations. Joining the ***trains*** table with itself, a self-join where the destination of the first train matches the origin of another train, fetches all the relevant rows.  
*NOTE* - WL is the station code for Warangal.  
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
Adding trains to the journey requires more self-joins - one for each train change.  
```sql
SELECT count(distinct train_4.destination_station_code)
FROM trains train_1
JOIN trains train_2 ON train_2.source_station_code = train_1.destination_station_code
JOIN trains train_3 ON train_3.source_station_code = train_2.destination_station_code
JOIN trains train_4 ON train_4.source_station_code = train_3.destination_station_code
WHERE train_1.source_station_code = 'WL';

 count 
-------
   684
```
 This is tedious, inflexible and bound to put any fledgling programmer off relational databases forever but this is exactly the kind of problem where recursion shines. 
&nbsp;  
### Recursive Queries
Most modern relaltional databases support recursive queries as common table expressions (CTE) using the ***WITH*** clause.  
Recursive queries consist of a non-recursive SQL statement and a recursive statement joined by a ***UNION ALL*** or a ***UNION***. The non-recursive part kicks off the process by selecting an initial set of rows from the database, which is contrary to how it works in programming languages where the non-recursive part terminates the recursion. Recursive queries terminate when no more rows are returned by the recursive SQL statement.  
To illustrate the process, I use ***WITH RECURSIVE*** in a query that lacks recursion, thus returning only the initial set of rows.  
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
Take special note of the ***depth*** column - it can be used to control how deep we want to look into the graph or act as a marker when formatting and printing the output data.  
This first set of rows is placed in an intermediate table that is used to join with the rows in the next pass.  
The recursive SQL statement joins the ***all_hops*** CTE with the main table to get the next set of rows - ***all_hops*** contains only the rows selected by the most recent query. As the PostgreSQL manual points out - "this process is iteration not recursion, but RECURSIVE is the terminology chosen by the SQL standards committee".  
```sql
WITH RECURSIVE all_hops AS
(SELECT train_no, train_name, source_station_code, destination_station_code, 1 as depth
 FROM trains
 WHERE source_station_code = 'WL'
 UNION ALL
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
This recursive query returned 27 rows, one more than the self-join solution. That is because I have selected ***destination_station_code*** at depth 1 as well as 2. I leave it as an exercise for the interested reader to select only the rows at depth 2 and also count how many places can be reached by changing 2 trains.  
&nbsp;  
### Find all trains between two stations
What trains can take me from my home town, Jalandhar, to Amritsar - a city famous for mouth-watering food and the Golden Temple - 90 kilometers away ? To keep travel plans simple I don't want to change trains on this journey.  
The query starts by finding all trains passing through Jalandhar and then recursively building the list of subsequent stations on each train's route, stored in an ARRAY data type. All trains with Amritsar (ASR) in their list will satisfy the requirement.   
The query output for two sample trains shows that ***train_no*** 1707 passes through Amritsar while 1708 does not.  
```sql
WITH RECURSIVE all_stations_list AS
(SELECT train_no, seq, station_code, distance_from_origin, ARRAY[]::text[] as stations, 1 as depth
 FROM train_stations
 WHERE station_code = 'JUC'
 UNION ALL
 SELECT t_s.train_no, t_s.seq, t_s.station_code, t_s.distance_from_origin, a_s_l.stations || ARRAY[t_s.station_code], a_s_l.depth + 1 as depth
 FROM train_stations t_s
 JOIN all_stations_list a_s_l ON a_s_l.train_no = t_s.train_no AND t_s.seq = a_s_l.seq  + 1)
SELECT train_no, seq, station_code, stations FROM all_stations_list ORDER BY train_no, seq LIMIT 18;

 train_no | seq | station_code |                        stations                         
----------|-----|--------------|---------------------------------------------------------
     1707 |  14 | JUC          | {}
     1707 |  15 | BEAS         | {BEAS}
     1707 |  16 | ASR          | {BEAS,ASR}
     1707 |  17 | ATT          | {BEAS,ASR,ATT}
     1708 |   4 | JUC          | {}
     1708 |   5 | LDH          | {LDH}
     1708 |   6 | UMB          | {LDH,UMB}
     1708 |   7 | NDLS         | {LDH,UMB,NDLS}
     1708 |   8 | MTJ          | {LDH,UMB,NDLS,MTJ}
     1708 |   9 | AGC          | {LDH,UMB,NDLS,MTJ,AGC}
     1708 |  10 | GWL          | {LDH,UMB,NDLS,MTJ,AGC,GWL}
     1708 |  11 | JHS          | {LDH,UMB,NDLS,MTJ,AGC,GWL,JHS}
     1708 |  12 | LAR          | {LDH,UMB,NDLS,MTJ,AGC,GWL,JHS,LAR}
     1708 |  13 | MAKR         | {LDH,UMB,NDLS,MTJ,AGC,GWL,JHS,LAR,MAKR}
     1708 |  14 | SGO          | {LDH,UMB,NDLS,MTJ,AGC,GWL,JHS,LAR,MAKR,SGO}
     1708 |  15 | DMO          | {LDH,UMB,NDLS,MTJ,AGC,GWL,JHS,LAR,MAKR,SGO,DMO}
     1708 |  16 | KMZ          | {LDH,UMB,NDLS,MTJ,AGC,GWL,JHS,LAR,MAKR,SGO,DMO,KMZ}
     1708 |  17 | JBP          | {LDH,UMB,NDLS,MTJ,AGC,GWL,JHS,LAR,MAKR,SGO,DMO,KMZ,JBP}
```
This query keeps going till the end of each train's journey even if it finds Amritsar in the middle of the route. The extra work can be avoided by removing rows which have ASR in their ***stations*** array from the recursion. For trains which do not pass through Amritsar we still go till the end of their journey.  
```sql
WITH RECURSIVE all_stations_list AS
(SELECT train_no, seq, station_code, distance_from_origin, ARRAY[]::text[] as stations, 1 as depth
 FROM train_stations
 WHERE station_code = 'JUC'
 UNION ALL
 SELECT t_s.train_no, t_s.seq, t_s.station_code, t_s.distance_from_origin, a_s_l.stations || ARRAY[t_s.station_code], a_s_l.depth + 1 as depth
 FROM train_stations t_s
 JOIN all_stations_list a_s_l ON a_s_l.train_no = t_s.train_no AND t_s.seq = a_s_l.seq  + 1
 WHERE 'ASR' != ALL(a_s_l.stations))
SELECT train_no, seq, station_code, stations FROM all_stations_list ORDER BY train_no, seq LIMIT 17;

 train_no | seq | station_code |                        stations                         
----------|-----|--------------|---------------------------------------------------------
     1707 |  14 | JUC          | {}
     1707 |  15 | BEAS         | {BEAS}
     1707 |  16 | ASR          | {BEAS,ASR}
     1708 |   4 | JUC          | {}
     1708 |   5 | LDH          | {LDH}
     1708 |   6 | UMB          | {LDH,UMB}
     1708 |   7 | NDLS         | {LDH,UMB,NDLS}
     1708 |   8 | MTJ          | {LDH,UMB,NDLS,MTJ}
     1708 |   9 | AGC          | {LDH,UMB,NDLS,MTJ,AGC}
     1708 |  10 | GWL          | {LDH,UMB,NDLS,MTJ,AGC,GWL}
     1708 |  11 | JHS          | {LDH,UMB,NDLS,MTJ,AGC,GWL,JHS}
     1708 |  12 | LAR          | {LDH,UMB,NDLS,MTJ,AGC,GWL,JHS,LAR}
     1708 |  13 | MAKR         | {LDH,UMB,NDLS,MTJ,AGC,GWL,JHS,LAR,MAKR}
     1708 |  14 | SGO          | {LDH,UMB,NDLS,MTJ,AGC,GWL,JHS,LAR,MAKR,SGO}
     1708 |  15 | DMO          | {LDH,UMB,NDLS,MTJ,AGC,GWL,JHS,LAR,MAKR,SGO,DMO}
     1708 |  16 | KMZ          | {LDH,UMB,NDLS,MTJ,AGC,GWL,JHS,LAR,MAKR,SGO,DMO,KMZ}
     1708 |  17 | JBP          | {LDH,UMB,NDLS,MTJ,AGC,GWL,JHS,LAR,MAKR,SGO,DMO,KMZ,JBP}

```
Finally, restricting the output to rows with ASR in their ***stations*** array gives the trains which can take me from Jalandhar to Amritsar.  
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
SELECT a_s_l.train_no, trains.train_name 
FROM all_stations_list a_s_l 
JOIN trains ON trains.train_no = a_s_l.train_no 
WHERE stations @> ARRAY['ASR'] 
ORDER BY train_no;

train_no |  train_name  
----------|--------------
     1707 | JBP-ATT BI-W
    11057 | AMRITSAR EXP
    12013 | NDLS-ASR SHA
    12029 | SWARNA SHATA
    12031 | NDLS-ASR-SHA
    12053 | HW-ASR EXP
    12203 | GARIB RATH
    12241 | CDG-ASR SUPE
    12317 | AKAL TAKHT E
    12357 | DURGIANA EXP
    .
    .
    .
    .
    64551 | LDH-ASR MEMU
    74643 | JUC-ASR DMU
    74923 | HSX-ASR DMU
(51 rows)
```
&nbsp;  
### If your company has a no recursion policy
You can tell your boss that this is iteration and not true recursion - as mentioned in the fine PostgreSQL manual. If that does not pass muster with your manager, all is still not lost.  
The list of stations, coming after another station on a train's route, can be precomputed and stored in a table.  
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
SELECT * FROM train_following_stations WHERE station_code = 'JUC' ORDER BY train_no LIMIT 10;

train_no | seq | station_code | following_stations  
----------------------------------------------------------------
     1707 |  14 | JUC          | {BEAS,ASR,ATT}
     1708 |   4 | JUC          | {LDH,UMB,NDLS,MTJ,AGC,GWL,JHS,LAR,MAKR,SGO,DMO,KMZ,JBP}
    11057 |  87 | JUC          | {BEAS,JNL,ASR}
    11058 |   5 | JUC          | {JRC,PGW,GRY,PHR,LDH,AHH,MET,DUI,NBA,PTA,RPJ,UBC,UMB,SHDM,KKDE,NLKR,TRR,KUN,GRA,PNP,SMK,GNU,SNP,RDDE,NUR,ANDI,SZM,NDLS,NZM,FDB,BVH,PWL,KSV,MTJ,RKM,AGC,DHO,MRA,GWL,DBA,SOR,DAA,JHS,BAB,BZY,TBT,LAR,JLN,DUA,BINA,MABA,BAQ,GLG,BHS,SCI,BPL,HBJ,MDDP,ODG,BNI,HBD,ET,BPF,TBN,HD,KKN,CAER,TLV,KNW,NPNR,BAU,RV,NB,SAV,BSL,JL,PC,CSN,MMR,NK,DVL,IGP,KYN,TNA,DR,CSMT}
    12013 |   6 | JUC          | {BEAS,ASR}
    12014 |   3 | JUC          | {PGW,LDH,SIR,UMB,NDLS}
    12029 |   6 | JUC          | {BEAS,ASR}
    12030 |   3 | JUC          | {PGW,LDH,RPJ,UMB,NDLS}
    12031 |   6 | JUC          | {BEAS,ASR}
    12032 |   3 | JUC          | {PGW,LDH,RPJ,UMB,NDLS}
```
And I can get trains from Jalandhar to Amritsar with this query.  
```sql
SELECT train_no FROM train_following_stations WHERE station_code = 'JUC' AND following_stations @> ARRAY['ASR'];

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
### Should I get rid of all my recursive queries ?
Consider a social network graph where fickle friendships hinge on a like or failure to provide one. Precomputing friends of friends will not work in this case as the graph keeps changing shape. There you will need to traverse the latest web of likes or lies each time.  
For situations like train routes where the data is relatively static - trains do not add or remove stops everyday - precomputing can be the best solution for single train journeys.
&nbsp;  
### What if I want to find routes with layovers ?
Airline flight searches typically show routes with layovers but the Indian government train ticket booking website does not. It only shows a single train that can take you from your origin to the destination station or nothing at all and other popular travel booking portals follow suit.  
Why do travel portals not support this feature ? I don't know.  
Is it technically possible ? I will try to answer this in the next part.  
