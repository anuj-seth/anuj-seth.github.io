---
layout: post
title: "Railways route graph - Part 3 - Breaking cycles"
date: 2020-11-29
categories: postgresql
tags: PostgreSQL data-modeling
style: |
  body {
  font-size: 15px;
  } 
---
Does my data have cycles ?  
How can I detect the cycles ?
How many rows are there with beas as the destination station at depth 1 ?


    AB5
    BC4
    CD8
    DC8
    DE6
    AD5
    CE2
    EB3
    AE7

```sql
CREATE TABLE sample_routes
(train_no integer NOT NULL,
 origin_seq smallint NOT NULL,
 origin_station_code text NOT NULL,
 destination_seq smallint NOT NULL,
 destination_station_code text NOT NULL,
 distance smallint NOT NULL
);

insert into sample_routes
values
(1, 1, 'A', 2, 'B', 5),
(1, 2, 'B', 3, 'C', 4), 
(1, 3, 'C', 4, 'D', 8),
(2, 1, 'D', 2, 'E', 6),
(2, 2, 'E', 3, 'B', 3),
(3, 1, 'A', 2, 'D', 5),
(4, 1, 'A', 2, 'E', 7),
(5, 1, 'D', 2, 'C', 8),
(6, 1, 'C', 2, 'E', 2);
```

```sql
drop table sample_train_stations;

CREATE TABLE sample_train_stations
(train_station_id serial,
 train_no smallint NOT NULL,
 seq smallint NOT NULL,
 station_code text NOT NULL
);

insert into sample_train_stations
(train_no, seq, station_code)
values
(1, 1, 'A'),
(1, 2, 'B'),
(1, 3, 'C'), 
(1, 4, 'D'),
(2, 1, 'D'),
(2, 2, 'E'),
(2, 3, 'B'),
(3, 1, 'A'), 
(3, 2, 'D'),
(4, 1, 'A'),
(4, 2, 'E'),
(5, 1, 'D'), 
(5, 2, 'C'),
(6, 1, 'C'),
(6, 2, 'E');
```

    The distance of the route A-B-C. Ans: 9
    The distance of the route A-D. Ans: 5
    The distance of the route A-D-C. Ans: 13
    The distance of the route A-E-B-C-D. Ans: 22
    The distance of the route A-E-D. Ans: No such route
    The number of trips starting at C and ending at C with a maximum of 3 stops. In the sample data below, there are two such trips: C-D-C (2 stops). and C-E-B-C (3 stops).
    The number of trips starting at A and ending at C with exactly 4 stops. In the sample data below, there are three such trips: A to C (via B,C,D); A to C (via D,C,D); and A to C (via D,E,B).
    The length of the shortest route (in terms of distance to travel) from A to C. Ans: 9
    The length of the shortest route (in terms of distance to travel) from B to B.  Ans: 9 (how?) B-C-E-B
    The number of different routes from C to C with a distance of less than 30. In the sample data, the trips are: CDC, CEBC, CEBCDC, CDCEBC, CDEBC, CEBCEBC, CEBCEBCEBC.

```sql 

with recursive multi_train_journeys as
(select train_no, origin_seq, origin_station_code, destination_seq, destination_station_code, distance, 0 as depth
from sample_routes
where origin_station_code = 'A'
UNION ALL
select s_r.train_no, s_r.origin_seq, s_r.origin_station_code, s_r.destination_seq, s_r.destination_station_code, s_r.distance, m_t_j.depth + 1 as depth
from sample_routes s_r
join multi_train_journeys m_t_j on m_t_j.destination_station_code = s_r.origin_station_code
where s_r.destination_station_code <> m_t_j.origin_station_code
and m_t_j.depth < 2
)
select count(*) from multi_train_journeys;

 count 
-------
    41
(1 row)
```

```sql 

with recursive multi_train_journeys as
(select train_no, origin_seq, origin_station_code, departure_from_origin, destination_seq, destination_station_code, arrival_at_destination, 0 as depth
from trains_route_graph
where origin_station_code = 'JUC'
UNION ALL
select t_r_g.train_no, t_r_g.origin_seq, t_r_g.origin_station_code, t_r_g.departure_from_origin, t_r_g.destination_seq, t_r_g.destination_station_code, t_r_g.arrival_at_destination, m_t_j.depth + 1 as depth
from trains_route_graph t_r_g
join multi_train_journeys m_t_j on m_t_j.destination_station_code = t_r_g.origin_station_code
where t_r_g.destination_station_code <> m_t_j.origin_station_code
and m_t_j.depth < 2
)
select count(*) from multi_train_journeys where depth = 0 and destination_station_code = 'BEAS';

 count 
-------
    41
(1 row)
```
And how many have BEAS as the origin station at depth 1 ?
```sql
with recursive multi_train_journeys as
(select train_no, origin_seq, origin_station_code, departure_from_origin, destination_seq, destination_station_code, arrival_at_destination, 0 as depth
from trains_route_graph
where origin_station_code = 'JUC'
UNION ALL
select t_r_g.train_no, t_r_g.origin_seq, t_r_g.origin_station_code, t_r_g.departure_from_origin, t_r_g.destination_seq, t_r_g.destination_station_code, t_r_g.arrival_at_destination, m_t_j.depth + 1 as depth
from trains_route_graph t_r_g
join multi_train_journeys m_t_j on m_t_j.destination_station_code = t_r_g.origin_station_code
where t_r_g.destination_station_code <> m_t_j.origin_station_code
and m_t_j.depth < 2
)
select count(*) from multi_train_journeys where depth = 1 and origin_station_code = 'BEAS';

count 
-------
  2665
(1 row)

```

```sql
vodafone=# select count(*) from trains_route_graph where origin_station_code = 'JUC' and destination_station_code = 'BEAS';
 count 
-------
    41
(1 row)

vodafone=# select count(*) from trains_route_graph where origin_station_code = 'BEAS' and destination_station_code != 'JUC';
 count 
-------
    65
(1 row)

```
And this should be fine since for each train we should follow outgoing trains from BEAS.
This is not a cycle.

with recursive multi_train_journeys as
(select edge_id, train_no, origin_seq, origin_station_code, departure_from_origin, destination_seq, destination_station_code, arrival_at_destination, 0 as depth, ARRAY['JUC'] as stations, ARRAY[edge_id] as edges
from trains_route_graph
where origin_station_code = 'JUC'
UNION ALL
select t_r_g.edge_id, t_r_g.train_no, t_r_g.origin_seq, t_r_g.origin_station_code, t_r_g.departure_from_origin, t_r_g.destination_seq, t_r_g.destination_station_code, t_r_g.arrival_at_destination, m_t_j.depth + 1 as depth, m_t_j.stations || ARRAY[t_r_g.origin_station_code], m_t_j.edges || ARRAY[t_r_g.edge_id]
from trains_route_graph t_r_g
join multi_train_journeys m_t_j on m_t_j.destination_station_code = t_r_g.origin_station_code
where t_r_g.destination_station_code <> ALL(m_t_j.stations) 
and m_t_j.depth < 2
)
select * from multi_train_journeys where 'NDLS' = ANY(stations); 

--order by train_no, origin_seq, origin_station_code, departure_from_origin, destination_seq, destination_station_code, arrival_at_destination, depth
;
